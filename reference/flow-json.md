# 流程 canvas JSON

控制台把编辑器画布存进 `workflow_version.workflow_json`。本 skill 生成的就是这份 **Vue Flow 文档**，
不是 worker 上的运行时 JSON。

`bundleId` 已废弃。节点必须写 `data.bundleName`。仓库里个别测试 fixture 仍用 `bundleId`，不要照抄。

## 顶层

```json
{
  "nodes": [],
  "edges": [],
  "viewport": { "x": 0, "y": 0, "zoom": 1 }
}
```

解析器只读 `nodes` 与 `edges`。`viewport` / `position` / `zoom` 是 UI 字段，但要给互不重叠的坐标，否则画布叠在一起。

**默认竖排**，主链一条垂直线，看起来整齐：

| | 建议 |
|---|---|
| 主链起点 | `{ "x": 360, "y": 80 }` |
| 主链步进 | `x` 不变，每步 `y += 140` |
| 分支 | 同 `y`，左右 `x ± 280`。能垂直汇合且边不穿过其他节点时，再回到主链一个节点；否则各支路继续竖排 |
| 对齐 | 同一列 `x` 相同，同一行 `y` 相同；不要斜向、交错、重叠、连线交叉 |

横排只在用户明确要求时用。

### 分支：必要时复制配置相同的节点

画布清晰优先于「少画一个节点」。两条支路都要做**同一件事**（同一 `data.name`、同一套 `parameters` / `credentials`），但共用一个节点会让边斜穿、交叉或绕行时：**每条支路各放一份**，不要强行汇合。

| 才复制 | 仍然共用 |
|---|---|
| 成功/失败（或多出口）各自收尾，例如各回一次 Webhook、各写一次库 | 线性流程，没有分叉 |
| 汇合边会穿过中间节点或和其他边交叉 | 左右支路正下方能干净接到同一个节点，边不穿过别人 |
| 为绕到共用节点必须斜线、回头、跨列 | 只是少画一个相同配置的节点 |

复制时：新 `id`；`name` / `bundleName` / `bundleVersion` / `icon` / `label` / `parameters` / `credentials` 与原配置一致（支路本身不同的字段除外）；`summary` 仍 ≤8 字，可相同或略区分（如「成功返回」「失败返回」）。**不要复制触发器**，也不要无意义地复制整条链。

## 节点（执行相关）

| 字段 | 必填 | 含义 |
|------|------|------|
| `id` | 是 | 稳定短 id，边的 source/target 引用它 |
| `type` | 否 | Vue Flow 视觉类型，用 `"default"` |
| `position` | 建议 | `{ "x": number, "y": number }`。**竖排**：主链同一 x、y 每步 +140 |
| `data.name` | 是 | Bundle 节点 id = `@FlowNodeDescription.name`，例如 `JsFunctionNode` |
| `data.bundleName` | 是 | 逻辑名，与 `buildify bundle list` 的 `bundleName` 完全一致 |
| `data.bundleVersion` | 是 | 钉死的版本；用 `bundle nodes` 回显的 `bundleVersion`，不要写 `latest` |
| `data.parameters` | 视表单 | 对象，键必须出现在该节点 `properties_json` |
| `data.credentials` | 视表单 | 对象：`{ "<槽名>": { "type": "PascalCase", "name": "实例名" } }`。`type`/`name` 必须来自用户从 `project credentials` 点选的那条，禁止默认第一条或编造 |
| `data.icon` | **建议** | 从 `bundle nodes`（或 `node-properties`）该项的 `icon` **原样拷贝**。通常是 `webhook.svg` 这种文件名，控制台靠它显示节点图标。不要改写成 OSS URL，也不要省略 |
| `data.label` | 建议 | 从目录 `label` **原样拷贝**节点类型名（如「Webhook」「JS 函数」）。不要把本流程职责写到这里 |
| `data.summary` | **建议** | 画布**默认显示**此字段。写该节点在**本流程**中的职责，**≤8 字**（最长 10 字）。不要照抄目录通用 `summary`，也不要写路径、SQL、字段名 |
| `data.isTrigger` | 建议 | 触发器写 `true`；目录里 `isTrigger: true` 的节点必须标上 |
| `data.disabled` | 否 | `true` 时跳过执行 |
| `data.isLayoutNode` | 否 | `true` 时解析器忽略（分组框、**批注**），不要给业务节点加这个 |

`type`（Vue Flow）**不是** bundle 节点类型。执行节点的类型是 `data.name`。批注用 Vue Flow `type: "comment"`，见下文。

槽名（credentials 的 key）等于表单里 `CredentialSelect` 字段的 `name`，常见为 `credentialsId`。用 `bundle node-properties` 确认，不要猜。实例名必须是用户从 `project credentials` 点选的，禁止默认第一条。

`label` 与 `summary` 分工：`label` 是节点类型名（跟目录走）；`summary` 是画布卡片上的默认文案，描述**这一颗**节点在本流程里做什么，**要短**。

| 差 | 好（≤8 字） |
|---|---|
| 接收 /devices Webhook | 接收请求 |
| 按 projectId 查询设备 | 查询设备 |
| 把 JSON 上传到 OSS all.json | 上传 OSS |

空 `summary` 时画布几乎没有可读信息，不要省略；超长会撑破卡片，也不要写。

## 批注节点（便签，不执行）

Markdown 便签，**默认不要加**。绝大多数流程只靠节点 `summary` 和连线就够读。解析器遇到 `data.isLayoutNode: true` 会跳过，**不要写** `bundleName` / `bundleVersion`，**不要连边**。可写字段与 Markdown 见 [comment-node.md](comment-node.md)。

**默认 0 张。** 只有下面这种情况才加，且通常 **最多 1 张**：

| 才加 | 仍然不加 |
|---|---|
| 用户明确要求批注 / 便签 / notes | 常规 Webhook→处理→应答、定时拉取、两节点回显 |
| 分支条件或互斥路径，光看 `summary` 会走错 | 复述节点 `summary` / `label` |
| 调用约定、凭证用途、单位/映射 **写进 `summary` 放不下**，不写会误用 | 空流程；每个执行节点贴一张 |

位置：主链 `x=360` 时批注 `x=20`，`y` 与被说明节点对齐；`style` 约 `280×160`～`320×220`，不要压住节点。`zIndex: 2`。

正文写 `parameters.commentMarkdown`（GFM，短：`##` 标题 + 一两句或要点）。已有正文时 **不要** `autoEdit: true`。**需要写时**用这个形状，不要默认拷进每份流程：

```json
{
  "id": "note-trigger",
  "type": "comment",
  "position": { "x": 20, "y": 80 },
  "zIndex": 2,
  "style": { "width": "280px", "height": "160px" },
  "data": {
    "name": "CommentNode",
    "label": "comment",
    "summary": "批注",
    "hasInput": false,
    "hasOutput": false,
    "isLayoutNode": true,
    "isShowConfigOnAdd": false,
    "autoEdit": false,
    "parameters": {
      "commentMarkdown": "## 怎么调用\n\n`GET /test2` 返回当前时间，无需登录。",
      "commentBackground": "amber",
      "commentBackgroundOpacity": 0.1
    }
  }
}
```

旧画布可能是 `type: "notes"`，读写时与 `comment` 同等对待。背景默认 `amber`；不要编造 `bundle` 节点名叫 CommentNode 去 `bundle nodes` 里找。

## 边

| 字段 | 必填 | 含义 |
|------|------|------|
| `id` | 建议 | `{source}_{target}` 即可 |
| `source` | 是 | 源节点 `id` |
| `target` | 是 | 目标节点 `id` |
| `data.relations` | 是 | 数组，至少一项。**整份拷贝**源节点 `bundle node-relations` 里对应的对象，不要只写 `name` |
| `data.relations[].name` | 是 | 必须是源节点 `node-relations` 里出现过的名字；MCP / RelationCollection 自定义关系用条目映射出的 `name`（如 `tools/get_user`） |
| `data.relations[].label` | **建议** | 画布出口文字。Success 一般是 `成功`。缺了控制台不显示连线文字 |
| `data.relations[].description` | 建议 | Tooltip；与目录一起拷贝 |
| `data.relations[].icon` | 视关系 | MCP 等自定义关系必拷（如 `mcp-tools` / `mcp-resources` / `mcp-prompts`）。目录没有 icon 的标准 Success/Failure 不要编造 |
| `data.relations[]._id` | 视关系 | 自定义关系（MCP Tools/Resources/Prompts、RelationCollection）必拷，用来把出口和图标绑到这条边 |
| `data.relations[].mcpKind` | 视关系 | MCP 自定义关系透传 `tools` / `resources` / `prompts` |

一条边可以带多个 relation（互斥出口）。普通成功路径用源节点目录里的成功项（通常 `name=Success`，`label=成功`）。

MCP / 动态出口：自定义关系还要写到**源节点** `data.relations`（与边上那份相同），否则画布左侧出口没有文字和图标。标准 Success/Failure 节点可以不写 `data.relations`，但**边上仍要带 label**。

MCP 自定义出口示例（源节点 `data.relations` 与边上都要同一份）：

```json
{
  "label": "获取用户信息",
  "name": "tools/get_user",
  "description": "获取用户的可用信息",
  "_id": "t-ozynvj4p",
  "mcpKind": "tools",
  "icon": "mcp-tools"
}
```

## 最小可运行例子

节点名、bundle 名、版本必须以当前环境的 CLI 输出为准。下面只说明形状：

```json
{
  "nodes": [
    {
      "id": "n1",
      "type": "default",
      "position": { "x": 360, "y": 80 },
      "data": {
        "name": "WebhookTrigger",
        "bundleName": "official/core",
        "bundleVersion": "1.0.0",
        "isTrigger": true,
        "icon": "webhook.svg",
        "parameters": {},
        "label": "Webhook",
        "summary": "接收请求"
      }
    },
    {
      "id": "n2",
      "type": "default",
      "position": { "x": 360, "y": 220 },
      "data": {
        "name": "JsFunctionNode",
        "bundleName": "official/core",
        "bundleVersion": "1.0.0",
        "icon": "code.svg",
        "parameters": { "code": "return { ok: true }" },
        "label": "JS",
        "summary": "处理请求"
      }
    }
  ],
  "edges": [
    {
      "id": "n1_n2",
      "source": "n1",
      "target": "n2",
      "data": { "relations": [{ "name": "Success", "label": "成功", "description": "节点执行成功，消息路由到此链路" }] }
    }
  ]
}
```

普通节点的输出在 payload 里包一层 `output`；触发器的 payload 在根级。占位名用 `msg`，不要用 `$json`。
**表达式按字段 `uiComponent` 写，不要一律 `={{ }}`**：JSON（`JsonExpressionInput`）用 `={{msg.xxx}}`；SQL（`SqlEditor`）用 `#{msg.xxx}` / `${msg.xxx}`；文本及其他用 `{{msg.xxx}}`。见 [expressions.md](expressions.md)。

## 校验顺序

`buildify flow validate` 按这个顺序报 `issues`：

1. 结构（缺 `node.id` / `data.name` / `data.bundleName` / `data.bundleVersion` / 边的 source/target / relation.name）
2. 节点在当前工作区是否存在且可访问
3. `data.parameters` 对照 `properties_json`（未知键、缺必填、枚举非法）
4. `edges[].data.relations[].name` 对照源节点 `relations_json`
5. `data.credentials` 对照项目里已有凭证（`bundleName + type + name`）
6. 图：至少一个触发器、孤立节点、环
7. 展示：目录里有 `icon` 但画布 `data.icon` 为空时记 **WARNING**（控制台节点图标会丢）
8. 展示：边上 `relations[]` 缺 `label`（或自定义关系缺 `icon` / `_id`）时记 **WARNING**（控制台出口文字/图标会丢）

`level=ERROR` 必须修；`WARNING` 尽量修。`valid: false` 时 CLI 退出码为 4。
