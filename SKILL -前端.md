# worker-h5 — 前端数字员工

从 PRD 到测试环境上线的一键全流程：ONES 分支创建 → PRD + API 文档拉取 → 前端代码生成 → Git 提交 → Talos 部署 + 等待完成。

## 调用方式

```
/worker-h5 <prd-url> <ones-url>                        # 全流程（api-url 从 prd 子文档自动发现）
/worker-h5 <prd-url> <api-url> <ones-url>              # 全流程，显式指定 api-url
/worker-h5 <prd-url> <ones-url> --swimlane=<name>      # 全流程，显式指定泳道名
/worker-h5 <name> --only=<step>                        # 只执行指定步骤
/worker-h5 --list-steps                                # 查看所有步骤
```

### `--only` 参数对照表

| 参数值 | 步骤 | 说明 |
|--------|------|------|
| `branch` | Step 0 | 创建 ONES 分支 |
| `fetch` | Step 1 | 从学城拉取 PRD + API 文档 |
| `code` | Step 2 | 前端代码生成 |
| `git` | Step 3 | Git 提交 |
| `deploy` | Step 4 | Talos 部署 + 等待完成 |

---

## 配置

启动时读取 `$PWD/.worker/h5.json`，文件不存在时报错退出：

```
❌ 找不到 .worker/h5.json，请先创建配置文件：

{
  "ones": {
    "branchType": "feature",
    "baseBranch": "master"
  },
  "talos": {
    "platform": "",
    "appId": "",
    "templateId": "",
    "target": "newtest",
    "pollIntervalSeconds": 15,
    "timeoutMinutes": 10
  }
}
```

```json
{
  "ones": {
    "branchType": "feature",
    "baseBranch": "master"
  },
  "talos": {
    "platform": "v1",
    "appId": "24845",
    "templateId": "189215",
    "target": "newtest",
    "pollIntervalSeconds": 15,
    "timeoutMinutes": 10
  }
}
```

**字段说明：**

| 字段 | 说明 |
|------|------|
| `ones.branchType` | 分支类型，默认 `feature` |
| `ones.baseBranch` | 基准分支，默认 `master` |
| `talos.platform` | 平台版本，`v1` 或 `v2` |
| `talos.appId` | Talos 项目 ID |
| `talos.templateId` | 发布模板 ID |
| `talos.target` | 目标环境，仅支持 `test01`~`test08`、`newtest` |
| `talos.pollIntervalSeconds` | 轮询间隔（秒），默认 15 |
| `talos.timeoutMinutes` | 部署超时（分钟），默认 10 |

---

## 执行流程

### 准备：读取配置和参数

```bash
PROJECT_ROOT="$PWD"
WORKER_START_TIME=$(date +%s)

# 读取配置
CONFIG=$(cat $PROJECT_ROOT/.worker/h5.json)
ONES_BRANCH_TYPE=$(echo $CONFIG | jq -r '.ones.branchType')
ONES_BASE_BRANCH=$(echo $CONFIG | jq -r '.ones.baseBranch')
TALOS_PLATFORM=$(echo $CONFIG | jq -r '.talos.platform')
TALOS_APP_ID=$(echo $CONFIG | jq -r '.talos.appId')
TALOS_TEMPLATE_ID=$(echo $CONFIG | jq -r '.talos.templateId')
TALOS_TARGET=$(echo $CONFIG | jq -r '.talos.target')
TALOS_POLL_INTERVAL=$(echo $CONFIG | jq -r '.talos.pollIntervalSeconds // 15')
TALOS_TIMEOUT_MINUTES=$(echo $CONFIG | jq -r '.talos.timeoutMinutes // 10')
```

环境检查（复用 talos-project-publish 的脚本）：

```bash
source /Users/liangyanghe/.claude/skills/talos-project-publish/check-env.sh
```

退出码非 0 时报错退出。

解析 ARGUMENTS，确定 `NAME`、`ONES_URL`、`PRD_URL`、`API_URL`（可选）、`TALOS_SWIMLANE`（可选），以及执行模式（全流程 / `--only`）。

`--swimlane=<value>` 出现时将其值赋给 `TALOS_SWIMLANE`；未出现时 `TALOS_SWIMLANE` 为空，**不在此处提示**，延迟到 Step 4 执行前按需提示。`--from` 或 `--to` 出现时立即报错退出。

输出：
```
🤖 [worker-h5] 开始执行：<name>
```

---

### Step 0：创建 ONES 分支

需要 `<ones-url>` 参数。未传入时报错：`❌ 缺少 ONES 工作项链接，创建分支需要提供 ones-url`

```bash
# 解析 ONES URL，提取 ONES_SPACE_ID 和 ONES_ITEM_ID
ones url-parse -u "<ones-url>"

# 查询该工作项下是否已有当前 appkey 的分支
EXISTING_BRANCH=$(ones branch list -p $ONES_SPACE_ID -i $ONES_ITEM_ID --json | \
  jq -r '[.items[] | select(.appkey=="com.sankuai.freelance.operation.web")] | first | .name // empty')
```

**分两条路：**

**已有分支（`EXISTING_BRANCH` 非空）→ 直接复用，记录分支名：**
```bash
BRANCH_NAME=$EXISTING_BRANCH
```

**无分支（`EXISTING_BRANCH` 为空）→ 新建分支：**
```bash
# 获取应用 appId
APP_ID=$(ones apps -p $ONES_SPACE_ID -n "com.sankuai.freelance.operation.web" --json | jq -r '.[0].id')
# APP_ID 为空时报错退出：❌ 无法获取 appId，请检查 ONES 应用配置

# 创建分支，捕获分支名
CREATE_OUTPUT=$(ones branch create -i $ONES_ITEM_ID -a $APP_ID -t $ONES_BRANCH_TYPE -b $ONES_BASE_BRANCH -y 2>&1)
BRANCH_NAME=$(echo "$CREATE_OUTPUT" | grep "生成的分支名:" | awk '{print $NF}')
# BRANCH_NAME 为空时报错退出：❌ 无法从 ones branch create 输出中提取分支名
```

**无论复用还是新建，最后统一 checkout：**
```bash
git fetch origin
git checkout $BRANCH_NAME
git branch --set-upstream-to=origin/$BRANCH_NAME $BRANCH_NAME 2>/dev/null || true
```

输出：
```
🌿 [0/4] 创建 ONES 分支...
   ✅ 已创建分支：<branch-name>
（或）
   ⏭️  已存在分支：<branch-name>，跳过创建
```

---

### Step 1：拉取 PRD + API 文档

仅在以下情况执行：
- `--only=fetch` 显式指定
- 第一个参数以 `https://km.` 开头（触发全流程）

**1.1 拉取 PRD**

从 URL 中提取 contentId（km.sankuai.com/collabpage/<id> → <id>），执行：

```bash
oa-skills citadel getSimpleMarkdown --contentId <id>
```

提取返回内容中的标题和正文。

从标题推导 `name`：翻译为英文、取核心业务词、snake_case（如「品牌管理」→ `brand`）。

写入 `$PROJECT_ROOT/.worker/<name>/prd.md`，文件已存在时覆盖写入。

**写入格式**：第一行保留学城原始链接：
```
<!-- km-url: https://km.sankuai.com/... -->
```

**1.2 拉取 API 文档**

API 文档来源优先级（依次尝试）：
1. 用户显式传入的 `<api-url>`
2. 自动查找 PRD 子文档列表：

   ```bash
   oa-skills citadel getChildContent --contentId <prd-id>
   ```

   将返回结果保存为 `CHILD_DOCS`（供 Step 1.4.1 复用，避免重复查询）。找标题含「API」或「接口」的子文档，取其 contentId，再执行：

   ```bash
   oa-skills citadel getSimpleMarkdown --contentId <api-id>
   ```

3. 均未找到 → 跳过，输出 `⏭️  未找到 API 文档，代码生成将仅依据 PRD`

找到后将内容写入 `$PROJECT_ROOT/.worker/<name>/api.md`。

**1.3 写入后标题规范化（两个文件都执行）**

每个 `.md` 文件写入后，对正文标题做规范化处理（第一行 `<!-- km-url: ... -->` 注释保持不变）：

1. **层级规范化**：扫描全文所有标题行（`#` 开头），找出最小级别作为 h1 基准，其余标题按相对层级依次映射，确保不跳级（h1→h2→h3 顺序连续）
2. **重复标题去重**：同一文件中若有相同文本的标题，从第二个起依次追加后缀 `（二）`、`（三）`...

规范化后覆盖写入原文件，无需输出额外日志。

**1.4 场景判断**

解析 `prd.md` 文档标题，提取版本号，正则：`{功能名}V(\d+)-`（如「精选帖子提报V02-新增沉底池」→ 版本号 `02`）：

- **版本号为 01，或标题不含版本号** → **新增场景**，将 `SCENE=new` 保存供 Step 2 使用
- **版本号 ≥ 02** → **变更场景**，将 `SCENE=change` 保存供 Step 2 使用

**1.4.1 拉取或生成变更清单**（仅 `SCENE=change` 时执行）

变更清单追加写入 `prd.md` 末尾，供 Step 2 精确控制前端改动范围。

**两条路：**

**路 A：从学城子文档拉取**（worker-java 已跑过时）

在 PRD 子文档列表中查找标题匹配 `变更清单 V{N-1}→V{N}` 的子文档。使用 Step 1.2 已缓存的 `CHILD_DOCS`（若 Step 1.2 走了自动查找路径）；否则重新查询：

```bash
oa-skills citadel getChildContent --contentId <prd-id>
```

找到 → 执行 `getSimpleMarkdown` 拉取内容，追加写入 `prd.md` 末尾

**路 B：AI 对比生成**（未找到学城子文档时）

1. 在 PRD 父目录下查找上一版本文档（版本号为 `N-1`）：

   ```bash
   oa-skills citadel getDocumentMetaInfo --contentId <prd-id>
   # 取 parentId，再查子文档列表找 V{N-1} 文档
   oa-skills citadel getChildContent --contentId <parentId>
   ```

   找不到 → 跳过变更清单，输出 `⏭️  未找到上一版本文档，Step 2 将依据 api.md 标记和完整 PRD 确定变更范围`

2. 找到 → 拉取旧版 PRD 内容，与新版对比，生成前端视角的变更清单，追加写入 `prd.md` 末尾：

   ```markdown
   ## 变更清单（V{N-1}→V{N}）

   > 本节由 worker-h5 自动生成，供后续步骤精准识别本次变更范围，不属于产品规格正文。

   ### 新增
   - **[页面/字段/交互]**：[一句话说明]

   ### 修改
   - **[页面/字段/交互]**：[原内容] → [新内容]
   ```

**1.5 冲突检查**（仅在 api.md 成功写入时执行，未找到 api.md 则跳过）

| SCENE | api.md | 冲突原因 |
|-------|--------|---------|
| new（新增） | 存在，含 🆕/✏️ 变更标记的接口 | PRD 是 V01 新增场景，但 API 文档标记了变更接口 |
| change（变更） | 存在，不含任何 🆕/✏️ 标记 | PRD 是 V02+ 变更场景，但 API 文档无变更标记，无法确定修改范围 |

命中任一冲突 → 报错退出：
```
❌ PRD 与 API 文档场景不一致，无法继续执行。
   PRD：<新增场景（V01）/ 变更场景（V0N）>
   API：<含变更标记 / 无变更标记>
   请确认两份文档已同步更新后重新运行。
```

输出：
```
📖 [1/4] 拉取文档...
   ✅ PRD 已写入 .worker/<name>/prd.md（文档标题：<title>，name：<name>）
   ✅ API 文档已写入 .worker/<name>/api.md（来源：<url>）
（或）
   ⏭️  未找到 API 文档，代码生成将仅依据 PRD
   🔄 变更场景（V0N）：
      ✅ 变更清单已追加至 prd.md（来源：学城子文档）
   （或）
      ✅ 变更清单已追加至 prd.md（AI 对比 V{N-1}→V{N}）
   （或）
      ⏭️  未找到上一版本文档，Step 2 将依据 api.md 标记和完整 PRD 确定变更范围
（或）
   🆕 新增场景（V01）：后续步骤全量生成
```

---

### Step 2：前端代码生成

读取 `.worker/<name>/prd.md` 和 `.worker/<name>/api.md`（如存在），根据 Step 1 确定的 `SCENE` 变量分流处理。

**生成前必读：**
- 读取 `CLAUDE.md` 中的样式规范、组件用法规范
- 读取 `src/pages/ActivityManage/List.vue` 作为列表页标准结构参考
- 读取 `src/pages/_template/FormModal.vue` 作为弹窗表单标准结构参考
- 使用任何 mtd- 组件前必须先读取 `docs/mtd/` 下对应文档

---

#### 新增场景（`SCENE=new`）

全量生成所有文件，按顺序执行：

**① API 层** `src/apis/<name>.ts`
- 根据 `api.md`（如存在）中的接口定义生成请求函数；无 api.md 时依据 PRD 推断
- 使用项目现有的请求封装方式（读取其他 apis 文件确认风格）
- 每个接口对应一个导出函数，函数名与接口路径对应

**② Store 层** `src/store/modules/<name>.ts`
- 使用 Vuex 4，包含 state、mutations、actions、getters
- state 包含列表数据、分页信息、加载状态
- actions 封装 API 调用，处理请求/响应

**③ 列表页** `src/pages/<PascalName>/List.vue`
- 严格遵循 `ActivityManage/List.vue` 的标准结构
- 顶部操作区 `.action-bar`（新建按钮等）
- 筛选区 `.filter` + `mtd-form`，筛选框统一加 `.filter-input`
- 查询/重置按钮区 `.filter-action`（flex-end 对齐）
- 表格区 `<div class="mt-md">` + `mtd-table.w-full`
- 分页 `.pagination`
- 禁止内联样式，使用 class
- 颜色间距使用变量，不写魔法数字

**④ 新建/编辑弹窗** `src/pages/<PascalName>/FormModal.vue`
- 严格遵循 `_template/FormModal.vue` 的标准结构
- `v-model="visible"` 控制显隐，`:mask-closable="false"`
- `mtd-form` 绑定 `ref`、`:model`、`:rules`、`:label-width`
- 每个 `mtd-form-item` 必须设置 `prop`
- `#footer` 插槽：左取消（`type="panel"`）、右确定（`type="primary" :loading`）
- `handleClosed`（`@closed`）用于 `resetFields()`
- 对外暴露 `open(row?)` 方法

**⑤ 路由注册**（如项目有路由配置文件）
- 读取现有路由配置文件确认注册方式
- 在对应位置追加新模块路由

**⑥ Store 注册**（如项目有 store index 文件）
- 读取现有 store/index.ts 确认注册方式
- 追加新模块注册

**⑦ 菜单注册**（如项目有菜单配置文件）
- 搜索项目中包含已有页面路由路径的菜单配置文件（如 menu.ts / menus.ts / nav.ts 等）
- 读取该文件，理解菜单树结构和权限路径列表的格式
- 在菜单树中追加新模块的菜单节点（folder + 子 node）
- 在权限路径列表（如 authMenuUrls 或等价字段）中追加新路由路径
- 菜单注册与路由注册同等重要：路由注册让页面「可访问」，菜单注册让页面「可导航到」，缺一不可

---

#### 变更场景（`SCENE=change`）

> **⚠️ 硬约束：禁止整文件重写，所有已有文件必须先 Read 再 Edit。**
>
> **⚠️ 硬约束：路由、Store 模块注册、菜单节点不得重复注册。**
>
> **⚠️ 硬约束：改动范围以 `prd.md` 末尾的变更清单（如有）为准；结合 api.md 中的 🆕/✏️ 标记。禁止因"顺手"改动变更范围之外的代码。**

根据 api.md 是否存在及是否含变更标记，分两条路：

**变更场景 A：api.md 不存在**（接口协议未变，纯前端改动）

- `src/apis/<name>.ts`：不修改
- `src/store/modules/<name>.ts`：Read → Edit，按 prd.md 变更清单修改对应 action/state
- `src/pages/<PascalName>/List.vue`：Read → Edit，按 prd.md 变更清单修改列/筛选项/交互逻辑
- `src/pages/<PascalName>/FormModal.vue`：Read → Edit，按 prd.md 变更清单修改表单字段

**变更场景 B：api.md 存在且含 🆕/✏️**

- `src/apis/<name>.ts`：Read → Edit；🆕 追加新接口请求函数，✏️ 只更新对应函数的入参/出参类型，不重写整文件
- `src/store/modules/<name>.ts`：Read → Edit；🆕 追加新 action/mutation，✏️ 修改对应 action
- `src/pages/<PascalName>/List.vue`：Read → Edit，按 prd.md 变更清单和 api.md 标记修改
- `src/pages/<PascalName>/FormModal.vue`：Read → Edit，按 prd.md 变更清单和 api.md 标记修改

---

每生成一个文件打印：`   📄 生成：<相对路径>`
每修改一个文件打印：`   ✏️  修改：<相对路径>`

输出：
```
⚙️  [2/4] 前端代码生成...
   📄 生成：src/apis/<name>.ts
   📄 生成：src/store/modules/<name>.ts
   📄 生成：src/pages/<PascalName>/List.vue
   📄 生成：src/pages/<PascalName>/FormModal.vue
   ✏️  修改：src/router/index.ts
   ✏️  修改：src/store/index.ts
   ✏️  修改：<菜单配置文件路径>
   ✅ 生成完成（生成 N 个文件，修改 M 个文件）
```

---

### Step 3：Git 提交

```bash
git add -A
git commit -m "feat(<name>): <从 PRD 标题提取的简短描述>"
git push -u origin HEAD
```

输出：
```
📦 [3/4] Git 提交...
   ✅ 已提交并推送：feat(<name>): <描述>
```

---

### Step 4：Talos 部署 + 等待完成

**4.1 触发部署**

**泳道名确认（懒加载）：** 在执行任何部署命令之前，检查 `TALOS_SWIMLANE` 是否已有值：
- 已有值（来自 `--swimlane` 参数）→ 直接使用，输出 `   🏊 泳道：<swimlane>`
- 为空 → 交互式提示用户输入：
  ```
  🏊 请输入泳道名：
  ```
  用户输入后赋值给 `TALOS_SWIMLANE`，不得为空，为空则重复提示。

```bash
# 重新 source check-env.sh 以确保 USE_NPX_TALOS / EXPECTED_TALOS_CLI_VERSION 在本 bash 调用中可用
# （bash 状态不跨 tool call 持久化，准备阶段的 source 结果不会保留到此处）
source /Users/liangyanghe/.claude/skills/talos-project-publish/check-env.sh > /dev/null 2>&1

CURRENT_BRANCH=$(git branch --show-current)

if [ "$USE_NPX_TALOS" = true ]; then
  DEPLOY_OUTPUT=$(npx -y --registry=http://r.npm.sankuai.com @ee/talos-cli@${EXPECTED_TALOS_CLI_VERSION} \
    flow publish \
    -a $TALOS_APP_ID \
    -t $TALOS_TEMPLATE_ID \
    --target $TALOS_TARGET \
    -b $CURRENT_BRANCH \
    --swimlane $TALOS_SWIMLANE \
    -n "feat(<name>): <描述> | Powered by worker-h5" \
    -P $TALOS_PLATFORM 2>&1)
else
  DEPLOY_OUTPUT=$(talos \
    flow publish \
    -a $TALOS_APP_ID \
    -t $TALOS_TEMPLATE_ID \
    --target $TALOS_TARGET \
    -b $CURRENT_BRANCH \
    --swimlane $TALOS_SWIMLANE \
    -n "feat(<name>): <描述> | Powered by worker-h5" \
    -P $TALOS_PLATFORM 2>&1)
fi

# 从输出中提取 flowId
FLOW_ID=$(echo "$DEPLOY_OUTPUT" | grep -oE 'Flow ID: [0-9]+' | grep -oE '[0-9]+')
[ -z "$FLOW_ID" ] && echo "❌ 无法从部署输出中提取 flowId，请检查部署是否成功" && exit 1
```

输出：
```
🚀 [4/4] Talos 部署...
   ✅ 部署已触发（Flow ID：<flowId>，分支：<branch>，泳道：<swimlane>）
```

**4.2 轮询等待**

**⚠️ 严禁使用 while 循环或任何长时间阻塞的单次 bash 命令实现轮询。必须每次轮询单独调用一次 bash。**

```bash
if [ "$USE_NPX_TALOS" = true ]; then
  STATUS_OUTPUT=$(npx -y --registry=http://r.npm.sankuai.com @ee/talos-cli@${EXPECTED_TALOS_CLI_VERSION} \
    flow describe $FLOW_ID --app-id $TALOS_APP_ID -P $TALOS_PLATFORM 2>&1)
else
  STATUS_OUTPUT=$(talos flow describe $FLOW_ID --app-id $TALOS_APP_ID -P $TALOS_PLATFORM 2>&1)
fi
```

判断逻辑：
- 输出包含 `success` → 部署成功，进入完成阶段，**不再 sleep**
- 输出包含 `failed` 或 `error` → 报错退出：`❌ 部署失败，请检查 Talos 日志。可用 --only=deploy 重试`
- 其他 → `sleep $TALOS_POLL_INTERVAL`，继续轮询
- 累计超过 `$TALOS_TIMEOUT_MINUTES` 分钟 → 超时退出：`⚠️  等待超时，请手动检查部署状态。查看链接：<url>`

输出（每次轮询）：
```
   🔄 [00:15] 当前状态：running，继续等待...
   ✅ [01:30] 部署成功！
```

**4.3 部署链接**

- Talos 2.0：`https://talos-better.sankuai.com/publish-log?appId=<appId>&flowId=<flowId>`
- Talos 1.0：`https://talos.sankuai.com/#/project/<appId>/log/<flowId>`

---

### 完成

```bash
WORKER_END_TIME=$(date +%s)
WORKER_ELAPSED=$((WORKER_END_TIME - WORKER_START_TIME))
WORKER_MINUTES=$((WORKER_ELAPSED / 60))
WORKER_SECONDS=$((WORKER_ELAPSED % 60))
```

打印执行摘要（仅展示本次实际执行的步骤）：

```
🎉 [worker-h5] 执行完成！（<name>）
   ⏱️  总耗时：${WORKER_MINUTES}m ${WORKER_SECONDS}s

📋 执行摘要：

| 步骤 | 产物 |
|------|------|
| Step 0  branch | 分支：<branch-name> |
| Step 1  fetch  | .worker/<name>/prd.md + api.md |
| Step 2  code   | 生成 N 个文件，修改 M 个文件 |
| Step 3  git    | feat(<name>): <描述> |
| Step 4  deploy | Flow ID：<flowId>，部署成功 |

   Talos: <部署链接>
```

---

## 错误处理原则

- 每步失败后打印清晰错误信息和 `--only=<step>` 重试提示
- Step 1 API 文档未找到不退出，降级为仅依据 PRD 生成代码
- Step 4 部署失败退出，给出重试命令 `--only=deploy`
- 除上述情况外，其他步骤失败均退出并给出重试命令

## 步骤流转铁律（最高优先级）

**无论任何步骤内部经历了多少轮错误修复和重试，只要该步骤最终成功，必须立即继续执行后续步骤，不得停止。**

- 某步骤内部自动修复错误（如 lint、markdownlint、编译错误等），修复后重试成功 → 继续下一步
- 修复过程可能经历多轮，每次重试成功后**第一件事是检查当前执行模式还有哪些后续步骤未执行**
- 只有步骤**最终失败**（无法自动修复、超出重试次数）才退出并给出重试命令

---

## ARGUMENTS

用户调用格式：
```
/worker-h5 <prd-url> <ones-url>                        # 全流程（api-url 从 prd 子文档自动发现）
/worker-h5 <prd-url> <api-url> <ones-url>              # 全流程，显式指定 api-url
/worker-h5 <prd-url> <ones-url> --swimlane=<name>      # 全流程，显式指定泳道名
/worker-h5 <name> --only=<step>                        # 只执行指定步骤
/worker-h5 --list-steps                                # 查看所有步骤
```

解析规则：
- `--list-steps` → 输出步骤列表后立即退出
- 第一个非 `--` 参数以 `https://km.` 开头 → 作为 prd-url；后续非 `--` 参数中，以 `https://km.` 或 `https://` 开头且不含 `ones.` 的为 api-url，含 `ones.` 的为 ones-url
- 第一个非 `--` 参数以 `https://ones.` 开头 → 作为 ones-url（仅用于 `--only=branch` 场景）
- 第一个非 `--` 参数不以 `https://` 开头 → 作为 name（仅用于 `--only=<step>` 场景，需 `.worker/<name>/prd.md` 已存在）
- `--from` 或 `--to` 出现时报错：`❌ 不支持 --from/--to，请使用 --only 进行单步执行`
- 执行到分支创建步骤但 ones-url 缺失时，报错：`❌ 缺少 ONES 工作项链接，创建分支需要提供 ones-url`
- `--swimlane=<value>` → 将值赋给 `TALOS_SWIMLANE`；未提供时为空，Step 4 执行前再交互式提示

`--list-steps` 输出：
```
📋 [worker-h5] 步骤列表

| 步骤 | Code     | 说明                          |
|------|----------|-------------------------------|
| 0    | branch   | 创建 ONES 分支                |
| 1    | fetch    | 从学城拉取 PRD + API 文档     |
| 2    | code     | 前端代码生成                  |
| 3    | git      | Git 提交                      |
| 4    | deploy   | Talos 部署 + 等待完成         |

参数说明：
  - 第一个非 -- 参数以 https://km. 开头 → prd-url
  - 第一个非 -- 参数以 https://ones. 开头 → ones-url（仅用于 --only=branch）
  - 第一个非 -- 参数为普通字符串 → name（仅用于 --only=<step>）

用法示例：
  /worker-h5 https://ones.sankuai.com/... --only=branch
  /worker-h5 https://km.sankuai.com/... https://ones.sankuai.com/...
  /worker-h5 https://km.sankuai.com/... https://ones.sankuai.com/... --swimlane=my-lane
  /worker-h5 https://km.sankuai.com/... --only=fetch
  /worker-h5 brand --only=code
  /worker-h5 brand --only=git
  /worker-h5 brand --only=deploy
  /worker-h5 brand --only=deploy --swimlane=my-lane
```

ARGUMENTS: <此处由调用者传入实际参数>
