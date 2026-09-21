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
默认**竖排**：主链同一 x、y 递增；分支左右对称。能干净汇合再共用下游节点；否则各支路继续竖排，必要时复制配置相同的节点，避免连线交叉。
边上的 `relations` 必须从 `node-relations` **整份拷贝**（至少 `name` + `label` + `description`）。MCP 自定义关系还要带 `_id` / `icon` / `mcpKind`，并写到源节点 `data.relations`。
参数里的表达式按 `uiComponent` 写（JSON `={{ }}` / SQL `#{}` `${}` / 文本 `{{ }}`），见 [expressions.md](expressions.md)。不要把 Webhook 响应当成一个叫 `body` 的字段。

## 1. Webhook 触发 → 处理 → 应答

典型链路：入站 Webhook → JS/HTTP 处理 → Webhook 响应。

```json
{
  "nodes": [
    {
      "id": "trigger",
      "type": "default",
      "position": { "x": 360, "y": 80 },
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
      "position": { "x": 360, "y": 220 },
      "data": {
        "name": "JsFunctionNode",
        "label": "JS",
        "summary": "回显请求",
        "icon": "COPY_FROM_BUNDLE_NODES",
        "bundleName": "official/core",
        "bundleVersion": "REPLACE_ME",
        "parameters": {
          "code": "return { echo: msg }"
        }
      }
    },
    {
      "id": "reply",
      "type": "default",
      "position": { "x": 360, "y": 360 },
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
  ]
}
```

处理节点如果声明了凭证槽，先 `project credentials` 把该类型的实例（`label`/`name`）列给用户，**等用户指定后再写**，不要默认 `prod` 或第一条：

```json
"credentials": {
  "credentialsId": { "type": "HttpBasicAuth", "name": "用户选定的实例名" }
}
```

`type` / `name` 必须能在 `buildify project credentials -p <proj>` 里找到。项目与试跑服务器同样要用户点选（见 SKILL「必须用户点选」）。

`jsonValue` 是 `JsonExpressionInput`，整对象注入用 `={{ msg.output }}`。若改成纯文本响应（`outputType: text`，字段 `textValue`），变量写成 `{{ msg.output.title }}`，不要套 JSON 的 `={{ }}`。

## 2. 定时 → 拉取 → 落库

```
ScheduleTrigger → HttpRequestNode → 写入节点（以实际 bundle 为准）
```

1. `bundle nodes` 找到定时触发器，确认 `isTrigger`；`label` 拷目录类型名
2. 每个节点写短 `summary`（≤8 字，画布默认显示），如「每小时触发」「拉取列表」「写入 MySQL」
3. HTTP 节点的 URL / method 以 `node-properties` 为准，不要抄本文件的字段名。URL 等文本字段用 `{{ msg.xxx }}`；请求体若是 `JsonExpressionInput` 才写 `={{ msg.xxx }}`
4. 写入节点几乎一定要凭证：先 `project credentials` 列出名称请用户选；没有则请用户在控制台创建后再编，不要猜实例名
5. `SqlEditor` 字段用 `#{msg.xxx}`（值）和 `${msg.xxx}`（表名/列名），**不要**写 `={{ }}` 或 `{{ }}`：

```sql
SELECT * FROM ${msg.output.table}
WHERE id = #{msg.output.id}
```

坐标：**竖排**。主链 x=360，y=80 起每步 +140；分支同 y、x ±280。不要横排一条线。
**默认不加批注。** 仅当用户要求或分支/`summary` 写不下的关键约定不写会误用时，才在 x=20 加一张 `type: "comment"`（见 [flow-json.md](flow-json.md)）。

## 3. 分支：必要时复制节点

成功/失败（或多出口）各自还要做同一件事时，**不要**把两边绕到同一个节点上交叉。每条支路复制一份相同配置的节点，继续竖排：

```
              [判断]  x=360
             /      \
   [处理A] x=80    [处理B] x=640     ← 同 y
   [返回]  x=80    [返回]  x=640     ← 同一 Webhook 响应配置，两个 id
```

- 两份「返回」：`data.name` / `parameters` / `credentials` 相同，只换 `id`；`summary` 可写成「成功返回」「失败返回」
- 边只在本列向下走，不要斜穿到对面列
- 左右正下方能干净接到同一个节点、边不穿过别人时，仍共用，不必复制
- 不要复制触发器

## 4. 改已有流程

```bash
buildify --json flow get-draft -p "$PROJ" -w "$WF" > draft.json
# 取出 data.workflowJson 作为画布，改完后：
buildify --json flow validate -p "$PROJ" -f ./flow.json
buildify --json flow save-draft -p "$PROJ" -w "$WF" -f ./flow.json
```

不要从空模板覆盖已有草稿，除非用户明确要求重写。改节点职责时同步改 `data.summary`（仍 ≤8 字）。不要仅为竖排而大挪已有节点，用户要求重排时再对齐。
