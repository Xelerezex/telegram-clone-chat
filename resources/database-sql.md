
```sql
Project telegram_clone_mvp {
  database_type: "PostgreSQL"
  Note: '''
  Логическая схема БД для MVP мессенджера.
  Cloud Chats используют client-server encryption.
  Secret Chats используют end-to-end encryption и не входят в cloud history.
  '''
}

Table chat_types {
  id integer [pk, not null]
  code varchar(32) [not null, unique]
  name varchar(64) [not null]

  Note: 'Справочник типов чатов'
}

Table users {
  id uuid [pk, not null]
  username varchar(64) [not null, unique]
  phone_number varchar(32) [not null, unique]
  display_name varchar(128) [not null]
  bio varchar(512)
  created_at timestamp [not null]
  updated_at timestamp [not null]

  Note: 'Пользователи'

  Indexes {
    username [name: 'idx_users_username', unique]
    phone_number [name: 'idx_users_phone_number', unique]
  }
}

Table user_sessions {
  token varchar(128) [pk, not null]
  user_id uuid [not null]
  device_id varchar(128) [not null]
  user_agent varchar(255) [not null]
  ip_address varchar(64) [not null]
  created_at timestamp [not null]
  expires_at timestamp [not null]

  Note: 'Активные пользовательские сессии'

  Indexes {
    user_id [name: 'idx_user_sessions_user_id']
    expires_at [name: 'idx_user_sessions_expires_at']
  }
}

Table chats {
  id uuid [pk, not null]
  chat_type_id integer [not null]
  title varchar(255) [not null]
  last_message_id uuid
  created_at timestamp [not null]
  updated_at timestamp [not null]

  Note: 'Чаты, группы и каналы'

  Indexes {
    chat_type_id [name: 'idx_chats_chat_type_id']
    updated_at [name: 'idx_chats_updated_at']
  }
}

Table chat_members {
  chat_id uuid [not null]
  user_id uuid [not null]
  role varchar(32) [not null, note: 'owner | admin | member']
  last_read_seq_no bigint [not null, default: 0]
  unread_count integer [not null, default: 0]
  joined_at timestamp [not null]
  updated_at timestamp [not null]

  Note: 'Участники чатов и пользовательское состояние'

  Indexes {
    (chat_id, user_id) [pk]
    user_id [name: 'idx_chat_members_user_id']
    (chat_id, role) [name: 'idx_chat_members_chat_role']
  }
}

Table messages {
  id uuid [pk, not null]
  chat_id uuid [not null]
  sender_id uuid [not null]
  seq_no bigint [not null]
  body text [not null]
  created_at timestamp [not null]
  edited_at timestamp

  Note: 'История сообщений Cloud Chats'

  Indexes {
    (chat_id, seq_no) [name: 'uq_messages_chat_seq', unique]
    sender_id [name: 'idx_messages_sender_id']
    created_at [name: 'idx_messages_created_at']
  }
}

Table secret_chats {
  id uuid [pk, not null]
  initiator_user_id uuid [not null]
  initiator_device_id varchar(128) [not null]
  peer_user_id uuid [not null]
  peer_device_id varchar(128) [not null]
  key_fingerprint varchar(128) [not null]
  ttl_seconds integer
  created_at timestamp [not null]

  Note: 'Метаданные Secret Chats. Device-specific и вне cloud history.'

  Indexes {
    initiator_user_id [name: 'idx_secret_chats_initiator_user_id']
    peer_user_id [name: 'idx_secret_chats_peer_user_id']
  }
}

Table user_updates {
  id uuid [pk, not null]
  user_id uuid [not null]
  chat_id uuid [not null]
  message_id uuid [not null]
  update_seq_no bigint [not null]
  update_kind varchar(32) [not null, note: 'new_message | edit_message | delete_message | read_state']
  created_at timestamp [not null]

  Note: 'Поток пользовательских изменений только для Cloud Chats'

  Indexes {
    user_id [name: 'idx_user_updates_user_id']
    chat_id [name: 'idx_user_updates_chat_id']
    message_id [name: 'idx_user_updates_message_id']
    (user_id, update_seq_no) [name: 'uq_user_updates_user_seq', unique]
  }
}

Table device_sync_state {
  user_id uuid [not null]
  device_id varchar(128) [not null]
  last_update_seq_no bigint [not null, default: 0]
  last_sync_at timestamp [not null]

  Note: 'Состояние синхронизации устройства'

  Indexes {
    (user_id, device_id) [pk]
  }
}

Table user_presence {
  user_id uuid [not null]
  device_id varchar(128) [not null]
  status varchar(32) [not null, note: 'online | offline | away']
  last_seen_at timestamp [not null]
  updated_at timestamp [not null]

  Note: 'Состояние присутствия пользователя'

  Indexes {
    (user_id, device_id) [pk]
  }
}

Ref: user_sessions.user_id > users.id

Ref: chats.chat_type_id > chat_types.id
Ref: chats.last_message_id > messages.id

Ref: chat_members.chat_id > chats.id
Ref: chat_members.user_id > users.id

Ref: messages.chat_id > chats.id
Ref: messages.sender_id > users.id

Ref: secret_chats.initiator_user_id > users.id
Ref: secret_chats.peer_user_id > users.id

Ref: user_updates.user_id > users.id
Ref: user_updates.chat_id > chats.id
Ref: user_updates.message_id > messages.id

Ref: device_sync_state.user_id > users.id

Ref: user_presence.user_id > users.id
```