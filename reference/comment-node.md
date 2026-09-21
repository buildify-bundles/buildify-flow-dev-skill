# 画布批注（怎么写进流程 JSON）

Markdown 便签，**不执行、不连线**。权威正文是 `parameters.commentMarkdown` 字符串，不要写成 HTML。不要和 `type: "text"` 的 TextNode 搞混。

**默认不加。** 何时才加、坐标，见 [flow-json.md](flow-json.md)「批注节点」。下面只列写入画布时要用的字段。

## 必写字段

| 字段 | 取值 |
|---|---|
| `type` | `"comment"`（旧数据可能是 `"notes"`，读写同等对待） |
| `zIndex` | `2` |
| `style` | `{ "width": "280px", "height": "160px" }`（可按字数略调，约 180×88 起） |
| `data.name` | `"CommentNode"`（不要去 `bundle nodes` 里找这个名字） |
| `data.label` | `"comment"` |
| `data.summary` | `"批注"` |
| `data.hasInput` / `hasOutput` | `false` |
| `data.isLayoutNode` | `true`（解析器跳过，校验不会要 bundle） |
| `data.isShowConfigOnAdd` | `false` |
| `data.autoEdit` | 已有正文时 `false` 或省略 |

不要写 `bundleName` / `bundleVersion` / `credentials`，不要出现在任何 `edges` 里。

## parameters

编排时只写这三项（其余旧字段不用写）：

| 字段 | 用法 |
|---|---|
| `commentMarkdown` | GFM 正文。短：`##` 标题 + 一两句或要点 |
| `commentBackground` | 预设 id，默认 `amber` |
| `commentBackgroundOpacity` | `0.04`–`1`，默认 `0.1` |

背景常用：`amber`（默认）、`sand`、`sky`、`ghost`（无框）、`transparent`（透明虚线框）。不要写已废弃的 `paper` / `frost` / `lemon`。

## commentMarkdown 能写什么

预览按 GFM 渲染。工具栏支持的写法也可以直接写进 JSON：

- 标题 `## `，列表 `- ` / `1. `，引用 `> `
- `**粗体**` `*斜体*` `~~删除线~~` `` `行内代码` ``
- 代码块围栏、`[文字](url)`
- 图片 `![alt](url)`，可带宽度 `![alt](url =24)` 或 `![alt](url =24x24)`

不要把整段改成 HTML。颜色高亮只有画布编辑器会插入 `<span style="color:…;background-color:…">`，编排时一般不必手写。
