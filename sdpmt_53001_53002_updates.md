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

## Method notes for the next sibling watcher

- **Copy the debug file, then apply the same 4 hunks.** Metadata addition,
  webhook body (subject/description/priority/udf_65102/udf_65104), and the
  transform enrichment.
- **Watch for a `_FromAdmin`-style suffix** in the sessions transform (or
  any other suffix). If present, the errors-bucket lookup must be keyed on
  the *cleaned* session id (`host.session_id`), not the loop variable.
- **Watch for the top_hits agg name.** 53001 calls it `details`; 53002
  calls it `data`. The path in the transform
  (`bucket.details…` vs `bucket.data…`) must match.
- **Structural gotcha.** In the webhook block, `headers` is a sibling of
  `params`, not a child. Nesting it inside `params` produces
  `[script] unknown field [Content-Type]` at parse time.
