# Real-Time потоки, статические файлы и тестирование

Библиотека `cryonet` предоставляет первоклассную поддержку интерактивных двусторонних соединений, потоковой передачи данных, производительной раздачи статических файлов и встроенных инструментов для тестирования маршрутов в оперативной памяти.

---

## 1. Подключение: load cryonet

```yukino
load cryonet: ConnectionStream, WebSocketSession, SseWriter, SseEvent, StreamWriter, sse_event_new, ServeDir, serve_dir, ServeFile, serve_file, serve_file_response, TestClient, test_client
```

---

## 2. Real-Time коммуникации: WebSockets, SSE и Chunked Transfer

В основе всех потоковых протоколов `cryonet` лежит класс `ConnectionStream`. Он прозрачно абстрагирует различия между обычным `TcpStream` и зашифрованным `TlsStream`. Метод `set_timeouts` применяет дедлайны к сетевому сокету даже при активном TLS, а `close()` защищен флагом `is_closed` от повторного закрытия дескрипторов.

### 2.1 WebSockets (RFC 6455)

`cryonet` включает полнофункциональную поддержку протокола WebSocket:
- Автоматическая проверка заголовков `Upgrade: websocket` и `Connection: Upgrade`.
- Вычисление ответа `Sec-WebSocket-Accept` по алгоритму SHA-1 и Base64 (GUID: `258EAFA5-E914-47DA-95CA-C5AB0DC85B11`).
- Поддержка текстовых (`0x1`) и бинарных (`0x2`) фреймов, сборка фрагментированных сообщений.
- Автоматическая обработка управляющих кадров: `Ping` (`0x9`), `Pong` (`0xA`) и `Close` (`0x8`).
- Опциональные таймауты неактивности сокета.

#### Способ 1: Декларативный хендлер через `ctx.websocket`

```yukino
r.get("/ws", fun(ctx) => {
    return ctx.websocket_with_timeout(fun(ws: WebSocketSession) => {
        println("WebSocket клиент подключен")

        // Отправка приветственного сообщения
        ws.send_text("Добро пожаловать в Cryonet WebSocket!")

        // Цикл приема сообщений
        while true {
            set msg_opt = ws.recv()
            if msg_opt == null {
                println("Клиент отключился")
                break
            }
            set msg = msg_opt ?? WsMessage { opcode: 0, text: "", data: cast("", "bytes"), is_binary: false }

            if !msg.is_binary {
                println("Получен текст: " + msg.text)
                // Отправляем эхо-ответ
                ws.send_text("Эхо: " + msg.text)
            } else {
                println("Получены бинарные данные: " + cast(msg.data.len(), "string") + " байт")
                ws.send_binary(msg.data)
            }
        }
    }, 30000) // таймаут неактивности 30 секунд
})
```

#### Способ 2: Ручной перехват через `ctx.upgrade_websocket()`

```yukino
r.get("/ws/custom", fun(ctx) => {
    set ws_opt = ctx.upgrade_websocket_with_timeout(60000)
    if ws_opt == null {
        return ctx.bad_request("Ожидался WebSocket handshake запрос")
    }

    set ws = ws_opt ?? init WebSocketSession(stream_noop(), 0)
    ws.send_text("Соединение установлено")
    ws.close()

    return response_hijacked()
})
```

---

### 2.2 Server-Sent Events (SSE)

Server-Sent Events позволяют организовать одностороннюю отправку событий с сервера в браузер через постоянное HTTP-соединение (`Content-Type: text/event-stream`):

```yukino
load std.threads: sleep

r.get("/events", fun(ctx) => {
    return ctx.sse(fun(sse: SseWriter) => {
        // Отправка простого события
        sse.send_data("Система инициализирована")

        // Отправка именованного события с полями
        set event = sse_event_new("Температура: 24.5C", "sensor_update", "msg_001", 3000)
        sse.send(event)

        // Периодическая отправка тиков
        for i in 1..5 {
            sleep(1000u64)
            sse.send_event("tick", "Секунда: " + cast(i, "string"))
        }

        sse.close()
    })
})
```

---

### 2.3 Потоковая передача данных (Chunked Transfer)

Используется для передачи файлов неизвестной заранее длины или генерации данных на лету по частям (`Transfer-Encoding: chunked`):

```yukino
load std.threads: sleep

r.get("/stream", fun(ctx) => {
    return ctx.chunked("text/plain; charset=utf-8", fun(stream: StreamWriter) => {
        stream.write_chunk("Начало передачи данных...\n")
        sleep(500u64)

        stream.write_chunk("Промежуточный блок 1\n")
        sleep(500u64)

        stream.write_chunk("Промежуточный блок 2\n")
        stream.end_stream()
        stream.close()
    })
})
```

---

## 3. Раздача статических файлов (Static Files)

Модуль `static_files` предоставляет производительные обработчики раздачи файлов с диска с соблюдением стандартов кэширования и частичной загрузки:

### 3.1 Возможности модуля

1. **Определение MIME-типов:** автоматически сопоставляет расширения файлов с соответствующими типами контента (HTML, CSS, JS, JSON, WASM, PNG, JPEG, SVG, MP4, PDF, шрифты и др.).
2. **Кэширование ETag и статус 304:** генерирует уникальный ETag на основе размера файла и времени последней модификации. При совпадении заголовка `If-None-Match` возвращает легкий статус `304 Not Modified` без чтения тела файла с диска.
3. **Частичная загрузка по RFC 9110 (Range requests):**
   - При валидном диапазоне (`Range: bytes=0-499`) отдает статус `206 Partial Content` с заголовком `Content-Range`.
   - При запредельном диапазоне (`Range: bytes=5000-6000` для файла в 100 байт) отдает `416 Range Not Satisfiable`.
   - При синтаксически невалидном заголовке `Range` (мусор, опечатка) заголовок игнорируется и отдается полный файл со статусом `200 OK`.
4. **Защита от Path Traversal:** блокирует попытки выхода за пределы корневой папки через сегменты `..`.

### 3.2 Раздача директорий (`serve_dir`) и одиночных файлов (`serve_file`)

```yukino
load cryonet: router, serve_dir, serve_file

set r = router()

// 1. Раздача целой директории по префиксу /public/*
// Файл ./static/css/style.css будет доступен по адресу /public/css/style.css
r.route("/public/*filepath", serve_dir("./static"))

// 2. Раздача одиночного файла (например, главной страницы)
r.route("/", serve_file("./static/index.html"))

// 3. Или через вспомогательные методы Router
r.static_dir("/assets", "./public/assets")
r.static_file("/favicon.ico", "./public/favicon.ico")
```

---

## 4. Тестирование без открытия сокетов (`TestClient`)

Класс `TestClient` позволяет выполнять интеграционные и модульные тесты обработчиков маршрутов прямо в оперативной памяти без привязки к реальным портам ОС:

```yukino
load std.io: println
load cryonet: router, get, post, response_text, response_json, test_client

fun main() {
    // 1. Создаем маршрутизатор и регистрируем маршруты
    set r = router()
    r.get("/api/user/:id", fun(ctx) => {
        set id = ctx.param("id") ?? "0"
        return response_json("{\"id\":${id},\"active\":true}", 200)
    })

    r.post("/api/echo", fun(ctx) => {
        return response_text("Echo: " + ctx.body(), 200)
    })

    // 2. Создаем тестовый клиент поверх маршрутизатора
    set client = test_client(r)

    // 3. Выполняем GET-запрос в памяти
    set resp_get = client.get("/api/user/42")
    assert(resp_get.status == 200)
    assert(resp_get.text() == "{\"id\":42,\"active\":true}")
    println("GET тест успешно пройден")

    // 4. Выполняем POST-запрос с телом
    set resp_post = client.post("/api/echo", "Тестовое сообщение")
    assert(resp_post.status == 200)
    assert(resp_post.text() == "Echo: Тестовое сообщение")
    println("POST тест успешно пройден")
}
```

---

[Назад: 3. Конвейер промежуточного ПО (Middleware)](3-middleware.md) | [Наверх: 0. Обзор и быстрый старт](0-overview.md)
