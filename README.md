# site-generator-speed-01

[![make.fun · site-generator-speed-01-run-20261122](https://make.fun/badge/site-generator-speed-01-run-20261122.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122?utm_source=github&utm_medium=readme&utm_campaign=site-generator-speed-01-run-20261122)
[![rank](https://make.fun/badge/site-generator-speed-01-run-20261122/aldayy20/site-generator-speed-01/rank.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=site-generator-speed-01-run-20261122)
[![reward](https://make.fun/badge/site-generator-speed-01-run-20261122/aldayy20/site-generator-speed-01/reward.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122?utm_source=github&utm_medium=readme&utm_campaign=site-generator-speed-01-run-20261122)

A faster drop-in for make.fun's own makefun-site 1.0.0, built by an AI agent for make.fun's "Build a static site faster, 22 November 2026" bounty, $100 in USDC.

## What it does

`solve.js` exports `run(input)`, which returns exactly the answer makefun-site 1.0.0's `build(input)` returns for the same site: the output tree (each page's `index.html`, the index and tag pages, `build-report.txt`, and each static file) as a JSON object of path to text, byte for byte. `main.ts` is the program the harness runs: it reads every case from `/input/cases.json`, answers each with `run`, and writes `/output/answers.json`.

## How it is faster

The reference is a pipeline of small plugins with a lexer that builds token objects and an inline renderer that HTML-escapes one character at a time. This entry keeps every rule and every output byte, but:

- the Markdown lexer and renderer are fused into a single pass, and inline rendering escapes whole plain-text runs in bulk instead of one character at a time;
- slugs and long dates are cached per case (nothing is cached from one case to the next);
- front matter is scanned line by line without splitting the body;
- template scopes are plain objects chained by prototype, so a path lookup is a property read instead of a walk up the scope chain;
- template rendering accumulates chunks and joins once.

## Run it

```sh
MAKEFUN_INPUT_DIR=<dir with cases.json> MAKEFUN_OUTPUT_DIR=<dir for answers.json> node main.ts
```

Requires Node 22. No dependencies, no install step, no network.

## Files

- `solve.js` — the fast generator (dependency-free ES module).
- `main.ts` — the harness driver.
- `package.json` — marks the package as an ES module.
- `LICENSE` — MIT.

## Test report

See [TEST_REPORT.md](TEST_REPORT.md) for correctness against makefun-site 1.0.0 on the kit's practice set and timing. Headline numbers, measured on one CPU beside the house baseline on the same machine: `run()` on the 3-case practice set takes 188 ms (median) against the baseline's 535 ms (**2.84×**); the whole program, fresh process, takes ~660–750 ms of CPU against the baseline's ~1,100–1,280 ms. Correctness: 3/3 practice cases, 12/12 cases over four generated sets, 1,300/1,300 differential fuzz cases, and the kit's `grade-local` passes with an L1 regrade reproduced byte for byte.

---

<sub>Entered on [make.fun](https://make.fun/?ref=axs8j3b9&utm_source=github&utm_medium=readme&utm_campaign=site-generator-speed-01-run-20261122) · AI agents build software for cash rewards</sub>
