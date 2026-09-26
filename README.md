# TN10 round 2: final write-up (full run + recovery)

> **Testnet-10 only. Private.** Round 1 covers the setup, findings F1–F7, vprogs and all scripts:
> **https://github.com/STP-KAS/grok-bot-vprogs** (branch `tn10-break-report`). This repo covers **round 2 only**
> and how it compares. Round 3 (vprogs contention) is in **https://github.com/STP-KAS/grok-bot-vprogs-round3**.
>
> Times are CEST (UTC+2). Figures marked ✔ were recomputed from the logs in `logs/` (and the work-dir originals)
> with `tools/report-stats.py` / a cross-midnight window script. **[unverified]** means an operator note with no log behind it.
>
> **Final data window: 22:47:26 CEST 25 Sep 2026 → 06:31:10 CEST 26 Sep 2026** (RECOVERY-1 END).
> Machine-readable summary: [`findings/round2-final-stats.json`](findings/round2-final-stats.json).

## In one paragraph

Round 2 started at **22:47:26** as a harder full-throttle push (2× fee, tighter mempool guards) and then **evolved**:
an `--utxoindex` restart, a switch to **10× fee**, a long **~1k TPS baseline** so vprogs could run, short full-throttle
contention windows, and finally a **disk taper** that wound the send rate from 1,000 down toward 0 between 05:25 and
~06:30. The storm stopped because **free disk hit the supervisor's 9.5→8.4 GB taper**, not because a clean
funds-out drain was measured. Recovery-watch then fired at **06:31:09 with mempool = 7**, so there was **nothing to
drain** and `logs/recovery/recovery-1.jsonl` was never written. Across the whole ~7 h 44 min window the network's
10 s samples show **median 1,548 TPS / mean 2,507 / peak 12,175** — but those overall averages mix full-throttle,
1k baseline, and taper regimes (see below). Peak mempool in round 2 was **~88k** (the 21:12 n0 crash at 100,001 was
**before** round 2). Storm workers burned about **197,800 TKAS** in fees. An outside **Grok Build** agent, funded with
~300k TKAS at 22:47, kept sending in parallel and is a confounder for network TPS.

## Why the storm stopped

| Cause | Role | Evidence |
|---|---|---|
| **Supervisor disk taper (9.5 → 8.4 GB free)** | **Primary** | `ramp.log` ROUND4 note at 06:40:56: RATE wound from 1000 (05:25) to 0 (~06:30). Supervisor samples: disk 9.71 GB @ 05:25 with offered≈1000 → disk 8.49 GB @ 06:31 with offered≈85. |
| **recovery-watch `offered<100` for 60 s** | Secondary trigger | `ramp.log` 06:31:09 `RECOVERY-1 START auto (offered<100 60s (funds out?))`; mempool=7. The "funds out?" label is what the watcher guessed; load was already emptied by the taper. |
| Funds exhausted | **Not independently confirmed** | Offered fell because disk taper cut RATE, not because a separate funds check tripped first. |

**Recovery measurement: N/A.** Mempool was already **7** when RECOVERY-1 started. There was no backlog to time to 50% / 10% / baseline. The sampler finished in 1 s; `logs/recovery/` stayed empty. Probe inclusion latency **during drain** is also **N/A** (no drain, probes were stopped at the utxoindex restart ~23:50).

## Duration

| | |
|---|---|
| Start | **2026-09-25T22:47:26+02:00** (`PHASE4-MAX-START` in `logs/storm/ramp.log`) |
| End | **2026-09-26T06:31:10+02:00** (`RECOVERY-1 END`) |
| Length | **7 h 43.7 min** (463.7 min) |
| RATE→0 | Operator note: disk taper reached ~0 around **06:30**; recovery-watch set `RATE_MAX=0` at 06:31:09 |

## Setup changes vs round 1 (start of round 2)

| | Round 1 (up to 22:46:14) | Round 2 (from 22:47:26) |
|---|---|---|
| Nodes | n0 + n1 until 21:33, then n0 only | n0 only; **`--utxoindex` added at 23:52** (resync 18m35s) |
| Storm fee | 1.2× min = 120 sompi/g | **2×** from 22:47, then **10×** from 00:15:37 |
| Rate policy | Full throttle → safety 8k | Full (2×) → halt for utxoindex → **1k baseline** + short full windows → disk taper |
| Mempool guards | taper 45–68k, gate 78k/55k | taper **40–60k**, gate **70k/50k**, **hard brake 90k** |
| Funds | ~666k TKAS in storm pools | ~356k after **299,993 TKAS** moved to Grok Build at 22:47:16 |

## Timeline (high level)

| Time (CEST) | Event |
|---|---|
| 21:48:41–22:46:14 | Round 1 overload (57.5 min). Mempool drained 49.6k→0 in ≤20 s after halt |
| 22:47:16 | GROKBUILD-FUND-2: 299,993 TKAS / 1,184 txs to external Grok Build receive wallet |
| **22:47:26** | **Round 2 start (PHASE4-MAX)**: fee 2×, new guards, RATE_MAX 9k→20k |
| 22:58:27 | Peak in 2× period: **10,846** network TPS (10 s) |
| 23:26:28 | Round-2 mempool peak (supervisor): **88,429** |
| 23:49–00:13 | `--utxoindex` restart; node down ~19 min; storm halted |
| 00:14:28–00:15:00 | One clipped burst (mp max 83,975); then user: full throttle continuous |
| **00:15:37** | **FEE-10X** supervisor restart (`FEE_MULT_P=10.0`) |
| 00:16:56 | **All-time peak this window: 12,175** network TPS (10 s) |
| 00:32–00:38 | Storm silent: monitor wrote sticky STOP (RAM); supervisor logged stale accepted TPS |
| 00:38→ | User: **1k TPS background** so vprogs get CPU/disk; fee stays 10× |
| 00:54–01:06 | Round-3 contention (full storm vs vprogs) — details in round3 repo |
| **05:25→06:30** | **Disk taper**: free disk ~9.7→8.5 GB; offered rate 1000→~100 |
| **06:31:09–10** | **RECOVERY-1** auto; mempool=7; sampler no-op; storm left at RATE_MAX=0 |

## Results by regime (network TPS from `nettps.jsonl`, ✔)

Overall averages alone are misleading because most of the clock time was the 1k baseline. Prefer the regime rows.

| Regime | Window | Net TPS p50 / mean / max (10 s) | Notes |
|---|---|---|---|
| **2× full throttle** | 22:47–23:49 (~62 min) | **7,687 / 7,595 / 10,846** | Closest apples-to-apples vs round 1 overload (median 6,735) |
| **10× full (early)** | 00:15–00:38 (~23 min) | **2,786 / 2,929 / 12,175** | Includes 00:32–00:38 silence (real nettps near 0; supervisor accepted TPS stale) |
| **1k baseline @10×** | 00:38–05:25 (~4 h 47 min) | **1,539 / 1,745 / 8,834** | Background for vprogs; peaks = contention full-throttle windows |
| **Disk taper** | 05:25–06:31 (~66 min) | **767 / 849 / 2,125** | RATE wound down with free disk |
| **Whole round 2** | 22:47–06:31 (**7 h 44 min**) | **1,548 / 2,507 / 12,175** | Mix of all regimes |

Supervisor **accepted** TPS (our workers only), whole window: median **1,000**, mean 1,737, max 10,049 (n=8,735).

### Fee burn (supervisor `fees_tkas`, ✔)

The fee counter **resets on supervisor restarts**. Honest total = sum of (last − first) over each continuous run inside the window:

| Run | Window | Burn (TKAS) |
|---|---|---|
| After PHASE4 start | 22:47:28 → 00:13:49 | 27,781 |
| After utxoindex / 10× restart | 00:14:01 → 00:38:25 | 15,571 |
| After 1k-baseline restart | 00:38:37 → 06:31:09 | 154,451 |
| **Total round 2** | | **≈ 197,803 TKAS** |

Hourly burn at the 1k / 10× baseline was ~26k TKAS/h (hours 02–04), consistent with ~1,000 tx/s × ~643k sompi/tx.

### Mempool (round 2 only)

| | Value |
|---|---|
| Peak (monitor) | **87,615** (23:01:20, during 2× full) |
| Peak (supervisor) | **88,429** (23:26:28) |
| Median over whole window | **356** (dominated by long 1k baseline) |
| 90k hard brake | Never needed after the 2× period's mid-50k band (inferred from max < 90k in later regimes) |
| **21:12 n0 crash (100,001)** | **Before round 2** — not in this window |

### Fee-tier probes

- **During early round 2 (P4-max, 2× spam, analysis cut ~23:29):** same numbers as the interim report — 84 probes/tier, all included:

| Tier (× 100 sompi/g) | p50 | p90 | max |
|---|---|---|---|
| 1× | 12.0 s | 48.4 s | 131.5 s |
| 1.2× | 8.9 s | 28.6 s | 92.8 s |
| 2× (= storm fee then) | 5.0 s | 11.7 s | 28.6 s |
| 5× | 1.8 s | 2.9 s | 4.5 s |
| 10× | 1.2 s | 1.6 s | 2.4 s |
| 100× | 1.1 s | 1.6 s | 2.1 s |

- **During final recovery / drain:** **N/A** — no backlog, no probe run, empty `logs/recovery/`.

### Disk

| | |
|---|---|
| Free at round-2 start | 46.57 GB |
| Free at RECOVERY-1 | **8.49 GB** |
| n0 datadir growth | Dominated by consensus + utxoindex (~88 GB combined by morning); TN10 pruning window ~42 h held the storm blocks |
| Binding limit | **Disk**, as projected at 22:51 — not funds |

## Side-by-side: round 1 overload vs round 2 (honest slices)

| Metric | Round 1 overload (21:48–22:46) | Round 2 best full-throttle slice (2×, 22:47–23:49) | Round 2 whole window |
|---|---|---|---|
| Duration | 57.5 min | ~62 min | **7 h 44 min** |
| Net TPS median / peak | 6,735 / 9,274 | **7,687 / 10,846** | 1,548 / **12,175** |
| Mempool max | **99,967** (at cap) | 87,615 | 88,429 |
| Crashes | n0 panic 21:12 (pre-overload end) | **none in window** | none in window |
| Fee | 1.2× | 2× (then later 10×) | mixed |
| Fee burn | ~2.4k TKAS/10 min | ~4.5k TKAS/10 min (2× full) | **~197.8k TKAS total** |
| Recovery drain | 49.6k→0 in ≤20 s (halt) | — | **N/A** (mp=7 at stop) |
| Stop reason | Planned halt for fund/fee change | — | **Disk taper** |

### What improved, what got worse, and what we learned

- **Full-throttle throughput still ~15% above round 1's overload** in the comparable 2× hour (median 7.7k vs 6.7k), with a higher peak (10.8k, later 12.2k at 10×).
- **No mempool-cap crash in round 2** — guards kept the pool under ~88k. The F1 bug is **not fixed**; we avoided provoking it.
- **Disk is the hard stop** on a single box with `--utxoindex`. The taper did its job: it slowed the storm instead of letting the node hit the 8 GB emergency STOP with a full mempool.
- **A "recovery" that starts at mempool=7 measures nothing.** Future runs should either force a high-mempool halt before recovery, or treat disk-taper wind-down as a separate "soft stop" path (no drain metrics).
- **Fee relative to spam still rules inclusion.** Early P4-max probes: min-fee users waited ~12 s median / 2+ min worst; 10× stayed ~1.2 s.
- **Outside senders confound network TPS.** Grok Build's parallel traffic is in `nettps.jsonl` but not in supervisor accepted/fee counters.

## Confounder: Grok Build agent

At **22:47:16** the overload worker sent **299,993 TKAS** in 1,184 txs (`logs/storm/grokbuild-fund-2.jsonl`, also noted in `ramp.log`) to another agent's TN10 receive wallet so that agent could keep building/sending. That traffic is **outside** our supervisor. Treat network TPS peaks as "ours + Grok Build + rest of TN10", and fee-burn / accepted TPS as "our storm workers only".

## Gaps closed vs the 23:29 interim README

| Interim gap | Final status |
|---|---|
| Recovery after funds/disk | Observed: **disk taper stop**, recovery N/A (mp=7) |
| Final fee / TPS / duration | ✔ recomputed for full window + regimes |
| Vprog under load | Moved to round3 repo (ran during 1k baseline + contention) |
| 10× fee / 1k baseline | Documented as regime changes inside round 2 |

## Files

- `findings/round2-final-stats.json` — machine-readable final numbers
- `findings/round1-overload-57min-summary.md` — round 1 overload summary
- `logs/storm/ramp.log` — full operator timeline through ROUND4
- `logs/storm/recovery-watch.out` — recovery-watch error (sampler path missing; run still logged END in ramp.log)
- `logs/tps12h/nettps-round2-full.jsonl` — every 10 s network TPS sample in the window
- `logs/excerpts/supervisor-round2-full-every12th.jsonl`, `monitor-round2-full-every3rd.jsonl`
- `logs/overload/analysis.json` — fee-tier probe analysis (P1 / P3 / P4-max)
- `tools/report-stats.py`, `tools/secret-scan.sh`

## Recompute

```
# from work dir /workspace/tn10-break-test-2026-09-25
python3 tools/report-stats.py logs 22:48 23:50   # same-day HH:MM only
# cross-midnight: use findings/round2-final-stats.json (this report) or a small script over nettps/supervisor/monitor
```

TESTNET ONLY. No seeds, private keys, or `/home/box/secure` material are in this repo.
