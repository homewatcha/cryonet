# Библиотека cryonet

Библиотека `cryonet` — высокопроизводительный, легковесный веб-фреймворк и HTTP/HTTPS сервер для языка Yukino. Фреймворк спроектирован с упором на эргономику, строгую статическую типизацию, безопасность памяти и полную прозрачность исполнения без скрытых рантаймов и скрытых фоновых потоков.

`cryonet` предоставляет модульную маршрутизацию с поддержкой параметров путей (`:id`) и подстановок (`*path`), конвейер промежуточного ПО (middleware), потоковую передачу данных (chunked transfer), Server-Sent Events (SSE), WebSockets с контролем таймаутов, эффективную раздачу статических файлов с поддержкой ETag и RFC 9110 Range, встроенную поддержку TLS, а также тестовый клиент для модульного и интеграционного тестирования маршрутов в оперативной памяти.

---

## 1. Введение в веб-разработку на Yukino

Веб-сервер — это основа любого сетевого сервиса: он слушает сетевой порт, принимает входящие соединения от браузеров или API-клиентов, преобразует байты сетевого протокола HTTP в понятные структуры запроса и отправляет сформированный приложением ответ.

### Анатомия взаимодействия клиент-сервер

Типовой обмен сообщениями по протоколу HTTP/1.1 состоит из двух фаз:
1. **HTTP-запрос (Request):**
   - **Метод и URI:** например, `GET /api/users/42?details=true HTTP/1.1`. Метод определяет желаемое действие (`GET` — чтение, `POST` — создание, `PUT`/`PATCH` — изменение, `DELETE` — удаление).
   - **Заголовки (Headers):** пары «ключ-значение» с метаданными (`Content-Type`, `Authorization`, `Cookie`, `Accept-Encoding`).
   - **Тело запроса (Body):** полезная нагрузка (JSON, бинарные данные, форма или файлы).
2. **HTTP-ответ (Response):**
   - **Код состояния (Status Code):** числовой статус (`200 OK`, `201 Created`, `400 Bad Request`, `404 Not Found`, `500 Internal Server Error`).
   - **Заголовки ответа:** служебные инструкции для клиента (`Content-Type: application/json`, `Set-Cookie`, `Content-Length`).
   - **Тело ответа:** данные, возвращаемые клиенту.

### Почему веб-фреймворк cryonet лучше ручной работы с сокетами

При низкоуровневом программировании через `std.net.tcp` разработчик вынужден вручную:
- Читать сырые байты из сокета, склеивать фрагменты пакетов и парсить заголовки HTTP.
- Писать сложные деревья `if / else` для поиска нужного пути и извлечения идентификаторов из URL.
- Следить за кодировками, парсить Cookie, Form-urlencoded, Multipart и JSON.
- Вручную настраивать защиту от зависаний сокета и перехватывать паники в коде, чтобы одна ошибка не обрушила процесс всего сервера.

Библиотека `cryonet` автоматизирует всю рутину сетевого протокола, сохраняя при этом стопроцентный контроль разработчика над выделением памяти, потоками и поведением сокетов.

---

## 2. Подключение: load cryonet

Чтобы подключить библиотеку `cryonet` к проекту, добавьте зависимость в секцию `[libs.runtime]` файла манифеста `yuki.toml`:

```toml
[libs.runtime]
cryonet = { git = "https://github.com/homewatcha/cryonet", tag = "0.1.1", path = "cryonet" }
```

Либо укажите локальный путь при разработке в монорепозитории:

```toml
[libs.runtime]
cryonet = { path = "../offlibs/cryonet" }
```

### Импорт в коде программы

В файле точки входа (`src/main.ykn`):

```yukino
load cryonet: *
```

Доступные сущности:
- **Сервер:** `Server`, `serve`, `serve_tls`, `request_from_http`.
- **Маршрутизация:** `Router`, `router`, `Route`, `clean_path`, `split_path`, `path_matches_prefix`.
- **Обработчики методов:** `MethodRouter`, `method_router`, `HandlerFn`, `get`, `post`, `put`, `delete`, `patch`, `head`, `options`, `any`.
- **Контекст и состояние:** `Context`, `ContextError`, `AppState`, `SubState`, `StateEntry`, `strip_port`.
- **Запрос и ответ:** `Request`, `Response`, `RouteParam`, `response_new`, `response_text`, `response_bytes`, `response_json`, `response_json_str`, `response_html`, `response_redirect`, `response_empty`, `response_not_found`, `response_bad_request`, `response_payload_too_large`, `response_server_error`, `response_hijacked`, `response_ndjson`.
- **Заголовки и аутентификация:** `Cookie`, `cookie_new`, `cookie_format`, `cookie_parse`, `BasicAuth`, `HttpRange`, `parse_basic_auth`, `parse_bearer_token`, `parse_range_header`.
- **Потоки real-time:** `ConnectionStream`, `stream_from_tcp`, `stream_from_tls`, `stream_from_tls_tcp`, `stream_noop`, `WebSocketSession`, `SseWriter`, `StreamWriter`, `WsMessage`, `WsFrame`, `SseEvent`.
- **Статические файлы:** `ServeDir`, `serve_dir`, `ServeFile`, `serve_file`, `serve_file_response`.
- **Формы и multipart:** `Form`, `FormEntry`, `parse_form`, `url_decode`, `Multipart`, `MultipartPart`, `parse_multipart`, `parse_multipart_bytes`.
- **Middleware:** `logger_middleware`, `recovery_middleware`, `cors`, `request_id_middleware`, `compression_middleware`, `body_limit`, `rate_limit_middleware`, `latency_guard_middleware`, `timeout_middleware`.
- **Тестирование:** `TestClient`, `test_client`.

---

## 3. Архитектура и модель многопоточности

Ключевой архитектурный принцип `cryonet`: **многопоточностью управляет пользователь библиотеки, а не сама библиотека**.

### Почему функции `serve()` и `serve_tls()` однопоточные

Встроенные функции быстрого старта `serve()` и `serve_tls()`, а также метод `server.listen()` работают **строго в одном системном потоке** (single-threaded).

В этом режиме сервер в цикле последовательно вызывает `accept()` на сетевом сокете, полностью считывает запрос клиента, выполняет обработчик роутера, отдает ответ и только затем переходит к следующему входящему подключению.

Преимущества однопоточного режима:
- Абсолютная предсказуемость исполнения и простота отладки.
- Минимальное потребление оперативной памяти (нет оверхеда на выделение стеков неиспользуемых потоков ОС).
- Полное отсутствие гонок данных при обращении к общим структурам.
- Идеально для локальных утилит, внутренних микросервисов с умеренной нагрузкой и тестовых окружений.

### Как пользователь управляет многопоточностью

Если вашему приложению требуется одновременная параллельная обработка сотен или тысяч соединений на всех ядрах процессора, **вы сами определяете стратегию многопоточности**.

Для этого `cryonet` предоставляет низкоуровневые, не привязанные к скрытому рантайму методы:
1. `server.accept() -> TcpStream?` — неблокирующий или синхронный прием сырого входящего соединения от ОС.
2. `server.handle_connection(stream: TcpStream)` — полная обработка HTTP-соединения в вызывающем потоке.
3. `server.handle_tls_connection(stream: TcpStream)` — полная обработка защищенного HTTPS-соединения в вызывающем потоке.

#### Паттерн: Выделение воркера на каждое соединение через `spawn`

```yukino
load std.threads: spawn
load std.net.tcp: TcpStream
load std.io: println
load cryonet: Server, router, get, response_text

fun handle_worker(s: Server, stream: TcpStream) {
    // Вся обработка запроса выполняется внутри выделенного рабочего потока
    s.handle_connection(stream)
}

fun main() {
    set r = router()
    r.route("/compute", get(fun(ctx) => response_text("Parallel result", 200)))

    set s = init Server("0.0.0.0:8080", r, 10000, 2097152)
    println("HTTP Server listening on 0.0.0.0:8080 with worker threads...")

    // Accept-цикл находится под полным контролем пользователя
    while true {
        set stream_opt = s.accept()
        if stream_opt != null {
            set stream = stream_opt ?? init TcpStream(0)
            // Пользователь сам решает, как запустить задачу: spawn, пул или очередь
            spawn fun() => handle_worker(s, stream)
        }
    }
}
```

#### Паттерн: Защищенный многопоточный сервер с TLS (HTTPS)

Для HTTPS метод `server.handle_tls_connection(stream: TcpStream)` принимает **сырой незашифрованный `TcpStream`**:

```yukino
load std.threads: spawn
load std.net.tcp: TcpStream
load std.io: println
load cryonet: Server, router, get, response_text

fun handle_tls_worker(s: Server, stream: TcpStream) {
    // Криптографическое рукопожатие tls_accept выполняется внутри рабочего потока
    s.handle_tls_connection(stream)
}

fun main() {
    set r = router()
    r.route("/secure", get(fun(ctx) => response_text("Encrypted data", 200)))

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
> Передача сырого сокета `TcpStream` в `handle_tls_connection` гарантирует, что тяжелое криптографическое рукопожатие (`tls_accept`) выполняется **внутри рабочего потока пользователя**, а не в accept-цикле. Медленный клиент или зависшее TLS-рукопожатие не смогут заблокировать прием новых подключений для остальных пользователей.

---

## 4. Быстрый пример

### Шаг 1. Создание маршрутизатора и обработчиков

В файле `src/main.ykn`:

```yukino
load std.io: println
load cryonet: router, get, post, response_text, response_json, serve, logger_middleware, recovery_middleware

fun main() {
    set r = router()

    // Подключение middleware
    r.use(recovery_middleware())
    r.use(logger_middleware())

    // Определение маршрутов
    r.route("/", get(fun(ctx) => response_text("Cryonet Server is running", 200)))

    r.route("/api/users/:id", get(fun(ctx) => {
        set user_id = ctx.param("id") ?? "0"
        return response_json("{\"id\":${user_id},\"name\":\"Yukino Developer\"}", 200)
    }))

    r.route("/api/echo", post(fun(ctx) => {
        set payload = ctx.body()
        return response_text("Echo: " + payload, 200)
    }))

    println("Запуск сервера на http://127.0.0.1:8080...")
    serve("127.0.0.1:8080", r, 10000, 2097152)
}
```

### Шаг 2. Запуск и проверка через curl

```sh
# Запуск сервера
yukino run

# Проверка главного маршрута
curl -i http://127.0.0.1:8080/
# HTTP/1.1 200 OK
# Content-Type: text/plain; charset=utf-8
# Content-Length: 26
#
# Cryonet Server is running

# Проверка динамического параметра пути
curl -i http://127.0.0.1:8080/api/users/42
# HTTP/1.1 200 OK
# Content-Type: application/json; charset=utf-8
# Content-Length: 38
#
# {"id":42,"name":"Yukino Developer"}

# Проверка отправки POST-запроса
curl -i -X POST -d "Hello Cryonet" http://127.0.0.1:8080/api/echo
# HTTP/1.1 200 OK
# Content-Type: text/plain; charset=utf-8
# Content-Length: 19
#
# Echo: Hello Cryonet
```

---

## 5. Разделы документации

- [1. Маршрутизация и обработчики методов](1-routing.md) — детальное руководство по дереву маршрутов: именованные параметры (`:id`), подстановки (`*path`), система приоритетов, монтирование под-маршрутизаторов (`mount`), группировка (`scope`), автоответ `405 Method Not Allowed` и префиксные fallback-обработчики.
- [2. Контекст, запросы и ответы](2-context-handlers.md) — справочник по объекту `Context`: чтение query-параметров, заголовков, кук, подписанных HMAC-SHA256 кук, form-urlencoded и multipart файлов, конструкторы `Response`, нормализация IP через `strip_port`, а также работа с разделяемым состоянием `AppState` и `SubState`.
- [3. Конвейер промежуточного ПО (Middleware)](3-middleware.md) — луковичная модель middleware: встроенные компоненты перехвата паник (`recovery_middleware`), логгер (`logger_middleware`), CORS (`cors`), сквозной ID (`request_id_middleware`), сжатие ответов (`compression_middleware`), лимит тела (`body_limit`), рейт-лимитер по IP (`rate_limit_middleware`), а также руководство по созданию собственных middleware.
- [4. Real-Time потоки, статические файлы и тестирование](4-realtime-files.md) — справочник по интерактивным протоколам и тестированию: абстракция `ConnectionStream`, двусторонние сокеты WebSockets (`WebSocketSession`), Server-Sent Events (`SseWriter`), чанковый стриминг (`StreamWriter`), раздача статики (`ServeDir`, `ServeFile`, ETag 304, RFC 9110 Range 206/416), а также выполнение тестов в памяти через `TestClient`.

---

[Вперед: 1. Маршрутизация и обработчики методов](1-routing.md)
