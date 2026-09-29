# Cashflow: Rat Race — project context

Browser version of Robert Kiyosaki's *Cashflow* board game (Rat Race only), played online with friends.
Everything is in one self-contained file: `cashflow.html` (HTML + CSS + vanilla JS, no build step, no framework).

- **Live site:** https://federico-pereira.github.io/cashflow/cashflow.html (GitHub Pages, `main` branch, root)
- **Owner/host:** Federico. He always hosts; friends join.

## How to work on it

- Edit `cashflow.html` directly. No dependencies except PeerJS, which is loaded at runtime from `cdn.jsdelivr.net`.
- After every edit, check the JS still parses:
  ```
  node -e "new Function(require('fs').readFileSync('cashflow.html','utf8').match(/<script>([\s\S]*)<\/script>/)[1]); console.log('OK')"
  ```
- Commit and push to `main`; GitHub Pages redeploys in about a minute. Testers should hard-refresh (Ctrl+Shift+R) or use incognito, because the CDN caches old copies.
- To test multiplayer locally, open the page in a normal window and an incognito window.

## Architecture

- One global `state` object plus a `render()` that rebuilds `#app.innerHTML` from template strings. Inline `onclick` handlers call global functions.
- Phases: `"setup"` (lobby) → `"playing"` → `"over"`. `render()` falls back to the lobby if a game phase has no valid player data, rather than crashing.
- `MARKET` is a static array of 8 stocks and 8 properties. Their live fields (price, prevPrice, history, high52, low52, basePrice0) change during play and are synced via `snapshotMarket()` / `applyMarketSnapshot()`.
- Board centre: title, whose turn, a public **economy strip** (mortgage rate for new purchases, bank loan rate, global rent rate — each with its change in points vs last week; click the rent box for each type's rate (local `showRentTable`) — and stock/property averages as % change vs last week, from `prevMortgageRate`/`prevPrice`/`prevRent` snapshotted in `updateMarket` — plus week and last market event; `economyHTML`) and a collapsible board guide (legend; local `showLegend`, remembered in localStorage `cf_legend`). Indexes are market-wide only, never single-offer values.
- UI: a 7×7 perimeter board in the middle; Finance (left) and Market (right) open as side popups; event/charity/flash-deal prompts are modals over the board; shared "Rat Race Progress" bars below.

## Multiplayer (PeerJS / WebRTC, star topology)

- Fixed room: `FIXED_ROOM = "HOME"` → host peer id `cashflow-ratrace-HOME`. There is no room code in the UI.
- Joining: the page tries to claim the host id. If it's taken (PeerJS error `unavailable-id`, logged as `ERROR PeerJS: ID ... is taken` — **this is expected, not a bug**), it joins as a guest instead.
- Messages: guest → host `hello {name}`; host → guest `welcome {myIdx}`; anyone → `state {data}`. The host applies incoming state and relays it to the other guests.
- Every mutating action updates local `state`, then calls `syncRender()` (`pushState()` + `render()`). `pushState()` sends `sharedSnapshot()`; receivers call `applyShared()`.
- **Shared fields:** phase, speed, numPlayers, names, profs, ready, players, turn, pending, hasRolled, log, listings, listingSeq, week, mortgageRate, prevMortgageRate, bankRate, prevBankRate, rentRate, prevRentRate, rentAdj, lastMarketEvent, alert, lastRoll, market. Each player also carries `costBasis`/`buyMarks` (what they paid) for the statement's holdings charts.
- **Local-only fields (per browser):** solo, showRentTable, showLegend, alertSeen, panel, marketTab, selectedStock, selectedHolding, qty, confirm, debtSel, flashQty, loanTakeAmt, loanPayAmt, viewTab, myIdx, isHost, online.
- Guests never run `startGame()` (only the host does). Any local-only default the UI needs must be seeded at page load, not in `startGame()`. Missing `state.finance` on guests caused a crash before.
- Uses Google STUN plus the free OpenRelay TURN servers (`ICE_CONFIG`). A 15-second timeout falls back to local pass-and-play.
- **Does not work inside a claude.ai artifact:** that sandbox blocks WebRTC by CSP. It must be hosted normally (GitHub Pages).

## Game rules and design decisions (from Federico)

- Lobby: type a name, join, host picks 2–4 players, everyone clicks **Ready up**, only the host sees **Start Game** (enabled when all seats are filled and ready). Rejoining with the same name reclaims your seat.
- Professions are **random** at game start (no picker).
- **Game speed** (host picks in the lobby, shared `state.speed`, default medium): `SPEEDS` sets property deal flow (offers per week, snap-up chance, max on board), the starting rent rate and extra starting cash. Fast = more deals + 12% rent + $800 per player; Medium = more deals, 10%; Slow = scarce deals, 10%. Measured 2-player winner: Fast ~19 weeks, Medium ~29, Slow ~45 (≈ minutes at 1 min/turn); professions within ~4 pts with 2 players, up to ~6 with 3. A bigger Fast cash bonus makes the Janitor too strong; more deals alone makes rich professions too strong.
- Per-speed, per-player-count profession tuning: `SPEEDS[k].prof = {2:{...},3:{...},4:{...}}` scales each profession's expenses/debts at game start (`professionSheet`; solo uses the 2-player set). So far only **Medium 2-player** is tuned (all professions within ±0.8 pts of fair: 49.2–50.8%); the other 8 combinations use the base sheets (about ±2–3 pts). Tune more with the scratch `tune3.js <mode> <players>` bot tuner.
- **Solo** (`chooseSoloMode`, local `state.solo`): one player racing the clock; the game ends on escape or bankruptcy (`checkGameOver` allows 0 racers when alone); skipped Downsized turns still advance the week; best time per speed saved in localStorage `cf_best_<speed>`. Bot-measured solo escape: Fast ~18 weeks, Medium ~27, Slow ~40.
- Turns: the current player clicks **Roll Dice**, resolves any prompt, then clicks **End Turn**. Rolling never ends the turn automatically. `state.hasRolled` controls which button shows.
- The **market is open to every player at all times**, not just on your turn. Trades apply to the acting player: `actingIdx()` = `myIdx` online, `turn` in local mode.
- Only the current player can roll, end the turn, or answer charity/event/flash-deal prompts.
- Each player sees **only their own financial statement** online. Progress bars are visible to everyone.
- Finance sheet has two overview cards (monthly cash flow, escape progress); net worth was removed on request.
- Financial statement mirrors the real game sheet: 1 Income, 2 Expenses (left column), 3 Assets, 4 Liabilities (right column); Cash on top; goal bar can go past 100%.
- Stocks: always-open table with random starting prices each game, a simulated 15-week history (random walk), 52-week range from that history, first "Chg" = change from the last simulated week, click a row to expand a labelled sparkline.
- Real estate: **only** through time-limited live offers (FOR SALE / WANTED) that expire and get replaced each turn. Full data on each listing (rent, cap rate, appreciation, vs-market %).
- Real estate: each offer (listing or flash deal) has its own terms, all random: down payment `downPct` (bell curve ~10%, 0–30%) and running costs `costPct` (bell curve ~26% of rent, 10–42%). The rest of the price is a per-property mortgage (`p.reMortgage`), **fixed-rate**: it keeps the mortgage rate from the day it was bought (`p.reRate`, balance-weighted); rate events only affect new purchases. Each game starts at a random 4.5–5.5% (limits 2–12%). Property cash flow = lease rent − running costs (% of that rent, unit-weighted `p.reCostPct`) − mortgage payment (balance × locked rate ÷ 12). FOR SALE prices are a bell curve ~1% under market value (±9%), WANTED around +5% (±8%). Selling pays off that mortgage from the sale price.
- Rent: market rent = the property's normal value (`anchor`, moved only by events) × rent rate ÷ 12. Rent rate = global `state.rentRate` (starts at 10%/yr, 6–15%) × per-type `state.rentAdj` (starts ×1, 0.6–1.6). Rent Control / Rental Demand move the global rate; single-property events move one type. Owners earn **lease** rent (`p.leases[id]` = lots of {qty, rent, until}): signed at purchase, renewed to the current market rent every 5 weeks (`LEASE_WEEKS`, `renewLeases()` in `nextTurn`). Property flash deals come with their own lease rent, market rent ±5%.
- Events: every event rolls its own size (e.g. a crash is −20…−50%) and its popup/log show the actual numbers. Drawn 1/3 personal, 1/3 market-wide (crashes, rallies, interest/mortgage rates), 1/3 specific to one stock or property type (can also change rents and dividends, which sync with the market snapshot and reset each game).
- Bank loans: $1,000 increments; partial payoff allowed. Monthly payment = balance × the shared bank rate (`state.bankRate`, starts at 10%/month, moved by Interest Rate Hike/Cut, 6–16%). Always recomputed from the rate (`bankPmtFor`), so borrowing or paying part of a loan never resets an event's effect.
- Any liability (home mortgage, car, credit card, retail, bank loan, property mortgages) can be paid down from the Finance panel, partly or in full. Personal debt payments shrink proportionally; paying one off removes that expense.
- "Market" board squares give a private one-turn **Flash Deal**. Its price vs market is a bell curve (mean 8% off, σ 12%, −20%…+35%): ~3 in 4 are bargains, the rest fair or overpriced. The game never says which (no "% vs market" anywhere, flash deals or real estate listings) — players judge prices themselves. Real estate offers (flash deals and listings) show the current market value as a reference, since there is no public property price board; stock offers link to the always-open stock table instead. No quantity cap on stocks; a live preview shows total, cash used, loan needed and cash after.
- Random global/personal events (crashes, rallies, rate changes, lawsuits, inheritance...).
- Winning and places: first player to escape (passive income ≥ total expenses) takes #1; others keep playing for #2, #3… Bankrupt players take places from the bottom. The game ends when at most one player is still racing; a final standings table shows. Shared `alert` popups announce wins, fire sales and bankruptcies (dismissed per browser via local `alertSeen`).
- Bills from the board (doodads, Downsized, lawsuit/identity-theft events) become a `pending` of kind `bill` with a **Pay** button (`payPendingBill`); the turn can't end until it's paid. Short on cash → the loan popup ("Take $X loan and pay"); bank refuses → the popup shows what a fire sale could raise and offers "Sell assets and pay" (or "Sell everything and go bankrupt" when that won't be enough). Cancel lets the player sell things first. Negative paydays are still paid automatically.
- Mandatory costs go through `payBill()`: cash → bank loan within the lending limit → fire sale (stocks at market, properties at 50% of value, bank forgives underwater mortgages) → if still short with nothing to sell and negative cash flow: bankrupt (out); otherwise an emergency loan.
- Payday is collected when passing a Payday square, not only landing (real rule).
- Pacing, tuned with a smart-bot simulation: listings have a 30%/week chance to be snapped up, ~1.5 new ones appear per week (max 10). With fixed-rate mortgages starting at 4.5–5.5% and rent at 10%/yr of value, measured winner week (2 players): mean ~37, SD ~22 (random on purpose). Lowering the starting mortgage rate by 2 points alone speeds games up ~10 weeks — it is the strongest pacing dial.
- Profession balance: each profession's expenses and debts were scaled from the originals (roughly Doctor −24%, Janitor −6%, Engineer +1%, Secretary +12%, Teacher +19%; salary and savings unchanged) so win rates are within 5 points of each other; last measured gaps: 2 players 2.8 pts, 3 players 2.7, 4 players 3.4. Re-measure with the bot sim after changing anything economic.
- Downsized: pay a month of expenses and sit out the next 2 turns (`p.skipTurns`, handled in `nextTurn()`).
- Prices move only by the weekly random step and by events. Buying/selling never changes a price. Market events move both price and `anchor` (the level the random walk drifts to), so there is no guaranteed rebound after a crash.
- Flash-deal **stock** buys are locked for the buyer's next 3 turns (`p.locks`, `p.turnsTaken`) — no instant flipping. Property flash deals aren't locked (property can only be sold into WANTED listings). The flash-deal popup shows only the deal (price, terms, cash flow); if cash is short, Buy opens the loan popup with a "Take $X loan and buy" button.
- Bank lending limit: voluntary loans (Take Loan, or buying on credit) are only allowed while monthly cash flow stays ≥ $0 after the new payment (`loanCheck()`). Mandatory costs still auto-borrow.
- Total expenses include taxes everywhere (payday, Downsized, escape check), like the real game sheet. Event/charity/flash-deal popups have no backdrop, so Finance and Market stay usable while one is open.
- Popups sized to content, no page scrolling on the main board view, no layout jumping when numbers change width.

## Sound

- All sound effects are generated in code (Web Audio API, `SYNTH` + `sfx(name)`); no files needed. A mute button (🔊/🔇) sits in the title bar and is remembered per browser (localStorage `cf_muted`).
- Sounds play only in the browser where they matter (your roll, buy, payday…). Wins, fire sales and bankruptcies play for everyone, and "your turn" plays online; both come from `syncSounds()` after each render.
- Dice: every roll is shared (`state.lastRoll`), so all players see the dice tumble for ~0.9 s in the box above the game log (`diceBoxHTML`/`syncDice`) and hear the dice sound; other sounds from that roll wait until the dice settle.
- To swap in a real recording: add `sounds/<name>.mp3` and list the name in `sounds/index.json` (see `sounds/README.md` for names).

## Known pitfalls

- Manual copy/paste uploads to GitHub have repeatedly left a half-updated file live. Before testing, diff what's on `main` against what you meant to push.
- GitHub Pages / CDN caching makes old code appear to still be running.

## Current status

Pushed 2026-09-28: lease-based rent (10%/yr of value, 5-week leases, global + per-type rent rates), fixed-rate mortgages starting at 4.5–5.5%, rent-rate box with per-type table, three game speeds with lobby cards (Fast ~20 / Medium ~30 / Slow ~45 min for a 2-player winner), Solo mode with best times, Medium 2-player profession tuning, and a fire-sale lock bookkeeping fix. Earlier pushes (2026-09-27): loans, mortgages, events, places, fire sale/bankruptcy, sounds, dice animation, economy strip. Still to do: tune the remaining speed/player-count combinations; real two-player online test.
