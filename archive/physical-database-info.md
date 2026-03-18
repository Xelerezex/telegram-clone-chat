# Содержание
- [Содержание](#содержание)
	- [Таблица по запросам](#таблица-по-запросам)
	- [POST messages.send](#post-messages-send)
	- [GET messages.history](#get-messages-history)
	- [GET updates](#get-updates)
	- [GET dialogs](#get-dialogs)
	- [POST auth.validate](#post-auth-validate)
	- [GET presence](#get-presence)
---
## Таблица по запросам

Обозначения:
- $N_{members}$ - число участников чата.
- $N_{recv}=N_{members}-1$ - число получателей сообщения, кроме отправителя.
- $L_{history}$ - размер страницы истории, обычно `50`.
- $L_{updates}$ - размер пачки обновлений, обычно `100`.
- $L_{dialogs}$ - размер страницы диалогов, обычно `50`.

|API-вызов|Какие таблицы участвуют|Читается строк за 1 запрос|Пишется / обновляется строк|Итого логически затронуто|
|---|---|--:|--:|--:|
|`POST /messages/send`|`chat_members`, `messages`, `chats`, `user_updates`|$1$ в `chat_members` для проверки отправителя + $1$ в `messages` для вычисления следующего `seq_no` + $N_{recv}$ в `chat_members` для списка получателей|$1$ insert в `messages` + $1$ update в `chats` + $N_{recv}$ insert в `user_updates`|$2 + N_{recv}$ чтений и $2 + N_{recv}$ записей|
|`GET /messages/history`|`messages`|$L_{history}$|$0$|$L_{history}$|
|`GET /updates`|`device_sync_state`, `user_updates`|$1$ в `device_sync_state` + до $L_{updates}$ в `user_updates`|$0$|$1 + L_{updates}$|
|`GET /dialogs`|`chat_members`, `chats`|до $L_{dialogs}$ строк из `chat_members` + до $L_{dialogs}$ строк из `chats`|$0$|до $2 \cdot L_{dialogs}$|
|`POST /auth/validate`|`user_sessions`|$1$|$0$|$1$|
|`GET /presence`|`user_presence`|$1$|$0$|$1$|

### POST messages.send

При одной отправке сообщения логически происходит следующее:
1. Проверка, состоит ли отправитель в чате:  
    читается 1 строка из `chat_members`.
2. Получение следующего `seq_no` внутри чата:  
    читается 1 строка из `messages`, если брать последнее сообщение через `ORDER BY seq_no DESC LIMIT 1`.
3. Получение списка получателей:  
    **читается $N_{recv}$ строк** из `chat_members`.
4. Запись самого сообщения:  
    пишется 1 строка в `messages`.
5. Обновление `last_message_id` у чата:  
    обновляется 1 строка в `chats`.
6. Создание событий синхронизации для получателей:  
    пишется $N_{recv}$ строк в `user_updates`.
Итого:
- **читается:** $2 + N_{recv}$
- **пишется:** $2 + N_{recv}$
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
    $6,
    'new_message',
    now()
FROM chat_members cm
WHERE cm.chat_id = $1
  AND cm.user_id <> $2;
COMMIT;
```
### GET messages.history

Если история отдается страницей по `LIMIT 50`, то:
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

### GET updates

Обычно здесь, при $L_{updates} = 100$:
1. Читается курсор устройства из `device_sync_state`:  
    1 строка.
2. Читается пачка новых событий из `user_updates`:  
    до $L_{updates}$ строк.
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
### GET dialogs

Если список диалогов строится через `chat_members JOIN chats` и $L_{dialogs}​=50$, то логически:
- читается до $L_{dialogs}$ строк из `chat_members`;
- читается до $L_{dialogs}$ строк из `chats`.
Итого:
- **читается:** 100
- **пишется:** 0
```sql
SELECT c.id,
       c.title,
       c.last_message_id,
       c.updated_at,
       cm.role,
       cm.last_read_seq_no,
       cm.unread_count
FROM chat_members cm
JOIN chats c
  ON c.id = cm.chat_id
WHERE cm.user_id = $1
ORDER BY c.updated_at DESC
LIMIT 50;
```
### POST auth.validate

Здесь все просто:
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
### GET presence

Если смотреть логически по таблице `user_presence`, то:
- читается **1 строка**;
- запись не происходит.
Итого:
- **читается:** 1
- **пишется:** 0
```sql
SELECT status,
       last_seen_at,
       updated_at
FROM user_presence
WHERE user_id = $1
  AND device_id = $2;
```