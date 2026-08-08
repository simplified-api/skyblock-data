# Contributing to the SkyBlock Data Corpus

Thank you for your interest in contributing! This document explains how to get started, what to expect during the review process, and the conventions this repository follows.

## Table of Contents

- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Development Setup](#development-setup)
  - [Editor Setup](#editor-setup)
- [Making Changes](#making-changes)
  - [Branching Strategy](#branching-strategy)
  - [Editing Data](#editing-data)
  - [Adding a Table](#adding-a-table)
  - [Commit Messages](#commit-messages)
  - [Validating Output](#validating-output)
- [Submitting a Pull Request](#submitting-a-pull-request)
- [Reporting Issues](#reporting-issues)
- [Repository Architecture](#repository-architecture)
- [Legal](#legal)

## Getting Started

### Prerequisites

| Requirement | Version | Notes |
|-------------|---------|-------|
| Python | **3.8+** | Standard library only - no virtualenv, no `pip install` |
| Git | 2.x+ | For cloning and contributing |
| Editor | Any | Anything that can be told not to reformat JSON |

There is no build, no test suite and no toolchain. The whole contract is: edit JSON, run one script, commit both.

### Development Setup

1. **Fork and clone the repository**

   [Fork the repository](https://github.com/simplified-api/skyblock-data/fork), then clone your fork:

   ```bash
   git clone https://github.com/<your-username>/skyblock-data.git
   cd skyblock-data
   ```

2. **Confirm the index is in sync before you change anything**

   ```bash
   python scripts/generate_index.py --check
   ```

   A clean checkout should print `ok: data/v1/index.json is in sync (41 entries)`. If it does not, your git configuration is rewriting line endings - see [Line endings](#line-endings) before going further.

3. **Confirm Python**

   ```bash
   python --version    # 3.8 or newer
   ```

### Editor Setup

The digests in `index.json` are taken over the file bytes exactly as stored, so anything your editor does on save is a content change.

- **Turn off format-on-save for `.json` in this repository.** A reformat rewrites every line of a 7 MB file, produces an unreviewable diff, and changes the digest.
- **Do not sort keys or re-indent existing files.** Match the surrounding file's style when you add an entry.
- **Ensure LF line endings.** `.gitattributes` declares `* text=auto eol=lf`; an editor forcing CRLF fights it.
- Leave the trailing newline on every file.

## Making Changes

### Branching Strategy

- Create a feature branch from `master` for your work.
- Use a descriptive branch name: `fix/hyperion-rarity`, `data/add-attribute-shards`, `docs/index-schema`.

```bash
git checkout -b data/my-change master
```

### Editing Data

1. Edit or add files under `data/v1/<category>/`.
2. Regenerate the manifest:

   ```bash
   python scripts/generate_index.py
   ```

3. Stage **both** the data edits and the refreshed `data/v1/index.json`, in one commit.

> [!IMPORTANT]
> Never hand-edit `data/v1/index.json`. It is generated output; a hand-written digest that happens to be wrong is worse than a stale one, because the check compares content and not intent.

Conventions for the data itself:

- Every file is a JSON **array** of objects, each carrying a natural `id`.
- An id is uppercase snake case, matching what Hypixel sends. Some ids carry a colon (`INK_SACK:3`); leave them alone.
- A table with nothing in it ships as `[]` rather than being deleted - a registered table with no file aborts the generator, so the empty array is how a table stays declared and unpopulated. `pet_items.json` and `fairy_souls.json` are both in that state today.
- Keep entries in the order the file already uses. Re-sorting a whole file to add one entry buries the change.

### Adding a Table

Adding a JSON file is not enough on its own - three things move together, and the generator refuses to emit an index if any is missing.

1. **The file** - `data/v1/<category>/<table>.json`, where `<table>` is the `@Table(name = ...)` of the Java model.
2. **The mapping** - an entry in `MODEL_CLASS_BY_TABLE` at the top of `scripts/generate_index.py`. The class name cannot be derived from the file name; English plurals are irregular enough (`bestiary_categories` to `BestiaryCategory`, `essences` to `Essence`) that a rule would be wrong more often than the dict is tedious.
3. **The model** - a matching `JpaModel` entity in [simplified-api/skyblock](https://github.com/simplified-api/skyblock) under `dev.sbs.skyblockdata.model`.

Removing a table is the same three in reverse. A mapping left behind after its file is deleted aborts the generator as a stale entry, which is the intended failure.

> [!TIP]
> Land the model class in `simplified-api/skyblock` first or in parallel. `master` there must be able to connect against `master` here, and a table whose `model_class` names a class that does not exist yet fails at bind rather than at index generation.

### Commit Messages

Write clear, concise commit messages that describe *what* changed and *why*.

```
Add the 2025 anniversary balloon hat to the items extra

The bulk item dump is generated from an upstream source that has never
carried the anniversary hats, so they are maintained by hand in the
extra rather than being reintroduced and lost on the next regeneration.
```

- Use the imperative mood ("Add", "Fix", "Update", not "Added", "Fixes").
- Keep the subject line under 72 characters.
- Add a body when the *why* isn't obvious from the subject.
- Say where a value came from. A stat correction with no source is unreviewable.

### Validating Output

- **The index check** - the only gate, and it is the same one CI runs:

  ```bash
  python scripts/generate_index.py --check
  ```

- **JSON validity** - the generator hashes bytes and does not parse your data, so malformed JSON passes the check here and fails at the consumer:

  ```bash
  python -m json.tool data/v1/<category>/<table>.json > /dev/null
  ```

- **Diff size sanity** - `git diff --stat` before you push. A one-entry change touching thousands of lines means your editor reformatted the file; revert and redo it without the reformat.

#### Line endings

If `--check` fails on a clean checkout, or your diff shows every line changed, git is rewriting line endings:

```bash
git config core.autocrlf false
git rm --cached -r .
git reset --hard
```

`.gitattributes` forces `eol=lf` for exactly this reason: the digest is over the working-tree bytes, so a CRLF checkout hashes differently from what Linux CI computes and every check fails.

## Submitting a Pull Request

1. **Push your branch** to your fork.

   ```bash
   git push origin data/my-change
   ```

2. **Open a Pull Request** against the `master` branch of [simplified-api/skyblock-data](https://github.com/simplified-api/skyblock-data).

3. **In the PR description**, include:
   - What changed and where the values came from - a wiki page, an API response, an in-game observation.
   - Whether a companion change in `simplified-api/skyblock` is needed, and a link to it.
   - Confirmation that `python scripts/generate_index.py` ran and its output is in the commit.

4. **Respond to review feedback.** PRs may go through one or more rounds of review before being merged.

### What gets reviewed

- **The index is in the commit.** The PR check enforces it, but a PR that needs a second push to add it is a PR that regenerated after review started.
- **The diff is the change.** Reformatting noise around a one-line correction blocks a merge - not because the data is wrong, but because nobody can see whether it is.
- **Sourcing.** A number changed without a stated source is not reviewable. Say where it came from.
- **Schema stability.** A field added, renamed or retyped inside `v1/` is a breaking change for every consumer binding it. If the shape has to change, it goes in `v2/`.

## Reporting Issues

Use [GitHub Issues](https://github.com/simplified-api/skyblock-data/issues) to report incorrect data or request a new table.

When reporting bad data, include:

- **The file and the entry id**
- **The current value and the correct one**
- **Where the correct value comes from** - a source is what makes the report actionable
- **The game version or date** you observed it, if the value changes over time

When reporting a consumer-side failure, include the `content_sha256` your consumer holds for the file alongside the one currently in `index.json` - a mismatch means a stale cache rather than bad data.

## Repository Architecture

```
skyblock-data/
├── data/v1/
│   ├── index.json          # generated; the sole entry point for consumers
│   ├── items/              # accessories, bits_items, item_categories, items, pet_items
│   ├── mobs/               # bestiary_*, mob_types
│   ├── modifiers/          # stats, enchantments, reforges, gemstones, bonus_*
│   ├── player/             # collections, essences, minions, pets, skills, slayers
│   └── world/              # regions, zones, mayors, keywords, fairy_souls, ...
├── scripts/generate_index.py
└── .github/workflows/regenerate-index.yml
```

### How a change reaches a consumer

```
edit data/v1/<cat>/<table>.json
  -> python scripts/generate_index.py     # recomputes content_sha256, bytes, count
    -> commit both, open PR
      -> CI --check                       # fails if the index is stale
        -> merge to master
          -> consumer re-fetches index.json, sees a moved digest, re-fetches that file
```

There is no release and no version bump. `master` is what consumers read.

### Why the generator refuses so much

The manifest is the only thing standing between a data edit and a consumer's binder, and every failure it prevents is silent downstream:

| Refusal | What it would otherwise be |
|---------|----------------------------|
| Unregistered table | A file nobody loads, because it is in no index |
| Stale mapping | An index entry naming a file that 404s |
| Orphan extra | Hand-maintained entries silently dropped at merge time |
| Duplicate primary or extra | One of two files winning arbitrarily |

## Legal

By submitting a pull request, you agree that your contributions are licensed under the [Apache License 2.0](LICENSE.md), the same license that covers this repository.

Data content is derived from public Hypixel API responses and community knowledge. **Do not contribute data obtained by scraping private endpoints, from another project's proprietary dataset, or from any source whose terms forbid redistribution.** Individual entry-level copyrights, where they exist, belong to their respective owners; the repository as a compilation is licensed under Apache 2.0.

Hypixel and SkyBlock are properties of Hypixel Inc.; Minecraft is a trademark of Mojang AB. This repository is independent and is not affiliated with, endorsed by, or sponsored by either.
