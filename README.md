# Token Radar — Build the best token launch radar

A program that reads a sealed DefiLlama protocol snapshot and ranks the protocols whose
DefiLlama record records no token, by an implied value the model states, with a farm plan for
every ranked protocol and a static dashboard a reader can use offline.

## What it ranks

- **Pre-token board (226 rows):** every protocol in the snapshot that records no token
  (no market cap, no CoinGecko id, no symbol) and is rankable — not a CEX and not a versioned
  pool. That covers all 40 watchlist protocols (TVL ≥ $50,000,000, no token on record, not a
  wrapped/staked/bridged product of a token that trades) plus all 186 rankable pre-token
  protocols below the floor. The watchlist is the floor, never the ceiling.
- **Live-token view (24 rows):** the largest protocols whose token already trades, shown with
  their market caps. They are set aside from the ranking: a market cap is a fact, not a model.
- The board says a protocol **records no token**, never that it **has** no token: the snapshot
  is DefiLlama's record, not proof, and a protocol on the list may still have a token its
  record does not carry. The board never predicts whether or when a token is issued.

## The value model (radar-2)

One function scores every row — no protocol name is special-cased, no row is hand-picked:

```
growth      = clamp(change_7d, -50, 150) per cent
growthMult  = 1 + (growth / 100) x 0.6
maturity    = 0.65 + min(age_days, 1460) / 1460 x 0.35   (age unknown: 365 days)
chainMult   = 1 + min(max(chains - 1, 0), 5) x 0.02     (capped at 1.10)
categoryMult = 1.00 for Lending, Dexes, Derivatives, Perpetuals, Yield, CDP;
               0.95 for every other category or none
implied     = max(1, round(tvl_usd x 0.30 x growthMult x maturity x chainMult x categoryMult))
```

- **TVL** is the protocol's size; **seven-day growth** is its momentum.
- **Age** is the time since DefiLlama first listed the protocol — the best record of how long
  it has run with no token on record. A protocol that held large TVL for years without issuing
  a token is worth more than a week-old pool of the same size.
- **Chain count** is a small reach bonus (at most +10%).
- **Category** is deliberately small (5%) so no category dominates the ranking.

The dashboard recomputes every row with the same function (`impliedOf` in
`dashboard/app.js`) and shows whether its number matches `board.json`.

## Farm plans

Each row carries six steps, derived from the protocol's own record (category, chains, TVL,
age) — never from a hand-written list of names. Every step states what it costs and how long
it takes: the capital depends on TVL ($50–$250), the gas figure is a rough per-chain estimate
for one transaction at ordinary congestion (Ethereum about $4, Solana about $0.02, Base about
$0.05, and so on — estimates, not quotes), and the hold period is 30–60 days depending on age.
The plan never claims the protocol has a token, an airdrop, a date, or a points program: it
only says to check the protocol's own rewards page if it publishes one, and to claim only
from a link the protocol itself publishes.

## How to run

```sh
mkdir -p out && MAKEFUN_OUTPUT_DIR=out node main.ts --input ../input/snapshot.json
```

Node.js 22.18 or later, standard library only, no install step, no network. The program reads
the snapshot, writes `out/board.json`, `out/data.js`, copies `dashboard/` and `screenshots/`
into `out/`, and writes `out/dossier.json` with `buildDossier` from `vendor/makefun-board.ts`.
Run it twice and the outputs are byte-identical.

Open `out/dashboard/index.html` in a browser (no server needed, no network): the board is
sortable and filterable by search, category, chain, and watchlist-only, with a detail view
for every protocol showing its model breakdown and farm plan, plus the live-token view.

## Layout

```
main.ts                  entry point (Node 22, standard library only)
lib/model.ts             the radar-2 model, farm plans, snapshot parsing, board checks
vendor/makefun-board.ts  the make.fun dossier library (vendored from the kit)
dashboard/               index.html, app.css, app.js — static, no network
screenshots/              1-board.png, 2-detail.png, 3-filtered.png
LICENSE                  MIT
```

## Screenshots

- `1-board.png` — the board, ranked by implied value.
- `2-detail.png` — a protocol's detail view (rank 1, opened from the board).
- `3-filtered.png` — the board filtered to one category (Lending).

Each is 1440×900 and 300–350 KB, taken with headless Chromium from the built dashboard in
`out/dashboard/`.
