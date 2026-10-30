**中文** · [README.md](./README.md)

> [!WARNING]
> **⚠️ This tool has been merged into the [doctors](https://github.com/huanweide/doctors) monorepo and is no longer maintained here.**
>
> The source now lives at [`doctors/packages/awaitscan/`](https://github.com/huanweide/doctors/tree/main/packages/awaitscan). Inside doctors it runs as the `doctors await` subcommand (or `node packages/awaitscan/index.js`). This repository no longer accepts changes — please use <https://github.com/huanweide/doctors>.

<!-- badges -->
[![License](https://img.shields.io/github/license/huanweide/awaitscan)](LICENSE)
[![Stars](https://img.shields.io/github/stars/huanweide/awaitscan)](https://github.com/huanweide/awaitscan/stargazers)
[![zero-dep checkup CLI](https://img.shields.io/badge/zero--dep%20checkup%20CLI-18%20tools-22c55e)](https://github.com/huanweide/family-showcase)
<!-- /badges -->

**🔧 Part of the [huanweide zero-dependency CLI family](https://github.com/huanweide/family-showcase) (18 single-file tools). Use it standalone, as a subcommand inside the [doctors](https://github.com/huanweide/doctors) monorepo, or run everything together via [enghealth](https://github.com/huanweide/enghealth).**

# awaitscan

> Zero-dependency, single-file Node CLI · static scanner for asynchronous performance anti-patterns

Spot the async code that quietly slows you down. `awaitscan` statically detects three high-signal, zero-dependency, deterministic performance anti-patterns, then gives you a **0–100 health score** and a CI-ready **gate**.

Zero dependencies. Zero config. Offline. Single file. Runs out of the box.

## Who should use this

- Node / TypeScript projects that rely heavily on `async/await`.
- Teams that want an async-performance gate in pre-commit or CI without pulling in a heavy linter.
- Anyone who wants a deterministic, reproducible report (same code → same score).

## Who should not use this

- Projects that are not JS/TS — it only scans `.js/.mjs/.cjs/.jsx/.ts/.mts/.cts/.tsx/.vue/.svelte`.
- If you already run a full linter (e.g. ESLint with `perf` rules) and don't need a focused, dependency-free gate.
- Note: this repo is deprecated — prefer `doctors await` for new setups.

## Why

`async/await` makes async code read like sync code, but it also hides performance traps: `await` inside a `for` loop (should be `Promise.all`), `await` inside a `.map()` callback (map doesn't parallelize), and hand-written nested loops (O(n²)). Either you catch these in manual review, or you install a heavy linter.

`awaitscan` collapses all of that into one command: zero dependencies, zero config, one file. It hands you the issue list, the health score, and the CI gate.

## The three anti-patterns

| Rule ID | Severity | What it catches |
| --- | --- | --- |
| `serial-await-in-loop` | high | `await` directly inside a loop body (`for`/`for-of`/`while`/`do`) — should be `Promise.all` |
| `await-in-map` | high | `await` inside a `.map()` callback (map doesn't parallelize; wrap in `Promise.all`) |
| `nested-loop` | medium | A loop nested inside another at the surface level (O(n²) risk; confirm whether a hash/set or split helps) |

> Legitimate patterns are not false-flagged: `for await (const x of asyncIter)` consuming an async iterator is skipped correctly; `await` inside a callback (not at the loop's surface) is not treated as serial-in-loop; `Promise.all(arr.map(async x => ...))` is recognized as real parallelism and exempted.

## Features

- **Zero-dependency single file**: just one `index.js`, run with `node index.js ./` — no `npm install`.
- **Deterministic static scan**: uses bracket-balance to locate loop bodies / map-callback bodies, strips the inner block to get the "surface" before detecting, so an `await` in an inner function is not miscounted as serial-in-loop.
- **Two-stage false-positive reduction**: a character-by-character tokenizer strips comments, strings and template literals (keeping newlines) before scanning, so fake `await`/`for` inside strings or docs are excluded.
- **Health score + gate**: 0–100 score (1 weighted issue per 1k lines is free; beyond that it deducts by factor 25). Every gate threshold is validated with `Number.isFinite`; a non-integer value exits `2` immediately — never silently passes.
- **Safe**: pure local read-only, no network, no execution surface; `statSync` validates the root before scanning; files >5MB are skipped to avoid OOM; `node_modules/.git` and similar are auto-skipped.

## Install and run

This repository is deprecated (merged into doctors). The standalone tool is not published on npm — run the single file directly:

```bash
git clone https://github.com/huanweide/awaitscan.git
node awaitscan/index.js ./your-repo
```

Or copy `index.js` into your own repo (it only uses Node built-ins, no `npm install`). Inside the doctors monorepo you can also run `doctors await`.

> Requires Node 18+.

## Usage

```bash
# Scan the current directory by default
awaitscan

# Scan a specific directory, output JSON (for pipelines)
awaitscan --root ./src --json

# CI gate: fail on any high-severity anti-pattern (serial await / await-in-map)
awaitscan --root . --fail-on-high

# Fail if more than 0 high-severity issues (equivalent)
awaitscan --root . --max-high 0

# Fail if more than 3 medium issues (nested loops)
awaitscan --root . --max-medium 3

# Fail if more than 5 total issues
awaitscan --root . --max-issues 5
```

### Gate parameters

| Parameter | Meaning | Fails when |
| --- | --- | --- |
| `--max-high <n>` | count of high-severity anti-patterns (serial await / await-in-map) | exceeds `n` |
| `--max-medium <n>` | count of medium anti-patterns (nested loops) | exceeds `n` |
| `--max-issues <n>` | total issue count | exceeds `n` |
| `--fail-on-high` | fail on any high-severity anti-pattern | any high present |
| `--fail-on-issues` | fail on any warning | any warning present |

> Any non-integer threshold exits `2` (foolproof) — it will never quietly pass.

## CI integration

```yaml
# .github/workflows/perf.yml
- name: Async performance gate
  run: node .github/awaitscan/index.js --root . --fail-on-high
```

## How it works

`awaitscan` is a heuristic scanner built on **bracket-balance + surface-layer detection** rather than a full AST. It:

1. Tokenizes the source character-by-character, blanking out comments and strings (newlines preserved) so only "bare code" is inspected.
2. Uses bracket balance to find loop bodies and `.map()` callbacks, then removes the inner block to analyze only the surface.
3. Flags the three anti-patterns above and weights them (high ×3, medium ×2).
4. Computes the health score and evaluates the gate.

A sample run looks like:

```
awaitscan · async performance scan · root=./src
  source files : 12
  source lines : 1840
  total issues : 3 (weighted 8)
  severity     : high=2 medium=1 low=0
  rules        : serial-await-in-loop=2  await-in-map=0  nested-loop=1
  health score : 71 / 100  [basically healthy]
  gate: pass
```

## How it differs from oxlint

[oxlint](https://github.com/oxc-project/oxlint) (Rust, zero-config, 700+ rules) has a `perf` category, but it is a general-purpose linter and does not focus specifically on the high-signal "serial await semantics" slice. `awaitscan` is a zero-dependency single file, pure-JS static scan that is **deterministic and reproducible**, purpose-built as a ruler for the most-overlooked yet throughput-killing async anti-patterns (`serial await in loop` / `await in map`). It slots neatly into any Node project's CI as an async-performance checkup.

## In the same family

`awaitscan` is the "async performance checkup" member of the huanweide zero-dependency CLI family (same pattern as):

- **devdoctor** — dependency checkup (bloat / licenses / circular deps / plaintext secrets)
- **testlite** — test-hygiene checkup (framework / density / coverage)
- **debtlens** — tech-debt density checkup (TODO/FIXME markers + health score)
- **a11ydoctor** — frontend accessibility checkup (Web a11y anti-patterns + health score)
- **awaitscan** — async performance checkup (serial await / await-in-map / nested loops) ← you are here

All five are zero-dependency single files sharing the same engineering rules; use them alone or combine them into a project checkup pipeline.

## Limitations

- Built on the "bracket-balance + surface scan" heuristic, not a full AST; very unusual constructions (e.g. a loop assembled via dynamic `eval`) may be missed. The numbers are **estimates** — pair with manual review in heavy cases.
- Only scans JS/TS ecosystem source files (`.js/.mjs/.cjs/.jsx/.ts/.mts/.cts/.tsx/.vue/.svelte`); no docs/config.
- Nested loops (medium) are a hint signal: many legitimate 2-D traversals are nested loops — judge with context before "optimizing".

## Roadmap

This standalone repository is frozen (merged into doctors). New work happens in the [doctors](https://github.com/huanweide/doctors) monorepo and the [family-showcase](https://github.com/huanweide/family-showcase) overview.

## Contributing

This repo no longer accepts changes. Please open issues or PRs against [doctors](https://github.com/huanweide/doctors). To experiment with the standalone tool, fork it and run `node index.js ./your-repo` to test.

## License

MIT © huanweide

---

## Author

Maintained by **huanweide** · [GitHub](https://github.com/huanweide)

## More zero-dependency CLIs

huanweide maintains a series of checkup CLIs with the same pattern (18 single-file tools in total). Visit [family-showcase](https://github.com/huanweide/family-showcase) for the overview, or use the aggregate runner [doctors](https://github.com/huanweide/doctors).

## Star this project

If `awaitscan` helps you catch a slow async path before it ships, give it a ⭐ — and check out the rest of the [family](https://github.com/huanweide/family-showcase).

← [Back to the zero-dependency CLI family overview](https://github.com/huanweide/family-showcase)
