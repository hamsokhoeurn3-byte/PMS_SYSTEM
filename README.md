# Hotel PMS — Installation, Configuration & System Flow

A hotel / property management system rebuilt on Laravel 11 + Inertia.js + React,
preserving the original UI and features 1:1.

---

## 1. Requirements

- PHP 8.2 or higher
- Composer 2.x
- Node.js 18+ and npm
- SQLite (default, built in) — or MySQL 8+ if you prefer

---

## 2. Installation

```bash
# 1. Install PHP dependencies
composer install

# 2. Install JS dependencies
npm install

# 3. Create your environment file
cp .env.example .env

# 4. Generate the application encryption key
php artisan key:generate

# 5. Run database migrations (creates all tables)
php artisan migrate

# 6. Seed demo data (users + sample bookings)
php artisan db:seed

# 7. Build the frontend
npm run build

# 8. Start the server
php artisan serve
```

Visit the URL it prints — usually **http://127.0.0.1:8000**.

For active development (auto-reload on file changes), run `npm run dev`
in a separate terminal instead of `npm run build`.

---

## 3. Configuration

All configuration lives in `.env`.

### Database

SQLite is preconfigured and needs no setup — `database/database.sqlite`
is already included empty.

To use MySQL instead:

```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=hotel_pms
DB_USERNAME=root
DB_PASSWORD=your_password
```

Then create that database yourself before running `php artisan migrate`.

### App settings

```env
APP_NAME="Hotel PMS"
APP_ENV=local          # change to "production" when deploying
APP_DEBUG=true         # set to false in production
APP_URL=http://localhost:8000
```

### Sessions / cache

Both default to the database driver — no extra setup needed after
`php artisan migrate` (it creates the `sessions` and `cache` tables).

---

## 4. Demo accounts

| Role | Email | Password |
|---|---|---|
| Super Admin | admin@hotelgroup.com | Admin@123 |
| General Manager | manager@hotelgroup.com | Manager@123 |
| Front Desk | frontdesk@hotelgroup.com | Desk@123 |
| Housekeeping | cleaning@hotelgroup.com | Clean@123 |
| Accountant | accountant@hotelgroup.com | Account@123 |
| Property Manager | property@hotelgroup.com | Prop@123 |
| Front Desk (deactivated) | tom.h@hotelgroup.com | Tom@123 |

---

## 5. How the system works (step by step)

**Step 1 — Request hits Laravel**
Every URL is served by a single route (`/`) since this is a single-page
app. Laravel checks the session: is someone logged in?

**Step 2 — Laravel hands off to React via Inertia**
Laravel renders one Inertia page (`Pages/App.tsx`) and passes the
current user (or `null`) as a prop. No separate API call is needed just
to know who's logged in.

**Step 3 — Login**
If no one is logged in, the React app shows `LoginPage.tsx`. Submitting
the form sends `POST /login` to `AuthController`, which checks the
email/password against the real `users` table. On success, Laravel
starts a real session and the page reloads with the authenticated user.

**Step 4 — Role-based navigation**
Once logged in, `AppShell` reads the user's role and only shows the
tabs that role is allowed to see (Bookings, Cleaning, Accounting, etc.)
— this mapping lives in `AuthContext.tsx` and mirrors
`User::ROLE_PERMISSIONS` on the backend.

**Step 5 — Using a module (e.g. Bookings)**
- On open, `Bookings.tsx` calls `GET /api/bookings` and displays the
  real rows from the database.
- Clicking **Add Booking** opens a form; submitting it calls
  `POST /api/bookings`, which validates and saves a new row, then adds
  it to the table without a page reload.
- Clicking the trash icon calls `DELETE /api/bookings/{id}` and removes
  that row.

**Step 6 — Other modules (Cleaning, Accounting, Messages, etc.)**
These still run exactly like the original UI — their data is generated
in the component itself, not fetched from a database. Nothing is broken;
they're simply not wired to a backend yet.

**Step 7 — Logout**
Clicking logout calls `POST /logout`, Laravel destroys the session, and
the page reloads back to the login screen.

---

## 6. Extending another module

Follow the same pattern used for Bookings:

1. `php artisan make:model Room -mfc` — creates a model, migration,
   factory, and controller together
2. Fill in the migration's columns to match that component's existing
   mock data shape
3. Add a route in `routes/api.php`:
```php
   Route::apiResource('rooms', RoomController::class);
```
4. In the component, replace the local mock array with an `axios.get()`
   call inside a `useEffect`, the same way `Bookings.tsx` does it
5. Wire the "add new" button to `axios.post()`

`app/Http/Controllers/Api/BookingController.php` and
`resources/js/Components/Bookings.tsx` are the reference implementation.

---

## 7. Project structure

```
app/Http/Controllers/AuthController.php         real login/logout
app/Http/Controllers/Api/BookingController.php   real Bookings CRUD
app/Models/User.php, Booking.php
database/migrations/                             users, bookings, sessions, cache
database/seeders/                                demo users + demo bookings
routes/web.php                                   the single SPA route + login/logout
routes/api.php                                   /api/bookings (auth-protected)
resources/js/Pages/App.tsx                       original App.tsx entry point
resources/js/Components/                         all original components, unchanged
resources/js/Components/Bookings.tsx             rewired to the real API
resources/js/Components/ui/                      original shadcn/radix primitives
resources/js/context/AuthContext.tsx             rewired for real session auth
resources/js/context/LanguageContext.tsx         unchanged (EN/JA/KH translations)
```