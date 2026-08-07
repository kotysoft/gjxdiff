# gjxdiff — Quick Reference

```
gjxdiff [OPTIONS] <FILE_A> <FILE_B>
gjxdiff --about
gjxdiff --credits
gjxdiff --completions <bash|zsh|fish>
```

`-` as one input reads that side from standard input. Not both sides.
An input that is not a regular file (process substitution, named pipe, device)
is copied into the run's temp directory first and counts against `--max-temp`;
`gjxdiff <(cmd) <(cmd)` works, and messages keep the name you typed.
Terminal stdout renders the human view; a pipe or redirect carries the machine
NDJSON report. All progress and disclosures go to stderr.

---

## Flags by task

### Choosing the output

| Flag | Effect |
|------|--------|
| `-H`, `--human` | Human view even when piped |
| `--ndjson` | Machine NDJSON report even on a terminal |
| `--stat` | Only the stat footer, no per-record output |
| `--color auto\|always\|never` | Human-view color. Default `auto`; `NO_COLOR` respected under `auto` |
| `-q`, `--quiet` | No output on either stream; the exit code answers |
| `--max-diffs N` | Emit at most N records. Truncates output only — totals and exit code are complete |

### Pairing records

| Flag | Effect |
|------|--------|
| `--key FIELDS` | Force record identity. Comma-separated, compound, max 16 fields; repeatable; `meta.id` for one level of nesting; `\,` for a literal comma |
| `--key none` | Disable keyed matching; arrays align by order |
| `--record-order` | Also compare record order inside key-matched arrays: every record that must move is listed as a `moved_from` + `moved_to` pair. Off by default — the count is disclosed on stderr either way |
| *(default)* | Autodetect, falling back to order-based alignment |

Keyed matching pairs records by key value and ignores where they sit. Every
keyed run's stderr note states how many records must move to reconcile the two
record orders — the minimum number that have to be relocated, so one record
jumping over 6000 others is `1`. `--record-order` puts those records in the
report; without it the stat footer's move count stays 0 and carries
`(+N reordered, not listed)`. A displaced item that is not an object or array is
counted but never listed, so with the flag the note reads `N of M record(s)
listed as moved` and the clause keeps the remainder. Where several sets of the
same size would each reconcile the order, one is reported: the count is stable,
which records carry it is not.

### Narrowing

| Flag | Effect |
|------|--------|
| `--ignore PATTERNS` | Exclude fields/paths from both sides. Repeatable, comma-split |
| `--path PATH` | Compare only this subtree. Single occurrence, anchored, no wildcards |

Ignore pattern forms: `ts` (bare name, any depth) · `$.meta.ts` (anchored) ·
`items[0].v` (exact position) · `$.arr[*].ts` (any position) · `$.cfg.*` (any member)
· `$["dotted.name"]` (quoted) · `$[""]` (empty name).

`--path` forms: `$.data.items[3].users` · `$[7]` (NDJSON record 7 = line 8) ·
`$["dotted.name"]` · `$` (whole document).

### Large inputs

| Flag | Default | Effect |
|------|---------|--------|
| `--memory-limit SIZE` | Fixed 4 GiB | Heap budget, the same on a laptop and a build server. `512M`, `2G`, or plain bytes. Used verbatim, never clamped. Mapped file pages are outside the budget (but still count against a container's limit). Raise it to spend a big machine: bigger budget, bigger alignment cap and pools, fewer coarse regions. A separate guard aborts the run if the machine or a container memory limit cannot supply it — it never changes output |
| `--max-temp SIZE` | Unlimited | Cap on total temp bytes for the run — a backstop against runaway disk use, not a meter. Measured peaks ranged from roughly 0.15x to 2.1x of the combined input size, driven mainly by how many records the inputs pack per byte (many tiny records high, fewer larger records low), with difference density secondary — no reliable rule from input size alone. Size a cap by measuring an uncapped run, then keep generous headroom; enforcement is sampled, so a fast spool can pass a tight cap by a lot before the run stops |
| `--temp DIR` | System temp dir | Root for `<DIR>/gjxdiff/run-<pid>-<nonce>`, removed on exit; a kill that outruns cleanup is swept at the next start |
| `--align-cap N\|unlimited` | Budget / 640, min 500000 (6710886 by default) | Max direct children aligned exactly; larger containers become coarse regions. Also raises the work-stack limit to twice the cap |
| `--large-arrays coarse\|exact` | `coarse` | `exact` is planned; passing it exits 2 |

### Patch export

| Flag | Effect |
|------|--------|
| `--patch FILE` | Also write an RFC 6902 JSON Patch (A into B). Atomic; `[]` for an empty diff |
| `--patch -` | Write the patch to stdout instead of the report |

Refuses with exit 2 (and leaves no file) when a needed record has no concrete RFC
6901 address, or when keyed pairing absorbed a record reorder — whether the
reorder is the only difference or sits alongside value changes, and with or
without `--record-order`. Retry with `--key none` for a positional patch. A
representation-only difference is exit 0 with a `[]` patch.

### Filtering and bundles

| Flag | Effect |
|------|--------|
| `--only KINDS` | Emit only these record kinds. Report-only: totals and exit code stay complete |
| `--profile ci\|quick\|thorough` | Flag bundle, applied before explicit flags |

### Progress and info

| Flag | Effect |
|------|--------|
| `--progress auto\|always\|never` | `auto` = TTY status line; `always` = plain CI lines (≤ 1 per stage per ~2 s); `never` = off. `--quiet` overrides |
| `-h`, `--help` | Help (`-h` is the summary) |
| `-V`, `--version` | Version and attribution block |
| `--about` | The same block plus a description and the licensing contact. Runs without input files |
| `--credits` | Third-party open-source license notices. Runs without input files |
| `--completions SHELL` | Print a bash/zsh/fish completion script. Runs without input files |

---

## Exit codes

| Code | Meaning |
|------|---------|
| `0` | Identical — byte-identical, semantically identical, or identical apart from ignored content |
| `1` | Differences found. Includes order-only differences (zero records) and fully suppressed `--only` runs |
| `2` | Error: malformed input, invalid flags, failed patch export, tripped memory or temp budget |
| `130` | Interrupted. Temps removed (a kill can outrun cleanup; next start sweeps); any stdout already written is a truncated prefix |

---

## Meta line (report line 1)

```json
{"gjxdiff":1,"tool":"0.8.3","stability":"draft","a":{...},"b":{...},"filters":{...}}
```

| Key | Value |
|-----|-------|
| `gjxdiff` | Report format version (currently `1`). Gate on this |
| `tool` | gjxdiff version |
| `stability` | `"draft"` until the v1 freeze — tolerate added fields |
| `a`, `b` | `{"name": basename only, "bytes": size, "format": "json"\|"ndjson"}` |
| `filters` | Eight keys, always present: see below |

`filters` keys: `key` · `ignore` · `path` · `align_cap` · `move_cap` · `max_diffs` ·
`only` · `large_arrays`.
`null` / `[]` when inactive. `align_cap` echoes the resolved cap only on runs that
entered a cap-dependent rail (an over-cap container, or a work-stack trip);
`move_cap` echoes the resolved per-side pool cap only on runs where the
move-candidate pool saturated. `large_arrays` is always `"coarse"`.
Every budget-derived limit resolves from the memory budget alone, and the default
budget is a fixed 4 GiB — so the same inputs and flags give the same report on any
machine.

## Record line fields

| Field | Notes |
|-------|-------|
| `i` | 0-based index, stable; keeps the unfiltered index under `--only` |
| `op` | `added` `removed` `changed` `moved_from` `moved_to` `key_renamed` `type_changed` `region_changed` |
| `path` | Wildcard pattern, e.g. `$.users[*].name`; a name containing `.`, `[`, `]`, edge spaces or nothing at all is a quoted step (`$["http.status_code"]`, `$[""]`) and pastes back into `--path` / `--ignore`; `$` is root and the overflow bucket (table capacity: 65535 distinct paths, fixed) |
| `ptr` | RFC 6901 pointer with concrete indices, op-side (A for `removed`/`moved_from`, else B). Omitted when unprovable |
| `a_off`, `a_len` | Byte span in A (`0` for pure additions) |
| `b_off`, `b_len` | Byte span in B (`0` for pure removals) |
| `move_pair` | Moves only: the `i` of the counterpart |
| `flags` | When non-empty: `verified` · `coarsened` · `keyed` |

---

## `--only` kinds

| Kind | Report ops |
|------|------------|
| `add` | `added` |
| `change` | `changed` |
| `coarse` | `region_changed` |
| `move` | `moved_from` + `moved_to` |
| `remove` | `removed` |
| `rename` | `key_renamed` |
| `type` | `type_changed` |

No `order` kind. Under positional alignment a displaced object/array of ≥ 64 bytes
reports as a move pair; displaced scalars and smaller containers report as
add+remove. Keyed arrays absorb reorders: a record that only changed position
produces no records, so `--only move` comes up empty there however many records
moved — the stderr key note gives the count, and `--record-order` puts the pairs
in the report where this filter selects them.

## Profile expansions

| Profile | Expands to |
|---------|-----------|
| `ci` | `--ndjson --color=never --progress=never` |
| `quick` | `--max-diffs 1000 --progress=auto` |
| `thorough` | `--align-cap unlimited --progress=auto` |

Explicit flags always override a profile component. The meta line records the
resolved values, never the profile name.

---

## Conflicts

`--human` vs `--ndjson` · `--stat` vs either · `--patch` vs `--max-diffs` · `--patch`
vs `--only` · `--patch` vs `--profile quick` · `--patch -` vs `--human` / `--ndjson`
/ `--stat` / `--quiet`. Each exits 2 with a message.

## Environment

| Variable | Effect |
|----------|--------|
| `NO_COLOR` | Set and non-empty: disables color under `--color=auto` |

---

## Recipes

```sh
gjxdiff a.json b.json                                   # human view
gjxdiff --ndjson a.json b.json > report.ndjson          # machine report
gjxdiff --stat a.json b.json                            # totals only
gjxdiff --quiet a.json b.json; echo $?                  # exit code only

gjxdiff --profile ci a.ndjson b.ndjson > report.ndjson  # CI gate
gjxdiff --key region,id --ignore ts a.ndjson b.ndjson   # keyed, volatile fields out
gjxdiff --path '$.data.items' a.json b.json             # one subtree
gjxdiff --only add,remove a.json b.json                 # content census
gjxdiff --patch changes.json a.json b.json              # RFC 6902 export
curl -s https://api.example.com/items | gjxdiff - snapshot.json

gjxdiff --ndjson a.json b.json | jq -r 'select(.op) | .op' | sort | uniq -c
gjxdiff --completions bash > /etc/bash_completion.d/gjxdiff
```

---

Free for individuals and organizations under 100 people; commercial license required
for automated use in larger organizations and for embedding in commercial products.
Attribution required: `gjxdiff by Tibor Kovacs (Kotysoft)`. See `LICENSE`.
