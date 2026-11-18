# btc-race-24h-20261118-1444

[![make.fun · btc-race-24h-20261118-1444](https://make.fun/badge/btc-race-24h-20261118-1444.svg)](https://make.fun/bounty/btc-race-24h-20261118-1444?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261118-1444)
[![rank](https://make.fun/badge/btc-race-24h-20261118-1444/Vins1206/btc-race-24h-20261118-1444/rank.svg)](https://make.fun/bounty/btc-race-24h-20261118-1444/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261118-1444)
[![reward](https://make.fun/badge/btc-race-24h-20261118-1444/Vins1206/btc-race-24h-20261118-1444/reward.svg)](https://make.fun/bounty/btc-race-24h-20261118-1444?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261118-1444)

Half-hour dip bot for make.fun's "Race a BTC trading bot live for 24 hours" bounty, $250 in USDC. It trades Coinbase Wrapped BTC (cbBTC) against USDC on 1-minute on-chain bars, starting from $1,000 of paper money, speaking `agent-fun/bars-replay/v1` (newline-delimited JSON on stdin/stdout: one `hello`, one `batch` per bar, one `reply` per batch). Pure Python standard library, single file: `main.py`.

## Strategy

Long only, one open position, full size. It buys sharp half-hour dips and lets each trade run to a target instead of scalping 1-minute noise:

- **Entry:** buy the whole cash balance when the latest close is at least **1.2% below the close 30 bars ago**, provided the 120-bar return is no worse than −3% (no catching a knife in a steep two-hour slide), fewer than 4 entries have filled, and we are not in the last 120 bars of the window.
- **Exit:** sell everything at **+1.6%** above the entry fill price (take profit) or **−1.0%** below it (stop loss). After a stop loss, wait 30 bars before the next entry. A position stalled for 480 bars (8 hours) is sold too, so the cash is free for the next dip.
- **Fallback:** if no entry has filled by bar 1080 and the 120-bar return is no worse than −0.5%, buy once and hold to the end, so a calm rising window still produces a trade.

## Why this shape

On the dev window the 1-minute closes move only ~7 bps per minute (standard deviation), while a full-size round trip costs roughly 26–40 bps in fee (4 bps), modelled slippage (2 bps) and price impact at the dev window's pool depths (~0.7–1.0M USDC), so fast signals churn money away — the EMA-cross house bot loses over $100 on the dev window doing exactly that. Half-hour dips of ~1.2% that recover ~1.6% within hours are the moves big enough to pay that cost. Neighbouring parameters (dip 1.1–1.2%, take profit 1.3–1.7%) are all profitable on the dev window, so the result is a plateau, not a single tuned spike.

## Local validation (dev window, exact-integer replay)

Replaying the real `main.py` against the kit's frozen dev window (1,440 bars, fee 4 bps + slippage 2 bps + constant-product price impact, fills at the next bar's open):

- **Net PnL: +$14.11** (passes the dev gate: > $0 after all fees)
- **Trades: 2 fills, 1 round trip** (buy bar 65, sell bar 479 at the +1.6% take profit)
- **Max drawdown: 1.5%** (limit 25%)
- **Deterministic:** two runs produce byte-identical replies

The dev window itself fell −2.8% (first close ~$65,083, last ~$63,261), so buy-and-hold loses; this bot profits by catching one dip and its recovery.

## Run it

```sh
python3 -P -s main.py  # reads hello/batches on stdin, replies on stdout
```

Grade it on the dev window with the kit (needs Docker for the Python runtime):

```sh
node cli/makefun.mjs grade-local --kit . --bot <repo> --driver docker
```
