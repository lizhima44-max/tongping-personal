# TongPing queued work

This file is a lightweight entry point for work that has been designed but is **not yet authorized for implementation**.

## Social Browser patterns adoption

Design: `docs/superpowers/specs/2026-10-16-social-browser-patterns-adoption-design.md`

Reference baseline: `ceniran/ai-social-browser@6bc21a18425da4f0042fac1ef6b3e30aa5aec7f2`

Recommended queue:

- [ ] **B1 — Effect outcome core**
  - Add shared `confirmed / failed / uncertain / challenge` semantics.
  - Preserve approval as a separate pre-action concept.
  - Central invariant: `uncertain` must never trigger blind automatic retry.
  - Start with: `src/approval-store.ts`, `src/audit.ts`, `src/mcp-server.ts`.

- [ ] **A1 — Shared adapter contract + browse continuity**
  - Generalize the pattern already present in `src/adapters/xianyu.ts`.
  - Add per-profile/site short-lived list context and navigation metadata.
  - Prefer list-card continuity when available; keep generic browser fallback.

- [ ] **A2 — Xianyu pilot migration**
  - Move current Xianyu functions onto the shared adapter contract.
  - Prove search -> item continuity and stale/wrong-profile isolation.
  - No new product behavior until the shared contract is proven.

- [ ] **A3 — Xiaohongshu read-only adapter**
  - `feed / search / profile / read / watch`.
  - Add structured challenge detection and human handoff.
  - Keep writes disabled initially.

- [ ] **B2 — Semantic action budgets + first verified write flow**
  - Server-side `engage / publish` budgets below the model layer.
  - Verify side effects with network/UI/read-back evidence.
  - Propagate `uncertain` and `challenge` through MCP results.

- [ ] **A4 — X adapter**
  - Reads first: `feed / search / profile / read / watch`.
  - Writes only after B1/B2 and existing approval policy are wired through.

- [ ] **C1 — “帮我看完” fast video digest**
  - Explicit full-media mode, separate from no-spoiler companion mode.
  - Baseline: 8 timestamped timeline frames + timestamped ASR when available.
  - Reuse low-level media/audio plumbing without mutating companion coverage.
  - Enforce local-root allowlist, public-HTTPS validation, byte/duration caps, redirect revalidation, and cleanup.

- [ ] **Polish — doctor/selftest/docs**
  - Add deterministic self-tests for landed contracts.
  - Keep live third-party-account checks optional.
  - Optional later refinement: scene-aware frame selection for video digest.

## How to start one item

1. Read `AGENTS.md` and the design linked above.
2. Pick **one** queue item only.
3. Write its focused implementation plan under `docs/superpowers/plans/`.
4. Implement test-first.
5. Run focused tests, then `npm run check`.
6. Do one final independent review before PR/merge.

Do not implement the entire queue as one branch.
