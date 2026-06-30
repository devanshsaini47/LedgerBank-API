# Bank Backend App

A RESTful banking backend built with **Node.js, Express, and MongoDB**, implementing core banking primitives the way real financial systems do: an **immutable double-entry ledger**, **idempotent money transfers**, and **atomic, session-based transactions** — instead of naively updating a single "balance" field.

**Live demo:** https://bank-backend-app.onrender.com

## Why this project

Most CRUD-style "bank app" tutorials just increment/decrement a `balance` field on a user document, which is unsafe under concurrent requests and impossible to audit. This project instead models money movement the way actual banking systems do:

- **Balances are derived, not stored.** Every account's balance is computed on demand from its ledger entries (`SUM(credits) - SUM(debits)`), so there's a permanent, tamper-evident audit trail of every rupee that ever moved.
- **Ledger entries are immutable.** Mongoose pre-hooks block `updateOne`, `deleteOne`, `findOneAndUpdate`, and every other mutation path on the ledger collection — once a debit or credit is written, it cannot be altered or deleted.
- **Transfers are idempotent.** Every transaction requires a client-supplied `idempotencyKey`, so retried/duplicate requests (e.g. from network retries or double-clicks) never double-charge an account.
- **Transfers are atomic.** Each transfer runs inside a MongoDB session/transaction — the debit ledger entry, credit ledger entry, and status update either all commit together or none do.

## Features

- **Authentication** — registration, login, and logout with JWT (HTTP-only cookie or Bearer token), bcrypt password hashing, and a **token blacklist** (with TTL auto-expiry) to properly invalidate tokens on logout.
- **Accounts** — create accounts, list a user's accounts, and fetch a real-time balance computed from the ledger.
- **Transactions / Transfers** — a 10-step transfer flow (validate → check idempotency → check account status → derive balance → open DB transaction → write debit entry → write credit entry → mark completed → commit → notify) with insufficient-balance and frozen/closed-account checks.
- **System / seed transactions** — a separate, privileged endpoint (gated by a `systemUser` flag) for crediting initial funds into accounts, e.g. for onboarding or testing.
- **Email notifications** — registration and transaction-confirmation emails sent via Nodemailer using Gmail OAuth2.

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js |
| Framework | Express 5 |
| Database | MongoDB with Mongoose 9 (multi-document ACID transactions) |
| Auth | JSON Web Tokens (`jsonwebtoken`), `bcryptjs`, `cookie-parser` |
| Email | Nodemailer (Gmail OAuth2) |
| Config | dotenv |
| Deployment | Render |

## Architecture

```
src/
├── app.js                       # Express app setup & route mounting
├── config/
│   └── db.js                    # MongoDB connection
├── controllers/
│   ├── auth.controller.js       # register / login / logout
│   ├── account.controller.js    # create / list accounts, get balance
│   └── transaction.controller.js# transfer logic, ledger writes
├── middleware/
│   └── auth.middleware.js       # JWT auth + system-user auth guards
├── models/
│   ├── user.model.js            # user schema + password hashing
│   ├── account.model.js         # account schema + getBalance() from ledger
│   ├── ledger.model.js          # immutable double-entry ledger
│   ├── transaction.model.js     # transaction (transfer) records
│   └── blackList.model.js       # logged-out / revoked JWTs (TTL indexed)
├── routes/
│   ├── auth.routes.js
│   ├── account.routes.js
│   └── transaction.routes.js
└── services/
    └── email.service.js         # transactional emails via Nodemailer
server.js                        # entry point
```

## Data Model

**Account** holds no balance field directly — `account.getBalance()` runs a MongoDB aggregation over the `ledger` collection to sum credits and debits for that account in real time.

**Ledger** is an append-only collection: every transfer writes exactly one `DEBIT` entry (sender) and one `CREDIT` entry (receiver), each linked to the originating `Transaction`. All mutating Mongoose hooks are blocked at the schema level.

**Transaction** tracks the lifecycle of a transfer through `PENDING → COMPLETED` (or `FAILED` / `REVERSED`), and enforces a unique `idempotencyKey` so a transfer can never be processed twice.

## API Endpoints

### Auth
| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/auth/register` | Register a new user |
| POST | `/api/auth/login` | Log in and receive a JWT |
| POST | `/api/auth/logout` | Log out and blacklist the token |

### Accounts (auth required)
| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/accounts` | Create a new account for the logged-in user |
| GET | `/api/accounts` | List all accounts for the logged-in user |
| GET | `/api/accounts/balance/:accountId` | Get an account's current balance |

### Transactions
| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/transactions` | Transfer funds between two accounts (auth required) |
| POST | `/api/transactions/system/initial-funds` | Seed an account with initial funds (system-user only) |

## Getting Started

### Prerequisites
- Node.js (v18+ recommended)
- A MongoDB instance (local or Atlas)
- A Gmail account configured for OAuth2 (for email notifications)

### Installation

```bash
git clone https://github.com/devanshsaini47/Bank-backend-app.git
cd Bank-backend-app
npm install
```

### Environment Variables

Create a `.env` file in the project root:

```env
MONGO_URI=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret
EMAIL_USER=your_gmail_address
CLIENT_ID=your_google_oauth_client_id
CLIENT_SECRET=your_google_oauth_client_secret
REFRESH_TOKEN=your_google_oauth_refresh_token
```

### Run

```bash
node server.js
```

The server starts on `http://localhost:3000`.

## Future Improvements

- Input validation layer (e.g. Joi/Zod) on request bodies
- Automated tests (unit + integration) for the transfer flow
- Rate limiting and request logging
- Pagination for transaction/ledger history endpoints
- Reversal/refund flow for `FAILED` transactions

## Author

**Devansh Saini**
GitHub: [@devanshsaini47](https://github.com/devanshsaini47)
