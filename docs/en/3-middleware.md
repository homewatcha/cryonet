# Middleware Pipeline and Built-in Guards

Middleware in `cryonet` wraps route handlers using the classic onion execution model. Each middleware component can intercept requests before they reach the route handler, inspect and modify the `Context`, terminate execution early with an error response, or alter the outgoing `Response` before it is transmitted to the client.

---

## 1. Signature and Registration

Middleware functions follow a strictly typed signature:

```yukino
type MiddlewareFn = fun(Context, fun(Context) -> Response) -> Response
```

- The first parameter is the active `Context`.
- The second parameter `next` is a closure that invokes the next middleware in the pipeline or the matched route handler.

Register middleware onto the router:

```yukino
load cryonet: router, recovery_middleware, logger_middleware, cors, CorsConfig

set r = router()
r.use(recovery_middleware())
r.use(logger_middleware())
```

---

## 2. Built-in Middleware Components

`cryonet` bundles production-grade middleware for common security, reliability, and monitoring tasks:

### 2.1 Panic Recovery (`recovery_middleware`)

Shields the server process from crashing when unexpected panics occur inside route handlers:
- Catches panics using a guarded `sus / catch` block.
- Logs diagnostics to standard error.
- Responds with `500 Internal Server Error`, keeping the server socket alive and healthy.

```yukino
r.use(recovery_middleware())
```

### 2.2 Request Logger (`logger_middleware`)

Outputs structured diagnostic logs to standard output for every HTTP interaction:
- HTTP method and path.
- HTTP status code.
- Request processing latency in microseconds.

```yukino
r.use(logger_middleware())
```

### 2.3 Cross-Origin Resource Sharing (`cors`)

Configures CORS policies, handles preflight `OPTIONS` requests automatically, and injects the necessary CORS headers:

```yukino
set cors_cfg = CorsConfig {
    allow_origins: ["https://example.com", "https://app.example.com"],
    allow_methods: ["GET", "POST", "PUT", "DELETE", "OPTIONS"],
    allow_headers: ["Content-Type", "Authorization", "X-Requested-With"],
    allow_credentials: true,
    max_age: 86400,
    expose_headers: ["X-Request-ID"]
}

r.use(cors(cors_cfg))
```

Calling `cors()` without arguments applies permissive default settings (`*` wildcard).

### 2.4 Request Tracing (`request_id_middleware`)

Injects a unique identifier into every request for distributed tracing across system logs:
- Retains existing incoming `X-Request-ID` headers if supplied.
- Generates a unique ID if the header is absent.
- Adds `X-Request-ID` to outgoing response headers and to the local context (`ctx.get_local("request_id")`).

```yukino
r.use(request_id_middleware("X-Request-ID"))
```

### 2.5 Response Compression (`compression_middleware`)

Inspects the client's `Accept-Encoding` header and compresses eligible text-based responses:
- Supports `gzip`, `deflate`, and `zstd` encodings.
- Injects the appropriate `Content-Encoding` header.
- Automatically skips pre-compressed binary media (images, videos, audio, zip archives).

```yukino
r.use(compression_middleware())
```

### 2.6 Body Size Guard (`body_limit`)

Protects memory from denial-of-service attempts sending unbounded payload sizes:
- Evaluates `Content-Length`.
- Aborts processing immediately with `413 Payload Too Large` if the threshold is exceeded.

```yukino
// Limit payload size to 5 megabytes (5 * 1024 * 1024)
r.use(body_limit(5242880))
```

### 2.7 IP Rate Limiting (`rate_limit_middleware`)

Applies a sliding-window algorithm to throttle incoming requests by client IP address:
- Normalizes addresses via `strip_port`, ensuring rate counters track distinct IPs rather than ephemeral socket ports.
- Emits `429 Too Many Requests` with a `Retry-After` header when limits are reached.

```yukino
load cryonet: rate_limit_middleware, RateLimiter

// 100 requests per minute per IP
set limiter = init RateLimiter(100, 60000)
r.use(rate_limit_middleware(limiter))
```

### 2.8 Latency Guard (`latency_guard_middleware`, `timeout_middleware`)

Configures socket read/write timeouts at the middleware stage:

```yukino
// Set 5000 ms (5 second) socket timeout
r.use(latency_guard_middleware(5000))
// Also available under the ergonomic alias timeout_middleware:
r.use(timeout_middleware(5000))
```

---

## 3. Authoring Custom Middleware

Writing custom middleware is straightforward. Consider the following practical examples:

### Example 1: Authentication Guard (Auth Guard)

```yukino
fun auth_guard_middleware() -> MiddlewareFn {
    return fun(ctx: Context, next: fun(Context) -> Response) => {
        set token = ctx.bearer_token()
        if token == null || token != "secret-token-value" {
            // Abort pipeline and respond with 401
            return ctx.bad_request("Unauthorized: invalid or missing bearer token")
        }

        // Store identity in request-local storage
        ctx.set_local("user_role", "admin")

        // Proceed to downstream handler
        return next(ctx)
    }
}

r.use(auth_guard_middleware())
```

### Example 2: Security Response Headers

```yukino
fun security_headers_middleware() -> MiddlewareFn {
    return fun(ctx: Context, next: fun(Context) -> Response) => {
        // 1. Invoke handler pipeline
        set resp = next(ctx)

        // 2. Append security headers to the resulting response
        resp.header("X-Content-Type-Options", "nosniff")
        resp.header("X-Frame-Options", "DENY")
        resp.header("X-XSS-Protection", "1; mode=block")

        return resp
    }
}

r.use(security_headers_middleware())
```

---

[Back: 2. Context, Request, and Response](2-context-handlers.md) | [Next: 4. Real-Time Streams, Static Files, and Testing](4-realtime-files.md)
