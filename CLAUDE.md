# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A webpage to compare TC39 spec revisions and PRs. It has two parts that live on the same `gh-pages` branch:

1. **Comparator website** — static HTML/JS/CSS served from `index.html`
2. **Build scripts** — Python tooling (`build.py`) that fetches spec snapshots from GitHub and generates the `history/` data

## Commands

### Setup

```sh
make init          # create Python venv, install deps, clone spec repo
npm install        # install eslint (for linting only)
```

Optionally create `token.json` with `{ "token": "GITHUB_TOKEN" }` to avoid GitHub API rate limits.

### Lint

```sh
make lint          # eslint on all JS files + flake8 on build.py
```

### Update history data

```sh
make update        # process all unprocessed revisions
make update1       # process 1 revision
make pr            # update all open PRs
./venv/bin/python build.py rev <sha>   # process a specific revision
./venv/bin/python build.py pr <number> # process a specific PR
./venv/bin/python build.py revs        # list revisions
./venv/bin/python build.py prs         # list PRs
```

### Build WebAssembly

```sh
cd lib && make     # requires wasm-pack and wasm-snip; outputs ../js/gunzip.wasm
```

## Architecture

### Data pipeline (`build.py`)

`build.py` is the core build script. It:
1. Clones the spec repo (`./ecma262/`) via `LocalRepository`
2. Fetches commit/PR metadata from the GitHub API via `GitHubAPI` / `RemoteRepository`
3. For each revision, runs the spec's own build toolchain inside the cloned repo to produce an HTML snapshot
4. Parses the snapshot with `lxml` to extract per-section data (`SectionExtractor`)
5. Saves everything to `history/` as gzip-compressed files: `index.html.gz`, `sections.json.gz`, `parent_diff.json.gz`
6. Updates the index files `history/revs.json` and `history/prs.json`

Key classes: `Config`, `Paths`, `GitHubAPI`, `LocalRepository`, `CacheChecker`, `Revisions`, `PRs`.

`broken_revs.json` lists revision SHAs known to fail building; they are skipped automatically.

### Frontend (`js/`, `index.html`)

All diffing happens client-side. The main entry points are:

- `js/compare.js` — main comparator logic; loads `revs.json`/`prs.json`, fetches `sections.json.gz` for selected revisions, and orchestrates the diff
- `js/base.js` — WebAssembly bootstrap; manually written bindings to `gunzip.wasm` for in-browser gzip decompression
- `js/path-diff-worker.js` / `js/tree-diff-worker.js` — web workers that perform the actual section diff computation off the main thread
- `js/snapshot-list.js` / `js/snapshot-loader.js` — support for the static snapshot viewer pages

### WebAssembly (`lib/`)

A tiny Rust crate (`lib/src/lib.rs`) wrapping `libflate` for gzip decompression, compiled with `wasm-pack`. The output `js/gunzip.wasm` is committed to the repo. The hand-written JS bindings in `js/base.js` replicate what `wasm-pack` would generate.

### Configuration

`config.json` controls which repository is tracked and which revision/PR numbers are the starting points. The `first_rev` / `update_first_rev` fields determine what gets processed; `releases` lists named release tags to include.

## JS style

ESLint is configured via `.eslintrc.js`: 2-space indent, double quotes, semicolons required, `prefer-const`, strict equality. `FALLTHROUGH` comments (uppercase) are the recognized fallthrough marker in switch statements.

## Branch layout

The `gh-pages` branch holds both the website source and all generated history data. The `master` branch is the upstream default (see git status). All automation pushes to `gh-pages`.
