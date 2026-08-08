# SkyBlock Data Corpus

Versioned JSON corpus of Hypixel SkyBlock entity data - items, mobs, progression systems, stat modifiers and world content - served straight out of git and consumed over the GitHub Contents API. 41 tables, one file each, discovered through a generated manifest.

> [!IMPORTANT]
> This is a **data-only repository**. There is no build, no package and no artifact - consumers read the files over HTTP at runtime. The Java conventions of the surrounding workspace do not apply here; the only executable thing in the tree is one dependency-free Python script.

## Table of Contents

- [Features](#features)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Repository Layout](#repository-layout)
  - [Consuming the Data](#consuming-the-data)
- [The Index](#the-index)
- [Data Files](#data-files)
  - [Extras](#extras)
  - [Versioning](#versioning)
- [Determinism](#determinism)
- [Scripts](#scripts)
- [Continuous Integration](#continuous-integration)
- [Repository Structure](#repository-structure)
- [Contributing](#contributing)
- [License](#license)

## Features

- **One manifest, no hard-coded paths** - `data/v1/index.json` names every file, its table, its Java model class and its digest
- **SHA-256 change detection** - a consumer re-fetches the manifest, diffs `content_sha256`, and downloads only what moved
- **Registered tables only** - the generator refuses to emit an index for a file with no model mapping, and refuses to keep a mapping with no file
- **Byte-identical across platforms** - forced LF endings plus a byte-writing serializer, so a Windows contributor and Linux CI compute the same digests
- **Self-healing CI** - pull requests fail on a stale index; pushes to `master` regenerate and commit it
- **No dependencies** - Python 3.8+ standard library, no virtualenv, no lockfile

## Getting Started

### Prerequisites

| Requirement | Version | Notes |
|-------------|---------|-------|
| [Python](https://www.python.org/) | **3.8+** | Standard library only - no virtualenv, no `pip install` |
| [Git](https://git-scm.com/) | 2.x+ | For cloning and contributing |

Nothing else. There is no build system and no test suite.

### Repository Layout

```
data/
└── v1/
    ├── index.json               # generated manifest - the entry point
    ├── items/                   # 5 tables - item registry, categories, accessories
    ├── mobs/                    # 4 tables - monster taxonomy and bestiary
    ├── modifiers/               # 19 tables - stats, enchantments, reforges, bonuses
    ├── player/                  # 6 tables - progression systems
    └── world/                   # 7 tables - regions, zones, events
```

Category directories are **presentational only**. Consumers read `index.json` and never walk the tree, so moving a file between categories changes its `path` and nothing else about how it is found.

### Consuming the Data

```
GET data/v1/index.json                       # once
  -> for each entry, compare content_sha256 against your cache
  -> GET entry.path                          # only for the ones that moved
  -> bind into entry.model_class
```

The reference consumer is [simplified-api/skyblock](https://github.com/simplified-api/skyblock): `GitHubIndexProvider` fetches the manifest and holds it, `GitHubFileFetcher` pulls each file, and `RemoteJsonSource` binds the result into an H2-backed JPA repository.

> [!WARNING]
> `data/v1/items/items.json` is over **7 MB**. The GitHub Contents API returns a base64 envelope capped at 1 MB unless the request carries `Accept: application/vnd.github.raw+json`, so any consumer that omits that media type fails on this one file and succeeds on the other forty.

> [!TIP]
> Unauthenticated GitHub allows 60 requests per hour per IP; a full cold load spends 42 of them (the manifest plus 41 files). Send a personal access token and the budget becomes 5000.

## The Index

`data/v1/index.json` is generated, never hand-edited. Every entry names a file and the model class that binds it.

```json
{
  "version": 1,
  "generated_at": "2026-08-07T19:00:00Z",
  "commit_sha": "46991d526775319fc79eb8518bf9a85ea4ed29f3",
  "count": 41,
  "files": [
    {
      "path": "data/v1/items/items.json",
      "category": "items",
      "table_name": "items",
      "model_class": "dev.sbs.skyblockdata.model.Item",
      "content_sha256": "f9e8d7...",
      "bytes": 7102271,
      "has_extra": true,
      "extra_path": "data/v1/items/items_extra.json",
      "extra_sha256": "deadbeef...",
      "extra_bytes": 247
    }
  ]
}
```

| Field | Meaning |
|-------|---------|
| `count` | Number of **tables** (41), not files - an extra does not add to it |
| `path` | Repo-root-relative, forward slashes, exactly what a consumer should request |
| `category` | The directory the file happens to sit in; presentational |
| `table_name` | Matches the `@Table(name = ...)` of the Java model |
| `model_class` | Fully-qualified class in `simplified-api/skyblock` |
| `content_sha256` | Lowercase hex digest of the file bytes **as stored on disk** |
| `bytes` | File size on disk |
| `has_extra` | Whether an `_extra` companion exists; the three `extra_*` fields appear only when it does |

`generated_at` and `commit_sha` are informational. Both are excluded from the `--check` comparison, so a regeneration that finds no content change is a no-op rather than a diff.

## Data Files

Each primary file is a JSON array of entities keyed by natural id. The file stem is the table name.

| Category | Tables |
|----------|--------|
| `items` | `accessories`, `bits_items`, `item_categories`, `items`, `pet_items` |
| `mobs` | `bestiary_categories`, `bestiary_families`, `bestiary_subcategories`, `mob_types` |
| `modifiers` | `bonus_armor_sets`, `bonus_enchantment_stats`, `bonus_item_rarities`, `bonus_item_stats`, `bonus_pet_ability_stats`, `bonus_reforge_stats`, `brews`, `enchantments`, `gemstones`, `hot_potato_stats`, `hotm_perks`, `mixins`, `potion_groups`, `potions`, `powers`, `reforges`, `shop_perks`, `stat_categories`, `stats` |
| `player` | `collections`, `essences`, `minions`, `pets`, `skills`, `slayers` |
| `world` | `fairy_souls`, `keywords`, `mayors`, `melody_songs`, `regions`, `zodiac_events`, `zones` |

`pet_items.json` and `fairy_souls.json` are currently empty arrays. They are registered and shipped rather than absent, because a registered table with no file aborts the generator - an empty array is how a table declares itself known and unpopulated.

### Extras

`<table>_extra.json` is an optional companion merged into its primary at load time. Extras exist for entries that are maintained separately from a bulk-generated primary - `items_extra.json` carries the anniversary balloon hats that no upstream dump contains.

An extra has no model class of its own and no index entry of its own; it appears as three fields on its primary's entry. An extra without a matching primary aborts the generator as an orphan.

### Versioning

`v1/` is a **schema-version boundary**. A breaking schema change ships as `v2/` alongside it, never as a mutation of `v1/` in place - existing consumers keep reading `v1/index.json` until they move deliberately.

## Determinism

`content_sha256` has to match bit-for-bit between a Windows contributor and Linux CI, or every pull request fails its own check. Three rules hold that:

- **`.gitattributes` forces `* text=auto eol=lf`.** Without it, `core.autocrlf=true` on Windows produces CRLF working-tree files that hash differently from what CI sees.
- **The generator writes with `write_bytes`, not `write_text`.** `write_text` on Windows would translate `\n` to `\r\n` and produce an index that differs from the one Linux CI writes.
- **Output is `indent=2, sort_keys=True` plus a trailing newline.** Key order is therefore alphabetical, which is why `commit_sha` sorts to the top of the document rather than `version`.

## Scripts

```bash
python scripts/generate_index.py                     # regenerate data/v1/index.json
python scripts/generate_index.py --check             # verify it is in sync; exit 1 if stale
python scripts/generate_index.py --repo-root <dir>   # run against a tree elsewhere
```

Write mode compares content first and prints `already in sync, not rewriting` rather than churning the timestamp when nothing moved.

The generator aborts, rather than emitting a partial index, on any of:

| Condition | Message |
|-----------|---------|
| A file whose table has no `MODEL_CLASS_BY_TABLE` entry | `no entry in MODEL_CLASS_BY_TABLE` |
| A registered table with no file on disk | `entries with no matching primary file` |
| An `_extra` with no primary | `orphan extra` |
| Two primaries or two extras for one table | `duplicate primary` / `duplicate extra` |

## Continuous Integration

`.github/workflows/regenerate-index.yml` runs on changes to `data/v1/**`, the generator, or the workflow itself.

| Event | Mode | Outcome |
|-------|------|---------|
| Pull request | `--check` | Fails if the committed index is stale; the contributor regenerates and pushes, and the diff stays visible in review |
| Push to `master` | write | Regenerates and auto-commits as `github-actions[bot]` if anything changed |

The push job exists to catch squash-merges that dropped the regenerated index, not to excuse skipping it in a PR.

## Repository Structure

```
skyblock-data/
├── data/
│   └── v1/
│       ├── index.json                    # generated manifest
│       ├── items/  mobs/  modifiers/  player/  world/
├── scripts/
│   └── generate_index.py                 # the only executable in the tree
├── .github/workflows/regenerate-index.yml
├── .gitattributes                        # forces LF; load-bearing for the digests
└── LICENSE.md  README.md  CONTRIBUTING.md  CLAUDE.md
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for how to add or edit data, regenerate the index, and open a pull request.

## License

This project is licensed under the **Apache License 2.0** - see [LICENSE](LICENSE.md) for the full text.

Data content is derived from public Hypixel API responses and community knowledge. Individual entry-level copyrights, where they exist, belong to their respective owners; the repository as a compilation is licensed under Apache 2.0.

Hypixel and SkyBlock are properties of Hypixel Inc.; Minecraft is a trademark of Mojang AB. This repository is independent and is not affiliated with, endorsed by, or sponsored by either.
