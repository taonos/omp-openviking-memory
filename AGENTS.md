# Repository Guidelines

OMP plugin that gives Oh My Pi agents long-term semantic memory through an OpenViking server: auto-recall before each prompt, capture after each turn, 11 `viking://` tools, slash `/ov`.

## Structure

- `extensions/openviking.ts` — the single OMP extension entry (agent hooks, `viking://` uri-guard).
- `src/` — TypeScript internals: `client.ts`, `config.ts`, `recall.ts`, `sync.ts`, `takeover.ts`, `tools.ts`.
- `lib/` — adapters binding vendored harness code to this plugin.
- `shared/` — vendored harness runtime; `servers/shared/` — vendored MCP-proxy cores.
- `servers/mcp-proxy.mjs` + `mcp.json` — MCP fallback (manual tool calls only, no auto-recall).
- `skills/openviking-memory/` and `commands/ov.md` — agent skill and slash command.
- `config.json` / `extensions/config.json` — behavior config. `assets/` — README imagery.
- `docs/solutions/`  # documented solutions to past problems (bugs, best practices, workflow patterns), organized by category with YAML frontmatter (module, tags, problem_type)
- `CONCEPTS.md`  # shared domain vocabulary (entities, named processes, status concepts) — relevant when orienting to the codebase or discussing domain concepts

Files under `shared/`, `lib/`, and `servers/shared/` carry a `// GENERATED FROM ... DO NOT EDIT.` header; edit upstream and regenerate with `npm run sync` (needs a sibling `../_ov` checkout of OpenViking).

## Development Commands

```bash
npm ci                # Node >=18; CI pins 20
npm test              # node --test tests/*.test.mjs — 23 cases
npm run typecheck     # tsc --noEmit (strict, allowJs)
npm run check         # syntax check extension + src
npm run e2e           # live; needs OPENVIKING_URL, E2E_LLM_API_KEY, omp
node scripts/setup.mjs  # credentials wizard -> ~/.openviking/ovcli.conf
```

Credentials resolve `OPENVIKING_*` env → `~/.openviking/ovcli.conf` → `~/.openviking/ov.conf` → `http://127.0.0.1:1933`.

## Style

ESM only. TypeScript `strict` in `src/` and `extensions/`; plain `.mjs` under `lib/`, `shared/`, `servers/`. No linter or formatter is configured — `tsc --noEmit` plus the test suite are the gate. Match the surrounding file's comment density and naming.

## Testing

`node:test` with `node:assert/strict`, one file per area (`plugin`, `recall`, `sync`, `uri-guard`, `v2`). Tests assert on shipped manifests, config defaults, and module behavior; add cases to the matching file instead of a new one.

## Commits and Pull Requests

Conventional prefixes, scoped when useful: `feat(P2):`, `fix(audit):`, `docs:`, `chore:`, `infra:`. Version bumps update `package.json` and `plugin.json` together as `chore: bump to X`. Target `main`; CI (`.github/workflows/ci.yml`) runs `npm ci --ignore-scripts`, `npm test`, `npm run typecheck`, `npm pack --dry-run`, and all must pass. Releases publish to npm through trusted publishing on `v*` tags.

## Working Agreements

After a solved, verified problem, automatically invoke the `ce-compound` skill with `mode:non-interactive` at the completion checkpoint only when the work produced durable project reasoning that is not readily recoverable from the final code, tests, types, comments, or existing documentation, and losing it would plausibly cause recurrence, material risk, or substantial rediscovery. Apply this counterfactual: if the learning document disappeared, would a future engineer reading the final implementation still be likely to repeat the mistake or redo substantial investigation? If not, do not invoke it. Completion, effort, and diff size alone are not enough. Capture at the checkpoint so a qualifying learning can ship in the PR that produced it, and only where the repository treats captured learnings as tracked, committed knowledge.

Write every report, summary, or handoff to the user through the `ce-noslop` skill. This applies when you are the top-level agent writing to the user, not when you are a subagent reporting to its caller. Do not apply it to code, config, verbatim quotes, or text the user asked to post as written.
