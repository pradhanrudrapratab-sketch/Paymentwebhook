# MrKrishna Gateway

A lightweight Node.js/Express UPI payment-link and QR workflow that delegates payment-status verification to an external gateway API. This README documents the project logic so it can be rebuilt consistently.

> **Important:** This project is an integration layer, not a bank or UPI payment processor. A QR code or UPI deep link does not prove that money was received. Only mark an order paid after a trusted server-side verification response. Test the provider's real response schema and authentication before using this for real payments.

## Contents

- [What it does](#what-it-does)
- [Architecture](#architecture)
- [Technology](#technology)
- [Project structure](#project-structure)
- [Configuration](#configuration)
- [Order ID generation](#order-id-generation)
- [Order database method](#order-database-method)
- [Order lifecycle](#order-lifecycle)
- [Payment link and QR generation](#payment-link-and-qr-generation)
- [Gateway verification](#gateway-verification)
- [Discord success notifications](#discord-success-notifications)
- [HTTP endpoints](#http-endpoints)
- [Run locally](#run-locally)
- [Deploy to Render](#deploy-to-render)
- [Testing](#testing)
- [Known limitations](#known-limitations)
- [Production hardening checklist](#production-hardening-checklist)
- [Troubleshooting](#troubleshooting)

## What it does

1. Accepts a requested payment amount.
2. Creates a unique order ID and stores a pending order.
3. Produces a customer payment page with a UPI payment URI and QR code.
4. Checks the order against the configured external gateway API from the server.
5. Stores the verified status and timestamps in a JSON file.
6. Sends a Discord notification when a payment is first verified as successful, if Discord is configured.
7. Exposes a health endpoint for basic deployment checks.

The customer-facing payment page should stay simple. Administrative configuration belongs in environment variables, not in the customer page. This project has no wallet or stored customer balance.

## Architecture

```text
Customer browser
    |
    | open payment link
    v
Express app ---- create order ----> orders.json
    |
    +---- generate UPI URI / QR image
    |
    +---- request status verification
                 |
                 v
       External gateway status API
                 |
                 v
       Validate provider response
                 |
                 +----> update orders.json
                 |
                 +----> Discord success alert (optional)
```

The app must never trust a status supplied by the browser. The server makes the verification request using its secret API key and the order's stored amount and ID.

## Technology

- Node.js (use a supported LTS release; Node 22 is a sensible baseline)
- Express for HTTP routes
- `qrcode` for QR image generation
- `node-cron` for periodic checks while the process is running
- Built-in `fs`, `path`, and `crypto` modules for JSON persistence and random IDs
- Discord bot REST API for optional notifications

There is deliberately no SQLite dependency in this version. The working storage method is a JSON file.

## Project structure

```text
mrkrishna-gateway/
├── server.js          # Express routes, order logic, verification, Discord alerts
├── package.json       # Runtime dependencies and scripts
├── orders.json        # Local JSON order store; starts as {}
├── .env.example       # Names of required/optional environment variables
├── .gitignore         # Excludes secrets and local dependencies
└── README.md
```

Keep `.env` out of Git. Never commit a real gateway API key or Discord bot token.

## Configuration

Configure these values in a local `.env` file or in the hosting provider's environment-variable settings:

| Variable | Required | Purpose |
|---|---:|---|
| `GATEWAY_API_KEY` | Yes for verification | Secret key used to call the upstream payment status API. |
| `UPI_ID` | Yes for UPI payment URI | Merchant UPI ID/VPA to receive payments. |
| `UPI_PAYEE_NAME` | Recommended | Merchant name shown in the UPI payment URI. |
| `DISCORD_BOT_TOKEN` | Optional | Bot token used to send deposit notifications. |
| `DISCORD_GUILD_ID` | Optional with Discord | Expected Discord server ID; used to check the target channel belongs to this server. |
| `DISCORD_CHANNEL_ID` | Optional with Discord | Channel where verified-success notifications are posted. |
| `PORT` | Usually supplied by host | HTTP port; the app should listen on `process.env.PORT` with a local fallback. |
| `ORDERS_FILE` | Optional implementation setting | Path to the JSON database file, if the code supports overriding it. |

Do not place secrets in frontend HTML, query strings, public repositories, screenshots, or Discord messages. If a key/token was ever exposed, rotate it.

## Order ID generation

The order ID is generated on the server, not supplied by the customer. The documented implementation uses a prefix, a time-derived component, and cryptographically generated random characters. The historical pattern in this project is similar to:

```text
ORD + date/time component + 6 random characters
```

The implementation uses Node's `crypto.randomInt()` to select random characters from an allowed character set. It checks the in-memory order map for a collision and generates another ID if one already exists.

### Requirements for a correct implementation

- Generate IDs on the server only.
- Use a cryptographically secure random source (`crypto.randomInt` or `crypto.randomBytes`), not `Math.random()`.
- Check for collisions before accepting an ID.
- Store the ID as the stable primary key of the order.
- Do not reuse an order ID for a second payment attempt.
- Keep IDs opaque; do not embed API keys, user secrets, or sensitive customer data in them.

A timestamp helps with readability but is not itself a security control. The random component and collision check are important. If multiple app instances can create orders concurrently, a JSON-file collision check is not sufficient; use a database with a unique primary-key constraint.

## Order database method

### Current method: JSON file

The app loads `orders.json` at startup into an in-memory object/map. When an order is created or its status changes, it writes the updated order collection to a temporary file and then renames that temporary file over `orders.json`. The temp-file-plus-rename pattern reduces the chance of leaving a half-written JSON file if the process stops during a write.

An initial database file can be:

```json
{}
```

An order record may contain fields like these (the exact property names should match `server.js`):

```json
{
  "ORD202610130001ABC123": {
    "order_id": "ORD202610130001ABC123",
    "amount": 100,
    "status": "pending",
    "created_at": "2026-10-13T00:00:00.000Z",
    "updated_at": "2026-10-13T00:00:00.000Z",
    "verified_at": null,
    "discord_notified": false
  }
}
```

The example is illustrative; retain the exact field names used by the source code. Amounts should be validated as finite positive numbers and should not be accepted as arbitrary text. For financial-grade systems, store amounts as integer paise rather than binary floating-point values.

### Persistence flow

1. On startup, read and parse `orders.json`.
2. If the file does not exist on a fresh installation, initialize an empty object and create the file.
3. Create/update the in-memory order record.
4. Serialize the complete order collection to a temporary file in the same directory.
5. Rename the temporary file to the configured database path.
6. Handle and log read/write errors; do not silently pretend a failed write succeeded.

### JSON storage caveats

JSON storage is suitable for a small prototype or a single-process test. It is not a transactional database. Concurrent writes can overwrite each other, multiple instances do not share state, and a corrupt file can prevent startup unless recovery is implemented.

Render's ordinary filesystem for a free web service is ephemeral. Orders can disappear after a restart, redeploy, or instance replacement. For durable real payments, move orders to a managed database (for example PostgreSQL) or attach supported persistent storage. Do not claim durable storage while using only an ephemeral filesystem.

## Order lifecycle

A typical order moves through these states:

```text
created -> pending -> success
                   \-> failed
```

- `pending`: created but not yet verified as paid or failed.
- `success`: the trusted upstream API returned a response that the server has explicitly recognized as a successful payment.
- `failed`: the upstream API explicitly reports a final failure.
- Network errors, timeouts, malformed JSON, unexpected status values, or provider errors are **not** proof of failure and should not be converted to `success` or a terminal `failed` status. Keep the order pending and log/report the verification error.

If the current provider only returns `success` or `failed`, map those exact documented values after validating the response. Do not use loose substring matching, browser input, or a QR scan as payment confirmation. Confirm whether the provider's response means *paid/settled* rather than merely *request accepted*.

## Payment link and QR generation

The app exposes a payment link for an order and a QR image endpoint. The payment page obtains the order from the server's stored record, then builds a UPI URI using the configured merchant VPA, payee name, amount, and an order/reference identifier where supported. The `qrcode` package turns that URI into a QR image.

Key rules:

- Use the amount saved in the order record when generating the payment URI; do not trust a changed amount from the browser during verification.
- URL-encode UPI URI parameters correctly.
- Treat a QR code as payment instructions only, not proof of payment.
- Keep the customer page minimal: amount, merchant/payee, QR, payment link/instructions, and a status-check action.
- Escape user-controlled text before inserting it into HTML.
- Do not expose the gateway API key to the browser.

## Gateway verification

The server-side verification logic calls the configured upstream endpoint:

```text
https://vcgatewaypro.com/payment_api.php
```

The integration sends the secret API key, the stored `order_id`, and the stored `amount` as request parameters. In the existing integration these are sent as query parameters using a URL/search-parameter encoder rather than concatenating untrusted values manually.

Conceptual request:

```text
GET /payment_api.php?api_key=...&order_id=...&amount=...
```

### Verification algorithm

1. Receive an order ID from a route or API request.
2. Look up that ID in the local order store. Return `404` if it does not exist.
3. Read the amount from the stored record; do not accept a client-supplied replacement amount.
4. Call the provider with a timeout and the server-only API key.
5. Check the HTTP response and parse JSON safely.
6. Validate the provider response schema and the exact documented status field/value.
7. If the provider explicitly confirms success, update the order to `success`, set its verification/update timestamps, and persist it.
8. If the provider explicitly confirms a final failure, update to `failed` only if that failure is truly terminal under the provider's contract.
9. If the request fails, times out, returns invalid JSON, or has an unknown schema/status, preserve the current state (normally `pending`) and return a clear temporary verification error.
10. When the status first becomes successful, attempt the Discord notification if configured.

### Important provider contract checks

Before relying on this endpoint, verify with the provider's current documentation or test transactions:

- Exact parameter names and whether the API key should be sent in query parameters or a header.
- Exact success/failure values and whether status is case-sensitive.
- Whether the response includes and validates the order ID and amount.
- Whether the API response is authenticated/trustworthy and whether requests need signing.
- Rate limits, timeout behavior, and how pending payments are represented.
- Whether a status lookup can return success for a different merchant/order.

If available, compare the returned order ID and amount to the stored values before accepting success. Never mark an order successful merely because the endpoint responded with HTTP 200.

### Idempotency

Verification may be requested repeatedly by a customer, cron task, or retry. The transition to `success` and the notification path must be idempotent: verifying an already successful order should not create a new order or send unlimited duplicate alerts. Persist the status update before reporting success to the caller.

## Discord success notifications

Discord notifications are optional. Configure a bot token, expected server ID, and channel ID. The bot needs permission to view the target channel and send messages there.

When a payment is verified as successful:

1. Build a short embed with the order ID, amount, status, and verification time.
2. Confirm the configured channel is available and belongs to the expected server, if the code performs this check.
3. Send the embed using the Discord API.
4. Set and persist a `discord_notified` flag only after Discord confirms the send succeeded.
5. If Discord fails, log the error and leave the notification eligible for a later retry; do not revert a verified payment to pending because Discord is unavailable.

Do not include API keys, tokens, full payment credentials, or unnecessary customer data in the message. Discord notifications are operational alerts, not the source of truth for payment state.

## HTTP endpoints

The existing project includes routes in this general shape. Check the actual `server.js` before relying on the exact methods and response bodies, since endpoint behavior is defined by code.

| Endpoint | Purpose |
|---|---|
| `GET /` | Simple form/page for generating a payment link. |
| `GET /link?value=10` | Creates an order for the requested amount and returns/redirects to its payment link. Validate the amount server-side. |
| `GET /pay/:id` | Displays the payment page for a stored order. |
| `GET /qr/:id.png` | Returns the QR image for the order's payment URI. |
| `POST /api/check/:id` | Requests server-side verification with the upstream gateway. |
| `GET /cheque?value=ORDER_ID` | Legacy/status-check route that looks up the specified order ID. Despite the spelling, this is a payment status route, not a bank cheque service. |
| `GET /health` | Basic health response for deployment checks. It should not expose secrets or internal data. |

For production, prefer a consistent REST design and appropriate HTTP status codes. Add request throttling to order creation and verification endpoints. If the application is public, protect administrative actions and do not rely on obscurity of an order ID as authorization.

## Run locally

1. Install Node.js LTS.
2. Download/clone the project and enter its directory.
3. Install dependencies:

   ```bash
   npm install
   ```

4. Copy `.env.example` to `.env` and set the required values. Do not commit `.env`.
5. Ensure `orders.json` exists and contains `{}`.
6. Start the server using the script defined in `package.json`, typically:

   ```bash
   npm start
   ```

7. Open `http://localhost:PORT/`, using the port printed by the app or configured in `.env`.
8. Use a test order and confirm that order creation, QR rendering, and status checking work. Do not test live payment verification with made-up success responses.

If the project does not use a dotenv loader, export environment variables in your shell instead; do not assume `.env` is loaded automatically.

## Deploy to Render

1. Push the project to a private or public Git repository, excluding `.env`, `node_modules`, and any real secrets.
2. Create a Render Web Service connected to the repository.
3. Use the Node runtime and set the build command to `npm install` (or the lockfile-appropriate install command).
4. Set the start command to `npm start`.
5. Add the environment variables in Render's Environment settings.
6. Do not hardcode a fixed listening port; bind to `process.env.PORT`.
7. Deploy and check `/health` and the deployment logs.
8. Run a test order and verify the external API response before accepting real payments.

**Storage warning:** a local `orders.json` on Render Free is not durable. A restart or redeploy may erase orders. For real transactions, connect durable storage before relying on order history or reconciliation. A production setup should also define backups, access controls, retention, and recovery procedures.

## Testing

Use a separate test environment and small controlled transactions. At minimum, test:

- Valid amount creates exactly one pending order.
- Zero, negative, missing, non-numeric, excessively large, and malformed amounts are rejected.
- A generated order ID is unique and is stored before the payment link is shown.
- Unknown order IDs return `404` and do not call the provider.
- A valid provider success response updates the correct order to success.
- A valid final failure response is mapped correctly.
- Network timeout, upstream `5xx`, invalid JSON, missing status, and unknown status leave the order unconfirmed rather than marking it paid.
- Repeated verification does not duplicate orders or spam Discord.
- Discord unavailable/permission denied does not change a successful payment back to pending.
- A process restart behaves as expected for the chosen storage type.
- QR and payment page use the stored amount and configured merchant details.

Mock provider responses in automated tests rather than fabricating real transactions. Keep sanitized examples of the actual provider responses in tests after confirming they contain no secrets or personal data.

## Known limitations

- JSON file persistence is not transactional and is not safe for multiple concurrent app instances.
- Ephemeral hosting storage can lose orders on restart/redeploy.
- A cron job only runs while the Node process is alive. Free hosting may sleep, so scheduled checks are not guaranteed to run on time.
- The gateway's current response schema and meaning of its status values must be independently confirmed.
- A public payment/status endpoint can be abused without throttling and input validation.
- A QR code alone cannot confirm a payment.

## Production hardening checklist

Before handling meaningful payment volume:

- [ ] Replace JSON persistence with PostgreSQL or another durable database.
- [ ] Add database transactions/unique constraints and atomic state transitions.
- [ ] Use integer paise for monetary values and strict amount limits.
- [ ] Confirm the provider's response schema, authentication, and final-settlement semantics.
- [ ] Add request timeouts, retry/backoff policy, rate limiting, and structured logs.
- [ ] Verify order ID and amount in the provider response when those fields are available.
- [ ] Keep all secrets in environment variables and rotate any exposed credentials.
- [ ] Make notifications idempotent and persist retry state.
- [ ] Add authentication/authorization for administrative routes.
- [ ] Add durable backups, monitoring, and reconciliation against provider records.
- [ ] Add automated tests for success, failure, pending, timeout, and malformed responses.
- [ ] Review applicable legal, tax, privacy, and payment-provider requirements before commercial use.

## Troubleshooting

**The app crashes while deploying**

Check the Node version, dependency install logs, and start command. Use a supported Node LTS release and avoid native database packages unless the deployment environment supports their build requirements.

**Orders disappear after a redeploy**

That is expected with ephemeral local storage. Use a persistent disk where supported or, preferably for payment records, a managed database.

**QR code displays but payment does not become successful**

The QR is only payment instructions. Inspect the server logs and sanitized upstream status response; confirm the provider expects the same order ID/amount and that the response schema matches the parser.

**Verification returns an error**

Check the API key, provider endpoint, URL parameter encoding, network timeout, HTTP status, JSON parsing, and provider rate limits. Do not convert unknown errors into success.

**Discord message does not arrive**

Confirm the bot token is valid, the guild and channel IDs are correct, the bot can view the channel and send messages, and the server logs show a successful Discord API response. A notification failure must not alter a verified payment status.

---

## License and responsibility

Add a license file if you intend others to reuse the code. This repository is a software integration example; you are responsible for verifying provider behavior, protecting secrets, safeguarding payment records, and meeting the requirements that apply to your use case.
