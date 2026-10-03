# Splyt

Splyt is an ad-revenue-sharing site. Visitors check in once a day, invite friends, and receive a proportional share of **50% of the site's ad revenue** each payout cycle, paid in USDT or USDC to a wallet address they provide.

There are no accounts, emails, or passwords. A visitor opens the site, enters a payout wallet, and checks in.

Live site: https://splytproject.github.io/Splyt/

> **Status: early prototype.** Check-ins, referrals, rate limiting and payout accounting work end to end. Payouts are executed manually by the operator, and some anti-abuse measures are intentionally deferred (see [Known limitations](#known-limitations)).

---

## Why daily check-ins (and not time on page)

The site is monetised with [AADS](https://aads.com), which pays publishers for their share of **globally unique impressions**: one impression per IP address per 24 hours across the whole AADS network, and only for the first ad unit that loads on a page. A visitor who keeps the page open for six hours is worth the same as one who stays for ten seconds.

Splyt's rewards therefore follow what actually generates revenue:

- **Daily check-ins** reward a returning visitor, which is a new unique impression every day.
- **Referrals** reward bringing in new visitors, who are new unique impressions.
- There is **no reward for time on page**.

The page uses two ad units (AADS allows at most three per page).

---

## How it works

```
Visitor opens page (optionally via an invite link ?ref=<code>)
      │
      ▼
Anonymous Supabase session is created silently (no email / password)
      │
      ▼
Visitor saves a payout wallet (network + address)
      │
      ▼
Check-in button unlocks after ~8 s (lets the ad units load)
      │
      ▼
Page calls RPC daily_checkin(p_ref)
      │   server checks: session, wallet present, not yet checked in today
      │   (per account, per IP and per wallet), streak, referral eligibility
      ▼
Points recorded in point_events, profile counters updated
      │
      ▼
End of cycle: operator runs close_cycle(<ad revenue USD>)
      │   50% of revenue → user pool, split by points, grouped by wallet
      ▼
Payout rows created → operator sends USDT/USDC → marks rows paid
```

### Earning rules

| Action | Points | Limits |
|---|---|---|
| Daily check-in | 10 on day 1, +1 per consecutive day, max 20 (day 11+) | Once per UTC day per **account**, per **IP**, and per **wallet** |
| Friend's first check-in via your link | +25 to the referrer | Friend's IP must be new to Splyt and different from the referrer's last IP; wallets must differ |
| Each later check-in by that friend | +2 to the referrer | Follows the friend's own daily limits |

- Streaks continue if the previous check-in was the previous UTC day; otherwise they reset to 1.
- A referral is attached only on the visitor's **first ever** check-in. The `ref` code is kept in `localStorage` until then, and the URL parameter is stripped on load.
- Referrers must have a wallet saved for referrals to count.
- All limits are enforced in the database. The client only decides when to show the button.

### Payout formula

For a cycle with ad revenue `R` (USD):

```
user_pool   = R × 0.5
total       = Σ points_current_cycle   (only profiles with a wallet)
payout(w)   = trunc( user_pool × points(w) / total , 6 )
```

`points(w)` is the sum of cycle points across **all sessions that saved wallet `w`** on the same network. A visitor who clears their browser or switches devices and re-enters the same wallet has their points combined at payout time.

---

## Architecture

| Layer | Technology |
|---|---|
| Frontend | Static `index.html` + `privacy.html` on GitHub Pages |
| Styling | Tailwind CSS (Play CDN) |
| Client SDK | `@supabase/supabase-js` v2 (jsDelivr CDN, UMD build) |
| API proxy | Cloudflare Worker (`worker.js`) on `workers.dev` |
| Auth | Supabase Auth, anonymous sign-ins |
| Database | Supabase Postgres with Row Level Security |
| Business logic | PL/pgSQL functions exposed as RPCs |
| Scheduled jobs | `pg_cron` (alert scan, IP purge) |
| Ads | AADS iframe units (728×90 leaderboard, 300×250 sidebar) |

There is no custom backend server. All trust-sensitive logic runs inside Postgres, and the browser only holds a Supabase **publishable** key, which is safe to expose because every table is protected by RLS.

### Cloudflare Worker proxy

Several Indian ISPs (Jio, Airtel, ACT) block `*.supabase.co`. The page therefore talks to a Cloudflare Worker on `workers.dev`, which forwards `/auth/v1/*` and `/rest/v1/*` to the Supabase project.

Behind the Worker, Supabase sees Cloudflare's egress IP rather than the visitor's. To preserve per-IP limits, the Worker adds two headers:

- `x-splyt-client-ip`: the visitor's `cf-connecting-ip`
- `x-splyt-proxy-secret`: a shared secret (Worker secret `PROXY_SECRET`, mirrored in Supabase Vault as `proxy_secret`)

`request_client_ip()` trusts `x-splyt-client-ip` **only** when the secret matches. Otherwise it falls back to `cf-connecting-ip` → `x-forwarded-for` → `x-real-ip`, so a client calling Supabase directly cannot spoof its IP.

Set `API_PROXY_URL` to an empty string in `index.html` to connect to Supabase directly.

---

## Data model

### `profiles`
One row per session (anonymous auth user), created by the `on_auth_user_created` trigger.

| Column | Type | Notes |
|---|---|---|
| `id` | `uuid` PK | = `auth.users.id`, cascades on delete |
| `wallet_network` | `text` | `tron`, `bsc`, `polygon`, `ethereum`, `solana` |
| `wallet_address` | `text` | Format validated per network; EVM addresses stored lowercase |
| `wallet_updated_at` | `timestamptz` | Set by trigger whenever the wallet changes |
| `points_current_cycle` | `int` | Reset to 0 when a cycle closes |
| `points_all_time` | `int` | Never reset |
| `streak_count` | `int` | Consecutive daily check-ins |
| `last_checkin_date` | `date` | UTC date of the last check-in |
| `referral_code` | `text` unique | 8 hex chars, generated on insert |
| `referred_by` | `uuid` | Referrer's profile; set once, on first check-in |
| `last_ip` | `text` | IP of the most recent check-in |
| `updated_at` | `timestamptz` | Maintained by trigger |
| `faucetpay_email` | `text` | Legacy, unused |

Clients may update only `wallet_network` and `wallet_address` (plus the legacy `faucetpay_email`), enforced with column-level grants.

### `point_events`
Ledger of every point award.

| Column | Notes |
|---|---|
| `user_id` | Profile that received the points |
| `kind` | `checkin`, `referral_join`, `referral_daily` |
| `points` | Points awarded |
| `ip` | Requesting IP (check-ins and the friend's IP for referral events) |
| `wallet_network`, `wallet_address` | Snapshot at check-in time (used for the per-wallet daily limit) |
| `related_user` | For referral events: the friend who checked in |
| `created_at` | Timestamp |

Partial indexes on `(ip, created_at)` and `(wallet_network, wallet_address, created_at)` for `kind = 'checkin'` back the daily-limit lookups.

### `impression_logs`
Legacy audit table from the earlier time-based model. It is no longer written to and is kept for history.

### `payout_cycles`
| Column | Notes |
|---|---|
| `id`, `started_at`, `ended_at` | |
| `status` | `open` / `closed`; a partial unique index allows only one open cycle |
| `ad_revenue_usd`, `user_pool_usd`, `total_points` | Filled when the cycle closes |

### `payouts`
One row per wallet per closed cycle.

| Column | Notes |
|---|---|
| `cycle_id` | → `payout_cycles.id` |
| `wallet_network`, `wallet_address` | Snapshot at close time; unique per cycle |
| `points`, `accounts` | Summed points and number of sessions merged into this wallet |
| `amount_usd` | Computed share, 6 decimal places |
| `status` | `pending` / `paid` / `failed` |
| `tx_hash`, `paid_at` | Filled in by the operator after sending funds |

### `security_alerts`
Written by the scheduled scanner; see [Monitoring](#monitoring). Owner-only (RLS enabled, no policies, no API grants).

### Supported payout networks

| Network | Address format | Tokens |
|---|---|---|
| Tron (TRC-20) | `T` + 33 base58 chars | USDT only |
| BNB Smart Chain (BEP-20) | `0x` + 40 hex | USDT, USDC |
| Polygon | `0x` + 40 hex | USDT, USDC |
| Ethereum (ERC-20) | `0x` + 40 hex | USDT, USDC (high fees) |
| Solana | 32–44 base58 chars | USDT, USDC |

Validation is format-only (regex, enforced both client-side and by a `CHECK` constraint). EVM checksums and on-chain existence are not verified.

---

## Database functions

| Function | Callable by | Purpose |
|---|---|---|
| `daily_checkin(p_ref text)` | `authenticated` | Awards the daily check-in, updates the streak, applies referral credit. Returns `points_awarded`, `streak`, `points_current_cycle`, `points_all_time`, `referral_applied`. Raises `Wallet required`, `Already checked in today`, `Network already checked in today`, or `Wallet already checked in today`. |
| `my_referral_stats()` | `authenticated` | The caller's `friends_joined` and `referral_points` (aggregates only). |
| `close_cycle(p_ad_revenue_usd numeric)` | Operator only | Closes the open cycle, writes payout rows, resets cycle points, opens the next cycle. |
| `scan_suspicious_activity()` | Operator / cron | Raises alerts; optionally posts them to Discord. |
| `purge_old_ips()` | Operator / cron | Deletes IPs older than 90 days. |
| `request_client_ip()` | Internal | Resolves the caller's IP (see [proxy](#cloudflare-worker-proxy)). |
| `handle_new_user()`, `stamp_wallet_change()`, `set_updated_at()` | Triggers only | Profile creation, wallet normalisation and timestamps. |

`daily_checkin` is `SECURITY DEFINER` with an empty `search_path`. The caller is identified by `auth.uid()` from the signed JWT. Concurrency is handled with a `FOR UPDATE` lock on the caller's profile plus transaction-scoped advisory locks keyed on the IP and on the wallet, so parallel requests from several tabs, sessions or devices cannot double-claim a day.

---

## Security model

- **RLS on every table.**
  - `profiles`: a session can `SELECT` and `UPDATE` only its own row.
  - `point_events`, `impression_logs`: `SELECT` own rows only; no direct writes.
  - `payout_cycles`: readable by everyone, for transparency.
  - `payouts`: readable only by sessions whose saved wallet matches the payout row.
  - `security_alerts`: no client access.
- **Column-level grants.** Point counters, streaks, referral fields and codes cannot be written from the client.
- **Single write path for points.** The only way to gain points is `daily_checkin()`.
- **Internal functions locked down.** Trigger functions, `close_cycle`, the scanner and the purge job have `EXECUTE` revoked from `anon`, `authenticated` and `public`.
- **Keys.** The frontend ships only the publishable key. The Supabase secret key and the proxy secret are never committed.

---

## Ads

Both ad slots are AADS iframes wrapped in a slot container:

- Iframes are sandboxed (`allow-scripts allow-same-origin allow-popups allow-popups-to-escape-sandbox`), so ads can open new tabs but cannot navigate the page.
- Fixed-size creatives are scaled down with a `ResizeObserver` on narrow screens.
- After 4 s, a slot manager checks whether the creative rendered and whether a bait element was hidden by a content blocker. If the ad is blocked, a fallback asks the visitor to disable AdBlock.
- AADS only counts revenue after its verification system detects the unit code on the page; check unit status in the AADS dashboard after any change.

---

## Running your own instance

1. **Create a Supabase project.**
2. **Apply the schema.** The schema lives in the Supabase project's migration history; export it with `supabase db dump --schema public`.
3. **Auth settings:** Authentication → Sign In / Providers → enable *Allow new users to sign up* **and** *Allow anonymous sign-ins*. Leave CAPTCHA off (the frontend does not send a token yet).
4. **Proxy (optional; required for users on networks that block Supabase):** deploy `worker.js` as a Cloudflare Worker, add the `PROXY_SECRET` secret, and store the same value in Supabase Vault as `proxy_secret`.
5. **Configure the frontend:** set `SUPABASE_PROJECT_URL`, `SUPABASE_ANON_KEY` (publishable key) and `API_PROXY_URL` in `index.html`.
6. **Ads:** set your AADS unit IDs in the two iframes.
7. **Host it** on any static host over HTTP(S).

---

## Operating a payout cycle

Run these from the Supabase SQL Editor as the project owner.

```sql
-- 1. Close the current cycle with that cycle's total ad revenue in USD
select * from public.close_cycle(25.00);

-- 2. Review what is owed
select wallet_network, wallet_address, points, accounts, amount_usd
from public.payouts
where cycle_id = <closed_cycle_id> and status = 'pending'
order by amount_usd desc;

-- 3. After sending funds, record the transaction
update public.payouts
set status = 'paid', tx_hash = '<tx hash>', paid_at = now()
where id = <payout_id>;
```

Before paying, review open rows in `security_alerts` and recent `profiles.wallet_updated_at` changes.

---

## Monitoring

`pg_cron` runs `scan_suspicious_activity()` every 15 minutes and writes to `security_alerts`, one alert per subject per UTC day:

| Alert | Trigger | Severity |
|---|---|---|
| `ip_many_accounts` | 3+ accounts checking in from one IP within 7 days | medium (high at 10+) |
| `wallet_many_ips` | one wallet checking in from 5+ IPs within 7 days | medium (high at 10+) |
| `referral_burst` | 5+ new referrals for one referrer in 24 h | medium (high at 20+) |
| `wallet_changed_after_earning` | wallet changed in the last 24 h after 50+ points were earned | medium |

Optional Discord notifications: store a webhook URL in Supabase Vault as `alerts_webhook_url`; new alerts are posted through `pg_net`.

A second job, `purge-old-ips`, runs daily at 03:17 UTC and removes IPs older than 90 days from `point_events`, `impression_logs` and `profiles`, along with alerts older than 90 days.

---

## Known limitations

- **No proof of a human.** A script with a fresh wallet and IP can check in daily. Limits make each extra identity cost a separate IP and wallet, but do not prevent it.
- **Sessions are per browser.** Clearing storage creates a new session. Points are recovered at payout time only if the same wallet is re-entered; streaks restart on the new session.
- **Shared IPs share a check-in.** Households, offices and carrier-grade NAT mobile networks get one check-in per day between them, and only the first visitor on a network can be credited as a referral.
- **Abuse is bounded, not prevented.** Payouts never exceed 50% of revenue, so abuse dilutes honest users' share rather than increasing the operator's costs.
- **Manual payouts.** No automated on-chain transfers and no minimum payout threshold yet.
- **Personal data.** IP addresses are stored in `point_events` and `profiles.last_ip`, disclosed in [`privacy.html`](privacy.html), and purged after 90 days.
- **Tailwind Play CDN.** It logs a production warning and compiles CSS in the browser. Switch to a Tailwind CLI build for production.

## Roadmap

- CAPTCHA (Cloudflare Turnstile) on session creation
- Optional wallet sign-in (Supabase Web3 auth: SIWE / Solana) to prove wallet ownership
- Public transparency page listing closed cycles and payout totals
- Scheduled cycle closing and a payout export
- Minimum payout threshold
- Tailwind CLI build, and schema migrations committed to the repo

## License

No license has been chosen yet. Until one is added, all rights are reserved by the authors.
