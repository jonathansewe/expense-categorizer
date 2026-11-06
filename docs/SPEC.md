# SEWE Finance V1 · Build Spec for Claude Code

Version 1.1 · 2026-11-06 · Owner: Jonathan Sewe

> **What changed in v1.1.** Thirteen patches on top of the original v1.0
> (2026-11-05). Each one is tagged in the text as `[D-xx]`, and the reason
> plus the rejected alternative for each is in `docs/DECISIONS.md`.
>
> | Patch | Decision | Sections touched |
> |---|---|---|
> | Dedupe by FITID / occurrence index | D-01 | 5, 7.1, 10 (M3) |
> | One domain, cookie and CORS rules | D-02 | 3, 6, 11 |
> | Database on Neon, not Render free | D-03 | 3, 11 |
> | Receipts on Cloudflare R2 | D-04 | 3, 5, 7.4 |
> | Passkeys web-only in M2, native in M9 | D-05 | 6, 10 |
> | Device cap and recovery at the cap | D-06 | 5, 6 |
> | AI chat by tool calling | D-07 | 6, 7.5, 10 (M7) |
> | User regex rules on google-re2 | D-08 | 7.2 |
> | Person-name masking before AI | D-09 | 6, 7.2 |
> | Voice input optional, feature-detected | D-10 | 7.5, 9 |
> | Monthly cost table | D-11 | 11.5 (new) |
> | Re-estimated timeline | D-12 | 10 |
> | LEARN.md and rewrite-yourself exercises | D-13 | 10, 12 |

---

## 1. How to use this spec

Build SEWE Finance V1 milestone by milestone (section 10), stopping after
each one for review. The owner, Jonathan Sewe, must be able to explain and
rewrite every line, so readable code beats clever code.

**Working rules for Claude Code**

1. Read this whole spec before writing code. It lives in the repo as `docs/SPEC.md`.
2. Work on one milestone at a time. Do not start the next until all its acceptance criteria pass.
3. Write tests with the feature, not after. Run the full test suite before every commit.
4. One commit per logical change, using Conventional Commits (`feat:`, `fix:`, `test:`, `docs:`).
5. Never commit secrets. Every secret lives in `.env`; ship `.env.example` with placeholder values.
6. Prefer the standard library and well-known packages. Add a dependency only when it saves real work, and note why in the commit message.
7. Comment the WHY, not the WHAT. Every module starts with a 3 to 5 line header comment explaining its job.
8. When the spec is ambiguous, choose the simplest option, write it in `docs/DECISIONS.md` (date, decision, reason, rejected alternative), and continue.
9. At the end of each milestone, write `docs/explain/M<n>.md` and the milestone's section of `LEARN.md`, and set up the rewrite exercise (section 12). `[D-13]`

**Definition of done for V1**

- Jonathan can sign up, import a real bank statement (CSV or OFX), and see categorized transactions, reports, and AI chat answers based on his own data.
- Works in a desktop browser and a phone browser; builds for iOS and Android through Expo.
- English, French and Spanish; dark and light themes.
- All tests pass in CI; the web app and API are deployed on one domain (`app.<domain>`, `api.<domain>`); the repo README has screenshots and setup steps. `[D-02]`

---

## 2. Product and V1 scope

SEWE Finance is a personal-finance app that reads your money, sorts it with
AI, and answers questions about it. V1 is read-only: it never moves money.

**Primary user:** an individual managing several bank accounts, who sends
money to family, pays cash for some purchases, and wants to understand where
money goes. The first real user is Jonathan himself.

| In V1 | Out of V1 (later versions) |
|---|---|
| Sign up, email verification, passkey login (web in V1; native in M9), TOTP two-factor, max 3 trusted devices `[D-05]` | Money transfers between accounts or banks |
| Bank statement import: CSV, OFX, QFX (PDF as a stretch goal) | Live bank connection (Plaid or similar) |
| AI categorization + user-defined categories (for example "Mom") that learn from corrections | Investment accounts and investment suggestions |
| Reports: day, week, month, year, custom range; by category; subscriptions; trends | Credit score tracking (never store SSNs) |
| Receipt scan with AI extraction, user review, cash expenses | Insurance tracking, tax reports |
| AI chat over the user's own data, with optional voice input `[D-10]` | Courses, quizzes, psychology assessment |
| Settings: theme (dark/light), language (EN/FR/ES), data export, AI provider | Fully local on-device AI model |
| Light gamification: XP, streak, 3 badges | App store release (prepared in M9, submitted after V1) |

The chat gives educational information only and must say so. It never
presents itself as a licensed financial advisor.

---

## 3. Tech stack

Decision: an Expo (React Native) app in TypeScript for web, iOS and Android,
talking to a FastAPI backend in Python, with PostgreSQL. One frontend
codebase reaches all three platforms, and the AI work stays in Python.

```
                                   ┌─────────────────────────────┐
                                   │ PostgreSQL (Neon)  [D-03]   │
                                   │ users, transactions, rules  │
┌──────────────────┐ HTTPS + JWT  ┌┴───────────────────────┐     └─────────────────────────────┘
│ Expo app         │─────────────▶│ FastAPI backend        │──▶  ┌─────────────────────────────┐
│ app.<domain>     │              │ api.<domain>   [D-02]  │     │ Cloudflare R2  [D-04]       │
│ + iOS, Android   │              │ auth, imports,         │     │ receipt images (private)    │
└──────────────────┘              │ categorizer, reports,  │     └─────────────────────────────┘
                                  │ receipts, chat         │──▶  ┌─────────────────────────────┐
                                  └────────────────────────┘     │ Claude API                  │
                                                                 │ minimal data, names masked  │
                                                                 └─────────────────────────────┘
```

The app only ever calls the API; secrets, the database, object storage and
the AI key never reach the phone or the browser.

| Option | Strength | Why not chosen |
|---|---|---|
| **Expo + FastAPI + PostgreSQL (chosen)** | One UI codebase for web and both stores; Python for AI and data parsing; reuses P1 categorizer logic (see OPEN_QUESTIONS Q3) | Two languages to learn, which is the point |
| Next.js full stack + PostgreSQL | Fastest web launch, one language | App stores need a second app or a wrapper Apple may reject |
| Streamlit (Python only) | Fastest to build | Not a real mobile app; looks like a prototype on a resume |
| Firebase or Supabase only | Little backend code | Hides the backend skills this project exists to show |

**Frontend:** Expo SDK (latest stable), Expo Router, TypeScript strict mode,
`i18next` + `react-i18next`, `react-native-svg` for charts, `expo-camera` and
`expo-image-picker` for receipts, `expo-document-picker` for statements,
`expo-secure-store` for tokens on device.

**Backend:** Python 3.12, FastAPI, SQLAlchemy 2 + Alembic migrations,
Pydantic v2, `pytest`, `argon2-cffi` for password hashing, `pyotp` for TOTP,
`webauthn` (py_webauthn) for passkeys, `ofxparse` for OFX/QFX, `google-re2`
for user regex rules `[D-08]`, `boto3` for R2 (S3-compatible) `[D-04]`, the
official `anthropic` SDK for AI.

**AI models:** set through env vars, never hard-coded.
`AI_MODEL_FAST=claude-haiku-5-5` for categorization;
`AI_MODEL_SMART=claude-sonnet-5-5` for chat and receipt vision. Wrap all AI
calls behind one `ai/provider.py` interface so another provider
(bring-your-own-key) can be added later.

**Hosting** `[D-02] [D-03] [D-04]`:

| Piece | Where | Address |
|---|---|---|
| Web build (static Expo web export) | Vercel | `https://app.<domain>` |
| API | Render web service | `https://api.<domain>` |
| Database | Neon Postgres (free plan to start) | private connection string |
| Receipt images | Cloudflare R2, private bucket | never public; short-lived signed URLs |
| Domain + DNS | one registrar; DNS can live on Cloudflare | `<domain>` |

Render's free Postgres is **not** used: free databases expire 30 days after
creation and are deleted after a 14-day grace period. Plan limits and the
backup plan are in `docs/DECISIONS.md` (D-03).

**Repo:** rename `github.com/jonathansewe/expense-categorizer` to
`sewe-finance` so the commit history shows the growth from P1. GitHub keeps a
redirect from the old name.

---

## 4. Repository layout

One monorepo with two apps. Keep the old P1 script in `legacy/` so its history stays visible.

```
sewe-finance/
  README.md            # pitch, screenshots, setup, architecture
  LEARN.md             # plain-English map of every key file, one section per milestone [D-13]
  .env.example
  .github/workflows/ci.yml
  docs/
    SPEC.md            # this document
    DECISIONS.md       # every judgment call, dated
    OPEN_QUESTIONS.md  # questions waiting on the owner
    explain/M1.md ...  # plain-language code walkthroughs (section 12)
  exercises/
    M1/ ... M9/        # rewrite-yourself exercises: README + tests (section 12) [D-13]
  legacy/              # original P1 expense-categorizer script
  api/                 # FastAPI backend
    app/
      main.py          # app factory, routers, CORS
      config.py        # settings from env vars
      db.py            # engine, session
      models/          # SQLAlchemy tables (section 5)
      schemas/         # Pydantic request/response models
      routers/         # auth, accounts, imports, transactions, categories,
                       # reports, receipts, chat, settings, gamification
      services/
        parsers/       # csv_parser.py, ofx_parser.py, pdf_parser.py
        dedupe.py      # FITID / occurrence-index keys [D-01]
        categorizer.py # rules first, then AI
        masking.py     # person-name masking before any AI call [D-09]
        subscriptions.py
        reports.py     # the only place totals are computed [D-07]
        storage.py     # R2 upload, signed URLs, delete [D-04]
        xp.py
      ai/
        provider.py    # interface
        anthropic_provider.py
        chat_tools.py  # tool definitions that wrap reports.py [D-07]
        prompts/       # one file per prompt, versioned
      security/        # passwords, totp, webauthn, tokens, encryption, devices
    alembic/
    tests/
  app/                 # Expo app (web, iOS, Android)
    app/               # Expo Router screens: (auth)/, (tabs)/accounts, reports,
                       # receipt, chat, settings; import.tsx
    components/        # Island nav, Card, Bar chart, Segmented, TxRow
    theme/             # tokens.ts (dark + light), ThemeProvider
    i18n/              # en.json, fr.json, es.json, index.ts
    lib/api.ts         # typed API client
    lib/features.ts    # platform feature detection (voice, passkeys) [D-10]
    tests/
```

---

## 5. Data model

Amounts are stored as integer cents (`amount_cents`), never floats; negative
= money out. Every table has `id` (UUID), `created_at`, `updated_at`, and
every user-owned row has `user_id`, checked on every query.

| Table | Key columns | Notes |
|---|---|---|
| users | email (unique), password_hash, email_verified_at, totp_secret_enc, display_name | `totp_secret_enc` encrypted at rest |
| user_settings | user_id, theme (`dark` / `light` / `system`), language (`en` / `fr` / `es`), currency (`USD`), ai_provider, monthly_budget_cents | One row per user |
| devices | user_id, kind (`web` / `ios` / `android`), label, user_agent_summary, last_seen_at, revoked_at | Max 3 active per user. On web, one device = one browser session record `[D-06]` |
| passkeys | user_id, credential_id, public_key, sign_count, device_id, transports | WebAuthn credentials; web only until M9 `[D-05]` |
| recovery_codes | user_id, code_hash, used_at | 10 per user, one-time use, stored hashed |
| sessions | user_id, device_id, refresh_token_hash, expires_at, scope (`full` / `device_management`) | Short-lived access JWT + rotating refresh token. `device_management` scope is used only at the device cap `[D-06]` |
| accounts | user_id, institution_name, nickname, type (`checking` / `savings` / `credit` / `cash`), last4, current_balance_cents | Only the last 4 digits, never a full account number |
| imports | user_id, account_id, filename, file_type, row_count, imported_count, skipped_duplicate_count, status, file_sha256 | `file_sha256` lets the UI warn "you already imported this exact file"; row-level dedupe is what actually prevents duplicates `[D-01]` |
| transactions | user_id, account_id, import_id, posted_on (date), description_raw, merchant, amount_cents, category_id, category_source (`rule` / `ai` / `user`), fitid (nullable), dedupe_hash, is_cash, receipt_id | **Unique on (account_id, dedupe_hash)** `[D-01]` |
| categories | user_id (null = built-in), name_key or custom_name, kind (`expense` / `income` / `transfer`), color | Built-ins have translation keys; custom ones ("Mom") store the user's text |
| category_rules | user_id, match_type (`exact` / `contains` / `regex`), pattern (max 200 chars), category_id, created_from (`user` / `chat`) | Learned from user corrections. Regex compiled with RE2 `[D-08]` |
| bank_csv_mappings | user_id, institution_name, column_map (JSON), delimiter, date_format | Remembered CSV column mapping per bank |
| receipts | user_id, object_key, merchant, total_cents, purchased_on, payment_method, raw_ai_json, status (`draft` / `confirmed`) | Image in R2 under `object_key`, never in the database and never on the API's disk `[D-04]` |
| subscriptions | user_id, merchant, typical_amount_cents, cadence_days, next_expected_on, last_seen_on | Rebuilt after each import |
| chat_messages | user_id, role (`user` / `assistant`), content, created_at | Last 50 kept per user |
| xp_events | user_id, kind, points | Badges are computed from events |

### 5.1 Deduplication `[D-01]`

Every transaction gets exactly one `dedupe_hash`. One unique constraint,
`UNIQUE (account_id, dedupe_hash)`, covers both rules below, so the database
itself refuses duplicates even if application code has a bug.

**Rule A: OFX / QFX with a FITID.** FITID is the bank's own transaction ID,
unique within an account.

```
dedupe_hash = SHA-256("fitid|" + account_id + "|" + FITID)
fitid       = FITID   (stored for traceability)
```

**Rule B: CSV, and OFX/QFX rows with no FITID.**

```
dedupe_hash = SHA-256("row|" + account_id + "|" + posted_on (YYYY-MM-DD)
                      + "|" + amount_cents + "|" + normalized_description
                      + "|" + occurrence_index)
```

- `occurrence_index` counts identical rows (same posted_on, amount_cents and
  normalized_description) within **one file**, in file order: the first is 1,
  the second is 2, and so on. Two $2.90 OMNY rides on the same day get index
  1 and 2, so both are kept.
- `normalized_description` = Unicode NFKC, then case-folded, then runs of
  whitespace collapsed to one space, then trimmed. Digits are kept. This
  function is **frozen** as `normalize_v1`. Changing it later would change
  every hash and break dedupe against old rows, so any change ships as
  `normalize_v2` with a migration.
- The `fitid|` and `row|` prefixes keep the two rules from ever colliding.

**Why overlapping statements work.** Statement A (Oct 1–15) and statement B
(Oct 10–31) both contain Oct 10–15. For any day in the overlap, both files
list the same identical rows, so they produce the same indexes (1, 2, …) and
the same hashes, and B's copies are skipped. If B lists one more identical
row than A did (A's export cut off mid-day), that extra row gets a new index
and is imported. Nothing is lost.

**Known limit, accepted for V1:** if a bank edits a pending transaction's
description between two exports, the row counts as new. The preview step
lets the user spot this before confirming.

**Import report.** After every import the API returns, and the app shows:
"X imported, Y skipped as duplicates" (both translated). The counts are
stored on `imports.imported_count` and `imports.skipped_duplicate_count`.

---

## 6. Security and privacy

The rule: store the least data possible, encrypt what is sensitive, and send
the AI only what it needs.

### 6.1 Domain, cookies and CORS `[D-02]`

- One registered domain. Web at `https://app.<domain>`, API at
  `https://api.<domain>`. They are the same *site* (same registrable domain),
  so a `SameSite=Strict` cookie set by the API is sent on requests from the app.
- **Web refresh token:** cookie `refresh_token` set by the API with
  `HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth/refresh`, and no
  `Domain` attribute (host-only on `api.<domain>`, so the browser sends it
  nowhere else).
- **Access token on web:** kept in memory only (a JS variable), never in
  `localStorage`. After a page reload the app calls `/auth/refresh` to get a
  new one.
- **Native refresh token:** stored in `expo-secure-store` and sent in the
  request body to `/auth/refresh`. Native apps don't use cookies.
- **CORS:** allow exactly one origin, `https://app.<domain>` (from
  `ALLOWED_ORIGINS`), with `allow_credentials=True`. Never `*`: browsers
  reject a wildcard combined with credentials, and it would be unsafe anyway.
- **WebAuthn:** `WEBAUTHN_RP_ID=<domain>` (the root domain, for example
  `sewe.app`); expected origin `https://app.<domain>`. Using the root domain
  lets the same passkeys work for the native apps in M9.
- **Local development:**
  - `http://localhost:8081` (Expo web) and `http://localhost:8000` (API) are
    the same site, because a site ignores the port. SameSite cookies work.
    Chrome and Firefox accept `Secure` cookies on `http://localhost`; Safari
    does not, so use Chrome or Firefox for local auth work.
  - CORS still applies, because an *origin* includes the port:
    `ALLOWED_ORIGINS=http://localhost:8081`.
  - `WEBAUTHN_RP_ID=localhost`; expected origin `http://localhost:8081`.
- **Fallback: Vercel rewrites proxying `/api`.** For Vercel preview
  deployments (`*.vercel.app`, which is not same-site with `api.<domain>`),
  or before the domain is bought, `vercel.json` rewrites
  `/api/:path*` → `https://api.<domain>/api/:path*`. The browser then sees a
  single origin, so the cookie is first-party and CORS is not needed. To
  test the same setup on a laptop, run `vercel dev`. Plain `npm run web`
  does not apply rewrites.

### 6.2 Authentication

1. Sign up with email + password (argon2id hash). Email verification by a 6-digit code, valid 10 minutes, max 5 tries.
2. TOTP two-factor is required before any bank data can be imported. Show a QR code plus 10 one-time recovery codes (stored hashed).
3. **Passkeys (WebAuthn), web only in V1 up to M9** `[D-05]`: after the first web login, offer a passkey (Face ID, Touch ID, Windows Hello, or a security key). Once a passkey exists, it is the default login **on web**. **Native (iOS, Android) uses password + TOTP** until M9, when native passkeys are added (dev build, `apple-app-site-association`, `assetlinks.json`).
4. **Device limit: 3 active devices** `[D-06]`.
   - On web, one device = one browser session record (a row in `devices`
     created at login and linked to that browser's refresh-token session).
     Clearing cookies or opening a private window creates a new device.
   - On native, one device = one app install.
   - **At the cap:** a 4th login must pass password **and** (TOTP code
     **or** one unused recovery code, which is then marked used). That login
     gets only a `device_management` session, which can call `GET /devices`
     and `DELETE /devices/{id}` and nothing else. The app shows the device
     list, the user revokes one, and only then is a full session issued. This
     keeps a user who cleared cookies from being locked out, without making
     the cap pointless.
   - Revoking a device deletes its sessions and passkeys immediately.
5. Tokens: access JWT valid 15 minutes; refresh token valid 30 days, rotated on every use. A reused (already-rotated) refresh token revokes that device's whole session family.
6. Rate-limit login, verification, TOTP and recovery-code endpoints (for example 5 tries per minute per IP and per account).

### 6.3 Data protection

- Never collect or store SSNs, full account numbers, card numbers, or bank passwords. The statement parser drops account numbers and keeps only the last 4 digits.
- Encrypt sensitive columns with AES-256-GCM (`cryptography` package); the key comes from `DATA_ENCRYPTION_KEY` in env, never the repo.
- HTTPS only; HSTS on; strict CORS allowing only the app's origin (6.1).
- Uploaded statements are parsed in memory or a temp file, then deleted; only the parsed rows and a SHA-256 of the file are kept. Nothing user-owned is ever kept on the API's local disk: free and small hosts have ephemeral filesystems. `[D-04]`
- Receipt images live in a **private** R2 bucket under `receipts/<user_id>/<uuid>.<ext>`. The app gets a signed GET URL valid 5 minutes. `[D-04]`
- Every query filters by the authenticated `user_id`. Add a test, on every router, that user A can never read user B's data.
- Log events, never log transaction descriptions, amounts, tokens, or email codes.
- Account deletion in Settings erases all user rows and every R2 object under that user's prefix within the same request.

### 6.4 What the AI sees

- **Masking first** `[D-09]`: before *any* text leaves the backend for the AI, `services/masking.py` replaces person names in peer-to-peer transfer descriptions with `[PERSON]`. It covers at least Zelle, Venmo and Cash App patterns, for example:
  - `Zelle payment to JANE DOE 12345` → `Zelle payment to [PERSON]`
  - `VENMO *JOHN SMITH` → `VENMO *[PERSON]`
  - `Cash App*Jane Doe` → `Cash App*[PERSON]`

  Masking applies only to what is sent to the AI. The database keeps the
  original description, so the user's own rules (for example "JANE DOE" →
  "Mom") still match, because rules run on the backend before any AI call.
- Categorization sends only the masked merchant or description text (account numbers stripped), amount, date, and the list of category names.
- Chat sends the question and the results of the report tools the model asks for (7.5). It never sends the raw transaction history. Tool results contain totals, category names and masked merchant names only. `[D-07]`
- Receipts send the image only after the user taps Scan.
- Settings shows this sentence, in all 3 languages:
  > "SEWE sends the AI only what it needs: masked descriptions to sort transactions, report totals to answer your questions, and a receipt image when you tap Scan. Names of people you pay are replaced with [PERSON]. Your account numbers, passwords and full history never leave our servers."

---

## 7. Features

Each feature below lists its exact behavior; the milestones in section 10 decide the build order.

### 7.1 Statement import

- Accept `.csv`, `.ofx`, `.qfx` up to 5 MB. PDF is a stretch goal (M8): extract text with `pdfplumber`, then let the AI turn lines into rows, and always show the preview.
- CSV: auto-detect the delimiter and the date, description and amount columns (also split debit/credit columns). If detection is unsure, show a column-mapping step and remember the mapping per bank (`bank_csv_mappings`) for next time.
- OFX/QFX: read FITID for every transaction when present (5.1). `[D-01]`
- Flow: pick file, choose or create the account, preview the first 20 rows with AI categories and a duplicate flag on each row, confirm, import.
- **After confirm, show the import report: "X imported, Y skipped as duplicates."** `[D-01]`
- If the file's SHA-256 matches an earlier import, warn before the preview ("You imported this exact file on <date>"). The user can still continue; row-level dedupe guarantees 0 new rows.
- After import: run categorization, rebuild subscriptions, recompute balances, award XP.

### 7.2 Categorization

Run in this order and stop at the first match:

1. The user's own `category_rules` (exact, then contains, then regex). These run on the **unmasked** description, on the backend.
2. Built-in keyword rules ported from the P1 categorizer (for example "OMNY" or "MTA" means Transport). *P1 has no keyword rules today; see OPEN_QUESTIONS Q3.*
3. AI: mask the description (6.4) `[D-09]`, then send uncategorized rows in batches of 50 to `AI_MODEL_FAST`, asking for strict JSON `[{"id": ..., "category": ...}]` limited to the user's category list; reject and retry once on invalid JSON; anything still unknown becomes "Uncategorized".

When the user changes a category, ask "Apply to all from this merchant?" and,
if yes, create a rule. Custom categories: the user can create one ("Mom"),
bulk-select transactions into it, or say it in chat ("these transfers were
for Mom"); the chat proposes the change and the user must confirm before
anything is saved.

**User regex rules** `[D-08]`:

- Compiled with **`google-re2`** (`import re2`), never Python's `re`. RE2 matches in linear time, so no pattern can freeze the server (no ReDoS).
- Max pattern length: **200 characters**.
- Invalid patterns are rejected at save time with HTTP 422 and a translated error: `{"error": {"code": "rule_pattern_invalid", "message": "..."}}`. Over-length patterns get `rule_pattern_too_long`. RE2 does not support backreferences or lookaround; those give `rule_pattern_unsupported`, and the message says so in plain words.
- Matching is case-insensitive.

### 7.3 Reports

- Periods: day, week, month, year, custom date range; filters by account, category, type (expense, income, transfer).
- Views: total for the period, bar chart by category, trend line over the last 6 or 12 periods, top merchants, income vs. spending.
- Subscriptions: same merchant, amount within 10%, interval of 7, 14, 30 or 365 days (plus or minus 3), seen at least twice.
- Export: CSV of the filtered transactions and a PDF report (server-side, `reportlab`).
- `services/reports.py` is the **single source of truth** for every total. The Reports screen and the AI chat call the same functions. `[D-07]`

### 7.4 Receipts

- Camera or photo library. The API uploads the image to R2 (`services/storage.py`) and stores only the `object_key` `[D-04]`. Then it sends the image to `AI_MODEL_SMART` and asks for JSON: merchant, total, date, payment method, line items (optional).
- The user reviews every field before saving. If the payment is cash, create a cash transaction; otherwise offer to match an existing card transaction (same amount, date within 3 days).
- Draft receipts never confirmed are deleted from R2 after 7 days.

### 7.5 AI chat `[D-07] [D-10]`

- **Tool calling, not free-form math.** The model gets a set of tools in `ai/chat_tools.py`, each a thin wrapper over one function in `services/reports.py`:
  - `get_period_summary(period, from, to, account?)` → total spent, total income, net
  - `get_spending_by_category(from, to, account?)`
  - `get_trend(category?, period, count)`
  - `get_top_merchants(from, to, limit)`
  - `get_subscriptions()`
  - `get_category_list()`
- The model decides which tools to call, the backend runs them **scoped to the authenticated user**, and the model only puts the results into words. **The model never computes totals itself.** The system prompt says so, and tool results already include every number the answer could need (sums, differences, percentages computed by `reports.py`).
- Tool results carry each money value twice: `amount_cents` (integer) and `display` (pre-formatted with `Intl`-equivalent locale rules for the user's language). The model is told to quote `display` exactly.
- **Grounding check:** before the reply is sent, the backend pulls every money figure out of the answer text and checks each one appears in that turn's tool results. If one doesn't, the backend retries once with a correction note; if it still fails, it replies with a translated "I couldn't verify that number. Please check the Reports screen."
- Tool definitions use `strict: true`. Current Claude models reject forced `tool_choice` (`any` / `tool`), so use `tool_choice: auto` and steer from the system prompt.
- Answers come in the user's selected language, cite the numbers used, and end every money suggestion with the educational-only notice.
- **Voice input is optional** `[D-10]`: speech-to-text on device (`expo-speech-recognition` on native, the browser's Web Speech API on web). Never send audio to the server in V1. `lib/features.ts` checks for support at runtime; if it's missing (for example Firefox, or a native build without the module), **the mic button is hidden, not disabled**, and text input works as normal.
- Suggested questions shown as chips, translated.

### 7.6 Settings

Theme (dark, light, follow system), language (English, French, Spanish),
import statement, connected accounts, export all data (JSON + CSV zip),
trusted devices, passkeys (web only until M9), two-factor, recovery codes
(regenerate), AI provider (SEWE AI default; bring-your-own-key is a disabled
"coming soon" row in V1), the AI privacy sentence (6.4), delete account.

### 7.7 Gamification

- XP: +50 first import of a month, +20 per confirmed receipt, +10 per day opened (streak), +15 per category correction.
- Level = floor(total XP / 500) + 1.
- 3 badges in V1: "First import", "7-day streak", "10 receipts".

---

## 8. API endpoints

All routes live under `/api/v1`, return JSON, and require auth except the
`auth` group. FastAPI's generated docs at `/docs` must stay accurate.

| Group | Method and path | Purpose |
|---|---|---|
| auth | POST `/auth/signup`, `/auth/verify-email`, `/auth/login`, `/auth/refresh`, `/auth/logout` | Account and session |
| auth | POST `/auth/totp/setup`, `/auth/totp/verify`, `/auth/recovery/verify`, `/auth/recovery/regenerate` | Two-factor and recovery codes `[D-06]` |
| auth | POST `/auth/passkey/register/options`, `/register/verify`, `/login/options`, `/login/verify` | Passkeys (web until M9) `[D-05]` |
| devices | GET `/devices`, DELETE `/devices/{id}` | Trusted devices (max 3). The only routes a `device_management` session may call `[D-06]` |
| accounts | GET, POST `/accounts`; PATCH, DELETE `/accounts/{id}` | Accounts and balances |
| imports | POST `/imports/preview` (multipart), POST `/imports/{id}/confirm`, GET `/imports` | Two-step import; confirm returns `{imported, skipped_duplicates}` `[D-01]` |
| transactions | GET `/transactions?from&to&account&category&q&page`; PATCH `/transactions/{id}`; POST `/transactions/bulk-categorize`; POST `/transactions/cash` | List, fix, bulk move, cash entry |
| categories | GET, POST `/categories`; PATCH, DELETE `/categories/{id}`; GET, POST, DELETE `/category-rules` | Categories and learned rules |
| reports | GET `/reports/summary?period&from&to`, `/reports/by-category`, `/reports/trend`, `/reports/subscriptions`, `/reports/export?format=csv or pdf` | Report data and export |
| receipts | POST `/receipts/scan` (image), PATCH `/receipts/{id}`, POST `/receipts/{id}/confirm`, GET `/receipts/{id}/image-url` | Scan, review, save, signed image URL `[D-04]` |
| chat | POST `/chat` (streamed), GET `/chat/history`, DELETE `/chat/history` | AI chat |
| settings | GET, PATCH `/settings`; GET `/settings/export`; DELETE `/me` | Preferences, export, delete account |
| gamification | GET `/progress` | XP, level, streak, badges |

Errors use one shape: `{"error": {"code": "string", "message": "string"}}`.
Error codes are stable keys so the app can translate them.

---

## 9. UI spec

Match the "Final V1" artboard on the SEWE Finance design canvas: dark theme
from direction B, light theme and reports from direction C, floating island
navigation from C. *(The canvas URL was a link in the v1.0 PDF; add it here.
See OPEN_QUESTIONS.md.)*

**Theme tokens** (in `app/theme/tokens.ts`; components read tokens, never raw hex)

| Token | Dark | Light |
|---|---|---|
| bg | #0F1115 | #F3F5FA |
| surface | #191C22 | #FFFFFF |
| line | #262A31 | #E6EAF2 |
| text | #F2F3F5 | #0B1023 |
| muted | #A0A6B1 | #5A6275 |
| chip / chipText | #262A31 / #D4D8DE | #EEF1F7 / #3B4357 |
| accent / onAccent | #B8F24A / #0F1115 | #2340D3 / #FFFFFF |
| accentSoft / accentSoftText | #262A31 / #B8F24A | #E3E8FB / #1A31A8 |
| hero card / hero text | #191C22 / #F2F3F5 | #2340D3 / #FFFFFF |
| island / island active | #20242B / #B8F24A | #0B1023 / #2340D3 |
| danger | #FF8A7A | #C2261A |

**Type:** Space Grotesk 600 to 700 for headings and big numbers, Inter 400 to 600 for body. Sizes: big number 40, title 21, section 16, body 15, small 12 to 13, nav 10.

**Shape:** cards radius 20 to 24, pills 999, inputs 12, spacing on an 8 px grid, touch targets at least 44 px.

**Navigation:** a floating island 12 px from the sides and 16 px from the bottom, 68 px tall, 5 tabs: Accounts, Reports, Receipt, AI Chat, Settings. The active tab is an accent pill. Screen content gets 112 px bottom padding so the island never covers it. On web wider than 900 px, keep a centered 480 px phone-width column for V1.

**Screens**

1. Accounts: empty state (no data, Import a statement, Connect a bank which shows "coming soon"), then hero total balance with account tiles, Spent and Mom-style top-custom-category tiles, XP and streak card, recent transactions with category chips and a "+ Import" button.
2. Import: file picker zone (CSV, OFX/QFX, PDF), preview with AI categories and duplicate flags, confirm, then the import report ("X imported, Y skipped as duplicates"). `[D-01]`
3. Reports: period pills (Day, Week, Month, Year), big total, vertical bar chart by category, subscriptions list, export button.
4. Receipt: camera frame, Take photo / From Photos, editable AI-read fields, Add to expenses.
5. AI Chat: bubbles, suggestion chips, text input with a mic button **only when voice is supported** `[D-10]`, educational notice.
6. Settings: Appearance (theme, language), Data, Security (devices, passkeys on web, two-factor, recovery codes), AI (provider, privacy sentence), app version.
7. Device limit screen `[D-06]`: shown after a login at the cap. Lists the 3 devices (label, kind, last seen); the user must revoke one to continue.

**Theme switching:** a sun or moon button in the header plus the Settings control; default follows the system; the choice is saved to `user_settings` and applied instantly with no reload.

**Languages:** every visible string comes from `i18n/en.json`, `fr.json`, `es.json`; no hard-coded text in components. A test fails if a key exists in one file and not the others. Money uses `Intl.NumberFormat` with locales `en-US`, `fr-FR`, `es-US` and currency USD; dates use the same locales. AI answers follow the selected language.

**Accessibility:** WCAG AA contrast in both themes, labels on every icon button, screen-reader names on charts, works with system font scaling up to 130%.

---

## 10. Milestones `[D-12] [D-13]`

Nine milestones, built in this order. Each ends with passing tests, a commit,
`docs/explain/M<n>.md`, the milestone's `LEARN.md` section, and one
rewrite-yourself exercise (section 12).

**Estimating assumptions:** 20 hours per week; a beginner in TypeScript,
React Native and FastAPI who knows Python basics. "Build" = Claude Code
builds it and Jonathan reviews, runs and debugs it with Claude. "Learn" =
reading `explain/M<n>.md` and `LEARN.md`, plus doing the rewrite exercise.

| # | Milestone | v1.0 est. | Build h | Learn h | Total h |
|---|---|---|---|---|---|
| M1 | Skeleton | 3 | 9 | 3 | 12 |
| M2 | Auth and security (web passkeys) | 6 | 16 | 4 | 20 |
| M3 | Import, dedupe and transactions | 6 | 13 | 3 | 16 |
| M4 | Categorization, RE2 rules, masking | 5 | 11 | 3 | 14 |
| M5 | Reports | 5 | 11 | 2 | 13 |
| M6 | Receipts and R2 | 4 | 9 | 2 | 11 |
| M7 | AI chat with tools | 4 | 11 | 3 | 14 |
| M8 | Settings, gamification, polish | 4 | 11 | 2 | 13 |
| M9 | Ship + native passkeys | 3 | 15 | 2 | 17 |
| | **Subtotal** | **40** | **106** | **24** | **130** |
| | Buffer for debugging and surprises (+25%) | | | | 32 |
| | **Total** | | | | **≈162 h ≈ 8 weeks at 20 h/week** |

Realistic range: **7 to 10 weeks.** The biggest risks to the schedule are
M2 (auth always takes longer than planned), M9 (store accounts, EAS builds
and native passkey domain files involve waiting on outside systems), and the
first week of M1 (new toolchain).

1. **M1 · Skeleton (12 h).** Monorepo, `legacy/` with the P1 script, FastAPI app with `/health`, Postgres via Docker Compose, Alembic set up, Expo app with Expo Router, theme tokens, i18n with 3 empty-but-matching files, island nav with 5 placeholder tabs, CI running lint + tests, `LEARN.md` and `exercises/` scaffolding.
   - Done when: `docker compose up` + `npm run web` shows the island; switching theme and language works; CI is green.
   - Rewrite exercise: `resolveTheme(preference, systemScheme)` in `app/theme/`.

2. **M2 · Auth and security (20 h).** Signup, email code, login, TOTP, recovery codes, **web passkeys**, refresh rotation with reuse detection, 3-device limit with the `device_management` flow, rate limits, cookie and CORS setup from 6.1. Native login = password + TOTP. `[D-05] [D-06]`
   - Done when: tests cover wrong password, expired code, reused refresh token, 4th device refused a full session, 4th device admitted with TOTP or a recovery code into `device_management` only, revoke-then-login works, a used recovery code is rejected the second time, CORS rejects any other origin, and user A cannot read user B.
   - Rewrite exercise: `verify_totp(secret, code, at_time)` in `security/totp.py`.

3. **M3 · Import, dedupe and transactions (16 h).** CSV + OFX/QFX parsers, FITID capture, column mapping, preview, confirm, dedupe per 5.1, import report, accounts and balances, transactions list. `[D-01]`
   - Done when:
     - a real statement from Jonathan's bank imports;
     - **two identical $2.90 OMNY rides on the same day are both kept;**
     - **re-importing the same file adds 0 rows** and reports them all as skipped duplicates;
     - **two overlapping statement files produce no duplicates and no losses** (total rows = rows in the union of both periods);
     - an OFX file with FITIDs dedupes by FITID, and one without FITIDs falls back to Rule B;
     - parser tests use sample files in `api/tests/fixtures/` with fake data.
   - Rewrite exercise: `compute_dedupe_hashes(rows, account_id)` in `services/dedupe.py`.

4. **M4 · Categorization (14 h).** User rules, built-in rules, AI batches, user corrections creating rules, custom categories, bulk move, RE2 regex rules, person-name masking. `[D-08] [D-09]`
   - Done when: at least 85% of the fixture rows land in the right category; a correction applied to a merchant re-categorizes its other rows; a catastrophic-backtracking pattern like `(a+)+$` either compiles in RE2 and matches in linear time or is rejected (never hangs); a 201-character pattern is rejected; a pattern with a lookahead is rejected with `rule_pattern_unsupported`; a test asserts that no string sent to the AI mock contains a fixture person's name; AI calls are mocked in tests.
   - Rewrite exercise: `categorize(row, user_rules, builtin_rules)` in `services/categorizer.py`.

5. **M5 · Reports (13 h).** Summary, by category, trend, subscriptions, CSV and PDF export, Reports screen with period pills and bar chart.
   - Done when: report numbers match a hand-computed fixture to the cent; the chart renders in both themes.
   - Rewrite exercise: `detect_subscriptions(transactions)` in `services/subscriptions.py`.

6. **M6 · Receipts (11 h).** Camera and library, R2 upload and signed URLs, AI extraction, review form, cash transactions, card matching, draft cleanup. `[D-04]`
   - Done when: a photo of a real receipt fills at least merchant, total and date; editing and confirming creates one transaction; the image is in R2 and not on local disk; deleting the account deletes the user's R2 objects (tested against a local S3-compatible mock).
   - Rewrite exercise: `match_receipt_to_transaction(receipt, candidates)`.

7. **M7 · AI chat (14 h).** Report tools, tool-calling loop, grounding check, streamed answers, history, suggestion chips, optional voice input, answers in the selected language. `[D-07] [D-10]`
   - Done when:
     - **for every question in a fixture set (for example "How much did I spend on food this month?"), every money figure in the chat answer equals the Reports value to the cent.** In CI the model is a recorded mock; a separate, manually run eval script checks the live model;
     - an answer containing a number not in the tool results is caught by the grounding check;
     - the mic button is hidden when voice is unsupported (tested with the feature flag off).
   - Rewrite exercise: `find_ungrounded_amounts(answer_text, tool_results)` in `ai/chat_tools.py`.

8. **M8 · Settings, gamification, polish (13 h).** Full Settings screen, export all data, delete account, XP and badges, empty, loading and error states, PDF import as a stretch.
   - Done when: every string is translated in all 3 languages; delete account leaves no rows and no R2 objects for that user.
   - Rewrite exercise: `compute_progress(xp_events, today)` in `services/xp.py`.

9. **M9 · Ship + native passkeys (17 h).** Buy and wire the domain, deploy API (Render), database (Neon) and web (Vercel); production env vars; nightly backup job; README with screenshots and a demo GIF; privacy policy page; EAS build profiles for iOS and Android; **native passkeys**: dev build, `apple-app-site-association` and `assetlinks.json` served from `<domain>`. `[D-02] [D-03] [D-05]`
   - Done when: Jonathan uses the live web app at `https://app.<domain>` on his phone with his own statement; `eas build` succeeds for both platforms; a passkey created on web signs in on a native dev build; the backup job has produced at least one restorable dump (restore tested once).
   - Rewrite exercise: `validate_production_settings(settings)` in `config.py`.

---

## 11. Testing, CI, deployment, app stores

### 11.1 Tests

- Backend: `pytest` with a throwaway Postgres (Docker or `testcontainers`); target at least 80% coverage on `services/` and `security/`. Mock every AI call with recorded fixtures; no CI test ever calls a paid API. R2 is mocked with a local S3-compatible server (for example `moto`).
- Frontend: Jest + React Native Testing Library for components; one Playwright smoke test on the web build (sign up, import fixture CSV, see the Reports total).
- Fixtures use invented merchants, amounts and person names. Never commit a real statement.
- Every rewrite-yourself exercise has its tests in `exercises/M<n>/` (section 12).

### 11.2 CI (GitHub Actions, on every push and pull request)

1. Backend: `ruff` lint, `mypy`, `pytest`.
2. Frontend: `eslint`, `tsc --noEmit`, `jest`, i18n key-parity check.
3. Block the merge if any step fails.

### 11.3 Deployment

- Environments: `local` and `prod` for V1. Migrations run on deploy (`alembic upgrade head`).
- Required env vars: `DATABASE_URL`, `JWT_SECRET`, `DATA_ENCRYPTION_KEY`, `ANTHROPIC_API_KEY`, `AI_MODEL_FAST`, `AI_MODEL_SMART`, `EMAIL_PROVIDER_KEY`, `ALLOWED_ORIGINS`, `WEBAUTHN_RP_ID`, `WEBAUTHN_ORIGIN`, `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET`, `COOKIE_SECURE`.
- Set a monthly spend cap on the AI API key and a per-user daily limit on chat and receipt calls.

### 11.4 Database backups `[D-03]`

- Provider: Neon Postgres. Neon's own restore history on the free plan is short, so it is not the backup plan.
- **Backup plan:** a nightly GitHub Actions job runs `pg_dump --format=custom`, encrypts the dump, and uploads it to a separate private R2 bucket (`sewe-backups`). Keep 14 nightly + 8 weekly dumps.
- Restore is tested once in M9 (into a scratch Neon branch) and after any schema-heavy milestone.

### 11.5 Monthly costs `[D-11]`

Estimates as of 2026-11-06; verify each price before paying. AI usage
assumes one heavy user: about 300 transactions, 100 chat questions and 30
receipts per month.

| Item | Free-tier setup | Small paid setup | Notes |
|---|---|---|---|
| Domain | ≈ $1/mo (≈ $10–15/yr) | ≈ $1/mo | Required in both setups (D-02) |
| Web hosting (Vercel) | $0 (Hobby) | $0, or $20 (Pro) | Hobby is for non-commercial personal use only; move to Pro if SEWE ever charges money |
| API hosting (Render) | $0 (free web service) | $7 (Starter) | **Free sleeps after 15 min idle; the next request waits ≈ 1 min (cold start).** Starter stays on |
| Database (Neon) | $0 (1 GB per project, 100 compute-hours/mo) | $0 until 1 GB | Compute suspends after 5 min idle; the first query after that wakes it (short delay) |
| Storage (Cloudflare R2) | $0 (10 GB, free egress) | $0 | Receipts + backups fit easily in 10 GB |
| Email (verification codes) | $0 (provider free tier) | $0 | Provider not chosen yet (OPEN_QUESTIONS) |
| AI: categorization (Haiku 5.5, $0.10 / $0.50 per M tokens) | ≈ $0.01 | ≈ $0.01 | ~12k input + ~5k output tokens/mo |
| AI: chat (Sonnet 5.5, $2 / $10 per M tokens) | ≈ $2.10 | ≈ $2.10 | ~8k input + 500 output tokens per question, incl. tool round-trips |
| AI: receipts (Sonnet 5.5) | ≈ $0.21 | ≈ $0.21 | ~2k input + 300 output tokens per receipt |
| **Monthly total** | **≈ $3.30** | **≈ $10.30** | Set the AI key's spend cap at $10/mo |
| *Later, app stores* | | | Apple Developer $99/yr; Google Play $25 one-time |

**Recommendation:** develop on the free setup. Switch the API to Render
Starter ($7) when other people start using it, because a 1-minute cold start
makes the app feel broken.

### 11.6 App stores (after V1 is live on the web)

- Apple Developer Program and a Google Play developer account in Jonathan's name.
- Google Play: new personal accounts must run a closed test with at least 12 testers for 14 continuous days before production. Recruit testers during M7 so the clock starts at M9.
- Both stores need a privacy policy URL, a data-safety form that matches section 6, screenshots from both themes, and a demo login for reviewers. The demo account must be able to pass TOTP; give reviewers a dedicated account plus instructions.
- Show the educational-only notice on the store listing and in onboarding.

---

## 12. Explanation and learning deliverables for Jonathan `[D-13]`

The goal is that Jonathan can rebuild every milestone himself without the AI.
Each milestone produces three things.

### 12.1 `LEARN.md` section (repo root)

One section per milestone, appended at the end of that milestone, titled
`## M<n> · <name>`. For **each key file** created or heavily changed in the
milestone, write 2–4 plain-English sentences: what the file is for, what
would break if it were deleted, and which other files call it. No code in
this file. It's the map, not the walkthrough.

### 12.2 `docs/explain/M<n>.md`

1. **What was built** in 3 sentences of plain English.
2. **The request path:** one real example traced end to end (for example: tap Import, `lib/api.ts`, `POST /imports/preview`, router, parser, database, response, screen).
3. **Key functions:** for each new file, its 3 most important functions and why each exists. (The plain "what this file does" summary lives in `LEARN.md`.)
4. **Key concepts** used for the first time (for example JWT, WebAuthn, SQL index, React state), each as: term, one-sentence definition, where it appears in the code.
5. **Commands** run in that milestone, each with what every part of the command does.
6. **Decisions and trade-offs** taken, linked to `docs/DECISIONS.md`.
7. **Interview questions:** 5 questions about the milestone with short model answers.
8. **Extra exercises:** 2–4 small tasks Jonathan can do alone (for example: add a "Gym" built-in category with a test).

### 12.3 Rewrite-yourself exercise (`exercises/M<n>/`)

One key function per milestone (named in section 10) that Jonathan rewrites
from scratch, with no AI help.

- `exercises/M<n>/README.md`: the function's signature, what it must do, edge cases, and hints.
- `exercises/M<n>/test_<function>.py` (or `.test.ts`): the tests his version must pass. They're the same tests the real function passes, so the exercise grades itself.
- How to run: one command in the README (for example `pytest exercises/M3 -q`), which runs the tests against `exercises/M<n>/my_<function>.py`.
- Jonathan's version stays in `exercises/` and does not replace the production function unless he chooses to swap it in.

| Milestone | Function | What the tests check |
|---|---|---|
| M1 | `resolveTheme(preference, systemScheme)` | `system` follows the OS; explicit `dark`/`light` override it; unknown values fall back to `system` |
| M2 | `verify_totp(secret, code, at_time)` | RFC 6238 test vectors; accepts ±1 time step; rejects 2 steps away; rejects non-6-digit input |
| M3 | `compute_dedupe_hashes(rows, account_id)` | Two identical OMNY rides get different hashes; same file twice gives identical hashes; overlap case; FITID rule wins when present |
| M4 | `categorize(row, user_rules, builtin_rules)` | Order: exact, contains, regex, built-in; first match wins; `None` when nothing matches |
| M5 | `detect_subscriptions(transactions)` | Monthly within 10% and ±3 days detected; seen once is not a subscription; 5-day drift is not |
| M6 | `match_receipt_to_transaction(receipt, candidates)` | Same amount within 3 days matches; closest date wins; no candidate returns `None` |
| M7 | `find_ungrounded_amounts(answer_text, tool_results)` | Finds `$12.40` missing from results; accepts amounts present in `display` form; handles fr-FR `12,40 $US` |
| M8 | `compute_progress(xp_events, today)` | Level formula; streak breaks after a missed day; badges unlock at thresholds |
| M9 | `validate_production_settings(settings)` | Refuses to start with a missing env var, `COOKIE_SECURE=false`, or a wildcard in `ALLOWED_ORIGINS` |

Write all of these for a computer science student who knows Python basics and
is new to TypeScript, React Native and FastAPI.
