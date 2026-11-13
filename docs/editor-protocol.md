# WebView 消息协议

Typedown 的 Windows 宿主和编辑器页面通过 WebView2 字符串消息交换 JSON。宿主入口是 `Typedown.Services.Transport`，页面入口是 `Dev/Typedown.Editor/src/services/transport.ts`。

## 信封格式

页面到宿主有三种消息。

普通事件：

```json
{"type":"message","name":"FileLoaded","args":{"text":"# title","loadId":7}}
```

差分事件：

```json
{"type":"diffmsg","diff":false,"name":"MarkdownChange","args":"{\"text\":\"…\",\"loadId\":7}"}
```

后续同名消息可以只发送变化区间：

```json
{"type":"diffmsg","diff":true,"name":"MarkdownChange","args":"replacement","start":20,"end":24}
```

`start` 和 `end` 是上一份 JSON 字符串中的替换范围。第一次必须发完整基线。`OnScroll`、`SelectionFormats` 和 `SelectionChange` 的完全相同状态可以跳过；命令和生命周期消息不能去重。

页面调用宿主函数：

```json
{"type":"invoke","id":"invoke_12","name":"GetSettings","args":null}
```

宿主通过标准的宿主到页面信封，以请求 `id` 作为 `name` 返回：

```json
{"name":"invoke_12","args":{"code":0,"data":{}}}
```

失败响应为 `{"code":1,"msg":"…"}`。页面默认等待 30 秒，并在成功、失败、发送异常或超时后移除监听器。导出和打印显式关闭默认超时，因为其完成时间由系统对话框决定。

宿主到页面统一使用：

```json
{"name":"LoadFile","args":{"text":"# title","basePath":"…","loadId":7}}
```

`MarkdownEditor.PostMessage` 序列化该对象并调用 `PostWebMessageAsString`；页面按 `name` 分发。

## 文档加载握手

```mermaid
sequenceDiagram
    participant H as Windows host
    participant E as Editor page
    H->>E: LoadFile(text, basePath, cursor, scrollTop, loadId)
    E->>E: 应用正文并等待首个内容报告
    E->>H: FileLoaded(same | text, loadId)
    E->>H: MarkdownChange / StateChange / CursorChange (..., loadId)
```

每次宿主加载文档都会递增 `loadId`。页面在所有文档相关报告中回传该值；宿主只接受当前值。快速切换时页面把同一帧内积压的 `LoadFile` 合并为最后一个。`FileLoaded` 报告页面实际载入的 Markdown，用于建立已保存基线：与该 `loadId` 收到的正文逐字相同时只发 `same: true`、不带正文（宿主用自己发出的那份；长文档回传一次要 20-40 ms），不同时才带 `text`，宿主据此记录改写并重载一次；不能用固定延迟猜测页面是否完成，也不能在握手过程中格式化源码。

保存、导出或关闭前，宿主发送：

```json
{"name":"FlushContent","args":{"token":18}}
```

页面立即完成节流中的正文报告并返回：

```json
{"type":"diffmsg","diff":false,"name":"ContentFlushed","args":"{\"token\":18,\"text\":\"…\"}"}
```

`token` 只完成对应等待。超时表示页面没有确认最新内容，调用方不能把它解释为“内容已同步”。

## 当前消息类别

| 方向 | 类别 | 主要名称 |
| --- | --- | --- |
| 页面 → 宿主事件 | 正文和生命周期 | `FileLoaded`, `MarkdownChange`, `ContentFlushed`, `StateChange`, `CursorChange`, `OutlineCurrent` |
| 页面 → 宿主事件 | 选择和滚动 | `SelectionChange`, `SelectionFormats`, `CodeMirrorSelectionChange`, `ReadingSelectionChange`（阅读模式下有没有选中文字：`{ selected }`，宿主据此启用复制）, `OnScroll`, `ScrollSettled` |
| 页面 → 宿主事件 | 查找 | `FindState`（查找栏的位置：`{ total, index }`，每次搜索、跳转、编辑后）, `FindResults`（结果列表开着时：`{ version, total, index, capped, entries: [{ text, start, end }] }`，最多 500 条，services/findResults.ts） |
| 页面 → 宿主命令 | 系统动作 | `OpenURI`, `PasteRequested`（浏览器自己的粘贴被页面拦下，交给宿主的粘贴） |
| 页面 → 宿主 invoke | 初始化和系统服务 | `ContentLoaded`, `GetSettings`, `GetCurrentTheme`, `GetStringResources`, `SetClipboard`, `OpenNewWindow`, `UnhandledException` |
| 页面 → 宿主 invoke | 导入导出工具 | `ExportCallback`, `PrintHTML`, `ResizeTable`, `GetPlantUmlPng` |
| 宿主 → 页面 | 文档和模式 | `LoadFile`, `SetMarkdown`, `FlushContent`, `SettingsChanged`, `RestoreScroll` |
| 宿主 → 页面 | 编辑命令 | `Copy`, `Cut`, `Paste`, `SelectAll`, `DeleteSelection`, `Format`, `InsertTable`, `InsertImage`, `ReplaceImage` |
| 宿主 → 页面 | 查找和导航 | `Search`, `Find`, `Replace`, `SearchOpenChange`, `FindResultsOpen`（侧栏的结果列表开/关：`{ open }`）, `FindActivate`（选中列表里的一条：`{ index, version }`）, `ScrollTo`, `OnScroll` |
| 宿主 → 页面 | 环境变化 | `ThemeChanged`, `LanguageChanged`, `Export`, `ImportFile` |

表格列出协议入口，具体 payload 以两端类型和处理器为准。`services/remote/common.ts` 中的 `LoadImage` 当前没有调用点和宿主处理器，不应作为可用协议使用；若恢复，必须同时实现并测试两端。

`GetPlantUmlPng` 接受一个 URL 字符串，返回 PNG 的 Base64 字符串或 `null`。只有打开 PlantUML 绘图时才请求配置的服务器，地址必须是该服务器的 `/png/~h` 十六进制图表路径；拒绝重定向，响应最多 16 MiB、15 秒。它用于保留不提供 CORS 的服务器的图片复制能力，不提供通用网络代理。

## 页面与本地资源边界

Windows 编辑器从 `https://editor.invalid/index.html` 加载打包的静态资源，开发配置只接受 `http://localhost:3000/`。原生消息必须同时匹配消息来源和当前页面的完整地址，不能只比较前缀；同文档的片段导航可用。打印预览只接受自己生成的临时 HTML 页面。

本地图片通过单独的 `https://local-images.invalid/__local_image` 请求。只接受图片上下文的 GET 请求和绝对图片路径，拒绝脚本、框架、fetch、设备路径和备用数据流。响应声明图片 MIME、`nosniff` 和不缓存，并只允许编辑器来源的匿名图片请求读取像素，供 SVG 复制为 PNG；Markdown、HTML 导出和剪贴板保留原有的本地路径，不包含内部地址。浏览器的跨源及文件访问保护保持开启。

本地图片同样最多 32 MiB、30 秒。本地和 HTTP 图片的原生读取总共最多并行四个，窗口关闭时取消等待及可取消的读取；UNC 文件的操作系统打开操作可能不会立即响应取消，但仍占用上述并行名额。

HTTP 图片（包括 HTTP PlantUML 服务器的预览）通过同一隔离来源的 `/__http_image` 显示，避免 HTTPS 页面的混合内容限制。此入口同样只接受图片上下文，限制响应为图片或二进制内容、32 MiB、30 秒；重定向最多五次且只能到无凭据的 HTTP(S) 地址。它不提供 fetch/文本读取能力，导出和复制仍使用原始网络地址。

## 扩展协议

新增或修改消息时必须：

1. 明确方向和唯一名称，给 payload 写 TypeScript 类型和对应 C# 类型或字段校验。
2. 与文档有关的异步结果携带 `loadId`；请求响应使用不可复用 token/id。
3. 定义发送失败、页面重载、超时、迟到和重复消息的行为。
4. 只有幂等状态才能进入去重集合。打开链接、加载完成、保存确认等命令不得去重。
5. 差分协议必须先有同名完整基线；修改差分算法时覆盖发送失败和基线失步。
6. 日志记录名称、id、大小和错误类别，不记录整篇正文、剪贴板内容或凭据。
7. 运行 `transport.test.ts`，并用 `Tools/EditorBench` 的 handshake、reading/source、security 和 spec 检查验证构建产物。

协议当前仍以字符串名称和 `JToken` 为主。重构时应逐步建立共享的消息清单和 payload 验证，避免一次性改动全部调用点。
