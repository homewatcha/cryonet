# cryonet

A high-performance, ergonomic, and lightweight web framework and HTTP/HTTPS server for the [Yukino](https://github.com/yukitsuri/yukino) programming language.

`cryonet` is engineered for reliability, strict typing, memory safety, and predictable execution with zero hidden runtime overhead. It provides modular routing with path parameters (`:id`) and wildcards (`*path`), a composable middleware pipeline, real-time WebSockets and Server-Sent Events (SSE), response chunking, static file streaming with ETag and RFC-compliant Range requests, native TLS encryption, and an in-memory testing client.

---

## Features

- **Expressive Modular Routing**: Dynamic path segments (`:param`), wildcards (`*path`), scoped routers (`scope`), mounted sub-routers (`mount`), and prefix-based fallbacks.
- **Strictly Typed Context**: Access parameters, query strings, cookies, signed HMAC-SHA256 cookies, headers, body streams, form data, and multipart file uploads through a unified `Context`.
- **Zero-Copy & Buffered Responses**: Fluent response builders for text, JSON, HTML, NDJSON, binary streams, file attachments, redirects, and HTTP partial content (Range).
- **Composable Middleware Pipeline**: Built-in recovery, structured request logging, CORS, unique request IDs, transparent response compression, body size limits, rate limiting, and socket timeouts.
- **Real-Time Streaming**: Native abstractions for WebSockets (`WebSocketSession`), Server-Sent Events (`SseWriter`), and HTTP/1.1 chunked transfer encoding (`StreamWriter`).
- **Static File Serving**: Directory and single-file handlers with automatic MIME detection, ETag caching (304 Not Modified), and full RFC 9110 byte-range slicing (206 Partial Content / 416 Range Not Satisfiable).
- **Integrated TLS (HTTPS)**: Secure transport with socket deadline enforcement prior to TLS handshake to prevent Slowloris attacks.
- **In-Memory Testing**: Full-featured `TestClient` allows end-to-end routing and handler tests without binding real network sockets.
- **Predictable Concurrency**: Single-threaded simplicity in `serve()` and `serve_tls()`, with user-managed worker thread dispatching via `server.accept()`.

---

## Installation

Add `cryonet` to your project manifest under the `[libs.runtime]` section in `yuki.toml`:

```toml
[libs.runtime]
cryonet = { git = "https://github.com/homewatcha/cryonet", tag = "0.1.0" }
```

Or reference a local workspace path during development:

```toml
[libs.runtime]
cryonet = { path = "../offlibs/cryonet" }
```

---

## Quick Example

### 1. Define Routes and Start Server (`src/main.ykn`)

```yukino
load std.io: println
load cryonet: router, get, post, response_text, response_json, serve, logger_middleware, recovery_middleware

fun main() {
    set r = router()

    // 1. Attach global middleware
    r.use(recovery_middleware())
    r.use(logger_middleware())

    // 2. Define endpoints
    r.route("/", get(fun(ctx) => response_text("Welcome to Cryonet!", 200)))

    r.route("/api/users/:id", get(fun(ctx) => {
        set user_id = ctx.param("id") ?? "0"
        return response_json("{\"id\":${user_id},\"name\":\"Yukino User\"}", 200)
    }))

    r.route("/api/echo", post(fun(ctx) => {
        set payload = ctx.body()
        return response_text("Echo: " + payload, 200)
    }))

    // 3. Start server on port 8080 (single-threaded)
    println("Listening on http://127.0.0.1:8080...")
    serve("127.0.0.1:8080", r, 10000, 2097152)
}
```

### 2. Multi-Threaded Server Example

For concurrent request processing across multiple CPU cores, decouple the accept loop from connection handling:

```yukino
load std.threads: spawn
load std.net.tcp: TcpStream
load cryonet: Server, router, get, response_text

fun worker(s: Server, stream: TcpStream) {
    s.handle_connection(stream)
}

fun main() {
    set r = router()
    r.route("/compute", get(fun(ctx) => response_text("Parallel response", 200)))

    set s = init Server("0.0.0.0:8080", r, 10000, 2097152)

    while true {
        set stream_opt = s.accept()
        if stream_opt != null {
            set stream = stream_opt ?? init TcpStream(0)
            spawn fun() => worker(s, stream)
        }
    }
}
```

---

## Documentation

Comprehensive documentation is available in English and Russian:

- **English Documentation**:
  - [0. Overview and Architecture](docs/en/0-overview.md)
  - [1. Routing and Method Handlers](docs/en/1-routing.md)
  - [2. Context, Request, and Response](docs/en/2-context-handlers.md)
  - [3. Middleware Pipeline and Built-in Guards](docs/en/3-middleware.md)
  - [4. Real-Time Streams, Static Files, and Testing](docs/en/4-realtime-files.md)

- **Russian Documentation**:
  - [0. Обзор архитектуры и быстрый старт](docs/ru/0-overview.md)
  - [1. Маршрутизация и обработчики методов](docs/ru/1-routing.md)
  - [2. Контекст, запросы и ответы](docs/ru/2-context-handlers.md)
  - [3. Конвейер промежуточного ПО (Middleware)](docs/ru/3-middleware.md)
  - [4. Real-Time потоки, статические файлы и тестирование](docs/ru/4-realtime-files.md)

---

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

Copyright (c) 2026 homewatcha.
