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
| **Error Format** | `{"detail": {"message": "Human-readable message", "code": "ERROR_CODE"}}` |
| **Error Example** | `400 {"detail": {"message": "Пароли не совпадают", "code": "password_mismatch"}}` |
| **Images** | Stored in S3/R2. Backend returns `image_url` or generates `presigned_url` for direct upload |

## 1. Auth Endpoints
Session-based authentication via `starlette.middleware.sessions.SessionMiddleware`

| Method | URL | Description | Request / Response |
|---|---|---|---|
| `POST` | `/auth/register` | Регистрация нового пользователя | **Req:** `{"username": "alex", "email": "alex@mail.com", "password": "secure123", "password_confirm": "secure123"}`<br>**Res:** `201 {"message": "Пользователь зарегистрирован", "user": {"id": "uuid", "username": "alex", "email": "alex@mail.com"}}` |
| `POST` | `/auth/login` | Вход в аккаунт (создание сессии) | **Req:** `{"email": "alex@mail.com", "password": "secure123"}`<br>**Res:** `200 {"message": "Вы вошли в систему"}` (Cookie: `session_id=abc123; HttpOnly; Secure; SameSite=Strict`) |
| `POST` | `/auth/logout` | Завершение сессии | **Req:** Cookie: `session_id=abc123`<br>**Res:** `204` |
| `GET` | `/auth/me` | Получение данных текущего пользователя | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"id": "uuid", "username": "alex", "email": "alex@mail.com", "avatar_url": null, "bio": "Fashion enthusiast"}`<br>**Error:** `401 {"detail": {"message": "Не авторизован", "code": "unauthorized"}}` |

## 2. Users & Profile
Public profile data + current user management

| Method | URL | Description | Request / Response |
|---|---|---|---|
| `GET` | `/users/{username}` | Публичный профиль пользователя | **Res:** `200 {"id": "uuid", "username": "alex", "bio": "...", "avatar_url": "...", "followers_count": 45, "following_count": 12, "is_following": false}` |
| `GET` | `/profile/me` | Данные текущего пользователя | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"id": "uuid", "username": "alex", "bio": "...", "avatar_url": "...", "followers_count": 45, "following_count": 12}` |
| `PATCH` | `/profile/me` | Обновление профиля | **Req:** `{"username": "alex_new", "bio": "New bio"}`<br>**Res:** `200 {"message": "Профиль обновлен", "user": {"id": "uuid", "username": "alex", "bio": "...", "avatar_url": "...", "followers_count": 45, "following_count": 12}}` |
| `POST` | `/profile/avatar` | Загрузка аватара | **Req:** `multipart/form-data: image`<br>**Res:** `200 {"message": "Аватар загружен", "file_url": "https://s3/..."}` |
| `GET` | `/profile/{username}/outfits` | Публикации пользователя | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"outfits": [{"id": "uuid", "image_url": "...", "likes_count": 12}], "total": 15, "page": 1}` |

> **Примечание:** обложка аутфита — поле `image_url` (картинка с `order=0`), не массив. Путь `/profile/{username}/outfits` (публикации любого пользователя). Эндпоинт «мои публикации» через сессию (`/profile/outfits` из спеки) — на согласовании.

## 3. Catalog & Items (Free Mode)
Свайп-механика, категории, стили, бренды

| Method | URL | Description | Request / Response |
|---|---|---|---|
| `GET` | `/items` | Каталог вещей с фильтрацией и поиском | **Req:** Cookie: `session_id=abc123` (для `is_liked`)<br>`?category_id=uuid&style_id=uuid&brand_id=uuid&search=leather&page=1&limit=20`<br>**Res:** `200 {"items": [{"id": "uuid", "name": "Leather Sneakers", "category": {"id": "uuid", "name": "shoes"}, "style": {"id": "uuid", "name": "minimalist"}, "brand": {"id": "uuid", "name": "Nike", "image_url": "..."}, "image_url": "...", "is_liked": false, "likes_count": 5}], "total": 45, "page": 1}`<br>**Error:** `404 {"detail": {"message": "Категория не найдена", "code": "category_not_found"}}` (при фильтре по несуществующему id) |
| `GET` | `/items/{item_id}` | Детали вещи | **Req:** Cookie: `session_id=abc123` (для `is_liked`)<br>**Res:** `200 {"id": "uuid", "name": "Leather Sneakers", "description": "...", "category": {"id": "uuid", "name": "shoes"}, "style": {"id": "uuid", "name": "minimalist"}, "brand": {"id": "uuid", "name": "Nike", "image_url": "..."}, "image_urls": ["...", "..."], "is_liked": false, "likes_count": 5}`<br>**Error:** `404 {"detail": {"message": "Вещь не найдена", "code": "item_not_found"}}` |
| `POST` | `/items/{item_id}/like` | Лайк вещи (Free Mode swipe) | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"message": "Добавлен лайк", "likes_count": 12}`<br>**Error:** `409 {"detail": {"message": "Вещь уже в лайках", "code": "already_liked"}}` |
| `DELETE` | `/items/{item_id}/like` | Отмена лайка вещи | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"message": "Лайк удален", "likes_count": 11}`<br>**Error:** `400 {"detail": {"message": "Вещь не в лайках", "code": "not_liked"}}` |
| `GET` | `/catalog/categories` | Список категорий | **Req:** `?page=1&limit=100`<br>**Res:** `200 {"categories": [{"id": "uuid", "name": "top"}, {"id": "uuid", "name": "shoes"}], "total": 10, "page": 1}` |
| `GET` | `/catalog/styles` | Список стилей | **Req:** `?page=1&limit=100`<br>**Res:** `200 {"styles": [{"id": "uuid", "name": "minimalist"}, {"id": "uuid", "name": "streetwear"}], "total": 8, "page": 1}` |
| `GET` | `/catalog/brands` | Список брендов | **Req:** `?page=1&limit=100`<br>**Res:** `200 {"brands": [{"id": "uuid", "name": "Nike", "image_url": "..."}, {"id": "uuid", "name": "Zara", "image_url": "..."}], "total": 50, "page": 1}` |

> **Примечание:** `is_favorited` для вещей удалён — избранного вещей нет, только лайки (`is_liked` / `likes_count`). Поиск (`search`) работает по названию вещи, описанию и имени бренда.

## 4. Outfits & Feed
Создание, лента, лайки, картинки, комментарии аутфитов

> **Статус:** реализовано (feat/6-week). Черновики и публикация (`status: draft`, `POST /publish`, `published_at`) **вырезаны из MVP** — созданный аутфит сразу опубликован. `is_saved` и сохранённые аутфиты вырезаны (см. раздел 5). Поле `style` в деталях не реализовано (в модели его нет — открытый вопрос). Лента — тикток-паттерн: карточка = полный пост.

| Method | URL | Description | Request / Response |
|---|---|---|---|
| `GET` | `/outfits` | Лента аутфитов (публичная, полные карточки) | **Req:** `?sort=date&page=1&limit=20` (cookie опционален — для `is_liked`)<br>**Res:** `200 {"outfits": [{"id": "uuid", "title": "...", "description": "...", "images": ["url0", "url1"], "items": [{"item_id": "uuid", "name": "...", "image_url": "..."}], "likes_count": 12, "comments_count": 3, "is_liked": false, "author": {"id": "uuid", "username": "alex", "avatar_url": "..."}}], "page": 1}`<br>**Без `total`** — конец ленты: пришло меньше `limit`. `images` — все холсты поста по `order` (свайп вбок). В `items` возможны дубли (две одинаковые вещи — легально). Анониму `is_liked=false` |
| `GET` | `/outfits/{outfit_id}` | Деталь аутфита (та же полная карточка) | **Req:** cookie опционален<br>**Res:** `200` — карточка как в ленте<br>**Error:** `404 {"detail": {"message": "Аутфит не найден", "code": "outfit_not_found"}}` |
| `POST` | `/outfits` | Создание аутфита (сразу опубликован) | **Req:** Cookie: `session_id`<br>`{"items": ["item_id_1", "item_id_2"], "title": "Beach day", "description": "Casual look"}` — `items` min 1, дубли легальны; `title` ≤100, `description` ≤500, оба опциональны<br>**Res:** `201 {"message": "Аутфит создан", "id": "uuid"}`<br>**Error:** `404 item_not_found` (несуществующая вещь), `422` (пустой items) |
| `PUT` | `/outfits/{outfit_id}` | Редактирование подписи (**только** title/description) | **Req:** Cookie: `session_id`<br>`{"title": "...", "description": "..."}` — оба опциональны, `null`/отсутствие = не менять. **Состав не редактируется** — полная переделка = DELETE + POST (лайки не переносятся)<br>**Res:** `200 {"message": "Аутфит обновлен"}`<br>**Error:** `404 outfit_not_found`; `403 {"detail": {"message": "Нет прав на изменение аутфита", "code": "forbidden_to_change_outfit"}}` |
| `DELETE` | `/outfits/{outfit_id}` | Удаление аутфита (только автор) | **Req:** Cookie: `session_id`<br>**Res:** `200 {"message": "Аутфит удален"}`<br>**Error:** `404 outfit_not_found`; `403 forbidden_to_change_outfit` |
| `POST` | `/outfits/{outfit_id}/images` | Загрузка холстов поста (PNG из коллаж-редактора) | **Req:** Cookie: `session_id`; `multipart/form-data: images[]` — порядок файлов = порядок холстов (`order` 0,1,2…; order=0 — обложка)<br>**Res:** `201 {"message": "Изображения аутфита загружены", "images": ["url", ...]}`<br>**Error:** `404 outfit_not_found`; `403 forbidden_to_change_outfit`; **`409 {"detail": {"message": "Аутфит уже имеет изображения.", "code": "outfit_already_has_images"}}`** — повторная загрузка запрещена (ретрай = успех первой) |
| `POST` | `/outfits/{outfit_id}/like` | Лайк аутфита | **Req:** Cookie: `session_id`<br>**Res:** `200 {"message": "Добавлен лайк", "likes_count": 13}`<br>**Error:** `404 outfit_not_found`; `409 {"detail": {"message": "...", "code": "already_liked"}}` |
| `DELETE` | `/outfits/{outfit_id}/like` | Отмена лайка | **Req:** Cookie: `session_id`<br>**Res:** `200 {"message": "Лайк удален", "likes_count": 12}`<br>**Error:** `404 outfit_not_found`; `400 {"detail": {"message": "...", "code": "not_liked"}}` |
| `POST` | `/outfits/{outfit_id}/comments` | Добавить комментарий | **Req:** Cookie: `session_id`<br>`{"text": "Огонь образ!"}` — 1..500 символов<br>**Res:** `201 {"message": "Комментарий добавлен", "id": "uuid"}`<br>**Error:** `404 outfit_not_found`; `422` (пустой/длинный text) |
| `GET` | `/outfits/{outfit_id}/comments` | Комментарии поста (публично, новые сверху) | **Req:** `?page=1&limit=20`<br>**Res:** `200 {"comments": [{"id": "uuid", "text": "...", "created_at": "2026-07-12T15:30:00", "author": {"id": "uuid", "username": "...", "avatar_url": "..."}}], "page": 1}`<br>**Error:** `404 outfit_not_found` |
| `DELETE` | `/comments/{comment_id}` | Удалить комментарий (автор или админ) | **Req:** Cookie: `session_id`<br>**Res:** `200 {"message": "Комментарий удален"}`<br>**Error:** `404 {"detail": {"message": "Комментарий не найден", "code": "comment_not_found"}}`; `403 {"detail": {"message": "...", "code": "forbidden_to_delete_comment"}}` |

> **Примечания по разделу 4:**
> - Комментарии — плоские (без тредов), редактирования и лайков комментов нет (кандидаты — на созвон).
> - `sort=date` — единственная реализованная сортировка; формат `sort=popular` — открытый вопрос.
> - Рекомендуемый `limit` для тикток-ленты — 5–8 (карточки полные, тяжёлые).
> - Открытые вопросы лиду: `style` в карточке; `sort=popular`; семантика бана; конвенция обложки (`order` vs `is_main`).

## 5. Favorites & Interactions
Сохранение, избранное, стили, бренды

| Method | URL | Description | Request / Response |
|---|---|---|---|
| `GET` | `/favorites/styles` | Избранные стили | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"styles": [{"id": "uuid", "name": "minimalist"}]}` |
| `POST` | `/favorites/styles/{style_id}` | Добавить стиль в избранное | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"message": "Стиль добавлен"}`<br>**Error:** `409 {"detail": {"message": "Стиль уже в избранном", "code": "already_in_favorites"}}` |
| `GET` | `/favorites/brands` | Избранные бренды | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"brands": [{"id": "uuid", "name": "Nike", "image_url": "..."}]}` |
| `POST` | `/favorites/brands/{brand_id}` | Добавить бренд в избранное | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"message": "Бренд добавлен"}`<br>**Error:** `409 {"detail": {"message": "Бренд уже в избранном", "code": "already_in_favorites"}}` |
| `DELETE` | `/favorites/brands/{brand_id}` | Удалить бренд из избранного | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"message": "Бренд удален из избранного"}` |

> **Примечание:** избранное вещей (`favorites/items`) и **сохранённые аутфиты (`favorites/outfits`) вырезаны из MVP** — `is_saved` нигде не отдаётся. `favorites/styles` — на будущие недели. `favorites/brands` — реализовано.

## 6. Social & Follows
Подписки, списки подписчиков/подписок

| Method | URL | Description | Request / Response |
|---|---|---|---|
| `POST` | `/profile/follow/{username}` | Подписаться на пользователя | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"message": "Вы подписаны"}` |
| `DELETE` | `/profile/follow/{username}` | Отписаться | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"message": "Вы отписались"}` |
| `GET` | `/profile/{username}/followers` | Список подписчиков | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"users": [{"id": "uuid", "username": "maria", "avatar_url": "..."}]}` |
| `GET` | `/profile/{username}/following` | Список подписок | **Req:** Cookie: `session_id=abc123`<br>**Res:** `200 {"users": [...]}` |

## 7. Admin & Moderation
Управление контентом, пользователями, справочниками. Все эндпоинты требуют роль `admin` (иначе `403 {"detail": {"message": "Доступ запрещен", "code": "forbidden"}}`).

| Method | URL | Description | Request / Response |
|---|---|---|---|
| `POST` | `/admin/items` | Создание вещи (мультизагрузка фото) | **Req:** `multipart/form-data: name, description, category_id, style_id, brand_id, images[]`<br>**Res:** `201 {"message": "Вещь создана", "item_id": "uuid"}` |
| `PUT` | `/admin/items/{item_id}` | Обновление вещи (частичное) | **Req:** `{"name": "New Name", "category_id": "uuid"}` (все поля опциональные)<br>**Res:** `200 {"message": "Вещь обновлена"}`<br>**Error:** `404 {"detail": {"message": "Вещь не найдена", "code": "item_not_found"}}` |
| `DELETE` | `/admin/items/{item_id}` | Удаление вещи | **Req:** Cookie: `session_id=admin_session`<br>**Res:** `200 {"message": "Вещь удалена"}` |
| `POST` | `/admin/styles` | Создание стиля | **Req:** `{"name": "y2k"}`<br>**Res:** `201 {"message": "Стиль создан"}` |
| `PUT` | `/admin/styles/{style_id}` | Обновление стиля | **Req:** `{"name": "y2k_v2"}`<br>**Res:** `200 {"message": "Стиль обновлен"}` |
| `DELETE` | `/admin/styles/{style_id}` | Удаление стиля | **Req:** Cookie: `session_id=admin_session`<br>**Res:** `200 {"message": "Стиль удален"}` |
| `POST` | `/admin/categories` | Создание категории | **Req:** `{"name": "outerwear"}`<br>**Res:** `201 {"message": "Категория создана"}` |
| `DELETE` | `/admin/categories/{category_id}` | Удаление категории | **Req:** Cookie: `session_id=admin_session`<br>**Res:** `200 {"message": "Категория удалена"}` |
| `POST` | `/admin/brands` | Создание бренда (с логотипом) | **Req:** `multipart/form-data: name, logo` (logo опционально)<br>**Res:** `201 {"message": "Бренд создан"}` |
| `GET` | `/admin/users` | Список пользователей | **Req:** Cookie: `session_id=admin_session`<br>`?page=1&limit=20`<br>**Res:** `200 {"users": [{"id": "uuid", "username": "...", "email": "...", "role": "user", "is_blocked": false}], "total": 100, "page": 1}` |
| `POST` | `/admin/users/{user_id}/block` | Блокировка пользователя | **Req:** Cookie: `session_id=admin_session`<br>**Res:** `200 {"message": "Пользователь заблокирован"}` |
| `POST` | `/admin/users/{user_id}/unblock` | Разблокировка пользователя | **Req:** Cookie: `session_id=admin_session`<br>**Res:** `200 {"message": "Пользователь разблокирован"}` |
| `DELETE` | `/admin/users/{user_id}` | Удаление пользователя | **Req:** Cookie: `session_id=admin_session`<br>**Res:** `200 {"message": "Пользователь удален"}` |
| `DELETE` | `/admin/outfits/{outfit_id}` | Удаление публикации | **Req:** Cookie: `session_id=admin_session`<br>**Res:** `200 {"message": "Публикация удалена"}` |

> **Примечания по разделу 7:**
> - `POST /admin/brands` — через `multipart/form-data` с опциональным логотипом (`logo`), загружается в S3. Спека ранее показывала JSON без файла — обновлено по факту реализации (отдельное ТЗ требовало логотип).
> - `POST /admin/items` — поле `tags[]` из старой спеки не реализовано (нет в ER-диаграмме). Требует решения нужны ли теги.
> - `DELETE /admin/categories/{id}` — добавлен сверх изначальной спеки (была только создание) для симметрии со стилями.
> - `PUT /admin/items` фактически работает как частичное обновление (PATCH-семантика): обновляются только переданные поля. Не отправлять пустые строки в UUID-поля.
