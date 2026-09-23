# 编排骨架

下面是形状示例，**节点 `name` / `bundleName` / `bundleVersion` / relation / 参数键必须用 CLI 向当前环境核实**：

```bash
buildify --json bundle list
buildify --json bundle nodes -b official/core
buildify --json bundle node-properties -b official/core -n WebhookTrigger
buildify --json bundle node-relations  -b official/core -n WebhookTrigger
```

若环境里没有 `official/core` 或节点名不同，换成 `bundle list` / `bundle nodes` **实际返回**的值，不要臆造。
当前工作区完全没有能覆盖需求的包或节点时：**不要用本文件的骨架硬编**，停下来请用户先用 `buildify-bundle-dev` 实现并发布，再回来编排。
示例里的 `icon: COPY_FROM_BUNDLE_NODES` 必须换成 `bundle nodes` / `node-properties` 返回的 `icon` 字符串（如 `webhook.svg`），否则控制台节点图标不正确。
`label` 从目录拷贝类型名；`summary` 按本流程职责自写且 **≤8 字**（画布默认显示它），不要照抄目录 `summary`。
默认按顶层 `direction` 排，节点按约 240×88 占位、同层对齐。**`TB` 竖排**：主链同一 x、`y += 220`，相邻列 `x` 相差 400。**`LR` 横排**（用户要求或草稿已是横排）：主链同一 y、`x += 480`，相邻行 `y` 相差 240。连线 label 在该边中点，中点要落在两卡片之间、并且只属于这一条边。一条边超过 2 个出口 label 时沿流向再加长。能直连汇合且中点彼此分开再共用下游；否则各支路沿主方向继续排，必要时复制配置相同的节点，避免连线交叉。执行节点写 `sourcePosition` / `targetPosition`，与 `direction` 一致。
边上的 `relations` 必须从 `node-relations` **整份拷贝**（至少 `name` + `label` + `description`）。**同一对 source+target 只有一条边**；接到同一下游的多个出口（如 Success+Failure）全部放进该边 `data.relations[]`，不要拆成两条边。MCP 自定义关系还要带 `_id` / `icon` / `mcpKind`，并写到源节点 `data.relations`。
参数里的表达式按 `uiComponent` 写（JSON 整段 `"=` + `{{ }}`，如 `"=this is title {{msg.title}}"`，禁止 `this is ={{msg.title}}`；SQL `#{}` `${}`；文本 `{{ }}`），见 [expressions.md](expressions.md)。JS 节点 `code` **分段 + 中文注释**，不要写成无注释单行。不要把 Webhook 响应当成一个叫 `body` 的字段。
**真正写入画布时，节点 `id` 用 `n-` + nanoid（如 `n-k7mX2pL9`，流程内唯一）**，不要照抄下面的 `trigger` / `work` / `reply`。顶层必写 `errors`（通过 `{}`，未通过 `"errors":{"n-k7mX2pL9":1}`）和 `direction`（默认 `"TB"`），见 [flow-json.md](flow-json.md)。下面骨架默认 `TB`；用户要横排时改成 `"direction": "LR"`，并按 [flow-json.md](flow-json.md)「画布排列」改坐标和锚点。

## 1. Webhook 触发 → 处理 → 应答

典型链路：入站 Webhook → JS/HTTP 处理 → Webhook 响应。

```json
{
  "nodes": [
    {
      "id": "trigger",
      "type": "default",
      "position": { "x": 480, "y": 80 },
      "sourcePosition": "bottom",
      "targetPosition": "top",
      "data": {
        "name": "WebhookTrigger",
        "label": "Webhook",
        "summary": "接收请求",
        "icon": "COPY_FROM_BUNDLE_NODES",
        "bundleName": "official/core",
        "bundleVersion": "REPLACE_ME",
        "isTrigger": true,
        "parameters": {}
      }
    },
    {
      "id": "work",
      "type": "default",
      "position": { "x": 480, "y": 300 },
      "sourcePosition": "bottom",
      "targetPosition": "top",
      "data": {
        "name": "JsFunctionNode",
        "label": "JS",
        "summary": "回显请求",
        "icon": "COPY_FROM_BUNDLE_NODES",
        "bundleName": "official/core",
        "bundleVersion": "REPLACE_ME",
        "parameters": {
          "code": "// 回显入站 Webhook，供下游应答使用\n// 入参：msg（触发器字段在根级）；出参：{ echo }\n\n// --- 1. 取入参 ---\nconst payload = msg; // 原样带回，方便对照请求\n\n// --- 2. 返回下游 ---\nreturn {\n  echo: payload\n};"
        }
      }
    },
    {
      "id": "reply",
      "type": "default",
      "position": { "x": 480, "y": 520 },
      "sourcePosition": "bottom",
      "targetPosition": "top",
      "data": {
        "name": "WebhookResponseNode",
        "label": "Webhook 响应",
        "summary": "返回结果",
        "icon": "COPY_FROM_BUNDLE_NODES",
        "bundleName": "official/core",
        "bundleVersion": "REPLACE_ME",
        "parameters": {
          "responseType": "custom",
          "outputType": "json",
          "jsonValue": "={{ msg.output }}",
          "options": { "responseStatusCode": 200 }
        }
      }
    }
  ],
  "edges": [
    {
      "id": "trigger_work",
      "source": "trigger",
      "target": "work",
      "data": { "relations": [{ "name": "Success", "label": "成功", "description": "节点执行成功，消息路由到此链路" }] }
    },
    {
      "id": "work_reply",
      "source": "work",
      "target": "reply",
      "data": { "relations": [{ "name": "Success", "label": "成功", "description": "节点执行成功，消息路由到此链路" }] }
    }
  ],
  "direction": "TB",
  "errors": {}
}
```

处理节点如果声明了凭证槽，先 `project credentials` 把该类型的实例（`label`/`name`）列给用户，**等用户指定后再写**，不要默认 `prod` 或第一条：

```json
"credentials": {
  "credentialsId": { "type": "HttpBasicAuth", "name": "用户选定的实例名" }
}
```

`type` / `name` 必须能在 `buildify project credentials -p <proj>` 里找到。项目与试跑服务器同样要用户点选（见 SKILL「必须用户点选」）。

`jsonValue` 是 `JsonExpressionInput`：`=` 必须在**整段字符串开头**。整对象注入用 `"={{ msg.output }}"`；混排用 `"=this is title {{msg.title}}"`，不要写成 `"this is ={{msg.title}}"`。若改成纯文本响应（`outputType: text`，字段 `textValue`），变量写成 `{{ msg.output.title }}`，不要套 JSON 的 `=` 前缀。

## 2. 定时 → 拉取 → 落库

```
ScheduleTrigger → HttpRequestNode → 写入节点（以实际 bundle 为准）
```

1. `bundle nodes` 找到定时触发器，确认 `isTrigger`；`label` 拷目录类型名
2. 每个节点写短 `summary`（≤8 字，画布默认显示），如「每小时触发」「拉取列表」「写入 MySQL」
3. HTTP 节点的 URL / method 以 `node-properties` 为准，不要抄本文件的字段名。URL 等文本字段用 `{{ msg.xxx }}`；请求体若是 `JsonExpressionInput`，字符串值整段以 `=` 开头、插值 `{{ msg.xxx }}`（如 `"=this is title {{msg.title}}"`）
4. 写入节点几乎一定要凭证：先 `project credentials` 列出名称请用户选；没有则请用户在控制台创建后再编，不要猜实例名
5. `SqlEditor` 字段用 `#{msg.xxx}`（值）和 `${msg.xxx}`（表名/列名），**不要**写 `{{ }}` 或给 SQL 加 `=` 前缀：

```sql
SELECT * FROM ${msg.output.table}
WHERE id = #{msg.output.id}
```

坐标按 `direction`：默认 **`TB` 竖排**（主链 x=480，y=80 起每步 +220；分支同 y，相邻列 x 相差 400）。用户要横排或草稿已是 `LR` 时用 **`LR`**（主链 y=280，x=80 起每步 +480；分支同 x，相邻行 y 相差 240），并写 `sourcePosition` / `targetPosition`。不要把 `TB` 排成一条横线，也不要把 `LR` 排成一列竖线。层距不要收成 140，否则连线 label 会压在卡片上，看不出属于哪条边。
**默认不加批注、不加分组。** 仅当用户要求或分支/`summary` 写不下的关键约定不写会误用时，才加一张 `type: "comment"`（放在整图外侧，不要挡住连线中点；见 [flow-json.md](flow-json.md)）。分组时组内节点必须写 `parentNode`。

## 3. 分支：必要时复制节点

成功/失败接到**同一个**下游时，用**一条边**把两个 relation 都放进 `data.relations`（不要拆边）：

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
  "zIndex": 2000
}
```

成功/失败（或多出口）各自还要做同一件事、且接到**不同**下游时，**不要**把两边绕到同一个节点上交叉。每条支路复制一份相同配置的节点，沿主方向继续排；每条边的 `relations` 只含该支路的出口。

`TB` 竖排（判断在 y=80，下一层 y=300，列距 400，label 落在各自斜线中段）：

```
                 [判断]  x=480
                /      \
   [处理A] x=80        [处理B] x=880     ← 同 y
   [返回]  x=80        [返回]  x=880     ← 同一 Webhook 响应配置，两个 id
```

`LR` 横排（判断在 x=80，下一列 x=560，行距 240）：

```
[判断] y=280
   |          \
[处理A] y=40  [处理B] y=520     ← 同 x
[返回]  y=40  [返回]  y=520     ← 同一配置，两个 id
```

- 两份「返回」：`data.name` / `parameters` / `credentials` 相同，**各抽一个新 `n-` + nanoid** 作 `id`；`summary` 可写成「成功返回」「失败返回」
- 边只沿本列/本行走，不要斜穿到对面；每条边 `id`=`{source}_{target}`，`relations` 只含该支路出口
- 支路沿流向正前方能干净接到同一个节点、边不穿过别人时，仍共用：一条边，`relations` 里同时放 Success 和 Failure
- 不要复制触发器

## 4. 改已有流程

```bash
buildify --json flow get-draft -p "$PROJ" -w "$WF" > draft.json
# 取出 data.workflowJson 作为画布，改完后：
buildify --json flow validate -p "$PROJ" -f ./flow.json
buildify --json flow save-draft -p "$PROJ" -w "$WF" -f ./flow.json
```

不要从空模板覆盖已有草稿，除非用户明确要求重写。改节点职责时同步改 `data.summary`（仍 ≤8 字）。沿用草稿的 `direction`，不要仅为改排列而大挪已有节点；用户要求重排或改方向时再对齐。

## 5. 节点分组

用户要求分组时：先放 `type: "group"` 的布局框，组内每个组件写 `parentNode` 指向框的 `id`。只重叠坐标不算分组。组内留白，不要贴边：首节点 `{ "x": 48, "y": 72 }`；组内间距跟顶层 `direction`（`TB` 时 `y += 220`、相邻列 `x` 相差 400，`LR` 时 `x += 480`、相邻行 `y` 相差 240），框比节点包络更大。

```json
{
  "id": "2",
  "type": "group",
  "position": { "x": 40, "y": 40 },
  "style": { "width": "400px", "height": "420px" },
  "data": { "label": "上报", "summary": "上报", "isLayoutNode": true }
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

真正写入时 `id` 换成 `n-` + nanoid；`2a` 补齐业务节点字段，建议 `"extent": "parent"`。边连业务节点，不连分组框。细则见 [flow-json.md](flow-json.md)「节点分组」。
