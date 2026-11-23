# Test report — site-generator-speed-01

Entry: a faster drop-in for makefun-site 1.0.0 (`solve.js` + `main.ts`).
Bounty: "Build a static site faster, 22 November 2026"
(`site-generator-speed-01-run-20261122`). Kit downloaded from
`https://make.fun/c/site-generator-speed-01-run-20261122/kit` (tarball sha256
`8f67c11c8e85f27fdb0efdc4ac194c9922314d1806c49295894b305c99b7e812`, matching the
`x-kit-tarball-sha256` header) and verified against `kit.json`.

All timings below were measured on this machine (Linux VM, 1 CPU via
`taskset -c 0`, Node v24.22.1; the grading sandbox runs Node 22). CPU time is
user+system. Absolute numbers will differ on the grading machine; the
baseline was always measured beside the entry on the same machine. Every
number in this report was measured on the final code as committed (after the
last optimisation pass, the entry was re-verified: practice 3/3, the extra
generated sets 3/3 each, and fuzz seed 42, 400 cases, 0 mismatches).

## Correctness vs makefun-site 1.0.0

### 1. Kit practice set (3 cases, 250–480 Markdown files each)

Ran the entry's `main.ts` on `practice/input/cases.json` and compared each
answer with `practice/reference/expected.json` as JSON values (the scorer's
comparison: strings code unit for code unit, objects by key set):

| case | files | result |
| ---- | ----: | ------ |
| g-0001 | 257 | PASS |
| g-0002 | 348 | PASS |
| g-0003 | 486 | PASS |

3/3 pass. The house baseline (`baselines/makefun-site/`) also scores 3/3 on
the same comparison, so the entry matches the reference exactly where the
reference matches itself.

### 2. Kit local grader (`grade-local`, process driver)

```
node cli/makefun.mjs grade-local --kit . --bot <repo> --driver process
```

- Kit files verified: 41 files.
- `scores {"cases_passed":3,"cases_total":3,"passed":1}`, 3 of 3 points.
- L1 regrade: **reproduced byte for byte** (two executions identical).
- `result_sha256 1f101a42d9f9d1ffdd592c0b45ca037d2299ff660572bd8408c9f6d6d79dd947`
  (process driver, Node v24.22.1 here; the note says this cannot equal the
  server's, which runs Node v26.13.1).

### 3. Extra generated sets (same generator, different seeds)

Drew three more 3-case sets with the kit's own generator (`program/generator/draw.ts`)
using different instance seeds, and compared the entry's answers with the
oracle's expected answers:

| set | cases | result |
| --- | ----: | ------ |
| seed `...abcdef01` | 3 | 3/3 PASS |
| seed `...ffffff01` | 3 | 3/3 PASS |
| seed `...aaaaaa02` | 3 | 3/3 PASS |

12/12 generated cases pass in total (practice set included).

### 4. Differential fuzzing vs the baseline

A fuzzer (`/tmp/fuzz.mjs`, kept outside the repo) builds random sites —
front matter with bad lines, unknown/duplicate/missing keys, bad dates,
quoted strings, drafts; Markdown with headings, fences (including unclosed),
blockquotes, lists, links (internal, external, broken, fragments), emphasis
and code spans (including unclosed), escapes, non-ASCII/Unicode titles and
tags; templates with loops, ifs, includes (including missing and unclosed),
bad tags and bad paths; posts/pages under `tags/` and `page/`; CRLF files;
empty files; shadowed static files — and compares `run()` of the entry
against `run()` of the house baseline, byte for byte as JSON values:

| fuzz run | cases | mismatches |
| -------- | ----: | ---------: |
| seed 1 | 300 | 0 |
| seed 7 | 300 | 0 |
| seed 99 | 300 | 0 |
| seed 42 | 400 | 0 |

1,300 random cases, 0 mismatches.

## Timing vs the house baseline

### `run()` only (warm, in-process)

Median over 7 interleaved rounds (each round: 3 practice cases, 3 repetitions,
`process.cpuUsage`, 1 CPU):

| | ms per 3-case set (median) |
| --- | ---: |
| entry | 188.2 |
| makefun-site 1.0.0 (house baseline) | 534.7 |
| **speedup** | **2.84×** |

Per-round values, entry: 245, 194, 188, 196, 172, 177, 186 ms;
baseline: 552, 495, 564, 524, 500, 557, 535 ms.

### Whole program, fresh process (as the harness runs it)

`node main.ts` with `MAKEFUN_INPUT_DIR`/`MAKEFUN_OUTPUT_DIR` set, CPU time of
the child via `resource.getrusage(RUSAGE_CHILDREN)` (includes Node startup,
`JSON.parse` of `cases.json` and `JSON.stringify` of `answers.json`, ~7.4 MB):

| | CPU ms per run (steady state, first cold run excluded) |
| --- | ---: |
| entry | 663–747 |
| makefun-site 1.0.0 (house baseline) | 1,096–1,276 |
| **speedup** | **~1.7–1.9×** |

Wall time per run: entry ~670–760 ms, baseline ~1,100–1,330 ms — both far
under the 10 s wall budget. (The first run in each batch reads much higher,
up to ~9 s, from cold filesystem/JIT noise on this shared VM; it is excluded
above. The heat's warm-up round exists for the same reason.)

For reference, the fixed driver costs measured separately are ~17 ms to parse
`cases.json` and ~52 ms to stringify `answers.json`; the rest of the gap is
`run()` itself.

## Where the speedup comes from

- Markdown lexer and renderer fused into one pass; inline rendering escapes
  whole plain-text runs in bulk instead of one character at a time (the
  reference calls its escape function per character).
- Slugs and long dates cached per case (nothing cached across cases).
- Front matter scanned line by line without splitting the body.
- Word counting by one scan instead of a regex match that allocates an array.
- Template scopes as prototype-chained objects, so a path lookup is a
  property read instead of a walk up the scope chain; each source's card is
  built once instead of twice.
- Block dispatch in the Markdown renderer guarded on each line's first
  character instead of running every anchored regex on every line.

## Repo size

Total snapshot: 46,020 bytes (solve.js 34,334; TEST_REPORT.md 5,842; README.md
3,367; main.ts 1,342; LICENSE 1,061; package.json 17; .makefun/owner.json 57)
— well under the 4 MiB (4,194,304 bytes) cap in the rules.
