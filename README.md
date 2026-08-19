# fossa-loc-scoper

Scope a repo's lines of code for a FOSSA one-time scan. Point it at a local checkout or a git URL, get a defensible number plus the breakdown to back it up.

Zero dependencies: python3 stdlib only. Standalone from the fossa-cli binary on purpose, for fast iteration.

## Usage

```bash
./fossa-loc-scoper.py /path/to/repo
./fossa-loc-scoper.py https://github.com/org/repo.git      # shallow-clones, counts, cleans up
./fossa-loc-scoper.py /path/to/repo --json                 # machine-readable
./fossa-loc-scoper.py /path/to/repo --exclude 'docs/**' --exclude 'examples/**'
```

## The metric

Headline = first-party code lines: non-blank, non-comment lines in recognized source languages, excluding vendored trees, build output, generated files, minified assets, data files, and binaries. Comment stripping runs per language family (C-style, hash, dash, HTML); docstrings count as code. The output also prints first-party + vendored for scans whose scope includes vendored code.

As of v1.1.0 the default output (no flag) also reports **overall codebase size** auto-scaled to B/KB/MB/GB — the full working tree excluding VCS metadata (`.git` etc.) and symlinks, with `.fossa.yml`/`--exclude`-filtered files still counted toward it — plus first-party code size and a per-bucket size column. JSON gains `codebase_size_bytes`, `codebase_size_human`, `first_party_code_size_bytes`, `first_party_code_size_human`, and `filtered_bytes`.

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
