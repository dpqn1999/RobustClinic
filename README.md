# NhuBase

A small full-stack project for learning how a React app talks to a Node.js backend,
built with the structure most companies use.

- **client/** – React + Vite + TypeScript, React Router, axios
- **server/** – Node.js + Express + TypeScript, JWT authentication, SQLite or PostgreSQL
- **desktop/** – Electron window around the same client

## Run it

```bash
npm install     # once, in this folder (installs client and server)
npm run dev     # starts both
```

Open http://localhost:5173 and log in with `admin@example.com` / `Admin123!`,
or register a new (normal) user.

| What   | URL                   | Started by               |
| ------ | --------------------- | ------------------------ |
| Client | http://localhost:5173 | `npm run dev -w client`  |
| API    | http://localhost:4000 | `npm run dev -w server`  |

The database is one file, `server/data/app.db`. Delete it to start from zero.

## Choose the database

`DB_CLIENT` in `server/.env` picks the database. Restart the API after changing it.

| `DB_CLIENT`        | Also set       | To install                         | Good for                          |
| ------------------ | -------------- | ---------------------------------- | --------------------------------- |
| `sqlite` (default) | `DATABASE_FILE`| nothing                            | learning, one computer, small office |
| `postgres`         | `DATABASE_URL` | PostgreSQL, on the server computer | many users, real backups          |

To use PostgreSQL (only on the computer that runs the API):

```bash
winget install PostgreSQL.PostgreSQL.17
```

```sql
-- in psql, as the postgres user
CREATE USER nhubase WITH PASSWORD 'choose-a-password';
CREATE DATABASE nhubase OWNER nhubase;
```

```
# server/.env
DB_CLIENT=postgres
DATABASE_URL=postgres://nhubase:choose-a-password@localhost:5432/nhubase
```

The tables are created at the first start, like with SQLite. The two databases
do not share data: switching starts with an empty database (and a new seeded admin).

How it works: `server/src/db/database.ts` loads `sqlite.ts` or `postgres.ts`.
Both offer the same `queryAll` / `execute`, so the repositories contain one
SQL text that runs on both. Every database call is `async`, because PostgreSQL
answers over the network.

## The big picture

```
Browser (React)                    Vite dev server            Express API                    SQLite
───────────────                    ───────────────            ───────────                    ──────
page -> api/*.api.ts -> axios  ->  /api/*  (proxy)  ->  middlewares -> route -> controller
                                                                                  -> service
                                                                                      -> repository -> SQL
page <- state update   <- JSON <-                   <-  res.json(...)
```

The browser only ever talks to port 5173. Vite forwards everything that starts
with `/api` to Express (see `client/vite.config.ts`).

## Follow one request: "add a task"

Open these files in order. This is the whole workflow.

| #  | File                                         | What happens                                                    |
| -- | -------------------------------------------- | --------------------------------------------------------------- |
| 1  | `client/src/pages/TasksPage.tsx`             | You submit the form, `handleAdd` calls `tasksApi.create(title)` |
| 2  | `client/src/api/tasks.api.ts`                | `POST /api/tasks` with body `{ title }`                         |
| 3  | `client/src/api/http.ts`                     | Request interceptor adds `Authorization: Bearer <access token>` |
| 4  | `server/src/app.ts`                          | Global middlewares run, then `/api/tasks` goes to `taskRouter`  |
| 5  | `server/src/modules/tasks/task.routes.ts`    | `authenticate` -> `validate(createTaskSchema)` -> controller    |
| 6  | `server/src/middlewares/authenticate.ts`     | Checks the token, sets `req.user`                               |
| 7  | `server/src/middlewares/validate.ts`         | Checks the body with zod                                        |
| 8  | `server/src/modules/tasks/task.controller.ts`| Reads the request, calls the service, sends `201` + JSON        |
| 9  | `server/src/modules/tasks/task.service.ts`   | Business rules (who owns the task)                              |
| 10 | `server/src/modules/tasks/task.repository.ts`| Runs the `INSERT`, returns the new row as an object             |
| 11 | back in `TasksPage.tsx`                      | `setTasks(...)` -> React draws the new task                     |

If anything throws on the server, `server/src/middlewares/errorHandler.ts`
turns it into a JSON error with the right status code.

## The layers on the server

Each feature is a folder in `server/src/modules/` with the same files:

| File              | Job                                         | Knows about      |
| ----------------- | ------------------------------------------- | ---------------- |
| `*.routes.ts`     | URL + method -> which middlewares + handler | Express          |
| `*.schema.ts`     | Shape of the request body (zod)             | nothing          |
| `*.controller.ts` | Read request, call service, send response   | HTTP             |
| `*.service.ts`    | Business rules                              | no HTTP, no SQL  |
| `*.repository.ts` | Talk to the database                        | SQL              |

If you come from ASP.NET: routes + controller = Controller, service = Service,
repository = Repository / DbContext, middlewares = middleware pipeline, zod schema = DTO validation.

## How authentication works

There are two tokens:

|            | Access token                          | Refresh token                          |
| ---------- | ------------------------------------- | -------------------------------------- |
| What       | JWT (signed, contains user id + role) | Random string, saved (hashed) in DB    |
| Lives      | 15 minutes                            | 7 days                                 |
| Kept in    | A JavaScript variable (memory)        | `httpOnly` cookie (JS cannot read it)  |
| Used for   | Every API call (`Authorization` header) | Only `POST /api/auth/refresh`        |

1. **Login** – `POST /api/auth/login`. Server checks the password hash (bcrypt),
   answers with `{ user, accessToken }` and sets the refresh cookie.
   `AuthContext.tsx` saves the user, `http.ts` saves the token.
2. **Normal calls** – the interceptor in `http.ts` adds the access token.
   `authenticate.ts` verifies it on the server.
3. **Token expired** – the API answers `401`. The response interceptor in `http.ts`
   calls `/api/auth/refresh`, gets a new access token, and repeats the request.
4. **Page reload (F5)** – memory is empty, so `AuthContext.tsx` calls
   `/api/auth/refresh` at startup to restore the session from the cookie.
5. **Logout** – the refresh token is deleted from the DB and the cookie is cleared.

**Authentication vs authorization**

- `authenticate` = *who are you?* -> `401` if not logged in.
- `authorize('admin')` = *are you allowed?* -> `403` if the role is wrong
  (see `server/src/modules/users/user.routes.ts`).
- Ownership (*is this your task?*) is checked in `task.service.ts`.

## Routing on the client

`client/src/router.tsx` is the URL map. Guards in `client/src/auth/guards.tsx`:

| URL                   | Guard                       | Who can see it     |
| --------------------- | --------------------------- | ------------------ |
| `/login`, `/register` | `PublicOnly`                | Not logged in      |
| `/`                   | `RequireAuth`               | Any logged-in user |
| `/admin/users`        | `RequireAuth roles=[admin]` | Admin only         |

Client guards are only for a good user experience. The server checks everything
again, because a user can change any code that runs in their browser.

## API

| Method | URL                    | Auth   | Purpose                         |
| ------ | ---------------------- | ------ | ------------------------------- |
| POST   | `/api/auth/register`   | –      | Create account                  |
| POST   | `/api/auth/login`      | –      | Log in                          |
| POST   | `/api/auth/refresh`    | cookie | New access token                |
| POST   | `/api/auth/logout`     | cookie | Log out                         |
| GET    | `/api/auth/me`         | user   | Current user                    |
| GET    | `/api/tasks`           | user   | My tasks                        |
| POST   | `/api/tasks`           | user   | Create task                     |
| PATCH  | `/api/tasks/:id`       | user   | Change title / done             |
| DELETE | `/api/tasks/:id`       | user   | Delete task                     |
| GET    | `/api/users`           | admin  | All users                       |
| PATCH  | `/api/users/:id/role`  | admin  | Change a user's role            |

## Desktop app (Electron)

`desktop/` wraps the **same React code** in a desktop window. Nothing in
`client/` or `server/` is copied or rewritten.

```bash
npm run dev:desktop     # API + Vite + desktop window (hot reload works in the window)
npm run start:desktop   # builds the client, then runs the desktop app without Vite
```

| File                          | Job                                                                 |
| ----------------------------- | ------------------------------------------------------------------- |
| `desktop/src/main.ts`         | Main process (Node.js): creates the window and decides what to load |
| `desktop/src/localServer.ts`  | Serves `client/dist` and forwards `/api/*` to the API               |
| `desktop/src/preload.ts`      | Exposes `window.desktop` to the page                                |

How the window gets the app:

```
dev:    window -> http://localhost:5173 (Vite)      -> /api proxied to Express
start:  window -> http://127.0.0.1:<random port>    -> /api forwarded to Express
                  (localServer.ts, inside the app)
```

In both cases the page and `/api` share one address, so `baseURL: '/api'`,
the login cookie and React Router work exactly like on the website.

The React code can ask "am I running on desktop?" with `window.desktop`
(see the badge in `client/src/layouts/AppLayout.tsx`). Features that only a
desktop app can do (files, notifications, tray icon) are added the same way:
write them in the main process and expose one small function in `preload.ts`.

The API address for `start` mode comes from the `API_URL` environment
variable (default `http://localhost:4000`). To ship a real installer
(`.exe`), add `electron-builder`; that step is not set up here.

## One computer as the office server (LAN)

One computer runs the API and the database and uses the app itself. Every
other computer on the same network connects to it.

On the server computer:

```bash
npm run host
```

This builds the client, starts the API in host mode and opens the desktop
window. The terminal prints the address for the others, for example
`Other computers can open http://192.168.1.10:4000`. Closing the window does
not stop the server; press `Ctrl+C` in the terminal to stop it.

On the other computers, either:

- open that address in a **browser** (nothing to install), or
- run the desktop app against it: `npm run join -- --api=http://192.168.1.10:4000`

```
server computer                          other computers
───────────────                          ───────────────
Express :4000  <──────── network ──────  browser  http://192.168.1.10:4000
  ├─ /api/*   -> API + SQLite file       desktop  --api=http://192.168.1.10:4000
  └─ the rest -> client/dist (web app)
desktop window (uses the same server)
```

What host mode changes is small: `--serve-client` in `server/src/server.ts`
makes Express also serve `client/dist` (see step 3 in `server/src/app.ts`).

Before using this for real:

- **Firewall**: the first time, Windows asks whether Node.js may accept
  connections. Allow it for private networks, or the others cannot connect.
- **Fixed address**: give the server computer a fixed IP (or use its computer
  name), otherwise the address can change after a restart.
- **Admin password**: the seeded `admin@example.com` / `Admin123!` is known to
  anyone who reads this file. Change it in `server/src/db/seed.ts` before the
  first start.
- **Backups**: with SQLite all data is the single file `server/data/app.db` on
  that computer. With PostgreSQL use `pg_dump nhubase > backup.sql`.
- **Plain http**: traffic is not encrypted. Fine inside a trusted office
  network, not for the internet.

## Exercise: add your own feature

Adding a `notes` feature touches every layer once:

1. Add a `notes` table in `server/src/db/sqlite.ts` and `server/src/db/postgres.ts`.
2. Copy `server/src/modules/tasks/` to `notes/` and rename.
3. Register it in `server/src/app.ts`: `app.use('/api/notes', noteRouter)`.
4. Add `client/src/api/notes.api.ts` and a `Note` type in `client/src/types.ts`.
5. Add `client/src/pages/NotesPage.tsx`, a route in `router.tsx`, and a link in `AppLayout.tsx`.

## What a real company adds on top

This project keeps the structure but leaves out some tools to stay readable:

- **Database**: an ORM (Prisma, TypeORM) and migrations, instead of raw SQL and `CREATE TABLE IF NOT EXISTS`.
- **Data fetching on the client**: TanStack Query instead of `useEffect` + `useState`.
- **Forms**: React Hook Form + zod.
- **Tests**: Vitest / Jest, Supertest for the API, Playwright for the browser.
- **Build**: compile the server with `tsc` and run plain `node`; serve the client's `dist/` from a CDN or nginx.
- **Ops**: ESLint + Prettier, Docker, CI, structured logging (pino), API docs (OpenAPI/Swagger).
"# RobustClinic" 
