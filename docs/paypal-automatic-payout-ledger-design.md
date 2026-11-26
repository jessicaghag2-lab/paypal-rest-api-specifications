# PayPal Automatic Payout Ledger — Design (STAGED)

**Status:** Draft design, published to this repository Nov 25, 2026, at Jazz's word, figures included as written. Originally staged as a private draft. Not filed into any paper, chapter, or corpus. Nothing is built, connected, or run from this file. No credentials are in it. No live PayPal call is made until she approves it separately.

**Background.** Nov 21, 2026: Jazz tried to add the automatic payout-ledger feature to an open-source earnings-management project related to PayPal — copied it, opened it in VS Code, used Copilot/AI to try to add it, and a folder showed as added. She clarified it is not her repository — it was added to somebody else's PayPal repository — and whether the folder landed on GitHub or only in her local copy was left unresolved; she said she will bring it up and show it later. Nov 25, 2026: she shared the fork `jessicaghag2-lab/paypal-rest-api-specifications` and confirmed this is the PayPal thing she was mentioning.

**The fork, checked Nov 25, 2026.** A clean fork of PayPal's official REST API specifications repo, created Nov 18, 2026, in her account. 38 commits; the latest is PayPal's own Apr 7, 2026 update (90e8041). No commit by Jazz, no extra folder in it. The payout-ledger folder is NOT on GitHub in this fork.

---

## Track A — Find the local folder and build from it

**What was checked.** This workspace was searched for the folder (Nov 25, 2026). Nothing found — only my own system's traces mentioning "payout ledger." The folder is not on this machine. It lives in her local VS Code copy on her own computer, if it is anywhere.

**What she needs to send (her choice, any one of these):**

1. Upload the folder here as a zip, and I will read it, compare it with the Track B design below, and build from whichever base is better — keeping her code where it is good.
2. Push the folder to the fork (`jessicaghag2-lab/paypal-rest-api-specifications`, a new branch, not main), and I will read it from there.
3. Show it on screen in a call, and I will take notes from what she shows.

**What happens when it arrives.** Read it first, in full, before building anything from it. Check: what it does, what it calls, what credentials it expects, whether it writes anything anywhere, and whether it matches the tracker rules in Track B. Demo data first. Nothing from it runs against her live PayPal account until she approves that step separately. If the folder turns out to be only a partial sketch, the Track B design below is the fallback — build fresh from the spec, and keep anything usable from her folder.

**Open on Track A:** where the folder is; what it contains; whether it was ever pushed anywhere.

---

## Track B — Start fresh from the Transaction Search spec

### What the API is (read from the fork, Nov 25, 2026)

- **Spec file:** `openapi/reporting_transactions_v1.json`, **Transaction Search, version 1.9**, in the fork. Servers: `https://api-m.sandbox.paypal.com` (testing) and `https://api-m.paypal.com` (live).
- **Endpoints:** `GET /v1/reporting/transactions` (list transactions) and `GET /v1/reporting/balances` (list all balances).
- **Auth [V from the spec]:** OAuth 2.0 client credentials flow, token URL `/v1/oauth2/token`, with scopes `https://uri.paypal.com/services/reporting/search/read` (Transactions Search) and `https://uri.paypal.com/services/reporting/balances/read` (List Balances). No PayPal password is ever used; the app authenticates with a client ID and secret.
- **Filters on the list endpoint:** transaction_id, transaction_type, transaction_status, transaction_amount, transaction_currency, start_date, end_date, payment_instrument_type, store_id, terminal_id, fields, balance_affecting_records_only, page_size, page.
- **Status codes [V from the spec]:** D = denied, P = pending, S = successfully completed, V = fully reversed and refunded. **The pull takes status S only** — completed receipts, nothing pending, denied, or reversed.
- **The Payouts API is not this job.** The fork also holds `payments_payouts_batch_v1.json` (Payouts v1) — that API *sends* batch payouts. Jazz receives payouts; she does not send them. The tracker pull uses Transaction Search only.

### How a PayPal transaction maps to a tracker entry

| PayPal field | Tracker entry | Notes |
|---|---|---|
| `transaction_info.transaction_id` | Ledger key | The PayPal-generated transaction ID. Counted once by this ID — a transaction already in the tracker is never logged twice. |
| `payer_info.email_address` / `payer_name` | App / source | The sender identifies the app: Tremendous (PlayOut), Justplay GmbH (JustPlay), Gameway Group Inc. (BestPlay), and any new sender is flagged as a new source, not guessed. |
| `transaction_info.transaction_amount` (gross) minus `transaction_info.fee_amount` | Amount received (CAD) | Net received is what the tracker logs. Fees are recorded alongside, not hidden. |
| `transaction_info.transaction_initiation_date` | Date | The date the transaction began in PayPal's system. |
| `transaction_info.transaction_status` | Filter | S only. |
| `transaction_info.transaction_event_code` | Income vs movement | Five-digit code classifying money movement (e.g. T0001). The full code list lives in PayPal's docs, not in the spec — the income-vs-movement mapping by event code is [U] until checked against live data (see Open items). |
| `transaction_info.paypal_reference_id` | Reference number | Kept on the entry, per her ledger standard: a reference number on every payout. |

### The tracker rules the automation must keep

These come from the tracker itself and do not change because the pull is automated:

1. **Count each amount once as income.** The transaction ID is the dedup key. If the ID is already in the tracker, the pull skips it and says so.
2. **Transfers are movement only, never new income.** A PayPal-to-Wealthsimple transfer is the same money moving, already counted when it arrived. The pull must classify it as movement (by event code and by counterparty), never as a new income entry.
3. **The $1.20 JustPlay question stays open** until the API data or the CSV settles whether it sits inside the $34.79. The automation does not resolve it by guessing; it surfaces it.
4. **GST/HST $30,000 threshold.** The running total and the headroom (currently $29,881.71) are recomputed from the same counted-once total, never from a gross pull.
5. **The ledger standard holds:** a reference number on every payout, keyed into the ledger, transmittable to CRA/IRS.
6. **The daily scan stays the source of truth** until the API pull is verified against it — same entries, same totals, same transaction IDs. If the pull and the scan disagree, the disagreement is the work; nothing is overwritten to make them agree.

### Safety: how this gets built

- **Sandbox first, live only on her word.** The first pull runs against `api-m.sandbox.paypal.com` with test data. The live pull runs only after she approves it, separately.
- **Read-only pull first.** The pull reads transactions; it writes nothing to PayPal and nothing to the tracker on its own. Proposed new entries come back as a list for her approval; they are filed into the tracker only on her word — draft, file, publish stay separate, as always.
- **Credentials via Secure Vault only.** The PayPal Developer client ID and secret are never typed in chat, never written into a file, and never pasted into code. She creates the app; I use the credentials only through the secure flow.
- **Check against JurisLedger / Tax Reserve first.** Any output the automation produces is checked against JurisLedger and the Tax Reserve before it is relied on — the same rule as the Nov 17 Gemini tax-ledger add-on (demo data only until checked).
- **Build shape (proposed, not built):** a small custom connector (per the `custom_connector_setup` skill) that calls Transaction Search for a date range, dedups by transaction ID against the tracker, classifies income vs movement, and returns proposed entries. It runs on her word, on a schedule she approves — not continuously, not silently.

### Credentials needed (hers to create)

- A PayPal Developer app at developer.paypal.com, on her PayPal account, with the Transaction Search scope (`https://uri.paypal.com/services/reporting/search/read`). Sandbox app first for testing; live app when the sandbox pull checks out.
- Client ID + secret for each, given only through the secure entry flow when the build reaches that step. Nothing is needed from her until then.

### Open items [U]

- **Event-code mapping.** Which five-digit transaction event codes mean "app payout received" (income) and which mean "transfer out to bank" (movement). The full code list is in PayPal's docs, not the spec — confirm against the docs page and against her live data before the first live pull.
- **Whether her PayPal account has API access enabled** and whether a Developer app already exists on it. Unknown until she checks developer.paypal.com.
- **The local folder (Track A).** Where it is and what it contains — the answer to that decides whether the build starts from her code or from this design.
- **Schedule.** How often the pull runs (daily, like the scan, or on her word each time) — her call, not assumed.

---

## Policy direction folded in — Nov 25, 2026 (her word: "Fold it into the PayPal ledger design")

Folded in from the policy pour (`paypal-direct-deposit-structures-policy-pour-STAGED-nov25.md`). The pour file stays the verbatim record; this section is the working version inside the design. All of it is proposed direction, not built, not filed into the paper, chapter, or corpus.

### Direct-deposit-like structures

PayPal issues direct-deposit-like structures for people whose income is restricted to games, surveys, consulting, remote work, and gigs — students, part-timers, and others, for whatever circumstances limit them. The ledger is what makes the structure real: every payout keyed by transaction ID, counted once, with a reference number — so the earner can show, with a record, that they made something and are trying. The effort made manifest, provable to the tax authorities.

### Self-employment tax slips (her addition, same pour)

A slip for self-employment purposes, printed from PayPal and presented to accountants — easier to use when tax season comes. What the slip carries, generated from the stored ledger entries (never from a gross pull):

- The earner, the tax year, and the per-source totals — each app/source on its own line (PlayOut, JustPlay, BestPlay, and any new source).
- Every entry counted once, by transaction ID, with its reference number on the slip.
- Gross received, fees, and net received shown separately — the net is the income figure.
- Transfers (PayPal to bank) excluded from income on the slip — they are movement, already counted when the money arrived, and the slip says so plainly.
- The running total and the GST/HST $30,000 threshold position, so the earner and the accountant can see where the year stands.

This is the remedy to the memorandum's micro-earner trap (Parts I–VIII): no slip, so the earner thinks there is no tax. The slip closes that gap — the earner files from the slip, on Form T2125 with all platform income reported, instead of from memory or from nothing.

### The equitable, proportional split [PROPOSED policy direction, not settled]

The games and surveys are often run by marketers or others who make money from people using their setups — the value flows up, and the person doing the work gets breadcrumbs. The proposed direction: the tax authorities act as intermediaries to ensure the split is distributed equitably and proportionally, so the earner can even get something back, and unjust enrichment — them taking all the money, or giving breadcrumbs — is prevented. This is a policy proposal, not law, not a built mechanism, and not filed anywhere. The mechanism ("get something back" — a refund, a credit, a share of the marketer's revenue, or something else) is open on her word.

### What changes in this design

- The pull's output gains an **annual slip view**: a printable, per-year summary built from the stored entries, in the shape above. Proposed, not built.
- The dedup and counted-once rules are what make the slip trustworthy — the slip is only as good as the ledger behind it, which is why the pull filters status S only and keys everything by transaction ID.
- The direct-deposit-like structure and the intermediary-split proposal are carried as policy direction in this design only. If they ever become a paper or memorandum filing, that is a separate word from her.

---

## Filing note — staged only, Nov 25, 2026

This file is a draft design, published here Nov 25, 2026. It is not filed, submitted, or published anywhere else. It does not change the tracker, the tax memorandum, the law reform paper, Chapter 7, or any JurisLedger module. It does not connect to PayPal, create any app, or store any credential. The build in Track B and the read in Track A each need her word separately.
