# MCPKit

[![Version](https://img.shields.io/badge/version-0.2.1-blue.svg)](https://github.com/Sunalamye/MCPKit/releases)
[![Swift](https://img.shields.io/badge/Swift-5.9+-orange.svg)](https://swift.org)
[![Platform](https://img.shields.io/badge/platform-macOS%2013%2B%20%7C%20iOS%2016%2B-lightgrey.svg)](https://developer.apple.com/macos/)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

Swift 版的 [Model Context Protocol](https://modelcontextprotocol.io/) 框架。
讓你的 App 開一個本機 endpoint，Claude Code 之類的 AI 助手就能直接呼叫你定義的工具。

---

## 設計

- **Protocol-first** — 工具是型別，不是字典。schema 與實作寫在一起，不會漂移
- **Core / Transport 分離** — 換傳輸層不用動工具
- **完整 async/await**
- **內建 HTTP server** — 不必自己接 networking

---

## 安裝

```swift
dependencies: [
    .package(url: "https://github.com/Sunalamye/MCPKit.git", from: "0.2.1")
]
```

---

## 快速開始

### 1. 定義工具

```swift
import MCPKit

struct MyTool: MCPTool {
    static let name = "my_tool"
    static let description = "做某件事"
    static let inputSchema = MCPInputSchema(
        properties: ["message": .string("輸入訊息")],
        required: ["message"])

    private let context: MCPContext
    init(context: MCPContext) { self.context = context }

    func execute(arguments: [String: Any]) async throws -> Any {
        guard let message = arguments["message"] as? String else {
            throw MCPToolError.missingParameter("message")
        }
        return ["result": "處理完成：\(message)"]
    }
}
```

### 2. 提供 Context

Context 是工具跟你的 App 之間的唯一介面。工具拿不到 App 的其他東西，
所以要暴露什麼由你決定。

```swift
@MainActor
final class MyAppContext: MCPContext {
    var serverPort: UInt16 = 8765

    func executeJavaScript(_ script: String) async throws -> Any? {
        // 沒有 WebView 就明確拒絕，不要回 nil 假裝成功
        throw MCPToolError.notAvailable("JavaScript execution")
    }

    func getLogs() -> [String] { myLogBuffer }
    func clearLogs() { myLogBuffer.removeAll() }
    func log(_ message: String) { print("[MyApp] \(message)") }
}
```

### 3. 啟動

```swift
@MainActor
func startServer() {
    let context = MyAppContext()
    let registry = MCPToolRegistry()

    registry.registerBuiltInTools()
    registry.register(MyTool.self)

    let server = MCPHTTPServer(context: context, registry: registry, port: 8765)
    server.start()
}
```

### 4. 自訂路由（選用）

MCP 之外還能掛一般的 HTTP endpoint，共用同一個 port：

```swift
server.get("/my-endpoint") { body, respond in
    respond(200, #"{"hello":"world"}"#, "application/json")
}

server.post("/my-action") { body, respond in
    respond(200, #"{"success":true}"#, "application/json")
}
```

---

## 結構

```
Sources/MCPKit/
├── Core/
│   ├── MCPTool.swift          工具協定與型別
│   ├── MCPContext.swift       執行脈絡協定
│   ├── MCPToolRegistry.swift  工具註冊表
│   └── MCPHandler.swift       JSON-RPC 處理
├── Transport/
│   └── MCPHTTPServer.swift    HTTP 傳輸層
└── Tools/
    └── BuiltInTools.swift     內建工具
```

### 內建路由

| 路徑 | 用途 |
|------|------|
| `/mcp` | MCP 協定端點（JSON-RPC） |
| `/status` | 伺服器狀態 |
| `/` | 說明頁 |

### 內建工具

| 名稱 | 說明 |
|------|------|
| `get_status` | 伺服器狀態 |
| `get_logs` | 取得日誌 |
| `clear_logs` | 清空日誌 |
| `execute_js` | 執行 JavaScript（需要 Context 有 WebView 支援） |

---

## 接到 Claude Code

```bash
claude mcp add --transport http my-app http://localhost:8765/mcp
```

> **只綁 loopback。** 建議把 server 的 `requiredInterfaceType` 設成 `.loopback`，
> 讓同網段的其他裝置連不進來——這個 endpoint 能執行你註冊的所有工具。

---

## 需求

| | 版本 |
|---|---|
| macOS | 13.0+ |
| iOS | 16.0+ |
| Swift | 5.9+ |

---

## License

[MIT](LICENSE)
