---
name: omp-openviking-memory
last_updated: 2026-09-20
---

# omp-openviking-memory Strategy

## Purpose

Every harness the operator runs — Claude Code, Codex, pi — shares one memory system, OpenViking; OMP is the only one with no path into it, and the port that would give it one does not work at all today. Because the path is a port rather than a first-party integration, the gap is total: nothing done in OMP is remembered anywhere the other three can read.

## Positioning

Parity with the official pi extension is the baseline guarantee, not a ceiling: hooks, lifecycle, tool semantics, and the `config.json` + `takeover.*` surface are reproduced exactly, and additions beyond upstream are allowed when they earn their place by an outcome. One shared `~/.openviking` home and one peer identity per worktree, so a memory written in OMP stays readable from the other three.

## Users

**Primary:** the operator — runs one self-hosted OpenViking server as the single memory substrate across Claude Code, Codex, pi, and OMP. They're hiring OMP's extension to make an OMP session pick up project history the other three already captured, and to write back what OMP learns.

## Boundaries

- A private config or state home — the shared `~/.openviking/`, harness-named subdirectories only.
- A second memory layer inside OMP — OpenViking is the exclusive one.
- Two harnesses in one worktree — one harness per worktree at a time.
- MCP as a second memory writer — `mcp.json` stays a fallback, manual tools only.
- Vendored files edited in place — patches are recorded and survive `npm run sync`.
- A second public persona — the repo is public as packaging; decisions stay the operator's.

_Resist a change when:_ it would split OMP off the shared OpenViking identity — one home, one peer identity, a harness label saying who wrote a memory — or it cannot be justified by an outcome for exclusive OpenViking memory in OMP.

## Key metrics

- **Parity conformance** — upstream pi behaviours reproduced ÷ upstream total; measured by a conformance comparison against the pi extension, which does not exist yet.
- **Zero-error operation** — sessions where init, recall, or sync raised an error: 0; today only the `OV ✓/✗` footer and `OV_DEBUG_LOG`.
- **Cross-harness read-through** — a fact captured in OMP surfaces later from pi, Claude Code, or Codex; measured by extending the golden-set runner to OMP-captured memories.

## Tracks

### Runtime bring-up

The port does not work end-to-end; this track owns hooks firing, recall injecting, capture landing, and correct session identity in the shared store.

_Why it serves the approach:_ parity is meaningless until the thing runs.

### Parity conformance

Inventory the official pi extension's surface — hooks, lifecycle, tool semantics, `config.json` and `takeover.*`, and how it registers itself per harness — close the gaps, and keep the comparison running.

_Why it serves the approach:_ it is the measure that proves parity, and it owns the evidence for cross-harness read-through.

### Making it OMP's memory authority

Adopt the shared `~/.openviking` home and peer derivation, and become the only memory layer active in OMP.

_Why it serves the approach:_ the purpose is one memory system across harnesses, not a fourth silo.
