# Context, Request, and Response

The `Context` object is supplied to every route handler and encapsulates the incoming HTTP request, environmental data, thread-safe application state access, and response generation utilities.

---

## 1. Import: load cryonet

```yukino
load cryonet: Context, Request, Response, AppState, SubState, BasicAuth, Cookie, cookie_new, strip_port, response_new, response_text, response_json, response_bytes
```

---

## 2. Inspecting the Incoming Request

### 2.1 Query Parameters

Extract values following the `?` character in the request URL:

```yukino
r.get("/search", fun(ctx) => {
    set q = ctx.query("q") ?? ""
    set page = ctx.query_int("page") ?? 1
    set ascending = ctx.query_bool("asc") ?? true

    return ctx.string("Search: ${q}, page: ${page}")
})
```

- `ctx.query(name)` — returns the string value (`string?`) or `null`.
- `ctx.query_int(name)` — parses the parameter as a 32-bit signed integer (`int32?`).
- `ctx.query_bool(name)` — parses boolean tokens (`true`, `false`, `1`, `0`) (`bool?`).
- `ctx.queries()` — returns all query parameters as an array (`QueryParam[]`).
- `ctx.raw_query()` — returns the raw, unparsed query string (`string?`).

### 2.2 Headers and Authentication

```yukino
r.get("/protected", fun(ctx) => {
    // 1. Extract Bearer token from the Authorization header
    set token = ctx.bearer_auth()
    if token == null {
        return ctx.bad_request("Missing Authorization: Bearer <token> header")
    }

    // 2. Extract Basic authentication credentials (base64 decoded)
    set basic = ctx.basic_auth()
    if basic != null {
        set b = basic ?? BasicAuth { username: "", password: "" }
        println("Authorized user: " + b.username)
    }

    // 3. Inspect custom headers
    set agent = ctx.header("User-Agent") ?? "Unknown"
    set accepts_json = ctx.accepts("application/json")

    return ctx.string("User-Agent: " + agent)
})
```

### 2.3 Client IP Address and `strip_port`

Network sockets deliver peer addresses in `IP:port` format. `cryonet` normalizes addresses by trimming ports to prevent rate limit key proliferation:

```yukino
// Direct client IP without trusting proxies
set ip = ctx.client_ip()

// Client IP with support for X-Forwarded-For and X-Real-IP headers
set real_ip = ctx.client_ip_opt(true)
```

The `strip_port(addr: string) -> string` function handles both IPv4 (`192.168.1.1:8080` -> `192.168.1.1`) and IPv6 bracketed addresses (`[::1]:8080` -> `::1`).

### 2.4 Cookies and HMAC-SHA256 Signed Cookies

`cryonet` supports reading standard cookies as well as cryptographically signed cookies to guard against client-side tampering:

```yukino
static COOKIE_SECRET: string = "secret-super-key-32-bytes-long"

r.get("/session", fun(ctx) => {
    // Plain cookie
    set theme = ctx.cookie("theme") ?? "light"

    // Signed cookie: automatically validates HMAC-SHA256 signature
    // Returns null if the value was altered or the signature is invalid
    set user_id = ctx.signed_cookie("session_id", COOKIE_SECRET)
    if user_id == null {
        return ctx.string("Invalid or expired session")
    }

    return ctx.string("Welcome back, user: " + (user_id ?? ""))
})
```

### 2.5 Request Body, JSON, Forms, and File Uploads (Multipart)

```yukino
// 1. Read body as plain text
set body_str = ctx.body()

// 2. Parse JSON body
set json_val = ctx.json() // returns JsonValue? or null on parse error

// 3. Parse HTML form data (application/x-www-form-urlencoded)
set form = ctx.form()
set username = form.get("username") ?? ""

// 4. Parse multipart/form-data (files and fields)
set mp = ctx.multipart()
set desc = mp.get_field("description") ?? ""
set uploaded_file = mp.get_file("avatar")
if uploaded_file != null {
    set file_part = uploaded_file ?? MultipartPart { name: "", filename: "", content_type: "", data: cast("", "bytes") }
    println("Received file: " + file_part.filename + " (" + cast(file_part.data.len(), "string") + " bytes)")
}
```

---

## 3. Generating HTTP Responses

Route handlers always return a `Response` instance. The `Context` provides ergonomic builder methods:

```yukino
// Plain text (Content-Type: text/plain)
return ctx.string("Operation successful")

// Plain text with specific status code
return ctx.text_response("Resource created", 201)

// Raw JSON string (Content-Type: application/json)
return ctx.json_str("{\"status\":\"ok\"}", 200)

// HTML document (Content-Type: text/html)
return ctx.html("<h1>Hello World!</h1>", 200)

// Binary stream
return ctx.bytes_data(my_bytes, "application/pdf")

// File download with Content-Disposition attachment header
return ctx.attachment("report.pdf", my_pdf_bytes, "application/pdf")

// Redirect response
return ctx.redirect("/login", 302)

// Empty response (204 No Content)
return ctx.empty(204)

// Standard error responses
return ctx.not_found("Resource not found")
return ctx.bad_request("Invalid arguments")
return ctx.server_error("Internal server error")
```

### Fluent Response API

When fine-grained header, cookie, or status control is required, use method chaining on `Response`:

```yukino
set resp = response_text("Hello", 200)
resp.header("X-Custom-Header", "Value")
resp.header("Cache-Control", "no-cache")

// Set plain cookie
resp.set_cookie(cookie_new("visited", "true"))

// Set signed cookie
resp.set_signed_cookie("session_id", "user_123", COOKIE_SECRET, "/", 86400)

return resp
```

---

## 4. State Management and Request-Local Storage

### 4.1 Request-Local State (`Local State`)

Used to pass request-scoped data across middleware and downstream handlers (e.g., authenticated user ID):

```yukino
// In middleware
ctx.set_local("current_user_id", "42")

// In handler
set user_id = ctx.get_local("current_user_id") ?? "anonymous"
```

### 4.2 Application-Wide Shared State (`AppState`)

`AppState` is a thread-safe, mutex-guarded key-value store shared across all concurrent worker threads:

```yukino
set r = router()

// Initialize global state
set state = r.app_state
state.set_string("app_name", "Cryonet Production")
state.set_int("counter", 0)

r.get("/stats", fun(ctx) => {
    set name = ctx.app_string("app_name") ?? ""
    set count = ctx.app_int("counter") ?? 0
    return ctx.string("App: ${name}, requests: ${count}")
})
```

Sub-states allow namespace isolation:

```yukino
set auth_sub = state.sub("auth")
auth_sub.set_string("jwt_issuer", "cryonet.io")

// Stored internally as "auth:jwt_issuer"
```

---

[Back: 1. Routing and Method Handlers](1-routing.md) | [Next: 3. Middleware Pipeline and Built-in Guards](3-middleware.md)
