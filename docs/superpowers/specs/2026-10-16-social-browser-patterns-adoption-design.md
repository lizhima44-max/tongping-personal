# Social Browser Patterns Adoption Design

> Status: **QUEUED ONLY**. This document records design decisions and implementation-ready work packages. It does **not** authorize implementation.
>
> Source baseline inspected on 2026-10-16: `ceniran/ai-social-browser` at commit `6bc21a18425da4f0042fac1ef6b3e30aa5aec7f2`.

## Intent

Absorb the strongest reliability and ergonomics patterns from `ai-social-browser` into TongPing without changing TongPing's product identity.

TongPing remains a **human–AI shared browser runtime**: the human, multiple AI clients, the live browser, co-watching, group chat, approvals, and takeover are first-class. Site-specific automation is an optional accelerator layered on top of the generic browser, never the only way to use the product.

Success means a future worker can open this document, choose one queued package, and know:

- why the package exists;
- which existing TongPing files it should build on;
- which new boundaries/interfaces should exist;
- which behaviors are explicitly required;
- which tests prove completion;
- what must not be coupled or regressed.

## Source provenance and copying rule

Reference project: <https://github.com/ceniran/ai-social-browser>

Useful source areas at the pinned baseline:

- `actions/README.md` — structured site actions, write verification semantics, challenge handling, rate groups;
- `actions/twitter-tool/app.py` — X-specific read/write flows and verification;
- `actions/xiaohongshu-tool/app.py` — Xiaohongshu read-only structured actions and challenge handling;
- `actions/social-browser-mcp/server.py` — unified social/video MCP and media-source policy;
- `relay/README.md` — session continuity and human takeover rationale.

The source repository is MIT licensed. Default implementation policy for TongPing is **clean reimplementation of ideas and contracts**, not verbatim code copying. If a future change copies a non-trivial source fragment, preserve the MIT copyright/license notice for that fragment in the repository.

## Product boundary

Do not turn TongPing into a dedicated social-media bot.

Keep these invariants:

1. Generic `browser_*` tools remain available and authoritative.
2. Human takeover remains a first-class authority boundary, not merely an error fallback.
3. Adapters never receive cookies, passwords, daemon tokens, or direct access to profile files.
4. An adapter uses the same browser/profile/session as the human and generic browser tools.
5. Site adapters may disappear or break without making the browser unusable.
6. Co-watching's "only what the human has seen" guarantee must remain isolated from any new full-video digest mode.
7. CAPTCHA/slider/challenge pages are never solved automatically; detection means stop and hand back to the human.

## Existing TongPing pieces to reuse

- `src/adapters/xianyu.ts` — current proof that a site-specific structured adapter can sit above `BrowserGateway`.
- `src/gateway.ts` — adapter boundary to browser actions.
- `src/browser-service.ts` / `src/browser-manager.ts` — generic browser primitives and profile lifecycle.
- `src/mcp-server.ts` — MCP registration and client-specific host behavior.
- `src/approval-store.ts` — pre-action one-shot approval for risky clicks.
- `src/audit.ts` — privacy-preserving action audit.
- `src/action-limits.ts` — existing pattern for server-side limits/validation.
- `src/audio-vision.ts` and companion perception/buffer files — media perception pieces that can be reused without merging semantics.
- `scripts/selftest.mjs`, `scripts/doctor.mjs`, and `tests/` — product-level verification surface.

---

# Package A — Structured Site Adapter Layer

## Goal

Give agents compact, stable, low-token verbs for high-value sites while preserving generic-browser fallback.

Examples: `search`, `feed`, `profile`, `read`, `watch`, and selectively approved write actions.

## Architecture

Introduce a shared adapter contract above `BrowserGateway` rather than growing one-off functions directly inside `mcp-server.ts`.

Tentative files:

- `src/adapters/types.ts` — shared adapter result/meta types;
- `src/adapters/browse-context.ts` — short-lived per-profile browse continuity;
- existing `src/adapters/xianyu.ts` — migrate to the shared contract first;
- later site files such as `src/adapters/xiaohongshu.ts` and `src/adapters/x.ts`;
- `src/mcp-server.ts` — thin tool registration only.

### Shared result metadata

Every structured adapter result should be able to report:

- `site` — stable adapter id;
- `profile` — browser profile actually used;
- `url` — canonical/current URL;
- `navigation` — how the detail page was reached;
- `challenge` — detected challenge state when applicable;
- `observedAt` — timestamp of the browser observation.

Recommended navigation values:

- `list_card_click` — detail opened from a recently observed list/search/profile card;
- `direct_open` — explicit URL/navigation was required;
- `already_open` — current page already represented the requested entity;
- `card_click_failed` — a remembered list item could not be re-found; do not silently pretend continuity succeeded.

## Browse continuity

Borrow the useful idea that `feed/search/profile -> read` should preserve the real browsing path when possible.

`browse-context.ts` should remember, per `(profile, site)`, a short-lived list of entities most recently observed on a list-like page:

- stable entity id when available;
- canonical URL;
- browser element `ref` when still meaningful;
- source page URL;
- observed timestamp.

Default TTL: **5 minutes**. The context is an optimization only and must never contain credentials or page text beyond the minimal entity metadata needed to reopen the item.

When a caller immediately reads one of those entities:

1. verify the same site/profile context is still fresh;
2. attempt to locate/click the observed card through normal browser primitives;
3. return `navigation: list_card_click` on success;
4. if the remembered card cannot be found, return a clear continuity failure for flows whose semantics depend on that continuity;
5. allow an explicit caller choice to use `direct_open` when continuity is not required.

Do not add stealth/evasion behavior. The value here is stable navigation history and lower token usage, not hiding automation.

## Read/write separation

Adapters should expose read verbs independently from write verbs.

Default policy:

- read adapters can ship before writes;
- write tools are opt-in per adapter;
- high-risk generic policy/approval remains in force even when an adapter offers a semantic write verb;
- a site adapter cannot bypass `ApprovalStore` by clicking through a private lower-level channel.

### Recommended first sites

1. **Xianyu pilot** — migrate the existing adapter first because it already exists and exercises search/detail continuity.
2. **Xiaohongshu read-only** — `feed/search/profile/read/watch`; challenge detection; no write actions initially.
3. **X read-first** — `feed/search/profile/read/watch`; write verbs only after Package B is complete.

## Challenge detection

A site adapter may recognize known challenge/verification pages or widgets and normalize that as a structured result.

Requirements:

- detection immediately stops adapter write execution;
- generic read of the challenge page may remain available for explanation;
- the result tells the model to hand control to the human;
- no adapter attempts to solve slider/CAPTCHA/verification flows;
- challenge detection should be site-local code, not hard-coded into the generic browser core unless a truly generic signal exists.

## Token discipline

Structured adapters should return the smallest useful JSON representation rather than dumping full DOM/page text.

Keep generic `browser_read` for cases where rich page text is actually needed.

## Package A acceptance

A future implementation is complete when:

- Xianyu uses the shared adapter contract without losing current behavior;
- a remembered search result can be opened through list-card continuity and reports `list_card_click`;
- stale/mismatched contexts cannot be reused across profiles or sites;
- an adapter failure never removes access to the generic browser;
- challenge results stop writes and point to human takeover;
- MCP tools are thin wrappers over adapter functions, not a second copy of site logic;
- tests cover fresh context, stale context, wrong profile, card disappearance, explicit direct-open fallback, and challenge stop.

---

# Package B — Side-Effect Outcome Protocol

## Goal

Make any action with external side effects safe against accidental duplicate retries.

This is broader than social media. It should become a reusable TongPing contract for "we clicked/submitted something, but what actually happened?".

## Key decision

Approval and outcome are separate concepts:

- **Approval** answers: "was this risky action authorized before execution?"
- **Outcome** answers: "after execution, how certain are we about what happened?"

Do not merge them.

## Core states

Introduce a shared post-action result state:

- `confirmed` — evidence shows the intended side effect happened;
- `failed` — evidence shows it did not happen, and retry semantics can be evaluated normally;
- `uncertain` — the action may have happened but confirmation failed; **automatic retry is forbidden**;
- `challenge` — execution encountered a human-verification barrier; stop and hand over to the human.

Recommended retry metadata:

- `safe` — a retry is safe under the operation's contract;
- `unsafe` — do not retry automatically;
- `after_human` — only retry after explicit human resolution/inspection.

Tentative shared type location: `src/effect-outcome.ts`.

## Verification evidence

Adapters should verify side effects using the strongest cheap evidence available, for example:

1. browser/network response tied to the action;
2. UI state transition;
3. read-back of the created/updated entity;
4. stable id returned by the site and re-read for exact semantic confirmation.

No single evidence type is globally required. The adapter declares what it observed.

The common result should be able to carry small, non-sensitive evidence metadata such as:

- `method: "network" | "ui" | "readback" | "mixed"`;
- `entityId` when safe;
- `observedUrl`;
- `detail` with bounded, non-secret diagnostic fields.

Do not log full typed message bodies by default; preserve TongPing's current privacy posture.

## Uncertain means no blind retry

This is the central invariant.

If the model pressed "send" and then lost the confirmation signal, the result is **not** automatically `failed`.

`uncertain` must propagate all the way through the MCP result and instructions so clients do not immediately call the same action again.

A human can inspect the page and decide what to do next.

## Challenge outcome

`challenge` is not a generic network failure.

On challenge:

- stop site writes;
- do not consume another retry budget automatically;
- surface the active profile/site and human-takeover instruction;
- after the human resolves the page, the caller may deliberately retry.

## Server-side action budgets

Borrow the concept of enforcing reasonable site-action budgets below the model layer.

Tentative file: `src/site-action-budget.ts`.

Do not modify the generic browser into a hard-coded social rate limiter. Budgets belong to semantic adapter actions.

Recommended groups:

- `read` — usually unmetered or high ceiling;
- `engage` — likes/follows/reposts/etc.;
- `publish` — posts/replies/messages/submissions.

Each adapter defines conservative defaults. The budget check must live below MCP prompt compliance so a model cannot talk its way around it.

Persistence across daemon restart is desirable for publish/engage budgets but can be implemented after the in-memory contract is proven. Do not couple Package B's first usable version to a database.

## Audit integration

Extend `AuditEntry` with optional bounded outcome fields rather than recording private payloads.

Useful additions:

- `effectState`;
- `retryPolicy`;
- `evidenceMethod`;
- adapter/site id.

Continue logging typed text as character counts/metadata, not content, unless a future explicit product decision changes that.

## Package B acceptance

A future implementation is complete when:

- a shared outcome type is used by at least one real side-effecting flow;
- `uncertain` cannot be auto-normalized to `failed`;
- an explicit test proves the caller does not retry on `uncertain`;
- challenge results stop site writes and require human resolution;
- approval remains a precondition and cannot be bypassed;
- audit records outcome state without leaking the action's private text;
- adapter budgets are enforced below the MCP/tool prompt layer;
- tests cover confirmed, failed, uncertain, challenge, budget exhaustion, approval mismatch, and restart semantics if persistence is included.

---

# Package C — "帮我看完" Fast Video Digest

## Goal

Add a second media mode for the user intent: "I do not want to watch this entire video; inspect the whole thing and tell me what is in it."

This is deliberately different from TongPing co-watching.

- **陪我看 / companion:** perception is limited to the user's watched coverage; no spoilers.
- **帮我看完 / digest:** explicit opt-in to inspect the complete media; full-video coverage is expected.

The two modes must never share a ledger in a way that falsely marks unseen companion content as human-seen.

## Architecture

Create a standalone media-digest pipeline that can reuse low-level media/audio utilities without reusing companion authorization semantics.

Tentative files:

- `src/video-digest.ts` — orchestration and digest packet construction;
- `src/video-source-policy.ts` — allowed sources, redirects, size/duration policy;
- reuse selected helpers from `src/audio-vision.ts` where boundaries are clean;
- `src/mcp-server.ts` — one explicit digest tool/entrypoint;
- dedicated tests under `tests/video-digest*.test.ts`.

## First version output

The tool should return perception material, not an embedded second LLM summary engine.

Return:

- duration and media metadata;
- **8 timeline-spread frames** as the baseline visual sample;
- timestamp for every frame;
- ASR transcript segments when local ASR is available;
- coverage statement showing that the full source timeline was sampled;
- warnings for missing audio, unreadable frames, truncation, or limits.

The calling AI synthesizes the actual natural-language summary.

## Source types

First implementation should support only sources we can bound safely:

1. media from the currently controlled browser when a stable media URL/source can be obtained through existing browser/media plumbing;
2. explicitly allowed local files;
3. explicit public HTTPS media URLs.

Do not accept arbitrary filesystem paths by default.

For local files, require configured allowlisted roots.

For public HTTPS URLs:

- reject embedded credentials;
- resolve/reject loopback, private, link-local, and other non-public targets;
- revalidate every redirect target;
- enforce download byte and duration limits;
- clean all temporary files on success and failure.

Baseline limits should start conservative and be configurable. A reasonable first default is **100 MiB / 10 minutes**, matching the reference project's scale, but TongPing may tighten this during implementation if local performance tests justify it.

## Frame selection

Version 1: deterministic timeline anchors across the full duration, producing eight representative frames.

Do not block the first version on scene-detection intelligence.

A later refinement may add scene-change-aware replacement frames, but the contract should remain "bounded representative frames with explicit timestamps".

## ASR

Prefer TongPing's existing local-ASR path when available.

Requirements:

- video digest still works visually when ASR is unavailable;
- ASR failure becomes a warning, not whole-tool failure;
- transcript segments retain timestamps;
- audio/transcript from digest mode does not mutate companion coverage state.

## UI/MCP semantics

Use an explicit name and wording that makes full-media inspection obvious. Do not overload `browser_companion_context`.

The tool result/instructions should make it clear that this operation may reveal the end of the video.

## Package C acceptance

A future implementation is complete when:

- an explicit digest call samples the full media timeline and returns eight timestamped frames;
- transcript segments are attached when ASR is available;
- no-ASR mode still returns useful visual coverage;
- digest state does not change companion/human-seen coverage;
- local paths outside allowlisted roots are rejected;
- HTTPS private-network/redirect SSRF cases are rejected;
- byte/duration limits are enforced;
- temporary media/audio/frame artifacts are removed after both success and failure;
- tests cover short video, long/oversize rejection, no audio, ASR unavailable, malicious redirects, cleanup, and companion-ledger isolation.

---

# Cross-cutting work worth borrowing

These ideas are useful but should attach to the three packages above rather than become separate architecture projects.

## 1. Structured "stop, do not retry" hints

Tool results should state retry semantics machine-readably, not only in prose.

## 2. Site-local challenge classifiers

Keep known platform challenge signatures close to each adapter. Generic browser core should expose primitives, not a giant site fingerprint registry.

## 3. Same-session continuity

Preserve one real browser profile/session shared between human and AI. TongPing already does this; adapters must not introduce a second hidden automation browser.

## 4. Small semantic tools, generic browser underneath

Site tools should be ergonomic shortcuts, not a forked browser engine.

## 5. Self-test coverage

When a package lands, extend `npm run selftest` only for deterministic product contracts. Do not make selftest depend on a live social account or mutable third-party page unless guarded as optional diagnostics.

`npm run check` remains the required local developer gate.

# Explicit non-goals

- No multi-account farm, proxy pool, CAPTCHA solver, or anti-detection framework.
- No attempt to guarantee avoidance of platform automation controls.
- No credential extraction/export.
- No replacement of generic `browser_*` with site-only APIs.
- No automatic write enablement for every adapter.
- No full-video digest data leaking into companion "user has seen this" state.
- No database migration solely to implement the first version of these packages.

# Queue order

Recommended dependency order:

1. **B1: effect outcome core** — tiny shared contract first; no site writes required yet.
2. **A1: shared adapter contract + browse context**.
3. **A2: migrate Xianyu as the pilot**.
4. **A3: Xiaohongshu read-only adapter**.
5. **B2: adapter budgets + first verified write flow**.
6. **A4: X adapter, reads first; writes only through Package B**.
7. **C1: fast video digest**.
8. **Cross-cutting polish: doctor/selftest/docs and optional scene-aware frame refinement**.

This order is a recommendation, not a hard dependency for C1: the video digest can be implemented independently when that is more useful.

# "Open and start" checklist for a future worker

When Moonny chooses one queue item:

1. Read `AGENTS.md` and this design.
2. Re-check the pinned reference commit only for the pattern relevant to that queue item; do not blindly sync upstream behavior.
3. Inspect the listed TongPing files before inventing new infrastructure.
4. Create a focused implementation plan for **one package/sub-package only** under `docs/superpowers/plans/`.
5. Use TDD: failing test -> minimal implementation -> passing focused test -> `npm run check`.
6. Keep generic browser behavior working at every step.
7. Run one final independent review before PR/merge.

# Design self-review

- Product identity preserved: yes; adapters remain optional and the human remains first-class.
- Security boundary preserved: yes; no credentials enter adapters and challenges hand off to the human.
- Duplicate-action risk addressed: yes; `uncertain` explicitly forbids blind retry.
- Media semantics separated: yes; digest and companion coverage are distinct.
- Scope decomposed: yes; Packages A, B, and C can be planned/implemented independently.
- Source/license provenance captured: yes; source commit and clean-reimplementation rule are recorded.
- No placeholder decisions required to start the first package: yes; defaults and initial queue order are explicit.
