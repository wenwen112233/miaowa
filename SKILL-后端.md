# worker-java — 后端数字员工

从 PRD 到测试环境上线的一键全流程：ONES 分支创建 → DB 模型 → API 设计 → 后端代码生成 → Git 提交 → Cargo 部署 → Shepherd 接口创建 → DDL 建表 → 生成测试用例 → 等待部署 → 接口测试。

> **⚠️ 硬约束：所有步骤必须严格串行执行，禁止并行。**
>
> **⚠️ 硬约束：变更场景下，只对 prd.md「变更清单」章节列出的接口、代码、表字段、牧羊人路由进行新增或修改；变更范围以外的已有内容，一律不得修改、重新生成或覆盖。**


## 配置

启动时读取 `$PWD/.worker/java.json`，文件不存在则报错退出：`❌ 找不到 .worker/java.json，请先创建配置文件`

```json
{
  "appkey": "com.sankuai.xxx",
  "backend": {
    "buildCommand": "mvn clean install -DskipTests",
    "buildMaxRetries": 3
  },
  "ones": {
    "branchType": "feature",
    "baseBranch": "master"
  },
  "cargo": {
    "pollIntervalSeconds": 30,
    "timeoutMinutes": 10
  },
  "shepherd": {
    "host": "https://shepherd.mws-test.sankuai.com",
    "groupId": 151374
  },
  "test": {
    "writeOpSleepSeconds": 2,
    "logcenterWaitSeconds": 5,
    "logcenterMaxRetries": 12,
    "logcenterRetryIntervalSeconds": 5
  },
  "db": {
    "name": "xxx"
  }
}
```

---

## 执行流程

### 准备：读取配置和参数

```bash
PROJECT_ROOT="$PWD"
WORKER_START_TIME=$(date +%s)

CONFIG=$(cat $PROJECT_ROOT/.worker/java.json)
APPKEY=$(echo $CONFIG | jq -r '.appkey')
BUILD_COMMAND=$(echo $CONFIG | jq -r '.backend.buildCommand')
BUILD_MAX_RETRIES=$(echo $CONFIG | jq -r '.backend.buildMaxRetries')
ONES_BRANCH_TYPE=$(echo $CONFIG | jq -r '.ones.branchType')
ONES_BASE_BRANCH=$(echo $CONFIG | jq -r '.ones.baseBranch')
CARGO_POLL_INTERVAL=$(echo $CONFIG | jq -r '.cargo.pollIntervalSeconds')
CARGO_TIMEOUT_MINUTES=$(echo $CONFIG | jq -r '.cargo.timeoutMinutes')
SHEPHERD_HOST=$(echo $CONFIG | jq -r '.shepherd.host')
SHEPHERD_GROUP_ID=$(echo $CONFIG | jq -r '.shepherd.groupId')
TEST_WRITE_OP_SLEEP=$(echo $CONFIG | jq -r '.test.writeOpSleepSeconds // 0')
TEST_LC_WAIT=$(echo $CONFIG | jq -r '.test.logcenterWaitSeconds')
TEST_LC_MAX_RETRIES=$(echo $CONFIG | jq -r '.test.logcenterMaxRetries')
TEST_LC_RETRY_INTERVAL=$(echo $CONFIG | jq -r '.test.logcenterRetryIntervalSeconds')
DB_NAME=$(echo $CONFIG | jq -r '.db.name // ""')
MOCK_GATEWAYS=""
```

解析 ARGUMENTS，确定 `NAME`、`ONES_URL`，以及执行模式（全流程 / --only）。

**运行时变量兜底规则（适用于 Step 10、11）：**
`--only=<step>` 直接进入某步骤时，`CARGO_UUID` 或 `SWIMLANE` 可能为空。遇到为空时统一执行：
```bash
SEARCH_OUTPUT=$(cargo-cli stack search 2>&1)
SWIMLANE=$(echo "$SEARCH_OUTPUT" | grep "泳道名:" | awk '{print $NF}')
CARGO_UUID=$(echo "$SEARCH_OUTPUT" | grep "编排 ID:" | awk '{print $NF}')
```
- search 成功 → 从结果提取对应变量继续
- search 失败 → 报错退出：`❌ 无法获取泳道信息，请确认泳道已创建`

输出：
```
🤖 [worker-java] 开始执行：<name>
```

---

### Step 0：创建 ONES 分支

需要 `<ones-url>` 参数。未传入时报错：`❌ 缺少 ONES 工作项链接，创建分支需要提供 ones-url`

```bash
# 1.1 解析 ONES URL，提取 ONES_SPACE_ID 和 ONES_ITEM_ID
ones url-parse -u "<ones-url>"

# 1.2 查询该工作项下是否已有当前 appkey 的分支
EXISTING_BRANCH=$(ones branch list -p $ONES_SPACE_ID -i $ONES_ITEM_ID --json | \
  jq -r --arg appkey "$APPKEY" \
  '[.items[] | select(.appkey==$appkey)] | first | .name // empty')
```

**分两条路：**

**已有分支（`EXISTING_BRANCH` 非空）→ 直接复用，记录分支名：**
```bash
BRANCH_NAME=$EXISTING_BRANCH
```

**无分支（`EXISTING_BRANCH` 为空）→ 新建分支：**
```bash
# 1.3 获取应用 appId
APP_ID=$(ones apps -p $ONES_SPACE_ID -n $APPKEY --json | jq -r '.data[0].id')
# APP_ID 为空时报错退出：❌ 无法获取 appId，请检查 ONES 应用配置

# 1.4 创建分支，捕获分支名
BRANCH_OUTPUT=$(ones branch create -i $ONES_ITEM_ID -a $APP_ID -t $ONES_BRANCH_TYPE -b $ONES_BASE_BRANCH -y 2>&1)
BRANCH_NAME=$(echo "$BRANCH_OUTPUT" | grep "生成的分支名:" | sed 's/.*生成的分支名:[[:space:]]*//')
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
🌿 [0/11] 创建 ONES 分支...
   ✅ 已创建分支：<branch-name>
（或）
   ⏭️  已存在分支：<branch-name>，跳过创建
```

---

### Step 1：PRD 拉取

使用 `citadel` skill 的 `getSimpleMarkdown` 命令读取学城文档，写入 `$PROJECT_ROOT/.worker/<name>/prd.md`。

从标题推导 `name`（翻译为英文、取核心业务词、snake_case，如「品牌管理」→ `brand`）。文件已存在时覆盖写入。

**写入格式**：在文件第一行保留学城原始链接，格式为：
```
<!-- km-url: https://km.sankuai.com/... -->
```
之后接正文内容。

**场景判断**：解析文档标题，提取版本号，正则：`{功能名}V(\d+)-`（如「精选帖子提报V02-新增沉底池」→ 版本号 `02`）：

- **版本号为 01** → 场景 = **新增**，无变更清单，直接输出后进入 Step 2
- **版本号 ≥ 02** → 场景 = **变更**，执行以下步骤生成变更清单：

  1. **找上一版本文档**：
     - 用 `citadel` skill 的 `getDocumentMetaInfo` 获取当前文档的父目录 ID（`parentId`）
     - 用 `getChildContent` 列出父目录下所有子文档标题
     - 用正则 `{功能名}V(\d+)-` 解析各文档版本号，找版本号为 `N-1` 的文档
     - **找不到** → 中断报错：`❌ 在父目录下未找到 {功能名}V{N-1}-* 文档，请确认上一版本已上传至同一目录后重新运行`
     - **找到** → 用 `getSimpleMarkdown` 读取旧版 PRD 内容

  2. **AI 对比生成变更清单**：逐章对比新旧两版 PRD，识别本次改动，生成结构化变更清单。

     **⚠️ 硬约束：变更清单只描述产品层面的变化，禁止出现任何技术实现细节：**
     - **禁止**出现接口名称（如「精选池列表查询接口」「加入沉底接口」）
     - **禁止**出现英文字段名（如 `postType`、`interventionCoefficient`、`inSinkPool`）
     - 每条描述必须能让运营人员/产品经理直接看懂，不需要研发背景
     - 判断标准：该变更能否在 PRD 的 UI 原型或交互说明中找到对应描述？能找到才写，找不到不写

     格式：
     ```markdown
     ## 变更清单（V{N-1}→V{N}）

     > 本节由 worker-java 自动生成，供后续步骤精准识别本次变更范围，不属于产品规格正文。

     ### 新增
     - **[功能模块/页面元素]**：[一句话说明新增内容，使用产品语言]
     - …

     ### 修改
     - **[功能模块/页面元素]**：[原内容] → [新内容，使用产品语言]
     - …
     ```

  3. **写入 prd.md**：将变更清单追加写入 prd.md 末尾

  4. **上传变更清单到学城**：用 `citadel` skill 在**当前 PRD 文档下**创建子文档，标题「变更清单 V{N-1}→V{N}」，内容为变更清单 Markdown；输出子文档链接

  > **注意**：此处调用 citadel 属于 worker-java 自动化流程，citadel 操作完成后直接继续执行 Step 2，跳过 citadel 的授权收尾询问。

输出（新增场景）：
```
📖 [1/11] 拉取 PRD...
   ✅ 已写入 .worker/<name>/prd.md（文档标题：<title>，name：<name>）
   🆕 新增场景（V01）：后续步骤照整份 PRD 实现
```
输出（变更场景）：
```
📖 [1/11] 拉取 PRD...
   ✅ 已写入 .worker/<name>/prd.md（文档标题：<title>，name：<name>）
   🔄 变更场景（V{N-1}→V{N}）：已生成变更清单，后续步骤仅处理变更范围
   📋 变更清单已上传学城：<子文档链接>
```

---

### Step 2：DB 模型生成

根据 PRD 及当前表/代码状态，生成对应 SQL，写入 `.worker/<name>/schema.sql`。

**禁止生成 ALTER TABLE，永远只生成 CREATE TABLE。**

> **⚠️ 硬约束：禁止通过任何方式查询 RDS / 数据库来获取表结构（包括 `mtdev rds`、`SHOW CREATE TABLE`、`DESC` 等）。表结构信息一律从代码链路读取（mapper XML + PO 类），不得绕过。**

#### 2.1 获取 ONES_ITEM_ID

全流程时直接复用 Step 0 已解析的 `ONES_ITEM_ID`。

`--only=schema-gen` 时从当前分支名解析：

```bash
CURRENT_BRANCH=$(git branch --show-current)
ONES_ITEM_ID=$(echo "$CURRENT_BRANCH" | grep -oE '[0-9]{8,}')
```

取不到时（`ONES_ITEM_ID` 为空），使用当前日期作为后缀：

```bash
[ -z "$ONES_ITEM_ID" ] && ONES_ITEM_ID=$(date +%Y%m%d)
```

#### 2.2 实体建模

通读 `prd.md` **完整正文**（不只看变更清单），列出所有业务实体及其核心字段。此阶段不区分新旧功能，目的是建立完整的实体视图。

**⚠️ 实体建模只做语义判断，不做存储优化：** 判断字段是否属于某实体，看的是「该概念在业务上是否存在」，而非「该值是否变化」。某字段对某类实体值恒定（如固定为 0），这是约束，不是字段不存在的理由，建模时不得因此剔除。字段是否持久化、是否用默认值，是建完实体模型后的下一步决策，不能反向影响建模。

对所有实体两两比较，判断是否应合并为单表，按业务字段覆盖度判断（先剔除基础字段：id、create_time、update_time、operator 等）：
- 剔除后业务字段 > 3 个：共享业务字段占比 > 70% → 合并为单表，用 `type` 列区分，差异字段保留，对不适用记录写入默认值
- 剔除后业务字段 ≤ 3 个：比例不可信，改为逐字段对比语义是否一致，一致则合并
- 不满足以上条件 → 各自建表

**必须显式输出实体建模结论：**

```
📊 [2.2] 实体建模：
   - <实体A>：<核心字段列表>
   - <实体B>：<核心字段列表>
   → <合并/拆分结论及理由>
   → 最终：<N> 张表
```

#### 2.3 定位现有实现

读取 `$PROJECT_ROOT/operation-client/src/main/java/com/sankuai/freelance/operation/client/service/proxy/` 下所有 proxy 文件，按以下三个维度**逐一**评估**每一个** proxy 候选类：

  1. **实体匹配**：PRD 操作的核心实体（如帖子、职人、订单）必须与 proxy 方法操作的实体一致；**必须读取 req/resp/PO 类名中的核心名词**，不得仅凭方法名或业务功能名称的表面相似性判断
  2. **动作匹配**：PRD 中已有功能对应的方法在 proxy 里已存在；PRD 中新增功能对应的方法在 proxy 里不存在
  3. **覆盖度**：PRD 已有功能在 proxy 里的覆盖率接近 100%

  三个维度同时满足才算匹配。

  **⚠️ 硬约束：禁止在输出评估结论前跳过任何维度的判断。**

  **必须在得出任何匹配结论之前，显式输出以下评估表（对每一个候选 proxy 类逐行填写）：**

  | proxy 类名 | 维度 1：实体（req/resp/PO 类名核心名词） | 维度 2：已有动作覆盖 / 新增动作缺失 | 维度 3：覆盖率 | 结论 |
  |------------|--------------------------------------|--------------------------------------|----------------|------|
  | XxxProxyTService | `Xxx`（实体名） | 已有: listXxx/addXxx/... / 缺失: updateYyy✅ | 5/5 = 100% | ✅ 匹配 |
  | YyyProxyTService | `Yyy`（与 PRD 实体不符） | — | — | ❌ 实体不匹配，跳过 |

  **维度 1 判断规则（最高优先级）：**
  - 从该 proxy 的方法入参/出参 class 名称中提取核心名词（如 `BatchAddFeaturedTechnicianReq` → 核心名词为 `Technician`/职人）
  - 与 PRD 操作的核心实体对比（如 PRD 操作对象为帖子 `Post`）
  - 两者不一致 → 该 proxy **立即判定为 ❌ 实体不匹配**，无需继续评估维度 2/3
  - **禁止因"业务功能名称相似"（如同为"精选"操作）而绕过实体名词不一致的判定**

**匹配结果处理：**

- 匹配到 0 个 → 打印错误信息，直接退出：
  ```
  ❌ 未找到匹配的 proxy 类，请检查 PRD 描述是否清晰或手动指定入口
  ```
- 匹配到 2+ 个 → 打印错误信息，直接退出：
  ```
  ❌ 匹配到多个 proxy 类：XxxProxyTService、YyyProxyTService，请手动确认后重新运行
  ```
- 匹配到 1 个 → 沿链路提取所有关联表名，建立「实体 → 现有表」的映射（无表记为空）：
  ```
  proxy → server 层实现 → application 层 service → infrastructure 层 mapper → mapper XML → 提取表名
  ```

**必须显式输出现有实现映射：**

```
🔍 [2.3] 现有实现：
   - <实体A> → <现有表名>
   - <实体B> → 无现有实现
```

#### 2.4 操作决策

对照 2.2 实体建模结论与 2.3 现有实现映射，再结合变更清单（确认本次需要落表的范围），为每个实体确定操作类型：

| 操作类型 | 触发条件 |
|---------|---------|
| **新建表** | 实体无现有表，且建模结论为独立建表 |
| **修改现有表** | 实体有现有表，且存在 DDL 级变更（新增/删除字段、修改字段类型/长度、新增/删除索引）；仅注释、DEFAULT 值、取值范围/校验规则等应用层逻辑变更不算 |
| **合并入现有表** | 新实体建模结论为合并，目标现有表已存在 |
| **跳过** | 变更清单未涉及该实体；或变更仅涉及注释、DEFAULT 值、取值范围/校验规则等应用层逻辑，无 DDL 级变更 |

操作类型有歧义时，中断并输出提示信息退出：
```
❌ 实体 <X> 操作类型存在歧义：<具体说明>，请手动确认后重新运行
```

**必须显式输出每个实体的操作决策：**

```
🎯 [2.4] 操作决策：
   - <实体A>（<现有表名>）→ 修改现有表（新增字段：xxx、yyy）
   - <实体B>             → 合并入现有表 <现有表名>（新增 pool_type 列）
   - <实体C>             → 跳过（变更清单未涉及）
```

#### 2.5 生成 SQL

按 2.4 的操作决策生成 SQL。

**表名规则：**
- **新增场景**（Step 1 判定为 V01）：表名为 `operation_{name}`，如 `operation_post_pool`
- **变更场景**（Step 1 判定为 V02+）：表名为 `operation_{name}_{ones_item_id}`，如 `operation_post_pool_95021373`

**变更场景字段设计规则：**

从 2.3 链路中已读取的 mapper XML（`resultMap` / `Base_Column_List`）和对应 PO 类获取当前完整字段列表；以此为基础，以 PRD 为准重新设计字段：
- PRD 有明确命名的字段 → 用 PRD 的字段名（不复用老表字段名）
- PRD 未提及的老表字段 → 保留老表原字段名
- PRD 新增的字段 → 追加
- PRD 删除的字段 → 去掉

**输出文件格式：**

```sql
-- ============================================================
-- worker-java 自动生成 | <name> | <日期>
-- ============================================================

CREATE TABLE `xxx` (
  ...
  created_time DATETIME NOT NULL COMMENT '创建时间',
  ...
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='...';
```

**注意事项：**
- 必须包含 `created_time DATETIME NOT NULL` 创建时间字段
- 枚举值用 TINYINT 存储，注释中说明各值含义
- 如果 PRD 中没有明确表结构需求，分析 PRD 推断所需字段

输出：
```
🗄️  [2/11] 生成 DB 模型...
   📊 [2.2] 实体建模：
      ...（实体列表及合并/拆分结论）

   🔍 [2.3] 现有实现：
      ...（实体 → 现有表映射）

   🎯 [2.4] 操作决策：
      ...（每个实体的操作类型及依据）

   ✅ [2.5] 已写入 .worker/<name>/schema.sql
```

---

### Step 3：API 设计生成

根据 PRD 生成 HTTP API 设计文档，写入 `.worker/<name>/api-design.md`。

**场景分流：**

- **新增场景** → 照整份 PRD 设计所有接口，直接进入「API 设计文档规范」
- **变更场景** → 执行以下前置步骤，再进入「API 设计文档规范」：

  **变更场景前置步骤：**

  1. **建现有接口字典**：从 `.worker/java.json` 读取 `shepherd.host` 和 `shepherd.groupId`，调用：
     ```bash
     mtcurl '{shepherd.host}/spapi/v1/apis/{shepherd.groupId}'
     ```
     从返回的 `data` 数组提取每条路由的 `name`（API名称）、`path`、`invokerViews[0].methodName`。**禁止通过关键词搜索或猜测端点的方式查询 Shepherd。**

     对每条路由按三类分类：
     - **有效路由**：路由指向的方法在当前 proxy 类中存在，且该方法只对应这一条路由 → 还原入参/出参字段（读取 req/resp 类，递归读取完整继承链字段，忽略 `SSOUser`），加入**有效路由字典**（API名称 + path + 入参字段 + 出参字段）；打印时只显示入参，出参仅供合并分析内部使用
     - **重复路由**：路由指向的方法在当前 proxy 类中存在，但该方法已有其他路由，这条多余 → 加入**重复路由列表**（API名称 + path + 主路由API名称），单独展示；path 视为已占用
     - **无效路由**：路由指向的方法在任何 proxy 类中都不存在（孤儿路由） → 加入**无效路由列表**（API名称 + path）；path 视为已占用，新接口设计时需避开
     - 路由指向的方法仅在其他 proxy 类中存在（其他领域路由）→ 忽略，不进任何列表

     **⚠️ 硬约束：从 req Java 类读取到的字段名必须原文照搬，禁止根据 REST 命名惯例（如将 `list` 改为 `items`）或任何其他理由重命名。**

     **原因：Thrift 序列化后的 HTTP 响应 JSON 字段名与 Java 类字段名严格一致，前端按 api-design.md 中的字段名取值。任何改名都会导致前端取不到数据，造成页面映射失败。api-design.md 的字段名必须与代码完全吻合，是前后端之间不可破坏的契约。**

     **必须显式输出字典内容（禁止使用 Markdown 表格，必须按以下列表格式输出）：**
     ```
     📚 [3.0] 建现有接口字典...
        有效路由（N 条）：
        - to_m_post_recommend_add     /api/.../add      入参：[postId]
        - to_m_post_recommend_list    /api/.../list     入参：[categoryId]
        ...

        重复路由（K 条）：
        - post_recommend_add          /api/.../add      # 重复路由：to_m_post_recommend_add
        ...

        无效路由（M 条）：
        - post_recommend_update  /api/.../update   # updateRecommend 方法在 proxy 中不存在
     ```

  2. **确定处理范围**：读取 `prd.md` 「变更清单」章节，识别本次涉及的接口：
     - **新增接口**（变更清单「新增」列表）→ 按 PRD 设计入参/出参（草稿）；URL 按有效路由字典中已有路径的前缀模式推导（如已有路径为 `/api/freelance/manage/operation/post-recommend/list`，新接口则为 `/api/freelance/manage/operation/post-recommend/<action>`）；**若推导出的路径与重复路由或无效路由列表中的路径冲突，追加具体业务后缀绕开**（如 `/update` 冲突 → 改用 `/update_intervention_coefficient`）
     - **修改接口**（变更清单「修改」列表）→ 直接从有效路由字典取字段，应用变更；URL 同样从字典取
     - **变更清单未提及的接口** → **跳过，不处理**

  3. 只将本次新增/修改的接口写入 `api-design.md`，未变更的已有接口不出现在输出文件中；每个接口标题后标记变更类型：
     - 新增接口：`## N. <接口名称> 🆕`
     - 修改接口：`## N. <接口名称> ✏️`

**接口合并分析（仅变更场景执行，写文档前必须完成）：**

> **⚠️ 硬约束：禁止在完成合并分析并输出结论前开始写接口文档。**
>
> **背景**：草稿接口由 AI 基于 PRD 生成，逻辑上应合并的操作早就会合成一个接口；真正的盲区是 AI 看不到 Shepherd 已有什么。合并分析的职责只有一件事：**判断每个草稿接口是否应并入某条现有 Shepherd 路由**。

**执行步骤：**

1. **列出草稿接口**：将本次所有新增草稿接口逐一列出（名称 + 入参字段列表 + 出参核心字段列表）

2. **草稿 × 有效路由字典，逐一比较**（重复路由和无效路由不参与）：
   - 先剔除公共字段（pageNum、pageSize、ssoUser、createTime、updateTime、operator 等通用字段）
   - 计算剩余字段的覆盖度：共有字段数 ÷ 字段并集总数
   - 入参覆盖度 > 70% **且** 出参覆盖度 > 70% → **并入现有路由**：将该草稿标记为 ✏️，在现有路由入参中加 `type` 参数区分两种用途
   - 不满足 → 草稿作为独立新接口，标记 🆕

3. **必须显式输出合并结论：**

```
🔀 [3.x] 接口合并分析：
   草稿接口：
   - <草稿A>：入参[x1, x2]，出参[y1, y2]
   - <草稿B>：入参[x1, x2]，出参[y1, y2]
   ...
   比对结论：
   - <草稿A> × <现有路由X>：入参覆盖度 100%，出参覆盖度 100% → 并入现有路由（✏️ 加 type 参数，1=X 2=Y）
   - <草稿B>：无匹配现有路由 → 独立新增（🆕）
   最终接口清单：共 <N> 个接口（✏️ <M> 个并入现有路由，🆕 <K> 个新增）
```

**并入现有路由规范：**
- 保留现有路由的接口名和 HTTP path 不变
- 在入参中加 `type`（或 `poolType` 等语义更明确的命名），取值用整数并在注释中说明含义，**必填，无默认值**
- 合并后的接口入参 = 两者入参的并集，对不适用某 type 的字段在业务规则章节说明

---

**API 设计文档规范：**

文件结构：

```markdown
# <领域名称> - HTTP API 设计

## API 概述

<简要描述本文档包含的接口范围>

---

## 1. <接口名称>

**接口**: `POST <完整path>`

**描述**: <接口功能说明>

**入参**:
\`\`\`lier
{
    <字段名>: <类型>
}
\`\`\`

**出参**:
\`\`\`lier
{
    code: int
    msg: str
    data: {}
}
\`\`\`

**业务规则**:
- <规则1>（可选章节）

---
```

字段规范：

| 字段 | 要求 |
|------|------|
| 接口名称（title） | 简洁的中文名称，**是接口的唯一标识**，修改后视为新接口 |
| METHOD | 统一使用 `POST` |
| 完整 path | 必须以 `/` 开头；具体前缀参考 CLAUDE.md 或已有接口路径风格 |
| 描述 | 非空，说明接口功能 |
| 入参 | 用 lier 格式描述；无入参时写 `无` |
| 出参 | 必须存在，用 lier 格式描述，不能为空 |

lier 格式说明：

```lier
{
    # 注释用 # 开头
    fieldName: str        # 字符串
    fieldName: int        # 整数
    fieldName: bool       # 布尔
    fieldName: float      # 浮点数
    fieldName: str[]      # 数组
    fieldName: {          # 嵌套对象
        subField: str
    }
    fieldName: {          # 对象数组
        subField: str
    }[]
}
```

注意事项：
- 接口名称（title）是同步到 papi 的唯一标识，**重命名等于新建**，原接口记录变为孤儿
- 序号（`## 1.`）仅用于文档阅读，不影响同步逻辑
- `**业务规则**:` 为可选章节，不影响同步

**API 字段设计规则（必须执行）：**
- 生成前先读取 `.worker/<name>/schema.sql`（如存在）
- 按以下规则设计字段：
  1. **DB 有、API 透传使用**：命名必须与 DB 字段一致（snake_case → camelCase），不得另立命名
  2. **DB 有、API 不需要暴露**：直接不返回
  3. **DB 没有、API 需要**：根据业务语义自行命名
  4. **DB 字段经过加工后返回**：属于加工字段，根据业务语义命名（如 `statusName`、`timeRange`）
  5. **创建接口必须在 `data` 中返回新建资源的主键 ID**（如 `brandId`、`activityId`），不得返回空 `data: {}`；即使前端当前不需要，后续 E2E 测试用例的步骤链路（如创建→删除）也依赖此 ID
  6. **入参标识符选择（写操作）**：接口入参用于定位一条记录时，先看同模块已有写操作（如删除、修改）用的是什么标识符，**新接口与之保持一致**。若同模块无可参考的已有写操作，再按业务语义选择：优先用外部唯一业务标识符（如 `postId`、`technicianId`）而非数据库主键 `id`
  7. **枚举/状态字段必须同时返回 code 和 name**：出参中所有枚举或有限状态字段，必须同时设计 code 字段（整数或字符串，供前端做逻辑判断）和对应的 `xxxName` 字段（中文文案，供前端直接渲染），禁止只返回 code 让前端自行维护映射表。示例：`postType: int` + `postTypeName: str`、`displayStatus: str` + `displayStatusName: str`。name 字段的解析方式按以下规则选择：
     - **本地持久化 / 业务逻辑分支**（如 poolType、activityStatus）：必须在 application 层定义枚举类（`XxxEnum`，含 code + name），Gateway 层将外部 code 转为本地枚举，BO 字段用枚举类型，Converter 调用 `enum.getName()`
     - **外部透传、仅用于展示、不持久化**（如来自外部服务的 postType、contentType）：在 Converter 中用 switch 解析 name 即可，无需建枚举类

**下钻场景接口设计规则（必须执行）：**

对于数据报表、数据看板、数据大盘等页面，某些汇总指标支持点击后查看明细信息（下钻）。这类场景下：
1. **页面加载时**：只查询页面上所需数据的信息，不把明细数据一起查出来
2. **点击时才查**：用户点击某个指标后，才发起请求加载该指标对应的明细数据
3. **只查被点击的那一个**：只加载用户当前点击的这个指标对应的明细，不把所有指标的明细都查出来

即：明细查询必须是独立的接口，按需懒加载，而不是在列表/看板接口里一次性把所有明细都塞进去返回。

**生成完毕后，同步上传到学城：**

从 `.worker/<name>/prd.md` 第一行读取 `<!-- km-url: ... -->` 中的学城链接。若链接存在：
1. 使用 `citadel` skill 查询该文档的子文档列表
2. 若标题为 `API 设计` 的子文档已存在 → 覆盖更新其内容
3. 若不存在 → 在该文档下新建子文档，标题为 `API 设计`，内容为 `api-design.md` 全文

**注意：此处调用 citadel 属于 worker-java 自动化流程，citadel 操作完成后直接继续执行 Step 4，跳过 citadel 的授权收尾询问。**

若 `prd.md` 中无学城链接，跳过上传，输出 `⏭️  未找到学城链接，跳过上传`。

上传成功后，将子文档链接赋值给变量：
```bash
API_KM_URL=<citadel 返回的子文档链接>
```
上传跳过时 `API_KM_URL` 留空。

输出：
```
📝 [3/11] 生成 API 设计...
   ✅ 已写入 .worker/<name>/api-design.md
   ✅ 已同步到学城（新建）：<子文档链接>
（或）
   ✅ 已同步到学城（更新）：<子文档链接>
（或）
   ⏭️  未找到学城链接，跳过上传
```

---

### Step 4：后端代码生成

#### 4.1 预编译

```bash
cd "$PROJECT_ROOT" && $BUILD_COMMAND
```

- 编译成功 → 继续
- 编译失败 → 退出，提示：`❌ 预编译失败，请先修复现有编译错误再重试`

输出：
```
⚙️  [4/11] 后端代码生成...
   🔨 预编译中：<buildCommand>
   ✅ 预编译通过（或 ❌ 预编译失败，退出）
```

#### 4.2 生成各层 Java 文件

**场景分流：**

- **新增场景** → 按下方各层清单全量生成新文件
- **变更场景** → 读取 `.worker/<name>/api-design.md`，按 🆕 / ✏️ 标记确定受影响文件范围：
  - 🆕 新增接口：各层已有类文件中追加新方法；req / resp 等确实不存在的类，按新增场景规则新建
  - ✏️ 修改接口：只改受变更标注影响的字段和方法
  - **每次操作已有文件前必须先 Read，再 Edit 追加/修改，禁止整文件重写**
  - 下方各层清单作为文件类型参考，用于定位需要修改的具体文件，而非全量新建
  - 数据来源 4 步判断照常执行（适用于本次变更引入的任何新数据依赖）

严格遵守 CLAUDE.md 的 4 层架构：

**infrastructure 层：**
- `persistence/po/<Entity>PO.java` — 数据库持久化对象，字段与 DB 表一一对应
- `persistence/mapper/<Entity>Mapper.java` — MyBatis Mapper 接口
- `resources/mapper2m/<Entity>Mapper.xml` — MyBatis XML，含基础 CRUD SQL

  **⚠️ 硬约束（变更场景）：Mapper XML 表名必须替换为 schema.sql 中的新表名**
  - 读取 `.worker/<name>/schema.sql`，提取 `CREATE TABLE` 后的表名（如 `post_recommend_95041023`）
  - 将 Mapper XML 中所有涉及该变更表的旧表名（如 `post_recommend`）全部替换为新表名
  - 原因：测试环境建的是新表，Mapper 必须指向新表才能跑通接口测试；测试通过后由研发手动 ALTER 老表并将表名改回

- `gateway/<Entity>GatewayImpl.java` — Gateway 接口实现，只做数据访问和映射，code→enum 转换在此；外部服务返回的有限枚举字段（如 postType、contentType）同样在此转为本地枚举，不得将裸 code 字符串透传至 BO

  **生成 GatewayImpl 前，按以下顺序判断数据来源（匹配即停止）：**

  1. PRD 中该数据由本服务拥有管理（有增删改操作）且代码库无对应 Mapper
     → 生成 PO + Mapper + GatewayImpl（MyBatis 实现）

  2. PRD 中该数据由本服务拥有管理且代码库已有对应 Mapper
     → 复用已有 Gateway，不重复生成

  3. PRD 中该数据为外部依赖（引用自其他服务）且在 CLAUDE.md 数据字典中
     → 按字典指定接口生成正常 RPC 调用实现

  4. PRD 中该数据为外部依赖且不在数据字典中
     → 生成 `MockXxxGatewayImpl`，基于 PRD 对该数据的描述硬编码假数据；类和每个方法均加 `// TODO: 待下游就绪后替换`
     → 同时追加记录：`MOCK_GATEWAYS="$MOCK_GATEWAYS\n   - <相对路径>（<该 Mock 代表的数据说明>）"`

**application 层：**
- `model/enums/<Xxx>Enum.java` — 业务枚举，适用于**本地持久化或业务逻辑分支**的枚举字段（含 code 和 name）；**外部透传、仅用于展示、不持久化**的枚举字段（如来自外部服务的 postType、contentType）无需建枚举类，在 Converter 中 switch 解析 name 即可
- `model/param/<Xxx>Param.java` — 服务入参，只含业务所需字段
- `model/bo/<Xxx>BO.java` — 业务输出对象，枚举字段用枚举类型
- `gateway/<Entity>Gateway.java` — Gateway 接口定义
- `service/<Xxx>Service.java` — 业务逻辑接口
- `service/impl/<Xxx>ServiceImpl.java` — 业务逻辑实现，所有业务规则在此

**client 层：**
- `req/<Xxx>Req.java` — 请求 DTO，使用 `@ThriftStruct`，字段用 `@ThriftField` 标注序号；**`SSOUser` 不得放入 req**
- `resp/<Xxx>Resp.java` — 响应 DTO，同上
- `service/proxy/<Xxx>ProxyTService.java` — Thrift 接口定义；运营侧 `SSOUser` 必须作为方法独立第二参数

**server 层：**
- `validator/<Xxx>RequestValidator.java` — 入参格式校验，只校验格式和必填
- `convert/<Xxx>Converter.java` — 对象转换，使用 `@MdpBeanCopy`；BO→Resp 时枚举字段必须同时写入 code（`enum.getCode()`）和 name（`enum.getName()`）两个字段，缺一不可
- `service/proxy/<Xxx>ProxyTServiceImpl.java` — Thrift 接口实现，只做：校验→转换→调 Service→转换出参；**禁止使用 `@RequirePermission`**

**通用约束：**
- **Thrift 返回值**：无业务返回值的接口（如删除），必须使用 `BasicResponse<EmptyResp>`，返回 `BasicResponse.success(new EmptyResp())`；不得使用 `BasicResponse<Void>`（Swift Thrift codec 不支持 Void 泛型，启动时会抛 `MetadataErrorException`）
- **PageCodeEnum**：新增枚举值时，`permissionEnabled` 必须设为 `false`（关闭鉴权，以便自动化测试不受权限拦截）
- **`@RequireAdmin`**：新增或变更的接口**严禁使用**，该注解已废弃，不得在任何新代码中出现；一旦添加将导致 Step 11（test-run）完全失效
- **浮点数比较**：`Double`/`Float` 禁止用 `==` 或 `!=` 与字面量比较（Sonar 静态扫描会拦截，且浮点运算结果存在精度误差，直接 `==` 不可靠）。判零用 `Double.compare(val, 0.0) == 0`；判近似相等用 `Math.abs(a - b) < epsilon`

每生成一个文件打印：`   📄 生成：<相对路径>`
修改已有文件打印：`   ✏️  修改：<相对路径>`

#### 4.3 验证编译（最多重试 $BUILD_MAX_RETRIES 次）

```bash
cd "$PROJECT_ROOT" && $BUILD_COMMAND
```

- 编译成功 → 继续
- 编译失败 → 分析错误，自动修复相关文件，重新编译
- 超过最大重试次数 → 退出，提示：`❌ 编译失败超过 <buildMaxRetries> 次，请手动排查`

输出：
```
   🔨 验证编译（第 N 次）：<buildCommand>
   ✅ 编译通过（或 🔧 发现错误，自动修复中...）
```

---

### Step 5：Git 提交

```bash
git add -A
git commit -m "feat(<name>): <从 PRD 标题提取的简短描述>"
git push -u origin HEAD
```

输出：
```
📦 [5/11] Git 提交...
   ✅ 已提交并推送：feat(<name>): <描述>
```

---

### Step 6：Cargo 部署

触发异步部署，**不等待**，立即进入下一步：

```bash
# 确保 Cargo 已登录（沙箱环境优先使用 MOA 无感换票）
cargo-cli sso status 2>/dev/null | grep -q "已登录" || cargo-cli sso login --moa

CURRENT_BRANCH=$(git branch --show-current)

# 检测泳道是否已存在，同时得到泳道名
SEARCH_OUTPUT=$(cargo-cli stack search 2>&1)
echo "🔍 [debug] stack search 原始输出："
echo "$SEARCH_OUTPUT"

SWIMLANE=$(echo "$SEARCH_OUTPUT" | grep "泳道名:" | awk '{print $NF}')
echo "🔍 [debug] 提取到 SWIMLANE：[$SWIMLANE]"

# 泳道不存在时创建
if echo "$SEARCH_OUTPUT" | grep -q "泳道不存在"; then
  CREATE_OUTPUT=$(cargo-cli stack create --swimlane $SWIMLANE 2>&1)
  echo "🔍 [debug] stack create 原始输出："
  echo "$CREATE_OUTPUT"
fi

# 从 create 或 search 结果中获取 UUID
CARGO_UUID=$(echo "$CREATE_OUTPUT$SEARCH_OUTPUT" | grep "编排 ID:" | awk '{print $NF}')
echo "🔍 [debug] 提取到 CARGO_UUID：[$CARGO_UUID]"

# 触发部署
cargo-cli stack deploy \
  --uuid $CARGO_UUID \
  --services '[{"appkey":"'$APPKEY'","release":"'$APPKEY'","branch":"'$CURRENT_BRANCH'"}]'
```

输出：
```
🚢 [6/11] Cargo 部署...
   ✅ 部署已触发（分支：<branch>）

## 🚀 泳道：<SWIMLANE>
```

---

### Step 7：DDL 建表

**前置检查：**
- `DB_NAME` 为空（未配置 `db.name`）→ 跳过，输出 `⏭️  跳过建表（未配置 db.name）`
- `.worker/<name>/schema.sql` 不存在 → 跳过，输出 `⏭️  跳过建表（无 schema.sql）`

**执行步骤（使用 Bash 直接调用 mtdev，不走 sub-agent）：**

```bash
# 1. 通过 db.name 查询数据库 ID
DB_ID=$(mtdev --format json rds search-db --keyword $DB_NAME --env test 2>&1 | python3 -c "
import json, sys
data = json.load(sys.stdin)
for db in data:
    if db.get('Database Name') == '$DB_NAME':
        print(db['ID'])
        break
")

[ -z "$DB_ID" ] && echo "❌ 未找到数据库 $DB_NAME，请检查 db.name 配置" && exit 1

# 2. 从 schema.sql 提取表名，预检是否已存在
TABLE_NAME=$(grep -iE "^CREATE TABLE" $PROJECT_ROOT/.worker/<name>/schema.sql | head -1 | \
  sed 's/CREATE TABLE[[:space:]]*`\?\([^`( ]*\).*/\1/I')

EXISTING=$(mtdev --format json rds list-tables $DB_NAME $TABLE_NAME --env test 2>&1 | \
  python3 -c "import json,sys; d=json.load(sys.stdin); print('exists' if d else 'not_exists')" 2>/dev/null)

# 3. 表已存在则跳过；不存在则执行 DDL
if [ "$EXISTING" = "exists" ]; then
    echo "   ⏭️  表已存在，跳过（$TABLE_NAME）"
else
    SQL=$(cat $PROJECT_ROOT/.worker/<name>/schema.sql)
    mtdev rds write-ddl $DB_ID "$SQL" --comment "worker-java: $NAME DDL建表" --env test --yes
fi
```

输出：
```
🗃️  [7/11] DDL 建表...
   ✅ 已创建表 <table_name>（数据库：<db_name>）
（或）
   ⏭️  表已存在，跳过
```

---

### Step 8：Shepherd 接口创建

**场景分流：**

读取 `.worker/<name>/api-design.md`，按标题标记判断：

- **变更场景**（api-design.md 中存在 🆕 / ✏️ 标记）：
  - 只处理 **🆕 新增接口**，需要在 Shepherd 创建路由并发布
  - **✏️ 修改接口**：Shepherd 路由是 HTTP path → Thrift 方法的透明代理，不感知入参/出参字段，**直接跳过，无需任何操作**
  - 若 api-design.md 中没有 🆕 接口（全为 ✏️）→ 整步跳过，输出 `⏭️  无新增接口，跳过 Shepherd 创建`
- **新增场景**（api-design.md 中无标记）：处理所有接口

确定待处理接口列表后，从 api-design.md 中提取对应的 `**接口**: POST <path>` 行，与 Thrift 方法名对应，构造显式路径映射表。

**⚠️ 硬约束：Shepherd 路由路径必须从 api-design.md 中原文提取，禁止根据 CLAUDE.md 前缀规则或任何其他规则修改。**

**原因：api-design.md 是前后端共同遵守的契约文档，前端按其中的路径发送请求，路径必须完全一致否则前端请求失败。api-design.md 已综合考虑了新老模块的路径规范——新模块按 CLAUDE.md 前缀规范设计，老模块为保持兼容性沿用既有路径风格。Step 8 直接使用 api-design.md 的路径即可，不得二次修改。**

路径映射的对应规则：按接口语义匹配，例如"查询品牌列表"对应 `listBrands`、"添加品牌"对应 `createBrand`，以此类推。

调用 `shepherd-api-create-assistant` skill 完成本步骤，传入以下上下文：
- 目标环境：`$SHEPHERD_HOST`
- Shepherd 分组 ID：`$SHEPHERD_GROUP_ID`
- Thrift 接口文件：`operation-client/src/main/java/**/service/proxy/<Name>ProxyTService.java`
- **URL 路径映射（必须严格按此表，不得推断）**：待处理接口的 `Thrift方法名 → HTTP path` 对照表
- **自动化模式：无需任何用户确认，创建完成后直接发布所有新建接口**

输出：
```
🐑 [8/11] Shepherd 接口创建...
   ✅ to_m_xxx_yyy 已创建并发布
   ⏭️  to_m_xxx_zzz 已存在，发布（幂等）
（或变更场景无新增接口时）
   ⏭️  无新增接口，跳过 Shepherd 创建
```

---

### Step 9：生成测试用例

**仅首次执行有实际效果**；若 `.worker/<name>/test-cases.md` 已存在则跳过，除非使用 `--only=test-gen` 强制重新生成。


根据 `.worker/<name>/prd.md` 和 `.worker/<name>/api-design.md` 生成 E2E 测试用例，写入 `.worker/<name>/test-cases.md`。

**场景分流：**

- **变更场景**（api-design.md 中存在 🆕 / ✏️ 标记）：只为 api-design.md 中列出的接口生成测试用例；prd.md 仅作为背景参考（理解业务语义），不驱动用例范围，不基于 prd.md 中未变更的功能生成新用例
- **新增场景**（api-design.md 中无标记）：按 prd.md + api-design.md 全量生成

生成前扫描 infrastructure 层下是否存在 `Mock*GatewayImpl.java` 文件：
- **有** → 读取这些文件，理解各方法返回的字段和具体值；API 响应中涉及这些字段的断言使用 Mock 文件中的具体值，而非从 PRD 推断
- **无** → 跳过，按原有逻辑生成

#### 9.1 读取测试数据集

读取 `.worker/test-data.json`（全局共享），提取：
- 测试环境 Base URL：`baseUrl`，写入 test-cases.md 头部
- 各业务域的测试数据（ID + 预期字段值）

若文件不存在，跳过数据引用，头部测试环境写「待填写」。

#### 9.2 E2E 用例生成策略

**只生成 E2E 测试用例，不生成单接口用例。**

E2E 用例必须是**多步骤的用户完整操作路径**，每一步依赖上一步的结果。以下内容不属于 E2E，**禁止放入 test-cases.md**：
- 单接口的入参校验（如空值、超长、格式错误）
- 单接口的边界值测试
- 无上下文依赖的独立接口调用

**E2E 场景识别规则：**
- 从 PRD 交互说明中识别，每个独立的操作章节对应一条路径
- 需要前置数据时用 `[Setup]` 标注，末步用 `[Cleanup]` 标注清理

**表格格式：**

```markdown
## E2E-1：<场景名称>

| 编号 | 接口 | 场景描述 | 请求体 | 预期结果 |
|------|------|---------|--------|---------|
| E2E-1-Step1 | POST /api/... | ... | {"name":"..._{timestamp}"} | code=0，记录 data.id 为 {Step1.id} |
| E2E-1-Step2 | DELETE /api/... | [Cleanup] 删除测试数据 | {"id":{Step1.id}} | code=0 |
```

**变量引用硬约束（违反则测试用例无法执行）：**

| 规则 | 正确 | 错误 |
|------|------|------|
| 引用同场景前序步骤的变量，只用短形式 | `{Step1.id}` | `{E2E-1-Step1.id}` ❌ |
| 不能跨 E2E 场景引用变量 | E2E-2 只能引用 E2E-2 内的步骤 | `{E2E-1-Step1.id}` ❌ |

**变量记录硬约束（违反则变量无法被 runner 提取，后续步骤替换失败）：**

在预期结果列记录变量时，**必须**使用以下固定短语格式：

```
记录 data.<字段名> 为 {StepN.<字段名>}
```

示例：
- ✅ `code=0，记录 data.brandId 为 {Step1.brandId}`
- ✅ `code=0，记录 data.id 为 {Step1.id}`
- ❌ `code=0，data.brandId 不为空，记录为 {Step1.brandId}`（语序错误，runner 无法解析）
- ❌ `code=0，brandId 不为空，记录为 {Step1.brandId}`（缺少 `data.` 前缀）

**头部统计规则：必须在所有表格写完后，实际数一遍步骤总数，再填入文档头部。禁止预估。**

生成写入后，立即执行格式校验：

```bash
STEP_COUNT=$(python3 ~/.claude/skills/worker-java/test-runner.py \
  $PROJECT_ROOT/.worker/$NAME/test-cases.md \
  $PROJECT_ROOT/.worker/test-data.json \
  --dry-run 2>&1 | grep -oE '^共识别到 [0-9]+' | grep -oE '[0-9]+')

if [ -z "$STEP_COUNT" ] || [ "$STEP_COUNT" = "0" ]; then
  echo "❌ test-cases.md 格式校验失败：识别到 0 个步骤，格式不符合 E2E 表格规范，请重新生成"
  exit 1
fi
```

格式校验通过后，同步上传到学城：

从 `.worker/<name>/prd.md` 第一行读取 `<!-- km-url: ... -->` 中的学城链接。若链接存在：
1. 使用 `citadel` skill 查询该文档的子文档列表
2. 若标题为 `测试用例` 的子文档已存在 → 覆盖更新其内容
3. 若不存在 → 在该文档下新建子文档，标题为 `测试用例`，内容为 `test-cases.md` 全文

**注意：此处调用 citadel 属于 worker-java 自动化流程，citadel 操作完成后直接继续执行 Step 10，跳过 citadel 的授权收尾询问。**

若 `prd.md` 中无学城链接，跳过上传，输出 `⏭️  未找到学城链接，跳过上传`。

上传成功后，将子文档链接赋值给变量：
```bash
TEST_CASES_KM_URL=<citadel 返回的子文档链接>
```
上传跳过时 `TEST_CASES_KM_URL` 留空。

输出：
```
🧪 [9/11] 生成测试用例...
   ✅ 已写入 .worker/<name>/test-cases.md（共 N 个 E2E 场景，M 个步骤）
   ✅ 格式校验通过：共 M 个步骤
   ✅ 已同步到学城（新建）：<子文档链接>
（或）
   ✅ 已同步到学城（更新）：<子文档链接>
（或）
   ⏭️  未找到学城链接，跳过上传
（或校验失败时）
   ❌ test-cases.md 格式校验失败：识别到 0 个步骤，格式不符合 E2E 表格规范，请重新生成
```

---

### Step 10：等待部署完成

**⚠️ 严禁使用 while 循环或任何长时间阻塞的单次 bash 命令实现轮询。必须每次轮询单独调用一次 bash。**

`CARGO_UUID` 为空时按准备节的兜底规则获取。

```bash
cargo-cli stack status --uuid $CARGO_UUID
```

判断逻辑：
- 输出包含 `部署状态: ✅ 部署完成` → 成功，进入 Step 11，**不得再 sleep**
- 输出包含 `部署失败` → 报错退出：`❌ 部署失败，请检查 Cargo 日志`
- 其他 → `sleep $CARGO_POLL_INTERVAL`，继续轮询
- 累计超过 `$CARGO_TIMEOUT_MINUTES` 分钟 → 超时退出：`⚠️ 等待超时，请手动检查部署状态`

输出：
```
⏳ [10/11] 等待部署完成...
   🔄 [00:30] 当前状态：deploying，继续等待...
   ✅ [01:30] 部署成功！
```

---

### Step 11：执行接口测试

⚠️ 本步骤结束前，test-report.md 必须包含以下四块，缺一不可：
   □ ## 📊 总览（含总步骤数、通过、失败、跳过、通过率）
   □ ## ✅ 通过用例（表格）
   □ ## ❌ 失败/跳过用例（无失败时省略）
   □ ## ✅ 结论与建议

读取 `.worker/<name>/test-cases.md` 和 `.worker/test-data.json`，执行所有步骤，完成后写入 `.worker/<name>/test-report.md`。

#### 11.1 准备

`SWIMLANE` 为空时按准备节的兜底规则获取。

#### 11.2 执行

**第一次**（执行测试，实时透传输出）：
```bash
python3 ~/.claude/skills/worker-java/test-runner.py \
  $PROJECT_ROOT/.worker/<name>/test-cases.md \
  $PROJECT_ROOT/.worker/test-data.json \
  --swimlane $SWIMLANE \
  --write-op-sleep $TEST_WRITE_OP_SLEEP \
  2>&1 | tee /tmp/worker-java-test-output.txt
```

**第二次**（读取最后一行）：
```bash
tail -1 /tmp/worker-java-test-output.txt
```

若最后一行以 `RUNNER_TIMEOUT:` 开头，进入 11.2.1 超时兜底流程，否则继续 11.3。

#### 11.2.1 超时兜底：查 LogCenter

解析标记格式：`RUNNER_TIMEOUT:{stepId}:{traceId}:{timestamp}`

等待 `$TEST_LC_WAIT` 秒后开始查日志，最多 retry `$TEST_LC_MAX_RETRIES` 次，每次间隔 `$TEST_LC_RETRY_INTERVAL` 秒：

```bash
echo "[retry N/$TEST_LC_MAX_RETRIES] 查询开始时间：$(date '+%Y-%m-%d %H:%M:%S')"
mtdev logcenter --format json query "$APPKEY" @/dev/stdin --env test << 'EOF'
{
  "query": {"bool": {"filter": [{"term": {"traceId__": "<traceId>"}}]}},
  "sort": [{"mt_datetime": {"order": "asc"}}],
  "size": 50
}
EOF
```

**根据查询结果分三条路：**

**① 查不到日志**（`$TEST_LC_MAX_RETRIES` 次后仍为空）：
- 输出：`❌ 超时且 LogCenter 无日志（已重试 <maxRetries> 次），请自行排查`，退出

**② 查到日志，code=0**（超时但业务成功）：
- 输出：`⚠️ {stepId} 接口超时但业务成功（LogCenter 确认），继续执行后续步骤`
- 从日志 `message` 字段中提取 `resp={...}` 里的 JSON，解析出需要传递给后续步骤的变量（如 `id`、`data.id`），构造 `inject_vars` JSON，格式：`{"E2E-1:Step1.id": 123}`
- 找到超时步骤的**下一个步骤 ID**（`next_step_id`），重新调用 test-runner.py 继续执行剩余步骤：
  ```bash
  python3 ~/.claude/skills/worker-java/test-runner.py \
    $PROJECT_ROOT/.worker/<name>/test-cases.md \
    $PROJECT_ROOT/.worker/test-data.json \
    --swimlane $SWIMLANE \
    --start-step <next_step_id> \
    --inject-vars '<inject_vars_json>' \
    --timestamp <timestamp> \
    --write-op-sleep $TEST_WRITE_OP_SLEEP \
    2>&1 | tee /tmp/worker-java-test-output.txt
  ```
- 读取新输出的最后一行，若仍以 `RUNNER_TIMEOUT:` 开头则再次进入 11.2.1；否则继续 11.3

**③ 查到日志，code≠0**（业务失败）：
- 输出：`❌ 业务失败（LogCenter 确认 code=<code>），请排查后重新部署`，退出

#### 11.3 生成测试报告

从 `RUNNER_RESULT` JSON 写入 `.worker/<name>/test-report.md`：

```markdown
# <模块名> 接口测试报告

> 执行时间：{yyyy-MM-dd HH:mm} · 环境：{baseUrl} · 执行人：Claude Code 自动生成

## 📊 总览

| 指标 | 数值 |
|------|------|
| 总步骤数 | {total}（共 {e2e_count} 个 E2E 场景） |
| ✅ 通过 | {pass} |
| ❌ 失败 | {fail} |
| ⏭ 跳过 | {skip} |
| 通过率 | {pass}/{total}（{percent}%） |

## ✅ 通过用例

| 步骤编号 | 接口路径 | 场景描述 | 实际请求体 | 预期结果 | 实际响应摘要 | TraceId | 数据来源 |
|---------|---------|---------|-----------|---------|------------|---------|---------|

## ❌ 失败 / 跳过用例

（无失败时省略此章节）

| 步骤编号 | 接口路径 | 场景描述 | 实际请求体 | 预期结果 | 实际响应摘要 | 状态 | TraceId | 根因 & 建议 |
|---------|---------|---------|-----------|---------|------------|------|---------|-----------|

## ✅ 结论与建议
```

测试报告写入后，同步上传到学城：

从 `.worker/<name>/prd.md` 第一行读取 `<!-- km-url: ... -->` 中的学城链接。若链接存在：
1. 使用 `citadel` skill 查询该文档的子文档列表
2. 若标题为 `测试报告` 的子文档已存在 → 覆盖更新其内容
3. 若不存在 → 在该文档下新建子文档，标题为 `测试报告`，内容为 `test-report.md` 全文

**注意：此处调用 citadel 属于 worker-java 自动化流程，citadel 操作完成后直接继续执行完成步骤，跳过 citadel 的授权收尾询问。**

若 `prd.md` 中无学城链接，跳过上传，输出 `⏭️  未找到学城链接，跳过上传`。

上传成功后，将子文档链接赋值给变量：
```bash
TEST_REPORT_KM_URL=<citadel 返回的子文档链接>
```
上传跳过时 `TEST_REPORT_KM_URL` 留空。

输出：
```
🧪 [11/11] 执行接口测试...
   （test-runner.py 的 stdout 直接透传）
   ✅ 已写入 .worker/<name>/test-report.md（通过 <pass>/<total>，失败 <fail>，跳过 <skip>）
   ✅ 已同步到学城（新建）：<子文档链接>
（或）
   ✅ 已同步到学城（更新）：<子文档链接>
（或）
   ⏭️  未找到学城链接，跳过上传
```

---

### 完成

```bash
WORKER_END_TIME=$(date +%s)
WORKER_ELAPSED=$((WORKER_END_TIME - WORKER_START_TIME))
WORKER_MINUTES=$((WORKER_ELAPSED / 60))
WORKER_SECONDS=$((WORKER_ELAPSED % 60))
```

每次执行结束，打印执行摘要，列出本次实际执行的步骤及其关键产物：

```
🎉 [worker-java] 执行完成！（<name>）
   ⏱️  总耗时：${WORKER_MINUTES}m ${WORKER_SECONDS}s

📋 执行摘要：按实际执行的步骤，列出编号、Code 和产物；跳过的步骤不出现。
```

若 `$MOCK_GATEWAYS` 非空，追加输出：

```
⚠️  Mock Gateway（待替换，下游就绪后需替换为真实实现）：
$MOCK_GATEWAYS
```

额外附上以下内容：

输出前先执行以下命令提取 PRD 链接：
```bash
PRD_KM_URL=$(head -1 $PROJECT_ROOT/.worker/$NAME/prd.md | sed 's/.*km-url:[[:space:]]*\(https:[^ ]*\).*/\1/')
```
`PRD_KM_URL` 为空时该行省略。

```
## 🚀 泳道：$SWIMLANE

| 链接 | 地址 |
|------|------|
| ONES | $ONES_URL |
| PRD  | $PRD_KM_URL |
| API  | $API_KM_URL |
| Test Cases | $TEST_CASES_KM_URL |
| Test Report | $TEST_REPORT_KM_URL |
| Shepherd | $SHEPHERD_HOST/api-group-detail?api_group_name=<groupName>&api_group_id=$SHEPHERD_GROUP_ID&group_tab=api-manage |
| Cargo | https://dev.sankuai.com/cargo/stack/detail/$CARGO_UUID/yaml |
```

`PRD_KM_URL`、`API_KM_URL`、`TEST_CASES_KM_URL`、`TEST_REPORT_KM_URL` 为空时对应行省略。

**强制约束：**
- `$SWIMLANE` 为空时，必须先执行兜底规则（`cargo-cli stack search`）获取后再输出，不得省略泳道行

---

## ARGUMENTS

用户调用格式：
```
/worker-java <prd-url> <ones-url>                  # 全流程
/worker-java <prd-url> --only=prd                  # 只拉取 PRD
/worker-java <name> --only=<step>                  # 只执行指定步骤（调试用）
/worker-java --list-steps                          # 查看所有步骤
```

解析规则：
- `--list-steps` → 输出步骤列表后立即退出
- 第一个非 `--` 参数以 `https://km.` 开头 → 作为 prd-url；第二个非 `--` 参数（如有）以 `https://ones.` 开头则为 ones-url
- 第一个非 `--` 参数以 `https://ones.` 开头 → 作为 ones-url（仅用于 `--only=branch` 场景）
- 第一个非 `--` 参数不以 `https://` 开头 → 作为 name（仅用于 `--only=<step>` 调试场景）
- `--from` 或 `--to` 出现时报错：`❌ 不支持 --from/--to，请使用 --only 进行单步调试`
- 执行到分支创建步骤但 ones-url 缺失时，报错：`❌ 缺少 ONES 工作项链接，创建分支需要提供 ones-url`

`--list-steps` 输出：按执行流程章节的步骤顺序，输出每步的 Step、Code 和说明，格式为表格，示例如下：

| Step | Code | 说明 |
|------|------|------|
| 0 | branch | 创建 ONES 分支 |
| 1 | prd | 拉取 PRD |
| 2 | schema-gen | 生成 DB 模型 |
| 3 | api-gen | 生成 API 设计 |
| 4 | code | 后端代码生成 |
| 5 | git | Git 提交 |
| 6 | deploy | Cargo 部署 |
| 7 | schema-run | DDL 建表 |
| 8 | api-run | Shepherd 接口创建 |
| 9 | test-gen | 生成测试用例 |
| 10 | wait | 等待部署完成 |
| 11 | test-run | 执行接口测试 |

用法示例：
  /worker-java https://ones.sankuai.com/... --only=branch               # 只创建 ONES 分支
  /worker-java https://km.sankuai.com/... https://ones.sankuai.com/...  # 全流程
  /worker-java https://km.sankuai.com/... --only=prd                    # 只拉取 PRD
  /worker-java brand --only=schema-gen                                  # 只重跑 DB 模型生成
  /worker-java brand --only=api-gen                                     # 只重跑 API 设计生成
  /worker-java brand --only=code                                        # 只重跑后端代码生成
  /worker-java brand --only=git                                         # 只重跑 Git 提交
  /worker-java brand --only=deploy                                      # 只重跑部署
  /worker-java brand --only=schema-run                                  # 只重跑 DDL 建表
  /worker-java brand --only=api-run                                     # 只重跑 Shepherd 接口创建
  /worker-java brand --only=test-gen                                    # 只重跑测试用例生成
  /worker-java brand --only=wait                                        # 只等待部署完成
  /worker-java brand --only=test-run                                    # 只重跑接口测试

ARGUMENTS: <此处由调用者传入实际参数>
