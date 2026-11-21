# archive-relay

A small private-data mover. It fetches a ZIP archive of CSV tables from a
source you configure, parses every table at runtime on the runner, and
delivers the rows as JSON to an ingestion endpoint you control.

- **Manual runs only** — nothing runs on a timer. You press the button
  (Actions → *refresh* → **Run workflow**), it does one pass, it finishes.
- **Standard library only** — the pipeline is a single Python script, no
  packages to install.
- **Nothing is kept** — the archive and parsed rows live in a scratch dir
  that is wiped before and after every run, and the runner VM itself is
  destroyed by GitHub after the job. Failure artifacts contain the audit log
  only, never row data.
- **Discreet by default** — table names are replaced with short SHA-256
  hashes in logs and summaries (Actions logs on a public repo are
  world-readable). Real names travel only inside the encrypted/authorized
  request bodies to your endpoint. Set the `DEBUG_NAMES` repo variable to
  `true` if you want plain names while debugging.

## Setup

Add these **repository secrets** (Settings → Secrets and variables → Actions):

| Secret | Required | Meaning |
| --- | --- | --- |
| `SOURCE_URL` | yes | Full URL of the ZIP archive to fetch |
| `SOURCE_TOKEN` | no | Bearer token if the source needs auth |
| `TARGET_URL` | yes* | Ingestion endpoint that accepts the JSON POSTs |
| `TARGET_TOKEN` | yes* | Bearer token sent as `Authorization: Bearer` |

\* Required for real runs; a **dry run** works without them.

Optional **repository variables** (same settings page, "Variables" tab):

| Variable | Default | Meaning |
| --- | --- | --- |
| `TARGET_GZIP` | `false` | `true` gzips request bodies (`Content-Encoding: gzip`) |
| `DEBUG_NAMES` | `false` | `true` logs plain table names instead of hashes |
| `HTTP_TIMEOUT` | `300` | Per-request timeout in seconds |
| `MAX_UNCOMPRESSED_MB` | `2048` | Refuses archives that decompress beyond this |

Run inputs on the **Run workflow** form:

- **dry_run** — download and parse, deliver nothing. Useful for a first run.
- **rows_per_post** — rows per delivery request (default `10000`). Lower it if
  your endpoint rejects large bodies; raise it to reduce request count.

## What one run does

1. `GET SOURCE_URL` (with retries; any 2xx = fetched). ETag/Last-Modified are
   recorded in the manifest. Nothing is cached between runs — every run
   downloads fresh.
2. Unzip (flattened) with a decompression-size cap; `.txt`/`.csv` members
   become tables. Junk entries (`__MACOSX`, dotfiles) are skipped.
3. Parse each table at runtime: header row + all data rows, UTF-8 (BOM
   tolerated). Values are passed through as strings; short rows are padded,
   long rows truncated. Zero-row tables are reported but send no chunks.
4. Deliver to `TARGET_URL` in three event kinds (below). Every request
   retries up to 5 times with exponential backoff + jitter on 5xx/429/408 and
   network errors; other 4xx (e.g. 401/403) fail fast — the endpoint is
   telling us the config is wrong.
5. Wipe the scratch dir. Audit log survives for diagnostics.

## Payload contract

All requests: `POST TARGET_URL`, `Authorization: Bearer <TARGET_TOKEN>`,
`Content-Type: application/json` (plus `Content-Encoding: gzip` when
`TARGET_GZIP=true`). Dispatch on the `event` field.

**`begin` — one per run:**

```json
{
  "schema": "archive-relay/v1",
  "event": "begin",
  "batch_id": "9f2c1a...",
  "generated_at": "2026-11-21T09:00:00Z",
  "dry_run": false,
  "rows_per_post": 10000,
  "source": { "url": "...", "final_url": "...", "etag": "\"...\"",
               "last_modified": "...", "bytes": 18400000, "sha256": "..." },
  "datasets": [ { "name": "items", "file": "items.csv", "rows": 1234,
                  "bytes": 45678, "sha256": "...", "chunk_count": 1 } ]
}
```

**`chunk` — one per slice of a table:**

```json
{
  "schema": "archive-relay/v1",
  "event": "chunk",
  "batch_id": "9f2c1a...",
  "dataset": "items",
  "chunk_index": 1,
  "chunk_count": 1,
  "columns": ["code", "label"],
  "rows": [ { "code": "A1", "label": "Example" } ]
}
```

**`complete` — one per run, after the last chunk:**

```json
{
  "schema": "archive-relay/v1",
  "event": "complete",
  "batch_id": "9f2c1a...",
  "status": "ok",
  "datasets": 8, "rows_delivered": 91234, "chunks_delivered": 9,
  "elapsed_seconds": 121.4
}
```

Receiver sketch (your side): buffer chunks keyed by `batch_id` + `dataset`;
on `begin` remember the manifest (per-table row counts and SHA-256 of the
source files); on `complete` verify counts and commit the batch. Any 2xx
response means "accepted"; anything the script cannot get a 2xx for (after
retries) fails the run.

## Failure handling and audit

- Progress and every retry/failure go to the run log **and** a JSON-lines
  audit file. If the run fails, the audit file is uploaded as an artifact
  (`audit-<run id>`, 7-day retention).
- Secret values (URLs, tokens) are never written to logs or the audit file;
  the script scrubs them explicitly.
- Exit code is non-zero on any failure, with a `::error::` annotation on the
  exact cause (config, source, parse, or delivery).

## Privacy / cleanup

- Row data exists only in the runner's scratch dir (`WORK_DIR`) and in
  request bodies, both gone by the end of the job.
- The scratch dir is wiped by the script itself (even on failure) and again
  by a workflow step that always runs.
- No state is persisted between runs: no caches, no commits, no artifacts on
  success.
