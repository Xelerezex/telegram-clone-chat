# Содержание
- [Содержание](#содержание)
	- [Ключи Primary Key](#ключи-primary-key)
	- [Таблица по запросам](#таблица-по-запросам)
	- [Краткое обоснование кэша](#краткое-обоснование-кэша)
	- [POST messages.send](#post-messagessend)
	- [GET messages.history](#get-messageshistory)
	- [GET updates](#get-updates)
	- [GET dialogs](#get-dialogs)
	- [POST auth.validate](#post-authvalidate)
	- [GET presence](#get-presence)

---

## Ключи Primary Key

| Физический объект | Ограничение | Поля | Для каких запросов нужен |
| --- | --- | --- | --- |
| `user_sessions` | `pk_user_sessions_token` | `token` | `POST /auth/validate` |
| `chats` | `pk_chats_id` | `id` | Локальный доступ к метаданным чата |
| `chat_members` | `pk_chat_members` | `(chat_id, user_id)` | Проверка ACL и роли отправителя |
| `device_sync_state` | `pk_device_sync_state` | `(user_id, device_id)` | Получение offset конкретного устройства |
| `dialogs_by_user` | `pk_dialogs_by_user` | `(user_id, chat_id)` | Upsert строки конкретного диалога пользователя |

---

## Таблица по запросам

Обозначения:
- $N_{members}$ — число участников чата.
- $N_{recv}=N_{members}-1$ — число получателей сообщения, кроме отправителя.
- $L_{history}$ — размер страницы истории, обычно `50`.
- $L_{updates}$ — размер пачки обновлений, обычно `100`.
- $L_{dialogs}$ — размер стартовой страницы диалогов, обычно `100`.

| API-вызов | Какие физические объекты участвуют | Читается строк / ключей за 1 запрос | Пишется / обновляется строк | Итого логически затронуто |
| --- | --- | --: | --: | --: |
| `POST /messages/send` | `chat_members`, `messages`, `chats`, `user_updates`, `dialogs_by_user` | $1$ в `chat_members` для проверки отправителя + $1$ в `messages` для вычисления следующего `seq_no` + $N_{members}$ в `chat_members` для fan-out и обновления dialog-проекции | $1$ insert в `messages` + $1$ update в `chats` + $N_{recv}$ insert в `user_updates` + $N_{members}$ upsert в `dialogs_by_user` | $2 + N_{members}$ чтений и $2 + N_{recv} + N_{members}$ записей |
| `GET /messages/history` | `messages` | $L_{history}$ | $0$ | $L_{history}$ |
| `GET /updates` | `device_sync_state`, `user_updates` | $1$ в `device_sync_state` + до $L_{updates}$ в `user_updates` | $0$ в рамках чистого чтения | $1 + L_{updates}$ |
| `GET /dialogs` | `dialogs_by_user` | $L_{dialogs}$ | $0$ | $L_{dialogs}$ |
| `POST /auth/validate` | `user_sessions` | $1$ | $0$ | $1$ |
| `GET /presence` | `user_presence` | $1$ ключ / hash | $0$ | $1$ |

---

## Краткое обоснование кэша

Для MVP отдельный server-side cache для `GET /dialogs` не нужен.

Берём продуктовые метрики:
- $DAU = 17{,}200{,}000$
- $DATSU = 41$ мин/день
- $DSU = 21$ сессия/день

Среднее число одновременно активных пользователей:

$$
Online_{avg}=DAU\cdot \frac{DATSU}{1440}
=17{,}200{,}000\cdot \frac{41}{1440}
\approx 489{,}722
$$

Если принять инженерную оценку размера одной строки `dialogs_by_user` как **256 Б**, то top-100 диалогов одного пользователя занимают:

$$
Size_{dialogs100}=100\cdot256=25{,}600\ \text{Б}\approx25.6\ \text{КБ}
$$

Средний hot set такого кэша:

$$
HotSet_{avg}=489{,}722\cdot25.6\ \text{КБ}\approx12.54\ \text{ГБ}
$$

Пиковый hot set при $k_{peak}=4$:

$$
HotSet_{peak}\approx50.15\ \text{ГБ}
$$

Средний интервал между сессиями одного пользователя:

$$
T_{gap}=\frac{1440}{21}\approx68.57\ \text{мин}
$$

Тогда грубая cache heat:
- при TTL = 5 минут: $\frac{5}{68.57}\approx7.3\%$
- при TTL = 10 минут: $\frac{10}{68.57}\approx14.6\%$

**Вывод:** для MVP выгоднее читать `dialogs_by_user` напрямую из user-shard по `user_id`.  
Отдельный server-side cache даёт мало повторных попаданий, но требует заметный объём памяти.  
Поэтому:
- `dialogs_by_user` читается напрямую из Citus;
- локальный кэш допустим на клиенте;
- Redis используется только для presence.

---

### POST messages.send

При одной отправке сообщения логически происходит следующее:

1. Проверка, состоит ли отправитель в чате:  
   читается 1 строка из `chat_members`.

2. Получение следующего `seq_no` внутри чата:  
   читается 1 строка из `messages`, если брать последнее сообщение через `ORDER BY seq_no DESC LIMIT 1`.

3. Получение списка участников чата для fan-out и обновления user-centric проекции:  
   читается **$N_{members}$ строк** из `chat_members`.

4. Запись самого сообщения:  
   пишется 1 строка в `messages`.

5. Обновление `last_message_id` у чата:  
   обновляется 1 строка в `chats`.

6. Создание пользовательских событий для всех получателей, кроме отправителя:  
   пишется $N_{recv}$ строк в `user_updates`.

7. Обновление `dialogs_by_user` для всех участников чата:  
   выполняется $N_{members}$ upsert-операций в `dialogs_by_user`, чтобы:
   - поднять чат наверх списка;
   - обновить preview последнего сообщения;
   - обновить `last_sender_id`;
   - обновить `unread_count`.

Итого:
- **читается:** $2 + N_{members}$
- **пишется:** $2 + N_{recv} + N_{members}$

Ниже показан упрощённый пример SQL-логики.  
Функции `next_update_seq_for_user(...)` и `next_dialog_order_key_for_user(...)` — это проектные псевдо-функции, которые обозначают получение следующего персонального номера события пользователя.

```sql
BEGIN;

-- 1. Проверка, что отправитель состоит в чате.
SELECT role
FROM chat_members
WHERE chat_id = $1
  AND user_id = $2;

-- 2. Получение следующего seq_no внутри чата.
SELECT COALESCE(seq_no, 0) + 1 AS next_seq_no
FROM messages
WHERE chat_id = $1
ORDER BY seq_no DESC
LIMIT 1;

-- 3. Вставка сообщения.
INSERT INTO messages (
    id,
    chat_id,
    sender_id,
    seq_no,
    body,
    created_at
) VALUES (
    $3,
    $1,
    $2,
    $4,
    $5,
    now()
);

-- 4. Обновление последнего сообщения в чате.
UPDATE chats
SET last_message_id = $3,
    updated_at = now()
WHERE id = $1;

-- 5. Создание user_updates для всех получателей, кроме отправителя.
INSERT INTO user_updates (
    id,
    user_id,
    chat_id,
    message_id,
    update_seq_no,
    update_kind,
    created_at
)
SELECT
    gen_random_uuid(),
    cm.user_id,
    $1,
    $3,
    next_update_seq_for_user(cm.user_id),
    'new_message',
    now()
FROM chat_members cm
WHERE cm.chat_id = $1
  AND cm.user_id <> $2;

-- 6. Обновление dialogs_by_user для всех участников чата.
INSERT INTO dialogs_by_user (
    user_id,
    chat_id,
    order_key,
    last_message_id,
    last_message_preview,
    last_sender_id,
    unread_count,
    last_read_seq_no,
    dialog_title,
    updated_at
)
SELECT
    cm.user_id,
    $1,
    next_dialog_order_key_for_user(cm.user_id),
    $3,
    LEFT($5, 120),
    $2,
    CASE
        WHEN cm.user_id = $2 THEN cm.unread_count
        ELSE cm.unread_count + 1
    END,
    cm.last_read_seq_no,
    c.title,
    now()
FROM chat_members cm
JOIN chats c
  ON c.id = cm.chat_id
WHERE cm.chat_id = $1
ON CONFLICT (user_id, chat_id) DO UPDATE
SET order_key = EXCLUDED.order_key,
    last_message_id = EXCLUDED.last_message_id,
    last_message_preview = EXCLUDED.last_message_preview,
    last_sender_id = EXCLUDED.last_sender_id,
    unread_count = EXCLUDED.unread_count,
    last_read_seq_no = EXCLUDED.last_read_seq_no,
    dialog_title = EXCLUDED.dialog_title,
    updated_at = EXCLUDED.updated_at;

COMMIT;
```

---

### GET messages.history

Если история отдаётся страницей по `LIMIT 50`, то:

- читается: $L_{history}$ строк из `messages`;
- пишется: `0`.

Итого:

- **читается:** 50
- **пишется:** 0
```sql
SELECT id,  
       sender_id,  
       seq_no,  
       body,  
       created_at,  
       edited_at  
FROM messages  
WHERE chat_id = $1  
  AND seq_no < $2  
ORDER BY seq_no DESC  
LIMIT 50;
```

---

### GET updates

Обычно здесь, при $L_{updates} = 100$:

1. Читается курсор устройства из `device_sync_state`:  
    1 строка.
2. Читается пачка новых событий из `user_updates`:  
    до $L_{updates}$ строк.

В чистом варианте этот запрос только читает.  
Продвижение `last_update_seq_no` выполняется отдельным подтверждением успешной синхронизации устройства.

Итого:

- **читается:** 101
- **пишется:** 0
```sql
-- 1. Получение текущего курсора устройства.  
SELECT last_update_seq_no,  
       last_sync_at  
FROM device_sync_state  
WHERE user_id = $1  
  AND device_id = $2;  
  
-- 2. Получение пачки новых обновлений.  
SELECT update_seq_no,  
       chat_id,  
       message_id,  
       update_kind,  
       created_at  
FROM user_updates  
WHERE user_id = $1  
  AND update_seq_no > $3  
ORDER BY update_seq_no  
LIMIT 100;
```

---
### GET dialogs

Список диалогов больше не строится через `chat_members JOIN chats`.  
Он читается напрямую из user-centric проекции `dialogs_by_user`.

Если стартовый экран загружает `top-100` последних диалогов, то:

- читается: $L_{dialogs}$ строк из `dialogs_by_user`;
- пишется: `0`.

Итого:

- **читается:** 100
- **пишется:** 0
```sql
SELECT chat_id,  
       dialog_title,  
       last_message_id,  
       last_message_preview,  
       last_sender_id,  
       unread_count,  
       last_read_seq_no,  
       updated_at  
FROM dialogs_by_user  
WHERE user_id = $1  
ORDER BY order_key DESC  
LIMIT 100;
```

---
### POST auth.validate

Здесь всё просто:

- читается 1 строка из `user_sessions`;
- запись не происходит.

Итого:

- **читается:** 1
- **пишется:** 0
```sql
SELECT user_id,  
       device_id,  
       expires_at  
FROM user_sessions  
WHERE token = $1  
  AND expires_at > now();
```

---
### GET presence

`GET /presence` в этой схеме работает не через PostgreSQL, а через `Redis Cluster`, потому что presence — это эфемерное состояние с минимальной задержкой доступа.

Логически здесь хранится состояние конкретного устройства пользователя:

- `status`
- `last_seen_at`
- `updated_at`

Физически запрос делает чтение **одного Redis key** по составному ключу вида:

```redis
presence:{user_id}:{device_id}
```

Поэтому стоимость запроса простая:

- **читается:** 1 key / hash
- **пишется:** 0

То есть это O(1)-доступ по ключу, без join, без обхода shard-ов и без чтения SQL-таблиц.

Пример логического чтения presence:

```redis
HMGET presence:{user_id}:{device_id} status last_seen_at updated_at
```

Если нужен весь объект состояния устройства:

```redis
HGETALL presence:{user_id}:{device_id}
```