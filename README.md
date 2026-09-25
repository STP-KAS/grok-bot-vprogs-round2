# TN10 round 2: a harder full-throttle run after the one-hour overload

> **Testnet-10 only. Private.** Round 1 covers the setup, findings F1–F7, vprogs and all scripts:
> **https://github.com/STP-KAS/grok-bot-vprogs** (default branch `tn10-break-report`). This repo covers only **round 2** and how it compares.
> Times are CEST. Figures marked ✔ were recomputed from the logs in `logs/` with `tools/report-stats.py`. **[unverified]** means an operator note with no log behind it.
> **Data cut-off: 23:29:34 CEST, 25 Sep 2026**, about 42 min into round 2 (the storm is still running). All numbers cover 22:48 → 23:29.

## In one paragraph
Round 2 pushed the same single node (`n0`, kaspad 2.1.0) harder. Every storm transaction now paid **2× the minimum fee** instead of 1.2×. The rate cap went up, and a new hard brake at 90k mempool kept the node away from its 100k cap. The network carried a **median of 7,732 TPS, up from 6,735 in round 1's overload** (10 s samples), with a **new peak of 10,846 TPS**. That came from **more blocks per second (10.1 vs ~9)**, not fuller blocks, which were already ~99% compute-mass full. The mempool ran lower (median 54k, max 88k vs 65k / 99.97k) and **the node never crashed**. The price: fee burn roughly doubled to **~4.5k TKAS per 10 min**. Ordinary users paying the minimum waited **longer** (p50 12 s, max 132 s) because the spam now outbid them by more. Disk use is now the limit that ends the run. At ~9 GB/h, the disk reaches its safety floor hours before the storm's funds run out.

## Setup changes vs round 1
| | Round 1 (up to 22:46:14) | Round 2 (from 22:47:26) |
|---|---|---|
| Nodes | n0 + n1 until 21:33, then n0 only | n0 only (kaspad 2.1.0, `--ram-scale=0.1`, `--async-threads=4`, no utxoindex) |
| Storm fee | 1.2× min = 120 sompi/g | **2× min = 200 sompi/g** (`FEE_MULT_P=2.0`, all 9 workers) |
| Rate cap | RMAX 30,000; then 60,000 (22:16); safety 8,000 (22:42) | 9,000 → **20,000** (22:48:55) |
| Mempool guards | taper 45–68k, gate 78k/55k | taper **40–60k**, gate **70k/50k**, new **hard brake at 90k** (PANIC) |
| Senders | 8 P2SH workers (4,000 tag wallets) + 0.5 TKAS lane (H) | same, 4,800 addresses total (not increased: blocks are the limit, not senders) |
| Funds | ~666k TKAS in storm pools | ~356k TKAS after 299,993 TKAS was moved to another agent's build wallet at 22:47:16 **[pool size: operator note]** |
| Probes | 6 fee tiers every 30 s | unchanged |

## Timeline (`logs/storm/ramp.log`, `monitor.jsonl`, `nettps.jsonl`)
| Time | Event |
|---|---|
| 21:48:41–22:46:14 | Round 1's overload (57.5 min). Summary: `findings/round1-overload-57min-summary.md` |
| 22:42:18 | Safety: RATE_MAX 60k → 8k after mempool hit 99.66k. The planned 23:00 break was cancelled |
| 22:46:11 | Fleet HALT (round 1 fees: 16,119 TKAS cumulative) |
| 22:46:09 → 22:46:29 | **Recovery: the mempool drained from 49,556 to 0 in ≤20 s** after the halt (10 s monitor samples). Idle probes were included in 0.5 s |
| 22:47:16 | 1,184 funding txs (299,993 TKAS) sent. All left the mempool by 22:47 |
| **22:47:26** | **Round 2 start (PHASE4-MAX)**: supervisor restarted, fee 2×, new guards. Net TPS reached 5,825 within 10 s and 9,130 within 20 s |
| 22:48:33 | A planned recovery pause was cancelled (0 s). Recovery will be measured when funds run out |
| 22:48:55 | RATE_MAX 9,000 → 20,000 |
| 22:58:27 | **Peak: 10,846 network TPS** (10 s window) |
| 23:29 | Data cut-off for this report. The storm continues until funds or disk run out |

The originally planned 5-min break (then 2 min at 23:00) **did not happen**. The only pause was the ~73 s halt at 22:46:14–22:47:26. The storm **did not settle at 2,000 TPS**. It stays at full throttle.

## Round 2 results (22:48 → 23:29, ✔ unless marked)
| Metric | Value |
|---|---|
| Network TPS (10 s samples, n=248) | median **7,732**, mean 7,660, min 4,375, **max 10,846** |
| Blocks per second | median **10.1** (max 12.9) |
| Txs per block | median 772 (max 853) |
| Supervisor accepted rate | median 5,872 tx/s, max 9,331 |
| Mempool (monitor) | median **53.7k**, max **87.6k**, min 27.3k. The 90k hard brake never fired **(inferred from max < 90k)** |
| Crashes / restarts | **none**. The node log's only mempool panic is the 21:12:50 one from round 1. n0 stayed synced in all 248 samples |
| RSS (kaspad) | 2.0–2.3 GB. RAM free ~5.4 GB. Load ~11.5 on 8 cores |
| Storm fee burn | 372 → 16,765 TKAS in 36 min (22:48:01 → 23:24:01) = **~4,550 TKAS / 10 min** (~27k TKAS/h) |
| Disk | n0 appdir 52.38 → 58.35 GB and free disk 46.45 → 40.47 GB in 40 min (22:48 → 23:28): **~9 GB/h on ONE node** ≈ ~325 B per tx |
| Disk runway | (40.5 − 8.4 GB floor) / 9 GB/h ≈ **3.5 h → around 03:00 on 26 Sep**. The disk taper will throttle the storm before the funds run out (operator projection for funds: ~11:00) |

### Fee tiers during round 2 (`logs/overload/analysis.json`, phase P4-max, 84 probes per tier, all included, 0 rejected, 0 evicted)
| Tier (× 100 sompi/g) | p50 | p90 | max | Round 1 overload p50 / max (117 probes) |
|---|---|---|---|---|
| 1× | **12.0 s** | 48.4 s | **131.5 s** | 7.0 s / 105 s |
| 1.2× | 8.9 s | 28.6 s | 92.8 s | 7.1 s / 92 s |
| 2× (= storm fee in round 2) | 5.0 s | 11.7 s | 28.6 s | 3.1 s / 48 s |
| 5× | 1.8 s | 2.9 s | 4.5 s | 1.4 s / 39 s |
| 10× | 1.2 s | 1.6 s | 2.4 s | 1.2 s / 3.7 s |
| 100× | 1.1 s | 1.6 s | 2.1 s | 1.2 s / 3.7 s |

**dApp impact in plain words:** when the spam pays 2×, anyone paying 2× or less joins the queue. The minimum-fee user's median wait
nearly doubled (7 → 12 s) and the worst case passed 2 minutes. Users at 5× got slightly slower but stayed under 5 s. At 10× nothing changed.
Priority tracks **fee relative to the spam**, so a dApp needs a fee that follows the market (e.g. the node's fee estimate plus headroom), not a fixed "high" fee.

## Side-by-side: round 2 vs round 1
| Metric | Round 1 | Round 2 | Change |
|---|---|---|---|
| Baseline (before any load) | 63.6 TPS on explorer, 1 h avg 267 **[unverified, explorer]** | n/a | — |
| First peak | >3,000 TPS explorer ~21:10 **[unverified]**, 2 nodes, 8 senders ~2,550 tx/s ✔ | — | — |
| n0-only full throttle, 21:36–22:00 | median 5,707, max 7,959 ✔ | — | — |
| Overload, 21:48–22:46 (57.5 min) | median 6,735, mean 6,456, max 9,274 ✔ | — | — |
| **Sustained network TPS** | **6,735** median (overload) | **7,732** median | **+15%** |
| **Peak (10 s)** | 9,274 | **10,846** | **+17%** |
| Blocks/s | ~8–10 (7.4 median 21:36–22:00) | **10.1** median | more blocks |
| Txs/block | ~800 (median 801, 21:36–22:00) | 772 | flat (blocks already ~99% compute-full) |
| Mempool | 50–96k band. Median 64.9k, **max 99,967** (at the cap) | median 53.7k, **max 87.6k** | lower, kept clear of the cap |
| Crashes | **n0 panic at 21:12:50 (100001 > 100000, F1)**. 58k lost at n1 restart (F2) | **none** | ✔ improved |
| Evictions | 30,298 low-fee txs evicted in the overload | not yet counted **[gap]** | — |
| Storm fee rate | 120 sompi/g, ~2.4k TKAS/10 min | 200 sompi/g, **~4.55k TKAS/10 min** | ×1.9 cost |
| Min-fee user p50 / max | 7.0 s / 105 s | **12.0 s / 131.5 s** | ✘ worse |
| 10× fee user p50 / max | 1.2 s / 3.7 s | 1.2 s / 2.4 s | unchanged |
| Disk growth | ~9 GB/h with 2 nodes at ~4k TPS. ~6 GB/h n0-only at ~5–6k | **~9 GB/h on 1 node** at ~7.7k | ✘ faster per node |
| Recovery | — | 49.6k → 0 mempool in ≤20 s after the halt | measured |

### What improved, what got worse, and why
- **Throughput up ~15% without fuller blocks.** Blocks were already ~99% full by compute mass in round 1. The gain comes from a higher
  block rate (10.1/s vs ~9/s). The most likely reason is our own miners: they found a large share of TN10 blocks (~65% in round 1's
  measured window), so the network block rate partly depends on our mining. This is **not a node or protocol improvement**. The honest
  reading is that capacity = block rate × ~500k compute mass, and a P2SH 1-in-1-out tx costs ~571–600 grams, so one node can carry ~8–10k TPS.
- **No crash despite the harder push.** The node sat far below the 100k cap because of the tighter guards (taper 40–60k, gate 70k/50k, hard brake 90k).
  The F1 bug is **not fixed**. We only stopped provoking it. Round 1 shows it fires when the pool reaches 100k.
- **Higher spam fee hurt ordinary users.** The storm at 2× outbids anything paying ≤2×, so these users wait longer. Fee protection is relative.
- **Cost doubled.** It rose from 120 to 200 sompi/g at ~20% more TPS.
- **Disk is now the binding limit.** ~325 B/tx at ~7.7k TPS on one node is ~9 GB/h. This run hits its disk floor in hours, not the 12 h planned (F4 in round 1).
- **Recovery is fast.** Once the spam stops, a ~50k backlog clears in about 20 s, because blocks keep pace at ~8–10k TPS.

## Monitoring the running round 2 / stopping it
```
tail -1 logs/tps12h/nettps.jsonl | jq '{t,net_tps,tpb}'
tail -1 logs/tps12h/supervisor.jsonl | jq '{t,accepted_tps,fees_tkas,disk_free_gb,mp:.nodes.n0.mempool}'
tail -3 logs/storm/ramp.log
```
(paths are relative to the work dir `/workspace/tn10-break-test-2026-09-25`)
- Change rate/guards: edit `/tmp/tps-storm.limits` (`LOW HIGH PANIC RATE_MAX`, re-read every 3 s). `RATE_MAX` 0 pauses sending.
- Stop everything cleanly: `touch /tmp/tps-storm.HALT`. Probes: `touch /tmp/overload-probe.STOP`.
- Recompute any window: `python3 tools/report-stats.py <workdir>/logs 22:48 23:59`.

## Gaps / to finalize
- The eviction count for round 2 was not extracted.
- The clean recovery measurement after funds run out is still pending. So is the final `overload-final-summary.md` from the overload worker.
- The block-rate increase is attributed to our miners by inference. Our miner share was not re-measured for round 2.
- Vprog moves under load are still skipped. They need a `--utxoindex` node (round 1, F7).

## Files
- `logs/storm/ramp.log`: full operator timeline (rounds 1+2)
- `logs/tps12h/nettps-round2.jsonl`: network TPS every 10 s from 22:47
- `logs/excerpts/supervisor-round2-every6th.jsonl`, `logs/excerpts/monitor-round2-every3rd.jsonl`: thinned samples
- `logs/overload/analysis.json`: fee-tier probe analysis for all phases (P1 overload, P3 drain, P4 max)
- `findings/round1-overload-57min-summary.md`: round 1 overload summary (by the overload worker)
- `tools/report-stats.py`, `tools/secret-scan.sh`
