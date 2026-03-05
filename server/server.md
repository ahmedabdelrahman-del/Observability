The transport layer in internal/mcp/transport/stdio/stdio.go decodes JSON-RPC and calls HandleRequest on the server.
The main router is in internal/mcp/server/server.go. It first validates JSONRPC = 2.0 and method is present, otherwise returns InvalidRequest.
Then it routes by method using a switch:
initialize → returns protocol version, capabilities, and server info
notifications/initialized → returns empty result
tools/list → returns all registered tools from toolRegistry
tools/call → executes one tool by name
default → MethodNotFound