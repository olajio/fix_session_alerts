# Watcher updates: `sdpmt_53001_updated.json` & `sdpmt_53002_from_admin_updated.json`

**Watchers updated:** `sdpmt_53001` (FIX Initial Logon Failure) and
`sdpmt_53002_from_admin` (FIX Heartbeat Check – FromAdmin).

## Problem

ServiceDesk tickets opened by these watchers had blank `cloud.account.name`
/ `cloud.region` fields (subject, description, `udf_sline_65102`,
`udf_sline_65104`), and priority was fixed at metadata-level regardless of
environment. Root cause: the webhook body referenced per-host cloud fields
that the top-level Painless transform never attached to `to_open` entries.

## Change set (identical shape in both watchers)

1. **Top-level `transform.script.source` (Painless)** — enrich each host
   built into `to_open` (and `to_comment` for 53002):
   - Build a `session_id → bucket` lookup from
     `ctx.payload.errors.aggregations.session_id.buckets` (in 53002, keyed
     by the raw `fix_sessionid`, before the `_FromAdmin` suffix is
     appended).
   - Attach `host.cloud_account_name` and `host.cloud_region` from
     `bucket.details.hits.hits.0._source` (53001) /
     `bucket.data.hits.hits.0._source` (53002 — the agg is named `data`,
     not `details`).
   - Set `host.ticket_priority` from `ctx.metadata.priority_by_account`
     keyed on `host.cloud_account_name`; fall back to
     `ctx.metadata.ticket_priority` if the account isn't mapped.
2. **`open_tickets` webhook body (`params.input_data`):**
   - Subject appended with `- {{ ctx.payload.cloud_account_name }}`.
   - Description gained a
     `Cloud Account: {{ ctx.payload.cloud_account_name }}` paragraph.
   - `priority.name` switched from `{{ ctx.metadata.ticket_priority }}` to
     `{{ ctx.payload.ticket_priority }}`.
   - `udf_sline_65102` = `{{ ctx.payload.cloud_account_name }}`,
     `udf_sline_65104` = `{{ ctx.payload.cloud_region }}`.
3. **Metadata** — added:
   ```json
   "priority_by_account": {
     "hedgeserv-app-prd": "1 - Critical",
     "hedgeserv-app-tst": "3 - Moderate"
   }
   ```

## Behavior after change

Tickets from `hedgeserv-app-prd` hosts open at **P1 "1 - Critical"**;
tickets from `hedgeserv-app-tst` open at **P3 "3 - Moderate"**; the cloud
account and region flow through the subject, description, and the two udf
fields on every ticket. Any account not in `priority_by_account` still
gets `metadata.ticket_priority` — so this change is safe to roll out to
sibling watchers before their metadata maps are populated.

## Not touched (intentional)

`ticket_group` (`"Monitoring and Analytics - Testing"`), the
`"TEST PLEASE IGNORE - "` subject prefix, and every other input / action /
condition. Both files structurally validated: `headers` sits as a sibling
of `params` under `webhook`, JSON parses cleanly.

## Follow-up fix: null_pointer_exception on bucket lookup

After the initial rollout, `sdpmt_53002_from_admin_updated.json` threw
`null_pointer_exception ... cannot access method/field [data] from a null
def reference` at
`def source = bucket.data.hits.hits.0._source`.

Cause: `session.replace('_FromAdmin', '')` replaces **all** occurrences,
not just the trailing suffix appended by the `sessions` transform. If a
real `labels.fix_sessionid` value ends in (or contains) `_FromAdmin`, the
strip removes too much, and the cleaned name no longer matches any
errors-bucket key. `session_details.get(host.session_id)` returns null,
and dereferencing `bucket.data` throws NPE.

Two targeted changes to the top-level transform:

1. **Strip only the trailing suffix.** Replace
   `session.replace('_FromAdmin', '')` with an
   `endsWith` + `substring` check:
   ```painless
   String suffix = '_FromAdmin';
   if (session.endsWith(suffix)) {
     host.session_id = session.substring(0, session.length() - suffix.length());
   } else {
     host.session_id = session;
   }
   ```
2. **Null-check the bucket lookup.** If `session_details.get(...)` ever
   returns null in the future, fall back to blank cloud fields and
   metadata priority instead of throwing:
   ```painless
   def bucket = session_details.get(host.session_id);
   if (bucket != null) {
     def source = bucket.data.hits.hits.0._source;
     host.cloud_account_name = source.cloud.account.name;
     host.cloud_region = source.cloud.region;
   } else {
     host.cloud_account_name = '';
     host.cloud_region = '';
   }
   ```

The same null-check was also added to `sdpmt_53001_updated.json` for
consistency, even though 53001 has no suffix step and shouldn't hit the
null path.

## Follow-up 2: split per-account watchers (`sdpmt_53001_prd_debug.json`, `sdpmt_53001_tst_debug.json`)

The metadata-driven `priority_by_account` map works only when the log for
a session is present in the errors search, because both the cloud
account and the priority are derived from that log bucket. That's fine
for the debug shape (`active.contains(session)`, alert-when-log-exists),
but 53001 in production runs on `!active.contains(session)` —
alert-when-log-is-missing. When the log is missing there is no bucket to
enrich from, so `cloud_account_name` ends up blank, the priority-map
lookup misses, and every ticket falls back to
`metadata.ticket_priority = "1 - Critical"` regardless of environment.

`fix_session_schedule` does not carry `cloud.account.name` /
`cloud.region`, and no field on the host/target identifies the account.
So the account can only be recovered by scoping the watcher itself to a
single account.

Two new files were added:

- `sdpmt_53001_prd_debug.json` — scoped to `hedgeserv-app-prd`, priority `1 - Critical`.
- `sdpmt_53001_tst_debug.json` — scoped to `hedgeserv-app-tst`, priority `3 - Moderate`.

The changes vs `sdpmt_53001_debug.json` (identical shape in both new files
except for the three metadata values):

1. **`errors` search filter** — `terms.cloud.account.name` array replaced
   with a `term` filter templated to the metadata value:
   ```json
   { "term": { "cloud.account.name": "{{ ctx.metadata.cloud_account_name }}" } }
   ```
2. **Metadata** — added `cloud_account_name` as a constant; `id` renamed
   to `sdpmt_53001_prd` / `sdpmt_53001_tst`; `ticket_priority` set to the
   correct value for that environment. `priority_by_account` is not
   used here (single-account scope makes the map redundant).
3. **Webhook body** —
   - `subject` and `description` reference `{{ ctx.metadata.cloud_account_name }}` (constant, always populated).
   - `udf_sline_65102` = `{{ ctx.metadata.cloud_account_name }}`.
   - `priority.name` = `{{ ctx.metadata.ticket_priority }}` (also constant).
   - `udf_sline_65104` = `{{ ctx.payload.cloud_region }}` (blank when log missing).
4. **Transform** — simplified: no more priority lookup, no more
   `cloud_account_name` enrichment. Only `cloud_region` is enriched from
   the log bucket when present, blank when missing.

Result: whether the log is present or missing, `cloud_account_name` and
`ticket_priority` are always correct because they come from metadata, and
tickets from `hedgeserv-app-tst` outages get P3 as expected.
`cloud_region` is populated when the log exists (debug flow) and blank
when it doesn't (real prod alert on missing log); acceptable given no
alternative source is available.

Both files ship with the production `sessions` transform shape
(`!active.contains(session)`), matching the flipped
`sdpmt_53001_debug.json` on `main`.

The same shape has now been mirrored onto `sdpmt_53002_from_admin`:

- `sdpmt_53002_from_admin_prd_debug.json` — scoped to `hedgeserv-app-prd`, priority `1 - Critical`.
- `sdpmt_53002_from_admin_tst_debug.json` — scoped to `hedgeserv-app-tst`, priority `3 - Moderate`.

Same four hunks apply, with two 53002-specific extras:

- The top-level transform continues to build **both** `to_open` and
  `to_comment` (53002 opens or comments existing tickets); only the
  `cloud_account_name` / `ticket_priority` / priority-map derivations were
  removed. `host.session_id` is now stripped with
  `endsWith` + `substring` (not `replace(...)`) to avoid the
  `_FromAdmin`-inside-a-session-id trap documented above.
- The `sessions` transform ships with the production shape
  `!active.contains(session)` matching the flipped 53001 files.

## Method notes for the next sibling watcher

- **Copy the debug file, then apply the same 4 hunks.** Metadata addition,
  webhook body (subject/description/priority/udf_65102/udf_65104), and the
  transform enrichment.
- **Watch for a `_FromAdmin`-style suffix** in the sessions transform (or
  any other suffix). If present, the errors-bucket lookup must be keyed on
  the *cleaned* session id (`host.session_id`), not the loop variable —
  and prefer `endsWith` + `substring` over `replace(...)` so real session
  ids that contain the marker aren't over-stripped (see the follow-up fix
  above).
- **Always null-check the bucket lookup** before dereferencing
  `bucket.details.…` / `bucket.data.…`. `sessions._value` is derived from
  the same errors buckets, so this branch shouldn't fire, but the null
  check prevents a bad strip (or any other future mismatch) from crashing
  the watcher.
- **Watch for the top_hits agg name.** 53001 calls it `details`; 53002
  calls it `data`. The path in the transform
  (`bucket.details…` vs `bucket.data…`) must match.
- **Structural gotcha.** In the webhook block, `headers` is a sibling of
  `params`, not a child. Nesting it inside `params` produces
  `[script] unknown field [Content-Type]` at parse time.
