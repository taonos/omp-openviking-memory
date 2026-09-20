# Concepts

Shared domain vocabulary for this project — entities, named processes, and status concepts with project-specific meaning. Seeded with core domain vocabulary, then accretes as ce-compound and ce-compound-refresh process learnings; direct edits are fine. Glossary only, not a spec or catch-all.

## Shared store

### Shared OpenViking home
The one OpenViking state directory that every harness extension reads and writes, holding credentials, the parked-payload queue, and any harness-local state. Distinct from the server: the server holds memories, while the home decides which server and account a harness talks to and where its local state lives. Two harnesses can point at the same server and still read different stores if they resolve different homes.

### Peer identity
The memory-scope identity a harness sends with each request, partitioning memories the way a project partitions work. Derived from the working directory rather than from the harness, so every harness working in one worktree shares it while different worktrees of one project do not. Because it is the worktree's identity, it cannot say which harness wrote a memory — that has to come from the harness label.

### Harness label
The part of a stored session id that names the harness which wrote the memory. It is the only harness-identifying information a memory carries, so a wrong or missing label makes one harness's memories indistinguishable from another's even when they share a home and a peer.

## Memory lifecycle

### Recall
Context assembled from the store and injected into a prompt before the agent starts work. Budgeted, and keyed to the prompt actually being answered rather than prefetched ahead of it, so a recall miss shows up as a prompt answered without context rather than as an error.

### Capture
Syncing a finished turn into the store so later sessions can recall it. Asynchronous from the conversation: a failed sync parks the payload instead of dropping it, and a later session replays the park when it next starts. Because the replay runs from whichever harness starts next, a park's ownership is only as good as the identity on the entry.
