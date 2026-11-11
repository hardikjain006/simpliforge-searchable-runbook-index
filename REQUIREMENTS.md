# Searchable Runbook Index — Requirements

## 1. Goal

Build a searchable runbook index from generated telemetry that produces one
actionable result, surfaced through a command-line interface (CLI) or a static
page.

Terms used in this document:

- **Telemetry**: machine-generated operational records (metrics, events, log
  excerpts) supplied as input data files. Whether the files are hand-written,
  recorded, or synthetically generated does not matter; only the file format
  below is contractual.
- **Runbook entry**: one troubleshooting unit with a stable id, a title, the
  telemetry signals that trigger it, and ordered remediation steps.
- **Index**: the mapping from telemetry-derived signals to runbook entries,
  built from the telemetry input.
- **Actionable result**: a single runbook entry returned for a query, always
  including at least one concrete remediation step the operator can act on.

## 2. Scope

In scope:

- Ingesting telemetry input files in the format defined in section 3.
- Building the runbook index from those files.
- Answering a free-text query with exactly one actionable result.
- Exactly one user surface: either a CLI or a static page (implementer
  chooses; both surfaces share the result contract in section 5).

Out of scope:

- Collecting live telemetry; all input arrives as files.
- Network access at build time or query time; everything works offline.
- Automatic remediation; the product recommends, the operator acts.

## 3. Public inputs

### 3.1 Telemetry files

- Input is one or more UTF-8 JSON files. Each file holds an array of telemetry
  records.
- Each record has these fields:

| Field | Type | Required | Meaning |
|---|---|---|---|
| `timestamp` | string (RFC 3339) | yes | When the signal was observed |
| `service` | string, non-empty | yes | Emitting service or component |
| `signal` | string, non-empty | yes | Metric or event name, e.g. `api_latency_p99` |
| `severity` | one of `info`, `warn`, `critical` | yes | Signal severity |
| `message` | string | no | Human-readable detail, defaults to `""` |

- Records carrying additional unknown fields are accepted; unknown fields are
  ignored.
- A record is an **invalid record** (skipped, never failing) when it is a
  JSON object in which any required field is missing, has the wrong JSON
  type, or — for `service`, `signal`, or `timestamp` — is an empty string;
  or when `severity` is outside the enum. `timestamp` is valid iff it is a
  non-empty string (producers SHOULD emit RFC 3339; consumers MUST NOT
  reject a non-empty string solely for calendar validity). Skipped records
  never appear in the index and never cause a failure; the build reports
  how many records were skipped (see section 6, item 6).

### 3.2 Telemetry fixtures shipped with the repo

- The repository ships at least one sample telemetry file under `data/` so
  the index can be built and queried without external input.
- Fixture content is ordinary project data, not a test oracle: behavior must
  follow the rules in this document for any conforming input, not just the
  fixtures.

## 4. Index construction (observable behavior)

- Building the index from a fixed set of input files is deterministic: the
  same files always yield the same index. The build processes input files
  in sorted byte order of their paths and records in array order.
- Partition rule: exactly one runbook entry exists per distinct
  (`service`, `signal`) pair observed among valid records (exact,
  case-sensitive string equality). Every valid record contributes to
  exactly the entry for its pair; pairs never observed produce no entries.
- Each runbook entry has:
  - `id`: stable, unique, lowercase slug derived as
    `slug(service) + "-" + slug(signal)`, where `slug(x)` lowercases ASCII
    `A-Z` to `a-z`, replaces each maximal run of characters outside
    `[a-z0-9]` with a single `-`, and trims leading/trailing `-`
    (an empty slug part becomes `unnamed`). Two distinct pairs yielding
    the same `id` is a duplicate-id build error (see section 6, item 4).
  - `title`: exactly `"<service>: <signal>"` using the first-observed
    original spellings for that pair; always non-empty because both
    fields are non-empty.
  - `triggers`: the singleton set containing exactly that pair,
    e.g. `{("checkout", "api_latency_p99")}`.
  - `actions`: ordered list with at least one step; every step is a
    non-empty string. The first step is exactly
    `Investigate "<signal>" on service "<service>" (highest observed severity: <severity>).`
    using the pair's first-observed spellings and its highest observed
    severity (`critical` > `warn` > `info`; ties keep the first observed).
    Any additional steps MUST also be non-empty and deterministically
    derived from that pair's records (no timestamps, randomness, or
    network input).
- Duplicate entry ids are a build error (see section 6, item 4).

## 5. Query behavior (observable behavior)

- The query input is a free-text string supplied either as a CLI argument or
  as the search input of the static page.
- Every query returns **exactly one** result:
  - If at least one entry matches, the result is the best-matching runbook
    entry and contains its `id`, `title`, and its full ordered `actions`
    list. The first action is the primary recommended step.
  - If no entry matches, the result is an explicit empty result whose
    content MUST contain the case-insensitive ASCII phrase
    `no runbook match` (other wording around it is free). A matching
    result MUST NOT contain that phrase.
- Tokenization (applies identically to queries and to entry
  `service`/`signal`/`title` text): ASCII-lowercase `A-Z` to `a-z`, then
  split on every character outside `[A-Za-z0-9]`; tokens are the maximal
  non-empty runs. Non-ASCII letters are separators, not token characters.
- Matching and ranking are defined over token sets: the entry token set is
  the union of the entry's `service`, `signal`, and `title` tokens; the
  query token set is the set of distinct non-empty query tokens. An entry
  matches iff the set intersection is non-empty. The score is the size of
  that set intersection: repeated query tokens collapse, and one query
  token hitting several fields of the same entry still counts once. A
  query with no tokens matches nothing. Ranking: the highest score wins;
  ties are broken deterministically by ascending entry `id` (byte order).
- The same corpus and the same query always return the same result.
- Numbered `actions` format (both surfaces): every action appears on its
  own line, in order, each line prefixed by its 1-based index, a period,
  and a single space (e.g. `1. Investigate "api_latency_p99" on service
  "checkout" (highest observed severity: critical).`).

### 5.1 CLI surface (if chosen)

- `runbook search "<query>"` prints the single result as human-readable
  text containing the entry `id`, `title`, and `actions` in the numbered
  format above.
- Exit code `0` when an entry matches, `1` for the explicit empty result,
  `2` for usage or input errors.

### 5.2 Static-page surface (if chosen)

- A single page with a search input renders the same single result content
  (`id`, `title`, `actions` in the numbered format above) or the explicit
  empty-result statement containing `no runbook match`. The page works
  from local files with no server and no network.

## 6. Boundary cases and errors

1. Empty telemetry input: a file containing a valid JSON array with zero
   elements (e.g. `[]`, JSON whitespace around it permitted) builds
   successfully and every query returns the empty result. A zero-byte
   file, a whitespace-only file, or any non-JSON content is case 2, not
   this case.
2. Malformed telemetry file: the build fails with a message naming the
   file (CLI exit code `2`) when the file content is not valid JSON, when
   the top-level JSON value is not an array, or when any array element is
   not a JSON object (e.g. `[123]`, `[null]`, `["x"]`, or a mixed array
   with valid records plus non-object elements). Per-record skipping
   applies ONLY to elements that are JSON objects failing a required-field
   constraint (see 3.1).
3. Empty or whitespace-only query: treated as matching nothing; returns the
   empty result (exit `1` on the CLI), never an error.
4. Duplicate runbook entry ids at build time: build fails with a message
   naming the duplicate id. CLI exit code `2`.
5. Telemetry file that cannot be read (missing path, permission): build
   fails with a message naming the path. CLI exit code `2`.
6. Skip-count reporting: let `N` be the total number of skipped invalid
   records across all input files. Skips never fail the build and never
   change the section 5.1 exit codes. Every CLI invocation that reads
   telemetry (including `runbook search`) writes exactly one line
   `skipped_invalid_records: N` to stderr (`N` in decimal, `0` when
   nothing was skipped). A build-only invocation, if the implementer
   provides one, exits `0` on build success including with `N > 0`. The
   static page, if chosen, renders the same `skipped_invalid_records: N`
   string in its footer. Machine check: one line matching
   `^skipped_invalid_records: [0-9]+$`.

## 7. Determinism and environment

- No network access is required or used to build the index or answer
  queries.
- Diagnostic marker: every diagnostic or timestamp-bearing line the
  product emits (stderr, logs) MUST begin with the exact literal prefix
  `diagnostic:`. No result-content line (CLI stdout, page result text,
  persisted index data) may begin with that prefix or embed a timestamp.
  On the static page, diagnostics outside the result content, if any,
  appear only as HTML comments beginning `<!-- diagnostic:`.
- Byte-identical scope: over unchanged inputs, repeated runs produce
  byte-identical (i) CLI stdout result text for a fixed query,
  (ii) generated static-page bytes excluding `<!-- diagnostic:` comments,
  and (iii) every persisted index file at the implementer-documented
  output path(s) (the implementer MUST document those paths in the repo).
  `diagnostic:` stderr lines are excluded from the comparison.

## 8. Unresolved requirements

None. Every behavior above is implementable and testable from public
information in this repository; no private task-generation information is
needed. If implementation surfaces a genuine ambiguity in this contract,
it belongs in a later requirements iteration as a newly discovered task,
not as a silent deviation.
