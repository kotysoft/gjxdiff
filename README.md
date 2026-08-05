# gjxdiff

Command-line structural diff for JSON and NDJSON files. Built for files that are
larger than your RAM. Written in Rust. Runs on Linux x86-64 — one static binary,
no runtime, no dependencies; works on any distribution and in containers.

The name stands for GiantJSON x86 DIFF: gjxdiff started as a side product of
[GiantJSON Viewer+](https://giantjson.com), an Android app for working with
multi-gigabyte JSON files on a phone. The same problem exists on servers, so
the diff idea got its own standalone tool.

## Features

- JSON and NDJSON, any mix; one side may be stdin (`-`)
- Any file size — inputs are memory-mapped, never loaded into RAM
- Fixed memory budget, default 4 GiB (`--memory-limit`); working state on disk
- Detects: added / removed / changed values, renamed keys, type changes,
  moved subtrees
- Three outputs: NDJSON machine report, color human view, RFC 6902 JSON Patch
- Record pairing by key: `--key`, compound keys, autodetection
- Narrowing: `--path` (one subtree), `--ignore` (volatile fields)
- Report filtering: `--only`; flag presets: `--profile ci|quick|thorough`
- Deterministic: byte-identical report on any machine
- Patch export is all-or-nothing — never a silently partial patch
- Every skip or coarsening disclosed on stderr; exit codes made for CI
- Container-aware memory guard: aborts cleanly instead of getting OOM-killed
- Temp files removed on every exit path, including Ctrl-C
- One static binary, ~2 MB, no dependencies

## Limitations

- Linux x86-64 only. No Windows, macOS or ARM builds.
- Single-threaded by design. A large diff takes the time it takes.
- Binary-only, proprietary. Free for individuals and organizations under 100
  people; see [LICENSING.md](LICENSING.md).
- Very large containers above the alignment cap are reported as one coarse
  region each (disclosed on stderr; the cap rises with `--memory-limit` or
  `--align-cap`).
- Moves are reported only for whole containers of at least 64 bytes. Smaller
  moved items appear as remove + add.
- At most 65,535 distinct paths per report.
- Pre-1.0: the report format and flags may still change between minor
  releases.
- It is a comparison tool, not a validator or repair tool. Malformed input is
  an error.

## Examples

```sh
gjxdiff old.json new.json                  # human view in the terminal
gjxdiff old.json new.json > report.ndjson  # machine report to a file
gjxdiff --key id old.ndjson new.ndjson     # pair NDJSON records by "id"
gjxdiff --ignore updated_at a.json b.json  # skip a noisy field
gjxdiff --patch out.json a.json b.json     # RFC 6902 patch from a to b
curl -s https://api.example.com/items | gjxdiff - snapshot.json
```

Exit codes: `0` identical, `1` differences found, `2` error, `130` interrupted.

## Install

Download the release from [`dist/0.8.1/`](dist/0.8.1/), verify the checksum,
put the binary on your `PATH`. Step-by-step instructions, man page and shell
completion setup: [INSTALL.md](INSTALL.md).

## Documentation

- [INSTALL.md](INSTALL.md) — download, verify, install.
- [LICENSING.md](LICENSING.md) — who can use it free, in plain language.
- [doc/MANUAL.md](doc/MANUAL.md) — full user manual, including a
  troubleshooting section that explains every disclosure the tool can print.
- [doc/QUICKREF.md](doc/QUICKREF.md) — one-page cheat sheet.
- [doc/gjxdiff.1](doc/gjxdiff.1) — man page (`man -l doc/gjxdiff.1`).
- [RELEASENOTES](RELEASENOTES) — what changed in each release, and the known
  limits.
- `gjxdiff --help` and `gjxdiff --about`.

## License

Proprietary. Free for individuals and organizations under 100 people, with
attribution. Source code is not distributed. Summary in
[LICENSING.md](LICENSING.md); the [LICENSE](LICENSE) file governs.

Copyright (C) 2026 Tibor Kovacs (Kotysoft). Contact: support@giantjson.com
