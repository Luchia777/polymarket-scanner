# Polymarket Rewards Scanner

A web application that scans active markets on Polymarket and surfaces the ones where a liquidity provider with a small stake (20, 40, or 50 shares) can realistically earn maker rewards. The app filters thousands of markets down to a short list by checking the reward program parameters, the spread, and — most importantly — the depth and shape of the order book.

Live deployment: `polymarket-scanner-peach.vercel.app`
Source: `github.com/Luchia777/polymarket-scanner`

---

## What problem this solves

Polymarket pays daily rewards in USDC to users who post limit orders close to the midpoint and meet certain spread and size requirements. The catch: most reward-eligible markets are crowded. Existing orders on the best price levels often total $5,000–$50,000, so a $4–$25 order from a small liquidity provider sits at the back of the queue and earns nothing.

This scanner does the math automatically. It looks through every active market on Polymarket and only shows the ones where a small order would actually take a meaningful share of the best level — without being instantly outbid by whales.

---

## How it works, step by step

### 1. Loading markets from the Polymarket API

On every page load the app paginates through the `gamma-api.polymarket.com/markets` endpoint in batches of 100. It keeps going until the API returns no more results or until 30,000 markets are loaded (whichever comes first). A counter at the top shows the progress in real time: `Loaded 12,400 markets...`.

For each market the app reads:
- `rewardsMinSize` — the minimum order size (in shares) required to earn rewards
- `rewardsMaxSpread` — the max distance from midpoint that still qualifies
- `clobRewards[].rewardsDailyRate` — the daily USDC pool advertised on the market's rewards tab
- `clobTokenIds` — the token IDs for the YES and NO outcomes
- `events[0].slug` — the parent event slug (needed to build the correct market URL)
- `events[0].tags` — used to assign a human-readable category

### 2. Pre-filtering by reward program parameters

Before touching the order book, the app discards every market that fails at least one of these hard rules:

| Rule | Reason |
|------|--------|
| Category is not "politics", "middle-east", or "iran" | User preference |
| `rewardsMinSize` is exactly 20, 40, or 50 | These are the only stake sizes we want to take |
| `rewardsMaxSpread` < 5¢ | We only want tight markets |
| Daily reward rate ≥ 20 USDC | Below this it isn't worth the capital lock-up |

Out of ~30,000 total markets, this typically leaves a few hundred candidates.

### 3. Loading the order book for each candidate

The candidates are interleaved by `minSize` group (one with 20, one with 40, one with 50, repeat) so that all three buckets get fair coverage. Then the app fetches the order book for each one from `clob.polymarket.com/book?token_id=...` in batches of 10 with a 100 ms pause between batches — fast enough to be useful, slow enough to respect the CLOB rate limit (~300 requests per 10 seconds).

Up to 1,000 order books are fetched per refresh cycle. A live count `books: 247` shows how many have been processed.

### 4. The order book check — the heart of the scanner

For each market with a loaded book, the app computes:

- **Best bid price** and **best ask price**
- **Top bid volume in USD** = `best_bid.size × best_bid.price`
- **Total bid volume in USD** = sum of `size × price` across all bid levels
- **Our stake in USD** = `rewardsMinSize × best_bid.price`
- **Our share of the best level** (`pct`) = `our_stake / top_bid_volume × 100`
- **Real spread in cents** = `(best_ask − best_bid) × 100`

A market is shown only if **all** of these are true:

```
totalBidVol >= $10        — book isn't empty
totalBidVol <= $4000      — book isn't so deep that we're invisible
pct >= 5%                 — our order takes a real share of the top level
pct <= 100%               — sanity check on the math
spread < 5¢               — actual current spread, not just the reward rule
```

The total-bid-volume ceiling is the most important number in the whole app. It's what stops the scanner from returning markets where the best level already has thousands of dollars sitting on it. If you want fewer, higher-quality candidates, lower this cap. If you want more candidates with smaller stake-to-book ratios, raise it.

### 5. Sorting and display

Markets that pass every check are shown in two formats — a dense table and a card grid — both switchable from the top of the page.

The table columns:
- **Market** — question text plus a category pill (Crypto, Sports, Esports, etc.)
- **Reward** — daily USDC pool for this market
- **Spread** — real current spread from the order book (not the static reward rule)
- **Min** — `rewardsMinSize` (20, 40, or 50)
- **Book** — a horizontal mini-visualization of the top 3 bids (green, left) and asks (red, right). A blue arrow marks where our stake lands.
- **% in Book** — our stake as a percentage of the best level. 50% means we'd own half that level
- **Open ↗** — opens the market on Polymarket in a new tab

Cards show the same information in a vertical layout, with a hero block highlighting the reward and the % in book, a 2×2 stat grid (spread / best bid / your stake / close date), a five-dot competition indicator, and a collapsible order book panel.

Default sort is by daily reward, descending. Click any column header to re-sort.

### 6. Refreshing

The full market list is reloaded every 5 minutes. Order books for the current candidate set are refreshed every 60 seconds. The "Live · 30s" pill in the header is the visual heartbeat.

---

## Technical notes

- **Stack:** a single `index.html` file with React 18, Babel Standalone, and no build step. Everything runs in the browser. Deployed on Vercel directly from a GitHub repo with one click.
- **No JSX:** the UI is built entirely with `React.createElement` calls. This avoids Babel transpilation edge cases that caused several blank-screen incidents during development.
- **No external state:** all data lives in React component state. There's no database, no backend, no API keys. The only external calls are to the public Polymarket Gamma and CLOB endpoints.
- **No build / no install:** to deploy, replace `index.html` in the GitHub repo. Vercel rebuilds automatically.

---

## Things to know before using it

**Rewards are not guaranteed.** Polymarket pays rewards based on a continuous scoring formula that depends on how close to the midpoint your orders are and how long they sit there. A market passing this scanner is a *good place to try* — it isn't a promise of payout.

**The order book moves fast.** A market that looks great when scanned can be flooded with whale orders ten minutes later. Always re-check the book on Polymarket itself before placing an order.

**Capital lock-up.** Maker orders tie up your USDC until they fill or you cancel them. Markets with later close dates lock capital longer.

**Categories are best-effort.** The category pill is assigned from event tags first, then guessed from keywords in the question if no tag matches. Some markets show "Other" because Polymarket doesn't tag them clearly.

**The 30,000-market cap is intentional.** Loading more than that hits API rate limits and slows the page load to over a minute. In practice 30,000 covers everything currently active with rewards.

---

## Configuration knobs (for editing the code)

If you want to tune the scanner, the important constants are:

- **Line ~199** — `var ok=totalBidVol>=10&&totalBidVol<=4000&&pct>=5&&pct<=100&&spreadCents<5;`
  Change `4000` to lower (e.g. 2000) for stricter, fewer candidates; raise to 6000 for more.
  Change `5` (in `pct>=5`) to raise the minimum share requirement.

- **Line ~478** and **~498** — `if(getRew(m)<20)return false;`
  Raise `20` to demand higher daily rewards.

- **Line ~215** — `if(all.length>=30000){done=true;}`
  Raise the market load cap if Polymarket adds many more active markets.

- **Line ~285** — `for(var i=0;i<Math.min(markets.length,1000);i+=BATCH)`
  Raise `1000` to check more order books per cycle (slower, but more coverage).

---

## Filter cheat sheet

| What you see on screen | What it actually means in code |
|------------------------|--------------------------------|
| Min stake size: 20 / 40 / 50 | `rewardsMinSize === 20 / 40 / 50` |
| Reward ≥ 20 | sum of `clobRewards[].rewardsDailyRate` ≥ 20 USDC |
| Spread < 5¢ | `rewardsMaxSpread` < 5 |
| Share ≥ 5% | our stake is at least 5% of best bid level volume |
| (silent rule) | total bid volume across all levels ≤ $4000 |
| (silent rule) | real spread in order book < 5¢ |
| % in Book column | `(our_stake_USD / top_bid_volume_USD) × 100` |
| Book mini-chart | bid bars left, ask bars right, width = relative volume |
| Closes | `endDate` from market |

---

## Known limitations

- The app can't read the exact reward formula Polymarket uses, only the published `rewardsDailyRate`. The real payout depends on how many other makers are providing liquidity and how close to mid you are.
- Markets sometimes redirect to localized URLs (e.g. `/ru/event/...`) that 404 — when this happens the slug is wrong. The app handles this by always using the parent event slug from the API.
- CLOB rate limits can briefly cause empty books for some markets. They reappear on the next 60-second refresh.
- The "Updating..." indicator stays visible while books are loading on first page open; this is normal and goes away after ~30-60 seconds.
