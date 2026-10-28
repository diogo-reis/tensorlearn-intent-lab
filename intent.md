# Intent

## Goal
Give the support coordinator a safe way to preview tickets described in a CSV
file before any import is approved. The outcome is a pure function that reads
CSV text and reports either the tickets it would create or exactly why the
file is rejected, so bad data (missing titles, bad priorities, too many rows,
wrong columns) is caught before anything is actually imported. This round
only produces this intent document; the function itself is built in a later
round from what is written here.

## Scope
Allowed: add `src/import-tickets.mjs` (the preview function) and
`tests/import-tickets.test.mjs` (its tests) in the future implementation
round. Excluded: `src/tickets.mjs`, anything under `.github/`, and
`CLAUDE.md` must not change — the preview reuses the existing ticket shape
but does not alter the module that defines it. Out of scope entirely: any
UI, database, network call, file write, or new package dependency. The
preview only parses text in memory and returns a result; it never persists
or imports anything. In this current round, only `intent.md` changes.

## Constraints
The preview is a pure function over CSV text: same input always produces the
same output, with no side effects. Only `title` (required) and `priority`
(optional) are recognized columns; any other column name, a missing `title`
column, a duplicate column name, or a structurally malformed row fails with
`unknown-column`, including a completely empty CSV with no header row at
all. Every value is trimmed of surrounding whitespace before it is checked.
An empty title after trimming fails with `empty-title`. Priority must be
`low`, `normal`, or `urgent`; a missing or blank priority defaults to
`normal`, and any other value fails with `invalid-priority`. At most 100
data rows are accepted, and that row count is checked before any row's
values are validated, so an oversized file fails fast with `row-limit`
rather than a content error. CSV quoting must be respected, so a quoted
field may safely contain a comma. On any failure the result is only
`{"error":"..."}` — never a partial ticket list alongside an error.

## Plan
1. Parse the header row first and validate its columns (reject unknown,
   missing, or duplicate column names, or no header at all, with
   `unknown-column`) before looking at any data row.
2. Count the data rows and fail with `row-limit` if there are more than 100,
   before validating anything about their content.
3. For each remaining row, parse it respecting CSV quoting, trim every
   field, then validate `title` (non-empty) and `priority` (blank becomes
   `normal`; otherwise must be `low`/`normal`/`urgent`), stopping on the
   first failing row with the matching error code.
4. If every row validates, return `{"tickets":[...]}` in the original input
   order.
5. Write `tests/import-tickets.test.mjs` to cover each rule above.
Ask the user before writing any implementation code, and ask again before
changing or expanding this plan or scope.

## Acceptance
Positive: a CSV with an explicit valid priority, and one that omits or
blanks priority and gets the `normal` default, both produce a `tickets`
list with trimmed values in input order; a quoted title containing a comma
is parsed as one field, not split. Negative: an empty title, an
unrecognized/duplicate/missing column, an invalid priority value, and more
than 100 data rows each produce the matching `{"error":"..."}` code and no
ticket list. The JSON contract below is the authoritative, checked version
of these cases.

```json
{
  "schema": 1,
  "change": "ticket-csv-preview",
  "allowed_changes": ["src/import-tickets.mjs", "tests/import-tickets.test.mjs"],
  "protected_paths": ["src/tickets.mjs", ".github/", "CLAUDE.md"],
  "non_goals": ["ui", "database", "network"],
  "requirements": {
    "columns": ["title", "priority"],
    "default_priority": "normal",
    "valid_priorities": ["low", "normal", "urgent"],
    "reject_unknown_columns": true,
    "reject_empty_title": true,
    "max_rows": 100,
    "side_effects": "preview-only"
  },
  "verification": ["npm test", "git diff --check"],
  "approval": "ask-before-implementation",
  "acceptance_cases": [
    {
      "csv": "title,priority\nPrinter issue,urgent\n",
      "expected": {"tickets":[{"title":"Printer issue","priority":"urgent"}]}
    },
    {
      "csv": "title,priority\nFix printer,\n",
      "expected": {"tickets":[{"title":"Fix printer","priority":"normal"}]}
    },
    {
      "csv": "title,priority\n\"Printer, paper jam\",low\n",
      "expected": {"tickets":[{"title":"Printer, paper jam","priority":"low"}]}
    },
    {
      "csv": "title,priority\n   ,urgent\n",
      "expected": {"error": "empty-title"}
    },
    {
      "csv": "title,status\nPrinter issue,open\n",
      "expected": {"error": "unknown-column"}
    },
    {
      "csv": "title,priority\nPrinter issue,critical\n",
      "expected": {"error": "invalid-priority"}
    },
    {
      "csv": "title,priority\nPrinter issue,normal\n",
      "repeat": 101,
      "expected": {"error": "row-limit"}
    }
  ]
}
```

## Verification
Verify with `npm test` and `git diff --check`; both must pass, and only
`src/import-tickets.mjs` and `tests/import-tickets.test.mjs` should appear
in the diff versus this intent. Before writing any code, Claude should ask
the user for approval of this plan, and should ask again before touching
any file outside the two listed above or before adding any requirement not
already stated here.


