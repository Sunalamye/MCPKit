# MCPKit Reference

Complete API reference for MCPKit Swift package.

## MCPTool Protocol

```swift
public protocol MCPTool {
    static var name: String { get }
    static var description: String { get }
    static var inputSchema: MCPInputSchema { get }

    init(context: MCPContext)
    func execute(arguments: [String: Any]) async throws -> Any
}
```

## MCPContext Protocol

```swift
public protocol MCPContext: AnyObject {
    var serverPort: Int { get }
    func executeJavaScript(_ script: String) async throws -> Any?
    func getBotStatus() -> [String: Any]?
    func triggerAutoPlay()
    func getLogs() -> [[String: Any]]
    func clearLogs()
    func log(_ message: String)
}
```

## MCPInputSchema

```swift
public struct MCPInputSchema {
    public static let empty = MCPInputSchema(properties: [:], required: [])
    public let properties: [String: MCPPropertySchema]
    public let required: [String]
}
```

## MCPPropertySchema

```swift
public enum MCPPropertySchema {
    case string(String)
    case integer(String)
    case number(String)
    case boolean(String)
    case object(String)
}
```

## MCPToolError

```swift
public enum MCPToolError: LocalizedError {
    case missingParameter(String)
    case invalidParameter(String, expected: String)
    case executionFailed(String)
    case notAvailable(String)
}
```

## MCPToolRegistry

```swift
public class MCPToolRegistry {
    public static let shared: MCPToolRegistry
    public func register<T: MCPTool>(_ toolType: T.Type, context: MCPContext)
    public func registerBuiltInTools(context: MCPContext)
    public func tool(named: String) -> MCPTool?
    public var registeredToolNames: [String]
    public func toolsList() -> [[String: Any]]
}
```

## MCPHandler

```swift
public class MCPHandler {
    public init(registry: MCPToolRegistry, context: MCPContext)
    public func handleRequest(_ request: [String: Any]) async -> [String: Any]
}
```

## MCPHTTPServer

```swift
public class MCPHTTPServer {
    public init(port: Int, context: MCPContext)
    public func start() async throws
    public func stop()
    public var actualPort: Int
}
```

## Complete Tool Example

```swift
struct CalculatorTool: MCPTool {
    static let name = "calculator"
    static let description = "Perform arithmetic operations"

    static let inputSchema = MCPInputSchema(
        properties: [
            "operation": .string("add, subtract, multiply, divide"),
            "a": .number("First operand"),
            "b": .number("Second operand")
        ],
        required: ["operation", "a", "b"]
    )

    private let context: MCPContext

    init(context: MCPContext) {
        self.context = context
    }

    func execute(arguments: [String: Any]) async throws -> Any {
        guard let op = arguments["operation"] as? String,
              let a = arguments["a"] as? Double,
              let b = arguments["b"] as? Double else {
            throw MCPToolError.missingParameter("operation, a, or b")
        }

        let result: Double
        switch op {
        case "add": result = a + b
        case "subtract": result = a - b
        case "multiply": result = a * b
        case "divide":
            guard b != 0 else { throw MCPToolError.executionFailed("Division by zero") }
            result = a / b
        default:
            throw MCPToolError.invalidParameter("operation", expected: "add/subtract/multiply/divide")
        }

        return ["operation": op, "a": a, "b": b, "result": result]
    }
}
```

## JSON-RPC Protocol

### Initialize
```json
{"jsonrpc": "2.0", "method": "initialize", "params": {"protocolVersion": "2024-11-05"}, "id": 1}
```

### Tools List
```json
{"jsonrpc": "2.0", "method": "tools/list", "id": 2}
```

### Tool Call
```json
{"jsonrpc": "2.0", "method": "tools/call", "params": {"name": "calculator", "arguments": {"operation": "add", "a": 5, "b": 3}}, "id": 3}
```
