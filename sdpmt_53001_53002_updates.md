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
   // session -> cloud.account.name from fix_session_schedule (available even when the log is missing).
   // Defensive: skip hosts whose schedule doc has no account populated, and handle both a
   // nested {"cloud": {"account": {"name": "..."}}} shape and a flat {"cloud.account.name": "..."} shape.
   Map session_to_account = new HashMap();
   for (def bucket: ctx.payload.groups.aggregations.hosts.buckets) {
     if (bucket.account == null || bucket.account.hits == null
         || bucket.account.hits.hits == null || bucket.account.hits.hits.length == 0) {
       continue;
     }
     def source = bucket.account.hits.hits.0._source;
     if (source == null) {
       continue;
     }
     def acct = null;
     if (source instanceof Map && source.containsKey('cloud.account.name')) {
       acct = source.get('cloud.account.name');
     } else if (source.cloud != null && source.cloud.account != null) {
       acct = source.cloud.account.name;
     }
     if (acct != null) {
       session_to_account.put(bucket.key, acct);
     }
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
     def source = bucket.data.hits.hits.0._source;
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
`"TEST PLEASE IGNORE - "` subject prefix, cron, index paths, and webhook
`headers` — all left as-is. Both files structurally validated: `headers`
sits as a sibling of `params` under `webhook`, JSON parses cleanly.

The `sessions` transform in all four watcher files
(`sdpmt_53001_{debug,updated}.json`,
`sdpmt_53002_from_admin_{debug,updated}.json`) now uses
`!active.contains(session)` — the production shape, which fires the
alert when the expected log is *missing*.

## Method notes for the next sibling watcher

- **Copy `_updated.json`, then apply the same 2 hunks:** `top_hits`
  sub-agg on the schedule aggregation, and the two-map enrichment in the
  top-level transform. `priority_by_account` and the webhook body
  references are already in place if you seed from an updated file.
- **Two named top_hits sub-aggs to keep straight.** The errors-side one
  under `session_id` is `data` in both watchers (53001's earlier `details`
  name has been renamed for consistency), dereferenced as
  `bucket.data.hits.hits.0._source`. The schedule-side one added by this
  change is `account`, dereferenced as
  `bucket.account.hits.hits.0._source`.
- **Watch for a `_FromAdmin`-style suffix.** If the `sessions` transform
  appends one (53002 does), the schedule map is keyed by the raw
  `fix_sessionid` — look it up with the *cleaned* `host.session_id`, and
  prefer `endsWith` + `substring` over `replace('_FromAdmin', '')` so
  real session ids that contain the marker aren't over-stripped.
- **Null-check the errors bucket before dereferencing.** On real
  missing-log alerts the errors bucket will not exist for the session,
  so region falls back to blank; the schedule-side account is always
  populated separately.
- **Defensive schedule-side lookup.** A schedule doc that hasn't been
  backfilled with `cloud.account.name`, or a mapping that stores the
  field as the flat dotted key `"cloud.account.name"` rather than as a
  nested `cloud.account.name` object, will make `source.cloud.account.name`
  throw `null_pointer_exception: cannot access method/field [account]
  from a null def reference`. The account-lookup snippet above handles
  both shapes and skips hosts where neither is present, leaving that
  ticket's account blank (priority falls back to
  `metadata.ticket_priority`) instead of failing the whole transform.
- **Structural gotcha.** In the webhook block, `headers` is a sibling of
  `params`, not a child. Nesting it inside `params` produces
  `[script] unknown field [Content-Type]` at parse time.

---

## Day-of-week schedules (`days`) — `FIX.4.4:HSDIAMETER44->NYFIX44`

### Request

| FIX Session ID | Expected Logon (ET) | Connection Window (ET) | cloud.account.name |
| --- | --- | --- | --- |
| `FIX.4.4:HSDIAMETER44->NYFIX44` | 17:47 | 17:45 to 17:40 (Su, Mo, Tu, We, Th) | `hedgeserv-app-prd` |

Unlike every other target in `fix_session_schedule`, this one is restricted to
specific days. Two things make it different from the existing docs:

1. **The window wraps.** 17:47 -> 17:40 is a ~23h53m window: the session logs on
   one evening and stays connected until late the following afternoon.
2. **`days` are logon days, not up days.** The Thursday 17:47 window is the last
   of the week and does not close until Friday 17:40, so the session is
   legitimately down from Friday 17:40 until Sunday 17:47. Testing "is *today*
   in `days`?" would wrongly stop alerting all Friday morning while the Thursday
   window is still open, and would wrongly start alerting on Friday evening.

### Document

`fix_session_schedule/FIX.4.4_HSDIAMETER44-NYFIX44.console`:

```json
PUT fix_session_schedule/_doc/FIX.4.4:HSDIAMETER44->NYFIX44
{
  "alert_status": "enabled",
  "cloud.account.name": "hedgeserv-app-prd",
  "end_alert_time": "17:40:00-05:00",
  "start_hour": 17,
  "start_minute": 47,
  "start_alert_time": "17:47:00-05:00",
  "end_hour": 17,
  "end_minute": 40,
  "days": ["SUNDAY", "MONDAY", "TUESDAY", "WEDNESDAY", "THURSDAY"],
  "target": "FIX.4.4:HSDIAMETER44->NYFIX44",
  "group": ["53001", "53002"]
}
```

Field conventions carried over from `FIX.4.4:PRODHSSYCAMORE->TRUMID`:

- `start_hour`/`start_minute` track the **expected logon time**, not the window
  open (PRODHSSYCAMORE opens at 05:00 but carries 5:02). `end_hour`/`end_minute`
  track the **window close**.
- `start_alert_time`/`end_alert_time` are the same clock times as time-only
  dates with the `-05:00` offset, matching the existing docs (they render as
  `Jan 1, 1970 @ HH:MM:00.000` in Discover). No watcher reads them today.
- `cloud.account.name` stays the flat dotted key, as in the other docs.
- `days` values are `java.time.DayOfWeek` names (uppercase), so the script can
  compare them without a lookup table.

### Watcher change (`sdpmt_53002_from_admin_{updated,debug}.json`)

**`days` on its own does nothing** — the window script in the `groups` input had
no day awareness, so the doc would have matched on Friday evening and Saturday
too. The script now:

```painless
boolean inWindow;
boolean openedToday;

if (startTotal <= endTotal) {
  inWindow = currentTotal >= startTotal && currentTotal < endTotal;
  openedToday = true;
} else {
  inWindow = currentTotal >= startTotal || currentTotal < endTotal;
  openedToday = currentTotal >= startTotal;
}

if (!inWindow) { return false; }

if (!doc.containsKey('days.keyword') || doc['days.keyword'].size() == 0) {
  return true;
}

String windowDay = openedToday
  ? nyTime.getDayOfWeek().toString()
  : nyTime.minusDays(1).getDayOfWeek().toString();

for (def day: doc['days.keyword']) {
  if (day.equals(windowDay)) { return true; }
}
return false;
```

- **Backward compatible.** A doc with no `days` short-circuits to `true`, so the
  six existing targets match on exactly the same minutes as before (verified
  minute-by-minute across a full week: zero differing minutes).
- **Both `days.keyword` and `days` are tried, in that order.** This is the part
  that is easy to get wrong. `doc.containsKey()` answers from the **index
  mapping**, not from the document, so it cannot be used to ask "does *this* doc
  have days?" — once one doc carries the field, `containsKey` is true for every
  doc in the index. Two separate conditions are doing two different jobs:

  | condition | means | outcome |
  | --- | --- | --- |
  | neither name in the mapping | no schedule doc uses `days` yet | no restriction (watcher safe to deploy first) |
  | mapped, but `size() == 0` | **this** doc has no `days` (PRODHSSYCAMORE) | no restriction — every day |
  | mapped, `size() > 0` | this doc has `days` (HSDIAMETER44) | day filter applies |

  Checking only `days.keyword` was a real bug in the first cut of this change:
  under dynamic mapping `days` becomes `text` + `days.keyword`, but under an
  explicit or dynamic-template mapping it becomes a bare `keyword` named `days`,
  `containsKey('days.keyword')` returns false, and the script falls through to
  "no restriction" — **silently** alerting HSDIAMETER44 on Friday evening and
  Saturday. Trying both names removes that failure mode. The one shape left that
  cannot work is `days` mapped as `text` with no sub-field, where `doc['days']`
  throws on disabled fielddata; that fails loudly rather than silently, and
  pinning the mapping (step 2 of the console file) prevents it.
- **`openedToday`** is what makes the day test correct on a wrapping window:
  when we are in the tail of a window that opened yesterday, the day compared is
  yesterday's.

Resulting coverage, simulated over a week: continuous from Sun 17:47 to Fri
17:40, with the 7-minute 17:40-17:47 maintenance gap each day, and nothing
expected between Fri 17:40 and Sun 17:47.

### Verifying it on the cluster

The simulations behind the claims above are models of the logic and of
`LeafDocLookup` semantics, not a live Elasticsearch. Step 4 of
`fix_session_schedule/FIX.4.4_HSDIAMETER44-NYFIX44.console` is the real check:
it runs the watcher's window script as a plain `_search` with a
`params.test_millis` override, so any point in the week can be tested without
waiting for Friday. The discriminating case is **Fri 18:00 ET**, where
`FIX.4.4:HSDIAMETER44->NYFIX44` must be absent while
`FIX.4.4:PRODHSSYCAMORE->TRUMID` is still present. If both come back, the day
filter is not taking effect and the `days` mapping is the thing to check.

### Open item: the 53002 cron does not fire on Sunday evenings

Not changed here, because it affects every session in group 53002.

The trigger is:

```json
"cron": ["0 */2 * ? * TUE-FRI", "0 */5 0-3 ? * SAT", "0 */5 4-23 ? * MON"]
```

Watcher cron is evaluated in **UTC**. Sunday UTC has no entry, and the MON entry
starts at 04:00 UTC, so the watcher does not run between **Sun 17:47 ET and Sun
23:00 ET (EST) / Mon 00:00 ET (EDT)** — a 313-373 minute blind spot landing
exactly on this session's weekly logon. A Sunday-evening logon failure would not
open a ticket until several hours later.

Closing it requires:

```json
"cron": ["0 */2 * ? * TUE-FRI", "0 */5 0-3 ? * SAT", "0 */5 * ? * MON", "0 */5 21-23 ? * SUN"]
```

(verified: zero uncovered in-window minutes in both EDT and EST). The reason it
is not applied: the added Sunday-evening runs would also evaluate the other
group-53002 docs, and `FIX.4.2:DMCP_EMSX_PROD->BLP_EMSX_PROD`
(00:02-23:55, no `days`) would begin alerting in a window it is not checked in
today. Give those docs a `days` list first, or accept the new tickets.

### 53001 is unaffected

`sdpmt_53001_updated.json` has no window script at all — it selects every
enabled doc in the group and checks for an `OnLogon` success within
`now-27d`. A 27-day lookback comfortably covers a session that logs on five days
a week, so leaving `FIX.4.4:HSDIAMETER44->NYFIX44` in `group: ["53001","53002"]`
(matching what was done for PRODHSSYCAMORE) does not introduce false positives
there, and 53001 needs no day handling.

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
