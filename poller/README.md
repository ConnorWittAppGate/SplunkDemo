# AppGate ZTNA — Splunk Collective Health Dashboard

A Splunk Dashboard Studio port of the AppGate ZTNA telemetry dashboard, built for an AppGate ZTNA demo
environment. Modeled on the CrowdStrike NG-SIEM version in `appgate-crowdstrike-integration-main`,
re-authored in SPL — the CQL dashboards there are not portable.

**Scope:** Collective Health. Telemetry only, no response actions. **Fed entirely by the AppGate
LogForwarder** — no Controller API poller, no additional services to run.

---

## What's here

```
appgate-splunk-demo/
  TA-appgate-demo.tar.gz                      <- UPLOAD THIS to Splunk (Step 1)
  dashboards/appgate_collective_health.json   <- PASTE THIS into Studio (Step 2). All three tabs.
  dashboards/archive/                         Superseded standalone investigation dashboards
  TA-appgate-demo/default/props.conf          KV_MODE=json only (optional)
  TA-appgate-demo/default/macros.conf         `appgate_lf` -- the ONE object panels depend on
  TA-appgate-demo/default/app.conf            App metadata
  TA-appgate-demo/metadata/default.meta       export=system (required -- see below)
  poller/config.yaml                          OPTIONAL upgrade path, not used
  README.md                                   This runbook
```

Only two files leave this directory: the `.tar.gz` gets uploaded, and
`appgate_collective_health.json` gets pasted. Everything else is source for rebuilding the package.

> **One dashboard, not two.** `appgate_collective_health.json` now carries all three tabs — Health
> Overview, Investigate User, Compare Users — in a single Studio definition. The two standalone
> `appgate_user_investigation*.json` files it replaced have moved to `dashboards/archive/`; they are
> kept only as a source of panels to pull back in, and pasting them creates redundant dashboards.

> `metadata/default.meta` sets `export = system`. Without it the macros stay private to
> `TA-appgate-demo`, and a dashboard living in the Search app fails every panel with *"The search
> specifies a macro that cannot be found"*. If you install by hand (Option C), the equivalent is
> **Sharing: Global** on the macro.

> **All field mapping lives in the `appgate_lf` macro, not in `props.conf`.** Field aliases key on
> sourcetype, so a stanza-name mismatch made every panel return zero rows *with no error* — the same
> symptom as having no data at all. The macro keys on `index=appgate` only, which removes that failure
> mode. `props.conf` now does nothing but enable JSON extraction, and is skippable if you already have
> it.

---

## How health is measured

There are no polled gauges here — every panel is derived from the `cz-logd` audit stream already
landing in `index=appgate`. That changes what "appliance health" means, and it's worth being able to
say this out loud during the demo:

**An appliance that stops emitting events is the failure signal.** The *Appliance Liveness* table
tracks the gap since each appliance's last event of any kind (amber past 15 min, red past 60), which
catches a dead or partitioned appliance faster than a status poll would. *Event Volume by Appliance*
shows the same thing as a shape — a line dropping to the floor is an appliance that went quiet.
*Last Known State* carries the most recent `appliance_status_changed` value where one exists.

Two things this genuinely cannot show, and no query will fix:

- **Resource utilization** — CPU, disk, RAM, and NIC throughput exist only in the Controller API's
  `/admin/stats/appliances` response. They are not in any audit event.
- **Entitlement/policy assignment** — who *qualifies* for what. `log.rule_name` on `ip_access` gives
  the entitlement that carried traffic (so "top entitlements by use" is solid), but policy names appear
  nowhere in the LogForwarder stream. Policies surface only as configuration-change audit: who edited
  which Policy/Entitlement/Condition and when.

See **What the poller would add** at the bottom if you later want those.

---

## Step 1 — Install the add-on

Pre-packaged as **`TA-appgate-demo.tar.gz`** so it installs on a remote Splunk (EC2, Cloud) with
nothing but browser access.

### Option A — Upload through Splunk Web *(recommended)*

**Apps** (gear icon, top left) → **Manage Apps** → **Install app from file** → choose
`TA-appgate-demo.tar.gz` → **Upload** → restart when prompted.

Rebuild after editing any `.conf`, then re-upload with **Upgrade app** ticked:

```powershell
Set-Location C:\Users\connor.witt\appgate-splunk-demo
Remove-Item TA-appgate-demo.tar.gz -Force -ErrorAction SilentlyContinue
tar -czf TA-appgate-demo.tar.gz TA-appgate-demo
```

### Option B — Shell access to the instance

```powershell
scp -i <your-key>.pem C:\Users\connor.witt\appgate-splunk-demo\TA-appgate-demo.tar.gz ec2-user@<host>:/tmp/
```
```bash
sudo tar -xzf /tmp/TA-appgate-demo.tar.gz -C /opt/splunk/etc/apps/
sudo chown -R splunk:splunk /opt/splunk/etc/apps/TA-appgate-demo
sudo /opt/splunk/bin/splunk restart
```

No inbound SSH? **AWS Systems Manager → Session Manager** gives a shell with no open port.

### Option C — Create the one macro by hand in Splunk Web

No app install needed. The dashboard depends on **exactly one object**.

Settings → Advanced Search → Search macros → New. Name `appgate_lf`, *Sharing: Global*, definition:

```
index=appgate | eval appliance=coalesce('log.hostname', hostname, 'log.log_source', log_source) | rename log.* AS * | eval ag_user=distinguished_name_user, ag_user_ou=distinguished_name_ou, entitlement=rule_name, ag_action=action, ag_status=status, ag_reason=coalesce(reason, drop_reason), dest_ip=destination_ip, dest_port=destination_port, ag_protocol=protocol, ag_bytes=packet_size, ag_client_ip=client_ip, ag_command=command, ag_lat='geoip.location.lat', ag_lon='geoip.location.lon', ag_country='geoip.country_name', ag_city='geoip.city_name'
```

There are no field aliases to create — all mapping is in the macro. The only other requirement is that
the raw JSON be extracted into fields (see step 1 of the ladder below).

---

## Diagnostics — run these in order

### 1. Is the JSON extracted into fields?

```
index=appgate | head 1 | table sourcetype, hostname, daemon, log.event_type, log.rule_name
```

- **All populated** → extraction works. Skip to step 2.
- **All null, `_raw` shows JSON** → extraction is off. Settings → Source types → *(the `sourcetype`
  value from this search)* → Advanced → add `KV_MODE = json`. Nothing else can work until this does —
  no macro can recover fields that were never extracted.

Note the real event shape, which is **not** uniformly nested:

```json
{"version":2, "date":"Aug  4 19:07:28", "timestamp":"2026-08-04T19:07:28.345Z",
 "hostname":"10.10.100.50", "daemon":"cz-vpnd",
 "log":{"action":"allow", "event_type":"ip_access", "rule_name":"...", ...}}
```

`hostname`, `daemon`, and `timestamp` are **top-level**; everything else is under `log`. The macro
coalesces the appliance name across both levels before flattening.

### 2. Does the macro resolve and map fields?

```
`appgate_lf` | table _time, appliance, daemon, event_type, ag_user, entitlement, ag_action
```

Every column should be populated for `ip_access` rows. If you get *"The search specifies a macro that
cannot be found"*, the add-on isn't installed or isn't shared globally (`export = system`).

### 3. Which panels will actually have data?

```
`appgate_lf` | stats count by daemon, event_type | sort - count
```

**This is the important one.** Each panel traces to specific event types, and a missing one is blank
with no error — indistinguishable from a broken panel:

| Missing from your data | Panels that stay blank |
|---|---|
| `cz-vpnd` / `ip_access` | Access Denials, all three entitlement/resource panels, Traffic |
| `cz-controllerd` / `authentication_*` | Auth Failure Rate, Auth Outcomes, Auth Failure Reasons |
| `cz-configd` / `appliance_status_changed` | Status Transitions, Last Known State column |
| `appliance_started` / `_shutdown` / `_rebooted` | Recent Appliance State Changes |
| `cz-sessiond` / `session_*`, `tunnel_*` | Sessions Established, Session & Tunnel Churn |
| `entity_created` / `_updated` / `_deleted` | Policy/Entitlement/Condition Changes |
| `rule_monitor_health_change` | Access Rule Health |
| `remedy*` | Posture Remediation |
| `audit_drop` | Dropped Audit Events (blank here is *good*) |

### 4. Is geo enrichment present?

```
`appgate_lf` | search event_type=ip_access | stats count(ag_lat) as with_geo, count as total
```

**Expect `with_geo` > 0 here.** Clients connect from routable addresses (AWS public IPs), so the
LogForwarder attaches a full `geoip` block — city, region, country, lat/lon — and the **Client Activity
Map works.** An earlier note in this file predicted the opposite based on one internal-only sample event;
that was wrong.

Geo also lands on `ssh_access_failed`, which is what makes the **Appliance SSH Attack Surface** panel
worth showing: unsolicited SSH brute-force against appliance IPs, attributed to origin country.

### 5. Are timestamps landing correctly?

```
index=appgate | eval real=strftime(strptime('log.timestamp', "%Y-%m-%dT%H:%M:%S.%3NZ"), "%F %T") | table _time, real, date, timestamp
```

`_time` and `real` must agree. These events carry **two** timestamps — a syslog-style `date`
("Aug  4 19:07:28", no year, no timezone) that appears *first* in the raw string, and an unambiguous
ISO-8601 UTC `timestamp`. Splunk's auto-detection tends to grab `date`, and with no timezone the events
can land hours off, which shifts every panel. If they disagree, uncomment the `TIME_PREFIX` block in
`props.conf`.

---

## Step 2 — Import the dashboard

Splunk → **Dashboards** → **Create New Dashboard** → **Dashboard Studio** → **Absolute** layout →
create, then open the source-code (`</>`) editor and paste
`dashboards/appgate_collective_health.json`, replacing everything.

The three tabs — **Health Overview**, **Investigate User**, **Compare Users** — appear on save; they
come from `layout.tabs` in the pasted JSON, so there is nothing to wire up by hand.

If Studio rejects a `columnFormat` / `fieldColors` key on your version, **delete that one key** — the
conditional coloring is cosmetic and every panel renders without it. Same rule for `eventHandlers`:
dropping those blocks costs you click-to-filter and nothing else.

---

## Panel inventory (31 data panels across 3 tabs)

42 visualizations total — 31 data panels plus 11 markdown headers.

### Tab 1 — Health Overview (22 panels)

**At a Glance** — Active Users · Appliances Reporting · Sessions Established (sparkline) · Access
Denials · Auth Failures · Appliance SSH Failures.

**Appliance Health** — Appliance Liveness · Users per Appliance · Event Volume by Appliance ·
Controller Messages.

**Entitlements** — Entitlements by Actual Use · **Assigned vs Actually Used**.

**Resource Access** — Traffic Allowed vs Dropped vs Rejected · Entitlement Access Outcomes (deny rate) ·
Top Resources Accessed.

**Authentication & Attack Surface** — Authentication Outcomes · Authentication Failures ·
**Appliance SSH Attack Surface** · Session & Tunnel Churn · Active Users Over Time · Client Activity Map.

**Admin Audit** — Admin & Configuration Activity (with controller execution time).

> **Top Policies, Entitlements by Assignment, and Conditions Satisfied are not in this build.** An
> earlier version of this inventory listed all three. Conditions was dropped because
> `entitlement_token_evaluated` is unconfirmed in this collective; the assignment side is folded into
> **Assigned vs Actually Used** rather than shown as its own table.

### Tab 2 — Investigate User (2 panels)

Users in Window · Identity Summary. See *Tab 2* below.

### Tab 3 — Compare Users (7 panels)

**Cohort (no input needed)** — User Comparison Matrix · Privilege Gap by User · Assigned vs Used chart.

**Head to head (`user_a` vs `user_b`)** — A vs B summary · Entitlement Differential · Policy
Differential · Resource Reach Differential.

### The two panels to build the demo around

- **Assigned vs Actually Used** — entitlements granted to users who never exercised them. The
  least-privilege gap, straight from the audit stream.
- **Appliance SSH Attack Surface** — geolocated brute-force against appliance IPs. Concrete, external,
  and it makes the case for not exposing appliance SSH better than any slide.

### Filters — wildcard text boxes, plus click-to-filter

Time picker plus five **free-text boxes**: **Appliance**, **User**, **Entitlement**, **User A**,
**User B**. All five default to `*`. Each is applied only by panels that can honor it — collective-wide
panels (tiles, Appliance Liveness, Users in Window, config activity) deliberately ignore them.

Every box accepts wildcards, so the old menu entries survive as things you can type: `*` for all,
`*loadgen*` for a naming convention, `10.10.*` for a subnet, `Acquisitions*` for an entitlement family.
**Setting a box back to `*` is how you clear a filter.**

> **Nothing is hardcoded any more.** The dashboard ships with no usernames, appliance hostnames, or
> entitlement names in it, and there is no list to regenerate when the environment changes. That was
> the previous design's standing defect: a static `items` array does not track identities that appear
> later, and a stale entry looks authoritative while silently filtering to nothing.
>
> **Dynamic dropdowns are still not the answer.** Studio's `input.dropdown` wants `items` as a chain
> expression in a form that differs from the one used by visualization options, and it fails *silently*,
> rendering an empty list with no error. Two attempts, both dead. `input.text` takes only `defaultValue`
> and `token`, is schema-safe, and cannot go stale.

**Discovery is done by the tables, not by a menu.** Appliance Liveness lists every appliance, Users in
Window lists every identity, Assigned vs Actually Used lists every entitlement — all three ignore their
own filter precisely so they stay complete, and all three are click-to-filter. That is now the only
value-discovery path, and it always reflects live data.

If you want the same lists as raw searches:

```
`appgate_lf` | search ag_user=*     | stats count by ag_user     | sort ag_user
`appgate_lf` | search appliance=*   | stats count by appliance   | sort appliance
`appgate_lf` | search event_type=ip_access | stats count by entitlement | sort entitlement
```

### Porting to another collective

One line: the `appgate_index` macro in `macros.conf`, which is all `appgate_lf` uses to select data.
Find yours with `| tstats count where index=* by index, sourcetype`. Everything else discovers itself.

**The Investigate and Compare tabs seed themselves from your data.** They used to be populated on load
by naming real identities in the input defaults, which is exactly the thing that doesn't port. Three
macros replace that:

| Macro | Used by | What it does |
|---|---|---|
| `appgate_top_user(u)` | Identity Summary | Resolves the **User** box to one identity — the exact name if you typed one, the busiest match if you typed a pattern, the busiest identity in the window if it's `*`. |
| `appgate_pair(a, b)` | all four head-to-head panels | Resolves **User A** / **User B** to two identities: pinned names first, then busiest, take two. So `*` / `*` compares the two busiest, one name plus `*` compares that name against the busiest other. |
| `appgate_slot(a, b)` | all four head-to-head panels | Labels each event `slot=A` or `slot=B`, so the differential tables can say `User A only` without a token holding a literal name. |

All three read `*`, an exact name, or a wildcard pattern, and all three rank by event volume in the
selected time range.

> **Why `appgate_slot` re-runs the ranking instead of reading it off the events.** The four head-to-head
> panels each filter to a different event type. If slot A meant "top-ranked identity present in this
> panel," then a pair where only one member has `authorization_succeeded` events would silently
> relabel the other as A, and Entitlement Differential would credit its rows to the wrong column of the
> head-to-head table sitting next to it. A is resolved by the same window-wide `head 1` in every panel.

> **One input shape will break the macro calls:** an identity containing a literal comma, since Splunk
> splits macro arguments on commas. `ag_user` resolves to `distinguished_name_user`, the user component
> only, so this has not come up — and it would be a loud parse error, not a silent empty panel.

### Click-to-filter (drilldowns)

Eleven panels carry a `drilldown.setToken` handler, so the normal way to drive the dashboard is
clicking, not typing:

| Click a row in | Sets |
|---|---|
| Appliance Liveness · Users per Appliance · SSH Attack Surface | `appliance` |
| Entitlements by Actual Use · Assigned vs Actually Used · Entitlement Access Outcomes | `entitlement` |
| Users in Window · Authentication Failures | `user` **and** `user_a` |
| User Comparison Matrix | `user_a` and `user` |
| Privilege Gap by User · Assigned vs Used chart | `user_b` |

The handler lives as a sibling of `options` inside the visualization:

```json
"eventHandlers": [
  { "type": "drilldown.setToken",
    "options": { "tokens": [ { "token": "user", "key": "row.User.value" } ] } }
]
```

`key` must name the panel's **renamed** output column (`row.User.value`), not the raw field
(`ag_user`). Two panels are deliberately exempt from their own filter so they never collapse to a
single row and always work as pickers: **Appliance Liveness** (ignores `appliance`) and **Users in
Window** (ignores `user`). **Assigned vs Actually Used** is collective-wide for the same reason, which
makes it the full-list picker for entitlements.

> A drilldown writes the clicked value straight into the text box, so what you clicked and what the
> panels are filtering on always agree. (With the old static dropdowns they could disagree: a clicked
> value absent from `items` filtered correctly but showed no selection.)

### Auto-refresh

`defaults.dataSources["ds.search"].options` sets `refresh: 5m` / `refreshType: delay`, so every panel
re-runs five minutes after its last completion without anyone touching the page. Delete those two keys
if you want a frozen dashboard for a screenshot.

**Site could be a filter.** `site_names` *does* exist, on `authorization_succeeded` — an earlier note
here said otherwise and was wrong. It's not wired as an input because it only appears on that one event
type, so it would filter authorization panels and nothing else. `dc(site_names)` is shown as a column in
the entitlement-assignment table instead.

### Event types that do NOT exist in this collective

Panels depending on these were **removed**, not left blank. All were inferred from the CrowdStrike
LogScale parser, which implies event names AppGate 6.7 doesn't emit here:

| Assumed | Reality |
|---|---|
| `tunnel_up` / `tunnel_down` | `tunnel_connected` / `tunnel_established` / `tunnel_closed` |
| `rule_monitor_health_change` | Not emitted — Access Rule Health panel dropped |
| `audit_drop` | Not emitted — telemetry-loss tile dropped |
| `appliance_status_changed` | Not emitted — Status Transitions and Last Known State dropped |
| `remedy*` | Not emitted — Posture Remediation dropped |
| daemons `cz-configd`, `sshd` | Real: `cz-appliance` (emits `ssh_access_failed`) |

Daemons actually present: **cz-controllerd** (auth, authorization, admin audit, admin messages),
**cz-vpnd** (`ip_access`, `tunnel_*`, `acl_rules_update`), **cz-sessiond** (`session_*`, `*_token_*`,
`device_claims_*`), **cz-appliance** (`ssh_access_failed`).

**Admin & Configuration Activity may still be thin.** It looks for `entity_created` / `_updated` /
`_deleted`, which only appear when an admin actually changes something. What this collective emits
constantly is `entities_listed` / `on_boarded_devices_listed` (API *reads*, largely from the CrowdStrike
poller service account), so the panel includes those plus `admin_authorization_*`. `Object Type` is a
visible column so you can see which entity kinds appear.

**Re-verify after any AppGate upgrade** with:

```
`appgate_lf` | stats count by daemon, event_type | sort - count
```

### Two panels worth building the demo narrative around

- **Entitlement Access Outcomes** — deny rate per entitlement. A high Deny % is a misconfigured
  entitlement or a condition failing unexpectedly, which is a concrete, diagnosable finding rather than
  a vanity metric.
- **Posture Remediation** — `remedy*` events are zero-trust enforcement actually firing: a device
  failed a posture check and got sent to remediation. The CrowdStrike original collected these and
  never displayed them.

---

## Tab 2 — Investigate User

**Not a separate dashboard.** This is the second tab of `appgate_collective_health.json`, scoped to one
identity by the shared **User** token.

**Start with the `Users in Window` panel** (left half of the tab). It lists every identity in the time
range with event counts and last-seen — **click a row** and Identity Summary loads that user; the same
click also sets User A on the Compare tab. The panel intentionally ignores the User filter so it always
shows the full list.

> **Correction — Dashboard Studio does support tabs.** This file previously claimed there was no tab
> container in the schema and that a "tab" had to be a second dashboard plus a link. That was wrong.
> `layout.tabs.items` maps a label to a `layoutId` in `layout.layoutDefinitions`, and all three views
> now live in one definition:
>
> ```json
> "tabs": { "items": [
>   { "label": "Health Overview",  "layoutId": "layout_1" },
>   { "label": "Investigate User", "layoutId": "layout_DiiilFMC" },
>   { "label": "Compare Users",    "layoutId": "layout_compare" }
> ] }
> ```
>
> Inputs listed in `layout.globalInputs` are shared across every tab, which is what makes the User
> filter follow you from tab to tab and what makes cross-tab drilldown work without a URL.

The panel table below describes the **archived** standalone investigation dashboard
(`dashboards/archive/appgate_user_investigation_full.json`), which is a superset of what tab 2 renders.
Tab 2 itself ships three panels — Users in Window, Identity Summary, and the header — and the archive is
where you go to pull the rest back in.

| Section | Panels |
|---|---|
| **Summary** | Auth Successes · Auth Failures (amber >1, red >5) · Entitlements Used · Resources Touched · Access Denials · Client IPs |
| **Pick a User · Identity & Device** | **Users in Window** · Identity Summary · IdP Claims · Device Posture |
| **Authentication** | Outcomes Over Time · Auth Event Breakdown · Failure Reasons · Recent Auth/Authorization/Token Events |
| **Access** | Entitlements Exercised · Resources Reached · Access Outcomes Over Time · Denied Access Attempts |
| **Footprint** | Session & Tunnel Timeline · Client IPs Used · Appliances Traversed · Full Activity Timeline |

### The claims panels discover their own schema

The CrowdStrike original hardcoded `user_claims.Branch`, `.Citizenship`, `.clearance`, `.riskScore`,
`.IA_Training_Status` — and its own comments admit those are *example* claims from a federal demo
schema. Hardcoding them here would mean a table of empty rows on any other tenant.

Instead, **IdP Claims** and **Device Posture** use `transpose` to render whatever attributes are
actually present:

```
`appgate_lf` | search (event_type=authentication_succeeded OR event_type=authorization_succeeded) ag_user="$user$"
| sort - _time | head 1
| fields user_claims.*, scripted_user_claims.*
| transpose 1 | rename column as Claim, "row 1" as Value
| rex mode=sed field=Claim "s/^(scripted_)?user_claims\.//"
| search Value=* | sort Claim
```

No claim schema assumed, so this works on any tenant and doubles as a discovery tool — it shows you what
your IdP emits. Two consequences worth knowing:

- **Blank for service accounts.** Your load-gen identities (`OU=service`) almost certainly carry no
  claims block. Pick a real interactive user to demo this section.
- **It reads one event.** `head 1` takes the most recent auth *or* authorization event. Per the poller's
  own notes, UCS-resolved attributes (`scripted_user_claims.*`) ride on `authorization_succeeded` while
  SAML claims ride on `authentication_succeeded` — so if one set looks missing, the other event type was
  simply more recent. Both are queried; only the latest is displayed.

**Identity Summary** is the panel that always populates — it's built from event metadata (first/last
seen, appliances, client IPs, entitlement count, OU, device ID, subsystems) rather than claims.

### Cross-tab drill-through is built in — no manual URL wiring

An earlier version of this section explained how to link two saved dashboards with
`drilldown.customUrl` and a hand-copied URL slug. That is no longer necessary: Investigate User is a
**tab in the same dashboard**, so a `drilldown.setToken` handler moves you between views with no URL,
no slug, and nothing to re-edit after a rename. Clicking a user in **Authentication Failures** on tab 1
sets `user`, and the Investigate User tab is already scoped to them when you switch to it.

`drilldown.customUrl` is still the right tool for leaving this dashboard entirely — linking out to a
Splunk search, a ticketing system, or the AppGate admin UI.

---

## Verification checklist

1. Work the **Diagnostics** ladder above (steps 1–5). Those five searches isolate every failure mode
   that produces a blank-but-error-free dashboard.
2. Every panel renders with data — no silently empty panels, no `field not found`.
3. **Confirm `appgate_index` selects your data** — `` `appgate_lf` | stats count by daemon `` should
   return the four daemons. There is nothing else to configure and no value list to regenerate.
4. **Click-test one drilldown per token.** Click a row in Appliance Liveness, in Assigned vs Actually
   Used, and in User Comparison Matrix, and confirm the matching box and panels react. If Studio
   rejects the `eventHandlers` key on your version, delete those blocks — every panel still renders,
   you just lose click-to-filter and have to type values instead.
5. Type a specific value into each box: filtering panels narrow, collective-wide panels don't
   (Appliance Liveness, Users in Window, Assigned vs Actually Used, and the tiles ignore filters
   by design). Then set it back to `*`.
6. **Tabs 2 and 3 must be populated on arrival with every box left at `*`** — Identity Summary should
   profile your busiest identity, and head-to-head should compare your two busiest. If they are blank,
   `appgate_top_user` / `appgate_pair` aren't resolving; check `` `appgate_lf` | search ag_user=* |
   stats count by ag_user `` returns rows. Then pin one name in **User A**, leave **User B** at `*`,
   and confirm A holds while B refills with the busiest other identity.
7. Demo dry run at 24h, then narrow to 1h. Nothing should go blank.
8. Sanity-check *Appliance Liveness* against reality — if an appliance you know is up shows 60+ minutes
   silent, the timestamp extraction is wrong (events indexing at ingest time rather than event time),
   not the appliance.

---

## Deliberate departures from the CrowdStrike original

Bugs in the reference build, not reproduced:

- Its **"Security & Compliance" section is empty** — declared with no `widgetIds`. Four widgets (Admin
  Configuration Activity, Access Denied, Polled Endpoints, Ingest Rate) plus the header note are
  orphaned and never render.
- Two sections both claim `order: 3`, leaving display order undefined.
- `Distinct Resources Today` used `groupBy(...) | count()`, which counts **groups**, not distinct
  resources.

Intentional additions: the liveness/staleness health model, Entitlement Access Outcomes, Posture
Remediation, Auth Failure Rate, and the Dropped Audit Events telemetry-loss tile.

---

## What the poller would add

`poller/config.yaml` is retained but unused. Standing up `02-source/appgate-api-poller/` from the
CrowdStrike repo (standalone Python; already emits Splunk HEC, no code changes) would unlock:

| Panel | Source |
|---|---|
| Appliance CPU / disk / RAM / NIC gauges | `/admin/stats/appliances` |
| Site health | `/admin/sites/status` |
| Live *concurrent* session count | `/admin/stats/active-sessions/dashboard` |
| Registered devices, licensed users | `/admin/on-boarded-devices` |
| Client version / OS distribution | `/admin/stats/active-sessions/dn` |

> **Correction.** This section previously claimed the poller was the *only* way to get policy names and
> the assigned-vs-used least-privilege gap. That was wrong. `authorization_succeeded` carries
> `policy_names`, `entitlement_names`, and `site_names` per user, and `entitlement_token_evaluated`
> carries `successful_condition_names` — so Top Policies, entitlement assignment, the drift panel, and
> the conditions panel are all built from the audit stream with no poller.
>
> What the poller would add over that is the **complete catalog** rather than only what appeared in an
> authorization during the window: entitlements nobody was granted, disabled entitlements, conditions
> nobody satisfied, and per-entitlement host/port/CIDR detail. Useful for a config audit, unnecessary
> for a telemetry demo.

**If you do enable it,** the `appgate_api`, `appgate_latest_snapshot`, and `appgate_latest_catalog`
macros and the `[appgate:api]` props stanza are already in the add-on. One trap: `entitlement_grant`
and `entitlement_catalog` are **full re-ships on a heartbeat, not deltas**, so any query must reduce to
the newest sweep (that's what the snapshot macros do) or counts multiply by the number of sweeps in the
window — roughly 1440× over 24h at a 60s cadence.

## Not in scope

- Response actions (quarantine / blacklist / revoke).
- The per-user **Identity Profile** drill-down dashboard.
- **ZTA score** panels — require CrowdStrike sensor data in Splunk. A composite posture score from
  `device_claims` (disk encryption + firewall + not-local-admin + Azure-joined + Intune-enrolled) is
  the natural AppGate-native substitute.
- **DPI / Layer-7** panels — need a passive Suricata + nDPI sensor that isn't part of this environment.
