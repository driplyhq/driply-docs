# REST API Specification for MVP Backend
**Project:** Fashion/Outfit Platform (MVP Scope)
**Stack Alignment:** FastAPI + PostgreSQL + SQLAlchemy + Alembic + Session-based Auth + S3/R2
**API Version:** v1 | **Base URL:** `/api/v1`

## Global Conventions
| Параметр | Значение |
|---|---|
| **Auth Mechanism** | `session_id` HTTP-only cookie (set on `/auth/login`, auto-sent by browser) |
| **Content-Type** | `application/json` (default) / `multipart/form-data` (file uploads) |
| **Pagination** | `?page=1&limit=20` (default limit=20, max 100) |
| **Filtering** | `?category_id=uuid&style_id=uuid&brand_id=uuid&search=keyword` |
| **Sorting** | `?sort=date` (or other supported fields) |
| **Error Format** | `{"detail": "Human-readable message", "code": "ERROR_CODE"}` |
| **Images** | Stored in S3/R2. Backend returns `image_url` or generates `presigned_url` for direct upload |

## 1. Auth Endpoints
Session-based authentication via `starlette.middleware.sessions.SessionMiddleware`

| Method | URL | Description | Request / Response |
|---|---|---|---|
| `POST` | `/auth/register` | Регистрация нового пользователя | **Req:** `{"username": "alex", "email": "alex@mail.com", "password": "secure123"}`<br>**Res:** `201 {"message": "User registered", "user": {"id": "uuid", "username": "alex", "email": "alex@mail.com"}}` |
| `POST` | `/auth/login` | Вход в аккаунт (создание сессии) | **Req:** `{"username": "alex", "password": "secure123"}`<br>**Res:** `200 {"message": "Logged in"}` (Cookie: `session_id=abc123; HttpOnly; Secure; SameSite=Strict`) |
| `POST` | `/auth/logout` | Завершение сессии | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"message": "Logged out"}` |
| `GET` | `/auth/me` | Получение данных текущего пользователя | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"id": "uuid", "username": "alex", "email": "alex@mail.com", "avatar_url": null, "bio": "Fashion enthusiast", "created_at": "2024-01-15T10:00:00Z"}` |

## 2. Users & Profile
Public profile data + current user management

| Method | URL | Description | Request / Response |
|---|---|---|---|
| `GET` | `/users/{user_id}` | Публичный профиль пользователя | **Res:** `200 {"id": "uuid", "username": "alex", "bio": "...", "avatar_url": "...", "followers_count": 45, "following_count": 12, "is_following": false}` |
| `GET` | `/profile` | Данные текущего пользователя | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"id": "uuid", "username": "alex", "bio": "...", "avatar_url": "...", "followers_count": 45, "following_count": 12}` |
| `PUT` | `/profile` | Обновление профиля | **Req:** `{"username": "alex_new", "bio": "New bio"}`<br>**Res:** `200 {"message": "Profile updated"}` |
| `POST` | `/profile/avatar` | Загрузка аватара | **Req:** `multipart/form-data: image`<br>**Res:** `200 {"message": "Avatar uploaded", "avatar_url": "https://s3/..."}` |
| `GET` | `/profile/outfits` | Публикации текущего пользователя | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"outfits": [...], "total": 15, "page": 1}` |

## 3. Catalog & Items (Free Mode)
Свайп-механика, категории, стили, бренды

| Method | URL | Description | Request / Response |
|---|---|---|---|
| `GET` | `/items` | Каталог вещей с фильтрацией и поиском | **Req:** `?category_id=uuid&style_id=uuid&brand_id=uuid&search=leather&page=1&limit=20`<br>**Res:** `200 {"items": [{"id": "uuid", "name": "Leather Sneakers", "category": "shoes", "style": "minimalist", "brand": "Nike", "image_url": "...", "is_liked": false, "is_favorited": false}], "total": 45, "page": 1}` |
| `GET` | `/items/{item_id}` | Детали вещи | **Res:** `200 {"id": "uuid", "name": "Leather Sneakers", "description": "...", "category": "shoes", "style": "minimalist", "brand": "Nike", "image_urls": ["...", "..."], "is_liked": false, "is_favorited": false}` |
| `POST` | `/items/{item_id}/like` | Лайк вещи (Free Mode swipe) | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"message": "Liked", "likes_count": 12}` |
| `DELETE` | `/items/{item_id}/like` | Отмена лайка вещи | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"message": "Unlike", "likes_count": 11}` |
| `GET` | `/catalog/categories` | Список категорий | **Res:** `200 {"categories": [{"id": "uuid", "name": "top"}, {"id": "uuid", "name": "shoes"}]}` |
| `GET` | `/catalog/styles` | Список стилей | **Res:** `200 {"styles": [{"id": "uuid", "name": "minimalist"}, {"id": "uuid", "name": "streetwear"}]}` |
| `GET` | `/catalog/brands` | Список брендов | **Res:** `200 {"brands": [{"id": "uuid", "name": "Nike"}, {"id": "uuid", "name": "Zara"}]}` |

## 4. Outfits & Feed
Создание, публикация, лента аутфитов

| Method | URL | Description | Request / Response |
|---|---|---|---|
| `GET` | `/outfits` | Лента аутфитов (по дате или популярности) | **Req:** `?sort=date` |
| `GET` | `/outfits/{outfit_id}` | Детали аутфита | **Res:** `200 {"id": "uuid", "author": {...}, "title": "...", "description": "...", "style": "minimalist", "published_at": "...", "items": [{"item_id": "uuid", "name": "...", "image_url": "..."}], "likes_count": 12, "is_liked": false, "is_saved": false}` |
| `POST` | `/outfits` | Создание черновика аутфита | **Req:** `{"items": ["item_id_1", "item_id_2"], "title": "Beach day", "description": "Casual look"}`<br>**Res:** `201 {"id": "uuid", "status": "draft"}` |
| `PUT` | `/outfits/{outfit_id}` | Редактирование черновика | **Req:** `{"items": ["item_id_1"], "title": "Updated look", "description": "..."}`<br>**Res:** `200 {"message": "Outfit updated"}` |
| `DELETE` | `/outfits/{outfit_id}` | Удаление аутфита | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"message": "Outfit deleted"}` |
| `POST` | `/outfits/{outfit_id}/publish` | Публикация аутфита | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"message": "Outfit published", "published_at": "2024-06-10T14:30:00Z"}` |
| `POST` | `/outfits/{outfit_id}/like` | Лайк аутфита | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"message": "Liked", "likes_count": 13}` |
| `DELETE` | `/outfits/{outfit_id}/like` | Отмена лайка аутфита | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"message": "Unlike", "likes_count": 12}` |

## 5. Favorites & Interactions
Сохранение, избранное, стили, бренды

| Method | URL | Description | Request / Response |
|---|---|---|---|
| `GET` | `/favorites/items` | Избранные вещи | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"items": [{"id": "uuid", "name": "...", "image_url": "..."}]}` |
| `POST` | `/favorites/items/{item_id}` | Добавить вещь в избранное | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"message": "Item added to favorites"}` |
| `DELETE` | `/favorites/items/{item_id}` | Удалить вещь из избранного | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"message": "Item removed from favorites"}` |
| `GET` | `/favorites/outfits` | Сохранённые аутфиты | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"outfits": [{"id": "uuid", "title": "...", "image_url": "...", "published_at": "..."}]}` |
| `POST` | `/favorites/outfits/{outfit_id}` | Сохранить аутфит | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"message": "Outfit saved"}` |
| `DELETE` | `/favorites/outfits/{outfit_id}` | Удалить сохранённый аутфит | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"message": "Outfit unsaved"}` |
| `GET` | `/favorites/styles` | Избранные стили | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"styles": [{"id": "uuid", "name": "minimalist"}]}` |
| `POST` | `/favorites/styles/{style_id}` | Добавить стиль в избранное | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"message": "Style added"}` |
| `GET` | `/favorites/brands` | Избранные бренды | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"brands": [{"id": "uuid", "name": "Nike"}]}` |
| `POST` | `/favorites/brands/{brand_id}` | Добавить бренд в избранное | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"message": "Brand added"}` |

## 6. Social & Follows
Подписки, списки подписчиков/подписок

| Method | URL | Description | Request / Response |
|---|---|---|---|
| `POST` | `/profile/follow/{user_id}` | Подписаться на пользователя | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"message": "Subscribed"}` |
| `DELETE` | `/profile/follow/{user_id}` | Отписаться | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"message": "Unsubscribed"}` |
| `GET` | `/profile/{user_id}/followers` | Список подписчиков | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"users": [{"id": "uuid", "username": "maria", "avatar_url": "..."}]}` |
| `GET` | `/profile/{user_id}/following` | Список подписок | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"users": [...]}` |

## 7. Admin & Moderation
Управление контентом, пользователями, справочниками

| Method | URL | Description | Request / Response |
|---|---|---|---|
| `POST` | `/admin/items` | Создание вещи | **Req:** `multipart/form-data: name, category_id, style_id, brand_id, description, tags[]`<br>**Res:** `201 {"message": "Item created", "item_id": "uuid"}` |
| `PUT` | `/admin/items/{item_id}` | Обновление вещи | **Req:** `{"name": "New Name", "category_id": "uuid"}`<br>**Res:** `200 {"message": "Item updated"}` |
| `DELETE` | `/admin/items/{item_id}` | Удаление вещи | **Req:** Cookie: `session_id=admin_session`<br>**Res:** `200 {"message": "Item deleted"}` |
| `POST` | `/admin/styles` | Создание стиля | **Req:** `{"name": "y2k"}`<br>**Res:** `201 {"message": "Style created"}` |
| `PUT` | `/admin/styles/{style_id}` | Обновление стиля | **Req:** `{"name": "y2k_v2"}`<br>**Res:** `200 {"message": "Style updated"}` |
| `DELETE` | `/admin/styles/{style_id}` | Удаление стиля | **Req:** Cookie: `session_id=admin_session`<br>**Res:** `200 {"message": "Style deleted"}` |
| `POST` | `/admin/categories` | Создание категории | **Req:** `{"name": "outerwear"}`<br>**Res:** `201 {"message": "Category created"}` |
| `POST` | `/admin/brands` | Создание бренда | **Req:** `{"name": "H&M"}`<br>**Res:** `201 {"message": "Brand created"}` |
| `GET` | `/admin/users` | Список пользователей | **Req:** Cookie: `session_id=admin_session`<br>**Res:** `200 {"users": [...], "total": 100, "page": 1}` |
| `POST` | `/admin/users/{user_id}/block` | Блокировка пользователя | **Req:** Cookie: `session_id=admin_session`<br>**Res:** `200 {"message": "User blocked"}` |
| `POST` | `/admin/users/{user_id}/unblock` | Разблокировка пользователя | **Req:** Cookie: `session_id=admin_session`<br>**Res:** `200 {"message": "User unblocked"}` |
| `DELETE` | `/admin/users/{user_id}` | Удаление пользователя | **Req:** Cookie: `session_id=admin_session`<br>**Res:** `200 {"message": "User deleted"}` |
| `DELETE` | `/admin/outfits/{outfit_id}` | Удаление публикации | **Req:** Cookie: `session_id=admin_session`<br>**Res:** `200 {"message": "Outfit removed"}` |