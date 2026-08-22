# @hy-sde-org/dsh-memory

**Agent-curated long-horizon memory** for DeepSeek Harness — the `ctx.memory`
service and its provider registry, ported from the [@oh-my-pi](https://github.com/oh-my-pi)
coding-agent memory surface (see `port_omp.md` item 4 — the port lives in the
[hy-sde fork](https://github.com/hy-sde/deepseek-harness)). Memory is durable,
**project-scoped** data the agent curates itself with the
`retain`/`recall`/`reflect`/`memory_edit`/`learn` tools (shipped by
`@hy-sde-org/dsh-tool-memory`), and it is **reloaded at the start of the next
session** through prompt injection. It complements DSH's session-query and
compaction instead of overlapping them: those replay conversation history,
this bank answers "what did we decide / prefer / learn here?" across sessions.

This is a **standalone plugin build**: the harness integration (the host-plane
`memory` row) ships in `cordis.patch.yml`, and the agent-plane tools live in
`@hy-sde-org/dsh-tool-memory`. Nothing in the upstream DeepSeek Harness
(`dsh-v0.1.1-rc.2` and later) needs to change.

Only the **`local` backend** ships. The registry keeps the seam open for
Hindsight/Mnemopi-style providers later — a future provider registers one
`MemoryBackend` and the same tools work unchanged.

## Install

```bash
pnpm install --global @deepseek-ai/dsh
```

### Direct from npm (published)

Both packages are published on the npm registry under the `hy-sde-org`
organization (`@hy-sde-org/dsh-memory` and `@hy-sde-org/dsh-tool-memory`,
version `0.1.1-rc.2`). Add the service, then mount the tools via a preset:

```bash
# one command; the tool package comes in as a transitive dependency
dsh plugin --profile web add @hy-sde-org/dsh-memory @hy-sde-org/dsh-tool-memory
```

Then copy `examples/agent-preset/` from the installed package (or this repo)
to `~/.dsh/.agent-presets/<id>/` and select it in the Web UI preset picker
(or `dsh agent`). The preset row uses `@hy-sde-org/dsh-tool-memory`.

### From the git checkout (pre-publish / development)

```bash
git clone git@github.com:hy-sde/dsh-memory.git
cd dsh-memory
pnpm install
pnpm run build

MEMORY_TGZ="$(cd packages/memory && pnpm pack --silent --pack-destination /tmp)"
TOOLMEMORY_TGZ="$(cd packages/tool-memory && pnpm pack --silent --pack-destination /tmp)"
dsh plugin --profile web add "$MEMORY_TGZ" "$TOOLMEMORY_TGZ"
```

## Layout

The service is host-plane: the store is durable project data that crosses
sessions, so `cordis.patch.yml` inserts one row into the profile; the
per-session tools resolve it. Data lives under `<harness home>/memories/<project>/`
where `<project>` is an encoded absolute cwd — one memory root per project,
shared by every session and tool on it.

Each project root holds three artifacts:

- `bank.jsonl` — editable working entries written by `retain` (id, content,
  context, source, importance, timestamps, active flag). Backs `memory_edit`.
- `learned.md` — newest-first, deduped, capped (100) lesson bullets written by
  `learn`; the same format and normalization omp keeps. Survives
  consolidation; `learn` writes are injection-neutralized and secret-redacted.
- `memory_summary.md` — optional consolidated summary (hand- or tool-maintained)
  that `recall`, `reflect`, and prompt injection surface.

## Service API

```ts ignore-check
const memory = ctx.memory                       // MemoryService
await memory.save({ cwd }, { content, context, source, importance })
await memory.learn({ cwd }, { content, context })
await memory.search({ cwd }, 'query', { limit: 10 })
await memory.edit({ cwd }, 'update' | 'forget' | 'invalidate', { id, content, importance, replacementId })
await memory.summaries({ cwd })                 // { summary?, learned?, block }
await memory.status({ cwd })
await memory.clear({ cwd })
```

Backends register into the service: `memory.register(backend)` returns a
disposer, and `memory.resolve()` returns the configured (default: first)
backend. Mutations emit `memory/change` (`{ cwd }`) so in-process consumers
can invalidate caches.

A `MemoryBackend` is a dozen methods over `@hy-sde-org/dsh-memory/types`;
the shipped one, `LocalMemoryBackend`, is pure-node (`node:fs`) with an
in-process per-file write chain so concurrent saves from sibling sessions can
never drop each other's writes.

## Config (the `memory` row)

| Key | Default | Meaning |
|---|---|---|
| `root` | `<harness home>/memories` | Memory root (`~`/`$HOME` expand). |
| `backend` | first registered | Backend id the service delegates to. |
| `defaultImportance` | `0.7` | Baseline importance when a save omits it. |
| `searchLimit` | `10` | Default result cap for one search. |

Per-entry caps: bank content 4000 chars, bank context 800, lesson content
2000, lesson context 400, lessons capped at 100 newest-first. All stored text
passes injection-neutralization (control chars, `<`/backticks, `~~~` fences)
then secret-redaction, on write and on read.

## Tests

```sh
pnpm -r --filter @hy-sde-org/dsh-memory test
```
