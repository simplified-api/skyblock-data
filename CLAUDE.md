# skyblock-data

Data-only repository: 41 JSON tables of Hypixel SkyBlock entity data under `data/v1/`, discovered
through a generated manifest and read over the GitHub Contents API. No build, no artifact, no tests -
the only executable in the tree is `scripts/generate_index.py`.

**The surrounding workspace's Java conventions do not apply here.** Lombok, javadoc style, Concurrent
collections, the exception pattern - none of it reaches a repository with no Java in it. The one
convention that does carry over is that a rule stated here is a rule.

## Gates

`python scripts/generate_index.py --check` is the whole gate, and CI runs the same command. Python
3.8+, standard library only.

It hashes bytes and **does not parse the data**, so malformed JSON passes here and fails at the
consumer. `python -m json.tool <file> > /dev/null` is the missing half.

Run the generator in the same commit as any edit under `data/v1/`. A PR whose index is stale fails
its own check.

## The digest is over working-tree bytes

`content_sha256` is taken over the file exactly as stored on disk, which makes three otherwise
cosmetic things load-bearing:

- `.gitattributes` forces `* text=auto eol=lf`. With `core.autocrlf=true` a Windows checkout hashes
  differently from what Linux CI computes and **every** check fails, not one.
- The generator writes with `write_bytes`, never `write_text` - the latter translates `\n` to `\r\n`
  on Windows and produces an index that disagrees with CI's.
- Output is `json.dumps(..., indent=2, sort_keys=True)` plus a trailing newline. Key order is
  alphabetical, which is why `commit_sha` sits at the top of the document rather than `version`.

A diff where every line of a file changed is a line-ending or a reformat, not a data change. Check
`git diff --stat` before believing one.

## Three things move together

A table is a file, a mapping and a Java class, and the generator aborts rather than emitting a
partial index when they disagree.

| Refusal | Trigger |
|---|---|
| `no entry in MODEL_CLASS_BY_TABLE` | a file whose stem is not registered |
| `entries with no matching primary file` | a registered table whose file is gone |
| `orphan extra` | an `_extra` with no primary |
| `duplicate primary` / `duplicate extra` | two files for one `(category, table)` |

`MODEL_CLASS_BY_TABLE` is a dict rather than a derived mapping because English plurals are irregular:
`bestiary_categories` to `BestiaryCategory`, `essences` to `Essence`, `mixins` to `Mixin`. Every rule
that would replace it is wrong more often than the dict is tedious.

The class it names must exist in `simplified-api/skyblock` under `dev.sbs.skyblockdata.model`.
`master` there must connect against `master` here, so a table lands with or after its model and never
before.

## What the manifest means

- **`count` is tables, not files.** 41 tables, 42 files - `items_extra.json` folds into its primary's
  entry as three fields rather than getting one of its own.
- **`category` is presentational.** Consumers read `index.json` and never walk directories, so moving
  a file between categories changes its `path` and nothing else about how it is found.
- **`generated_at` and `commit_sha` are excluded from the `--check` comparison**, and write mode
  compares content before writing. A regeneration with no content change prints `already in sync, not
  rewriting` rather than churning a timestamp into every commit.
- **`v1/` is a schema-version boundary.** A breaking shape change ships as `v2/` alongside; `v1/` is
  never mutated in place. `DATA_VERSION` in the generator pins which tree it walks.

## An empty table ships as `[]`

`pet_items.json` and `fairy_souls.json` are empty arrays, not absent files. A registered table with no
file aborts the generator, so the empty array is how a table stays declared and unpopulated. Deleting
one to "clean up" breaks the index; the consumer-side test for `FairySoul` asserts non-null rather
than non-empty for the same reason.

## Extras

`<table>_extra.json` is merged into its primary at load time. It exists where a primary is
bulk-generated from an upstream dump that has never carried certain entries - `items_extra.json`
holds the anniversary balloon hats, which a regeneration of `items.json` would drop every time.

An extra has no model class and no index entry of its own. Adding one without its primary is the
`orphan extra` abort.

## items.json is over the envelope cap

`data/v1/items/items.json` is ~7 MB. The GitHub Contents API returns a base64 envelope **capped at
1 MB** unless the request carries `Accept: application/vnd.github.raw+json`. That single file is why
the consumer's read contract pins the raw media type and why the read and write surfaces in
`simplified-api/github` cannot share one client - a consumer that omits it fails on this one file and
succeeds on the other forty.

A full cold load is 42 requests: the manifest plus 41 files. Unauthenticated GitHub allows 60 per hour
per IP, so a consumer without a token gets roughly one load per hour.

## CI

`.github/workflows/regenerate-index.yml`, triggered only by `data/v1/**`,
`scripts/generate_index.py`, and the workflow itself.

- **Pull request** - `--check`. A stale index fails, the contributor regenerates locally, and the
  generated diff stays visible in review.
- **Push to master** - write mode, auto-committing a changed index as `github-actions[bot]`.

The push job exists to catch a squash-merge that lost the regenerated index. It is a backstop, not a
reason to skip regenerating in the PR.

## Decisions that stay closed

- Do not hand-edit `data/v1/index.json`. A hand-written digest that happens to be wrong is worse than
  a stale one, because the check compares content rather than intent.
- Do not derive `model_class` from the file name. The plurals do not permit it.
- Do not delete an empty table's file. `[]` is the declaration.
- Do not reformat a data file to make an edit. The digest is over the bytes and the diff is the
  review.
- Do not change a field's name, shape or type inside `v1/`. Every consumer binds it; that is what
  `v2/` is for.
- Do not relax `.gitattributes` or the `write_bytes` call. Both exist because the digest must match
  across platforms, and neither failure is local to the file that caused it.
