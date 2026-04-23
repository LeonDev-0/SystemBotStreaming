# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

WhatsApp Bot Management System — a full-stack web app for managing WhatsApp bots with multi-user account support, automated message rules, and demo account distribution.

- **Backend**: Node.js + Express 5 + TypeScript (`registro/`)
- **Frontend**: React 19 + Vite + Tailwind CSS 4 (`velt/`)
- **Database**: SQLite via Prisma ORM (better-sqlite3 adapter)
- **Deployment**: Docker Compose (two services behind Nginx)

## Commands

### Running the full stack
```bash
docker-compose up --build   # Build and run both services
# Frontend: http://localhost:80  (Nginx proxies /api/ to backend)
# Backend: not published on host, only accessible within Docker network
```

### Backend (`registro/`)
```bash
npm run dev                     # Dev server with hot reload (tsx watch src/index.ts)
npm start                       # Production start (npx tsx src/index.ts)
npx prisma migrate deploy       # Apply pending migrations
npx prisma generate             # Regenerate Prisma client after schema changes
npx prisma studio               # GUI for the database
```

### Frontend (`velt/`)
```bash
npm run dev      # Dev server with HMR — proxies /api to localhost:3001
npm run build    # Production build → dist/
npm run lint     # ESLint
```

There are no tests in this project.

## Architecture

### Two-service Docker Compose
- `registro/` builds to a Node 22-alpine container (internal port 3001, not published to host)
- `velt/` builds to an Nginx container (port 80 published)
- Nginx proxies `/api/` and `/uploads/` requests to `http://registro:3001/`
- SQLite DB persisted in Docker volume `registro_db` at `/app/data/dev.db`
- WhatsApp auth sessions in volumes `auth_4`, `auth_5`; uploaded files in `uploads`

### Backend entry point: `registro/src/index.ts`
Seeds the admin user (`admin` / `admin123`) on first run, initializes the WhatsApp bot manager, then starts Express. The Express app is configured in `src/app.ts`.

Route groups mounted in `src/app.ts`:
- `POST /auth/login` / `GET /auth/me` — JWT auth (no auth required for login)
- `/admin/usuarios` — user CRUD, admin role only
- `/servicios`, `/dispositivos`, `/reglas`, `/plantillas`, `/clientes`, `/demos`, `/upload` — per-user resources

Auth middleware (`src/middleware/`) validates Bearer tokens and checks account expiration. JWT secret defaults to `"demo-bot-secret-2024"` if `JWT_SECRET` env var is unset (see `src/config.ts`).

> `registro/cliente.ts` is the old monolithic version of the server kept for reference — it is no longer the entry point.

### WhatsApp bot: `registro/bot.ts`
Manages WhatsApp socket connections via `@whiskeysockets/baileys`. Each `Dispositivo` record gets its own connection. Device states: `desconectado → esperando_qr → conectado | pausado`.

**Runtime vs. DB state**: The `dispositivos` Map is in-memory only. The DB is updated on state transitions (connect/disconnect/pause). On server restart, `iniciarBots()` re-reads DB state and auto-reconnects devices that have a `auth_{id}` directory on disk and are not in `pausado` state.

**Auth sessions**: Stored in `auth_{dispositivoId}/` directories relative to the backend CWD. `pausarDispositivo()` closes the socket but keeps the auth dir (QR not needed on next connect). `desconectarDispositivo()` calls `logout()` and deletes the auth dir (QR required again).

**Incoming message routing** (priority order):
1. Skip group messages (`@g.us`) and status broadcasts
2. Skip messages that have no text content or were sent by the bot itself
3. If sender's phone matches a `CuentaCliente` record → silently ignore
4. If text is exactly `"demo"` → run demo account delivery flow
5. Otherwise → match against `RespuestaRegla` (ordered by `orden`) by keyword substring

**Phone number format**: Numbers stored as `"591 64598812"` (country code + space + local). Incoming WhatsApp JIDs (e.g. `59164598812@s.whatsapp.net`) are parsed by `detectarPais()` which tries 3-digit, 2-digit, then 1-digit prefixes against `CODIGOS_PAIS`. JIDs with `addressingMode === "lid"` use `remoteJidAlt` instead of `remoteJid`.

### Data models (`registro/prisma/schema.prisma`)
```
Usuario ──┬──> Dispositivo ──> RespuestaRegla ──> RespuestaPaso
          ├──> Servicio
          ├──> CuentaCliente
          ├──> PlantillaRecordatorio
          └──> CuentaDemo ──> ClienteDemo
```
- `Usuario.rol`: `"admin"` or `"cliente"`; `expiraEn` = null means unlimited (admins only)
- `Dispositivo`: max 2 per user, linked to a WhatsApp session
- `RespuestaRegla` / `RespuestaPaso`: keyword-triggered automated reply chains (text, image, video)
- `CuentaDemo`: pre-created credentials; `ClienteDemo` tracks who received them via bot

### Frontend: `velt/src/`
React SPA structured as:
- `App.jsx` — root component that handles auth state and page routing
- `pages/` — one file per section: `PaginaLogin`, `PaginaAdmin`, `PaginaDispositivos`, `PaginaServicios`, `PaginaClientes`, `PaginaPlantillas`, `PaginaDemos`
- `components/ui.jsx` — shared primitives (Toast, Modal, Badge, etc.)
- `components/TelefonoInput.jsx` — phone number input with country detection
- `lib/api.js` — `apiFetch()` wrapper that injects Bearer token from `localStorage`
- `lib/paises.js` — country list

In development, Vite proxies `/api` to `localhost:3001`. In production, Nginx handles the proxy — there is no hardcoded backend URL in the frontend source.
