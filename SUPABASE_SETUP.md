# Настройка Supabase для Зеркало

## Ошибки 401 / 42501 (RLS)

Если при создании групп или вступлении в них вы видите ошибки `401 Unauthorized` или `42501 new row violates row-level security policy`, нужно настроить политики RLS в Supabase.

### 1. Таблица `group_members`

В Supabase Dashboard: **Table Editor** → **group_members** → **Policies**

Добавьте политику для **INSERT**:
- **Policy name**: `Allow insert for authenticated`
- **Allowed operation**: INSERT
- **Target roles**: (оставьте пустым для всех)
- **Using expression**: `true` (или более строгая: `auth.role() = 'authenticated'`)

Для упрощения можно временно отключить RLS на таблице (не рекомендуется для продакшена):
```sql
ALTER TABLE group_members DISABLE ROW LEVEL SECURITY;
```

### 2. Таблица `groups`

Убедитесь, что политики разрешают:
- **SELECT** — чтение всех групп (или публичных)
- **INSERT** — создание групп аутентифицированными пользователями
- **UPDATE** — обновление счётчика участников

### 3. Дубликат username (ошибка 23505)

Уникальность `groups.username` уже проверяется в приложении перед вставкой. Если ошибка всё равно появляется, выберите другой юзернейм для группы (например, `@mygroup2`).

## Таблицы проекта

Убедитесь, что в Supabase созданы таблицы:
- `users`, `posts`, `comments`, `likes`, `followers`
- `groups`, `group_members`
- `conversations`, `conversation_participants`, `messages`
- `notifications`

Связи и внешние ключи должны быть настроены согласно схеме приложения.
