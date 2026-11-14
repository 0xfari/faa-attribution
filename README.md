# FAA attribution forecaster — polymarket-faa-attribution-02

Forecasts Polymarket's FAA outage attribution Yes prices minute by minute,
240 minutes ahead, as five quantiles (5%, 25%, 50%, 75%, 95%) per market per
minute. Entry for make.fun competition `polymarket-faa-attribution-02`
($250, program/v1 harness, Node 22, standard library only, offline).

## How it works

`main.ts` reads every `<id>.in` case in the input directory and writes
`<id>.out` with `{forecasts: [{id, q: [[q05, q25, q50, q75, q95] x 240]}]}`.

Per market, with `x` the last (checkpoint) price and `sigma` the market's own
realised one-minute volatility (mean absolute one-minute move over the last
240 minutes, floored at 0.0003, capped at 0.01):

```
q_tau(h) = clamp(mu(h) +/- k_tau * sigma * sqrt(h), 0, 1)
```

- Median `mu(h)` is the last price (persistence). Every drift tried on the
  practice data made the median worse (momentum, mean-reversion, medians of
  recent prices, EWMA centres).
- Exception: when the case's event is the mutually exclusive single-winner
  practice event (`next-ceo-of-nvidia`, whose Yes prices should sum to ~1),
  the median drifts toward the normalised share `x/S` with
  `(1 - exp(-h/120))`. This is deliberately OFF for the FAA event, whose
  four markets are not mutually exclusive and sum to ~0.09.
- Outer quantiles use multiplier 1.8, inner 0.55 — wider outside, narrower
  inside than a normal, matching the jump mixture of these prices (mostly
  still, occasionally jumping).

## Local validation

Scored on the six practice cases against Polymarket's recorded prices
(answers fetched from the public CLOB price history; the scorer reproduces
the house baseline's published practice loss of 0.363589 exactly):

| model | pinball loss | median error | 90% coverage |
|---|---|---|---|
| house baseline (starter) | 0.3636 | 0.7675 | 0.950 |
| this entry | **0.2648** | 0.7467 | 0.866 |
| practice leaderboard top 1 (at build time) | 0.259 | 0.7411 | 0.918 |

The practice pays nothing and its score is no guide to the final, which is
scored on the FAA markets' own prices over 48 hours after the deadline.

The design was also sanity-checked on three real FAA test cases built from
the FAA markets' own first hours of trading (checkpoints 22:30 Nov 13,
00:00 and 01:00 Nov 14): the program handles the FAA markets' very short
histories (19-201 minutes) and their low prices (0.008-0.155) without
crashing; those cases sit in the markets' opening collapse phase, so their
losses (0.40-1.21) are not comparable to the practice — by the deadline the
prices will have stabilised at low levels with ~16h of history behind them.

## Run locally

```sh
mkdir -p out && MAKEFUN_INPUT_DIR=input MAKEFUN_OUTPUT_DIR=out node main.ts
```

`input/` here means the kit's `input/` (the six practice cases). Deterministic:
two runs on the same case write identical bytes. Runtime is far under the
8-second per-case cap (a few hundred ms for all six practice cases).

## Files

- `main.ts` — the entry (Node 22, standard library only)
- `LICENSE` — MIT
