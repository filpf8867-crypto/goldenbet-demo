# Goldenbet — Front-End Template

A responsive, dark/gold-themed sportsbook UI (desktop + mobile) built with plain HTML/CSS/JS — no build step, no framework. Open `index.html` in a browser or upload the folder to any static host.

## What's actually in this build

This is a **front-end only** template with realistic mock data and simulated behavior, so it *feels* like a working sportsbook (live odds ticking, bet slip math, bets settling, a wallet balance) without touching real money or real systems. Nothing here processes an actual payment or wager.

**Included UI/UX:**
- Responsive layout: 3-column desktop, single-column + bottom nav on mobile
- Multi-sport tabs (football, basketball, tennis, esports, baseball, hockey) with sport-appropriate markets (1X2 for football, moneyline/spread/total for the rest)
- Live ticker, in-play badge + live score simulation
- Bet slip with Singles and Parlay modes, stake inputs, live payout calculation
- Simulated odds movement with a coin-flip micro-animation (the brand's signature touch) and up/down indicators
- Wallet with deposit/withdraw (mock), transaction history
- My Bets (open + settled, auto-"settles" after a few seconds for demo purposes)
- Account page with **responsible-gambling controls**: deposit limit slider, cool-off buttons, self-exclusion (actually disables betting for the session)
- Age/jurisdiction gate on entry
- Search, favourites, filter chips, promo carousel, notifications

## What you must add before this can take real bets

Since you hold the license, here's what a developer needs to wire in — each spot is marked with a comment in `app.js`:

1. **Odds/data feed** — replace the `MATCHES` mock generator with calls to your real-time odds provider (e.g. via WebSocket for live updates instead of `simulateOddsMovement`).
2. **Authentication** — replace the login modal with your real auth/session provider.
3. **KYC & age verification** — the age gate here is just a click-through. Swap in your licensed identity-verification provider, and make sure verification is enforced server-side, not just in the browser.
4. **Payments** — `depositBtn`/`withdrawBtn` currently just add/remove a number from `localStorage`. Replace with your licensed payment processor's hosted fields/SDK.
5. **Bet placement & settlement** — `placeBet()` and `settleRandomBet()` simulate this client-side. Real bet placement, risk/limits checks, and settlement must happen server-side against your trading/risk engine.
6. **Geo-blocking & licensing footer** — add your actual license number(s), regulator, and jurisdiction restrictions; block access from locations where you're not licensed.
7. **Responsible-gambling tooling** — the limit/cool-off/self-exclusion controls here are UI-only placeholders. They need to actually write to your backend and be enforced immediately and server-side, and self-exclusion in particular usually has legal requirements about duration and reversal in most jurisdictions.

## File structure

```
goldenbet/
├── index.html   — markup, modals, view containers
├── styles.css   — design tokens, layout, responsive rules, animations
├── app.js       — mock data, state, rendering, all interactivity
└── README.md    — this file
```

## Notes

- All "money" is stored in the browser's `localStorage` for demo purposes only — clearing browser data resets the balance to the starting $1,000.
- Use the **Account → Log out** button to reset all demo data and start fresh.
