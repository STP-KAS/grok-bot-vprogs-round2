# TN10 round 2 — harder full throttle after the 30-min overload + 5-min break

> **Testnet-10 only. Private.** Round 1 (setup, findings F1–F7, vprogs, scripts): **https://github.com/STP-KAS/grok-bot-vprogs**
> (branch `tn10-break-report`). This repo only covers what changed in round 2 and compares it with round 1.
>
> **STATUS: SKELETON — round 2 is running. Numbers are filled in after ≥20–30 min at full throttle.**

## Round 2 plan (from the overload worker)
1. 21:48:41–22:18:41 CEST: phase 1, 30-min overload with fee-tier probes (end of round 1).
2. 22:18:41 → ~22:23:41: **5-min break** (`/tmp/tps-storm.limits` `RATE_MAX=0`) to measure recovery / mempool drain.
3. ~22:24 →: **back to full throttle, pushed harder, and it stays there** (no settle at 2,000 TPS). n0 only.

## Sections (to be filled)
- Setup changes vs round 1
- Timeline
- Peak / sustained TPS
- Mempool
- Crashes / restarts
- Fee tiers & dApp impact
- Disk
- Side-by-side comparison vs round 1
- Analysis: what improved, what regressed, why
