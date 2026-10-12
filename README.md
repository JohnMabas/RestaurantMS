# Restaurant Management System

A full-stack restaurant management system built with Node.js/Express (backend) and Next.js (frontend), using PostgreSQL and Sequelize ORM.

> **Security note:** Frontend route guards are for UX only — hiding buttons and redirecting to `/login`. The real security is the Express API middleware (JWT verification, role checks). A user could bypass the frontend entirely and call the API directly; the API will still enforce all rules.

---

## Prerequisites

- Node.js 18+
- PostgreSQL 14+

---

## Quick Setup

### 1. Create the database

```bash
psql -U postgres -c "CREATE DATABASE restaurant_db;"
```

### 2. Backend setup

```bash
cd backend
cp .env.example .env
```

Edit `.env` and fill in:
```
DB_PASSWORD=your_postgres_password
JWT_SECRET=<run: node -e "console.log(require('crypto').randomBytes(48).toString('hex'))">
```

```bash
npm install
npx sequelize-cli db:migrate
npx sequelize-cli db:seed:all
npm run dev
```

Backend runs on **http://localhost:5000**

### 3. Frontend setup

```bash
cd frontend
cp .env.local.example .env.local
npm install
npm run dev
```

Frontend runs on **http://localhost:3000**

---

## Demo Accounts (demo use only — never use these in production)

| Role     | Email                      | Password      |
|----------|----------------------------|---------------|
| Admin    | admin@restaurant.test      | Admin123!     |
| Staff    | staff@restaurant.test      | Staff123!     |
| Customer | alice@restaurant.test      | Customer123!  |
| Customer | bob@restaurant.test        | Customer123!  |

---

## API Endpoints

Base path: `http://localhost:5000/api`

### Health
| Method | Path         | Auth | Description |
|--------|--------------|------|-------------|
| GET    | /health      | None | Check DB connection |

### Auth
| Method | Path         | Auth     | Description |
|--------|--------------|----------|-------------|
| POST   | /auth/register | None  | Register (always creates customer) |
| POST   | /auth/login    | None  | Login, returns JWT |
| GET    | /auth/me       | Bearer | Get current user |
| PUT    | /auth/me       | Bearer | Update name/password |

### Categories
| Method | Path              | Auth  | Description |
|--------|-------------------|-------|-------------|
| GET    | /categories       | None  | List all |
| GET    | /categories/:id   | None  | Get one |
| POST   | /categories       | Admin | Create |
| PUT    | /categories/:id   | Admin | Update |
| DELETE | /categories/:id   | Admin | Delete (409 if items exist) |

### Menu Items
| Method | Path               | Auth         | Description |
|--------|--------------------|--------------|-------------|
| GET    | /menu-items        | None         | List (optional ?category_id=) |
| GET    | /menu-items/:id    | None         | Get one |
| POST   | /menu-items        | Admin        | Create |
| PUT    | /menu-items/:id    | Staff/Admin  | Update (staff: isAvailable only) |
| DELETE | /menu-items/:id    | Admin        | Delete |

### Orders
| Method | Path                    | Auth        | Description |
|--------|-------------------------|-------------|-------------|
| GET    | /orders                 | Bearer      | List (customers: own only) |
| GET    | /orders/:id             | Bearer      | Get one (customers: own only) |
| POST   | /orders                 | Bearer      | Create order |
| PUT    | /orders/:id             | Bearer      | Update status / items |
| DELETE | /orders/:id             | Admin       | Delete (cascades items + history) |
| GET    | /orders/:id/tracking    | Bearer      | Live tracking data |

### Users
| Method | Path                | Auth        | Description |
|--------|---------------------|-------------|-------------|
| GET    | /users              | Admin       | List all users |
| GET    | /users/customers    | Staff/Admin | List customers (for order dropdown) |
| GET    | /users/:id          | Admin       | Get one |
| POST   | /users              | Admin       | Create with any role |
| PUT    | /users/:id          | Admin       | Update |
| DELETE | /users/:id          | Admin       | Delete (409 if has orders) |

### Example: Create and track an order

```bash
# 1. Login
TOKEN=$(curl -s -X POST http://localhost:5000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@restaurant.test","password":"Customer123!"}' \
  | jq -r .data.token)

# 2. Place an order
ORDER_ID=$(curl -s -X POST http://localhost:5000/api/orders \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"items":[{"menu_item_id":1,"quantity":2},{"menu_item_id":7,"quantity":1}]}' \
  | jq .data.order.id)

# 3. Track it
curl -s http://localhost:5000/api/orders/$ORDER_ID/tracking \
  -H "Authorization: Bearer $TOKEN" | jq .
```

### Example tracking response

```json
{
  "success": true,
  "data": {
    "order_id": 4,
    "status": "preparing",
    "status_label": "Preparing",
    "status_description": "The kitchen is preparing your food.",
    "is_terminal": false,
    "is_cancelled": false,
    "estimated_ready_at": "2026-10-12T14:35:00.000Z",
    "placed_at": "2026-10-12T14:15:00.000Z",
    "total_amount": 19.5,
    "steps": [
      { "status": "pending",   "label": "Order Placed", "state": "completed", "at": "2026-10-12T14:15:00.000Z" },
      { "status": "preparing", "label": "Preparing",    "state": "current",   "at": "2026-10-12T14:17:00.000Z" },
      { "status": "ready",     "label": "Ready",        "state": "upcoming",  "at": null },
      { "status": "served",    "label": "Served",       "state": "upcoming",  "at": null },
      { "status": "paid",      "label": "Paid",         "state": "upcoming",  "at": null }
    ],
    "history": [
      { "status": "pending",   "note": "Order placed",    "changed_by_name": "Alice",          "at": "..." },
      { "status": "preparing", "note": "Kitchen started", "changed_by_name": "Restaurant Staff","at": "..." }
    ],
    "items": [
      { "name": "Classic Burger", "quantity": 2, "unit_price": 8.5, "line_total": 17 },
      { "name": "Coke",           "quantity": 1, "unit_price": 2.5, "line_total": 2.5 }
    ],
    "server_time": "2026-10-12T14:20:00.000Z"
  }
}
```

---

## Order Status State Machine

```
pending ──► preparing ──► ready ──► served ──► paid
   │            │
   └────────────┴──► cancelled
```

- `paid` and `cancelled` are terminal — no further changes
- Skipping steps or moving backward returns `400`
- All valid transitions are defined in `backend/src/utils/orderStatus.js`

---

## Permission Matrix

| Action                          | Customer      | Staff         | Admin |
|---------------------------------|---------------|---------------|-------|
| Browse menu / categories        | ✅            | ✅            | ✅    |
| Create order                    | ✅ (own)      | ✅            | ✅    |
| View orders                     | Own only      | All           | All   |
| Cancel order                    | Own, pending  | pending/prep  | ✅    |
| Advance order status            | ❌            | ✅            | ✅    |
| Toggle menu item availability   | ❌            | ✅            | ✅    |
| Create/edit/delete menu items   | ❌            | ❌            | ✅    |
| Manage categories               | ❌            | ❌            | ✅    |
| Manage users                    | ❌            | ❌            | ✅    |

---

## ER Diagram

```
users
  id, name, email, password_hash, role, created_at

categories
  id, name, description

menu_items
  id, name, description, price, category_id (FK→categories), is_available, created_at

orders
  id, user_id (FK→users), status, total_amount, estimated_ready_at, created_at

order_items
  id, order_id (FK→orders CASCADE), menu_item_id (FK→menu_items RESTRICT),
  quantity, unit_price
  UNIQUE(order_id, menu_item_id)

order_status_history
  id, order_id (FK→orders CASCADE), status, changed_by (FK→users SET NULL),
  note, created_at
  INDEX(order_id, created_at)
```

**Relationships:**
- `categories` 1──* `menu_items`
- `users` 1──* `orders`
- `orders` 1──* `order_items` *──1 `menu_items` (many-to-many through `order_items`)
- `orders` 1──* `order_status_history` *──1 `users` (who changed it)

---

## Data Flow

```
PostgreSQL
  ↕ Sequelize ORM (models, migrations, associations)
Express REST API (JWT auth + role checks + status-machine validation)
  ↕ HTTP + Authorization: Bearer <token>
Next.js frontend (client components, fetch, localStorage token)
  ↕
User's browser
```

---

## Authentication Flow

```
1. POST /api/auth/login { email, password }
2. Server: find user (withPassword scope), bcrypt.compare(password, hash)
3. Server: sign JWT { id, role } with JWT_SECRET
4. Client: store token in localStorage
5. Client: attach Authorization: Bearer <token> to every API request
6. Server middleware (authenticate.js): verify token → reload user from DB → attach req.user
7. Server middleware (authorize.js): check req.user.role against allowed roles
```

---

## Why a Separate History Table for Tracking?

Instead of just storing `status` in the `orders` table, we have `order_status_history`:

1. **Audit trail** — we know exactly when each status change happened and who made it
2. **Timeline display** — the tracking page shows the full journey: pending → preparing → ready → served
3. **Duration analysis** — SQL can calculate how long the order spent in each status (see `database/queries.sql`)
4. **Why one transaction?** — The status update on `orders` and the history insert happen in one transaction. If either fails, both roll back. An order can never have a mismatched `status` and history.

---

## Why Polling Instead of WebSockets?

- Polling (every 10 seconds) is simpler to implement, works with standard HTTP auth headers, and is debuggable in the Network tab
- For this use case (order updates every 15–30 seconds), polling is more than adequate
- Upgrade path: replace `usePolling` with a WebSocket connection or Server-Sent Events (SSE) — the backend tracking endpoint already returns the right data shape

---

## Validation Layers

Three layers protect data integrity:

1. **Validation middleware** (`middleware/validation.js`) — runs first, gives friendly 400 messages before any DB call
2. **Sequelize model validators** — runs on `.create()` / `.update()`, provides model-level checks
3. **Database CHECK constraints** — last line of defence, fires even if someone bypasses the ORM entirely

---

## Security Notes

- Passwords are hashed with bcrypt (cost 12) — never stored as plain text
- JWT payload contains only `{ id, role }` — no sensitive data
- `authenticate.js` reloads the user from the DB on every request — a deleted/demoted user loses access immediately
- Login returns the same error for wrong email AND wrong password (prevents user enumeration)
- Role is hardcoded to `customer` at registration — cannot be chosen by the client
- `user_id` in orders comes from the JWT, never the request body (for customers)
- **localStorage tokens** are readable by JavaScript on the page. An XSS vulnerability could steal them. `httpOnly` cookies are safer but require additional CORS/CSRF setup — that's the tradeoff for this project's simplicity.

---

## Useful Commands

```bash
# Migrations
npx sequelize-cli db:migrate              # apply all pending migrations
npx sequelize-cli db:migrate:undo         # undo the last migration
npx sequelize-cli db:migrate:undo:all     # undo all migrations

# Seeders
npx sequelize-cli db:seed:all             # load demo data
npx sequelize-cli db:seed:undo:all        # remove demo data

# Full reset
npm run db:reset                          # undo:all + migrate + seed:all

# Enable SQL logging (shows generated SQL)
DB_LOGGING=true npm run dev
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `Connection refused` | Make sure PostgreSQL is running: `sudo service postgresql start` |
| `CORS error` | Check `FRONTEND_URL` in `.env` matches exactly `http://localhost:3000` |
| `401 Unauthorized` | Token expired or missing `Bearer ` prefix. Log out and back in. |
| `relation does not exist` | Run `npx sequelize-cli db:migrate` — tables haven't been created yet |
| DECIMAL values are strings | Use `parseFloat()` — `pg` returns NUMERIC/DECIMAL as strings in Node.js |
| `JWT_SECRET is not set` | Copy `.env.example` to `.env` and fill in `JWT_SECRET` |
| Tracking page not updating | Check browser Network tab — the polling request should fire every 10s. If order is paid/cancelled, polling stops intentionally. |
| `Invalid status transition` | You're trying to skip a step or move backward. Check `orderStatus.js` for valid transitions. |
| Forgot `transaction: t` | If an order is created but items are missing, this is the cause. Always pass `{ transaction: t }`. |
| `alias mismatch in include` | The `as:` in the include must exactly match the `as:` in the association definition in `models/index.js` |
| New ENUM value not showing | Run the migration that adds it: `npx sequelize-cli db:migrate` |
# RestaurantMS
