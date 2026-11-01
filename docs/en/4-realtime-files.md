# Real-Time Streams, Static Files, and Testing

The `cryonet` library provides first-class support for bi-directional real-time protocols, chunked response streaming, static file delivery, and in-memory test harnesses.

---

## 1. Import: load cryonet

```yukino
load cryonet: ConnectionStream, WebSocketSession, SseWriter, SseEvent, StreamWriter, sse_event_new, ServeDir, serve_dir, ServeFile, serve_file, serve_file_response, TestClient, test_client
```

---

## 2. Real-Time Protocols: WebSockets, SSE, and Chunked Streaming

At the core of all real-time communication in `cryonet` is the `ConnectionStream` abstraction. It unifies operations across plain TCP and encrypted TLS streams. The `set_timeouts` method modifies the underlying network socket even on TLS streams, and `close()` is protected by an `is_closed` flag to guarantee idempotency.

### 2.1 WebSockets (RFC 6455)

`cryonet` implements native WebSocket server communication:
- Automated negotiation of `Upgrade: websocket` and `Connection: Upgrade` headers.
- Computation of `Sec-WebSocket-Accept` using SHA-1 and Base64 (GUID: `258EAFA5-E914-47DA-95CA-C5AB0DC85B11`).
- Support for text (`0x1`) and binary (`0x2`) frames, plus fragmented frame reassembly.
- Automatic handling of control frames: `Ping` (`0x9`), `Pong` (`0xA`), and `Close` (`0x8`).
- Socket inactivity deadlines.

#### Declarative Handler via `ctx.websocket`

```yukino
r.get("/ws", fun(ctx) => {
    return ctx.websocket_with_timeout(fun(ws: WebSocketSession) => {
        println("WebSocket client connected")

        // Transmit welcome message
        ws.send_text("Welcome to Cryonet WebSocket!")

        // Message receive loop
        while true {
            set msg_opt = ws.recv()
            if msg_opt == null {
                println("Client disconnected")
                break
            }
            set msg = msg_opt ?? WsMessage { opcode: 0, text: "", data: cast("", "bytes"), is_binary: false }

            if !msg.is_binary {
                println("Received text: " + msg.text)
                // Echo message back
                ws.send_text("Echo: " + msg.text)
            } else {
                println("Received binary payload: " + cast(msg.data.len(), "string") + " bytes")
                ws.send_binary(msg.data)
            }
        }
    }, 30000) // 30-second socket inactivity deadline
})
```

#### Manual Connection Hijacking via `ctx.upgrade_websocket()`

```yukino
r.get("/ws/manual", fun(ctx) => {
    set ws_opt = ctx.upgrade_websocket_with_timeout(60000)
    if ws_opt == null {
        return ctx.bad_request("Expected WebSocket upgrade handshake")
    }

    set ws = ws_opt ?? init WebSocketSession(stream_noop(), 0)
    ws.send_text("Session established")
    ws.close()

    return response_hijacked()
})
```

---

### 2.2 Server-Sent Events (SSE)

Server-Sent Events provide unidirectional real-time event streaming from the server to client browsers over an open HTTP connection (`Content-Type: text/event-stream`):

```yukino
load std.threads: sleep

r.get("/events", fun(ctx) => {
    return ctx.sse(fun(sse: SseWriter) => {
        // Send initial event
        sse.send_data("System initialized")

        // Send structured event with metadata
        set event = sse_event_new("Temperature: 24.5C", "sensor_reading", "msg_001", 3000)
        sse.send(event)

        // Stream periodic ticks
        for i in 1..5 {
            sleep(1000u64)
            sse.send_event("tick", "Second: " + cast(i, "string"))
        }

        sse.close()
    })
})
```

---

### 2.3 Chunked Transfer Encoding (Streaming)

Deliver unbounded or progressively generated payloads without buffering everything in memory (`Transfer-Encoding: chunked`):

```yukino
load std.threads: sleep

r.get("/stream", fun(ctx) => {
    return ctx.chunked("text/plain; charset=utf-8", fun(stream: StreamWriter) => {
        stream.write_chunk("Starting data stream...\n")
        sleep(500u64)

        stream.write_chunk("Chunk 1\n")
        sleep(500u64)

        stream.write_chunk("Chunk 2\n")
        stream.end_stream()
        stream.close()
    })
})
```

---

## 3. Static File Serving

The `static_files` module delivers file assets directly from disk according to modern HTTP caching and range delivery standards:

### 3.1 Module Capabilities

1. **MIME Detection:** automatically resolves content types for HTML, CSS, JS, JSON, WASM, PNG, JPEG, SVG, MP4, PDF, fonts, and binary assets.
2. **ETag Caching & 304 Not Modified:** computes distinct entity tags based on file size and modification timestamps. When `If-None-Match` matches the active ETag, the server returns `304 Not Modified` immediately without reading file contents from disk.
3. **RFC 9110 Byte-Range Slicing:**
   - Valid, satisfiable ranges (`Range: bytes=0-499`) return `206 Partial Content` with `Content-Range`.
   - Unsatisfiable ranges (`Range: bytes=5000-6000` for a 100-byte file) return `416 Range Not Satisfiable`.
   - Malformed or unreadable Range headers are safely ignored, returning full file content with `200 OK` as mandated by RFC 9110.
4. **Path Traversal Protection:** prevents directory climbing attacks attempting to escape the configured root via `..` segments.

### 3.2 Directory Serving (`serve_dir`) and File Handlers (`serve_file`)

```yukino
load cryonet: router, serve_dir, serve_file

set r = router()

// 1. Serve an entire directory tree under /public/*
// ./static/css/style.css is accessible at /public/css/style.css
r.route("/public/*filepath", serve_dir("./static"))

// 2. Serve a single file (such as a landing page)
r.route("/", serve_file("./static/index.html"))

// 3. Shorthand Router helpers
r.static_dir("/assets", "./public/assets")
r.static_file("/favicon.ico", "./public/favicon.ico")
```

---

## 4. In-Memory Testing without Sockets (`TestClient`)

The `TestClient` class enables fast integration and end-to-end tests without opening TCP listening sockets:

```yukino
load std.io: println
load cryonet: router, get, post, response_text, response_json, test_client

fun main() {
    // 1. Initialize router and endpoints
    set r = router()
    r.get("/api/user/:id", fun(ctx) => {
        set id = ctx.param("id") ?? "0"
        return response_json("{\"id\":${id},\"active\":true}", 200)
    })

    r.post("/api/echo", fun(ctx) => {
        return response_text("Echo: " + ctx.body(), 200)
    })

    // 2. Create in-memory test client
    set client = test_client(r)

    // 3. Perform in-memory GET request
    set resp_get = client.get("/api/user/42")
    assert(resp_get.status == 200)
    assert(resp_get.text() == "{\"id\":42,\"active\":true}")
    println("GET test passed")

    // 4. Perform in-memory POST request
    set resp_post = client.post("/api/echo", "Testing message")
    assert(resp_post.status == 200)
    assert(resp_post.text() == "Echo: Testing message")
    println("POST test passed")
}
```

---

[Back: 3. Middleware Pipeline and Built-in Guards](3-middleware.md) | [Top: 0. Overview and Quick Start](0-overview.md)
