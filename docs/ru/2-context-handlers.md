# Контекст, запросы и ответы

Объект `Context` передается в каждый обработчик маршрута и инкапсулирует входящий HTTP-запрос, параметры окружения, доступ к потокобезопасному состоянию приложения и методы формирования HTTP-ответов.

---

## 1. Подключение: load cryonet

```yukino
load cryonet: Context, Request, Response, AppState, SubState, BasicAuth, Cookie, cookie_new, strip_port, response_new, response_text, response_json, response_bytes
```

---

## 2. Чтение данных входящего запроса

### 2.1 Параметры строки запроса (Query Params)

Позволяет извлекать параметры после знака `?` в URL:

```yukino
r.get("/search", fun(ctx) => {
    set q = ctx.query("q") ?? ""
    set page = ctx.query_int("page") ?? 1
    set ascending = ctx.query_bool("asc") ?? true

    return ctx.string("Поиск: ${q}, страница: ${page}")
})
```

- `ctx.query(name)` — возвращает строковое значение `string?` либо `null`.
- `ctx.query_int(name)` — разбирает параметр как 32-битное целое число (`int32?`).
- `ctx.query_bool(name)` — разбирает флаги `true`, `false`, `1`, `0` (`bool?`).
- `ctx.queries()` — возвращает полный массив параметров `QueryParam[]`.
- `ctx.raw_query()` — возвращает исходную неразобранную строку запроса (`string?`).

### 2.2 Заголовки (Headers) и аутентификация

```yukino
r.get("/protected", fun(ctx) => {
    // 1. Извлечение Bearer токена из заголовка Authorization
    set token = ctx.bearer_auth()
    if token == null {
        return ctx.bad_request("Отсутствует заголовок Authorization: Bearer <token>")
    }

    // 2. Извлечение Basic аутентификации (логин и пароль в base64)
    set basic = ctx.basic_auth()
    if basic != null {
        set b = basic ?? BasicAuth { username: "", password: "" }
        println("Авторизация: " + b.username)
    }

    // 3. Произвольные заголовки
    set agent = ctx.header("User-Agent") ?? "Unknown"
    set accepts_json = ctx.accepts("application/json")

    return ctx.string("User-Agent: " + agent)
})
```

### 2.3 IP-адрес клиента и функция `strip_port`

По умолчанию сетевой сокет отдает адрес в формате `IP:порт`. В `cryonet` встроен метод нормализации IP-адресов без портов, что критично для работы рейт-лимитеров:

```yukino
// Без доверия прокси (прямой IP клиента)
set ip = ctx.client_ip()

// С поддержкой заголовков X-Forwarded-For и X-Real-IP
set real_ip = ctx.client_ip_opt(true)
```

Функция `strip_port(addr: string) -> string` поддерживает как IPv4 (`192.168.1.1:8080` -> `192.168.1.1`), так и IPv6 (`[::1]:8080` -> `::1`).

### 2.4 Куки и подписанные куки (HMAC-SHA256)

`cryonet` поддерживает чтение стандартных кук, а также криптографически подписанных кук для предотвращения подделки сессий на стороне клиента:

```yukino
static COOKIE_SECRET: string = "secret-super-key-32-bytes-long"

r.get("/session", fun(ctx) => {
    // Обычная кука
    set theme = ctx.cookie("theme") ?? "light"

    // Подписанная кука: автоматически сверяет HMAC-SHA256 подпись
    // Если подпись изменена злоумышленником, возвращается null
    set user_id = ctx.signed_cookie("session_id", COOKIE_SECRET)
    if user_id == null {
        return ctx.string("Сессия недействительна")
    }

    return ctx.string("Добро пожаловать, пользователь: " + (user_id ?? ""))
})
```

### 2.5 Тело запроса, JSON, формы и загрузка файлов (Multipart)

```yukino
// 1. Текстовое тело запроса
set body_str = ctx.body()

// 2. Разбор JSON
set json_val = ctx.json() // возвращает JsonValue? или null при невалидном синтаксисе

// 3. Разбор HTML-форм (application/x-www-form-urlencoded)
set form = ctx.form()
set username = form.get("username") ?? ""

// 4. Разбор multipart/form-data (файлы и поля)
set mp = ctx.multipart()
set desc = mp.get_field("description") ?? ""
set uploaded_file = mp.get_file("avatar")
if uploaded_file != null {
    set file_part = uploaded_file ?? MultipartPart { name: "", filename: "", content_type: "", data: cast("", "bytes") }
    println("Загружен файл: " + file_part.filename + " (" + cast(file_part.data.len(), "string") + " байт)")
}
```

---

## 3. Формирование HTTP-ответов

Обработчик маршрута всегда возвращает объект `Response`. `Context` предоставляет удобные фабричные методы для распространенных сценариев:

```yukino
// Простой текст (Content-Type: text/plain)
return ctx.string("Успешно")

// Текст со статусом
return ctx.text_response("Создано", 201)

// Сырой JSON (Content-Type: application/json)
return ctx.json_str("{\"status\":\"ok\"}", 200)

// HTML разметка (Content-Type: text/html)
return ctx.html("<h1>Привет, мир!</h1>", 200)

// Бинарные данные
return ctx.bytes_data(my_bytes, "application/pdf")

// Скачивание файла с заголовком Content-Disposition
return ctx.attachment("report.pdf", my_pdf_bytes, "application/pdf")

// Перенаправления
return ctx.redirect("/login", 302)

// Пустой ответ (204 No Content)
return ctx.empty(204)

// Стандартные ошибки
return ctx.not_found("Ресурс не найден")
return ctx.bad_request("Неверные аргументы")
return ctx.server_error("Сбой сервера")
```

### Свободный API объекта Response

Если требуется гибкая настройка заголовков, статуса и кук, используется цепочечный вызов методов `Response`:

```yukino
set resp = response_text("Hello", 200)
resp.header("X-Custom-Header", "Value")
resp.header("Cache-Control", "no-cache")

// Установка обычной куки
resp.set_cookie(cookie_new("visited", "true"))

// Установка подписанной куки
resp.set_signed_cookie("session_id", "user_123", COOKIE_SECRET, "/", 86400)

return resp
```

---

## 4. Состояние приложения и локальный контекст запроса

### 4.1 Локальное состояние запроса (`Local State`)

Используется для передачи данных между middleware и обработчиками в рамках одного HTTP-запроса (например, ID пользователя после авторизации):

```yukino
// В middleware
ctx.set_local("current_user_id", "42")

// В обработчике
set user_id = ctx.get_local("current_user_id") ?? "anonymous"
```

### 4.2 Глобальное состояние приложения (`AppState`)

`AppState` — потокобезопасное хранилище `ключ-значение`, защищенное мьютексом, разделяемое между всеми рабочими потоками:

```yukino
set r = router()

// Запись в глобальное состояние при инициализации
set state = r.app_state
state.set_string("app_name", "Cryonet Production")
state.set_int("counter", 0)

r.get("/stats", fun(ctx) => {
    set name = ctx.app_string("app_name") ?? ""
    set count = ctx.app_int("counter") ?? 0
    return ctx.string("Сервис: ${name}, счетчик: ${count}")
})
```

Для логической изоляции модулей поддерживаются подсостояния `SubState`:

```yukino
set auth_sub = state.sub("auth")
auth_sub.set_string("jwt_issuer", "cryonet.io")

// Внутри ключи будут сохранены с префиксом "auth:jwt_issuer"
```

---

[Назад: 1. Маршрутизация и обработчики методов](1-routing.md) | [Вперед: 3. Конвейер промежуточного ПО (Middleware)](3-middleware.md)
