-- ========================================
-- ВАЖНО: RLS и кастомная авторизация
-- ========================================
-- Приложение использует кастомную авторизацию (не Supabase Auth).
-- auth.uid() в RLS будет null. Для работы отключите RLS:
-- ALTER TABLE users DISABLE ROW LEVEL SECURITY;
-- ALTER TABLE followers DISABLE ROW LEVEL SECURITY;
-- (и остальные таблицы аналогично)

-- ========================================
-- УДАЛЕНИЕ СТАРЫХ ТАБЛИЦ И СОЗДАНИЕ НОВЫХ
-- ========================================

-- Удаляем старые таблицы если они существуют
DROP TABLE IF EXISTS notifications CASCADE;
DROP TABLE IF EXISTS messages CASCADE;
DROP TABLE IF EXISTS conversations CASCADE;
DROP TABLE IF EXISTS post_likes CASCADE;
DROP TABLE IF EXISTS comments CASCADE;
DROP TABLE IF EXISTS group_members CASCADE;
DROP TABLE IF EXISTS posts CASCADE;
DROP TABLE IF EXISTS groups CASCADE;
DROP TABLE IF EXISTS followers CASCADE;
DROP TABLE IF EXISTS users CASCADE;

-- Расширение для UUID
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- ========================================
-- ТАБЛИЦА ПОЛЬЗОВАТЕЛЕЙ
-- ========================================
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    username TEXT UNIQUE NOT NULL,
    display_name TEXT NOT NULL,
    emoji TEXT NOT NULL DEFAULT '😀',
    email TEXT UNIQUE,
    password_hash TEXT NOT NULL,
    description TEXT DEFAULT 'Пользователь пока ничего не написал в описании',
    status TEXT DEFAULT '',
    clan TEXT DEFAULT '😀',
    is_verified BOOLEAN DEFAULT FALSE,
    is_online BOOLEAN DEFAULT FALSE,
    last_seen TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    avatar_color TEXT DEFAULT '#6366f1',
    cover_image TEXT,
    posts_count INTEGER DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- ========================================
-- ТАБЛИЦА ПОДПИСОК
-- ========================================
CREATE TABLE followers (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    follower_id UUID REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(user_id, follower_id)
);

-- ========================================
-- ТАБЛИЦА ПОСТОВ
-- ========================================
CREATE TABLE posts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    content TEXT NOT NULL,
    media_url TEXT,
    media_type TEXT, -- 'image', 'video', 'audio', 'file'
    media_name TEXT,
    media_size INTEGER,
    voice_url TEXT,
    voice_duration INTEGER,
    original_post_id UUID REFERENCES posts(id) ON DELETE SET NULL,
    is_repost BOOLEAN DEFAULT FALSE,
    likes_count INTEGER DEFAULT 0,
    comments_count INTEGER DEFAULT 0,
    reposts_count INTEGER DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- ========================================
-- ТАБЛИЦА ЛАЙКОВ ПОСТОВ
-- ========================================
CREATE TABLE post_likes (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    post_id UUID REFERENCES posts(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(post_id, user_id)
);

-- ========================================
-- ТАБЛИЦА КОММЕНТАРИЕВ
-- ========================================
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    post_id UUID REFERENCES posts(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    content TEXT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- ========================================
-- ТАБЛИЦА ГРУПП
-- ========================================
CREATE TABLE groups (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    username TEXT UNIQUE NOT NULL,
    description TEXT,
    avatar TEXT DEFAULT '👥',
    cover_image TEXT,
    type TEXT DEFAULT 'group', -- 'group', 'channel', 'community'
    is_public BOOLEAN DEFAULT TRUE,
    creator_id UUID REFERENCES users(id) ON DELETE SET NULL,
    members_count INTEGER DEFAULT 0,
    posts_count INTEGER DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- ========================================
-- ТАБЛИЦА УЧАСТНИКОВ ГРУПП
-- ========================================
CREATE TABLE group_members (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    group_id UUID REFERENCES groups(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    role TEXT DEFAULT 'member', -- 'member', 'admin', 'moderator'
    joined_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(group_id, user_id)
);

-- ========================================
-- ТАБЛИЦА СООБЩЕНИЙ (ЧАТ)
-- ========================================
CREATE TABLE conversations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    is_group BOOLEAN DEFAULT FALSE,
    name TEXT,
    avatar TEXT,
    last_message TEXT,
    last_message_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- ========================================
-- ТАБЛИЦА УЧАСТНИКОВ ДИАЛОГА
-- ========================================
CREATE TABLE conversation_participants (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    conversation_id UUID REFERENCES conversations(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    joined_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(conversation_id, user_id)
);

-- ========================================
-- ТАБЛИЦА СООБЩЕНИЙ
-- ========================================
CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    conversation_id UUID REFERENCES conversations(id) ON DELETE CASCADE,
    sender_id UUID REFERENCES users(id) ON DELETE CASCADE,
    content TEXT,
    media_url TEXT,
    media_type TEXT,
    media_name TEXT,
    read_by UUID[] DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- ========================================
-- ТАБЛИЦА УВЕДОМЛЕНИЙ
-- ========================================
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    from_user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    post_id UUID REFERENCES posts(id) ON DELETE SET NULL,
    type TEXT NOT NULL, -- 'like', 'comment', 'repost', 'follow', 'message', 'mention'
    message TEXT NOT NULL,
    is_read BOOLEAN DEFAULT FALSE,
    metadata JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- ========================================
-- ИНДЕКСЫ ДЛЯ ОПТИМИЗАЦИИ
-- ========================================

-- Индексы для users
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_created_at ON users(created_at);
CREATE INDEX idx_users_is_online ON users(is_online);

-- Индексы для followers
CREATE INDEX idx_followers_user_id ON followers(user_id);
CREATE INDEX idx_followers_follower_id ON followers(follower_id);
CREATE INDEX idx_followers_created_at ON followers(created_at);

-- Индексы для posts
CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_posts_created_at ON posts(created_at);
CREATE INDEX idx_posts_original_post_id ON posts(original_post_id);

-- Индексы для post_likes
CREATE INDEX idx_post_likes_post_id ON post_likes(post_id);
CREATE INDEX idx_post_likes_user_id ON post_likes(user_id);

-- Индексы для comments
CREATE INDEX idx_comments_post_id ON comments(post_id);
CREATE INDEX idx_comments_user_id ON comments(user_id);
CREATE INDEX idx_comments_created_at ON comments(created_at);

-- Индексы для groups
CREATE INDEX idx_groups_username ON groups(username);
CREATE INDEX idx_groups_creator_id ON groups(creator_id);
CREATE INDEX idx_groups_type ON groups(type);

-- Индексы для group_members
CREATE INDEX idx_group_members_group_id ON group_members(group_id);
CREATE INDEX idx_group_members_user_id ON group_members(user_id);

-- Индексы для conversations
CREATE INDEX idx_conversations_last_message_at ON conversations(last_message_at);

-- Индексы для conversation_participants
CREATE INDEX idx_conversation_participants_conversation_id ON conversation_participants(conversation_id);
CREATE INDEX idx_conversation_participants_user_id ON conversation_participants(user_id);

-- Индексы для messages
CREATE INDEX idx_messages_conversation_id ON messages(conversation_id);
CREATE INDEX idx_messages_sender_id ON messages(sender_id);
CREATE INDEX idx_messages_created_at ON messages(created_at);

-- Индексы для notifications
CREATE INDEX idx_notifications_user_id ON notifications(user_id);
CREATE INDEX idx_notifications_is_read ON notifications(is_read);
CREATE INDEX idx_notifications_created_at ON notifications(created_at);

-- ========================================
-- ТРИГГЕРЫ И ФУНКЦИИ
-- ========================================

-- Функция обновления времени
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Функция обновления счетчиков постов пользователя
CREATE OR REPLACE FUNCTION update_user_posts_count()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE users 
        SET posts_count = posts_count + 1 
        WHERE id = NEW.user_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE users 
        SET posts_count = GREATEST(posts_count - 1, 0) 
        WHERE id = OLD.user_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Функция обновления счетчиков лайков поста
CREATE OR REPLACE FUNCTION update_post_likes_count()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE posts 
        SET likes_count = likes_count + 1 
        WHERE id = NEW.post_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE posts 
        SET likes_count = GREATEST(likes_count - 1, 0) 
        WHERE id = OLD.post_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Функция обновления счетчиков комментариев поста
CREATE OR REPLACE FUNCTION update_post_comments_count()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE posts 
        SET comments_count = comments_count + 1 
        WHERE id = NEW.post_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE posts 
        SET comments_count = GREATEST(comments_count - 1, 0) 
        WHERE id = OLD.post_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Функция обновления счетчика участников группы
CREATE OR REPLACE FUNCTION update_group_members_count()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE groups 
        SET members_count = members_count + 1 
        WHERE id = NEW.group_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE groups 
        SET members_count = GREATEST(members_count - 1, 0) 
        WHERE id = OLD.group_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Функция обновления счетчика подписчиков
CREATE OR REPLACE FUNCTION update_user_followers_count()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE users 
        SET followers_count = COALESCE(followers_count, 0) + 1 
        WHERE id = NEW.user_id;
        
        UPDATE users 
        SET following_count = COALESCE(following_count, 0) + 1 
        WHERE id = NEW.follower_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE users 
        SET followers_count = GREATEST(COALESCE(followers_count, 0) - 1, 0) 
        WHERE id = OLD.user_id;
        
        UPDATE users 
        SET following_count = GREATEST(COALESCE(following_count, 0) - 1, 0) 
        WHERE id = OLD.follower_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Триггеры для обновления updated_at
CREATE TRIGGER update_users_updated_at 
    BEFORE UPDATE ON users 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_posts_updated_at 
    BEFORE UPDATE ON posts 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_groups_updated_at 
    BEFORE UPDATE ON groups 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_comments_updated_at 
    BEFORE UPDATE ON comments 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- Триггеры для обновления счетчиков
CREATE TRIGGER update_posts_count_trigger 
    AFTER INSERT OR DELETE ON posts 
    FOR EACH ROW EXECUTE FUNCTION update_user_posts_count();

CREATE TRIGGER update_post_likes_count_trigger 
    AFTER INSERT OR DELETE ON post_likes 
    FOR EACH ROW EXECUTE FUNCTION update_post_likes_count();

CREATE TRIGGER update_post_comments_count_trigger 
    AFTER INSERT OR DELETE ON comments 
    FOR EACH ROW EXECUTE FUNCTION update_post_comments_count();

CREATE TRIGGER update_group_members_count_trigger 
    AFTER INSERT OR DELETE ON group_members 
    FOR EACH ROW EXECUTE FUNCTION update_group_members_count();

CREATE TRIGGER update_followers_count_trigger 
    AFTER INSERT OR DELETE ON followers 
    FOR EACH ROW EXECUTE FUNCTION update_user_followers_count();

-- ========================================
-- ПОЛИТИКИ БЕЗОПАСНОСТИ (RLS)
-- ========================================
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE followers ENABLE ROW LEVEL SECURITY;
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;
ALTER TABLE post_likes ENABLE ROW LEVEL SECURITY;
ALTER TABLE comments ENABLE ROW LEVEL SECURITY;
ALTER TABLE groups ENABLE ROW LEVEL SECURITY;
ALTER TABLE group_members ENABLE ROW LEVEL SECURITY;
ALTER TABLE conversations ENABLE ROW LEVEL SECURITY;
ALTER TABLE conversation_participants ENABLE ROW LEVEL SECURITY;
ALTER TABLE messages ENABLE ROW LEVEL SECURITY;
ALTER TABLE notifications ENABLE ROW LEVEL SECURITY;

-- Политики для пользователей
CREATE POLICY "Пользователи видят всех пользователей" ON users
    FOR SELECT USING (true);

CREATE POLICY "Пользователи могут создавать аккаунты" ON users
    FOR INSERT WITH CHECK (true);

CREATE POLICY "Пользователи могут обновлять свой профиль" ON users
    FOR UPDATE USING (auth.uid() = id);

-- Политики для постов
CREATE POLICY "Все пользователи видят посты" ON posts
    FOR SELECT USING (true);

CREATE POLICY "Пользователи могут создавать посты" ON posts
    FOR INSERT WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Пользователи могут обновлять свои посты" ON posts
    FOR UPDATE USING (auth.uid() = user_id);

CREATE POLICY "Пользователи могут удалять свои посты" ON posts
    FOR DELETE USING (auth.uid() = user_id);

-- Политики для сообщений
CREATE POLICY "Пользователи видят свои диалоги" ON conversations
    FOR SELECT USING (
        EXISTS (
            SELECT 1 FROM conversation_participants 
            WHERE conversation_id = conversations.id 
            AND user_id = auth.uid()
        )
    );

CREATE POLICY "Пользователи могут создавать диалоги" ON conversations
    FOR INSERT WITH CHECK (true);

CREATE POLICY "Пользователи видят сообщения в своих диалогах" ON messages
    FOR SELECT USING (
        EXISTS (
            SELECT 1 FROM conversation_participants 
            WHERE conversation_id = messages.conversation_id 
            AND user_id = auth.uid()
        )
    );

CREATE POLICY "Пользователи могут отправлять сообщения" ON messages
    FOR INSERT WITH CHECK (auth.uid() = sender_id);

-- ========================================
-- ТЕСТОВЫЕ ДАННЫЕ (ОПЦИОНАЛЬНО)
-- ========================================
-- Вставляем тестового пользователя
INSERT INTO users (username, display_name, email, password_hash, emoji, clan, is_verified)
VALUES 
    ('@onegin', 'Евгений Онегин', 'onegin@example.com', 'admin123', '👑', '👑', true),
    ('@pushkin', 'Александр Пушкин', 'pushkin@example.com', 'poet123', '📚', '📚', true),
    ('@lermontov', 'Михаил Лермонтов', 'lermontov@example.com', 'officer123', '⚔️', '⚔️', false)
ON CONFLICT (username) DO NOTHING;