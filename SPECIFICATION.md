# Kudos System Specification

> **Статус:** Утверждено  
> **Автор:** AI Architect + Graduate Developer (Datacom)  
> **Дата:** 2026-11-17  
> **Версия:** 1.0

---

## 1. Functional Requirements

### 1.1 User Stories

1. **US-01.** Как пользователь, я могу выбрать коллегу из выпадающего списка сотрудников, чтобы отправить ему благодарность.
2. **US-02.** Как пользователь, я могу написать сообщение с выражением признательности (максимум 500 символов).
3. **US-03.** Как пользователь, я могу отправить kudos, и оно будет сохранено в базе данных.
4. **US-04.** Как пользователь, я могу просматривать публичную ленту последних благодарностей на главной панели.
5. **US-05.** Как администратор, я могу **скрывать** неподобающие сообщения kudos (они перестают отображаться в публичной ленте, но сохраняются в БД).
6. **US-06.** Как администратор, я могу **удалять** сообщения kudos (полное удаление из системы).
7. **US-07.** Как администратор, я могу указать причину модерации и оставить след аудита (кто и когда модерировал).
8. **US-08.** Как пользователь, я могу получать уведомление, когда мне отправляют kudos.

### 1.2 Acceptance Criteria

| User Story | Acceptance Criteria |
|------------|---------------------|
| US-01 | Список загружается из Active Directory / HR-системы; поиск по имени работает; нельзя отправить kudos самому себе. |
| US-02 | Поле сообщения ограничено 500 символами; счётчик символов отображается в реальном времени; пустое сообщение нельзя отправить. |
| US-03 | После отправки пользователь видит подтверждение; kudos появляется в ленте в течение 2 секунд. |
| US-04 | Лента отображает последние 50 kudos; поддерживается пагинация; сортировка — по дате (новые сверху). |
| US-05 | Скрытый kudos не отображается в публичной ленте, но доступен администратору в панели модерации. |
| US-06 | Удалённый kudos физически удаляется из БД (soft-delete не применяется). |
| US-07 | В таблице `kudos` сохраняются поля `moderated_by`, `moderated_at`, `reason_for_moderation`. |
| US-08 | Уведомление появляется в bell-иконке и (опционально) отправляется на email. |

### 1.3 Edge Cases

- **Спам:** не более 10 kudos от одного пользователя в час (rate limiting).
- **Дубликаты:** нельзя отправить два одинаковых kudos одному и тому же пользователю в течение 24 часов.
- **Непристойный контент:** базовая фильтрация по списку запрещённых слов + возможность ручной модерации.
- **Неавторизованный доступ:** все API требуют JWT-токен.
- **Отключённый аккаунт:** пользователь не может отправлять kudos, если его аккаунт деактивирован.

---

## 2. Technical Design

### 2.1 Database Schema

```sql
-- Таблица пользователей (синхронизируется с HR-системой)
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) UNIQUE NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    department      VARCHAR(100),
    is_active       BOOLEAN DEFAULT TRUE,
    role            VARCHAR(20) DEFAULT 'user',  -- 'user' | 'admin'
    created_at      TIMESTAMP DEFAULT NOW()
);

-- Таблица благодарностей
CREATE TABLE kudos (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sender_id               UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    receiver_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    message                 TEXT NOT NULL CHECK (char_length(message) <= 500),
    is_visible              BOOLEAN DEFAULT TRUE,          -- ← НОВОЕ: для модерации
    moderated_by            UUID REFERENCES users(id),     -- ← НОВОЕ: кто модерировал
    moderated_at            TIMESTAMP,                     -- ← НОВОЕ: когда модерировали
    reason_for_moderation   TEXT,                          -- ← НОВОЕ: причина
    created_at              TIMESTAMP DEFAULT NOW(),
    
    CONSTRAINT chk_different_users CHECK (sender_id <> receiver_id)
);

-- Индексы для производительности
CREATE INDEX idx_kudos_receiver   ON kudos(receiver_id, created_at DESC);
CREATE INDEX idx_kudos_visible    ON kudos(is_visible, created_at DESC);
CREATE INDEX idx_kudos_created_at ON kudos(created_at DESC);