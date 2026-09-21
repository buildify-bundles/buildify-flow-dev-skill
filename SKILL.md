---
name: buildify-flow-dev
description: >-
  Authors Buildify workflows from installed Bundle nodes: discover capabilities,
  canvas JSON, validate, test-run, deploy. Do not add canvas comment/notes by
  default. Use when the user builds or publishes a flow, or mentions CommentNode,
  批注, notes 节点, commentMarkdown.
---

# Buildify Flow 编排

用已安装的 Bundle 节点编排工作流：发现能力、读节点表单与文档、引用已有凭证、生成 canvas JSON、
校验、保存草稿、同步试跑，试跑通过后**自动发布**到用户已确认的服务器，并汇总发布信息。

本 skill 与 `buildify-bundle-dev` 职责互斥：那个是「做节点」，这个是「用节点编流程」。
编排时若 `bundle list` / `bundle nodes` 里没有实现需求的软件包或节点，**停下来提示用户去 bundle-dev 实现并发布**，不要在本 skill 里编造节点或改写 Bundle 源码。

权威接口是 `buildify` CLI（`pip install buildify-cli`），它调 `buildify-open-api-server`。
不要直接猜 REST 路径，先 `buildify help`。

## 工作流

复制清单并逐项勾选：

```
流程编排进度：
- [ ] 1. 确认 CLI 已配置（buildify --json key test；缺 key 则提示用户签发并保存到 ~/.buildifyrc）
- [ ] 2. 列出项目并请用户确认存放位置（project list → 展示 projectName，禁止自作主张）
- [ ] 3. 列出该项目全部 worker（workerName + 描述 + 在线状态）并请用户确认服务器
- [ ] 4. 发现节点：`bundle list` → 只用到的包 `bundle nodes`。**当前环境没有对应软件包/节点时停止**，提示用户用 `buildify-bundle-dev` 实现并发布后再回来编流程；禁止臆造节点名。找到后再 **仅对将要放入画布的节点** 拉 `node-properties` / `node-relations`；`bundle doc` 只在参数含义不清时读
- [ ] 5. 按本流程实际用到的 bundle/类型列凭证并请用户确认（`project credentials -p -b bundle`）。没有则用 `create-credential`：先展示默认 **name/label**，列出表单参数请用户填写，**确认后再创建**；不要把密钥写进仓库或回显
- [ ] 6. 按 [reference/flow-json.md](reference/flow-json.md) 生成 canvas JSON（节点拷贝 `icon` / `label` / `isTrigger`；`summary` 写本流程职责且 **≤8 字**；**竖排**：主链同一 x、y 递增；边上的 `relations` 整份拷贝 `name`+`label`+`description`，MCP 自定义关系再带 `_id`/`icon`/`mcpKind`）。**表达式按 `uiComponent` 写**：JSON 用 `={{ }}`，SQL 用 `#{}` / `${}`，文本及其他用 `{{ }}`，见 [expressions.md](reference/expressions.md)。分支以不交叉为准：能干净汇合再共用；否则各支路继续竖排，**必要时复制配置相同的节点**。**默认不加批注**；仅当用户要求、或分支/调用方式/`summary` 写不下的关键约定不写就会误用时，才在主链左侧加 `type: comment`
- [ ] 7. buildify flow validate -p <proj> -f ./flow.json → 按 issues 修复，直到 valid=true
- [ ] 8. 新建流程或打开已有流程，save-draft
- [ ] 9. flow test-run，根据 nodes[].outputs / events 判断是否符合需求
- [ ] 10. 试跑成功后请用户确认**版本号**和**发布说明**，再 `flow deploy --wait --version-name … --remark …`（不要再传 -f / published），用返回的 `summary` 画表格
```

## 项目 / 服务器 / 凭证：必须用户点选

这三项关系到**数据落到哪、跑在哪台机器、用哪套密钥**。先 `project list` / `workers` / `credentials` 列出候选项，**用可读名称请用户选择并等回复**。未得到明确选择前，禁止 `flow create` / `save-draft` / `test-run` / `flow deploy`，也禁止往 JSON 里写 `data.credentials`。

不要默认第一项、唯一项、名称带「本地/测试」的项，或用户口语里含糊提到的环境。用户消息里已经点名了，仍要把完整名称回显请他确认一遍。

列出时带上 id，但**对用户说话用名称**：

| 选什么 | CLI | 展示字段 | 问法 |
|---|---|---|---|
| **项目**（流程存放位置） | `project list` | `projectName`、`remark`、`projectId` | 「流程将保存到哪个**项目**？例如「测试项目」」 |
| **服务器**（试跑/绑定 worker） | `project workers -p <proj>`（列出全部，不要只筛在线） | **对用户只显示** `workerName`、描述 `remark`、在线状态 `isOnline`。不要展示 hostname / workerId | 「在哪台**服务器**上试跑？例如「本地java进程测试」— 测试专用（在线）」 |
| **凭证**（节点密钥槽） | `project credentials -p <proj>`（可 `-b bundle` 收窄） | `label` 或 `name`、`type`、`bundleName`（**无密钥**） | 「节点要用哪条**访问凭证**？例如 MySQL「生产只读」」 |

问之前把候选项列成清单，例如：

```
请确认，避免流程存错项目或打到错误服务器/凭证：

1) 项目（流程将出现在该项目下）
   - 测试项目（p-krjfxeic）— 用于个人的测试使用
   - 测试项目2（p-j4xza62z）— 测试卷项目2

2) 服务器（选定项目后再列；格式：名称 — 描述 — 在线状态）
   - 本地java进程测试 — 测试专用 — 在线
   - 火山测试服务器 — fsdf — 在线
   - mac — （无描述） — 离线

3) 访问凭证（本流程需要的类型；没有则说明「本流程不需要凭证」）
   - buildify/mysql · Mysql · 生产只读（name=prod-ro）
```

规则：

- **项目**：create / save-draft 只用用户确认的那一个；回复时写清「已保存到项目「测试项目」」。
- **服务器**：对用户展示 `workerName` + `remark`（没有描述就写「无描述」）+ **在线/离线**。内部用 `workerId`，不要把 id/hostname 甩给用户。`flow create --worker`、`flow test-run --worker`、`flow deploy --worker` 用同一组用户指定的 worker。离线的可以列出来，但不能拿来试跑或发布；用户点了离线的要说明并请改选在线的。
- **凭证**：槽位（常见 `credentialsId`）的 `{type, name}` 必须是用户点的那条。同类型有多条时禁止猜。
- 需要凭证但列表为空 → 用 CLI 创建，**不要**让用户去控制台、不要跳过槽位硬跑：
  1. 必须已确认**项目**（`-p` 必填）。
  2. `bundle cred-types` / `cred-properties` 取类型与表单；或直接跑
     `buildify --json project create-credential -p … -b … --type …`（不传 name/label/data），
     用返回的 `data.userMessage` / `defaults` / `parameters`。
  3. **把默认 name、label 原样展示给用户**（可改），并列出每个参数的中文名（密钥类标明「密钥」）。
  4. 等用户确认名称并提供参数值。未确认禁止调用创建。
  5. 用 stdin 创建，不要 `--data` 写在命令行、不要把 JSON 落到仓库、不要在回复里回显密钥：
     `printf '%s' '{"host":"…"}' | buildify --json project create-credential -p "$PROJ" -b "$BUNDLE" --type "$TYPE" --name "$NAME" --label "$LABEL" --file -`
  6. 成功后只用返回的 `label` / `name` / `type` 告诉用户已创建。
- 本流程确实没有 `CredentialSelect` 时，明确说「不需要访问凭证」，不要随便绑一条。

**步骤 7 必须形成闭环**：`flow validate` 退出码 4 表示还有 ERROR。按 `issues[].path` 定位，改 JSON 后重跑，直到 `valid: true`。不要跳过校验直接试跑。
`data.icon` 缺失、以及边上 `relations[]` 缺 `label` / 自定义关系缺 `icon` 只记 WARNING（退出码仍为 0），但也要修：从 `bundle nodes` / `node-relations` 原样拷贝，否则控制台节点图标或连线出口文字/图标不正确。

## 环境里没有对应软件包 / 节点

步骤 4 必须用 CLI 核实，**禁止**按名称或 recipes 臆造 `bundleName` / 节点 `name`。

判定「没有」：

- `bundle list`（可加 `-k` 关键词）里没有能覆盖需求的软件包；或
- 包在，但 `bundle nodes` 展平结果里没有要用的节点类型。

此时**立刻停止**编 JSON / validate / 试跑 / 发布，向用户说明缺口，并请他用 `buildify-bundle-dev` 实现后发布，再回到本 skill 从步骤 4 继续。不要在本会话里偷偷写 Bundle 代码（除非用户明确说「那就去做这个 Bundle」）。

示例：

```
当前工作区没有能实现「xxx」的软件包/节点。

已查：bundle list / bundle nodes，未见 <包名> 或节点 <NodeName>。
（若有相近能力，列 1～3 个真实名称，问要不要改用；没有就不要编。）

请先用 buildify-bundle-dev 开发并发布该 Bundle（控制台可见、`buildify bundle list` 能列出），
然后再回来编排流程。需要的话我可以按 bundle-dev 帮你做节点。
```

相近但不完全匹配时：列出真实节点，请用户选「改用现有」或「新做 Bundle」。未选定前不要往下写流程。

### 按任务读取（勿一次全读）

| 场景 | 读 / 做 |
|---|---|
| **命令与退出码** | [reference/cli.md](reference/cli.md)，或 `buildify help` |
| **写 / 改流程 JSON** | [reference/flow-json.md](reference/flow-json.md) |
| **参数里的表达式** | 先看该字段 `uiComponent`。JSON（`JsonExpressionInput`）用 `={{msg.xxx}}`；SQL（`SqlEditor`）用 `#{msg.xxx}` / `${msg.xxx}`；文本及其他用 `{{msg.xxx}}`。细则 [reference/expressions.md](reference/expressions.md)，权威 [表达式总览](https://docs.buildify.cn/expr.html)。**禁止**把 `={{ }}` 写进 SQL 或纯文本 |
| **常见骨架** | [reference/recipes.md](reference/recipes.md)，节点名仍须用 CLI 向当前环境核实 |
| **节点参数合法键** | `buildify bundle node-properties -b <bundle> -n <NodeName>`（顺带看 `uiComponent`，用来选表达式语法） |
| **没有对应软件包 / 节点** | 停止编排。提示用户用 `buildify-bundle-dev` 实现并发布，`bundle list` 能看到后再从步骤 4 继续。禁止臆造 name / bundleName |
| **节点图标 / 类型名** | `bundle nodes` 展平后的 `icon` / `label` / `isTrigger`（分组型来自 `groups_json[].nodes[]`，扁平型来自 `nodes_json`）；`node-properties` 也会带回这些字段。写入 `data.icon` / `data.label` 时**原样拷贝**，不要编造 URL，也不要把场景描述写进 `label` |
| **节点 summary（画布默认文案）** | **必写**。画布节点卡片默认显示 `data.summary`，不是 `label`。按本流程职责写**短词**，**≤8 字**（如「接收请求」「查询设备」「上传 OSS」），不要写路径/字段名/长句，也**不要照抄**目录通用 `summary` |
| **画布排列** | **默认竖排**。主链同一 `x`（建议 360），`y` 从 80 起每步 +140；分支同 `y`、左右 `x ± 280`。能垂直汇合且边不穿过其他节点时再共用一个下游节点；否则各支路继续竖排。**必要时复制配置相同的节点**（新 `id`，同一套 parameters/credentials），避免连线交叉、斜穿。不要复制触发器。细则见 [flow-json.md](reference/flow-json.md) |
| **批注便签** | **默认不加**。见 [flow-json.md](reference/flow-json.md)「批注节点」；字段见 [comment-node.md](reference/comment-node.md)。只有用户明确要求，或分支条件 / 调用约定 / 凭证用途等 **`summary` 写不下且不写会误用** 时才加 1 张（主链左侧、不连线、`isLayoutNode: true`）。禁止每个节点贴一张，禁止复述 `summary` |
| **连线 relation** | `buildify bundle node-relations -b <bundle> -n <NodeName>`，把返回对象**整份**写入 `edges[].data.relations[]`（至少 `name`+`label`+`description`）。MCP / RelationCollection 自定义关系还要拷 `_id`、`icon`、`mcpKind`，并同步到源节点 `data.relations` |
| **项目 / 服务器 / 凭证（必问）** | 见上文「必须用户点选」。项目用 **projectName**；服务器用 **workerName + 描述 + 在线/离线**；凭证用 **label/name + type**。未确认不得写入 |
| **凭证引用格式** | 用户选定后填 `data.credentials.<槽> = {type, name}`；`type`/`name` 必须能在 `project credentials` 里找到 |
| **创建访问凭证** | 必须 `-p` 项目。先展示默认 name/label 和 `cred-properties` 字段，用户确认并给参数后再 `project create-credential`（stdin 传 data）。禁止臆造密钥、禁止 `--data` 上 argv、禁止回显明文 |
| **发布 / 上线** | 试跑成功后先确认版本号、发布说明，再 `flow deploy --version-name --remark --wait`。用返回的 `summary` 画表格。不要再 `-f`，不要再 `published` |

## 自动发布与发布摘要

步骤 9 试跑成功（`data.success == true` 且 `reason` 不是 timeout）后，发布到步骤 3 已确认的服务器。
草稿已在步骤 8 落库，**不要再传 flow.json**。

先确认（或让用户改）版本元数据，再 deploy：

```
准备发布，请确认版本信息（可改）：

- 版本号：v1.0.0          （用户没指定时给建议值，如 v1.0.0 或当前时间戳；不要擅自用时间戳直接发布）
- 发布说明：首次上线 /test2 当前时间接口   （根据本流程写一句，用户可改或说「不要说明」）
```

用户消息里已经写了版本号/说明，仍要回显一遍再发。未得到回复前不要 deploy。

```bash
buildify --json flow deploy -p "$PROJ" -w "$WF" --worker "$WKR" \
  --version-name "v1.0.0" --remark "首次上线 /test2 当前时间接口" --wait
```

省略 `--version-name` 时服务端会生成 `vyyyyMMdd-HHmmss`；省略 `--remark` 则该版本没有发布说明。这两项只写到**该发布版本**，不会改流程创建时的描述。

`--wait` 的 JSON 含 `versionName`、`publishRemark` 和完整 `summary`。直接用它画表格。

`flow published` **只在**用户事后问「现在线上是哪一版」时再调。

规则：

- 版本号、发布说明：deploy 前请用户确认或修改。`--version-name` 只作用于该发布版本；`--remark` 是发布说明，不是流程描述。未确认不要 deploy。
- 试跑失败、超时、或用户明确说「先不要发布」→ 停在草稿，不要 deploy。试跑后又改了 JSON，先 `save-draft` 再 deploy。
- `--wait` 会在 CLI 内轮询到结束，Agent **不要**再循环调 `flow deployment`。失败时退出码 4，把返回的 `summary.servers[].message` 原样告诉用户。
- 对用户说话只用名称：项目名、流程名、版本名、服务器表格里的 `workerName`。不要甩 projectId / workerId / deploymentId，除非用户要排障。
- **汇报必须用下方 Markdown 表格 + 状态图标**，不要改回纯列表。优先用响应里的 `deployStatusIcon` / `onlineIcon`；没有则按下表映射。

发布完成后**原样按这个结构输出**（字段来自 **这一次** `flow deploy --wait` 的 `data` / `data.summary`，不要为了填表再请求）：

```markdown
## ✅ 已发布

| 项 | 内容 |
| --- | --- |
| 项目 | 测试项目 |
| 流程 | 当前时间 API |
| 版本号 | `v1.0.0` |
| 发布说明 | 首次上线 /test2 当前时间接口 |
| 总体 | ✅ 成功 |
| 发布时间 | 2026-09-21 09:30:00 |

### 服务器

| 服务器 | 描述 | 在线 | 部署 | 说明 |
| --- | --- | --- | --- | --- |
| 本地java进程测试 | 测试专用 | 🟢 在线 | ✅ 成功 | — |

### 节点

**链路：** Webhook → JS → Webhook 响应

| 节点 | 职责 | 类型 | 对外路径 |
| --- | --- | --- | --- |
| Webhook | 接收请求 | ⚡ 触发 | `/test2` |
| JS | 返回时间 | 处理 | — |
| Webhook 响应 | 返回结果 | 处理 | — |

### 凭证

*本流程未引用访问凭证*
```

图标约定（与 API `*Icon` 字段一致）：

| 含义 | 图标 | 何时 |
| --- | --- | --- |
| 成功 | ✅ | `deployStatus=success`；标题写「已发布」 |
| 成功有警告 | ⚠️ | `success_with_warnings`；标题「已发布（有警告）」 |
| 失败 | ❌ | `failed`；标题「发布失败」，说明列写 `message` |
| 超时 | ⏰ | `timeout` |
| 部署中 / 待部署 | 🔄 / ⏳ | `running` / `pending` |
| 已取消 | 🚫 | `canceled` |
| 在线 / 离线 | 🟢 / ⚫ | `isOnline` |
| 触发节点 | ⚡ 触发 | `nodes[].isTrigger=true`，否则「处理」 |

失败时标题用 `## ❌ 发布失败`，服务器表「部署」列写 `❌ 失败`，说明列填 `message`。有凭证时改成表格，列：Bundle、类型、名称（`label` 优先，无密钥）。单元格里的 `|` 要转义。没有 webhookPath 的节点路径列写 `—`，不要整段省略节点表。

`flow published` 只用来事后查询线上版本，同样用这套表格。编排刚发布完不必再调。

## 工具（执行，勿读源码）

环境：Python 3.10+，已 `pip install buildify-cli`。

```bash
buildify help
printf '%s' 'keyId.secret' | buildify config set-key api_key
buildify --json key test
```

步骤 1 `key test` 若退出码 2 且 `data.reason=missing_api_key`：**停止编排**，把 `data.userMessage` 原样发给用户
（控制台「空间设置 → 开放 API 密钥 → 创建」，明文只显示一次）。用户把完整 `keyId.secret` 发过来后：

```bash
printf '%s' "$KEY" | buildify config set-key api_key
buildify --json config show    # 只看掩码
buildify --json key test
```

**密钥安全：** 不要 `--api-key`、不要写进仓库 / `.env` / commit、不要在后续回复里回显明文；保存后只展示 `config show` 的掩码。本 CLI 不能创建密钥。

机器可读输出一律加 `--json`，按退出码分支：`0` 成功，`2` 鉴权，`3` 本地 JSON/文件，`4` HTTP 4xx 或 validate 有 ERROR，`5` 网络。
