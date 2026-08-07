# Watcher updates: `sdpmt_53001_updated.json` & `sdpmt_53002_from_admin_updated.json`

**Watchers updated:** `sdpmt_53001` (FIX Initial Logon Failure) and
`sdpmt_53002_from_admin` (FIX Heartbeat Check — FromAdmin).

## Problem

ServiceDesk tickets opened by these watchers had blank
`cloud.account.name` / `cloud.region` fields (subject, description,
`udf_sline_65102`, `udf_sline_65104`), and priority defaulted to
`metadata.ticket_priority` for every environment. Root cause: the
webhook body referenced per-host cloud fields that the top-level
Painless transform never attached to `to_open` entries.

A first pass enriched those fields from the filebeat errors bucket. That
worked for the debug shape (`active.contains(session)`, alert-when-log-
exists) but broke in production, where the transform runs on
`!active.contains(session)` — alert-when-log-is-missing. With no log
bucket to enrich from, `cloud_account_name` came back blank, the
priority map missed, and every ticket collapsed to
`"1 - Critical"` regardless of environment. `hedgeserv-app-tst`
outages paged as P1 instead of P3.

## Change set (identical shape in both watchers)

Once `cloud.account.name` was added to `fix_session_schedule`, we could
identify the account for every session — log present or missing —
without splitting the watcher per account. Two hunks per file:

1. **`groups.aggs.hosts` — add a `top_hits` sub-agg** that returns
   `cloud.account.name` from the schedule doc for each host:
   ```json
   "hosts": {
     "terms": { "field": "target.keyword", "size": 5000 },
     "aggs": {
       "account": {
         "top_hits": {
           "size": 1,
           "_source": { "include": ["cloud.account.name"] }
         }
       }
     }
   }
   ```
2. **Top-level `transform.script.source`** — two-map enrichment:
   ```painless
   // session -> cloud.account.name from fix_session_schedule (always populated)
   Map session_to_account = new HashMap();
   for (def bucket: ctx.payload.groups.aggregations.hosts.buckets) {
     def source = bucket.account.hits.hits.0._source;
     session_to_account.put(bucket.key, source.cloud.account.name);
   }

   // session -> errors bucket (present only when the log is present); used for cloud.region
   Map session_details = new HashMap();
   for (def bucket: ctx.payload.errors.aggregations.session_id.buckets) {
     session_details.put(bucket.key, bucket);
   }

   // inside the existing per-session loop:
   host.cloud_account_name = session_to_account.getOrDefault(session, '');
   def bucket = session_details.get(session);
   if (bucket != null) {
     def source = bucket.details.hits.hits.0._source;  // 53002 uses bucket.data instead
     host.cloud_region = source.cloud.region;
   } else {
     host.cloud_region = '';
   }
   ```
   `priority_by_account` continues to drive the priority; because
   `cloud_account_name` is now always populated, the map lookup succeeds
   whether the log is present or not.

Also present from the earlier round (unchanged this pass):

- `metadata.priority_by_account`:
  ```json
  {
    "hedgeserv-app-prd": "1 - Critical",
    "hedgeserv-app-tst": "3 - Moderate"
  }
  ```
- Webhook body references `{{ ctx.payload.cloud_account_name }}`,
  `{{ ctx.payload.cloud_region }}`, `{{ ctx.payload.ticket_priority }}`
  in subject, description, `udf_sline_65102`, `udf_sline_65104`, and
  `priority.name`.

## Behavior after change

- Every ticket carries the correct `cloud.account.name` in subject,
  description, and `udf_sline_65102`, whether or not the log was present
  when the alert fired.
- `cloud.region` is populated when the log exists; blank on real
  missing-log alerts (no alternative source for region — schedule doc
  doesn't carry it).
- Priority is driven by the schedule-derived account:
  `hedgeserv-app-prd` → P1 "1 - Critical", `hedgeserv-app-tst` → P3
  "3 - Moderate". Accounts not in `priority_by_account` fall back to
  `metadata.ticket_priority`, so this stays safe to roll out to sibling
  watchers before their maps are populated.

## Not touched (intentional)

`ticket_group` (`"Monitoring and Analytics - Testing"`), the
`"TEST PLEASE IGNORE - "` subject prefix, cron, index paths, webhook
`headers`, and the `sessions` transform's `active.contains(session)` /
`!active.contains(session)` line — all left as-is. Both files
structurally validated: `headers` sits as a sibling of `params` under
`webhook`, JSON parses cleanly.

## Method notes for the next sibling watcher

- **Copy `_updated.json`, then apply the same 2 hunks:** `top_hits`
  sub-agg on the schedule aggregation, and the two-map enrichment in the
  top-level transform. `priority_by_account` and the webhook body
  references are already in place if you seed from an updated file.
- **Watch for the top_hits agg name on the errors side.** 53001 calls it
  `details`; 53002 calls it `data`. The path in the transform
  (`bucket.details…` vs `bucket.data…`) must match. The schedule sub-agg
  added by this change is named `account` in both watchers.
- **Watch for a `_FromAdmin`-style suffix.** If the `sessions` transform
  appends one (53002 does), the schedule map is keyed by the raw
  `fix_sessionid` — look it up with the *cleaned* `host.session_id`, and
  prefer `endsWith` + `substring` over `replace('_FromAdmin', '')` so
  real session ids that contain the marker aren't over-stripped.
- **Null-check the errors bucket before dereferencing.** On real
  missing-log alerts the errors bucket will not exist for the session,
  so region falls back to blank; the schedule-side account is always
  populated separately.
- **Structural gotcha.** In the webhook block, `headers` is a sibling of
  `params`, not a child. Nesting it inside `params` produces
  `[script] unknown field [Content-Type]` at parse time.

---

## History (superseded approaches)

For posterity — the path here wasn't straight.

- **First pass:** enrich `cloud_account_name`, `cloud_region`, and
  priority from the filebeat log bucket. Worked in debug mode
  (`active.contains(session)`) but produced blank fields and universal
  P1 in the production `!active.contains(session)` shape.
- **Second pass:** split into per-account watchers
  (`sdpmt_53001_prd_debug.json` and `sdpmt_53001_tst_debug.json`, plus
  the same for 53002_from_admin). Each hard-scoped `cloud.account.name`
  in the errors query and hard-coded `metadata.ticket_priority`. Solved
  the missing-log problem but doubled the number of watchers to
  maintain. Files removed once superseded.
- **Current:** `cloud.account.name` added to `fix_session_schedule`,
  which lets a single combined watcher identify the account for every
  session regardless of log presence, restoring correct priority
  routing without the per-account file duplication.
