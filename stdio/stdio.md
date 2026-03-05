Entry is Serve(ctx, r, w, h) in internal/mcp/transport/stdio/stdio.go.
It creates a streaming JSON decoder/encoder: json.NewDecoder(r) and json.NewEncoder(w) (stdio.go).
The loop keeps reading one JSON-RPC object at a time from stdin via dec.Decode(&req) (stdio.go).
io.EOF means input closed → clean shutdown (stdio.go).
Invalid JSON returns a JSON-RPC parse error on stdout and continues (stdio.go).
If req.ID == nil, it is treated as a notification (no response written) (stdio.go).
Otherwise, it calls handler routing and writes the response JSON to stdout (stdio.go).