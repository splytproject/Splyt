# Splyt

Splyt is an ad-revenue-sharing site. Visitors keep the page open while ads are displayed, earn points for active time, and receive a proportional share of **50% of the site's ad revenue** each payout cycle, paid in USDT or USDC to a wallet address they provide.

There are no accounts, emails, or passwords. A visitor opens the site, enters a payout wallet, and starts earning.

Live site: https://splytproject.github.io/Splyt/

> **Status: early prototype.** The earning loop, rate limiting and payout accounting work end to end. Payouts are executed manually by the operator, and several anti-abuse measures are intentionally deferred (see [Known limitations](#known-limitations)).

---

## How it works

```
Visitor opens page
      │
      ▼
Anonymous Supabase session is created silently (no email / password)
      │
      ▼
Visitor enters a payout wallet (network + address)
      │
      ▼
Progress bar fills over 30 s of *visible* page time
      │
      ▼
Page calls RPC increment_user_impression()
      │   server checks: session, wallet present, 30 s gap (account + IP),
      │   daily cap (account + IP)
      ▼
+1 point: row written to impression_logs, profile counters incremented
      │
      ▼
End of cycle: operator runs close_cycle(<ad revenue USD>)
      │   50% of revenue → user pool, split by points, grouped by wallet
      ▼
Payout rows created → operator sends USDT/USDC → marks rows paid
```

### Earning rules

| Rule | Value | Enforced by |
|---|---|---|
| Time per point | 30 seconds of visible page time | Client timer (UX) + server gap check (authoritative) |
| Minimum gap between claims | 30 s per account **and** 30 s per IP | `increment_user_impression()` |
| Daily cap | 720 points per UTC day per account **and** per IP (6 h of earning) | `increment_user_impression()` |
| Wallet required | Claims are rejected until a wallet is saved | `increment_user_impression()` |

The client timer pauses when the tab is hidden (`visibilitychange`) and caps each animation-frame delta at 250 ms so a throttled or sleeping tab cannot accumulate time in bulk. The client timer is a convenience only; the database decides whether a point is awarded.

### Payout formula

For a cycle with ad revenue `R` (USD):

```
user_pool   = R × 0.5
total       = Σ points_current_cycle   (only profiles with a wallet)
payout(w)   = trunc( user_pool × points(w) / total , 6 )
```

`points(w)` is the sum of cycle points across **all sessions that saved wallet `w`** on the same network. A visitor who clears their browser or switches devices and re-enters the same wallet has their points combined at payout time.

Points earned by sessions that never saved a wallet are not counted.

---

## Architecture

| Layer | Technology |
|---|---|
| Frontend | Single static `index.html`, hosted on GitHub Pages |
| Styling | Tailwind CSS (Play CDN) |
| Client SDK | `@supabase/supabase-js` v2 (jsDelivr CDN, UMD build) |
| Auth | Supabase Auth, anonymous sign-ins |
| Database | Supabase Postgres with Row Level Security |
| Business logic | PL/pgSQL functions exposed as RPCs |
| Ads | A-ADS iframe units (728×90 leaderboard, 300×250 sidebar) |

There is no custom backend server. All trust-sensitive logic runs inside Postgres, and the browser only holds a Supabase **publishable** key, which is safe to expose because every table is protected by RLS.

---

## Data model

### `profiles`
One row per session (anonymous auth user). Created automatically by the `on_auth_user_created` trigger.

| Column | Type | Notes |
|---|---|---|
| `id` | `uuid` PK | = `auth.users.id`, cascades on delete |
| `wallet_network` | `text` | `tron`, `bsc`, `polygon`, `ethereum`, `solana` |
| `wallet_address` | `text` | Format validated per network; EVM addresses stored lowercase |
| `wallet_updated_at` | `timestamptz` | Set by trigger whenever the wallet changes |
| `points_current_cycle` | `int` | Reset to 0 when a cycle closes |
| `points_all_time` | `int` | Never reset |
| `last_ip` | `text` | IP of the most recent successful claim |
| `updated_at` | `timestamptz` | Maintained by trigger |
| `faucetpay_email` | `text` | Legacy, unused by the current frontend |

### `impression_logs`
Audit trail: one row per awarded point.

| Column | Type |
|---|---|
| `id` | `bigint` identity PK |
| `user_id` | `uuid` → `profiles.id` |
| `claimed_at` | `timestamptz` default `now()` |
| `ip` | `text` |

Indexed on `(user_id, claimed_at desc)` and `(ip, claimed_at desc)` for the rate-limit lookups.

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
| `increment_user_impression()` | `authenticated` | Awards one point. Returns `points_current_cycle`, `points_all_time`, `points_today`, `daily_cap`. Raises `Wallet required`, `Rate limit exceeded`, or `Daily limit reached`. |
| `my_points_today()` | `authenticated` | Caller's points since 00:00 UTC (used to render the daily counter on load). |
| `close_cycle(p_ad_revenue_usd numeric)` | Operator only (no API grant) | Closes the open cycle, writes payout rows, resets cycle points, opens the next cycle. |
| `handle_new_user()` | Trigger only | Inserts a `profiles` row for each new auth user. |
| `stamp_wallet_change()` | Trigger only | Lowercases EVM addresses and sets `wallet_updated_at`. |
| `set_updated_at()` | Trigger only | Maintains `profiles.updated_at`. |

`increment_user_impression()` is `SECURITY DEFINER` with an empty `search_path`. It takes no parameters: the caller is identified by `auth.uid()` from the signed JWT, and the IP is read from the request headers set by Supabase's proxy (`cf-connecting-ip`, then `x-forwarded-for`, then `x-real-ip`). Concurrency is handled with a `FOR UPDATE` lock on the caller's profile row plus a transaction-scoped advisory lock keyed on the IP, so parallel requests from several tabs or sessions on one connection are serialised and only one can succeed per 30 s window.

---

## Security model

- **RLS on every table.**
  - `profiles`: a session can `SELECT` and `UPDATE` only its own row.
  - `impression_logs`: `SELECT` own rows only; no direct inserts.
  - `payout_cycles`: readable by everyone, for transparency.
  - `payouts`: readable only by sessions whose saved wallet matches the payout row.
- **Column-level grants.** Table-wide `UPDATE` is revoked from `authenticated`; only `wallet_network`, `wallet_address` (and the legacy `faucetpay_email`) are granted. Point counters cannot be written from the client.
- **Single write path for points.** The only way to gain points is `increment_user_impression()`.
- **Internal functions locked down.** Trigger functions and `close_cycle` have `EXECUTE` revoked from `anon`, `authenticated` and `public`.
- **Keys.** The frontend ships only the publishable key. The secret key must never be committed.

---

## Ads

Both ad slots are A-ADS iframes wrapped in a slot container:

- Iframes are sandboxed (`allow-scripts allow-same-origin allow-popups allow-popups-to-escape-sandbox`), so ads can open new tabs but cannot navigate the page.
- Fixed-size creatives are scaled down with a `ResizeObserver` on narrow screens.
- After 4 s, a slot manager checks whether the creative rendered and whether a bait element was hidden by a content blocker. If the ad is blocked, a fallback asks the visitor to disable AdBlock.
- Slots with a placeholder unit ID (`YOUR_UNIT_ID`) show a neutral "Ad space" label instead.

To configure, replace `YOUR_UNIT_ID` in both iframes with your A-ADS unit numbers. Script-based networks can be pasted into the same containers; add `onerror="__slotFail(this)"` to the `<script>` tag so load failures trigger the fallback.

---

## Running your own instance

1. **Create a Supabase project.**
2. **Apply the schema.** Tables, policies, grants, triggers and functions as described above. The schema currently lives in the Supabase project's migration history; export it with `supabase db dump --schema public` if you are setting up a copy.
3. **Enable anonymous sign-ins:** Authentication → Sign In / Providers → *Allow anonymous sign-ins*.
4. **Leave CAPTCHA off for now.** The frontend does not yet send a CAPTCHA token, so enabling it blocks all sessions.
5. **Configure the frontend.** In `index.html`, set `SUPABASE_URL` and `SUPABASE_ANON_KEY` (publishable key, `sb_publishable_…`).
6. **Keep constants in sync.** `INTERVAL_MS` and `DAILY_CAP` in the page must match `min_gap` and `cap` in `increment_user_impression()`.
7. **Host it.** Any static host works. Serve over HTTP(S); don't open the file via `file://`.

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

Before paying, reviewing `impression_logs` (claims per IP, round-the-clock activity) and `profiles.wallet_updated_at` (last-minute wallet changes) is recommended.

---

## Known limitations

- **No proof of human attention.** The server enforces timing, not presence. A script holding a session open can claim at the maximum rate.
- **Sessions are per browser.** Clearing storage creates a new session. Points are recovered at payout time only if the same wallet is re-entered.
- **Shared IPs share a quota.** Households, offices and carrier-grade NAT mobile networks get one 30 s slot and one daily cap between them.
- **Multiple wallets are not linked.** One person with several IPs and wallets can run several earning streams.
- **Abuse is bounded, not prevented.** Payouts never exceed 50% of revenue, so abuse dilutes honest users' share rather than increasing the operator's costs.
- **Manual payouts.** No automated on-chain transfers; no minimum payout threshold yet.
- **Personal data.** IP addresses are stored in `impression_logs` and `profiles.last_ip`. Operators must disclose this in a privacy notice appropriate to their jurisdiction.
- **Tailwind Play CDN.** It logs a production warning and compiles CSS in the browser. Switch to a Tailwind CLI build for production.

## Roadmap

- CAPTCHA (Cloudflare Turnstile) on session creation
- Optional wallet sign-in (Supabase Web3 auth: SIWE / Solana) to prove wallet ownership
- Public transparency page listing closed cycles and payout totals
- Scheduled cycle closing and a payout export
- Minimum payout threshold and configurable limits
- Tailwind CLI build, and schema migrations committed to the repo

## License

No license has been chosen yet. Until one is added, all rights are reserved by the authors.
