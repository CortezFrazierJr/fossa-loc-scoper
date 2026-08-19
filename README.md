# fossa-loc-scoper

Scope a repo for a FOSSA one-time scan. Point it at a local checkout or a git URL and get two defensible headline numbers — **first-party lines of code** and **overall codebase size** (MB/GB) — plus the bucket-level breakdown to back both up.

Zero dependencies: python3 stdlib only. Standalone from the fossa-cli binary on purpose, for fast iteration.

## Usage

```bash
./fossa-loc-scoper.py /path/to/repo
./fossa-loc-scoper.py https://github.com/org/repo.git      # shallow-clones, counts, cleans up
./fossa-loc-scoper.py /path/to/repo --json                 # machine-readable
./fossa-loc-scoper.py /path/to/repo --exclude 'docs/**' --exclude 'examples/**'
```

## The two headline metrics

**1. First-party code lines** — non-blank, non-comment lines in recognized source languages, excluding vendored trees, build output, generated files, minified assets, data files, and binaries. Comment stripping runs per language family (C-style, hash, dash, HTML); docstrings count as code. The output also prints first-party + vendored for scans whose scope includes vendored code.

**2. Overall codebase size** (since v1.1.0, always on — no flag) — total bytes of the full working tree, auto-scaled to B/KB/MB/GB/TB. VCS/IDE metadata (`.git`, `.idea`, `.vscode` etc.) and symlinks are excluded; `.fossa.yml`/`--exclude`-filtered files still count toward it (they exist in the codebase — they're just outside scan scope, and the output says so). First-party code size and a per-bucket size column back the number up. JSON carries `codebase_size_bytes`, `codebase_size_human`, `first_party_code_size_bytes`, `first_party_code_size_human`, `filtered_bytes`, and `size_note`.

## Sample output

Real run against a public repo (`./fossa-loc-scoper.py https://github.com/twbs/bootstrap.git`):

```
==================================================================
FOSSA one-time scan LOC scope  v1.1.0
target: https://github.com/twbs/bootstrap.git  @ 6177d5f
metric: first-party code lines (non-blank, non-comment)
==================================================================

  HEADLINE  first-party code lines:         31,165
  HEADLINE  codebase size:                 17.95 MB   (working tree, excl. VCS/IDE metadata)

  with vendored included:                    31,456
  (use the second number if the scan scope includes vendored code)
  first-party code size:                    1.28 MB

------------------------------------------------------------------
bucket                           files         lines          size
------------------------------------------------------------------
code                               274        31,165       1.28 MB
vendored                             1           291       9.78 KB
build-output                       105        60,189       8.73 MB
generated                            1            21       1.20 KB
data                                67        24,790     890.39 KB
other                              227        28,816       1.59 MB
binary                             115           n/a       5.46 MB
------------------------------------------------------------------

languages (first-party code):
language                                       files    code lines
------------------------------------------------------------------
JavaScript                                        85        17,930
SCSS                                             121         9,513
HTML                                              14         1,699
TypeScript                                        23         1,096
CSS                                               31           927

top-level dirs (first-party code lines):
js                                                          19,077
scss                                                         7,980
site                                                         4,083
(root)                                                          25
```

Read it the way a scoping call goes: the codebase is **17.95 MB on disk**, but only **1.28 MB / 31,165 lines is first-party code** — the committed `dist/` tree (8.73 MB of build output), binaries, and data files are bucketed out of the headline instead of silently inflating it.

Size fields in `--json`, same bootstrap run (excerpt):

```json
{
  "headline_first_party_code_lines": 31165,
  "codebase_size_bytes": 18817183,
  "codebase_size_human": "17.95 MB",
  "first_party_code_size_bytes": 1344229,
  "first_party_code_size_human": "1.28 MB",
  "filtered_bytes": 0,
  "size_note": "working tree, VCS/IDE metadata (.git, .idea, .vscode etc.) excluded; symlinks skipped; filtered files included in codebase size"
}
```

## Buckets (nothing is silently dropped)

| Bucket       | Contents                                             | In headline?                             |
| ------------ | ---------------------------------------------------- | ---------------------------------------- |
| code         | first-party source in recognized languages           | yes                                      |
| vendored     | node_modules, vendor, third_party, Pods, ...         | no (reported, added in the second total) |
| build-output | dist, build, target, dist-newstyle, .stack-work, ... | no                                       |
| generated    | "DO NOT EDIT"/"@generated" markers, protobuf outputs | no                                       |
| minified     | *.min.js, extreme line lengths                       | no                                       |
| data         | json, yaml, md, xml, lock, csv, ...                  | no                                       |
| other        | unrecognized-extension text files                    | no                                       |
| binary       | null-byte sniff or binary extension                  | no                                       |

## Scoping workflow

1. Run the script on the target repo.
2. Read the top-level dirs table. Repo-specific vendor or fixture dirs the defaults miss get `--exclude 'dir/**'` and a rerun.
3. Quote the headline, or the +vendored total when the scan covers vendored code. The bucket table answers "how did you get that number."
4. `--json` output archives the scope snapshot (includes commit SHA and timestamp).

## fossa-cli alignment

- Reads `.fossa.yml` `paths.only` / `paths.exclude` when present, so the scoped number tracks the scan scope (`--no-fossa-yml` to ignore).
- Exclusion defaults mirror the scan-relevance spirit of fossa-cli's default filters.

## Validation

Cross-checked against `scc` on the fossa-cli repo with matching exclusions: Haskell 74,264 (scoper) vs 73,777 (scc), a 0.66% delta from block-comment edge handling. scc's naive whole-repo view also counts ~130K lines of plain text, DOT, JSON, and Markdown; the scoper buckets those out of the headline, which is the point.

## Known limits

- Comment stripping is a pragmatic state machine (same tradeoff cloc makes): no string-literal awareness, so a `/*` inside a string can miscount a line. Deterministic either way.
- Vendor-dir detection is name-based. Unusual names need the human `--exclude` pass; the top-dirs table makes them visible.
- Jupyter notebooks count as data (JSON containers); Fortran/OCaml/Clojure/Erlang count non-blank lines only.
- URL mode shallow-clones with plain `git`. Git-LFS repos are hazardous both ways: with orphaned LFS gitconfig the clone fails outright; with no LFS config at all the clone succeeds and LFS objects arrive as ~68-byte pointer files, so codebase size and the binary bucket silently undercount. Either way, run LFS repos from a full local checkout.
