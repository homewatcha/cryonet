# Конвейер промежуточного ПО (Middleware)

Промежуточное ПО (Middleware) в `cryonet` оборачивает вызовы обработчиков маршрутов по принципу луковичной архитектуры («onion model»). Каждый компонент middleware может перехватить запрос до передачи в обработчик, изменить контекст, прервать выполнение с возвратом ошибки или модифицировать готовый ответ перед отправкой клиенту.

---

## 1. Сигнатура MiddlewareFn и подключение

Функция промежуточного ПО имеет строго типизированную сигнатуру:

```yukino
type MiddlewareFn = fun(Context, fun(Context) -> Response) -> Response
```

- Первый аргумент — текущий объект `Context`.
- Второй аргумент `next` — замыкание для вызова следующего middleware или целевого обработчика маршрута.

Подключение middleware к маршрутизатору:

```yukino
load cryonet: router, recovery_middleware, logger_middleware, cors, CorsConfig

set r = router()
r.use(recovery_middleware())
r.use(logger_middleware())
```

---

## 2. Встроенные компоненты Middleware

`cryonet` поставляется с набором готовых компонентов для решения типовых задач безопасности и мониторинга:

### 2.1 Перехват паник (`recovery_middleware`)

Защищает сервер от аварийного завершения при неожиданных сбоях в коде обработчиков:
- Перехватывает любые типы `panic` с помощью защищенного блока `sus / catch`.
- Логирует информацию о сбое в терминал.
- Автоматически отдает клиенту ответ со статусом `500 Internal Server Error`, сохраняя работоспособность сервера.

```yukino
r.use(recovery_middleware())
```

### 2.2 Логирование запросов (`logger_middleware`)

Выводит в стандартный вывод информацию о каждом прошедшем HTTP-запросе:
- HTTP-метод и путь запроса.
- Статус ответа.
- Время выполнения запроса в микросекундах.

```yukino
r.use(logger_middleware())
```

### 2.3 Управление доступом (`cors`)

Настраивает политику Cross-Origin Resource Sharing, автоматически обрабатывает предварительные preflight-запросы методом `OPTIONS` и выставляет необходимые CORS-заголовки:

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

Если конфигурация не указана, `cors()` применяет открытые настройки по умолчанию (`*` для всех источников).

### 2.4 Идентификатор запроса (`request_id_middleware`)

Присваивает каждому запросу уникальный идентификатор для сквозной трассировки в логах:
- Если клиент передал заголовок `X-Request-ID`, сохраняет его.
- Если заголовок отсутствует, генерирует уникальный случайный идентификатор на основе времени и криптографических хешей.
- Прокидывает `X-Request-ID` в исходящие заголовки ответа и в локальный контекст `ctx.get_local("request_id")`.

```yukino
r.use(request_id_middleware("X-Request-ID"))
```

### 2.5 Автоматическое сжатие ответов (`compression_middleware`)

Анализирует заголовок клиента `Accept-Encoding` и автоматически сжимает тело текстовых ответов, выставляя соответствующий заголовок `Content-Encoding`:
- Поддерживает форматы `gzip`, `deflate` и `zstd`.
- Не сжимает бинарные форматы, уже имеющие собственное сжатие (изображения, видео, архивы).

```yukino
r.use(compression_middleware())
```

### 2.6 Ограничение размера тела (`body_limit`)

Защищает сервер от атак исчерпания памяти при отправке гигантских тел запросов:
- Проверяет заголовок `Content-Length`.
- Если размер превышает лимит, немедленно прерывает обработку со статусом `413 Payload Too Large`.

```yukino
// Ограничение размера тела в 5 мегабайт (5 * 1024 * 1024)
r.use(body_limit(5242880))
```

### 2.7 Ограничение частоты запросов (`rate_limit_middleware`)

Реализует алгоритм скользящего окна (sliding window) для ограничения количества запросов по IP-адресу клиента:
- Автоматически нормализует адрес через `strip_port`, исключая учет по портам соединений.
- При превышении лимита возвращает статус `429 Too Many Requests` с заголовком `Retry-After`.

```yukino
load cryonet: rate_limit_middleware, RateLimiter

// 100 запросов в минуту на один IP
set limiter = init RateLimiter(100, 60000)
r.use(rate_limit_middleware(limiter))
```

### 2.8 Защита сокета от зависаний (`latency_guard_middleware`, `timeout_middleware`)

Позволяет индивидуально настроить таймауты сокета на уровне конвейера middleware:

```yukino
// Устанавливает дедлайн ожидания в 5000 мс (5 секунд)
r.use(latency_guard_middleware(5000))
// Также доступен синоним timeout_middleware:
r.use(timeout_middleware(5000))
```

---

## 3. Написание собственных Middleware

Создать собственное middleware предельно просто. Рассмотрим типовые практические примеры:

### Пример 1: Проверка авторизации (Auth Guard)

```yukino
fun auth_guard_middleware() -> MiddlewareFn {
    return fun(ctx: Context, next: fun(Context) -> Response) => {
        set token = ctx.bearer_token()
        if token == null || token != "secret-token-value" {
            // Прерываем цепочку и сразу отдаем 401
            return ctx.bad_request("Unauthorized: invalid or missing bearer token")
        }

        // Сохраняем данные пользователя в локальный контекст
        ctx.set_local("user_role", "admin")

        // Передаем управление следующему обработчику
        return next(ctx)
    }
}

r.use(auth_guard_middleware())
```

### Пример 2: Добавление пользовательских заголовков безопасности

```yukino
fun security_headers_middleware() -> MiddlewareFn {
    return fun(ctx: Context, next: fun(Context) -> Response) => {
        // 1. Вызываем обработчик и получаем ответ
        set resp = next(ctx)

        // 2. Добавляем заголовки к готовому ответу
        resp.header("X-Content-Type-Options", "nosniff")
        resp.header("X-Frame-Options", "DENY")
        resp.header("X-XSS-Protection", "1; mode=block")

        return resp
    }
}

r.use(security_headers_middleware())
```

---

[Назад: 2. Контекст, запросы и ответы](2-context-handlers.md) | [Вперед: 4. Real-Time потоки, статические файлы и тестирование](4-realtime-files.md)
