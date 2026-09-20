---
title: Shared-store identity contract for a harness port
date: 2026-09-20
category: conventions
module: openviking-extension
problem_type: convention
component: tooling
severity: high
applies_when:
  - Porting a harness extension onto a shared OpenViking home
  - Auditing whether a harness writes memories under its own label
  - Regenerating vendored shared files with npm run sync
  - Reviewing tests that assert file headers or config paths
  - Registering one extension file for more than one harness
symptoms:
  - Session ids carry another harness prefix
  - Two harnesses registered against the same extension file
  - One shared pending retry queue drained by the wrong harness
  - npm run sync silently reverts a local patch
  - Runtime and tests load different config files
tags: [harness-identity, shared-store, openviking, port-parity, session-id, pending-queue, config-drift, vendored-sync]
---

# Shared-store identity contract for a harness port

## Context

The operator runs four harnesses — Claude Code, Codex, pi, and Oh My Pi — against one self-hosted OpenViking server and one shared `~/.openviking` home. `omp-openviking-memory` is the fourth harness's entry point: a port of the official pi extension (`extensions/openviking.ts:1-9`) whose whole job is to make memories written in OMP land where the other three can read them, and memories the other three wrote show up in OMP.

The friction is that "shared store" is not a property of the server. It is a set of client-side choices, and each harness port makes them independently. Where credentials come from, what a session is named, which peer id is sent, where failed payloads are parked, which file holds behavior config, whether a local patch survives regeneration — every one of those is decided in this tree, in code the other harnesses do not share. Any single choice diverging produces two stores that look shared from the outside: health checks pass, sessions commit, tools return results, while the read-through the operator actually wants silently fails.

The pressure to keep re-litigating these choices is structural. This repo is a public fork of an upstream OMP port, it vendors runtime files from upstream with a `// GENERATED FROM ... DO NOT EDIT.` header, and it carries a `npm run sync` path meant to keep those files "drift-free" (`scripts/sync.mjs:2`). Every upstream sync is an opportunity to re-decide identity by accident.

Method note: a two-model oracle panel reviewed this tree and produced candidate findings; every load-bearing claim was then re-read at source, and the findings below are the ones that survived. Nothing was changed in this session — this is the contract plus the verified places the tree breaks it, not a fix report.

Harness-level configuration is a second, quieter way the sharing breaks (session history). The omp harness keeps its own state root separate from pi's, and its harness config is unmanaged hand-state rather than something a managed repository re-seeds per machine — so any per-harness setting that points at or identifies the shared store is per-machine drift. In omp, a whole-provider disable removes that provider *before* capability scanning, so its surfaces disappear silently: no error, no warning, just absent tools. If the memory integration is delivered to a harness as a provider or plugin surface, one local config line is enough to make a shared store stop being shared. Treat this paragraph as inference from one same-day session, not as a verified mechanism in this tree.

## Guidance

**Rule 1 — One home. Every harness resolves credentials and state from `~/.openviking`; no harness-private home.**
Mechanism: `shared/credentials.mjs:7-8` defaults `ovcli.conf` and `ov.conf` to `~/.openviking/ovcli.conf` and `~/.openviking/ov.conf`; the MCP fallback resolves the same two paths (`servers/config.mjs:32-33`, consumed at `:90-91`). Env overrides exist and are prioritized above the files (resolution order documented at `servers/config.mjs:10-14`), so the contract holds only while `OPENVIKING_URL`, `OPENVIKING_CONFIG_FILE`, and their siblings are unset for every harness.
Observable: `/viking status` prints the server-reported identity from `systemStatus()` — `user=` and `account=` (`extensions/openviking.ts:205-212`). Two harnesses showing different values are not sharing a home, whatever the file paths look like.

**Rule 2 — One peer identity per worktree, derived from the working directory, never from the harness.**
Mechanism: `src/config.ts:179` overwrites the credential-derived peer id with `resolveEffectivePeerId({ cfg, cwd: process.cwd() }).peerId`; `shared/workspace-peer.mjs:6-16` returns an explicitly configured peer first, otherwise sanitizes cwd, otherwise nothing. The value travels as `X-OpenViking-Actor-Peer` (`src/client.ts:115-116`).
Observable: two harnesses in the same worktree resolve the same peer id; two worktrees of the same project resolve different ones. Consequence for the rest of the contract: because the peer is the *worktree's* identity, nothing at the peer level says which harness wrote a memory — the harness label has to come from the session id (Rule 3). This is why the operator's one-harness-per-worktree rule is load-bearing (`STRATEGY.md:24`), and why the resist-a-change test names the harness label (`STRATEGY.md:29`).

**Rule 3 — A harness-distinct session id: the prefix must name the harness that wrote the memory.**
Mechanism: `deriveHarnessSessionId(prefix, sessionId, suffix)` rejects an empty prefix and returns `${prefix}${sessionId}` (`shared/session-model.mjs:42-50`). The prefix argument is the only harness label in the stored id; the id itself is what the server stores, commits, and archives.
Observable: the session id shown by `/viking status` (`extensions/openviking.ts:205-212`) and the archive overview fetched for it at session start (`extensions/openviking.ts:251-253`).
State of this tree: `src/sync.ts:56` passes `"pi-"`, so OMP sessions are written into the shared store under pi's namespace, while the same process sends the wire label `openviking-memory-omp/<version>` built at `src/config.ts:119` through `shared/credentials.mjs:37-39`. `README.md:94` documents the lifecycle step as `ensureSession(ov-…)`, a third name. The contract's required shape, not applied here, is `deriveHarnessSessionId("omp-", ompSessionId)`.

**Rule 4 — One memory extension per worktree, and the manifest must not hand the entry point to a second harness.**
Mechanism: `package.json:28-37` declares the same `./extensions/openviking.ts` under both the `omp` and the `pi` key; the extension registers its hooks unconditionally against whatever harness API it receives (`extensions/openviking.ts:107-186`), with capture at `extensions/openviking.ts:154-161`.
Observable, contingent on pi's own extension also being loaded — it is not vendored in this repo, so its handler cannot be checked here: this plugin registers one capture path per loaded session (`extensions/openviking.ts:154-161`), so a session loading both extensions runs two `turn_end` handlers and two sync managers, and the shared home receives the same turns twice, under two sessions that are both labeled `pi-`. The manifest's duplicate is deliberate compat, pinned by `tests/plugin.test.mjs:22-27` ("pi.extensions missing for compat").

**Rule 5 — Harness-local state inside the shared home must be namespaced or explicitly overridable.**
Mechanism, done right: the recall ledger lives at `~/.openviking/omp-recall-ledger`, honors `OPENVIKING_RECALL_LEDGER_DIR`, and is documented as harness-local state beside shared credentials (`shared/recall-ledger.mjs:19-21`, `shared/recall-ledger.mjs:36-39`).
Counterexample in the same tree: the pending queue is one global directory, `~/.openviking/pending`, with `OPENVIKING_PENDING_DIR` as the only escape (`shared/pending-queue.mjs:33`, `shared/pending-queue.mjs:56-58`). Entries record `type, sessionId, payload, createdAt, retries, dedupKey` and no harness tag (`shared/pending-queue.mjs:193-200`). `replayPending` lists the whole directory and posts every listed entry — bounded by the replay limit and skipping entries past the retry cap (`shared/pending-queue.mjs:362`, `:372-380`) — against whichever client called it (`shared/pending-queue.mjs:355-356`, `:371`, `:392-402`). OMP drains it during session start (`src/sync.ts:61-69`).
Observable: after a dirty shutdown in harness A, harness B's session start empties the directory — entries leave without the writing harness doing anything, and nothing in the removed entry says who wrote it. Replay with a file-hash-and-delete protocol is also how this is safe within one machine; the per-harness attribution is what is missing.

**Rule 6 — A local patch to vendored code is a contract item; the regeneration path and the test gate must both know it exists.**
Mechanism: `scripts/sync.mjs:13` lists 13 files regenerated as `HEADER + upstream body` (`scripts/sync.mjs:15-18`), written into `shared/` and `servers/shared/` at `scripts/sync.mjs:27-30`.
Breakage: `shared/debug-log.mjs:1` and `servers/shared/debug-log.mjs:1` both open with `// Patched: was GENERATED FROM examples/memory-plugin-shared/lib — perms fixed to 0o700/0o600`, over a body that creates the log directory with `mode: 0o700`, chmods it to `0o700`, and appends with `mode: 0o600` (`shared/debug-log.mjs:12-22`). `debug-log.mjs` is in the regeneration list, so a sync rewrites that file from upstream body text: the note is destroyed, and any body difference dies with it. The tests that guard headers match only `/GENERATED FROM/` (`tests/sync.test.mjs:9-14`, `tests/v2.test.mjs:47-50`), which the patched first line still contains, so the revert is invisible to CI. CI additionally has no parity or drift gate — it runs `npm ci`, `npm test`, `npm run typecheck`, `npm pack --dry-run` (`.github/workflows/ci.yml:16-19`).
Related drift, same mechanism: 5 of the 18 vendored `shared/` files are absent from the regeneration list (`scripts/sync.mjs:13`) — `agent-hook-runtime.mjs`, `agent-uri-guard.mjs`, `async-writer.mjs`, `batch-send.mjs`, `recall-ledger.mjs` — and all three headered `lib/` files are never copied at all, even though `AGENTS.md:16` states the header means "edit upstream and regenerate with `npm run sync`". So `shared/` is at once patched-and-revertible and forked-and-undetectable.

**Rule 7 — The config the tests read must be the config the runtime loads.**
Mechanism: `extensions/openviking.ts:26` loads through `loadConfigFromModuleUrl(import.meta.url)`, which resolves the module's own directory and then `join(extensionDir, "config.json")` — that is `extensions/config.json` (`src/config.ts:96-101`).
Breakage: the tests read the repo-root copy instead (`tests/plugin.test.mjs:29-33`, `tests/sync.test.mjs:16-17`). Both files ship (`package.json:55-62` includes `config.json`, and the `extensions/` directory wholesale) and they are byte-identical today — `diff config.json extensions/config.json` reported no differences when this was checked — so the split is latent rather than currently wrong. It becomes real on the first edit to one side only: edit the root copy and CI stays green for every field except the three the tests do assert (`takeover.enabled`, `recallTokenBudget`, `scoreThreshold` — `tests/plugin.test.mjs:31-33`, `tests/sync.test.mjs:18`) while runtime behavior does not change; edit `extensions/config.json` and the shipped behavior changes with no test asserting the value. The values at stake gate capture, takeover, and token budgets.

**Verification — the cross-harness read-through check.**
This is the test to run after any change to home resolution, peer derivation, session ids, extension registration, or the pending/ledger directories. It is deliberately end-to-end: it exercises the identity path rather than asserting on it.

1. In harness A, in worktree W, capture a fact carrying a unique token (a `viking_remember`, or a plain session turn with sync enabled).
2. Confirm the writing session id names harness A — `/viking status` prints it (`extensions/openviking.ts:205-212`). A `pi-` prefix from an OMP session fails here.
3. In harness B, start a session against the same home and search the token with `viking_search`.
4. Confirm the search left `peer_scope` off. The client adds `peer_scope: "actor"` only when `recallPeerScope === "actor"` (`shared/recall-core.mjs:83`, `shared/recall-core.mjs:109`), and the default is `"all"` with normalization enforcing at most that (`src/config.ts:61`, `src/config.ts:175`) — so `"actor"` in this tree would be an explicit narrowing, not a default.

Interpretation:
- Token surfaces in B, with the writer's session id naming A → contract satisfied end to end.
- Token surfaces with an `actor`-narrowed search → home and identity may be fine; peer scoping was narrowed. Re-run step 4 with the default before concluding.
- Token found only in A → narrow in this order: home (Rule 1), peer (Rule 2), session id (Rule 3).
- Token absent while `/viking status` reports healthy → the write went somewhere else; compare `user=`/`account=` between the two harnesses (`extensions/openviking.ts:205-212`).
- Token present twice under two session ids → duplicate registration (Rule 4).
- Because peer identity is cwd-derived (Rule 2), run the check once per worktree pairing; a pass in one worktree says nothing about another.

## Why This Matters

**Rule 3 is where the store stops being able to answer questions about itself.** With `src/sync.ts:56` writing `pi-`-prefixed sessions, the operator's four-harness store contains no way to distinguish an OMP memory from a pi memory. Attribution is not cosmetic here: the archive overview OMP rehydrates from at session start is fetched by that session id (`extensions/openviking.ts:251-253`), so OMP reads back its own history as if it were pi's, and the store's session namespace no longer describes the population that wrote it. Every later question — "is OMP capture actually working?", "which harness is double-writing?", "did the port regress?" — has no answering evidence in the data. `STRATEGY.md:29` names this as a first-class resist condition: the shared identity includes "a harness label saying who wrote a memory". Note the shape of the failure: the wire-facing label (`omp`, `src/config.ts:119`), the store-facing label (`pi-`, `src/sync.ts:56`), and the documented label (`ov-`, `README.md:94`) all disagree while every health check passes.

**Rule 4 turns a compat convenience into duplicated memory.** The duplicate manifest entry means a session that loads both extensions fires two capture paths (`extensions/openviking.ts:154-161` and pi's own counterpart, which is not vendored here), writing the same turns twice into the shared home under two sessions that both claim to be pi. Duplication in an append-only memory store is not self-healing: both copies get embedded, retrieved, and deduped only within a session, so recall budgets and search results are spent on redundant records, and the read-through check's failure signals get muddied by near-duplicate hits.

**Rule 5's queue is safe only because of an assumption Rule 1 makes explicit.** `replayPending` posts each pending entry against the calling client's connection while iterating every entry in the directory (`shared/pending-queue.mjs:371`, `:392-402`), and the entry carries no harness tag to check (`:193-200`). Under one shared home where every harness resolves the same endpoint, that is correct behaviour — a healthy harness replays a sick one's backlog, which is the point of a shared queue. The failure mode appears the moment any harness is started with an `OPENVIKING_URL` or `OPENVIKING_API_KEY` override, which the credentials loader explicitly supports and prioritizes over the files (`servers/config.mjs:10-14`): that process's session start replays another harness's transcript to a different server or account. With no tag on the entry, nothing afterwards can tell you it happened, and the entry is already deleted.

**Rule 6 makes an intentional hardening silently reversible.** Appending hook debug output without restricting the mode leaves a file that contains session and stage data readable to every local user; `shared/debug-log.mjs:12-22` restricts both the directory (`0o700`) and the appended file (`0o600`), and its first line records that this differs from upstream on purpose. `npm run sync` rewrites exactly that file as `HEADER + upstream body` (`scripts/sync.mjs:13`, `:15-18`), so the next regeneration restores upstream behaviour and deletes the note that a patch ever existed — and the header test still matches, so CI stays green (`tests/sync.test.mjs:12`). The same regeneration path never touches the 5 unlisted `shared/` files or the 3 `lib/` files that `AGENTS.md:16` claims are header-governed, so local edits there are permanent and invisible in both directions. The practical consequence is that "vendored" in this repo does not currently mean one checkable thing, and a reviewer cannot tell from the header text alone whether the file matches upstream.

**Rule 7 is quiet until it is not.** A behavior-config split where tests read the root copy and the runtime reads `extensions/config.json` produces both error directions with no CI signal: a change that ships without verification, or a fix that CI validates and the runtime never sees. It matters more than a normal config-path bug because the values at stake control capture, takeover, and token budgets — the settings whose silent misapplication look exactly like "memory is flaky".

## When to Apply

Apply the whole contract when any of these is true:

- **Porting or forking a harness extension**, or re-syncing with upstream. Rules 1, 3, 4 decide the identity the port will carry forever afterwards; the choices are cheap now and expensive once memories exist under the old labels.
- **Changing anything that decides identity or location**: session id derivation, peer derivation, credential/home resolution, extension registration, pending or ledger directories. Run the read-through check, not just the unit tests.
- **Regenerating vendored code** (`npm run sync`) or editing any file whose first line contains `Patched:` or `GENERATED FROM`. Rule 6 — expect the patch note to be destroyed and the body replaced.
- **Changing behavior config, or a test that asserts on it.** Rule 7 — confirm the edited file is the one the runtime loads.
- **Adding any new harness-local state under `~/.openviking`.** Rule 5 — name it for the harness, or give it an override env var, or both.
- **Reviewing a change that touches capture, recall, or session naming** in a repo that shares a memory home with other harnesses.

Apply it as diagnosis when one harness's runs become invisible to another. Narrow in this order: home (Rule 1, compare server-reported `user=`/`account=`), then peer (Rule 2, compare worktrees), then session id (Rule 3, read the prefix), then duplicate registration (Rule 4, check how many capture paths fired).

Do not apply it to OMP-internal behavior that never crosses the shared boundary — tool registration, UI notifications, and hook wiring inside one process are governed by the parity requirement in `STRATEGY.md:14`, not by this contract.

## Examples

**A. Session identity — the harness label lives in one argument.**
`shared/session-model.mjs:42-50` builds the stored id from the prefix:
```js
export function deriveHarnessSessionId(prefix, sessionId, suffix = "") {
  if (!prefix || typeof prefix !== "string") {
    throw new Error("deriveHarnessSessionId requires a non-empty prefix");
  }
  ...
  const base = `${prefix}${sessionId}`;
```
The caller chooses that label. In this tree (`src/sync.ts:56`):
```ts
const id = deriveHarnessSessionId("pi-", piSessionId);
```
Required shape for the contract, not applied:
```ts
const id = deriveHarnessSessionId("omp-", ompSessionId);
```
The same process already identifies as OMP on the wire — `userAgent: buildUserAgent("omp", EXTENSION_VERSION)` (`src/config.ts:119`), which yields `openviking-memory-omp/<version>` (`shared/credentials.mjs:37-39`) — while `README.md:94` documents the session as `ov-…`. Three labels, one session.

**B. Harness-local state — the ledger shows the pattern, the queue shows the gap.**
Namespaced and overridable (`shared/recall-ledger.mjs:36-39`):
```js
return process.env.OPENVIKING_RECALL_LEDGER_DIR
  || join(homedir(), ".openviking", "omp-recall-ledger");
```
Shared namespace, single escape hatch (`shared/pending-queue.mjs:33`, `:56-58`):
```js
const DEFAULT_PENDING_DIR = () => join(homedir(), ".openviking", "pending");
...
return process.env.OPENVIKING_PENDING_DIR || DEFAULT_PENDING_DIR();
```
And the entry that gets written into it carries no harness field (`shared/pending-queue.mjs:193-200`):
```js
const entry = {
  type,
  sessionId,
  payload,
  createdAt: now,
  retries: 0,
  dedupKey,
};
```
so `replayPending` can only assume the assumptions of the calling process (`shared/pending-queue.mjs:392-402` posts `entry.sessionId` and `entry.payload` via the caller's `fetchJSON`).

**C. Regeneration versus a recorded patch — three lines that cannot both be true.**
`shared/debug-log.mjs:1` and `servers/shared/debug-log.mjs:1`:
```js
// Patched: was GENERATED FROM examples/memory-plugin-shared/lib — perms fixed to 0o700/0o600
```
`scripts/sync.mjs:12-18`:
```js
const HEADER = "// GENERATED FROM examples/memory-plugin-shared/lib. DO NOT EDIT.\n";
const HARNESS = ["credentials.mjs", ..., "debug-log.mjs", ...];
async function copyFile(file, targetDir) {
  ...
  await writeFile(join(targetDir, file), HEADER + body, "utf-8");
```
`debug-log.mjs` is in `HARNESS`, so the next sync writes the pristine header plus upstream body over the patched file. The guard that should notice matches only the substring both versions contain (`tests/sync.test.mjs:12`):
```js
assert.match(s, /GENERATED FROM/, `${p} missing GENERATED header`);
```

**D. Behavior config — one loader, two readers.**
Runtime path (`extensions/openviking.ts:26` → `src/config.ts:96-101`):
```ts
const config = loadConfigFromModuleUrl(import.meta.url);
...
return loadConfig(dirname(fileURLToPath(moduleUrl)));
...
const configPath = join(extensionDir, "config.json");
```
resolves to `<repo>/extensions/config.json`. Tests read the root copy (`tests/sync.test.mjs:17`):
```js
const extCfg = JSON.parse(readFileSync(join(ROOT, "config.json"), "utf-8"));
```
Today `diff config.json extensions/config.json` is empty, so this is a latent split — but nothing in CI compares them (`.github/workflows/ci.yml:16-19`), so the next single-sided edit is undetectable. The contract-level answer is that the test should assert through the loader the extension actually uses, or the diff should be a gate; either way, one of the two files has to become the only one a test reads.

**E. Registration — one entry point, two harnesses.**
`package.json:28-37`:
```json
"omp": { "extensions": [ "./extensions/openviking.ts" ] },
"pi":  { "extensions": [ "./extensions/openviking.ts" ] },
```
with the intent pinned in `tests/plugin.test.mjs:26`:
```js
assert.ok(Array.isArray(j.pi?.extensions), "pi.extensions missing for compat");
```
The extension it points at registers capture on every turn of whatever session it is loaded into (`extensions/openviking.ts:154-161`), and its session ids all carry the `pi-` prefix from `src/sync.ts:56` — so a pi session loading both extensions writes the same turns twice into the shared home under two identically-prefixed sessions.

## Related

- `STRATEGY.md:14` — states the same contract as positioning (one shared home, one peer identity per worktree, read-through across harnesses); this document is its verification evidence.
- `STRATEGY.md:26` — the boundary "patches are recorded and survive `npm run sync`" is the intent Rule 6 shows the current regeneration path does not satisfy.
- `README.md:94` and `README.md:35` — both describe behavior this contract's findings contradict (a documented `ov-` session prefix against the shipped `pi-` one; a pending-queue replay described without its global scoping).
- `AGENTS.md:16` — the vendored-file rule whose stated regeneration coverage (`shared/`, `lib/`, `servers/shared/`) is broader than `scripts/sync.mjs:13` implements.
