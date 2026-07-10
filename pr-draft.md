# PR DRAFT — do not commit (untracked working note)

These are ready-to-paste drafts for opening the PR from this fork.
Nothing here is part of the `fixtures/measured.json` data contribution.

---

## Suggested PR title

`measurement: M5 Max 128GB oMLX — Qwen3-Coder-30B-A3B 8-bit (30.4 GiB) + Qwen3.6-27B 4-bit Gated DeltaNet (15.33 GiB)`

---

## PR body

Following up on #2 with the predicted-vs-measured data from the oMLX / M-series stack. Two real measurements from an **M5 Max 128GB** running **oMLX 0.4.5.dev1 (MLX 0.31.2)**. Both resident-byte figures come straight from oMLX's own per-engine accounting (the `actual_size` field it exposes for each loaded engine — the same metric #2 read via the fork's `/engine/loaded`).

### 1. Qwen3-Coder-30B-A3B-Instruct (MLX 8-bit) — `30.4 GiB`
- **How measured**: oMLX `/engine/loaded` actual resident bytes (the #2 table). oMLX estimates the full 8-bit MoE weights at 31.72 GiB; the 30.4 GiB resident reflects dFlash SSD expert offload (only active experts held in RAM).
- `ctx 32768` / `kvBits 16` are oMLX defaults; the figure is weight-residency-dominated and effectively ctx-independent.
- Figure taken from #2 (closed — the `parseHfConfig` bit-width fix landed) — not re-measured this session because the model wasn't resident at the time.

### 2. Qwen3.6-27B-4bit (Gated DeltaNet) — `15.33 GiB`  ← the linear-attention datapoint asked for in #2
- **How measured (live this session)**: `GET /admin/api/models` → `Qwen3.6-27B-4bit.actual_size = 16,457,805,464 bytes = 15.33 GiB` (oMLX's own formatted label). This is the per-engine resident-bytes field — identical to what `/engine/loaded` reports.
- Architecture: `qwen3_5`, hybrid 16 full-attention + 48 Gated-DeltaNet (linear) layers → the recurrent state is ctx-independent, so resident memory does **not** scale with context length. A useful calibration point for the linear-attention path.
- `ctx 32768` = oMLX global default `max_context_window` (per-model `null` → inherits default). `kvBits 16` from the loaded engine's settings (`turboquant_kv_enabled=false` → unquantized fp16 KV).
- Idle measurement (no active generation) — treat as a floor, not peak-during-gen.

**Units**: both figures are GiB, matching #2's convention and oMLX's own labels. Decimal-GB equivalents: 30.4 GiB ≈ 32.64 GB; 15.33 GiB ≈ 16.46 GB. Happy to switch the field to decimal GB — raw bytes are in each entry's `notes`.

**On the 4-bit variant**: I measured the 4-bit Qwen3.6-27B because it was the one resident at the time. The oQ8-mtp (8-bit) variant is registered but wasn't loaded, so I reported no number for it rather than guess one.

The `source` field on entry 2 points at this PR; repoint to the upstream PR URL if you'd prefer.

### Tests
`node --test test/*.test.mjs` → **8/8 pass** · `node vectors/run.mjs` → **14/14 pass**.
Heads-up (pre-existing, unrelated to this data PR): the `npm test` wrapper `node --test test/` errors under Node ≥23 because Node now resolves a bare directory argument as a module. `node --test test/*.test.mjs` (or `"test/**/*.test.mjs"`) works.

Thanks for the engine — the per-layer-type modeling is exactly right.

---

## Thank-you comment (post on the PR, or as a comment on #2)

Thanks for building this — the architecture-aware memory math (MLA / sliding-window / hybrid-linear / MoE residency) is the most accurate I've used for Apple Silicon + Qwen, and the public predicted-vs-measured ledger is a genuinely good idea. This PR follows up on #2 with the Qwen3.6 Gated-DeltaNet datapoint I mentioned there. Happy to keep feeding systematic predicted-vs-actual rows from the oMLX stack if it's useful.
