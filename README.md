# Atomic Ledger

Backend Ledger is an Express + MongoDB banking/ledger API for user authentication, account management, and double-entry style transaction tracking.

The service supports:

- User registration, login, and logout with JWT-based authentication
- Account creation and balance lookup
- Ledger-backed money transfers between accounts
- A special system-user endpoint for issuing initial funds
- Email notifications for registration and successful transfers

## Tech Stack

- Node.js
- Express 5
- MongoDB with Mongoose
- JSON Web Tokens
- bcryptjs for password hashing
- cookie-parser for cookie-based auth tokens
- Nodemailer for email notifications

## Project Structure

```text
backend-ledger/
├── server.js
├── package.json
└── src/
    ├── app.js
    ├── config/db.js
    ├── controllers/
    ├── middleware/
    ├── models/
    ├── routes/
    └── services/email.service.js
```

## How It Works

- Authentication is handled with a JWT stored in a cookie named `token`.
- The same token can also be supplied through the `Authorization: Bearer <token>` header.
- Logout blacklists the current token in MongoDB for 3 days.
- Account balances are not stored directly; they are derived from immutable ledger entries.
- Transactions use an idempotency key to prevent duplicate processing.
- A transaction must target two ACTIVE accounts and cannot proceed if the sender lacks sufficient balance.
- The transfer flow writes DEBIT and CREDIT ledger entries and marks the transaction `COMPLETED`.
- The transfer controller includes an intentional 15-second delay before the credit ledger write, so requests can take noticeably longer than a normal API call.

## Prerequisites

- Node.js 18+ recommended
- MongoDB connection string
- Gmail OAuth2 credentials for email delivery

## Installation

```bash
npm install
```

## Environment Variables

Create a `.env` file in the project root with the following values:

```env
MONGO_URI=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret

EMAIL_USER=your_gmail_address
CLIENT_ID=your_google_oauth_client_id
CLIENT_SECRET=your_google_oauth_client_secret
REFRESH_TOKEN=your_google_oauth_refresh_token
```

## Running the App

Development:

```bash
npm run dev
```

Production:

```bash
npm start
```

The server listens on port `3000`.

## Health Check

```http
GET /
```

Response:

```text
Ledger Service is up and running
```

## Authentication

Protected routes require a valid JWT.

You can send it either as:

```http
Cookie: token=<jwt>
```

or:

```http
Authorization: Bearer <jwt>
```

## API Endpoints

### Auth

#### Register

```http
POST /api/auth/register
```

Request body:

```json
{
  "name": "Alice",
  "email": "alice@example.com",
  "password": "password123"
}
```

Response:

```json
{
  "user": {
    "_id": "...",
    "email": "alice@example.com",
    "name": "Alice"
  },
  "token": "<jwt>"
}
```

#### Login

```http
POST /api/auth/login
```

Request body:

```json
{
  "email": "alice@example.com",
  "password": "password123"
}
```

#### Logout

```http
POST /api/auth/logout
```

This blacklists the current token and clears the `token` cookie.

### Accounts

All account routes are protected.

#### Create Account

```http
POST /api/accounts/
```

Creates a new account for the authenticated user.

Request body: none.

#### Get My Accounts

```http
GET /api/accounts/
```

Returns all accounts owned by the authenticated user.

#### Get Account Balance

```http
GET /api/accounts/balance/:accountId
```

Returns the balance derived from the account's ledger entries.

### Transactions

#### Create Transfer

```http
POST /api/transactions/
```

Protected route for moving funds between two accounts.

Request body:

```json
{
  "fromAccount": "account_id_1",
  "toAccount": "account_id_2",
  "amount": 100,
  "idempotencyKey": "unique-request-key"
}
```

Behavior:

- Requires both accounts to exist and be `ACTIVE`
- Verifies the sender has sufficient available balance
- Reuses the same result if the idempotency key already exists
- Writes immutable debit and credit ledger entries
- Marks the transaction `COMPLETED` on success
- Sends a success email to the authenticated user

#### Create Initial Funds Transaction

```http
POST /api/transactions/system/initial-funds
```

Protected route for system users only.

Request body:

```json
{
  "toAccount": "account_id",
  "amount": 1000,
  "idempotencyKey": "unique-seed-key"
}
```

This route requires the authenticated user to have `systemUser: true`.

## Data Model Summary

### User

- `email` is unique and lowercase-normalized
- `password` is hashed with bcrypt before save
- `systemUser` is a protected boolean flag used for privileged transfers

### Account

- Belongs to a user
- Status values: `ACTIVE`, `FROZEN`, `CLOSED`
- Currency defaults to `INR`
- Balance is computed from ledger entries

### Transaction

- References `fromAccount` and `toAccount`
- Status values: `PENDING`, `COMPLETED`, `FAILED`, `REVERSED`
- Uses a unique `idempotencyKey`

### Ledger

- Immutable debit/credit records
- Each entry belongs to one account and one transaction

### Token Blacklist

- Stores invalidated JWTs
- Entries expire automatically after 3 days

## Notes

- There are no automated tests defined in `package.json` yet.
- The current app expects MongoDB and Gmail OAuth2 configuration to be available before startup.
- The transaction code currently logs passwords during password comparison in development; avoid exposing logs in production.

## License

ISC
