# Agent SDK documentation — design

**Date:** 2026-08-05
**Repo:** `/Users/abhinavkumar/docs` (Argide Mintlify site)
**Feature being documented:** `@argide/agent` — the in-page agent runtime SDK (source of truth: `Ourguide-B2B/packages/agent/README.md` plus the shop-clone reference integration).

## Goal

Public integration docs for the `@argide/agent` browser SDK, structured per Diátaxis (quickstart → guides → reference) the way Sentry/PostHog/Clerk document their browser SDKs. A developer reading only the quickstart should reach a working first task in ~15 minutes.

## Audience

External developers at Argide customers who embed the SDK in their product's web app and own a backend that can hold an `ak_live_…` API key. Assumed to know JS/TS and their own framework; assumed to know nothing about Argide agent sessions.

## Navigation change (`docs.json`)

Add one group to the existing Documentation tab, placed directly after the "Agent" group:

```json
{
  "group": "Agent SDK",
  "pages": [
    "agent-sdk/overview",
    "agent-sdk/quickstart",
    "agent-sdk/claiming-sessions",
    "agent-sdk/running-tasks",
    "agent-sdk/api-reference"
  ]
}
```

The existing "Agent" group (widget agent behavior) is untouched. No rename in this change; if "Agent" vs "Agent SDK" proves confusing we rename later in a separate change.

## Pages

All pages live in a new `agent-sdk/` folder, MDX with `title` + `description` frontmatter, matching house style (`<Note>`, `<Warning>`, `<Tip>`, `<CodeGroup>`, `<Card>`/`<CardGroup>` cross-links, mermaid for sequence diagrams). Frontmatter titles in Title Case matching existing pages, kept short since the sidebar shows them under the "Agent SDK" group label: **Overview**, **Quickstart**, **Claiming Sessions**, **Running Tasks**, **API Reference**. Section headings inside pages use sentence case per AGENTS.md.

### 1. `agent-sdk/overview.mdx` — "Overview"

Explanation page (the "why/what").

- What it is: a headless runtime loaded in the end user's tab; an Argide agent drives that tab (click, fill, navigate) to complete a goal your own chatbot delegates to it.
- When to use it vs the chat widget (widget = Argide-owned chat UI; SDK = your UI, Argide does the driving).
- Architecture: mermaid sequence diagram with four participants — end user's page (your app + SDK), your server, Argide backend, the agent. Shows: init → session minted `{sessionId, pairingCode}` → browser posts pair to your server → your server claims with `ak_live_…` key → `do(goal)` → agent drives tab → `check()` outcome.
- Security model in brief: the browser never holds the API key; a session must be claimed by your server before `do()` works; pairing codes are single-use. Link to Claiming sessions for depth.
- Integration requirements (from README): tools execute client-side; your stack must tolerate a reconnect replaying still-pending tool dispatches after a hard navigation. Link to Client-Side Tools page.
- Card links: Quickstart, Claiming sessions.

### 2. `agent-sdk/quickstart.mdx` — "Quickstart"

Tutorial page. Numbered steps, copy-paste code, one happy path (Next.js for the server step since that's the reference integration; the claim endpoint is plain HTTP so any backend works).

1. **Install** — `npm install @argide/agent` (CodeGroup: npm / pnpm / yarn).
2. **Initialize in the browser** — `ArgideAgent.init({ apiUrl, productId, claim })` with the README's claim-callback example. Notes: idempotent / StrictMode-safe; SSR no-op (warns, mints nothing until it runs in a browser).
3. **Add the claim endpoint on your server** — the missing half the README doesn't show. Next.js route handler adapted from the shop-clone reference: validate `{sessionId, pairingCode}`, forward to `POST {apiUrl}/api/agent/v1/sessions/claim` with `Authorization: Bearer <ak_live_… key>`, return the upstream status. Env vars: public API URL + secret `ARGIDE_AGENT_API_KEY`. `<Warning>`: the `ak_live_…` key must never ship to the browser.
4. **Run a task** — `const { taskId } = await ArgideAgent.do({ goal: "…" })`, then `check({ taskId })` for the outcome. Note that `do()` resolves when the task exists (not when it finishes) and waits internally for init + claim, so there is no readiness check.
5. **Watch it work (optional)** — 5-line `onActivity` example.
- Next-steps CardGroup: Claiming sessions, Running tasks, API reference.

### 3. `agent-sdk/claiming-sessions.mdx` — "Claiming Sessions"

How-to/explanation page for the session security model.

- Why claims exist: pairing a browser session to your server proves the page is yours; unclaimed sessions can't run tasks.
- Lifecycle: minted (fresh `{sessionId, pairingCode}`) → claimed (your server, one shot) → recovered (SDK re-attaches after reload; already claimed).
- Default path: the `claim` callback on `init` — called only for freshly minted sessions, never on recovery.
- Custom channel path: omit `claim`, read `await ArgideAgent.session()` → `{sessionId, pairingCode, recovered}` and forward the pair yourself. The rule, stated hard: **claim only when `recovered` is `false`**. Re-claiming a recovered session returns `409 ALREADY_CLAIMED`, which is reserved as the stolen-code signal — treat it as a security event, not a retry case.
- Server claim endpoint reference: method, URL (`POST {apiUrl}/api/agent/v1/sessions/claim`), auth header, request body, success/error responses.
- `<Note>` cross-link to the quickstart's ready-made route handler.

### 4. `agent-sdk/running-tasks.mdx` — "Running Tasks"

How-to page for the task lifecycle.

- Start: `do({ goal })` → `{ taskId }`; resolves on task creation; queues internally behind init/claim.
- One goal at a time per session: a *different* goal while one runs rejects with `409 SESSION_BUSY` — wait or `cancel()` first. Re-sending the *same* goal is a rejoin: resolves with the same `taskId`. `<Tip>`: this is what makes re-dispatching after a hard navigation safe.
- Outcome: `check({ taskId })` → `{taskId, status, summary}` — the authoritative result. Ending early: `cancel({ taskId })`.
- Error contract: `check()`/`cancel()` reject on any non-2xx — `404` (unknown task, or a task belonging to another session), `409` (cancelling an already-terminal task). Show a try/catch example.
- Observing: `onActivity` (ephemeral, per-tab, `{kind: "tool-start"|"tool-end"|"idle"}`) vs `onStateChange` init option (facade lifecycle `"starting"|"ready"|"error"|"stopped"`) vs `check()` (authoritative). Small table: which to use for what.
- Hard navigations: session recovers, pending tool dispatches replay, same-goal `do()` re-attaches.

### 5. `agent-sdk/api-reference.mdx` — "API Reference"

Lookup page. No tutorials, tables + signatures only.

- `ArgideAgent.init(options)` — options table: `apiUrl`, `productId`, `claim?`, `onStateChange?`; idempotency/SSR notes.
- `ArgideAgent.do({ goal })` / `check({ taskId })` / `cancel({ taskId })` / `onActivity(cb)` (returns unsubscribe) / `session()` — signature, params, return, rejection cases for each.
- Types: task outcome `{taskId, status, summary}`, activity event kinds, state values, session `{sessionId, pairingCode, recovered}`.
- Errors table: `SESSION_BUSY` (409), `ALREADY_CLAIMED` (409), task `404`, cancel-terminal `409` — when each occurs and what to do.
- Advanced: `startAgentRuntime({ apiUrl, productId })` — the primitive `ArgideAgent` wraps; returns `{sessionId, pairingCode, recovered, do, check, cancel, onActivity, stop}`; use for multiple products per page or self-managed lifecycle; at this level the claim-on-recovered rule and concurrent-start guarding are the caller's responsibility.

## Content rules

- Code samples in TS where the README's are TS; server example labeled "Next.js" inside a CodeGroup so other backends aren't implied unsupported.
- Facts come only from the README and the shop-clone reference integration — no invented endpoints, options, or limits. Anything unknown (rate limits, key provisioning UI, npm publish status) is left out rather than guessed.
- Placeholders follow existing docs convention (`YOUR_PRODUCT_ID` style).
- Follow AGENTS.md style: second person, active voice, short sentences, bold UI elements, code-format paths.

## Out of scope

- Renaming the existing "Agent" group.
- Dashboard docs for provisioning `ak_live_…` keys (not verifiable from available sources; quickstart says "from your dashboard" without inventing a path).
- React/Flutter wrapper docs (none exist for this SDK).
- Changes to the package README itself.

## Verification

- `mint broken-links` passes.
- `mint dev` renders all five pages and the nav group locally.
- Every code sample is syntactically valid and consistent with the README's API surface.
