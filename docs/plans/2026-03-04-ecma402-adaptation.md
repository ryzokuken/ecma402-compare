# ECMA-402 Adaptation Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Retarget the ecma262-compare tooling to track and diff the tc39/ecma402 repository instead.

**Architecture:** Seven targeted file edits (no new files) cover all hardcoded ecma262 references. The
only non-trivial code change is fixing `remove_unnecessary_deps` in `build.py`, which currently assumes
ecmarkup lives in `devDependencies` — ecma402 puts it in `dependencies` instead. After the file edits,
the history data is re-initialized and a single revision is built to verify the pipeline works end-to-end.

**Tech Stack:** Python 3 (build pipeline), vanilla JS (frontend), Node/npm (spec build via ecmarkup)

---

## Background: what actually needs to change

The tool is 95% repo-agnostic. The only ecma262-specific coupling is:

| Location | What's hardcoded |
|---|---|
| `config.json` | GitHub repo + API URLs, starting revision SHA, starting PR number, release tags |
| `build.py:298` | `LocalRepository.DIR = 'ecma262'` (clone target dir name) |
| `build.py:700` | `__SPEC_DATE_PREFIX = 'Draft ECMA-262 / '` (date string in generated HTML) |
| `build.py:722-729` | `remove_unnecessary_deps` reads from `devDependencies`; ecma402 uses `dependencies` |
| `build.py:1311,1322` | argparse help strings |
| `js/compare.js:9-10` | `REPO_URL`, `REPO_API_URL` constants |
| `js/snapshot-list.js:9` | `REPO_URL` constant |
| `index.html`, `snapshot*.html` | Page titles, `<h1>` text, GitHub repo link |
| `package.json` | `name`, `description` metadata |
| `Local.md`, `README.md` | Documentation |

The JS diff workers, section extractor, CSS, and WASM are fully spec-agnostic and need no changes.

---

## Task 1: Update `config.json`

**Files:**
- Modify: `config.json`

**Step 1: Get the first ecma402 commit SHA you want to process**

Find a reasonable starting point — the oldest commit you want in the history. For an initial test
you can use a recent commit; you can always add older history later.

```sh
# Clone ecma402 temporarily to find a good starting SHA, or use the GitHub API:
curl -s "https://api.github.com/repos/tc39/ecma402/commits?per_page=1" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d[0]['sha'])"
```

For the initial test, grab the current HEAD SHA printed above. You can also browse
`https://github.com/tc39/ecma402/commits/main` and pick any commit.

**Step 2: Get the first open PR number you want to track**

```sh
curl -s "https://api.github.com/repos/tc39/ecma402/pulls?state=open&per_page=1" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d[0]['number'] if d else 'no open PRs')"
```

**Step 3: Edit `config.json`**

Replace the entire file contents with:

```json
{
 "repo_url": "https://github.com/tc39/ecma402/",
 "api_url": "https://api.github.com/repos/tc39/ecma402/",
 "first_rev": "<SHA from Step 1>",
 "update_first_rev": "<SHA from Step 1>",
 "first_pr": <number from Step 2, or 1 to process all PRs>,
 "releases": []
}
```

`releases` is set to `[]` initially. ecma402 does have release tags (e.g. `es2016` through `es2025`)
but getting them requires verifying which tag names exist in the repo — add them later once the
basic pipeline works.

**Step 4: Commit**

```sh
git add config.json
git commit -m "config: retarget to tc39/ecma402 repo"
```

---

## Task 2: Fix `build.py`

**Files:**
- Modify: `build.py`

Four independent edits in one file.

**Step 1: Fix the local clone directory name (line 298)**

Find:
```python
    DIR = os.path.join('.', 'ecma262')
```
Replace with:
```python
    DIR = os.path.join('.', 'ecma402')
```

**Step 2: Fix the spec date prefix (line 700)**

The ecma262 spec generates HTML containing `"Draft ECMA-262 / March 4, 2026"`.
The ecma402 spec generates `"Draft ECMA-402 / March 4, 2026"` (confirmed by checking tc39.es/ecma402/).

Find:
```python
    __SPEC_DATE_PREFIX = 'Draft ECMA-262 / '
```
Replace with:
```python
    __SPEC_DATE_PREFIX = 'Draft ECMA-402 / '
```

**Step 3: Fix `remove_unnecessary_deps` (lines 722-729)**

The ecma262 `package.json` has ecmarkup under `devDependencies`. The ecma402 `package.json`
has ecmarkup under `dependencies` alongside `@tc39/ecma262-biblio` (needed at build time).
The current code will throw a `KeyError` on ecma402.

Find the entire method:
```python
    @classmethod
    def remove_unnecessary_deps(cls, package_json_path):
        package = FileUtils.read_json(package_json_path)
        ecmarkup_ver = package["devDependencies"]["ecmarkup"]
        package["devDependencies"] = {
            "ecmarkup": ecmarkup_ver
        }
        FileUtils.write_json(package_json_path, package)
```
Replace with:
```python
    @classmethod
    def remove_unnecessary_deps(cls, package_json_path):
        package = FileUtils.read_json(package_json_path)
        # ecma262 puts ecmarkup in devDependencies; ecma402 puts it in dependencies
        # alongside @tc39/ecma262-biblio which is also required at build time
        if "devDependencies" in package and "ecmarkup" in package["devDependencies"]:
            ecmarkup_ver = package["devDependencies"]["ecmarkup"]
            package["devDependencies"] = {"ecmarkup": ecmarkup_ver}
        if "dependencies" in package and "ecmarkup" in package["dependencies"]:
            package["dependencies"] = {
                k: v for k, v in package["dependencies"].items()
                if k == "ecmarkup" or k.startswith("@tc39/")
            }
        FileUtils.write_json(package_json_path, package)
```

**Step 4: Update argparse strings (lines ~1311, ~1322)**

Find:
```python
    parser = argparse.ArgumentParser(description='Update ecma262 history data')
```
Replace with:
```python
    parser = argparse.ArgumentParser(description='Update ecma402 history data')
```

Find:
```python
                      help='Clone ecma262 repository')
```
Replace with:
```python
                      help='Clone ecma402 repository')
```

**Step 5: Verify the exact line numbers haven't shifted before editing**

```sh
grep -n "ecma262" build.py
```

Expected output lines: `DIR = os.path.join('.', 'ecma262')`, `ECMA-262`, and the two argparse strings.

**Step 6: Commit**

```sh
git add build.py
git commit -m "build: retarget to ecma402 (clone dir, date prefix, deps, help text)"
```

---

## Task 3: Update frontend JS constants

**Files:**
- Modify: `js/compare.js`
- Modify: `js/snapshot-list.js`

**Step 1: Edit `js/compare.js` lines 9-10**

Find:
```js
const REPO_URL = "https://github.com/tc39/ecma262";
const REPO_API_URL = "https://api.github.com/repos/tc39/ecma262";
```
Replace with:
```js
const REPO_URL = "https://github.com/tc39/ecma402";
const REPO_API_URL = "https://api.github.com/repos/tc39/ecma402";
```

**Step 2: Edit `js/snapshot-list.js` line 9**

Find:
```js
const REPO_URL = "https://github.com/tc39/ecma262";
```
Replace with:
```js
const REPO_URL = "https://github.com/tc39/ecma402";
```

**Step 3: Commit**

```sh
git add js/compare.js js/snapshot-list.js
git commit -m "js: update repo URLs to tc39/ecma402"
```

---

## Task 4: Update HTML titles and links

**Files:**
- Modify: `index.html`
- Modify: `snapshot.html`
- Modify: `snapshot_revs.html`
- Modify: `snapshot_prs.html`

Each of the four HTML files has the same set of changes. Do them in sequence.

**Step 1: In each file, change the `<title>` tags**

| File | Old title | New title |
|---|---|---|
| `index.html` | `ECMAScript Language Specification Comparator` | `ECMA-402 Internationalization API Comparator` |
| `snapshot.html` | `ECMAScript Language Specification Snapshots` | `ECMA-402 Internationalization API Snapshots` |
| `snapshot_revs.html` | `ECMAScript Language Specification Revision Snapshots` | `ECMA-402 Internationalization API Revision Snapshots` |
| `snapshot_prs.html` | `ECMAScript Language Specification PR Snapshots` | `ECMA-402 Internationalization API PR Snapshots` |

**Step 2: In each file, change the `<h1>` text**

All four files have:
```html
      <h1>ECMAScript Language Specification Comparator</h1>
```
Replace with:
```html
      <h1>ECMA-402 Internationalization API Comparator</h1>
```

**Step 3: In `index.html` only, update the GitHub link (line 41)**

Find:
```html
      <a href="https://github.com/arai-a/ecma262-compare"><img src="./img/github.png" ...></a>
```
Replace with your fork's URL, e.g.:
```html
      <a href="https://github.com/YOUR_USERNAME/ecma402-compare"><img src="./img/github.png" ...></a>
```

**Step 4: In `index.html` only, update the "File an Issue" link (line ~156)**

Find:
```html
          <a href="https://github.com/arai-a/ecma262-compare/issues/new">File an Issue</a>!
```
Replace with your fork's issues URL.

**Step 5: In `snapshot_revs.html` and `snapshot_prs.html`, update the GitHub link (line ~31)**

Same pattern as Step 3 — find the `arai-a/ecma262-compare` href and replace with your fork's URL.

**Step 6: Commit**

```sh
git add index.html snapshot.html snapshot_revs.html snapshot_prs.html
git commit -m "html: retitle pages and update links for ecma402-compare"
```

---

## Task 5: Update `package.json`

**Files:**
- Modify: `package.json`

**Step 1: Edit**

Find:
```json
  "name": "ecma262-compare",
  "description": "ECMAScript Language Specification Comparator",
```
Replace with:
```json
  "name": "ecma402-compare",
  "description": "ECMA-402 Internationalization API Specification Comparator",
```

**Step 2: Commit**

```sh
git add package.json
git commit -m "chore: rename package to ecma402-compare"
```

---

## Task 6: Initialize history data

The `history/` directory contains ecma262 revision data that needs to be replaced.

**Step 1: Reset the index files**

```sh
echo '[]' > history/revs.json
echo '[]' > history/prs.json
```

**Step 2: Remove old revision data (if any history/ SHA dirs exist)**

Check what's there first:
```sh
ls history/ | head -20
```

If there are SHA directories (40-char hex names) or `PR/` subdirectories, remove them. These are
large and should not be committed to the repo — they should already be gitignored or be the
ecma262 data from upstream. Check `.gitignore` first:

```sh
cat .gitignore 2>/dev/null || echo "no .gitignore"
```

If they are tracked, remove them:
```sh
# Only do this if old history dirs are tracked by git:
git rm -r --cached history/PR history/<SHA>...  # list specific dirs
```

**Step 3: Commit the reset index files**

```sh
git add history/revs.json history/prs.json
git commit -m "history: reset index files for ecma402"
```

---

## Task 7: Set up the Python environment and test a single revision

This validates that the entire pipeline works end-to-end before running a full update.

**Step 1: Set up venv (if not done already)**

```sh
make py-venv
```

**Step 2: Clone the ecma402 repo**

```sh
make clone
```

This runs `build.py clone` which clones `tc39/ecma402` into `./ecma402/`. Verify:
```sh
ls ecma402/
```
Expected: you see `spec/`, `package.json`, `img/`, etc.

**Step 3: Process exactly one revision (the HEAD commit)**

```sh
./venv/bin/python build.py rev -c 1 all
```

This will:
1. Fetch `origin/main` in `./ecma402/`
2. Check out the HEAD commit
3. Run `npm install` + `npm run build-only` inside `./ecma402/`
4. Extract section data to `history/<SHA>/sections.json.gz`
5. Write `history/revs.json` with one entry

**Expected output:** lines like:
```
[INFO] history/<SHA>/index.html.gz
[INFO] history/<SHA>/sections.json.gz
```

If `npm run build-only` fails:
- Check Node.js version: `node --version` (ecma402 requires a recent Node)
- Check that `ecma402/package.json` shows `ecmarkup` under `dependencies`
- Try running manually: `cd ecma402 && npm install && npm run build-only`

**Step 4: Verify `history/revs.json` has one entry**

```sh
python3 -c "import json; d=json.load(open('history/revs.json')); print(len(d), 'revisions')"
```

Expected: `1 revisions`

**Step 5: Open the comparator in a browser**

Open `index.html` in a local server:
```sh
python3 -m http.server 8080
```
Then visit `http://localhost:8080`. The revision dropdown should show one entry. With only one
revision there is nothing to diff yet — process a second revision:

```sh
./venv/bin/python build.py rev -c 1 all
```

Then select two revisions and verify a diff loads.

**Step 6: Commit any updated CLAUDE.md if needed**

```sh
git add history/revs.json history/prs.json  # updated by the build
git commit -m "history: initial ecma402 revision data"
```

---

## Notes for after the pipeline is working

**Release tags:** If you want named release snapshots (es2016 through es2025), verify the tag
names in tc39/ecma402:
```sh
cd ecma402 && git tag | grep -i es20
```
Add any valid tags to `config.json` `releases` array.

**GitHub Actions:** The upstream ecma262-compare automation lives in `.github/workflows/`. This
repo has no workflows yet. Once the pipeline is verified locally, port the upstream automation
(replacing `ecma262` with `ecma402` throughout) to automate periodic updates.

**Icons:** The `img/` files still reference `ecma262-compare` in their names. They're just PNGs/ICOs
and can be renamed/replaced anytime without affecting functionality.

**Update `CLAUDE.md`** to reflect these changes once everything is working.
