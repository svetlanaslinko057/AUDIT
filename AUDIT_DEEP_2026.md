# 🔍 Глибокий аудит системи Y-Store Marketplace

**Дата:** 2026-04-17
**Репозиторій:** https://github.com/thusuonghyc4k29-debug/Market
**Ціль аудиту:** знайти причини трьох конкретних проблем:
1. SMS-розсилка з Telegram-бота не працює
2. Оплата "накладеним платежем" (COD) — замовлення не формується (оплата карткою — працює)
3. При оформленні замовлення користувача "викидає з кабінету"
4. Бонус: глибокий огляд інтеграції Нова Пошта, адмінки, архітектури авторизації

---

## 📁 1. Загальна архітектура проекту

```
Frontend (React 19 + Tailwind + Radix UI)
   ↓ REST / cookies
Backend (FastAPI, 252 Python-файли, 4446 рядків у server.py + 40+ модулів)
   ↓
MongoDB (25 колекцій)
   +
Telegram Bot (aiogram 3.x) — окремий процес modules.bot.bot_app
   +
Зовнішні сервіси: Нова Пошта, WayForPay, Fondy, RozetkaPay, TurboSMS (заявлено, не підключено)
```

**Критична архітектурна проблема:** у системі співіснують **ДВІ паралельні системи авторизації**, **ТРИ версії API для замовлень** (`/api/orders`, `/api/v2/orders`, `/api/v3/*`) та **ТРИ реалізації Nova Poshta клієнта**. Це джерело більшості знайдених багів.

---

## 🔴 Проблема #1. SMS-розсилка з бота не працює

### Ланцюжок логіки
```
Telegram Bot → Broadcast Wizard (modules/bot/wizards/broadcast_wizard.py)
    └─ enqueue в колекцію db.notification_queue
Scheduler (modules/jobs/scheduler.py, кожні 30 сек)
    └─ NotificationsService.process_queue_once()  (modules/notifications/notifications_service.py)
         └─ TurboSMSProvider.send(to, text)       (modules/notifications/providers/sms_turbosms.py)
              └─ POST https://api.turbosms.ua/message/send.json
```

### 🎯 Кореневі причини

#### Причина 1.1 — SMS-провайдер завжди повертає MOCKED, але worker вважає успіхом
**Файл:** `backend/modules/notifications/providers/sms_turbosms.py`, рядки 10-14
```python
if not getattr(settings, 'TURBOSMS_TOKEN', None):
    logger.warning(f"TURBOSMS not configured, SMS to {to} MOCKED: {text[:50]}...")
    return {"status": "MOCKED", "to": to}   # ← НЕ кидає exception → NotificationsService.mark_sent
```
**Ефект:** бот звітує "розсилку поставлено в чергу", потім scheduler "відправляє" її, помічає як `SENT` — а реально жодне SMS не пішло.

#### Причина 1.2 — У Settings взагалі немає полів для TurboSMS
**Файл:** `backend/core/config.py`
Відсутні поля: `TURBOSMS_TOKEN`, `TURBOSMS_API_BASE`, `TURBOSMS_SENDER`.
З `extra="ignore"` в `Settings.Config` навіть якщо додати їх у `.env` — вони будуть тихо проігноровані.

#### Причина 1.3 — База `customers` порожня
Broadcast-мастер фільтрує аудиторію через `db.customers.find({...})`:
```python
customers = await self.customers.find(flt, {"_id": 0}).limit(5000).to_list(5000)
for c in customers:
    to = c.get("phone") if channel == "SMS" else c.get("email")
```
У свіжій системі `customers` = 0 записів → `inserted: 0` → "0 повідомлень у черзі".

#### Причина 1.4 — У розсилці немає каналу "Telegram"
`bot_keyboards.channel_kb()` пропонує тільки **SMS** та **EMAIL**. Навіть якщо налаштований сам бот — через нього розсилати клієнтам не можна без зовнішнього SMS/Email.

### ✅ Що потрібно зробити

1. **Додати поля у `core/config.py`:**
   ```python
   TURBOSMS_TOKEN: str = ""
   TURBOSMS_API_BASE: str = "https://api.turbosms.ua/message"
   TURBOSMS_SENDER: str = "YStore"
   SMTP_HOST: str = ""
   SMTP_PORT: str = "587"
   SMTP_USER: str = ""
   SMTP_PASS: str = ""
   EMAIL_FROM: str = ""
   ```
2. **Виправити mock-поведінку SMS-провайдера:** у dev-режимі MOCKED робити `status="MOCKED"` + НЕ помічати як SENT (або помічати як `MOCKED`), щоб у адмінці було видно що реально не пішло.
3. **Отримати TurboSMS токен:** https://turbosms.ua → Особистий кабінет → API → згенерувати Bearer-токен + зареєструвати alpha-name відправника.
4. **Заповнити колекцію `customers`** (seed-скрипт або синхронізація з `db.users` + `db.orders`).
5. **Додати канал Telegram у розсилку** (не потребує зовнішніх сервісів): для клієнтів у яких `telegram_chat_id` — слати через вже підключеного бота.

---

## 🔴 Проблема #2. Оплата накладеним платежем — замовлення не формується

### Ланцюжок логіки
```
Frontend CheckoutV3.jsx → POST /api/v2/orders/create
Backend  modules/orders/orders_v2_routes.py → create_order_v2()
         └─ для кожного item: перевірка product.stock_level >= quantity
         └─ insert в db.orders
```

### 🎯 Корeнева причина

**Файл:** `backend/modules/orders/orders_v2_routes.py`, рядки 121-125
```python
if product.get("stock_level", 0) < item.quantity:    # ← default = 0!
    raise HTTPException(
        status_code=400,
        detail=f"Insufficient stock for {product['title']}"
    )
```
Якщо товар створений через `POST /api/products` або інший модуль без явного `stock_level`, поле відсутнє → `get("stock_level", 0) == 0` → `0 < 1` → **400 "Insufficient stock"** → замовлення не створюється.

### Чому оплата карткою "працює"?
На тому ж ендпоінті перевірка `stock_level` спрацьовує однаково для card і cash_on_delivery. Але при card-оплаті фронтенд (`CheckoutV3.jsx`, рядок 362) **очікує** що бекенд поверне `payment_url` або `form_data` → ініціює редірект на `secure.wayforpay.com`. Реально картковий потік у вас скоріше за все йде **через окремий ендпоінт** `/api/v2/payments/wayforpay/create` (`modules/payments/wayforpay_routes.py`), який **не перевіряє stock_level** — створює платіжну сесію напряму, а замовлення формується пізніше у вебхуці `webhook()` при статусі `PAID`.

Таким чином:
- **Картка** → WayForPay → webhook → оновлення/створення order (обходить stock-check)
- **COD** → `/api/v2/orders/create` → stock-check → 400 → замовлення немає

### Додаткова проблема — розбіг форматів payment_method
| Endpoint | Очікує |
|----------|--------|
| `modules/orders/routes.py` (v3 `/api/orders`) | `payment_method: str = "cash"` |
| `modules/orders/orders_v2_routes.py` (`/api/v2/orders/create`) | `payment_method: str = "cash_on_delivery"` |
| `server.py:1795` (старий POST) | `"cash_on_delivery"` |
| Frontend `CheckoutV3.jsx` | надсилає `"cash_on_delivery"` |

Три різних значення в одній кодбазі — джерело "тихих" багів при будь-якому рефакторингу.

### ✅ Що потрібно зробити

1. **Виправити stock-check** у `orders_v2_routes.py`:
   ```python
   stock = product.get("stock_level")
   if stock is not None and stock < item.quantity:
       raise HTTPException(400, ...)
   ```
   АБО: при створенні товару завжди сетити `stock_level` (дефолт 100).

2. **Уніфікувати payment_method** — одне значення в усій системі (`cash_on_delivery`), з normalize-функцією на вході.

3. **Для COD повертати `payment_url=None` і залишати `status="new"`** (вже робиться), а не "pending" — це різні речі для state-machine.

---

## 🔴 Проблема #3. Користувача викидає з кабінету при оформленні замовлення

### 🎯 Коренева причина — агресивний 401-interceptor в Axios

**Файл:** `frontend/src/utils/api.js`, рядки 34-51
```js
api.interceptors.response.use(
  (response) => response,
  (error) => {
    const isAuthEndpoint = error.config?.url?.includes('/auth/login') || ...;
    const isAuthPage = window.location.pathname === '/login' || ...;
    const isGuestEndpoint = error.config?.url?.includes('/cart');

    if (error.response?.status === 401 && !isAuthEndpoint && !isAuthPage && !isGuestEndpoint) {
      localStorage.removeItem('token');
      localStorage.removeItem('user');
      window.location.href = '/login';   // ← примусовий редірект на /login
    }
    return Promise.reject(error);
  }
);
```

### Дві системи авторизації, що конфліктують

У проекті співіснують:

**A. Legacy JWT auth** (`/api/auth/login` → `access_token` у `localStorage.token`)
- `api.interceptors.request.use` додає `Authorization: Bearer ${localStorage.token}`
- Бекенд перевіряє через `core/security.get_current_user` (JWT)

**B. V2 Session auth** (`/api/v2/auth/*` → cookie `session_token`, httpOnly)
- Google OAuth і Guest Checkout створюють **тільки cookie**, `localStorage.token` = `null`
- Бекенд перевіряє через `get_optional_user` (сесія в `db.user_sessions`)

### Ланцюжок падіння при COD-замовленні

1. Користувач увійшов через **Google OAuth** → в `localStorage.token` немає, але є cookie `session_token`.
2. На CheckoutV3 натискає "Оформити" → `axios.post('/api/v2/orders/create', payload, { withCredentials: true })` — працює (cookies ОК).
3. Після успіху код викликає `clearCart()` з CartContext → `api.delete('/cart')` (це **v1-endpoint**).
4. В `api.interceptors.request.use` `localStorage.getItem('token') === null` → Bearer не додається.
5. Backend `/api/cart` очікує JWT → **401 Unauthorized**.
6. Interceptor бачить 401, URL містить `/cart` → `isGuestEndpoint = true` → **АЛЕ** ключова умова: перевіряє `.includes('/cart')` — це спрацьовує для `/cart`. Але для інших ручок (`/orders/my`, `/admin/*`, `/products` по favorite тощо) `isGuestEndpoint = false` → `localStorage.removeItem('token')` + `window.location.href = '/login'`.
7. Після редіректу `checkAuth` знову смикає `/api/v2/auth/me` → якщо session cookie ще валідне → повернеться, але localStorage очищений → `token = null` → вигляд "розлогінили".

### Другий сценарій — expired session
- V2 session має expiry 30 днів. Якщо `db.user_sessions` зачистили або cookie протухла — `/api/v2/auth/me` поверне 401 → fallback у `checkAuth` шукає `localStorage.token` (якого немає) → `user = null`.

### ✅ Що потрібно зробити

1. **Виправити interceptor** — НЕ редіректити на 401 для ендпоінтів замовлень/кабінету; робити м'який редірект тільки якщо немає ні JWT, ні session:
   ```js
   if (error.response?.status === 401 && !isAuthEndpoint && !isAuthPage) {
       // ТІЛЬКИ очистити токен, дати ui показати повідомлення
       // НЕ робити window.location.href
   }
   ```
2. **Додати session_token у Bearer** — щоб v1-ендпоінти теж розуміли V2-сесії:
   ```js
   const token = localStorage.getItem('token') || localStorage.getItem('session_token');
   if (token) config.headers.Authorization = `Bearer ${token}`;
   ```
3. **На бекенді в `core/security.get_current_user`** — спочатку пробувати як JWT, якщо не вдалось — шукати як `session_token` у `db.user_sessions`.

---

## 🟡 Проблема #4. Nova Poshta — три реалізації, мовчазні помилки

### Дублі коду

| Файл | Призначення | Технологія |
|------|-------------|------------|
| `backend/novaposhta_service.py` | Старий синхронний сервіс | `requests` |
| `backend/modules/delivery/np/np_client.py` | Новий async клієнт | `httpx` |
| `backend/modules/delivery/routes.py` | `/api/delivery/*` ендпоінти | `aiohttp` |
| `backend/modules/delivery/routes_v2.py` | Ще одна версія `/api/v2/delivery/*` | інше |
| `backend/modules/delivery/novaposhta_ttn_service.py` | TTN-сервіс окремо | mix |

### Поведінка без API-ключа

#### `modules/delivery/routes.py` (стр. 15-29)
```python
async def np_request(...):
    ...
    if not data.get("success"):
        return []   # ← мовчки повертає порожній масив
```
**Ефект:** фронт запитує міста — отримує `[]`, нічого не знаходить, але помилки ніде не видно. Користувач думає, що "Нова Пошта не працює". Потрібно хоча б логувати `data.get("errors")`.

#### `modules/delivery/np/np_client.py` (стр. 47-48)
```python
if not data.get("success"):
    logger.warning(f"NP API error: {data.get('errors', [])} | warnings: ...")
```
Логує, але повертає `{"success": false, ...}` — це вже краще, але далі по ланцюжку потрібно обробити правильно.

#### `backend/novaposhta_service.py` (стр. 23-24)
```python
self.api_key = os.environ.get('NOVAPOSHTA_API_KEY', '')
if not self.api_key:
    logger.warning("Nova Poshta API key not configured - using limited access")
```
Відправляє пустий `apiKey` — NP API відповідає `"API key is not provided"` → `success=False` → порожній результат.

### TTN (накладні) — не створюються навіть з ключем

**Файл:** `backend/core/config.py` (стр. 29-36)
```python
NP_SENDER_CITY_REF: str = ""
NP_SENDER_WAREHOUSE_REF: str = ""
NP_SENDER_COUNTERPARTY_REF: str = ""
NP_SENDER_CONTACT_REF: str = ""
NP_SENDER_PHONE: str = ""
NP_SENDER_NAME: str = "Y-Store"
```
**Без цих референсів відправника TTN Wizard у боті впаде** на кроці генерації — NP API повертає "Sender not defined".

### ✅ Що потрібно зробити

1. Отримати `NOVAPOSHTA_API_KEY` на https://my.novaposhta.ua/settings/index#apikeys
2. У кабінеті НП знайти свої refs (контрагент/контакт/відділення-відправник) — або використати `NP_SENDER_SETUP` хелпер-скрипт з `modules/delivery/np/np_sender_setup.py` для автоматичного заповнення.
3. Обрати **ОДНУ** реалізацію клієнта (рекомендую `modules/delivery/np/np_client.py` — async, логування) і видалити інші.
4. Додати коректний error-response: замість `return []` повертати `HTTPException(503, "Nova Poshta unavailable: {errors}")`.

---

## 🟡 Проблема #5. Адмінка — немає ручного створення замовлення

**Розслідування:**
- У `frontend/src/components/admin/OrdersAnalytics.js` є тільки перегляд і фільтрація замовлень.
- У `backend/modules/admin/routes.py` немає endpoint `POST /orders` для ручного створення.
- Адмін не може оформити замовлення від імені клієнта (наприклад, коли клієнт дзвонить по телефону).

### ✅ Що додати
1. **Backend:** `POST /api/admin/orders/manual` — приймає `customer.phone`, `items[]`, `payment_method`, без stock-check, створює `db.orders` і `db.customers`.
2. **Frontend:** новий компонент `components/admin/ManualOrderCreate.jsx` + таб у `AdminPanelRefactored.js`.
3. **Зв'язок з CRM:** автоматично створювати картку клієнта, якщо телефон новий.

---

## 🟡 Проблема #6. Дублі та хвости коду (з AUDIT_REPORT.md репозиторію)

Підтверджую знахідки попереднього аудиту:

### Backend (server.py)
- **8 моделей дублюються** (рядки 385-454 і 635-715): `CustomerNote`, `CRMTask`, `CRMTaskUpdate`, `Lead`, `LeadUpdate` тощо.
- **Хвости коду** поза класами (рядки 565-567, 583-587): осиротілі поля `active`, `notes`, `assigned_to`.
- **39 помилок ruff:** 25× `F811` (переозначення), 8× `F821` (`request`, `current_user`, `categories`, `category_dict` — **неіснуючі змінні у боді функцій**), 1× `E722` (bare except), 1× `F541` (пуста f-string).
- **Критичні (undefined):**
  - Рядок 2900-2907: `request` не визначено (падає при виклику)
  - Рядок 2933: `current_user` не визначено
  - Рядок 3695: `categories` не визначено
  - Рядок 3806: `category_dict` не визначено

### Frontend
| Файл | Статус |
|------|--------|
| `pages/AdminPanel.js` (489 рядків, 22 inline-табу) | ❌ застаріла версія |
| `pages/AdminPanelRefactored.js` (394 рядки, lazy load) | ✅ актуальна, `/admin` |
| `pages/AdminPanelV2.js` (15 рядків) | ❓ не підключена |
| `components/admin2/` (AdminShell + модулі) | ❌ не використовується |
| `components/admin/shared/ProductAttributesEditor.jsx` | ✅ канонічна |
| `components/admin/catalog/ProductAttributesEditor.jsx` | ❌ дубль |
| 6 компонентів (`GuardIncidents`, `CustomerTimeline`, ...) | ❓ не імпортуються |

---

## 🟡 Проблема #7. MongoDB — порожня БД

```
Products total: 0
Customers: 0
Notification queue: 0
```

Є seed-скрипти, але вони **не виконані**:
- `backend/seed_categories.py`
- `backend/seed_categories_v2.py`
- `backend/seed_products.py`
- `backend/scripts/seed_categories.py`

Без даних:
- Каталог пустий → не можна покласти в кошик → не можна оформити замовлення
- Broadcast-wizard знайде 0 клієнтів
- Аналітика в адмінці показує нулі

### ✅ Що зробити
```bash
cd /app/backend
python seed_categories_v2.py
python seed_products.py
```

---

## 🟡 Проблема #8. Telegram Bot — chat_id адмінів порожні

```
📬 Admin chat_ids: []
👤 Admin user_ids: []
```

**Файл:** `modules/bot/bot_app.py` (стартові логи).

Щоб бот **@p_fomo_bot** слав алерти адміну — потрібно:
1. Написати `/start` боту в Telegram.
2. Виконати `/debug` — бот покаже свій `chat_id`.
3. Додати `chat_id` у колекцію `bot_settings`:
   ```python
   await db.bot_settings.update_one(
       {"setting_type": "admin_chat_ids"},
       {"$set": {"chat_ids": [YOUR_CHAT_ID]}},
       upsert=True
   )
   ```
   АБО через `/setadmin` handler у боті (якщо реалізовано).

Без цього алерти про нові замовлення, проблеми з оплатою, VIP-апгрейди **нікому не надсилаються**.

---

## 📊 Матриця пріоритетів

| # | Проблема | Файл | Пріоритет | Хто виправить | Потрібні ключі |
|---|----------|------|-----------|---------------|----------------|
| 1 | Stock-check блокує COD | `modules/orders/orders_v2_routes.py:121` | 🔴 P0 | код | — |
| 2 | 401-interceptor викидає з кабінету | `frontend/src/utils/api.js:44` | 🔴 P0 | код | — |
| 3 | SMS-провайдер повертає SENT при MOCKED | `providers/sms_turbosms.py:14` | 🔴 P0 | код | — |
| 4 | Дві системи auth без моста | `AuthContext.js` + `utils/api.js` | 🔴 P0 | код | — |
| 5 | TURBOSMS поля відсутні у Settings | `core/config.py` | 🟡 P1 | код + ключ | TurboSMS token |
| 6 | Розбіг payment_method | orders_v2 vs orders/routes | 🟡 P1 | код | — |
| 7 | Nova Poshta мовчазні 500 | `modules/delivery/routes.py:15` | 🟡 P1 | код | — |
| 8 | NP_SENDER_* порожні → TTN не створити | `.env` | 🟡 P1 | ключі | NP API key + refs |
| 9 | Admin chat_ids порожні | bot_settings | 🟡 P1 | ручна дія | — |
| 10 | 8 моделей дублюються у server.py | `server.py:635-715` | 🟢 P2 | код | — |
| 11 | 39 ruff-помилок | `server.py` | 🟢 P2 | код | — |
| 12 | Немає ручного створення заказу в адмінці | новий код | 🟢 P2 | фіча | — |
| 13 | 3 версії адмін-панелі | frontend | 🟢 P2 | cleanup | — |
| 14 | 3 реалізації Nova Poshta | `backend/` | 🟢 P2 | cleanup | — |
| 15 | БД порожня (0 товарів) | MongoDB | 🟢 P2 | seed | — |

---

## 🛠 План дій (запропонований порядок)

### Фаза 1 — Термінові фікси (30-40 хв, без зовнішніх ключів)
1. Виправити stock-check у `orders_v2_routes.py`
2. Виправити 401-interceptor у `utils/api.js`
3. Додати fallback у `get_current_user` — перевіряти `session_token` з БД
4. Виправити TurboSMS — не помічати MOCKED як SENT
5. Уніфікувати `payment_method` (normalize-функція)

### Фаза 2 — Інтеграції (потрібні ключі від власника)
6. Зареєструватись на **TurboSMS** → отримати Bearer-токен
7. Отримати **NOVAPOSHTA_API_KEY** + налаштувати NP_SENDER_*
8. Отримати **WayForPay merchant credentials** (для реальних платежів)

### Фаза 3 — Дані та налаштування
9. Виконати seed: `python seed_categories_v2.py && python seed_products.py`
10. Зареєструвати свій `chat_id` у `bot_settings.admin_chat_ids`

### Фаза 4 — Чистка коду (можна робити пізніше)
11. Видалити 8 дублів моделей з `server.py` (рядки 635-715)
12. Видалити `/pages/AdminPanel.js`, `/pages/AdminPanelV2.js`, `/components/admin2/`
13. Видалити 2 зайві NP-клієнти, залишити тільки `modules/delivery/np/np_client.py`
14. Виправити 39 ruff-помилок у `server.py` (4 critical undefined змінних)

### Фаза 5 — Нові фічі
15. Ручне створення замовлення в адмінці (`POST /api/admin/orders/manual` + UI)
16. Канал "Telegram" у broadcast-wizard (розсилка через самого бота)
17. CRM-автосинхронізація `users` → `customers`

---

## 📎 Додатки

### Файли, які потребують змін у Фазі 1
- `backend/modules/orders/orders_v2_routes.py` (рядки 121-125)
- `frontend/src/utils/api.js` (рядки 34-51)
- `backend/core/security.py` (додати session-auth fallback у `get_current_user`)
- `backend/modules/notifications/providers/sms_turbosms.py` (рядки 10-14)
- `backend/core/config.py` (додати TurboSMS, SMTP поля)

### Корисні посилання
- TurboSMS API: https://turbosms.ua/api.html
- Nova Poshta API: https://developers.novaposhta.ua/
- WayForPay docs: https://wiki.wayforpay.com/
- aiogram 3.x: https://docs.aiogram.dev/

---

**Підготував:** AI-агент Emergent E1
**Версія документа:** 1.0
**Контакт для питань:** через адмінку Y-Store або @p_fomo_bot
