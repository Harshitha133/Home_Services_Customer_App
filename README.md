# Home Services App (Door Service)

## Project Overview

An on-demand home services marketplace app. Customers can browse and book services — cleaning, salon & beauty, AC & appliance repair, electrician & plumbing — from verified professionals, pay online, and track bookings end to end.

The platform consists of three applications sharing one backend:

- **Customer App** — Flutter
- **Professional App** — Flutter
- **Admin Dashboard** — React

## Technology Stack

- **Customer / Professional Apps:** Flutter / Dart
- **Admin Dashboard:** React + TypeScript + Tailwind
- **Backend:** Node.js + Express 5
- **Database:** PostgreSQL via Prisma 7 ORM (`@prisma/adapter-pg`)
- **Cache / OTP storage:** Redis
- **Auth:** Self-hosted — bcrypt password hashing, self-issued JWTs (access + refresh), phone+OTP verification for signup/password-reset, phone+password for login
- **Payments:** Razorpay
- **Storage:** AWS S3 (planned)
- **Notifications:** Firebase FCM (planned) — in-app notifications module already built
- **Maps:** Google Maps APIs (planned)
- **Deployment:** Docker + AWS ECS Fargate (planned)
- **CI/CD:** GitHub Actions (planned)

## Project Structure

```
Door_Service_Customer_app/
├── backend/                     # Node/Express + Prisma backend
│   ├── prisma/                  # schema.prisma, migrations, seed.js
│   └── src/
│       ├── config/               # Redis, Razorpay config
│       ├── middlewares/          # authenticate, requireRole, error handling
│       ├── utils/                # JWT tokens, OTP generation/storage
│       └── modules/
│           ├── auth/              # register/login/verify/refresh — customer, professional, admin
│           ├── customer/          # profile, addresses, categories, services, bookings, reviews, offers
│           ├── payment/           # Razorpay order creation + verification
│           ├── notification/      # in-app notifications
│           └── support/           # FAQs + support tickets
│
├── apps/
│   ├── customer-app/             # Flutter customer app
│   ├── professional-app/         # Flutter professional app (not yet built)
│   └── admin-dashboard/          # React admin dashboard (not yet built)
│
├── packages/
│   └── shared_auth/               # Shared Flutter auth module (customer-app + professional-app)
│
├── database/                     # Database-related reference files
└── docs/                          # Project and API documentation
```

## Backend — Getting Started

```bash
cd backend
npm install
npm rebuild            # ensures bcrypt's native binding is compiled — see Known Issues
npx prisma migrate deploy
npx prisma db seed
npm run dev            # starts on http://localhost:4000
```

Requires a running PostgreSQL database (`DATABASE_URL` in `.env`) and a running Redis instance (used for OTP storage).

### API base URL
All endpoints are under `http://localhost:4000/api/v1`.

| Module | Base path | Auth required |
|---|---|---|
| Auth | `/auth/*` | No (login/register endpoints) |
| Customer | `/customer/*` | Yes (`CUSTOMER` role) |
| Payments | `/customer/payments/*` | Yes (`CUSTOMER` role) |
| Notifications | `/notifications/*` | Yes (any authenticated role) |
| Support | `/customer/help-support/*` | FAQs public, tickets require `CUSTOMER` role |

See `docs/` for the full endpoint reference with request/response examples.

## Flutter Apps — Getting Started

```bash
cd apps/customer-app
flutter pub get
flutter run -d chrome    # or -d windows / an Android emulator
```

The `customer-app` depends on the local `shared_auth` package (`packages/shared_auth`) via a path dependency — no separate install needed, `flutter pub get` resolves it automatically.

### Platform-specific backend URL
The API client resolves the backend URL per platform:
- **Web / Windows / macOS / Linux:** `http://localhost:4000/api/v1`
- **Android emulator:** `http://10.0.2.2:4000/api/v1` (Android emulators can't reach the host machine via `localhost`)

## Auth Architecture

One backend serves all three roles through parallel routes:
- `POST /auth/customer/register`, `.../login`, `.../register/verify`, `.../forgot-password/request`, `.../forgot-password/reset`
- `POST /auth/professional/*` — mirrors customer exactly
- `POST /auth/admin/login` — email + password only, no OTP flow
- `POST /auth/refresh`, `POST /auth/logout` — shared across all roles

On the Flutter side, auth is implemented once in `packages/shared_auth`, parameterized by role (`'customer'` or `'professional'`), and reused by both apps rather than duplicated.

**Signup flow:** register → OTP sent → verify OTP (marks phone verified, issues no tokens) → user must log in separately to get a session.

**OTP delivery:** Real SMS sending is not yet implemented. In development, the OTP is returned directly in the API response (`devOnlyCode` field) and also printed to the backend console — **must be removed before production.**

## Booking Status Values

```
CONFIRMED → ASSIGNED → ON_THE_WAY → STARTED → COMPLETED
                                              ↘ CANCELLED
```

These exact strings are used across the backend's validation, the Prisma `BookingStatus` enum, and must be matched exactly on the frontend.

## Known Issues / Setup Gotchas

- **bcrypt native binding:** `npm install` may silently skip install scripts for `bcrypt`, `@prisma/engines`, and `prisma`, leaving bcrypt's native binding uncompiled — this causes login requests to silently drop the connection. Fix: `npm install-scripts approve <pkg>` for each, then `npm rebuild`.
- **No request logging by default:** `morgan` request logging should be added (`app.use(morgan("dev"))` in `app.js`) for local debugging visibility.
- **`node --watch` watches `node_modules`:** can cause spurious server restarts. Consider running with `--watch-path=src` to limit watching to source files only.

## Test Account (Development)

A pre-verified test customer is seeded for local testing:
- **Phone:** `9999999998`
- **Password:** `Test@123`

## Development Workflow

1. Create a feature branch from `main`.
2. Work on the assigned feature/module.
3. Commit changes with a clear message.
4. Push the feature branch.
5. Open a Pull Request.
6. Review and test.
7. Merge once approved.

## Status

- **Backend:** Auth, Customer module (profile/addresses/categories/services/bookings/reviews/offers), Payments (Razorpay), Notifications, Search, Popular/Recommended Services, Offers — all built and tested.
- **Customer App (Flutter):** Auth screens fully wired to the real backend and live-tested (signup, OTP verify, login, forgot/reset password). Customer module (profile/addresses/categories/services/bookings/reviews) service layer written, screen wiring in progress.
- **Professional App:** Not yet started.
- **Admin Dashboard:** Not yet started (a separate earlier admin-dashboard prototype exists with UI-only screens and mocked auth).
