# gjxdiff — User Manual

Structural diff for JSON and NDJSON files of any size.

This manual documents observable behavior: what you pass in, what comes out, and
what the tool guarantees. For a one-page summary see [QUICKREF.md](QUICKREF.md);
for the terse reference see the man page (`gjxdiff.1`).

---

## Contents

1. [Introduction and guarantees](#1-introduction-and-guarantees)
2. [Quick start](#2-quick-start)
3. [Inputs](#3-inputs)
4. [The machine report](#4-the-machine-report)
5. [The human view](#5-the-human-view)
6. [Pairing and keys](#6-pairing-and-keys)
7. [Narrowing the comparison](#7-narrowing-the-comparison)
8. [Large inputs](#8-large-inputs)
9. [Patch export](#9-patch-export)
10. [Filters and profiles](#10-filters-and-profiles)
11. [Progress and automation](#11-progress-and-automation)
12. [Troubleshooting](#12-troubleshooting)
13. [Limits at a glance](#13-limits-at-a-glance)
14. [License and attribution](#14-license-and-attribution)
15. [Planned](#15-planned)

---

## 1. Introduction and guarantees

`gjxdiff` compares two JSON or NDJSON documents structurally. It reports added,
removed and changed values, renamed keys, type changes, moved subtrees, and — for
containers too large to align exactly — coarse changed regions. It does this
without ever loading either file into memory, so the same command works on a
200-byte config and a 50 GB database export.

### A first comparison

Two versions of a small service config:

```
// a.json                                   // b.json
{                                           {
  "service": "orders-api",                    "service": "orders-api",
  "image": "registry.local/orders-api:2.4.1", "image": "registry.local/orders-api:2.5.0",
  "port": 8080,                               "port": 9090,
  "replicas": 2,                              "replicas": 4,
  "features": {                               "features": {
    "rate_limit": true,                         "rate_limit": true,
    "retries": 3,                               "retries": 3,
    "legacy_auth": true                         "tracing": false
  },                                          },
  "regions": ["eu-west", "us-east"]           "regions": ["eu-west", "us-east", "ap-south"]
}                                           }
```

One command, no flags:

```
$ gjxdiff a.json b.json
gjxdiff 0.8.3
a: a.json (json, 233 bytes)
b: b.json (json, 242 bytes)

$
  ~ image: "registry.local/orders-api:2.4.1" -> "registry.local/orders-api:2.5.0"
  ~ port: 8080 -> 9090
  ~ replicas: 2 -> 4
$.features
  - legacy_auth: true
  + tracing: false
$.regions
  + [2]: "ap-south"

3 changed, 2 added, 1 removed, 0 moved
3 container pairs compared
```

How to read it:

- Records are grouped under a **bold header naming their parent** — `$` is the
  document root, `$.features` the `features` object.
- `~` (yellow) is a changed value, shown as `old -> new`; `+` (green) is added;
  `-` (red) is removed. The full mark table is in
  [§5, The human view](#5-the-human-view).
- The last two lines are the totals footer. The exit code is `1` because
  differences were found; identical files exit `0`.

This colored, grouped rendering is the **human view**, and it appears when
stdout is a terminal. Pipe or redirect the same command and it becomes the
**machine report** instead — NDJSON, one JSON object per record, with an
RFC 6901 pointer and exact byte offsets into the input files
([§4](#4-the-machine-report)). Same comparison, two presentations.

### The promises

The rest of this manual documents behavior feature by feature. These guarantees
hold everywhere:

- **It never loads a whole input file into RAM.** Inputs are memory-mapped and read
  in streams.
- **Memory is bounded by a budget, not by input size.** The default budget is a fixed
  4 GiB; `--memory-limit` overrides it. Work that does not fit in the budget goes
  to temporary files on disk. If the budget would be exceeded anyway, the run aborts
  with a clear error rather than exhausting the machine. The default is the same on
  every machine, which is what makes reports reproducible — and also means a large
  machine is only used if you say so ([§8](#8-large-inputs)).
- **Output is never accumulated in memory.** Diff records spool to disk as they are
  found, and the report is written to stdout when the comparison completes. The meta
  line is first in stream order, not first in time — expect the whole report near
  the end of the run, not a trickle during it.
- **It is deterministic.** The same inputs and flags produce a byte-identical report,
  run after run, on this machine and on any other. Because the default budget is a
  fixed number rather than a share of free RAM, no internal limit — and so nothing in
  the report — depends on how much memory happens to be available. The one thing the
  environment still decides is whether the run *completes*: a machine or container
  that cannot supply the budget aborts with an error (see [§8](#8-large-inputs))
  instead of quietly reporting something different.
- **Temporary files are cleaned up** — on success, on error, on panic, and on
  Ctrl-C. Two narrow cases can outrun the cleanup: a second SIGINT/SIGTERM arriving
  within the shutdown's first milliseconds, and the kernel killing the run (exit
  code 137) inside a container too small for the workload while an abort is already
  under way. Whatever such a kill leaves behind — like directories left by a killed
  process (SIGKILL, power loss) — is swept at the next start.
- **Nothing is silently partial.** Anything the comparison skips, absorbs or coarsens
  — keyed pairing ignoring record order, ignored fields, coarsened regions,
  suppressed record kinds — is disclosed on stderr, and the exit code always reflects
  the full comparison.
- **Malformed input is an error, not a crash.** Parse errors exit 2 and name the file
  and the byte offset.
- **Terminal output is safe.** Every stderr line, and every value shown in the human
  view, passes an escaping pass: control bytes and bidirectional override characters
  derived from file content or the command line are rendered visibly and can never
  inject terminal control sequences.

### What is compared

Comparison is semantic, not textual. Whitespace, number representation and string
escape representation normalize — `1.0` and `1.00`, `"A"` and `"A"` compare
equal. Array order is part of equality unless keyed pairing takes over for that
array (see [Pairing and keys](#6-pairing-and-keys)).

---

## 2. Quick start

The four ways to ask, from most to least verbose:

```sh
# Human view on a terminal, machine NDJSON when piped.
gjxdiff a.json b.json

# Force the machine report and save it.
gjxdiff --ndjson a.json b.json > report.ndjson

# Totals only.
gjxdiff --stat a.json b.json

# Just the answer.
gjxdiff --quiet a.json b.json; echo "exit=$?"
```

### Scaling up

The same no-flag command on two exports of **five million order records,
555 MB each**, where the second export is sorted differently — so a line-based
`diff` would flag nearly every line:

```
$ gjxdiff orders-monday.ndjson orders-tuesday.ndjson
gjxdiff 0.8.3
a: orders-monday.ndjson (ndjson, 554700225 bytes)
b: orders-tuesday.ndjson (ndjson, 554700227 bytes)

$[=12228932]
  ~ status: "cancelled" -> "shipped"
$[=11441889]
  ~ status: "pending" -> "cancelled"
$[=11978411]
  ~ status: "pending" -> "cancelled"

3 changed, 0 added, 0 removed, 0 moved (+4995557 reordered, not listed; --record-order)
4 container pairs compared
gjxdiff: note: $[*] matched by auto key "order_id" (5000000 items compared by key) — record order not compared; 4995557 record(s) must move to reconcile the two orders (--record-order lists them as moves)
```

Three things happened without being asked for:

- The NDJSON format was detected per file, and each file is treated as an array
  of records ([§3](#3-inputs)).
- A record key, `order_id`, was **autodetected**, so records are matched by key
  value wherever they sit — the re-sort produces no records, and the three real
  changes surface. Group headers show the key value: `$[=12228932]` is the
  record whose `order_id` is 12228932 ([§6](#6-pairing-and-keys)).
- The absorbed reorder is **disclosed**, not hidden: the stderr note names the
  key, and both the note and the summary line carry the number of records that
  must move to reconcile the two orders — `--record-order` lists them as move
  pairs ([§6](#6-pairing-and-keys)). Nothing the comparison skips or absorbs is
  ever silent.

That run finished in about 16 seconds on a small 4-core VM, in bounded memory.
Memory is governed by a budget, not by input size ([§8](#8-large-inputs)).

From here, the flags follow the questions:

```sh
# Volatile fields drowning the diff? Exclude them.            (§7)
gjxdiff --ignore ts,updated_at a.ndjson b.ndjson

# Autodetection picked nothing, or the wrong field? Force it. (§6)
gjxdiff --key region,id a.ndjson b.ndjson

# Only one subtree matters.                                   (§7)
gjxdiff --path '$.data.items' a.json b.json

# CI gate: machine report, no color, no progress.             (§10, §11)
gjxdiff --profile ci a.ndjson b.ndjson > report.ndjson

# Apply the changes elsewhere: RFC 6902 JSON Patch.           (§9)
gjxdiff --patch changes.json a.json b.json
```

### Argument rules

Two positional arguments are required for every comparison. The only flags that run
without them are `--about`, `--credits` and `--completions`.

Print-and-exit flags (`--about`, `--credits`, `--completions`, `--help`, `--version`) run before
run-time validation: combined with flags that would be refused at run time (for
example the unavailable `--large-arrays exact`), they still print and exit 0.
Malformed flag syntax and unknown values remain parse errors (exit 2) in any
combination — `gjxdiff --about --only bogus` exits 2.

---

## 3. Inputs

### Format detection

Format is detected per file, from the file name extension and the content. A JSON
file can be compared against an NDJSON file — the NDJSON side is treated as an array
of records.

NDJSON files are compared record by record. Arrays of records with a detected or
manually set key are matched by key value, order-insensitively.

### Standard input

Pass `-` as one of the two inputs to read that side from standard input:

```sh
curl -s https://api.example.com/items | gjxdiff - snapshot.json
```

Details:

- The stream is buffered to a temporary file under `--temp` before the comparison
  starts. The buffer is run data like everything else in the run directory: removed
  on exit, swept at the next start when a kill outran the cleanup (see
  [§8](#8-large-inputs), Temporary files).
- Format is detected from the content alone. Standard input has no file name, so the
  `.ndjson` / `.jsonl` extension shortcut never applies; a single-record NDJSON
  stream is detected as JSON, exactly as a `.txt` file would be.
- The report and every message name that input `-`. The temporary buffer path never
  appears in any output.
- Both sides cannot be `-` — a stream can only be read once (exit 2).
- If standard input is an interactive terminal, `-` exits 2 with a message instead of
  appearing to hang.
- While the stream is being buffered, the progress line shows bytes read with no
  percentage — the total is unknown until the pipe ends.
- `-` composes with `--patch -`: standard input and standard output are different
  streams.
- Empty input is a format error: `-: format: no JSON root value`, exit 2.

### Streams and process substitution

An input that is not a regular file — a process substitution, a named pipe, a
socket, a device — is a stream, and a stream cannot be memory-mapped or read twice.
Such an input is copied into the run's temporary directory first, and the comparison
then runs on the copy:

```sh
gjxdiff <(curl -s "$URL_A") <(curl -s "$URL_B")
mkfifo feed; producer > feed & gjxdiff feed snapshot.json
```

Details:

- Both sides may be streams at once. Both are read before either is indexed, so a
  producer feeding one side is not left blocked while the other side is compared.
- The copy is run data under `--temp`: it counts against `--max-temp` and is removed
  on exit like every other temporary file. Comparing two 3 GB streams needs room for
  6 GB of copies plus the usual working files, where two regular 3 GB files need
  none of it.
- The report and every message name the input the way you typed it. The copy's path
  never appears in any output.
- Reading a stream is interruptible: Ctrl-C ends the run with exit 130 and removes
  the partial copy. An input that never ends (`/dev/zero`, `yes`) is stopped by
  `--max-temp` if you set one; otherwise it keeps reading, like `cat` would.
- A named pipe with no writer waits for one, as any reader does. The progress line
  shows `reading input` with the bytes taken so far.
- Regular files are never copied — nothing about a normal comparison changes.

### Documented leniency

`gjxdiff` is a diff tool, not an RFC 8259 validator. The following do **not** abort
a comparison:

| Tolerated | Behavior |
|-----------|----------|
| Unescaped control characters in strings | Compared as raw bytes |
| Lone surrogates | Compared as raw bytes |
| Invalid UTF-8 in strings | Compared as raw bytes |
| Leading UTF-8 BOM | Stripped from structural consideration |
| UTF-8 BOM at the start of any NDJSON record line | Stripped, counted, and disclosed on stderr (`UTF-8 BOM stripped on N line(s)`, one note per file) |

String contents are compared as **raw bytes**. Two strings that differ only in escape
form are equal under semantic normalization; two strings whose decoded bytes differ
are not. A BOM difference is not a byte-identical match — it compares equal only
under the same normalization that covers any whitespace change.

In JSON mode, a BOM anywhere other than the very start of the file is a structural
error.

### What is rejected

These are structural errors: exit 2, no report, nothing written.

- Unclosed containers or strings
- Trailing commas
- Unquoted keys
- Non-JSON input

The error line names the file and the byte offset:

```
data-b.json: format: trailing comma at byte 41208
```

The offset is the offending byte, or the byte where the problem became detectable —
for example, the opening byte of a container left unclosed at end of input, or the
`]`/`}` that closes a trailing comma. It is omitted only where no single offending
byte exists (see [Troubleshooting](#12-troubleshooting)).

### Flag value whitespace

Flag values are whitespace-trimmed before parsing. A member name with leading or
trailing whitespace therefore needs the quoted form in `--path` and `--ignore`:

```sh
gjxdiff --path '$["name "]' a.json b.json
```

---

## 4. The machine report

The machine report is NDJSON: one JSON object per line. It is emitted when stdout is
a pipe or a redirect, or when `--ndjson` is passed on a terminal. The bytes are
identical either way.

The report is present whenever stdout carries a report (exit 0 or exit 1, and not
`--quiet`). An empty diff emits the meta line alone. Exit 2 carries no report.

### Schema stability

The meta line carries `"stability":"draft"`. Until the v1 freeze, field names may
change and fields may be added. Gate on the `gjxdiff` field (the format version,
currently `1`), and write consumers that tolerate unknown fields rather than
rejecting them.

### The meta line

The first line is always a meta record. Everything in it is known before comparison
output starts — no totals, no timestamps — so repeat runs are byte-identical, meta
line included. (Two fields are run-derived by design: `filters.align_cap` and
`filters.move_cap` echo a resolved capacity exactly on the runs where that capacity
influenced the records — see the table below.)

```json
{"gjxdiff":1,"tool":"0.8.3","stability":"draft","a":{"name":"a.json","bytes":10485760,"format":"json"},"b":{"name":"b.json","bytes":10493284,"format":"json"},"filters":{"key":null,"ignore":[],"path":null,"align_cap":null,"move_cap":null,"max_diffs":null,"only":null,"large_arrays":"coarse"}}
```

| Field | Type | Meaning |
|-------|------|---------|
| `gjxdiff` | int | Report format version, currently `1`. Its presence on line 1 identifies the meta record. Gate on this. |
| `tool` | string | The `gjxdiff` version that produced the report. |
| `stability` | string | `"draft"` until the v1 schema freeze. |
| `a`, `b` | object | Input identification, one per side. |
| `a.name`, `b.name` | string | **Basename only** — full paths never enter the report. `"-"` for the standard-input side. |
| `a.bytes`, `b.bytes` | int | Size of that input in bytes. |
| `a.format`, `b.format` | string | `"json"` or `"ndjson"`, as detected for that side. |
| `filters` | object | The active comparison-narrowing flags in canonical form. Always all eight keys below. |

The `filters` object duplicates the stderr disclosures, so the report alone is never
silently partial:

| Key | Value |
|-----|-------|
| `key` | `null` (autodetect), `"none"` (keyed matching disabled), or the forced field list in display form, e.g. `"region,meta.id"`. |
| `ignore` | Array of canonical `--ignore` patterns; `[]` when inactive. Bare names first, then anchored paths, in parse order, deduplicated. |
| `path` | The canonical `--path` address string, or `null`. |
| `align_cap` | The explicit `--align-cap` value (a number or `"unlimited"`) when one was passed. Under the default cap it is `null` on runs that never entered a cap-dependent rail, and the **resolved** number on runs that did (an over-cap container, or a work-stack trip — the work stack is twice the cap) — the records then depend on the cap, so the report states which cap produced them. |
| `move_cap` | `null` on runs where the move-candidate pool never saturated. The **resolved** per-side pool cap on runs where it did — candidates past the cap are dropped and their pairs report as plain `removed`+`added` records, so the records depend on it. There is no flag for this pool; it scales from the memory budget alone. |
| `max_diffs` | The `--max-diffs` value, or `null`. |
| `only` | The sorted canonical kind array of an active `--only` filter, e.g. `["add","remove"]`; `null` when inactive. |
| `large_arrays` | The over-cap handling mode. Always `"coarse"` — the only mode that exists in this version. |

Note: `--path $` selects the whole document. Record lines stay byte-identical to an
unscoped run, but the meta line differs (`"path":"$"` instead of `null`).

### Record lines

Every line after the meta line is one diff record. Records appear in engine stream
order, which is roughly — but not strictly — file B byte order: sibling regions can
interleave.

```json
{"i":0,"op":"changed","path":"$.users[*].email","ptr":"/users/12/email","a_off":4192,"a_len":21,"b_off":4192,"b_len":25}
```

| Field | Type | Present | Meaning |
|-------|------|---------|---------|
| `i` | int | always | Record index, 0-based, stable across runs. Under `--only` it keeps the **unfiltered** index. |
| `op` | string | always | One of `added`, `removed`, `changed`, `moved_from`, `moved_to`, `key_renamed`, `type_changed`, `region_changed`. |
| `path` | string | always | Wildcard path pattern of the change site, e.g. `"$.users[*].name"`. Array indices fold to `[*]`. A member name that would collide with the grammar is quoted: `$["http.status_code"]` (see below). `"$"` is the root, and also the overflow bucket (see below). |
| `ptr` | string | usually | RFC 6901 JSON Pointer of the change site with **concrete** array indices. Rooted where `path` is rooted. Omitted — never guessed — when no reliable single-node address exists. |
| `a_off` | int | always | Byte offset in file A. `0` for pure additions. |
| `a_len` | int | always | Byte length in file A. `0` for pure additions. |
| `b_off` | int | always | Byte offset in file B. `0` for pure removals. |
| `b_len` | int | always | Byte length in file B. `0` for pure removals. |
| `move_pair` | int | moves only | On `moved_from` / `moved_to`: the `i` of the counterpart record. |
| `flags` | array | when non-empty | Any of `"verified"`, `"coarsened"`, `"keyed"`. |

#### Names that need quoting

Member names containing `.`, `[` or `]`, names that begin or end with a space, the
empty name and names made only of `*` would read as something else in the path
grammar, so `path` writes them as a quoted step — the same spelling `--ignore` and
`--path` accept:

| In the file | In `path` |
|-------------|-----------|
| `{"http.status_code": 500}` | `$["http.status_code"]` |
| `{"x[*]": [1, 2]}` | `$["x[*]"][*]` |
| `{"": 1}` | `$[""]` |
| `{"a\"b.c": 1}` (name contains `"`) | `$['a"b.c']` |
| `{"status": 500}` (ordinary name) | `$.status` — unchanged |

So a path the report prints can be pasted straight back:

```sh
gjxdiff --ignore '$["http.status_code"]' a.json b.json
```

A name that both needs quoting and contains `"` **and** `'` has no spelling in this
grammar (there are no escapes). It is printed in the double-quoted form so the step
boundaries are still visible, but it cannot be fed back; use `ptr`, which addresses
every name losslessly.

#### About `ptr`

`ptr` is the machine-safe address: standard RFC 6901 escaping (`~` becomes `~0`, `/`
becomes `~1`), tokens are the decoded member names, and array steps carry concrete
ordinals rather than `[*]`. The root record's pointer is `""`. It addresses a single
node where `path` describes a pattern, and it carries every name losslessly,
including the ones the path grammar cannot spell.

One pointer per record, addressing the record's **op-side** instance: file A for
`removed` and `moved_from`, file B for everything else. Under keyed pairing the two
sides' positions can differ; the pointer points where the record's byte span is.

`ptr` is **omitted** rather than guessed when:

- a member name on the op-side chain is duplicated within its object (an RFC resolver
  would take the first occurrence, which may be the wrong node);
- a member name does not decode losslessly (lone surrogate, malformed escape,
  invalid UTF-8) — two distinct raw names could collapse into one token;
- the record landed in the path-pattern overflow bucket, or its stored path exceeds
  the storage limit;
- an array ordinal on the chain could not be proven.

A pointer is never truncated: a shortened pointer would address the wrong node.

#### Path-pattern overflow

The report's path-pattern table holds at most **65,535 distinct paths** besides the
root. The capacity is fixed: it does not scale with `--memory-limit` and no flag
raises it. On a document with more distinct change-site paths than that, the
overflow records carry the aggregated path `"$"` instead of their own and omit
`ptr`. One aggregate stderr note counts them and names the capacity, so they are
never mistaken for root records:

```
gjxdiff: note: pattern table full (65535 distinct paths) — 812 record(s) carry the aggregated path "$" instead of their own (ptr omitted for them)
```

The note counts the records actually shown: `--stat` reads back no records and
prints no overflow note, and a report truncated by `--max-diffs` counts — or, when
the truncation cuts every degraded record, omits — only what was emitted.

#### NDJSON addressing

An NDJSON file is addressed as an array of records: `$[N]` in `path`, `/N/...` in
`ptr`, where N is the record ordinal (0-based). For well-formed NDJSON this equals
the physical line number minus one.

The ordinal follows the scanner's value-counting rules, which matter only in leniency
corners: blank lines do not count; several values on one line each count; a raw
newline inside a string does not split a record.

### Recovering values with the byte spans

The report never carries file content; the byte spans are how a consumer reads the
actual values. `a_off`/`a_len` address file A, `b_off`/`b_len` file B.

- Offsets are **absolute byte offsets into the original input file** on their side.
  They stay absolute under `--path`: a scoped run re-roots `path` and `ptr`, but the
  spans still point into the whole file, so the same extraction code works scoped
  and unscoped.
- A span covers **the value only** — for strings including the enclosing quotes,
  never the member name, the colon or any separator. A container span covers the
  whole `{...}` / `[...]` token. The exception is `region_changed`: a coarse span
  covers the divergent byte range, which is not a single JSON value
  ([§9](#9-patch-export)).
- On a `changed` record the A span is the old value, the B span the new one. Pure
  additions carry `a_off`/`a_len` 0; pure removals `b_off`/`b_len` 0.

Extract one value — here a record with `"b_off":114,"b_len":6` from `b.json`:

```sh
$ dd if=b.json bs=1 skip=114 count=6 status=none
"deux"
```

Pull every changed record's new value out of a report:

```sh
gjxdiff --ndjson a.json b.json > report.ndjson
jq -r 'select(.op=="changed") | "\(.b_off) \(.b_len)"' report.ndjson |
  while read off len; do
    dd if=b.json bs=1 skip=$off count=$len status=none; echo
  done
```

The output is the raw value bytes exactly as they stand in the file — `"beta.example"`,
`9090`, `"deux"` on a three-change config pair. In Python, `f.seek(off); f.read(len)`
on the file opened in binary mode does the same.

### The summary line

The comparison summary goes to **stderr**, not into the report:

```
gjxdiff: 5015 changed, 571 added, 1000 deleted, 12 moved
```

`changed` aggregates `changed`, `key_renamed`, `type_changed` and `region_changed`;
`moved` counts subtree pairs once. The summary's `deleted` is the same record class
the report and the stat footer call `removed` — one counter, two words. Clauses are
appended when relevant:

```
gjxdiff: 5015 changed, 571 added, 1000 deleted, 12 moved; 1571 suppressed by --only
gjxdiff: 5015 changed, 571 added, 1000 deleted, 12 moved (stdout truncated at 1000 of 6598 records)
gjxdiff: 1 changed, 0 added, 0 deleted, 0 moved (+996 reordered, not listed; --record-order)
```

The `(+N reordered, not listed)` clause appears when keyed pairing absorbed a
record reorder: `moved` counts move RECORDS, and a keyed reorder produces none
unless [`--record-order`](#--record-order) is given. `N` is the minimum number of
records that must be relocated — see
[Record order under a key](#record-order-under-a-key). With the flag the listed
moves are in the count itself, so the clause covers only what no listing can
express — displaced scalars — and drops the flag hint, there being nothing left
to switch on.

---

## 5. The human view

When stdout is a terminal, `gjxdiff` renders a human view instead of the machine
report. `--human` / `-H` forces it even when piped (useful with `less -R`).

The view opens with a header block carrying the same information as the machine meta
line:

```
gjxdiff 0.8.3
a: orders-2026-07.ndjson (ndjson, 2684354560 bytes)
b: orders-2026-08.ndjson (ndjson, 2691823104 bytes)
filters: key=order_id · ignore=ts,updated_at
```

The `filters:` line appears only when at least one narrowing flag is active.

### Records

Records are grouped under a header naming their parent, and indented beneath it. The
group header repeats when a group recurs non-contiguously — records arrive in engine
stream order, and sibling regions can interleave.

| Mark | Color | Meaning |
|------|-------|---------|
| `+` | green | added |
| `-` | red | removed |
| `~` | yellow | changed, and `(type changed)`, `(key renamed)`, coarse regions |
| `>` | cyan | `(moved away)` / `(moved here)` |

Changed values render as `old -> new`.

Group headers and array leaves show the most concrete identity available:

- keyed pairing shows the key **value**: `users[=alice]`, compound keys comma-joined
  as `users[=alice,eu]`;
- positional records show the concrete ordinal on their op side: `users[23]`;
- `[*]` only when neither is available.

### Value display

The human view is a display channel; the machine report is the lossless one.

- Values are capped at 160 display characters, with an exact `… (+N bytes)` tail
  where N is the number of raw bytes not shown.
- ESC, all C0 control bytes, DEL and C1 bytes render as `\xNN`.
- Bidirectional controls and line/paragraph separators render as `\u{....}` — an
  invisible direction override may *be* the difference, so it is revealed rather than
  hidden.
- Invalid UTF-8 renders lossily, with U+FFFD as the visible marker.
- Container spans collapse whitespace runs outside strings to one space, so a
  pretty-printed subtree reads on one line.

### Aggregation

Long runs of similar records collapse to one line so a 20,000-item change does not
become 20,000 lines. This affects the human view only — the machine report, the
footer, `--stat` and the summary always count the underlying records.

| Line shape | When |
|------------|------|
| `~ tags: [8 items rewritten]` | Four or more consecutive scalar `changed` records at array positions under one parent array, with no other visible record in that contiguous group. The claim is that the array was fully rewritten, so any add, remove, move, type change or non-scalar change in the group disables it. |
| `~ data: 12431 changed, 300 added, 1 removed` | A contiguous run of 100 or more item-level records of any kind under one parent array. The line claims only counts. Zero parts are omitted. |

Records suppressed by `--only` render nothing, but they still break runs and disable
the full-rewrite claim — a filter can never make `[N items rewritten]` over-claim.

Coarse regions render as counts and a byte range, never as raw file content:

```
~ data[1024..8110004]: coarse region (8108980 -> 7990122 bytes)
```

The range is the A-side byte range; the numbers in parentheses are the per-side byte
counts.

### Truncation and the footer

With `--max-diffs`, a line precedes the footer:

```
… 5598 more record(s) not shown
```

The human view always ends with the stat footer, after a blank line:

```
5015 changed, 571 added, 1000 removed, 12 moved
84213 container pairs compared
3 regions coarsened
412 difference sites suppressed by ignore rules, 96 container pairs equalized
1571 records suppressed by --only
```

The last three lines appear only when relevant: coarsened regions when nonzero, the
ignore line when `--ignore` is active, the `--only` line whenever the filter is
active (including at zero).

`container pairs compared` counts the container pairs the comparison actually
walked: the pair of document roots plus every matched pair of objects or arrays it
had to open on the way to the differences. Pairs whose contents are already known
equal are skipped without being counted, so the figure tracks how much of the
structure diverged, not how big the inputs are — a 20,000-record compare with three
changed records reports `4 container pairs compared` (the root pair plus the three
changed records). Identical inputs report `0`. The line has one fixed form; a run
with a single pair prints `1 container pairs compared`.

### `--stat`

`--stat` prints exactly the footer and nothing per-record. The comparison still runs
in full; only the per-record read-back is skipped. On identical, masked-identical or
order-only runs, a verdict line precedes the footer — `--stat` output is always
identical to the tail of the `--human` output for the same pair.

### Color

`--color=auto|always|never`, default `auto`.

| Value | Behavior |
|-------|----------|
| `auto` | Color only when the human view goes to a terminal. The `NO_COLOR` convention is respected: set and non-empty disables color. |
| `always` | ANSI even when piped (for `less -R`). Beats `NO_COLOR`. |
| `never` | Plain text. |

The machine report is never colored, in any mode. Color is presentation only:
stripping ANSI recovers the colorless bytes exactly.

---

## 6. Pairing and keys

When two arrays are compared, `gjxdiff` has to decide which element on the A side
corresponds to which element on the B side. By default it autodetects a record key,
falling back to order-based (positional) alignment.

With a key, records are matched by key value, order-insensitively. A pure reorder
then produces **no records** — but the file-level verdict is order-sensitive, so the
run still exits 1 and stderr discloses the order-only difference.

Order is still *measured*. Whether the reorder is the only difference or sits
alongside value changes, every keyed run's stderr note says how many records
must move to reconcile the two record orders, and
[`--record-order`](#--record-order) lists them as move pairs. See
[Record order under a key](#record-order-under-a-key).

### `--key FIELDS`

Force record identity globally:

```sh
gjxdiff --key order_id a.ndjson b.ndjson
gjxdiff --key region,id a.ndjson b.ndjson          # compound key
gjxdiff --key region --key id a.ndjson b.ndjson    # the same thing
gjxdiff --key meta.id a.ndjson b.ndjson            # one level of nesting
gjxdiff --key 'odd\,name' a.ndjson b.ndjson        # literal comma in a name
gjxdiff --key none a.ndjson b.ndjson               # disable keyed matching
```

- Comma-separated field names that jointly identify a record. At most 16 fields.
- Repeatable; occurrences accumulate.
- One level of nesting via a dot. `\,` escapes a literal comma inside a field name.
- Fields are forced **globally**: arrays where they resolve are matched by them;
  other arrays fall back to autodetection, then to order-based alignment.
- `--key none`, alone, disables keyed matching entirely — every array aligns by
  order.
- Default: autodetect.

### Key disclosures

Every keyed pairing, every rejected key and every quality degradation is disclosed on
stderr. Examples of what you will see:

```
gjxdiff: note: $.orders[*] matched by auto key "order_id" (1491 items compared by key) — record order not compared
gjxdiff: note: $.orders[*] matched by auto key "order_id" (1491 items compared by key) — record order not compared; 1 record(s) must move to reconcile the two orders (--record-order lists them as moves)
gjxdiff: note: keyed matching disabled (--key none) — arrays align by order only
gjxdiff: note: --key "orderId" matched no array — the field was not found in any record (typo?); autodetect/order-based alignment used instead
gjxdiff: note: auto key "id" at $[*] withdrawn (keyed pairing would report more churn than order-based alignment) — positional alignment used
gjxdiff: note: key candidate "id" rejected (duplicate values) at $[*] — positional alignment used
gjxdiff: note: auto key "id" has duplicate values (2 of 400000 records) at $ — pairing quality degraded
gjxdiff: note: forced key "id" has duplicate values (200 of 200 records) at $ — pairing quality degraded
gjxdiff: note: 37 key-matched array(s) in total (first 10 listed)
```

The matched note names the array by its item pattern — `$.orders[*]` for a nested
array, `$[*]` for the record array an NDJSON file is treated as. A key you pass
with `--key` is a *forced* key; its successful pairings read `matched by manual
key`. Quality notes (duplicate values; missing or null key fields, which only a
forced key can reach) read `… key … — pairing quality degraded`, name the array by
its container path, and count affected and considered records over both sides.

**Duplicate key values.** Records sharing a key value cannot be told apart by it:
they pair in the order they appear, which may pair the wrong two. Two things
happen, and both are reported.

An **autodetected** key is re-checked against every record of the compared range
once the pairing is done, before any record is reported — detection itself only
samples, so duplicates spread thinly through a large file can pass it. A field
whose values turn out to be less than **99% distinct** across that range is not
identifying records at all: the key is dropped, the array falls back to
order-based alignment, and the reason is named.

```
gjxdiff: note: key candidate "id" rejected (duplicate values) at $[*] — positional alignment used
```

At or above that threshold the key is kept — discarding a five-million-record pairing
over one collision would produce a far worse report than a couple of doubtful
rows — and the duplicates are disclosed by count instead, so you can judge them:

```
gjxdiff: note: auto key "id" has duplicate values (2 of 400000 records) at $ — pairing quality degraded
```

The count is the records involved in a repeat, both sides added together; `--key
none` or a `--key` naming a genuinely unique field removes the doubt. A key you
force with `--key` is never dropped — forcing is your decision — but it gets the
same note, reading `forced key` instead of `auto key`, whenever its values repeat.

**The `(N items compared by key)` figure is not the array's length.** It counts the
items the keyed join actually covered: the larger side of the array pair's divergent
middle, after leading and trailing runs of already-identical records are trimmed
away (summed when one note aggregates several array pairs). A 20,000-record file
whose three changed records sit at ordinals 100, 150 and 10,000 reports
`(9901 items compared by key)` — the window from 100 through 10,000 — not 20,000
and not 3. An array whose divergent
window is smaller than 4 items on either side is never auto-keyed (the positional
aligner handles it directly), so a one-record edit produces no key note at all; a
forced key is not subject to that minimum and joins — and reports — any window
size.

At most 10 key notes of each class are listed individually; a totals line follows
when there are more.

### Record order under a key

Keyed matching answers "which record is which" by key value, not by position —
that is the whole point on a re-sorted export, and it is why order-only
differences produce no records. What it must never do is leave you thinking
nothing moved. Every keyed note therefore ends with the order verdict, and when
records did move, with a count:

```
gjxdiff: note: $[*] matched by auto key "id" (7900 items compared by key) — record order not compared; 1 record(s) must move to reconcile the two orders (--record-order lists them as moves)
```

**What the number means.** It is the **minimum number of records that must be
relocated** so that the key-matched records appear in the same relative order on
both sides: the number of matched records minus the size of the largest set of
them that already reads in ascending A-side order when taken in B's order (the
members of that set need not be adjacent). It is *not* the number of records
whose index differs. One record lifted from index
300 to index 6999 shifts the 6699 records between them by one position each, and
the count is **1** — the answer a person comparing the two files would give. A
2000-record array re-sorted by a different field typically reports around 1000.
You can check it by hand on a small file: write down the A positions of the
matched records in B's order, cross out the fewest entries needed to leave an
ascending sequence, and count what you crossed out.

**The count is exact; which records it names is not always unique.** Where two
different sets of records of the same size would each reconcile the order, the
tool reports one of them. Any given run is deterministic and repeatable, and the
count is the same whichever set is chosen — but comparing the same two files with
the arguments in the other order can name a different set. That is visible only
where the choice decides whether a mover is listable: with one set a displaced
scalar carries the move and nothing can be listed, with the other a neighbouring
record carries it and the pair appears in the report.

The same number reaches every output mode. The stat footer and the machine
summary line qualify their move count with it, because that count is a count of
move *records* and a keyed reorder produces none:

```
1 changed, 0 added, 0 removed, 0 moved (+996 reordered, not listed; --record-order)
```

A file with more than 256 distinct key-use rows keeps only the first 256. When
such a run counted any reordered records, one whole-run note gives the total
across every key-matched array, so a count carried by a dropped row is never
lost:

```
gjxdiff: note: 300 record(s) must move to reconcile record order across all key-matched arrays
```

### `--record-order`

Puts those records in the report, as the same `moved_from` + `moved_to` pairs
order-based alignment produces:

```sh
gjxdiff --record-order a.json b.json
```

```
{"i":0,"op":"moved_from","path":"$[*]","ptr":"/300","a_off":77785,"a_len":255,"b_off":0,"b_len":0,"move_pair":1,"flags":["keyed"]}
{"i":1,"op":"moved_to","path":"$[*]","ptr":"/6999","a_off":0,"a_len":0,"b_off":1837586,"b_len":255,"move_pair":0,"flags":["keyed"]}
```

The key note then says order was compared, and how many move records the report
carries:

```
gjxdiff: note: $[*] matched by auto key "id" (7900 items compared by key) — record order compared; 1 record(s) listed as moved
```

- One pair per moved record. A fully re-sorted array produces a pair for nearly
  every record, which is exactly why this is opt-in: keyed matching exists to
  keep a re-sort from burying the value-level differences.
- `--only move` then selects these like any other move pair, and the stat
  footer's move count includes them.
- The pairing is proven by the **key**, not by content. There is no 64-byte
  floor and no move-candidate pool here (both belong to the removed/added
  subtree join, which has to guess what pairs with what). A record that moved
  *and* changed produces its move pair **plus** its value-level records, so the
  two halves of a keyed move pair may differ in content.
- A displaced item that is not an object or array is counted but not listed — a
  scalar is never a move record, exactly as under order-based alignment. The key
  note then names both numbers, a second note gives the reason, and the move
  total still carries the remainder, so nothing claims that nothing moved:

```
gjxdiff: note: $[*] matched by auto key "id" (2001 items compared by key) — record order compared; 0 of 1 record(s) listed as moved
gjxdiff: note: 1 moved item(s) are not containers and are counted but not listed — a scalar is never a move record
gjxdiff: 1 changed, 0 added, 0 deleted, 0 moved (+1 reordered, not listed)
```

- Which records are named as the movers is one of possibly several equally
  minimal answers, so the listing can differ between a run and the same run with
  the files in the other order — see
  [Record order under a key](#record-order-under-a-key). How many moved does not
  change; which records carry them can.
- Arrays aligned by order always compare order, so the flag changes nothing
  under `--key none` or on any array that was not keyed.
- The human view labels key-matched records by key value, so both halves of a
  pair read `[=300]`; each line names its own position:

```
$
  > [=300]: {"id":300,"name":"india india 300",… (+95 bytes) (moved away from index 300)
  > [=300]: {"id":300,"name":"india india 300",… (+95 bytes) (moved here, index 6999)
```

- `--patch` refuses an absorbed reorder with or without this flag (see
  [When the export refuses](#when-the-export-refuses)).

When a rejected or unused key is the reason a comparison came out coarse, a
`gjxdiff: hint:` line names the field to try first (see [Large inputs](#8-large-inputs)).

---

## 7. Narrowing the comparison

### `--ignore PATTERNS`

Exclude fields and paths from the comparison. Matching nodes are treated as **absent
on both sides**: no records for them or beneath them, and subtrees that differ only
in ignored content compare *and pair* as identical. When nothing but ignored content
differs, the exit code is 0.

```sh
gjxdiff --ignore ts,updated_at a.json b.json
gjxdiff --ignore '$.meta.ts' --ignore '$.arr[*].seq' a.json b.json
```

Repeatable; commas split patterns (`\,` for a literal comma).

| Pattern | Matches |
|---------|---------|
| `ts` | The member `ts` at **any** depth (a bare name). |
| `$.meta.ts` | An anchored path from the root. The leading `$.` is optional. |
| `items[0].v` | Exact array position. Positional only — key-matched records are not masked by position. |
| `$.arr[*].ts` | Any array position. |
| `$.cfg.*` | Any member name at that level. |
| `$["dotted.name"]` | A quoted name, for dots, special characters and edge whitespace. |
| `$[""]` | The empty member name. |

Anything containing `$`, `.`, `[` or `]` is treated as an anchored path; anything
else is a bare name.

Active patterns and masked sites are disclosed on stderr:

```
gjxdiff: note: 2 ignore pattern(s) active — fields matching them are excluded from comparison
gjxdiff: note: ignore masking excluded 4120 value site(s) at compared containers; equalized 96 container pair(s) differing only in ignored content; 412 difference site(s) suppressed
```

The second note's three counters mean:

- **value site(s)** — member or item occurrences excluded from comparison at
  containers the walk actually compared, counted on both sides. Occurrences whose
  values happen to be equal count too: this is "sites masked", not "differences
  found".
- **equalized container pair(s)** — container pairs whose raw content differs but
  whose masked views are equal. The pair is absorbed as identical, which also hides
  every masked site beneath it (those sites are not added to the first counter). A
  nonzero count is the direct signal that a pattern absorbed real differences.
- **difference site(s) suppressed** — difference sites the ignore set removed from
  the report: masked sites whose two sides did not agree (differing values, or a
  member or item present on one side only), plus one for each pair absorbed as
  identical on the masked view. The equalized count is the container subset of
  those absorptions.

The document root is the one pair these counters never absorb: whole-document
masked identity is decided by a separate check and reported by the verdict line
(`files are identical apart from ignored fields`, exit 0), not counted as an
equalized pair. Two one-field documents differing only in an ignored member
therefore read `equalized 0 container pair(s)` alongside that verdict — the root's
masked sites still feed the value-site and suppressed counters.

### `--path PATH`

Compare only the subtree at one exact address:

```sh
gjxdiff --path '$.data.items[3].users' a.json b.json
gjxdiff --path '$[7]' a.ndjson b.ndjson     # NDJSON record 7, i.e. line 8
gjxdiff --path '$["dotted.name"]' a.json b.json
gjxdiff --path '$' a.json b.json            # the whole document
```

- Single occurrence. Anchored address, no wildcards. The leading `$` is optional.
- Both files must contain the address. Missing on either side is an error naming the
  file.
- Report paths are re-rooted at the selected subtree: `$` then means the selected
  path. `ptr` values are rooted the same way.
- A scalar target compares as a single value.
- A member name duplicated at its level in either file makes the address ambiguous —
  that is an error; compare the parent instead.
- `--ignore` patterns stay anchored at the **original** document root, so the same
  pattern set works with and without `--path`. `--key` applies inside the subtree.

The applied path is disclosed on stderr:

```
gjxdiff: note: comparing only the subtree at $.data.items (--path) — report paths are rooted at the selected subtree
```

`--ignore` and `--path` are checked against each other: an ignore pattern that
excludes the entire selected subtree leaves nothing to compare, and is an error
naming both flags rather than a vacuous "no differences".

---

## 8. Large inputs

### The memory budget

`--memory-limit SIZE` sets the memory budget: the anonymous memory (heap) the engine
may plan for, and the number every internal limit is derived from. Accepted forms:
plain bytes, or a `K`/`M`/`G` suffix (KiB/MiB/GiB, case-insensitive) — `512M`, `2G`.

Default: **a fixed 4 GiB, on every machine.** An explicit `--memory-limit` replaces
it verbatim — nothing clamps, raises or reinterprets your value in either direction,
not even above the machine's physical RAM. The engine's internal pool sizes (the
alignment cap, work stack, move-candidate and memo pools) scale with the
budget, so a bigger budget buys item-exact alignment of bigger structures and a
smaller one coarsens sooner.

**The default trades precision for reproducibility, and on a large machine you
should raise it.** A fixed default is what makes a report reproducible everywhere
(see the determinism contract below), but it also means the tool does not help
itself to memory you have. The default's limits are the same on a laptop and on a
64 GB build server: 6710886 children aligned exactly per container, 3919580
move candidates per side. Give a big machine a budget to match and both grow in
proportion — `--memory-limit 32G` raises the alignment cap to 53687091 children and
the move pool to 31356644 per side, so structures that the default coarsens into one
`region_changed` are reported element by element instead:

```sh
gjxdiff --memory-limit 32G huge-a.json huge-b.json
```

Nothing about this is silent. A run that actually hit a limit says so on stderr,
names the resolved number, echoes it in the machine meta line, and — for the
alignment cap — prints the `--align-cap … --memory-limit …` command that would have
produced the exact result. Raising the budget never changes a report that did not
hit a limit in the first place.

**Determinism contract.** Every budget-derived limit resolves from the budget alone,
and the budget is a constant unless you change it. So the same inputs and flags
produce the same report on every machine — including the runs where a limit actually
bit (an over-cap container, a work-stack trip, a saturated move-candidate pool), each
of which is disclosed on stderr and echoed in the machine meta line
(`filters.align_cap`, `filters.move_cap`). Nothing in the report depends on free RAM.
Passing `--memory-limit` changes the budget and may therefore change those limits and
the records that depend on them — but any given value is exactly as reproducible as
the default.

The budget covers heap memory only. Memory-mapped file pages sit outside it: the
kernel reclaims them under pressure rather than failing an allocation. If you watch
the process with a tool that reports resident memory, expect the file-backed share to
grow with the inputs — that is normal and reclaimable. Inside a container those pages
are still charged to the container's memory limit alongside the heap, which is why
the guard below measures the container differently.

**The physical guard.** The budget says what the tool may plan for; it cannot conjure
memory that is not there. A separate guard watches what the run can actually get and
aborts when a threshold is passed. Each threshold governs the quantity it actually
bounds:

| Source | Threshold | Measured against |
|--------|-----------|------------------|
| The memory budget | `--memory-limit`, or the fixed 4 GiB default | Heap use |
| Machine memory | 85% of `MemAvailable` at launch | Heap use |
| Each container memory limit | 85% of the room free under that cgroup's limit at launch | Growth of the container's memory use since launch |

The container check is what makes the guard work inside Docker, Kubernetes, systemd
slices and LXC. Both cgroup v1 and cgroup v2 are read, every ancestor cgroup is
watched (a parent can be tighter than your own), and every mounted view of the
hierarchy is inspected. Growth is measured **live**: inside a cgroup the memory-mapped
inputs and the temporary files count against the same limit as the heap, and another
process growing inside the same cgroup counts too — the kernel's kill decision does
not care whose pages fill the limit, so neither does the guard, and a neighbour eating
the remaining room still ends in this tool's clean abort rather than a kernel kill.
What a neighbour **already held at launch** is different: that memory was never this
run's to use, so it is measured once at launch and subtracted — a container 88% full
of someone else does not refuse a small comparison that fits in the room that is left.
(Room a neighbour frees after launch lowers the measured growth as it happens, so the
run benefits immediately; the abort thresholds themselves are fixed at launch and
never loosen mid-run.) Page cache the kernel can still reclaim is excluded, so reading
or mapping a large file in a container does not by itself bring the run near its
limit. If no cgroup limits the run, this source contributes nothing.

`memory.max` always counts. `memory.high` throttles instead of killing, and what it
means depends entirely on swap. While the cgroup can still swap, the throttle just
slows the run down and it finishes, so it is not treated as a limit. The moment the
swap allowance is exhausted — a finite `memory.swap.max` used up, or the machine's
swap effectively full — there is nothing left to reclaim into and the throttle stops
the run from making any progress at all: measured on an earlier build, a 6-second
comparison under `memory.high` with a 64 MiB swap allowance was still sitting in the
kernel's reclaim stall after 10 minutes, with zero output. The guard therefore watches
the swap allowance and treats `memory.high` as a limit from the moment the allowance
runs out, aborting promptly instead of stalling; a cgroup that cannot swap at all
(`memory.swap.max` 0, or a machine without swap) makes `memory.high` a limit from the
start. v1's `memory.soft_limit_in_bytes` is deliberately not treated as a limit — it
only biases reclaim priority while the machine is under pressure, and a cgroup may sit
above it indefinitely.

The guard can only stop a run: it never changes a limit and never changes output, so
a small machine or a tight container still produces byte-identical results for every
run that fits — only a run that genuinely needs more memory than it can get aborts.
The message names which number bound the run and reports the figures:

```
gjxdiff: memory budget exceeded (rss 4.1 GiB > budget 4 GiB); partial temp files removed
gjxdiff: the budget bound this run (physical headroom was 6.8 GiB) — raise --memory-limit, or narrow the comparison with --path / --ignore
```

```
gjxdiff: machine memory exhausted (rss 1.8 GiB > physical guard 1.7 GiB); partial temp files removed
gjxdiff: the machine bound this run — the guard is 85% of MemAvailable at launch (1.7 GiB), below the 4 GiB memory budget; free memory on this machine, or pass a smaller --memory-limit (a smaller budget may coarsen large arrays and objects instead of aligning them exactly)
```

```
gjxdiff: container memory limit reached (growth since launch 679.8 MiB > container guard 679.6 MiB); partial temp files removed
gjxdiff: a container memory limit bound this run — the cgroup v2 limit is 800 MiB, 523.4 KiB of it was in use at launch, and the guard stops the run at 85% of the 799.5 MiB that was free (679.6 MiB), below the 4 GiB memory budget; reclaimable page cache is not counted; raise the container's memory limit, free memory in the container, or pass a smaller --memory-limit (a smaller budget may coarsen large arrays and objects instead of aligning them exactly)
```

In a container so tight that the run cannot even shut down in the room left above the
guard, a third line says the run was ended immediately rather than unwound. The exit
code, the message and the temp-file cleanup are the same — only the orderly shutdown
is skipped. That makes a clean early exit overwhelmingly likely, but it is not a
guarantee: in a narrow band of container limits the kernel's OOM killer can still win
the race against the abort itself, in which case the run ends with the kernel's kill
(exit code 137) and temporary files may be left behind — the next start's sweep
removes them.

If neither `/proc/meminfo` nor any cgroup limit can be read, the guard has no physical
component and falls back to the budget alone.

A rarer degraded shape is a cgroup limit that is readable while its **usage** file is
not (a masked or restricted `/sys/fs/cgroup` view). The guard then watches the nearest
parent cgroup whose usage is readable — that includes everything else charged under
it, so the run may abort earlier than the limit alone requires — and says so in a
`gjxdiff: warning:` line at launch. The room the guard shares out is then the limit
minus everything that parent already held at launch, which can be far smaller than
what the limit alone would leave — down to nothing, when the parent already held more
than the limit — so a run that would fit under the limit itself can still be refused.
The guard also depends on the usage file it watches staying readable for the whole
run: one that turns unreadable mid-run and stays that way leaves the limit unwatched
from that point on, without a further message. If no usage file is readable at any level, the
warning states instead that the limit cannot be watched at all: if the container then
runs out of memory, the kernel ends the run with no message, and temporary files may
be left behind (a later run with the same `--temp` removes them). These warnings print
even under `--quiet`: a guard running degraded is never silent about it.

### Temporary files

Working state lives in a per-run directory under `--temp` (default: the system temp
directory):

```
<DIR>/gjxdiff/run-<pid>-<nonce>
```

It is removed on exit — success, error, panic, SIGINT/SIGTERM. Two narrow cases can
outrun that cleanup: a second SIGINT/SIGTERM arriving within the shutdown's first
milliseconds, and the kernel killing the run (exit code 137) inside a container too
small for the workload while an abort is already under way. Whatever such a kill
leaves behind — like directories left by SIGKILL or power loss — is swept at the next
start with the same `--temp`. The root directory is created if missing.

`--max-temp SIZE` caps the total size of everything under the run's temp directory.
Same size grammar as `--memory-limit`. **There is no default: temp use is unlimited
unless you set it.** Enforcement is sampled several times a second rather than
checked on every write, so a run that spools quickly can pass the cap by what it
wrote within one sampling interval before it is stopped. Exceeding the cap is a
clean exit 2 with every partial temp file removed:

```
gjxdiff: temp budget exceeded (temp 739.1 MiB > limit 50 MiB); partial temp files removed
```

There is no reliable sizing rule from input size alone. Measured across varied
corpora, peak temp use ranged from roughly 0.15x to 2.1x of the two inputs'
combined size. The main driver is how many records and nodes the inputs pack per
byte: files made of many tiny records peaked near the high end of that range,
files of fewer, larger records near the low end — with how much of the input
actually differs a secondary factor. A cap derived from input size alone can
therefore abort a legitimate run. `--max-temp` is a backstop against runaway disk
use, not a meter: size it by measuring — run the job once without a cap on
representative inputs and watch the run's temp directory (for example `du -s` on
the `--temp` root) — or give it generous headroom above anything you expect. A cap
set tight against the expected peak is not a precise trip wire either: on fast
storage a run can write far past it within one sampling interval, as the 50 MiB
cap above shows (passed at 739.1 MiB).

### The alignment cap and coarse regions

Aligning a container element by element costs memory proportional to its number of
direct children. `--align-cap` bounds that:

```sh
gjxdiff --align-cap 2000000 a.json b.json
gjxdiff --align-cap unlimited a.json b.json
```

- A positive count, or `unlimited` to align any size (memory is then governed by the
  memory budget alone).
- Default: scaled from the memory budget as budget/640, never below 500000 — with the
  fixed 4 GiB default budget that is exactly **6710886** children, the same number on
  every machine and in every run. Raising the cap also raises the work-stack limit to
  twice the cap; the move-candidate pool is not affected by this flag and follows the
  budget instead.

A container with more direct children than the cap does not silently produce wrong
results. Its divergent region degrades to **one coarse `region_changed` record**,
which reports the byte range that differs rather than the individual items inside it.
This is disclosed on stderr, per container and in total:

```
gjxdiff: note: coarsened $.data (5241880 vs 5240012 children; cap 500000)
gjxdiff: note: 3 containers coarsened in total (first 3 listed) — those regions are coarse, not item-exact
```

The parenthesis names the limit that degraded the container: `cap N` for the alignment
cap, `cap N, record pairing unproven` when an over-cap array could not prove its
positional pairing, and `work-stack limit` for the work stack. Individual coarsen
lines are listed for at most the first 10 containers.

Because the records then depend on the cap, the meta line's `filters.align_cap`
echoes the resolved value on any run that entered a cap-dependent rail, and the same
value is disclosed as a stderr note in every output mode (human and `--stat` have no
meta line). The note states where the cap came from — under the default budget:

```
gjxdiff: note: over-cap container(s) encountered — resolved alignment cap 6710886 (derived from the default 4 GiB budget)
```

With the budget set by `--memory-limit`, the tail reads `derived from --memory-limit`
instead; with an explicit `--align-cap` the note reads
`alignment cap N (--align-cap)`. Both numbers are reproducible: the tail says which
input produced the cap, not how much it might wobble.

On runs that never entered a cap-dependent rail, `filters.align_cap` stays `null` and
no note appears.

A container coarsened by the **work-stack limit** gets its own note instead — no
container was over the cap there, the pending work stack was. The work stack is
`max(budget-scaled pool, 2 × the alignment cap)`, so raising `--align-cap` raises it:

```
gjxdiff: note: work-stack limit 13421772 pending descents (derived from the default 4 GiB budget)
```

Two further disclosures may accompany coarsening:

```
gjxdiff: note: coarsened regions are compared without ignore masking — a coarse region may reflect only ignored-field differences
gjxdiff: note: positional ignore patterns ([N]/[*]) were not fully applied to 2 over-cap array(s) — raise --align-cap to apply them exactly
```

### The hint block

After the coarsen notes, `gjxdiff` prints one computed `hint:` block with concrete
flag values for the run you just did. It is advice on stderr; it changes nothing.

```
gjxdiff: hint: try --key "order_id" first — fastest and most precise; the auto candidate was rejected: <recorded reason>
gjxdiff: hint: --align-cap 5241880 --memory-limit 4800M would exactly align the coarsened array region(s)
```

- A key suggestion comes first when the tool recorded why keyed pairing was not used
  — keying an array is cheaper and more precise than raising the cap.
- The cap suggestion computes the count from the largest coarsened container of that
  class and the matching memory budget. Objects and arrays get separate lines, since
  the array-specific options do not help objects.
- A container coarsened by the **work-stack limit** gets its own line, computed from
  the recorded stack demand rather than any container's child count (the work stack
  is what tripped, not a per-container cap; following the hint raises the limit to
  twice the recorded demand):

```
gjxdiff: hint: --align-cap 1200000 --memory-limit 1099M would raise the work-stack limit past the 1200000 pending descents the rail refused and let the coarsened container(s) align
```
- When the array pairing could not be proven, the suggestion says "attempt item-level
  alignment" rather than promising exactness.
- When the required budget would be more than eight times the current one, no cap is
  recommended. Instead:

```
gjxdiff: hint: exact alignment of arrays this size needs --large-arrays=exact, which is planned but not available in this version
```

`--quiet` silences the hint block along with everything else on stderr.

### The move-candidate pool

Move detection collects removed and added subtrees as candidates. A candidate is a
whole removed or added **container** — an object or an array — spanning at least
**64 bytes** in its own file; scalar values and smaller containers are never
reported as moves (their pairs stay plain `removed` + `added` records, whatever
their content). Candidates are capped per side
by a pool that scales with the memory budget, never below 1000000 per side — with
the default 4 GiB budget it is **3919580** per side. When a side offers more
candidates than the pool admits, the smallest candidates are dropped
deterministically — their pairs report as plain `removed` + `added` records
instead of `moved_from` + `moved_to`. That is a change in what the report *says*,
so it is never silent. Under a budget small enough to leave the pool on its floor
(here `--memory-limit 1000M`):

```
gjxdiff: note: move-candidate pool saturated — 1200000 removed-side / 1200000 added-side candidate(s), pool cap 1000000 per side; moves past the cap report as plain removed+added records (derived from --memory-limit)
gjxdiff: hint: --memory-limit 1255M would fit all 1200000 move candidate(s) per side
```

The note appears in every output mode; the machine meta line echoes the resolved
cap as `filters.move_cap` on exactly these runs (`null` otherwise). The hint's
`--memory-limit` value is computed from the recorded demand, so following it fits
every candidate; like the coarsen hints, it is suppressed when the required budget
exceeds eight times the current one. On a default run the tail reads `derived from
the default 4 GiB budget` instead. There is no separate flag for this pool — the
budget is the knob.

### `--large-arrays MODE`

How containers larger than the alignment cap are handled.

| Mode | Behavior |
|------|----------|
| `coarse` | The default, and the only mode available in this version. Each over-cap divergent region degrades to one coarse `region_changed` record, disclosed on stderr. |
| `exact` | An exact path with no cap. **Planned for a future version.** Passing it today exits 2 with a clear message; it never silently falls back to coarse. |

The active mode is always echoed in the meta line's `filters.large_arrays`.

The flag exists now so that scripts written today keep working unchanged when the
exact mode ships.

---

## 9. Patch export

`--patch FILE` writes an RFC 6902 JSON Patch alongside the normal run: one JSON array
of `add` / `remove` / `replace` operations that transforms input A into input B under
`gjxdiff`'s semantic equality. An empty diff produces `[]`.

```sh
gjxdiff --patch changes.json config-a.json config-b.json
gjxdiff --patch - config-a.json config-b.json | jq .
```

- The patch is one valid JSON array and nothing more granular. Values keep B's byte
  representation and may span lines; operations are not line-delimited.
- Untouched regions keep A's representation. That is fine: whitespace, number form
  and string escape form all normalize under the equality the patch is measured
  against.
- Array **order** is part of equality, so every order difference the report expresses
  is reproduced.
- `--patch -` writes the patch to stdout **instead of** the report — the two formats
  never share a stream. There is no meta line and no records; the verdict and summary
  stay on stderr.
- The file target is written atomically (temporary sibling, then rename). An error,
  an interrupt or a panic leaves neither the patch nor a temporary file behind.
- Deterministic: identical runs produce byte-identical patches.
- Exit codes are unchanged: 0 identical (with a `[]` patch), 1 differences, 2 export
  failure.

On success:

```
gjxdiff: note: RFC 6902 patch written to changes.json (4812 operation(s))
```

### Operation encoding

Only `add`, `remove` and `replace` are emitted — no RFC `move` or `copy`. A move is a
remove plus an add. A renamed key is a remove of the old name plus an add of the new
name with B's bytes.

Operations are ordered so that sequential application is correct: whole-item removes
first (descending in A order), then whole-item adds (ascending in B order), then
member-level operations at their final positions. Apply the array in order, from the
first operation to the last.

### Coarse regions

A coarse region covers a byte range that is not itself a JSON value. The patch
therefore expresses it as a `replace` of the **whole container**, carrying the
container's full B-side value. The operation is valid and honest; on a large
container it is also large. The record spans in the report keep the narrower,
honest divergent range.

### NDJSON convention

Pointers address record N as `/N/...`. Applying a patch to an NDJSON file means
treating it as an array of records: wrap the records in `[ ... ]`, apply, unwrap.
This is exactly the address space the report uses for `$[N]` and `ptr`.

### When the export refuses

The export either succeeds completely or fails with exit 2 and leaves no file. A
silently partial patch would corrupt data, so refusal is the designed behavior:

| Situation | Why |
|-----------|-----|
| A needed record has no concrete RFC 6901 address (duplicate or undecodable member names, an unprovable array ordinal, the path-overflow bucket, an unrecoverable old name for a rename) | Writing it would patch the wrong node. The error names the total and the per-reason counts. |
| Keyed pairing absorbed a record reorder — the order-only case, where the reorder is the only difference and the report has zero records, and equally the mixed case, where value changes are reported alongside it | Keyed matching ignores order, so the reorder produces no records, and the patch export does not support one. The error states how many records must move. Rerun with `--key none` for a positional patch. The refusal stands under `--record-order` too: a record that moved *and* changed carries value-level records alongside its move pair, and applying both would patch it twice. |
| A coarsened NDJSON virtual root | The whole record array is not one JSON value. |

A representation-only difference does **not** refuse: files that differ only in
whitespace, number form or string-escape form compare identical (exit 0) and export
the empty patch `[]`, exactly like byte-identical inputs.

Records at or below the NDJSON record level whose address cannot be proven do **not**
fail the export. They degrade to a remove plus an add of the whole record — exact and
complete — and are disclosed:

```
gjxdiff: note: --patch: 3 record pair(s) written as whole-record remove+add (no member-level address there)
```

### Flag interactions

| Combination | Result |
|-------------|--------|
| `--patch` + `--max-diffs` | Exit 2. A truncated patch would corrupt data. |
| `--patch` + `--only` | Exit 2. A filtered patch would be silently partial. |
| `--patch` + `--profile quick` | Exit 2. The profile's `--max-diffs 1000` component would truncate the patch. Drop the profile. |
| `--patch -` + `--human` / `--ndjson` / `--stat` | Exit 2. They all own stdout. |
| `--patch -` + `--quiet` | Exit 2. The patch would have nowhere to go. |
| `--patch FILE` + `--quiet` | Allowed. The file is written silently. |
| `--patch` + `--ignore` / `--path` | Allowed. The patch covers the narrowed comparison only, disclosed on stderr; under `--path` the pointers are rooted at the selected subtree. |

```
gjxdiff: note: the patch covers the narrowed comparison only (2 ignore pattern(s) active) — applying it will not fully reproduce the second input
```

---

## 10. Filters and profiles

### `--only KINDS`

Emit only these record kinds. Comma-separated; repeatable, occurrences accumulate.

| Kind | Report ops |
|------|------------|
| `add` | `added` |
| `change` | `changed` |
| `coarse` | `region_changed` |
| `move` | `moved_from` + `moved_to` pairs |
| `remove` | `removed` |
| `rename` | `key_renamed` |
| `type` | `type_changed` |

```sh
gjxdiff --only add,remove a.json b.json
```

Every op belongs to exactly one kind, and `move` covers both halves of a pair, so a
filtered report never shows one-legged moves. There is no `order` kind: under
order-based alignment, a displaced item reports as a move pair when it is an object
or array of at least 64 bytes in its own file ([§8](#8-large-inputs), the
move-candidate pool) — displaced scalars and smaller containers report as `removed`
+ `added` instead, so `--only move` on an array of small values shows nothing while
`--only add,remove` shows the reorder. Keyed arrays absorb reorders: a record that
only changed position produces no records, so `--only move` comes up empty there
however many records moved — the key note on stderr gives the count, and
[`--record-order`](#--record-order) puts the pairs in the report, where this filter
selects them like any other move. A pure reorder without that flag surfaces as the
verdict line and the exit code, which `--only` never alters.

**Only the report is filtered. The comparison is not narrowed.**

- The summary and stat footer keep the full totals, plus the suppressed count.
- stderr discloses the suppression per kind, on every verdict path, including exit 0
  (where it reports `0 record(s) suppressed`).
- The meta line's `filters.only` echoes the active set.
- Emitted record lines are byte-identical to the unfiltered report's; the `i` field
  keeps the unfiltered index, so a filtered report is a pure subset.
- The **exit code always reflects the full comparison**. A run whose every difference
  is suppressed still exits 1.

```
gjxdiff: note: --only change — 1571 record(s) suppressed (571 add, 1000 remove); totals and the exit code reflect the full comparison
```

An unknown or empty kind exits 2 naming the valid set — at parse time, so
`gjxdiff --about --only bogus` also exits 2 rather than printing the about block.

### `--profile NAME`

Flag bundles, applied **before** explicit flags. An explicit flag always overrides
its profile component, regardless of argument order.

| Profile | Expands to |
|---------|-----------|
| `ci` | `--ndjson --color=never --progress=never` |
| `quick` | `--max-diffs 1000 --progress=auto` |
| `thorough` | `--align-cap unlimited --progress=auto` |

```sh
gjxdiff --profile ci a.ndjson b.ndjson > report.ndjson
gjxdiff --profile thorough --progress=always huge-a.json huge-b.json
```

Profiles never change comparison semantics silently. A limiting component — `quick`'s
`--max-diffs` — is disclosed exactly as the explicit flag would be.

`ci`'s `--ndjson` fills the output-mode slot: any explicit `--human`, `--ndjson` or
`--stat` overrides it, and it yields to `--patch -`, which owns stdout outright.

Profiles are **not** echoed in the meta line. The meta carries the resolved component
values (`"max_diffs":1000`, `"align_cap":"unlimited"`), so a consumer never needs a
profile table to interpret a report, and a future redefinition of a profile cannot
change what an old report meant.

An unknown profile name exits 2 listing the valid names.

### Precedence, in order

1. Profile components are applied first.
2. An explicit flag overrides its profile component.
3. Declared conflicts are errors, not silent resolutions (`--human` vs `--ndjson`,
   `--stat` vs either, `--patch` vs `--max-diffs` / `--only`, `--profile quick` vs
   `--patch`, `--patch -` vs the stdout owners and `--quiet`).
4. Output mode: `--stat` beats `--human` beats `--ndjson`; with none of them, a
   terminal gets the human view and a pipe gets the machine report.

### `--max-diffs N`

Emit at most N diff records.

This truncates the **output** only. The full comparison still runs and no work is
skipped: the exit code, the totals and the summary reflect the complete diff, and the
summary discloses the truncation. In the human view a `… N more record(s) not shown`
line precedes the footer.

Under `--only`, `--max-diffs` counts the records actually emitted, and the truncation
clause compares against the kept total, not the full total.

Conflicts with `--patch`.

---

## 11. Progress and automation

### `--progress WHEN`

Progress goes to stderr. Default `auto`.

| Value | Behavior |
|-------|----------|
| `auto` | A rewritten status line, on a terminal only. |
| `always` | Plain sequential lines with no rewriting and no ANSI, at most one line per stage per two seconds. For CI logs. The opening line is emitted immediately, so even short runs log their start. |
| `never` | Off. |

`--quiet` overrides all three.

Lines carry the stage, a percentage and, for byte-based stages, a throughput figure:

```
gjxdiff: indexing A 42% - 510 MB/s
```

There is no ETA. While standard input is being buffered, the `reading stdin` stage
shows bytes read with no percentage and no rate — the total is unknown until the pipe
ends.

### `--quiet`

Suppresses all output on both streams. The exit code alone answers. `--patch FILE`
still writes its file.

### Exit codes

| Code | Meaning |
|------|---------|
| `0` | Identical: byte-identical, identical under semantic comparison, or identical apart from `--ignore`d content. |
| `1` | Differences found. Includes order-only differences (zero records) and runs whose every record was suppressed by `--only`. |
| `2` | Error: malformed input, invalid flags, a failed patch export, or a tripped memory or temp budget. No report is emitted — with one exception: a `--patch FILE` run whose final rename fails exits 2 with the already-streamed report on stdout. |
| `130` | Interrupted (SIGINT/SIGTERM). Temporary files are removed (a second signal within the shutdown's first milliseconds can outrun the cleanup; the next start sweeps what remains). stdout may carry an already-flushed prefix of the report; treat it as truncated. |

A second Ctrl-C hard-exits.

### CI examples

Fail a pipeline when a generated artifact drifts:

```sh
if ! gjxdiff --quiet --ignore generated_at expected.json actual.json; then
  echo "artifact drift detected"
  exit 1
fi
```

Record a report and act on it:

```sh
gjxdiff --profile ci --key id a.ndjson b.ndjson > report.ndjson
status=$?
case $status in
  0) echo "no changes" ;;
  1) jq -r 'select(.op) | "\(.op) \(.ptr // .path)"' report.ndjson ;;
  *) echo "gjxdiff failed"; exit $status ;;
esac
```

Count records by op, skipping the meta line:

```sh
gjxdiff --ndjson a.json b.json | jq -r 'select(.op) | .op' | sort | uniq -c
```

Gate on a bounded run on a small build agent:

```sh
gjxdiff --profile ci --memory-limit 1G --max-temp 8G big-a.ndjson big-b.ndjson > report.ndjson
```

### Environment

| Variable | Effect |
|----------|--------|
| `NO_COLOR` | Set and non-empty: disables color under `--color=auto`. An explicit `always` or `never` wins. |

### Shell completions

Generated by the binary from its live flag table, so they cannot drift:

```sh
gjxdiff --completions bash > /etc/bash_completion.d/gjxdiff
gjxdiff --completions zsh  > "${fpath[1]}/_gjxdiff"
gjxdiff --completions fish > ~/.config/fish/completions/gjxdiff.fish
```

`bash`, `zsh` and `fish` are supported; any other value exits 2.

---

## 12. Troubleshooting

### Format errors

```
data-b.json: format: trailing comma at byte 41208
-: format: no JSON root value
```

The input is not structurally valid JSON or NDJSON. The offset is the offending byte,
or the byte where the problem became detectable — the `]`/`}` that closes a trailing
comma, the opening byte of the deepest container or string still open at end of
input, the first byte of a trailing top-level value, or a BOM byte found mid-file.

The offset is omitted where no single offending byte exists:

| Message | Meaning |
|---------|---------|
| `format: no JSON root value` | The input is blank — there is nothing to point at. |
| `format: source exceeds u40 addressing (1 TB)` | The input is larger than the maximum supported file size. |
| Internal-consistency and working-file integrity messages | These describe derived files, not a source byte. |

Malformed input costs a little extra time before it fails: roughly one additional
scan of the prefix up to the rejection point. Valid inputs never pay it.

### Budget errors

```
gjxdiff: memory budget exceeded (rss 4.1 GiB > budget 4 GiB); partial temp files removed
gjxdiff: the budget bound this run (physical headroom was 6.8 GiB) — raise --memory-limit, or narrow the comparison with --path / --ignore
```

The heap budget was hit while the machine still had room. Raise `--memory-limit`, or
narrow the comparison with `--path` / `--ignore`, or lower `--align-cap` (a smaller
cap uses less memory and produces coarse regions instead).

```
gjxdiff: machine memory exhausted (rss 1.8 GiB > physical guard 1.7 GiB); partial temp files removed
gjxdiff: the machine bound this run — the guard is 85% of MemAvailable at launch (1.7 GiB), below the 4 GiB memory budget; free memory on this machine, or pass a smaller --memory-limit (a smaller budget may coarsen large arrays and objects instead of aligning them exactly)
```

The machine ran out before the budget did — this one needed more memory than this
machine has. Free memory, or pass a smaller `--memory-limit`: a smaller budget makes
the tool plan for less, at the cost of coarsening large arrays and objects instead of
aligning them item by item. See [§8](#8-large-inputs) for the guard.

```
gjxdiff: container memory limit reached (growth since launch 679.8 MiB > container guard 679.6 MiB); partial temp files removed
gjxdiff: a container memory limit bound this run — the cgroup v2 limit is 800 MiB, 523.4 KiB of it was in use at launch, and the guard stops the run at 85% of the 799.5 MiB that was free (679.6 MiB), below the 4 GiB memory budget; reclaimable page cache is not counted; raise the container's memory limit, free memory in the container, or pass a smaller --memory-limit (a smaller budget may coarsen large arrays and objects instead of aligning them exactly)
```

A cgroup memory limit ran out before the budget did — the room that was free in the
container when the run started is smaller than the run needs. The limit may belong to
a parent cgroup rather than the one you set. Raise it (`docker run --memory`, a
Kubernetes memory limit, a systemd `MemoryMax=`), free memory held by other processes
in the same container, or pass a smaller `--memory-limit` with the same trade as
above. The figures quoted are the container's **growth since launch** and the room
that was **free at launch**, not just this tool's heap: inside a container the
memory-mapped inputs, the temporary files and every other process in the cgroup count
against the same limit, while whatever the container already held when the run
started is subtracted first. Page cache the kernel can still reclaim is excluded, so
reading a large file in a container does not by itself bring the run near the limit.
If the message names `memory.high` and an exhausted swap allowance instead, the
container's throttle had run out of swap to reclaim into and the run was stopped
before it could stall indefinitely; raise `memory.high` or the swap allowance
([§8](#8-large-inputs)).

A container that is too tight for the run to unwind in adds a third line:

```
gjxdiff: the container was too close to its limit to unwind normally — the run was ended immediately; temporary files removed
```

The exit code, the message and the cleanup are unchanged; only the orderly shutdown is
skipped. A clean exit 2 with the message and the cleanup is the overwhelmingly likely
outcome, but not a guaranteed one: in a narrow band of container limits the kernel's
OOM killer can win the race against the abort itself, and the run then ends with the
kernel's kill (exit code 137) and temporary files may be left behind — the next
start's sweep removes them.

```
gjxdiff: temp budget exceeded (temp 739.1 MiB > limit 50 MiB); partial temp files removed
```

`--max-temp` was hit. Raise it, point `--temp` at a larger filesystem, or narrow the
comparison. Nothing is left behind. A change-dense pair can need more temp space
than the two inputs put together ([§8](#8-large-inputs)) — and enforcement is sampled: the
figures above show how far a fast spool can pass a tight cap before the next sample
stops it.

### Disk and I/O errors

I/O failures name what they were working on:

| Shape | When |
|-------|------|
| `<name>: io: <error>` | A failure while reading or building working files for that input. `<name>` is the argument as you typed it — `-` for the standard-input side. Also used for the `--patch` target. |
| `cannot read <name>: <error>` | The input could not be opened, sized, or format-detected. |
| `cannot map <name>: <error>` | The input could not be memory-mapped. |
| `cannot create <path>: <error>` | The temporary sibling for a `--patch` target could not be created. |
| `cannot write the report`, `cannot write records`, `cannot write the report meta header` | A write to stdout failed. These have no user-typed name. |

A full temp filesystem surfaces as `io: No space left on device`, exit 2, with every
temp file removed. A file-size limit (`RLIMIT_FSIZE`) surfaces as `io: File too
large`, the same way. Neither is ever a silent kill or a leftover directory.

### Flag errors

| Message (abridged) | Fix |
|--------------------|-----|
| `--large-arrays=exact is not available in this version (planned); the current mode is coarse` | Use the default `coarse` mode. |
| `"-" (standard input) cannot be both inputs …` | Only one side can be `-`. |
| `"-" reads from standard input, but stdin is an interactive terminal …` | Pipe or redirect data in. |
| `--patch -` combined with `--human` / `--ndjson` / `--stat` / `--quiet` | Use `--patch FILE` to keep the report, or drop the conflicting flag. |
| `--profile quick sets --max-diffs 1000, which conflicts with --patch …` | Drop the profile. |
| `--ignore "<pattern>" excludes the entire subtree selected by --path <address> …` | Drop one of the two. |
| `--path <address>: <file> does not contain this path` | The address is missing on that side. Check spelling and quoting. |
| `--path <address>: the member name "<name>" is duplicated at its level …` | Compare the parent instead; the message names it. |
| `--key "<field>" matched no array …` | Likely a typo; the run continues with autodetection or order-based alignment. |

### "It says the files differ but shows no records"

That is an order-only difference, and it exits 1:

```
gjxdiff: files differ only in record order or representation; no value-level changes
```

Keyed pairing ignores record order, so a pure reorder produces no records — but the
file-level verdict is order-sensitive. The key note on the same run says how many
records must move; `--record-order` lists them as move pairs, and `--key none`
aligns positionally instead.

A difference in representation alone — whitespace, `1.0` vs `1.00`, escape form,
object member order — never produces this verdict: such inputs compare identical
and exit 0 (`files are identical under semantic comparison`).

### "The result is coarser than I expected"

An over-cap container was coarsened. Read the coarsen notes and the hint block on
stderr: keying the array is usually the better fix, raising `--align-cap` (with a
matching `--memory-limit`) is the direct one, and `--profile thorough` sets
`--align-cap unlimited`.

Mind the budget when you reach for `thorough`: it removes the alignment cap, not
the memory limit, and aligning what the cap would have coarsened raises the run's
memory demand. On a constrained machine or under a tight `--memory-limit`, that can
turn a coarse-but-complete exit 1 into a `memory budget exceeded` exit 2. Raise
`--memory-limit` along with the cap — the hint block's computed flag pair does both
at once.

---

## 13. Limits at a glance

| Item | Limit |
|------|-------|
| Maximum input file size | 1 TB per side |
| Inputs per run | Exactly two; at most one may be `-` |
| Heap budget (default) | A fixed 4 GiB on every machine — the same limits on a laptop and a build server; raise it with `--memory-limit` to spend a big machine |
| Physical guard | The run aborts if heap use passes the budget or 85% of `MemAvailable` at launch, or if a container's memory use grows, since launch, past 85% of the room that was free under its cgroup limit at launch (v1 or v2, any ancestor, measured live). Never changes output |
| Temp budget (default) | Unlimited; set `--max-temp` to cap it |
| Alignment cap (default) | Memory budget / 640, never below 500000 — 6710886 at the default budget; override with `--align-cap` |
| Move-candidate pool (default) | 3919580 per side at the default budget; scales with the budget, never below 1000000 |
| Move-report eligibility | Only whole objects/arrays of at least 64 bytes report as `moved_from`/`moved_to`; displaced scalars and smaller containers report as `removed` + `added`. The floor and the pool are properties of the removed/added subtree join: a keyed reorder listed with `--record-order` is paired by key, so any container size qualifies (scalars still cannot be move records) |
| Keyed record order | Not compared: records pair by key value wherever they sit. Every keyed run discloses how many records must move; `--record-order` lists them as move pairs |
| Containers above the alignment cap | Reported as one coarse `region_changed` region each. The report and stderr disclose this; the meta line echoes the cap that produced the records |
| Exact alignment above the cap | `--large-arrays=exact` is planned, not available in this version |
| Compound key fields | At most 16 |
| Key field nesting | One level (`meta.id`) |
| `--path` | Single occurrence, anchored, no wildcards |
| Human view value width | 160 display characters, with an exact `… (+N bytes)` tail |
| Coarsen notes listed individually | First 10, then a totals line |
| Key notes listed individually | First 10 per class, then a totals line |
| Human aggregation thresholds | 4 consecutive scalar changes for `[N items rewritten]`; 100 records for the counts line |
| Distinct report paths | 65535 besides the root — fixed; does not scale with `--memory-limit`. Overflow records carry the aggregated path `"$"`, omit `ptr`, and are counted in one stderr note |
| Stored path length for `ptr` | Paths beyond the storage limit omit `ptr` rather than truncate it |
| `ptr` availability | Omitted when the address cannot be proven; never guessed |
| Report schema | Draft until the v1 freeze — tolerate added fields |
| Threading | Single-threaded by design |
| Platform | Linux (native, static binary) |

---

## 14. License and attribution

`gjxdiff` is **not** open-source software. It is proprietary software that is free to
use for individuals and small organizations. The `LICENSE` file shipped with the
binary is the governing text; this is a summary.

| You are | Interactive use | Automated use (CI, servers, scheduled jobs, production) |
|---------|-----------------|---------------------------------------------------------|
| An individual, or an organization with fewer than 100 employees and contractors (counting affiliates, in the current or preceding calendar year) | Free | Free |
| A larger organization | Free | Requires a commercial license |

Embedding or redistributing `gjxdiff`, or a product that includes or invokes it,
requires a commercial license — with one exception: an unmodified copy may be
included in a genuinely free application (no purchase price, subscription, paid tier,
advertising, sale of user data, and not used to promote a commercial product or
service), with attribution. Standalone redistribution of the unmodified software free
of charge, with all notices intact, is permitted.

**Attribution.** All free use requires author credit:

```
gjxdiff by Tibor Kovacs (Kotysoft)
```

A non-prominent placement is sufficient — an about, info or credits section of the
embedding application, or the accompanying documentation of a redistribution.

You may not modify or create derivative works of the software, reverse engineer it,
remove or alter its notices, sublicense or sell it standalone, or circumvent the
license terms.

The software is provided "as is", without warranty of any kind, and the author's
liability is limited to the maximum extent permitted by law.

Copyright (C) 2026 Tibor Kovacs (Kotysoft). Commercial licensing:
support@giantjson.com

`gjxdiff --version` and `gjxdiff --about` print the attribution block.

**Third-party components.** The binary statically links a number of
open-source components. Their licenses and copyright notices are listed in the
`THIRD-PARTY-NOTICES.md` file shipped alongside the binary, and
`gjxdiff --credits` prints the same list, so the attribution travels with the
binary itself.

---

## 15. Planned

None of the following are available in this version. They are listed so you know what
the reserved names mean.

| Item | Status |
|------|--------|
| `--large-arrays=exact` | Reserved. Exact alignment of containers above the cap, without coarse regions. The flag is accepted by the parser today and exits 2 with a message, so scripts written now keep working when it ships. |
| `--numeric-tolerance` | Planned. Compare numbers within a tolerance instead of exactly. |
| `--array-as-set` | Planned. Compare arrays as sets, ignoring order without a key. |
| `gz` / `zst` input | Planned. Transparent decompression of compressed inputs. |
| v1 report schema freeze | Planned. The meta line will then carry `"stability":"stable"`. Until then, gate on the `gjxdiff` version field and tolerate added fields. |
