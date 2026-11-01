# Routing and Method Handlers

The routing system in `cryonet` provides declarative route tree construction, priority-based path matching, dynamic parameter extraction, and modular route grouping.

---

## 1. Import: load cryonet

```yukino
load cryonet: Router, router, Route, MethodRouter, method_router, HandlerFn, get, post, put, delete, patch, head, options, any, clean_path, split_path, path_matches_prefix
```

---

## 2. Router Architecture

The `Router` class is responsible for registering routes and dispatching incoming HTTP requests to their corresponding handlers.

Instantiate a router using the factory function:

```yukino
set r = router()
```

### 2.1 HTTP Method Handlers

Every route is bound to one or more method handlers through a `MethodRouter`. Convenient builder functions are available for standard HTTP verbs:

```yukino
r.route("/users", get(fun(ctx) => ctx.string("User list")))
r.route("/users", post(fun(ctx) => ctx.string("Create user")))
r.route("/users", put(fun(ctx) => ctx.string("Full update")))
r.route("/users", patch(fun(ctx) => ctx.string("Partial update")))
r.route("/users", delete(fun(ctx) => ctx.string("Delete user")))
r.route("/users", head(fun(ctx) => ctx.empty(200)))
r.route("/users", options(fun(ctx) => ctx.empty(204)))
```

To handle all HTTP verbs simultaneously on a single path, use the `any` handler:

```yukino
r.route("/webhook", any(fun(ctx) => {
    println("Request received with method: " + ctx.method())
    return ctx.empty(200)
}))
```

Shorthand helper methods are also provided directly on the `Router` instance:

```yukino
r.get("/items", fun(ctx) => ctx.string("All items"))
r.post("/items", fun(ctx) => ctx.string("Create item"))
r.put("/items/:id", fun(ctx) => ctx.string("Update item"))
r.delete("/items/:id", fun(ctx) => ctx.string("Delete item"))
r.patch("/items/:id", fun(ctx) => ctx.string("Patch item"))
```

### 2.2 Automatic 405 Method Not Allowed Handling

When an incoming request matches a registered path pattern but does not match any configured HTTP verb, `cryonet` automatically responds with `405 Method Not Allowed` and supplies an `Allow` header listing all permitted methods:

```http
HTTP/1.1 405 Method Not Allowed
Allow: GET, POST
Content-Type: text/plain; charset=utf-8
```

---

## 3. Dynamic Path Parameters and Wildcards

`cryonet` supports two categories of dynamic path matching:

### 3.1 Named Route Parameters (`:param`)

Segments prefixed with a colon match a single path segment between slashes:

```yukino
r.get("/users/:user_id/posts/:post_id", fun(ctx) => {
    set user_id = ctx.param("user_id") ?? ""
    set post_id = ctx.param_i64("post_id") ?? 0
    return ctx.string("User ${user_id}, post ${post_id}")
})
```

Helper methods provide type-safe numeric parsing:
- `ctx.param_int(name)` — parses the parameter as an `int32?` or returns `null` if absent or not an integer.
- `ctx.param_i64(name)` — parses the parameter as an `int64?` or returns `null`.

### 3.2 Catch-All Wildcards (`*name`)

Segments prefixed with an asterisk match all remaining segments greedily, including subsequent slashes:

```yukino
r.get("/assets/*filepath", fun(ctx) => {
    set path = ctx.param("filepath") ?? ""
    return ctx.string("Requested asset: " + path)
})
```

A request to `GET /assets/css/themes/dark.css` resolves `filepath` to `css/themes/dark.css`.

### 3.3 Route Priority Scoring

Routes are evaluated based on the specificity score of each segment:
1. **Static segment** (e.g. `/users/new`) carries highest priority (score `10000`).
2. **Named parameter** (e.g. `/users/:id`) carries intermediate priority (score `100`).
3. **Wildcard segment** (e.g. `/users/*rest`) carries lowest priority (score `1`).

This guarantees that a request to `/users/new` is never shadowed by `/users/:id`, regardless of definition order.

---

## 4. Route Grouping and Nesting

### 4.1 Sub-Router Mounting (`mount`)

Mount an independent sub-router under a specific URL prefix:

```yukino
set api_router = router()
api_router.get("/status", fun(ctx) => ctx.string("OK"))
api_router.get("/version", fun(ctx) => ctx.string("1.0.0"))

set app_router = router()
app_router.mount("/api/v1", api_router)
```

The routes become accessible at `/api/v1/status` and `/api/v1/version`.

### 4.2 Route Scoping (`scope`)

Define a logical group of routes under a common prefix using a closure:

```yukino
r.scope("/admin", fun(admin) => {
    admin.get("/dashboard", fun(ctx) => ctx.string("Dashboard"))
    admin.get("/settings", fun(ctx) => ctx.string("Settings"))
})
```

---

## 5. Fallback Handlers (404 Not Found)

### 5.1 Global Fallback

Executed when no registered route matches the request URI:

```yukino
r.fallback(fun(ctx) => {
    return ctx.not_found("Resource not found. Please verify the URL.")
})
```

### 5.2 Prefix-Specific Fallback (`fallback_for`)

Configure specialized 404 responses for distinct subsystems (such as JSON for API endpoints and HTML for web pages):

```yukino
r.fallback_for("/api", fun(ctx) => {
    return ctx.json_str("{\"error\":\"endpoint not found\"}", 404)
})
```

---

## 6. Path Utility Functions

The module exports low-level path manipulation functions:

- `clean_path(path: string) -> string` — removes query parameters, strips redundant trailing slashes, and ensures a leading `/`.
- `split_path(path: string) -> string[]` — splits a path into non-empty segments.
- `path_matches_prefix(path: string, prefix: string) -> bool` — verifies whether a path matches a given prefix respecting segment boundaries.

---

[Back: 0. Overview and Quick Start](0-overview.md) | [Next: 2. Context, Request, and Response](2-context-handlers.md)
