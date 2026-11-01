# Маршрутизация и обработчики методов

Модуль маршрутизации библиотеки `cryonet` предоставляет декларативное построение дерева маршрутов, сопоставление путей с приоритетами, извлечение параметров и группировку эндпоинтов.

---

## 1. Подключение: load cryonet

```yukino
load cryonet: Router, router, Route, MethodRouter, method_router, HandlerFn, get, post, put, delete, patch, head, options, any, clean_path, split_path, path_matches_prefix
```

---

## 2. Анатомия маршрутизатора (Router)

Маршрутизатор `Router` отвечает за регистрацию путей и диспетчеризацию входящих запросов к соответствующим обработчикам.

Создание экземпляра маршрутизатора выполняется через фабричную функцию:

```yukino
set r = router()
```

### 2.1 Обработчики HTTP-методов

Каждый маршрут связывается с одним или несколькими обработчиками методов через объект `MethodRouter`. Для удобства доступны хелперы стандартных HTTP-глаголов:

```yukino
r.route("/users", get(fun(ctx) => ctx.string("Список пользователей")))
r.route("/users", post(fun(ctx) => ctx.string("Создание пользователя")))
r.route("/users", put(fun(ctx) => ctx.string("Полное обновление")))
r.route("/users", patch(fun(ctx) => ctx.string("Частичное обновление")))
r.route("/users", delete(fun(ctx) => ctx.string("Удаление")))
r.route("/users", head(fun(ctx) => ctx.empty(200)))
r.route("/users", options(fun(ctx) => ctx.empty(204)))
```

Для регистрации обработчика под все методы сразу используется функция `any`:

```yukino
r.route("/webhook", any(fun(ctx) => {
    println("Получен запрос методом: " + ctx.method())
    return ctx.empty(200)
}))
```

Также доступны сокращенные методы прямо на объекте `Router`:

```yukino
r.get("/items", fun(ctx) => ctx.string("Все элементы"))
r.post("/items", fun(ctx) => ctx.string("Новый элемент"))
r.put("/items/:id", fun(ctx) => ctx.string("Обновление"))
r.delete("/items/:id", fun(ctx) => ctx.string("Удаление"))
r.patch("/items/:id", fun(ctx) => ctx.string("Патч"))
```

### 2.2 Автоматическая обработка 405 Method Not Allowed

Если запрошенный путь зарегистрирован в системе, но для конкретного HTTP-метода обработчик не задан, `cryonet` автоматически возвращает статус `405 Method Not Allowed` с обязательным заголовком `Allow`, перечисляющим все разрешенные методы для данного пути:

```http
HTTP/1.1 405 Method Not Allowed
Allow: GET, POST
Content-Type: text/plain; charset=utf-8
```

---

## 3. Динамические параметры пути и подстановки

`cryonet` поддерживает два типа динамических сегментов URL:

### 3.1 Именованные параметры (`:param`)

Сегменты, начинающиеся с двоеточия, захватывают ровно один сегмент пути между слешами:

```yukino
r.get("/users/:user_id/posts/:post_id", fun(ctx) => {
    set user_id = ctx.param("user_id") ?? ""
    set post_id = ctx.param_i64("post_id") ?? 0
    return ctx.string("Пользователь ${user_id}, пост ${post_id}")
})
```

Для безопасного преобразования в числа доступны вспомогательные методы:
- `ctx.param_int(name)` — возвращает `int32?` либо `null`, если параметр отсутствует или не является числом.
- `ctx.param_i64(name)` — возвращает `int64?` либо `null`.

### 3.2 Подстановочные знаки (Wildcards: `*name`)

Сегмент, начинающийся со звездочки, захватывает остаток пути целиком, включая все последующие вложенные слеши:

```yukino
r.get("/assets/*filepath", fun(ctx) => {
    set path = ctx.param("filepath") ?? ""
    return ctx.string("Запрошен файл: " + path)
})
```

При запросе `GET /assets/css/themes/dark.css` параметр `filepath` будет равен `css/themes/dark.css`.

### 3.3 Приоритет сопоставления маршрутов

Маршруты вычисляются на основе приоритетного веса каждого сегмента:
1. **Статический сегмент** (например, `/users/new`) имеет максимальный приоритет (вес `10000`).
2. **Именованный параметр** (например, `/users/:id`) имеет средний приоритет (вес `100`).
3. **Подстановка** (например, `/users/*rest`) имеет наименьший приоритет (вес `1`).

Благодаря этому путь `/users/new` никогда не будет ошибочно перехвачен обработчиком `/users/:id`, независимо от порядка их регистрации в коде.

---

## 4. Группировка и вложенность маршрутов

### 4.1 Метод `mount` (монтирование поддеревьев)

Позволяет смонтировать независимый под-маршрутизатор по определенному префиксу:

```yukino
set api_router = router()
api_router.get("/status", fun(ctx) => ctx.string("OK"))
api_router.get("/version", fun(ctx) => ctx.string("1.0.0"))

set app_router = router()
app_router.mount("/api/v1", api_router)
```

Маршруты станут доступны по адресам `/api/v1/status` и `/api/v1/version`.

### 4.2 Метод `scope` (группировка с замыканием)

Позволяет компактно объявить группу маршрутов со сдвигом базового пути:

```yukino
r.scope("/admin", fun(admin) => {
    admin.get("/dashboard", fun(ctx) => ctx.string("Dashboard"))
    admin.get("/settings", fun(ctx) => ctx.string("Settings"))
})
```

---

## 5. Обработчики ошибок 404 (Fallback)

### 5.1 Глобальный fallback

Вызывается, если ни один зарегистрированный маршрут не подошел под запрос:

```yukino
r.fallback(fun(ctx) => {
    return ctx.not_found("Страница не найдена. Попробуйте перейти на главную.")
})
```

### 5.2 Префиксный fallback (`fallback_for`)

Позволяет настроить специализированные ответы 404 для определенных подсистем (например, JSON для API и HTML для веб-страниц):

```yukino
// Для всех ненайденных путей в /api/* отдаем JSON
r.fallback_for("/api", fun(ctx) => {
    return ctx.json_str("{\"error\":\"endpoint not found\"}", 404)
})
```

---

## 6. Вспомогательные функции путей

Библиотека экспортирует низкоуровневые утилиты нормализации путей:

- `clean_path(path: string) -> string` — удаляет query-параметры, убирает дублирующиеся хвостовые слеши, гарантирует ведущий `/`.
- `split_path(path: string) -> string[]` — разбивает путь на непустые сегменты.
- `path_matches_prefix(path: string, prefix: string) -> bool` — проверяет соответствие пути заданному префиксу с учетом границ сегментов.

---

[Назад: 0. Обзор и быстрый старт](0-overview.md) | [Вперед: 2. Контекст, запросы и ответы](2-context-handlers.md)
