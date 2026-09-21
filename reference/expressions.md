# 表达式：按 uiComponent 选语法

权威说明：[表达式类型总览](https://docs.buildify.cn/expr.html)。
占位名一律 `msg`，不要用 `$json`。触发器字段在根级（`msg.xxx`），处理节点结果在 `msg.output`。

写 `data.parameters` 前先看 `node-properties` 的 **`uiComponent`**。**不要一律写 `={{ }}`**——那只适用于 JSON 表达式字段。

## 速查

| uiComponent | 变量语法 | 画布里怎么写 |
|---|---|---|
| **`JsonExpressionInput`** | `={{msg.xxx}}` | JSON **字符串值**写成 `"={{msg.output.id}}"`（整字段替换，**保留**数字/布尔/对象类型）。混排：`"=[{{msg.type}}] {{msg.title}}"` |
| **`SqlEditor`** | `#{msg.xxx}` / `${msg.xxx}` | **禁止** `{{ }}` 和 `={{ }}`。值用 `#{msg.id}`（预编译，防注入）；表名/列名用 `${msg.orderColumn}`（直接替换） |
| **文本及其他**（`Input` / `Password` / `ExpressionInput` / `CodeEditor` 文本·XML·HTML 等） | `{{msg.xxx}}` | 文案、URL、标题里写 `{{msg.output.title}}`。字段若 `expression: true`、或是 `ExpressionInput`、或 `CodeEditor` 默认开了表达式，**整段存储值以 `=` 开头**（模式标记，不是 JSON 语法）：`"={{msg.output.path}}"` 或 `"=https://x.com/{{msg.id}}"` |
| **`BooleanExpressionInput`** | 无括号 | 直接 SpEL：`"msg.output.status == 'ok'"`，不要写 `{{ }}` / `={{ }}` |
| **`CodeEditor` 且 `enableExpression: false`**（如 JS 节点） | 不用模板 | 脚本里用语言自身读 payload，例如 `return { echo: msg }` |

`JsonEditor` 是纯 JSON 数据，不要往里面塞 `={{ }}`；要在 JSON 里注入变量并保留类型，用 `JsonExpressionInput`。

## JSON：`={{ }}`（JsonExpressionInput）

```json
{
  "title": "={{msg.output.title}}",
  "level": "={{msg.output.level}}",
  "sender": "={{msg.output.sender}}"
}
```

| 写法 | 结果 |
|---|---|
| 整字段 `"level": "={{msg.output.level}}"` | 类型与变量一致（`3` 仍是数字，`false` 仍是布尔） |
| 混文本 `"subject": "=[{{msg.output.type}}] {{msg.output.title}}"` | **一定是字符串** |
| `"data": "={{msg.output}}"` | 整个对象/数组原样注入 |

`={{ }}` **不支持运算**（不要写 `={{ a + b }}`）。要算，用 JS 节点或布尔表达式。

## SQL：`#{}` / `${}`（SqlEditor）

```sql
SELECT * FROM ${msg.table}
WHERE id = #{msg.id}
  AND name = #{msg.name}
ORDER BY ${msg.orderColumn}
```

| 语法 | 作用 | 示例 |
|---|---|---|
| `#{msg.xxx}` | 预编译参数（推荐，防注入） | `WHERE id = #{msg.output.userId}` |
| `${msg.xxx}` | 直接字符串替换（表名、列名、排序字段） | `FROM ${msg.output.table}` |

`${}` **不转义**，不要把用户输入直接塞进去。动态条件/循环用 XML Dynamic SQL（`<if>` / `<foreach>` 等），见 [XML Dynamic SQL](https://docs.buildify.cn/concepts/xml_sql.html)。

## 文本及其他：`{{ }}`

```
{{msg.output.title}}：共 {{msg.output.count}} 条
订单号：{{msg.input.orderId}}
```

中间层可能为 null 时用 `?.`：`{{msg.output.user?.name}}`。路径区分大小写。

不要把文本字段写成 JSON 整字段替换——文本替换结果永远是字符串，语法是 `{{ }}`，不是 JSON 的 `={{ }}`。

## 布尔（If / Switch）

```
msg.output.status == 'ok' and msg.output.level > 2
```

字符串比较用**单引号**。可能为 null 的字段用 `?.`。
