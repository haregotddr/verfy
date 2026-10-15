# Payment Automation Agent (Android, Kotlin + Jetpack Compose)

Logs in to the management system, syncs pending payment requests, reads incoming Telebirr SMS,
and automatically decides CONFIRM / REJECT / WAIT -> EXCEPTION. The agent only handles exceptions.

## Build the APK (no Android Studio needed)
1. Create a GitHub repo and push this folder to the `main` branch.
2. Open the repo > Actions > "Build APK" (it runs on push; or use "Run workflow").
3. Download the `payment-agent-apk` artifact > `app-debug.apk` and install it (allow "install unknown apps").
Or: open the folder in Android Studio and press Run.

## Management system API contract (edit `Net.kt` > `Api` if yours differs)
- `POST /api/auth/login` `{username,password}` -> `{token | access_token, expires_in?}`
- `GET  /api/payment-requests?status=PENDING` (Bearer token) -> array or `{items:[...]}` with
  `request_id,user_id,customer_name,wallet_number,amount,currency,provider,created_at,expiration_time`
- `POST /api/decisions` (Bearer token, `Idempotency-Key` = transaction ID) with
  `request_id,transaction_id,decision,amount,wallet,reason,source,decided_at`. Return 409 if already recorded.
The server must re-validate every decision (never trust the device alone).

## Receipt verification (Telebirr + CBE Birr)
Uses the Payment Verification API (the verifier-api project). In Settings enter: Verifier API URL, Verifier API key,
and your own receiving wallet number.
- Requests from the management system may carry `screenshot_url` (or `screenshot`/`image_url`/`proof_url`) and/or `reference`.
- The app downloads the screenshot, sends it to `/verify-image` to read the reference, then verifies it with
  `/verify-telebirr` or `/verify-cbebirr` (chosen by the request's `provider`).
- CONFIRM only if: receipt successful, reference unused, amount equal, receiver is your wallet, payer wallet equals the request wallet, time ok.
- Anything unreadable, unsupported or unavailable (after the retry limit) becomes an EXCEPTION.
- **Verify tab** = manual check: type a reference, pick a request, see the verifier result and rule checks, then Confirm or Reject.
- The Telebirr verifier must run from inside Ethiopia (Telebirr blocks foreign servers).
- CBE Birr verification needs the receiving phone number (your wallet) - assumed to be the number the receipt was sent to.

## System ON/OFF switch
- A switch at the top of every screen turns automation ON or OFF. When OFF, SMS and proofs are recorded but nothing is
  decided automatically (agent manual actions still work). Switching ON resumes processing.
- The app reports its state to the management system (`POST /api/agent/status` with `{enabled, active, at}`) when you
  switch it and on every sync. If the response contains `{"enabled": false}` the management system pauses the app remotely;
  automation runs only when both the agent and the management system allow it. The endpoint is optional.
- Switching ON/OFF is written to the audit log.

## How it behaves
- Login uses the management system session token (stored encrypted, auto-logout on expiry or 401).
- A foreground service syncs requests, retries WAITING items and reports decisions every N seconds (default 10).
- Offline or stale sync (>60 s) = NO decision is made; payments stay pending until sync is fresh again.
- Duplicate transaction IDs are rejected, and a database PRIMARY KEY on `used_txn` blocks double confirmation.
- Every decision is written to the audit log with the rules that passed or failed.
- Only SMS from allow-listed sender IDs (Settings, default `127,telebirr`) are read. Verify the real Telebirr sender ID.

## Known limits (be aware before production)
- Not compiled or tested by the author; first build may need small fixes.
- SMS can be spoofed. For real money, also verify the transaction with an official Telebirr/merchant API or statement.
- Telebirr SMS often masks the sender number; the wallet check then cannot match. Tune `Parser` to the real message format,
  or match on transaction ID against the provider's API.
- `RECEIVE_SMS` is restricted on Google Play; distribute as a private/sideloaded APK.
- The SQLite database is app-private but not encrypted; add SQLCipher for full at-rest encryption.
- Debug-signed APK; create a release keystore for production.
