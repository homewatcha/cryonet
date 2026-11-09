# Library cryonet

The `cryonet` library is a high-performance, lightweight web framework and HTTP/HTTPS server for the Yukino programming language. The framework is engineered for ergonomics, strict static typing, memory safety, and complete operational transparency without hidden runtimes or undisclosed background thread pools.

`cryonet` provides modular routing with path parameters (`:id`) and wildcards (`*path`), a composable middleware pipeline, HTTP/1.1 chunked streaming, Server-Sent Events (SSE), WebSockets with socket timeout controls, high-throughput static file serving with ETag and RFC 9110 Range slicing, integrated native TLS encryption, and an in-memory testing client for fast integration testing without binding real network ports.

---

## 1. Introduction to Web Development in Yukino

A web server forms the foundation of modern network services: it binds to a network port, accepts incoming connections from browsers or HTTP clients, translates raw protocol bytes into structured requests, and transmits the application's response back to the client.

### Anatomy of an HTTP Interaction

Standard HTTP/1.1 communication consists of two distinct phases:
1. **HTTP Request:**
   - **Method and URI:** e.g., `GET /api/users/42?details=true HTTP/1.1`. The verb specifies the desired action (`GET` for retrieval, `POST` for creation, `PUT`/`PATCH` for modification, `DELETE` for removal).
   - **Headers:** key-value metadata pairs (`Content-Type`, `Authorization`, `Cookie`, `Accept-Encoding`).
   - **Body:** optional payload (JSON, form data, multipart files, or raw binary streams).
2. **HTTP Response:**
   - **Status Code:** integer status (`200 OK`, `201 Created`, `400 Bad Request`, `404 Not Found`, `500 Internal Server Error`).
   - **Headers:** instructions and metadata for the client (`Content-Type: application/json`, `Set-Cookie`, `Content-Length`).
   - **Body:** the payload delivered to the client.

### Why cryonet is Superior to Manual Socket Programming

When working directly with raw sockets via `std.net.tcp`, developers must manually:
- Read raw byte buffers, buffer incomplete packets, and parse HTTP headers.
- Build cumbersome `if / else` branching trees to match paths and extract parameters.
- Handle encoding conversions, cookies, form bodies, multipart uploads, and JSON serialization.
- Configure socket deadlines and safeguard against unhandled panics so an individual faulty request does not crash the entire process.

The `cryonet` library automates repetitive protocol mechanics while granting developers complete authority over memory allocation, socket lifecycles, and concurrency models.

---

## 2. Installation: load cryonet

Add `cryonet` to your project manifest under the `[libs.runtime]` section in `yuki.toml`:

```toml
[libs.runtime]
cryonet = { git = "https://github.com/homewatcha/cryonet", tag = "0.1.1", path = "cryonet" }
```

Or reference a local workspace path during development:

```toml
[libs.runtime]
cryonet = { path = "../offlibs/cryonet" }
```

### Import in Application Code

In your entry point (`src/main.ykn`):

```yukino
load cryonet: *
```

Exported entities:
- **Server:** `Server`, `serve`, `serve_tls`, `request_from_http`.
- **Routing:** `Router`, `router`, `Route`, `clean_path`, `split_path`, `path_matches_prefix`.
- **Method Handlers:** `MethodRouter`, `method_router`, `HandlerFn`, `get`, `post`, `put`, `delete`, `patch`, `head`, `options`, `any`.
- **Context & State:** `Context`, `ContextError`, `AppState`, `SubState`, `StateEntry`, `strip_port`.
- **Request & Response:** `Request`, `Response`, `RouteParam`, `response_new`, `response_text`, `response_bytes`, `response_json`, `response_json_str`, `response_html`, `response_redirect`, `response_empty`, `response_not_found`, `response_bad_request`, `response_payload_too_large`, `response_server_error`, `response_hijacked`, `response_ndjson`.
- **Headers & Authentication:** `Cookie`, `cookie_new`, `cookie_format`, `cookie_parse`, `BasicAuth`, `HttpRange`, `parse_basic_auth`, `parse_bearer_token`, `parse_range_header`.
- **Real-Time Streams:** `ConnectionStream`, `stream_from_tcp`, `stream_from_tls`, `stream_from_tls_tcp`, `stream_noop`, `WebSocketSession`, `SseWriter`, `StreamWriter`, `WsMessage`, `WsFrame`, `SseEvent`.
- **Static Files:** `ServeDir`, `serve_dir`, `ServeFile`, `serve_file`, `serve_file_response`.
- **Forms & Multipart:** `Form`, `FormEntry`, `parse_form`, `url_decode`, `Multipart`, `MultipartPart`, `parse_multipart`, `parse_multipart_bytes`.
- **Middleware:** `logger_middleware`, `recovery_middleware`, `cors`, `request_id_middleware`, `compression_middleware`, `body_limit`, `rate_limit_middleware`, `latency_guard_middleware`, `timeout_middleware`.
- **Testing:** `TestClient`, `test_client`.

---

## 3. Architecture and Concurrency Philosophy

A foundational architectural principle of `cryonet` is that **multi-threading is managed explicitly by the user, not hidden within the library**.

### Why `serve()` and `serve_tls()` are Single-Threaded

The convenience runner functions `serve()` and `serve_tls()`, as well as `server.listen()`, operate **strictly on a single operating system thread** (single-threaded).

In this mode, the server executes a sequential `accept()` loop on the listening socket, completely parses the incoming request, dispatches it to the router, transmits the response, and only then transitions to the next client connection.

Advantages of single-threaded mode:
- Absolute determinism and simple debugging.
- Minimal memory consumption (no thread stack allocations).
- Total absence of data races when accessing shared data structures.
- Ideal for command-line tools, microservices with bounded loads, and automated testing suites.

### How the User Manages Multi-Threading

When an application demands concurrent request processing across multiple CPU cores, **the user determines the multi-threading strategy**.

To support this, `cryonet` exposes low-level building blocks free of hidden runtimes:
1. `server.accept() -> TcpStream?` — accepts raw incoming socket connections.
2. `server.handle_connection(stream: TcpStream)` — handles an HTTP connection completely in the calling thread.
3. `server.handle_tls_connection(stream: TcpStream)` — handles an HTTPS/TLS connection completely in the calling thread.

#### Pattern: Dedicated Worker Thread per Connection via `spawn`

```yukino
load std.threads: spawn
load std.net.tcp: TcpStream
load std.io: println
load cryonet: Server, router, get, response_text

fun handle_worker(s: Server, stream: TcpStream) {
    // Request processing executes entirely within the dedicated worker thread
    s.handle_connection(stream)
}

fun main() {
    set r = router()
    r.route("/compute", get(fun(ctx) => response_text("Parallel result", 200)))

    set s = init Server("0.0.0.0:8080", r, 10000, 2097152)
    println("HTTP Server listening on 0.0.0.0:8080 with worker threads...")

    // Accept loop is under complete developer control
    while true {
        set stream_opt = s.accept()
        if stream_opt != null {
            set stream = stream_opt ?? init TcpStream(0)
            // The user decides how to schedule the task (spawn, thread pool, or task queue)
            spawn fun() => handle_worker(s, stream)
        }
    }
}
```

#### Pattern: Multi-Threaded TLS (HTTPS) Server

For encrypted transport, `server.handle_tls_connection(stream: TcpStream)` takes a **raw unencrypted `TcpStream`**:

```yukino
load std.threads: spawn
load std.net.tcp: TcpStream
load std.io: println
load cryonet: Server, router, get, response_text

fun handle_tls_worker(s: Server, stream: TcpStream) {
    // The cryptographic TLS handshake and request parsing execute inside the worker thread
    s.handle_tls_connection(stream)
}

fun main() {
    set r = router()
    r.route("/secure", get(fun(ctx) => response_text("Encrypted response", 200)))

    set s = init Server("0.0.0.0:8443", r, 10000, 2097152)
    s.enable_tls("cert.pem", "key.pem")

    println("HTTPS Server listening on 0.0.0.0:8443...")

    while true {
        set stream_opt = s.accept()
        if stream_opt != null {
            set stream = stream_opt ?? init TcpStream(0)
            spawn fun() => handle_tls_worker(s, stream)
        }
    }
}
```

> [!IMPORTANT]
> Passing a raw `TcpStream` into `handle_tls_connection` guarantees that the computationally intensive TLS handshake (`tls_accept`) runs **inside the user-managed worker thread**, outside the accept loop. A stalled client or slow handshake cannot block incoming connections for other users.

---

## 4. Quick Example

### Step 1. Define Router and Handlers

In your entry point (`src/main.ykn`):

```yukino
load std.io: println
load cryonet: router, get, post, response_text, response_json, serve, logger_middleware, recovery_middleware

fun main() {
    set r = router()

    // Attach middleware
    r.use(recovery_middleware())
    r.use(logger_middleware())

    // Define routes
    r.route("/", get(fun(ctx) => response_text("Cryonet Server is running", 200)))

    r.route("/api/users/:id", get(fun(ctx) => {
        set user_id = ctx.param("id") ?? "0"
        return response_json("{\"id\":${user_id},\"name\":\"Yukino Developer\"}", 200)
    }))

    r.route("/api/echo", post(fun(ctx) => {
        set payload = ctx.body()
        return response_text("Echo: " + payload, 200)
    }))

    println("Starting server on http://127.0.0.1:8080...")
    serve("127.0.0.1:8080", r, 10000, 2097152)
}
```

### Step 2. Run and Verify with curl

```sh
# Start server
yukino run

# Inspect root endpoint
curl -i http://127.0.0.1:8080/
# HTTP/1.1 200 OK
# Content-Type: text/plain; charset=utf-8
# Content-Length: 26
#
# Cryonet Server is running

# Test dynamic path parameter
curl -i http://127.0.0.1:8080/api/users/42
# HTTP/1.1 200 OK
# Content-Type: application/json; charset=utf-8
# Content-Length: 38
#
# {"id":42,"name":"Yukino Developer"}

# Test POST request body
curl -i -X POST -d "Hello Cryonet" http://127.0.0.1:8080/api/echo
# HTTP/1.1 200 OK
# Content-Type: text/plain; charset=utf-8
# Content-Length: 19
#
# Echo: Hello Cryonet
```

---

## 5. Documentation Sections

- [1. Routing and Method Handlers](1-routing.md) — Comprehensive guide to the route tree: dynamic parameters (`:id`), wildcards (`*path`), priority scoring, mounting sub-routers (`mount`), route scoping (`scope`), automatic `405 Method Not Allowed` responses, and prefix-specific fallbacks.
- [2. Context, Request, and Response](2-context-handlers.md) — Reference for the `Context` API: reading query parameters, headers, cookies, signed HMAC-SHA256 cookies, form-urlencoded and multipart uploads, response builder factories, IP normalization via `strip_port`, and shared application state (`AppState`, `SubState`).
- [3. Middleware Pipeline and Built-in Guards](3-middleware.md) — The onion execution model: panic recovery (`recovery_middleware`), logging (`logger_middleware`), CORS (`cors`), request tracing (`request_id_middleware`), response compression (`compression_middleware`), body size limits (`body_limit`), sliding-window rate limiting (`rate_limit_middleware`), and custom middleware implementation.
- [4. Real-Time Streams, Static Files, and Testing](4-realtime-files.md) — Interactive protocols and testing: `ConnectionStream` abstraction, bi-directional WebSockets (`WebSocketSession`), Server-Sent Events (`SseWriter`), chunked transfer encoding (`StreamWriter`), static asset delivery (`ServeDir`, `ServeFile`, ETag 304, RFC 9110 Range 206/416), and in-memory test execution via `TestClient`.

---

[Next: 1. Routing and Method Handlers](1-routing.md)
