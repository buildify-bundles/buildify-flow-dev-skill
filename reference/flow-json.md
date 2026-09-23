# 流程 canvas JSON

控制台把编辑器画布存进 `workflow_version.workflow_json`。本 skill 生成的就是这份 **Vue Flow 文档**，
不是 worker 上的运行时 JSON。

`bundleId` 已废弃。节点必须写 `data.bundleName`。仓库里个别测试 fixture 仍用 `bundleId`，不要照抄。

## 顶层

```json
{
  "nodes": [],
  "edges": [],
  "viewport": { "x": 0, "y": 0, "zoom": 1 },
  "direction": "TB",
  "errors": {}
}
```

解析器只读 `nodes` 与 `edges`。`viewport` / `position` / `zoom` 是 UI 字段，但要给互不重叠的坐标，否则画布叠在一起。
顶层 **`errors` 必写**：表单校验结果按节点 id 汇总。全部通过写 `{}`；未通过写 `{"<nodeId>": <未通过字段数>}`。`flow validate` 之后把这份汇总写回 JSON 再 save-draft。
顶层 **`direction` 必写**：控制台画布排列方向，决定主链怎么走、连线从哪边进出。缺了控制台按 `TB`。

## 画布排列（按 `direction`）

控制台工具栏只有两种：`TB` 竖向、`LR` 横向。编排时**必须写顶层 `direction`**，并按同一方向排 `position`、给每个执行节点写 `sourcePosition` / `targetPosition`。不要混用（例如 `direction: "LR"` 却竖着排）。

**默认 `TB`。** 用户明确要求横排 / 横向 / 从左到右 / `LR` 时才用 `LR`。改已有流程时沿用草稿里的 `direction`，不要仅为换方向而大挪节点，除非用户要求重排。

节点不要当成一个点。执行节点按卡片 **宽 240、高 88** 占位来留白。连线出口文字画在**这条边的路径中点**。中点必须落在两张卡片之间的空白里，并且这片空白只属于这一条边，读者才能看出字在哪条线上。

| `direction` | 含义 | 主链 | 沿流向步进 | 分支（垂直于流向） | 执行节点锚点 |
|---|---|---|---|---|---|
| `TB`（默认） | 上→下竖排 | 同一 `x`（建议 480），`y` 从 80 起 | `y += 220` | 同一层 `y` 相同，相邻列 `x` 相差 **400** | `sourcePosition: "bottom"`，`targetPosition: "top"` |
| `LR` | 左→右横排 | 同一 `y`（建议 280），`x` 从 80 起 | `x += 480` | 同一列 `x` 相同，相邻行 `y` 相差 **240** | `sourcePosition: "right"`，`targetPosition: "left"` |

一条边上的出口 label 多于 2 个时，它们叠在**同一条线**的中点，这条边要沿流向再加长：每多 1 个，`TB` 再 `+36`，`LR` 再 `+48`。不要把多个出口拆成多条边来「分开 label」。

`TB` 竖排（主链直线，label 在竖线中段）：

```
[触发]  x=480 y=80
   |        ← 「成功」在这条竖线中点
[处理]  x=480 y=300
   |
[应答]  x=480 y=520
```

`LR` 横排：

```
[触发] x=80 y=280  →  [处理] x=560 y=280  →  [应答] x=1040 y=280
```

分叉时目标放在**下一层**，左右（或上下）对称，列距/行距用上表。不要和源节点放在同一层紧旁边——那条边又短又斜，两条 label 会挤在出口上。

```
TB                         LR

        [判断] y=80                 [判断] x=80
       /      \                        |        \
[A] x=80    [B] x=880         [A] y=40  [B] y=520
 y=300       y=300             x=560     x=560
```

`TB` 里 `[判断]` 的 `x=480`，左右列是 `480 ± 400`。`LR` 里 `[判断]` 的 `y=280`，上下行是 `280 ± 240`。

规则（两种方向共用）：

- 同层对齐：`TB` 同一层 `y` 相同，`LR` 同一列 `x` 相同。用上表的间距，不要斜着错开半格
- 主链优先走直线（`TB` 同 `x`，`LR` 同 `y`）。直线的中点就在那条线上
- 斜边只用于「源在这一层、目标在下一层的另一列/行」。沿流向至少用满上表步进，曲线到中点时已经分开，label 靠近各自那一列，而不是堆在源节点出口
- 相邻两条边的中点至少分开约 **160**。会贴在一起就加大列距/行距，或改成各自沿主方向直走并复制下游节点
- 不要交叉、不要穿过别的节点、不要让边的中点落在卡片上。层距不要收成 140：那时两卡片之间只剩约 50px，label 会压在端点上
- 能沿流向直连汇合、且汇入边的中点彼此分开时，再共用下游；否则各支路继续沿主方向排
- 分组框、批注不要写 `sourcePosition` / `targetPosition`
- 批注放在整图外侧，不要挡住任何边的中点。见下文「批注节点」

### 分支：必要时复制配置相同的节点

画布清晰优先于「少画一个节点」。两条支路都要做**同一件事**（同一 `data.name`、同一套 `parameters` / `credentials`），但共用一个节点会让边斜穿、交叉或绕行时：**每条支路各放一份**，不要强行汇合。

| 才复制 | 仍然共用 |
|---|---|
| 成功/失败（或多出口）各自收尾，例如各回一次 Webhook、各写一次库 | 线性流程，没有分叉 |
| 汇合边会穿过中间节点或和其他边交叉 | 支路沿流向正前方能干净接到同一个节点，边不穿过别人 |
| 为绕到共用节点必须斜线、回头、跨列 | 只是少画一个相同配置的节点 |

复制时：**重新抽一个 `n-` + nanoid** 作新 `id`（不要复用、不要 `n1`/`n2`）；`name` / `bundleName` / `bundleVersion` / `icon` / `label` / `parameters` / `credentials` 与原配置一致（支路本身不同的字段除外）；`summary` 仍 ≤8 字，可相同或略区分（如「成功返回」「失败返回」）。**不要复制触发器**，也不要无意义地复制整条链。

## 节点（执行相关）

| 字段 | 必填 | 含义 |
|------|------|------|
| `id` | 是 | **`n-` + nanoid**（如 `n-k7mX2pL9`），本流程内唯一。边的 `source` / `target` 引用它。真正生成时不要用 `n1` / `trigger` 这类可读名 |
| `type` | 否 | Vue Flow 视觉类型，用 `"default"` |
| `position` | 建议 | `{ "x": number, "y": number }`。按顶层 `direction`：`TB` 主链同一 x、y 每步 +220，相邻列 x 相差 400；`LR` 主链同一 y、x 每步 +480，相邻行 y 相差 240。一条边超过 2 个出口 label 时沿流向再加长 |
| `sourcePosition` | 建议 | 执行节点连线出口边。与 `direction` 一致：`TB` 用 `"bottom"`，`LR` 用 `"right"`。分组框/批注不要写 |
| `targetPosition` | 建议 | 执行节点连线入口边。与 `direction` 一致：`TB` 用 `"top"`，`LR` 用 `"left"`。分组框/批注不要写 |
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
| `parentNode` | 组内必写 | 分组框的 `id`。组内组件**必须**设置后才是真实分组；不写则只是画在附近。`position` 相对分组框左上角，**留白不要贴边** |

`type`（Vue Flow）**不是** bundle 节点类型。执行节点的类型是 `data.name`。批注用 Vue Flow `type: "comment"`，见下文。

槽名（credentials 的 key）等于表单里 `CredentialSelect` 字段的 `name`，常见为 `credentialsId`。用 `bundle node-properties` 确认，不要猜。实例名必须是用户从 `project credentials` 点选的，禁止默认第一条。

### 节点 id：`n-` + nanoid，流程内唯一

每个节点（含批注、分组框、复制出来的节点、组内子节点）的 `id` 用 **`n-` 前缀 + 8 位 nanoid** 生成，保证**本流程内**不重复。边的 `source` / `target` 必须引用这个 id。已有草稿上的旧 id 不要改（改了边会对不上）；只给**新生成**的节点抽新 id。

```bash
python3 -c "import secrets,string; a=string.ascii_letters+string.digits; print('n-' + ''.join(secrets.choice(a) for _ in range(8)))"
```

得到如 `n-k7mX2pL9`。抽到已占用的值就重抽。边的 `id` 仍用 `{source}_{target}`。文档示例里的 `n1` / `trigger` 只为便于阅读，真正写入画布时不要照抄。

### 表单校验：`errors` 写入流程 JSON

画布顶层必须带 `errors`，按节点 id 汇总未通过的表单字段数。`flow validate` 之后**写回这份 JSON**，不要只读响应、不落盘。

全部通过：

```json
{"errors":{}}
```

未通过（key 是节点 `id`，value 是该节点未通过字段数）：

```json
{"errors":{"n-k7mX2pL9":1}}
```

多个节点：`"errors":{"n-k7mX2pL9":1,"n-CIq2dZyc":2}`。只统计表单参数 ERROR（缺必填、枚举非法、未知键），不要把 WARNING 写进去。

处理：用 key 在 `nodes[]` 里定位 → 对照 `node-properties` 修 `data.parameters` → 再 `flow validate` → 把新的 `errors` 写进 JSON。用户明确说稍后自填时，仍要把当前汇总写进去再 save-draft。`issues[].nodeId` 与此同一套 id。

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

位置在整图外侧，不要压住节点，也不要挡住连线中点（label 所在的那一段空白）：

- `TB`：放在最左一列的左侧。`x = 最左节点 x - 340`；若小于 40，把所有节点和分组框整体右移，使批注 `x = 40`。`y` 与被说明节点对齐
- `LR`：放在最上行的上方。`y = 最上节点 y - 200`；若小于 40，把所有节点和分组框整体下移，使批注 `y = 40`。`x` 与被说明节点对齐

`style` 约 `280×160`～`320×220`。`zIndex: 2`。不要写 `sourcePosition` / `targetPosition`。无分支、主链 `x=480` 时，批注可以用 `{ "x": 40, "y": 80 }`。

正文写 `parameters.commentMarkdown`（GFM，短：`##` 标题 + 一两句或要点）。已有正文时 **不要** `autoEdit: true`。**需要写时**用这个形状，不要默认拷进每份流程；`id` 换成 `n-` + nanoid，不要照抄 `note-trigger`：

```json
{
  "id": "note-trigger",
  "type": "comment",
  "position": { "x": 40, "y": 80 },
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

旧画布可能是 `type: "notes"`，读写时与 `comment` 同等对待。背景默认 `amber`；不要编造 `bundle` 节点名叫 CommentNode 去 `bundle nodes` 里找。**不要用批注当分组框**（分组见下一节）。

## 节点分组

画布分区用 Vue Flow 父子节点。**默认不加。** 只在用户要求分组，或画布上有两块以上独立子流程需要视觉分区时才加。

**组内组件必须写 `parentNode`，否则不是真实分组。** 只把节点画在框的坐标范围内、或用批注撑一块背景，控制台不会把它们当成一组。

分组框（父节点）是布局节点：`type: "group"`，`data.isLayoutNode: true`，给足 `style` 宽高。**不要写** `bundleName` / `bundleVersion`，**不要出现在任何 `edges` 里**。组内业务节点照常写 `data.name` / `bundleName` / 参数，并连边。

| 字段 | 谁写 | 含义 |
|------|------|------|
| 分组框 `id` | 框 | **`n-` + nanoid**，流程内唯一 |
| 分组框 `type` | 框 | `"group"` |
| 分组框 `position` | 框 | 画布绝对坐标（框的左上角） |
| 分组框 `style` | 框 | 必须带 `width` / `height`。按子节点包络**再加边距**算出，不要刚好卡住 |
| 分组框 `data.isLayoutNode` | 框 | `true`（解析器跳过） |
| 分组框 `data.label` / `summary` | 框 | 分区名，短词（如「上报」「查询」） |
| 子节点 `parentNode` | **组内每个组件必写** | 分组框的 `id`。不写就分不进去 |
| 子节点 `position` | 组内 | **相对**分组框左上角。留白：左/右 ≥40，上 ≥64（给标题），下 ≥40。**不要贴边** |
| 子节点 `extent` | 建议 | `"parent"`，拖动时留在框内 |

形状（子节点用 `parentNode` 挂到框上）：

```json
{
  "id": "2",
  "type": "group",
  "position": { "x": 40, "y": 40 },
  "style": { "width": "400px", "height": "420px" },
  "zIndex": 0,
  "data": {
    "label": "分组",
    "summary": "分组",
    "isLayoutNode": true
  }
}
```

```json
{
  "id": "2a",
  "data": { "label": "child node" },
  "position": { "x": 48, "y": 72 },
  "parentNode": "2"
}
```

真正写入画布时：框和子节点的 `id` 都换成 `n-` + nanoid；子节点补齐业务字段（`type: "default"`、`data.name` / `bundleName` / `bundleVersion` / `icon` / `summary` / `parameters`），并建议加 `"extent": "parent"`。

| 才加 | 仍然不加 |
|---|---|
| 用户明确要求分组 / 分区 / group | 线性 Webhook→处理→应答 |
| 两块以上独立子流程（如「上报」和「查询」）要视觉分开 | 用批注复述 `summary`；每个节点一个框 |

规则：

- 组内每个执行节点都要写 `parentNode`，漏一个就漂在框外。
- **组内留白，不要贴边。** 首个子节点 `{ "x": 48, "y": 72 }`；组内排版跟顶层 `direction`，间距与顶层相同：`TB` 时 `y += 220`、相邻列 `x` 相差 400，`LR` 时 `x += 480`、相邻行 `y` 相差 240。左/右 ≥40，上 ≥64，下 ≥40。不要再用画布绝对坐标，也不要用 `{ "x": 10, "y": 50 }` 这种贴边坐标。
- 框的 `width` / `height` = 子节点包络 + 四周边距。`TB` 单列大约宽 400，高 ≈ 72 + 节点数 × 220 + 40；`LR` 单行大约高 280，宽随节点数按步进 480 加。组内有分支时，框要包住最外一列并再留 ≥40。不要刚好包住卡片。
- 边仍连业务节点，不要连分组框。组与组之间的连线照常 `source`/`target` 业务 id。
- 不要用 `type: comment` 当分组框。不要给业务节点加 `isLayoutNode`。
- 一层即可，不要嵌套分组，除非用户明确要求。

## 边

**同一对 `source` + `target` 只能有一条边。** 多个出口接到同一个下游时，不要拆成两条边，把这些 relation 全部放进这一条的 `data.relations[]`。

| 字段 | 必填 | 含义 |
|------|------|------|
| `id` | 建议 | `{source}_{target}` |
| `type` | 建议 | `"default"` |
| `source` | 是 | 源节点 `id` |
| `target` | 是 | 目标节点 `id` |
| `data.relations` | 是 | 数组，至少一项。接到**本 target** 的出口全部列在这里。**整份拷贝**源节点 `bundle node-relations` 里对应的对象，不要只写 `name` |
| `data.relations[].name` | 是 | 必须是源节点 `node-relations` 里出现过的名字；MCP / RelationCollection 自定义关系用条目映射出的 `name`（如 `tools/get_user`） |
| `data.relations[].label` | **建议** | 画布出口文字。Success 一般是 `成功`，Failure 一般是 `失败`。缺了控制台不显示连线文字 |
| `data.relations[].description` | 建议 | Tooltip；与目录一起拷贝 |
| `data.relations[].icon` | 视关系 | MCP 等自定义关系必拷（如 `mcp-tools` / `mcp-resources` / `mcp-prompts`）。目录没有 icon 的标准 Success/Failure 不要编造 |
| `data.relations[]._id` | 视关系 | 自定义关系（MCP Tools/Resources/Prompts、RelationCollection）必拷，用来把出口和图标绑到这条边 |
| `data.relations[].mcpKind` | 视关系 | MCP 自定义关系透传 `tools` / `resources` / `prompts` |
| `label` | 建议 | 边自身标签，固定 `""`（出口文字在 `relations[].label`） |
| `zIndex` | 建议 | `2000` |
| `sourceX` / `sourceY` / `targetX` / `targetY` | 否 | Vue Flow 锚点，生成时可省略（控制台会算） |

规则：

- **多个出口 → 同一下游**：一条边，`relations` 里放多项（如 Success + Failure 都接到下一节点）。这些 label 叠在同一条线的中点，把这条边按「画布排列」加长，不要拆边。
- **多个出口 → 不同下游**：每个 target 一条边，各自 `relations` 只含该支路的出口。目标放在下一层的不同列/行，让每条边的中点分开，label 才能对上自己那条线。
- 禁止两条边共用同一 `source` + `target`。
- 普通成功路径只用源节点目录里的成功项（通常 `name=Success`，`label=成功`）。

多个输出接到同一下游的边（控制台结构）：

```json
{
  "id": "n-k1V6QF1P_n-8aunB6B4",
  "type": "default",
  "source": "n-k1V6QF1P",
  "target": "n-8aunB6B4",
  "data": {
    "relations": [
      {
        "name": "Failure",
        "label": "失败",
        "description": "节点执行失败，消息路由到此链路"
      },
      {
        "name": "Success",
        "label": "成功",
        "description": "节点执行成功，消息路由到此链路"
      }
    ]
  },
  "label": "",
  "zIndex": 2000,
  "sourceX": 456.875,
  "sourceY": 990.875,
  "targetX": 456.875,
  "targetY": 1049.875
}
```

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

节点名、bundle 名、版本必须以当前环境的 CLI 输出为准。下面只说明形状；**真正生成时把 `n1` / `n2` 换成 `n-` + nanoid**（流程内唯一）。

```json
{
  "nodes": [
    {
      "id": "n1",
      "type": "default",
      "position": { "x": 480, "y": 80 },
      "sourcePosition": "bottom",
      "targetPosition": "top",
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
      "position": { "x": 480, "y": 300 },
      "sourcePosition": "bottom",
      "targetPosition": "top",
      "data": {
        "name": "JsFunctionNode",
        "bundleName": "official/core",
        "bundleVersion": "1.0.0",
        "icon": "code.svg",
        "parameters": { "code": "// 处理请求，返回成功标记\n// 入参：msg；出参：{ ok }\n\nreturn { ok: true };" },
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
  ],
  "viewport": { "x": 0, "y": 0, "zoom": 1 },
  "direction": "TB",
  "errors": {}
}
```

普通节点的输出在 payload 里包一层 `output`；触发器的 payload 在根级。占位名用 `msg`，不要用 `$json`。
**表达式按字段 `uiComponent` 写**：JSON（`JsonExpressionInput`）字符串值**整段以 `=` 开头**，插值用 `{{msg.xxx}}`（✅ `"=this is title {{msg.title}}"` / `"={{msg.xxx}}"`；❌ `"this is ={{msg.title}}"`）；SQL（`SqlEditor`）用 `#{msg.xxx}` / `${msg.xxx}`；文本及其他用 `{{msg.xxx}}`。JS 节点 `code` 分段并加中文注释。见 [expressions.md](expressions.md)。

## 校验顺序

`buildify flow validate` 按这个顺序报 `issues`。每次校验后把表单 ERROR 按节点 id 汇总写入画布顶层 `errors`：通过为 `{}`，未通过如 `"errors":{"n-k7mX2pL9":1}`。

顺序：

1. 结构（缺 `node.id` / `data.name` / `data.bundleName` / `data.bundleVersion` / 边的 source/target / relation.name）
2. 节点在当前工作区是否存在且可访问
3. `data.parameters` 对照 `properties_json`（未知键、缺必填、枚举非法）
4. `edges[].data.relations[].name` 对照源节点 `relations_json`
5. `data.credentials` 对照项目里已有凭证（`bundleName + type + name`）
6. 图：至少一个触发器、孤立节点、环
7. 展示：目录里有 `icon` 但画布 `data.icon` 为空时记 **WARNING**（控制台节点图标会丢）
8. 展示：边上 `relations[]` 缺 `label`（或自定义关系缺 `icon` / `_id`）时记 **WARNING**（控制台出口文字/图标会丢）

`level=ERROR` 必须修；`WARNING` 尽量修。`valid: false` 时 CLI 退出码为 4。
