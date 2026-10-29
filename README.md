# Polymarket Copy Trading App

A local copy-trading app and rolling research workflow for **Polymarket International**
(not Polymarket US). It screens the observed macro/geopolitical books of other accounts,
allocates to every qualifying account using inverse volatility, and follows changes in their
relative macro-position values, in sandbox or real-money mode. Sandbox fills are assumed at the
current bid/ask midpoint; real trading uses midpoint limit orders and confirmed fills.

## Get the app

Download [`copy_trading_app.zip`](copy_trading_app.zip) and extract it into a writable
directory. It contains the `Copy Trading App/` folder; run every command below from inside it.
The detailed guides referenced below (data retrieval, final simulation, results, validation)
are inside that folder.

| Item | Value |
| --- | --- |
| Project folder | `Copy Trading App` |
| Local address | http://127.0.0.1:8790 |
| Venue | Polymarket International |
| Workflow | Account screening, inverse-volatility allocation, position copying |

## Install and prepare

Use Windows, Python **3.14**, an internet connection, and a current browser.
Python 3.14 is needed by collectors that use `Executor.map(buffersize=...)`.
PDF generation also uses the standard Windows Arial and Arial Bold fonts at
`C:/Windows/Fonts/arial.ttf` and `arialbd.ttf`.

Extract into a writable directory. Open PowerShell in `Copy Trading App`:

```powershell
py -3.14 -m venv .venv
& .\.venv\Scripts\python.exe -m pip install -r requirements.txt
& .\.venv\Scripts\python.exe prepare_project.py
& .\.venv\Scripts\python.exe data_retrieval\prepare_history.py init
```

`prepare_project.py` assembles the source into `data_storage/runtime`, preserving
the original sibling-module imports and the app's relative research-data paths.
It does not modify trading logic. Run it before startup, not while that generated
instance is running. Re-running it updates source copies and retains stored data.

**Historical data preparation is required before the current app can start.**
Its configuration selector, replay, and history reuse read prepared research
outputs. Empty storage is not a completed simulation. Follow
Data retrieval (`Copy Trading App/data_retrieval/README.md`) in order: discover markets, collect
their trade histories, prefilter activity/frequency, complete candidate histories,
collect inventory actions and historical prices, reconstruct books, then simulate.
Use Final simulation (`Copy Trading App/final_simulation/README.md`) for the screening formulas,
parameter grid, report commands, and data-dependent checks.

Preparation can be substantial: the research uses a year of warm-up and a year
of replay. Public API coverage, retention, rate limits and available historical
prices determine what can be reconstructed. New downloads need not reproduce
the original market universe or exact published numbers.

```powershell
& .\.venv\Scripts\python.exe prepare_project.py --check-ready
```

This read-only check must pass before startup. Do not create fictitious account
histories, selected-account files or equity curves to satisfy it.

## Start and stop

After preparation, double-click **Start Copy Trading App.cmd**, or run:

```powershell
& .\.venv\Scripts\python.exe data_storage\runtime\polymarket_mock_app\server.py --port 8790 --open
```

Open http://127.0.0.1:8790. Keep the server and computer running for forward
monitoring. The root launcher keeps a server console open; press **Ctrl+C** there
to stop it. Pause copying first, and for real trading let outstanding order
cancellations/fills reconcile before stopping. Closing only the browser does not
stop the server. If port 8790 is occupied, use a different `--port` and its matching
browser URL rather than starting a second process against the same ledger.

The retained app-level `launch.ps1` is a background launcher. Use it only in
the generated runtime app directory, with the intended Python environment on
PATH. It writes `data/server.pid`; closing its app window leaves its server
running. The root launcher above is the simpler foreground startup/shutdown path.

## Use the app

1. **Strategy settings:** create a named sandbox or real-money portfolio, choose
   a prepared configuration, and enter its capital budget. Portfolios have
   separate ledgers; the portfolio selector changes the view, not the mode of an
   existing portfolio and not whether another portfolio keeps running.
2. **Start copying:** discover current macro/geopolitical participants and screen
   their preceding history. Allocation starts when a complete selection snapshot
   is ready. The historical followed set is not automatically selected today.
3. **Portfolio:** view NAV, compounded return, daily strategy Sharpe, allocation,
   current status, next reset and the execution journal.
4. **Followed accounts:** inspect every selected wallet/name, historical metrics,
   inverse-volatility weight, reset budget, invested value, cash and current
   contribution. Account detail shows portfolio performance and, when separately
   prepared, the account's historical macro-book chart.
5. **Positions held:** view current shares and market values, largest first, and
   filter by followed account. **Transaction history** links our fills to source
   accounts and position-change decisions; exports provide fills, selections and
   NAV. An observed source trade is attribution evidence, not a promise of a
   one-for-one copied execution.
6. **Discovery & screening:** monitor collection, activity checks, reconstruction,
   qualification and unresolved history. Pause/resume discovery here.
7. **Execution:** inspect order states; connect, enable or disable real routing;
   reconcile an uncertain submission using its actual exchange order ID.
8. **Historical replay:** explore the prepared configuration's research curve
   and account sets. This is separate from current sandbox/live performance.

**Pause copying** retains holdings and continues marking them; it cancels resting
limits and reconciles already submitted real orders. Manual account exclusion in
sandbox requests an exit/reallocation and persists for future selections in that
portfolio. On restart, sandbox restores its saved holdings and catches up to the
latest observed source book. It does not backdate executions during downtime.
Real portfolios restart paused, with credentials disconnected.

## Account selection

The default defined in the active code is **Effective Sharpe in (3, 4], a 75-day
account reset, and a 365-day maximum volatility window**. Other prepared cases
have different thresholds, ranges, reset periods and volatility windows; the
selected configuration controls them. There is no top-N selection or return rank.

At each reset, discovery uses market participants from the classified macro/geo
catalog, including newly discovered markets. The classifier covers economic
indicators, central banks, macro assets, geopolitical events, national government,
elections and policy. It uses category metadata plus text rules and explicit
subject exclusions; it is not a perfect human taxonomy.

The current activity/style rules are:

- At least **30 distinct active UTC macro/geo trading days in the preceding 40**.
  An active day contains a macro/geo BUY or SELL.
- Average **strictly fewer than 50 macro/geo trades per calendar day**, over
  available preceding history, capped at 365 days. Inactive calendar days remain
  in the denominator. Trade fills are grouped using transaction/token/side/time.
- At most **10% of closed shares held less than one hour**, at most **20% of
  token/hour groups with both BUY and SELL**, and at most **20% of market/hour
  groups involving both outcome tokens**, using the preceding-year window.
- The chosen Raw or Effective Sharpe cutoff/range must pass, and the lagged
  allocation volatility must be positive and computable.

Activity/frequency gates precede expensive portfolio reconstruction, with exact
checks again on reconstructed inputs. No macro-participation percentage, median
holding-time band, minimum completed-market count, P&L ranking or minimum selected
account count is applied. Missing/reconciling histories are identified separately.

Screening returns describe **the trader's active macro/geopolitical book,
conditional on being invested**, not their whole-account returns. Earlier marked
position values weight subsequent price returns. Absolute book-size increases
are not gains, and no unobservable category cash is inferred. Unpriced positions
enter the return book only when historically observed prices become available.
Available invested observations within the preceding 365 calendar days determine
the sample; a full year is not fabricated. See the formulas in
Final simulation (`Copy Trading App/final_simulation/README.md`).

## Allocate between accounts

Every qualifying account receives a capital weight proportional to `1 / sigma`,
normalized across that selected set. `sigma` is the sample standard deviation
of its macro-book daily returns, using the configuration's lagged window. The
return ending at the reset boundary is omitted from allocation volatility.

For example, volatilities of 1% and 2% produce account weights of 66.67% and 33.33%.
At reset, those weights multiply **current portfolio NAV**, including prior gains
and losses. This compounds; it is not a repeated fixed-dollar investment.
Between account resets, slices gain/lose value and their portfolio percentages
drift. One trader changing positions does not trigger a transfer of another
trader's allocation. Cross-account target budgets are recalculated at resets or
an explicit manual removal/reallocation.

## Copy positions, purchases, sales and cash

The app follows the **relative marked values of the source's macro positions**,
including positions already open when the source is first selected. It does not
observe or reproduce that trader's total-account cash percentage.

Within a followed account's slice, the target cash value for a position is the
slice's current equity multiplied by the source position's share of the macro
book; target shares divide that value by the observed price. The executor trades
the difference between target and already-held shares, selling reductions before
funding purchases. Each slice is limited to its own available cash and shares.

- **A purchase or addition** changes relative source weights and may require
  buying that position and reducing another position within the same slice.
- **A partial sale** reduces that source position's relative weight. Because the
  remaining book is normalized again, proceeds may be redistributed to its other
  positions. A SELL does not necessarily leave permanent cash.
- **A full exit from one market** targets zero shares there and may increase the
  remaining market weights. If the source exits the entire macro book, its slice
  stays in cash until a later observed position change.
- **Absolute scaling without changing relative weights** need not generate any
  trade. The app copies allocation proportions, not absolute source share counts.
- **Cash also remains** when quotes are unavailable, purchases lack slice cash,
  real limits are unfilled, or resolved/redeemable source value reserves cash.
  The target denominator includes resolved source value, but resolved targets are
  not newly purchased.
- **At account reselection**, continuing accounts keep their existing shares
  subject to the new targets; removed accounts are targeted flat and new accounts
  receive their selected allocations. Multi-order execution is not instantaneous
  or atomic in real mode.

Example: an allocated trader initially has equal-value B and C positions and then
sells C, leaving only B. The app targets the available slice into B, so it can sell
C and buy more B. It does not reproduce an unobservable source cash balance.

## Position monitoring versus account resets

Source positions/activity refresh continuously, targeting **five seconds after
each completed account check**; the portfolio worker also wakes approximately
every five seconds. API latency and indexing delays make actual intervals longer.
Public order books update independently through the WebSocket and REST checks.
An observed inventory change triggers new targets; price movement alone does not
continually rebalance unchanged source quantities.

Account reselection uses the chosen **reset_days** (75 by default). The first
screen uses the current UTC date boundary and available preceding history; later
resets repeat discovery, filters and inverse-volatility weights. A late screen
executes at then-current prices. The UI displays Chicago time, while historical
screening boundaries and stored epoch timestamps use UTC.

## Sandbox, real execution and historical simulation

| Mode | Position adjustment | Fill/return assumption |
| --- | --- | --- |
| Sandbox app | Observed source-book changes between resets | Immediate BUY/SELL at exact `(bid + ask) / 2`; no depth, trade-print, queue, fee or venue-minimum requirement; valid quotes and slice cash/shares still required |
| Real app | Same source targets and account allocations | Post-only GTC midpoint limits for both sides; BUY rounds down and SELL up to the valid tick; check pending limits every 60 seconds and replace an unfilled remainder only after cancellation; confirmed exchange fills only |
| Final historical research | Daily macro-book return changes; account weights reset by configuration | Compound each selected account's daily modeled book return; zero costs; no live order queue or limit-fill simulation |

Real execution respects venue minimum quantities/ticks, available collateral,
share balances and reservations. Partial fills remain partial; unknown submissions
block automatic resubmission until reconciled. The code records confirmed maker/
taker fills and accounts for its implemented exchange-fee calculation. It does
not assume real midpoint limits will fill. Sandbox has no spread cost at entry
by construction and does not estimate executable market impact.

Real fills can leave cash or incomplete target adjustments; historical returns
therefore are not a live execution forecast. Research is daily, whereas the app
responds to observed inventory changes throughout the day. The saved replay is
not recalculated from the app's forward fills.

Source settlement values enter historical marks only after their recorded
availability. Sandbox credits observed resolved payouts. Real holdings require
the user to redeem through Polymarket; the app verifies token extinction and a
matching collateral change before treating proceeds as spendable cash.

## Real-money credentials

Creating a real portfolio requires the existing **trading wallet address, signing
private key, CLOB API key, API secret and API passphrase**. The app checks funding
and permissions before creating it. Use the local form, then explicitly enable
real routing and start copying. Merely connecting does not submit orders.

The helper `data_storage/runtime/polymarket_mock_app/generate_clob_credentials.py`
prompts for a signing key and creates/derives international CLOB credentials. It
does not place an order. Keep its displayed secrets private. The application's
credentials remain in server memory, not its persistent ledger. `.env.example`
is a reference template only; the current app does **not** automatically load it.

Use an account dedicated to this strategy: unrelated orders and external balance
changes require reconciliation. This package's offline checks do not validate a
funded wallet, approvals, venue access or actual authenticated order execution.

## Results and validation

Result notes (`Copy Trading App/final_simulation/RESULTS.md`) summarize the existing completed
research and its limits. They are not a new historical replay of freshly downloaded
data. Validation (`Copy Trading App/VALIDATION.md`) distinguishes source/accounting checks from
historical-data and authenticated checks.

```powershell
& .\.venv\Scripts\python.exe prepare_project.py --syntax
& .\.venv\Scripts\python.exe run_checks.py
```

Run these after assembling the runtime. The runner omits three explicitly listed
historical-data tests; the remaining checks use local fixtures and do
not submit real orders. Do not run the separate public-feed checker or discovery
when expecting an offline test. Research/account return moments use invested days;
strategy Sharpe includes calendar cash days. Forward app Sharpe uses completed
24-hour periods since funding and is unavailable until there is enough valid
variation/history; total return includes current marked NAV.
