# HANDOFF: LuckyPot

No-loss weekly USDC prize pool on Arc. Deposits are pooled and each week a random draw gives that week's whole yield to a few people. Nobody loses principal: 100% of deposited USDC can be withdrawn at any time.

History: [docs/archive/handoff-through-2026-09-11.md](docs/archive/handoff-through-2026-09-11.md) (Vietnamese; sections referenced below are there).

## Key facts

- **Repo:** https://github.com/KattyFury/LuckyPot (private; renamed from `LuckyStaker`).
- **Web:** https://luckypot.cc (Cloudflare Pages project `luckypot`). Landing at root, dashboard at `/app`. **No** auto-deploy from GitHub.
- 🔴 **Deploy from the REPO ROOT, not `frontend/`:** `npx wrangler pages deploy frontend/dist-site --project-name=luckypot --branch=main --commit-dirty=true`, or simplest `npm run deploy` from `frontend/`. Running `wrangler pages deploy dist-site` inside `frontend/` does **not** upload Functions (`functions/` is at the repo root), so all `/api/*` silently falls back to static while wrangler still reports success. It has happened twice. **Always confirm the output has "✨ Compiled Worker successfully" / "Uploading Functions bundle".**
- ⚠️ **The Solidity contract is deliberately still named `LuckyStakerPool`.** That is the real technical name, not a missed rename.
- ⚠️ **Current proxy: `0xBdE568986a009eBaAE31Cb78033470c334Fad698`** (deploy block `59715964`, redeployed 2026-08-31). Old proxy `0x88dCB2f36356AA8DADdC2e8fb4A3E122Ba9D0Beb` is abandoned.
- **Spec:** [`arc-prize-pool-spec.md`](./arc-prize-pool-spec.md).
- **Database:** Cloudflare D1 `luckypot-history` (read via `HISTORY_DB` in the root `wrangler.toml`, written by `automation/src/indexHistory.ts` on the `keeper.yml` cron). `Won` is written when `Drawn` fires (gross prize), `Claimed` when `Claimed`/`Swept` fires (net of 5% referral cut): two separate rows.
- **Language:** EN|VI toggle in the navbar, covering landing + app **except** `AdminPage.tsx` and the `formatCountdown` value.

## Where we stopped (2026-09-11)

Three commits pushed to `main` (`7d10a67`, `c8c4a9f`, `88d582b`):
1. Slogan "Your idle money can become lottery tickets – for free" added to `README.md`, the landing kicker above the H1 (EN + VI via `heroKicker`), and the GitHub description. The hero H1 stays untouched.
2. Scratch card showing "You win 0 USDC" after a sweep fixed: `ResultModal.tsx` now computes the win from `epoch.winners` + `prizeForRank()` instead of `owedTo()`, which returns 0 once claimed.
3. Balance updates: `refetchOnWindowFocus: true` re-enabled only for `usePoolTotals` / `useUserPosition` (one multicall per refocus). Passive changes still rely on the 15 s `BALANCE_POLL_MS` poll.

Verified with `tsc`, build, and Playwright against the real proxy (refocus fires 3 new RPC requests; no console errors or 4xx/5xx).

## Pending

- **Admin rights** are still 2-of-2 Safe + 1 EOA with equal power. The owner planned to move to multisig-only around 2026-09-10/11; no confirmation seen.
- **Decision not made, do not build without asking:** the 3-day `SWEEP_DELAY` feels too short. Options discussed:
  - **A. Longer claim/sweep window:** change `SWEEP_DELAY` (contract line ~36) to near a week, or gate sweep on the next epoch being drawn. Low risk: redeploy implementation + upgrade proxy through Safe. **Recommended first.**
  - **B. Auto-push prizes in `revealAndDraw()`:** without `try/catch` and a `pendingRef`-style fallback, one failing winner transfer would revert the whole draw. Bigger change, needs more testing.
  - Independent of `forceEndEpoch()`.
