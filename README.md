# Probe: `pylock-toml-hash-algorithms`

## Purpose

Exercises UV 0.12.11's PEP 751-conformant multi-algorithm artifact
hash recording in `uv.lock`. Every `[package.sdist]` and
`[[package.wheels]]` block carries two `hash =` lines — one `sha256`
and one `sha512`. Mend UA must parse ALL hash entries per artifact,
not truncate to the first one.

## Pattern

`pylock-toml-hash-algorithms` (added from UV 0.12.11 release,
categories: `lockfile_format`, `checksum_signing`).

## PM version tested

`uv >= 0.12.11`

## Package graph

```
hash-probe 0.1.0
├── certifi 2024.2.2        (direct, registry, leaf)
├── idna 3.7                (direct, registry, leaf)
└── urllib3 2.2.1           (direct, registry, leaf)
```

All three packages are resolved from PyPI (`https://pypi.org/simple`).
Each lockfile entry carries both an sdist block and a wheel block,
each listing two hash algorithms (`sha256` + `sha512`).

## What Mend must detect

1. All three packages present in the output tree.
2. `hashes[]` array for each package contains **two** entries
   (one `sha256:...` and one `sha512:...` per artifact type
   represented in the lockfile).
3. Hash algorithm prefixes preserved verbatim.
4. `source` = `registry` for all three packages.
5. Versions exactly as pinned in `uv.lock`.

## Mend failure modes targeted

| Failure | Symptom in expected-tree comparison |
|---------|-------------------------------------|
| UA reads only first `hash =` per block | `hashes[]` has length 1 instead of 2 |
| UA drops `sha512` entries (unknown algo) | `sha512:...` entries absent |
| UA drops sdist hash when wheel present | Fewer hashes than expected |
| UA re-resolves instead of reading lock | Version differs from `2024.2.2` / `3.7` / `2.2.1` |
| `hashes[]` field absent entirely | Key missing from output node |

## Files

| File | Role |
|------|------|
| `pyproject.toml` | PEP 621 manifest with three direct PyPI deps |
| `uv.lock` | Hand-authored lockfile with multi-algo hashes per artifact |
| `.python-version` | Pins Python 3.11 for Mend version detection |
| `src/hash_probe/__init__.py` | Minimal source stub |
| `expected-tree.json` | Ground truth for downstream comparison |

## Python version detection

Both `.python-version` (value: `3.11`) and `pyproject.toml`
`[project] requires-python = ">=3.11"` are present. Per the Mend
PIP-chain precedence, Mend will use `.python-version`. Both files
declare a compatible Python 3.11.

## Mend config

**Bucket B — no `.whitesource` emitted.**

`python-uv` is a Bucket B plugin: Mend has partial dynamic Python
version detection via `requires-python` and `.python-version`. This
probe does NOT target a version mismatch or branch-routing behavior,
so no `.whitesource` is needed.

The uv tool itself is NOT in the `install-tool` list — the uv
binary version cannot be pinned via `scanSettings.versioning`. The
probe was generated against uv 0.12.11 and documents that in
`pm_version_tested`. Operators must ensure uv >= 0.12.11 is
installed out-of-band if they need to reproduce the exact lockfile
hash format.

## Resolver note (UA behavior)

Mend UA's Python resolver for uv projects flows through the pip
resolver path (see `UV Project Filtering` in the upstream resolver
knowledge). The lockfile is parsed statically; hash fields in
`[package.sdist]` and `[[package.wheels]]` map to the `hashes[]`
array in the output tree. If UA only reads the first `hash =` entry
per TOML table (a common off-by-one in TOML key deduplication), the
multi-algorithm probe will surface the truncation.

This probe was added as an *exploratory* probe — the resolver file
does not explicitly document multi-algorithm hash parsing, so the
downstream comparator should treat any `hashes[]` mismatch as an
exploratory finding rather than a confirmed regression.
