# Collective Health — Panel Data Reference

Where every panel on `dashboards/appgate_collective_health.json` gets its data: which daemon emits
it, which event type carries it, which raw JSON field it reads, and what a blank panel means.

The operational runbook is `README.md`. This file is the reference you consult when a panel is
empty, wrong, or you're asked on stage "where does that number come from?"

**Contents**

- [How to read this file](#how-to-read-this-file)
- [The `appgate_lf` pipeline](#the-appgate_lf-pipeline)
- [Field mapping table](#field-mapping-table)
- [Filter tokens — which panels honor which](#filter-tokens--which-panels-honor-which)
- [Panel reference](#panel-reference) — all 22 data panels
- [Removed panels](#removed-panels) — what was cut, and when to restore it
- [Event-type → panel index](#event-type--panel-index)
- [Known issues](#known-issues)

---

## How to read this file

Every panel entry has the same five fields:

| Field | Meaning |
|---|---|
| **Source** | Daemon → event type(s). This is the upstream dependency — if AppGate stops emitting it, the panel dies. |
| **Raw fields** | The JSON keys read out of the event, at their real nesting depth. |
| **Via macro** | How `appgate_lf` renames them into what the query uses. A dash means the field passes through unchanged. |
| **Filters** | Which dashboard tokens narrow this panel. All panels inherit the time picker. |
| **Blank means** | How to tell "no data" from "broken." |

---

## The `appgate_lf` pipeline

One macro (`TA-appgate-demo/default/macros.conf`) does index selection *and* all field mapping. No
field aliases, no sourcetype dependency. Four stages, and **the order is load-bearing**:

```
index=appgate
  |
  |-- 1. eval appliance, device_hostname    <- runs BEFORE the rename, on purpose
  |
  |-- 2. rename log.* AS *                  <- flattens the nested block
  |
  |-- 3. rename "*{}" AS "*"                <- strips Splunk's array suffix
  |
  \-- 4. eval <40+ friendly aliases>        <- names the flattened fields
```

### Stage 1 — why it runs first

The raw event carries **two different hostnames at two different depths**, and they mean opposite
things:

```json
{"version":2, "date":"Aug  6 15:00:31", "timestamp":"2026-08-06T15:00:31.568Z",
 "hostname":"controller.sdpfederal.com",   <- the APPLIANCE (gateway IP / controller FQDN)
 "daemon":"cz-controllerd",
 "log":{"hostname":"ip-10-10-10-46",       <- the CLIENT DEVICE
        "action":"allow", "event_type":"ip_access", "rule_name":"...", ...}}
```

After `rename log.* AS *` they would collide and one would win silently. So stage 1 reads both while
they are still distinguishable:

- `appliance` = **top-level** `hostname` only
- `device_hostname` = `log.hostname`, falling back to `log.device_claims.os.hostname`

> An earlier macro version coalesced `log.hostname` first, which populated the appliance column of
> **Appliance Liveness** with client device names — plausible-looking output, entirely wrong. Do not
> reintroduce the coalesce. See [Known issues](#known-issues), which affects the README's Option C.

The fallback matters: on `authorization_succeeded` there is **no `log.hostname` at all** — the client
device name lives at `log.device_claims.os.hostname`. Without the coalesce, the Device column would
be empty on exactly the events that carry the richest identity data.

### Stage 2 — what survives

`rename log.* AS *` flattens `log.*` to top level. Fields **outside** `log` are untouched and remain
directly queryable: `daemon`, `timestamp`, `date`, `version`, and the `hostname` already captured as
`appliance`.

### Stage 3 — the array suffix

See [Array fields](#array-fields--the--suffix-then-mvexpand) below. This stage exists because Splunk
names JSON array fields with a trailing `{}`, and without stripping it every array-backed panel
renders blank with no error.

### Stage 4 — the aliases

Everything in the [field mapping table](#field-mapping-table). Fields not listed there pass through
under their post-rename name (`event_type`, `policy_names`, `entity_type`, and so on).

### The macro contains pipes

`appgate_lf` ends mid-pipeline, so panels **must** filter with an explicit `| search`:

```
`appgate_lf` | search event_type=ip_access      <- correct
`appgate_lf event_type=ip_access`               <- syntax error
```

---

## Field mapping table

### Computed before the rename

| Query field | Raw JSON | Note |
|---|---|---|
| `appliance` | `hostname` (top level) | The AppGate appliance. Never `log.hostname`. |
| `device_hostname` | `log.hostname` → `log.device_claims.os.hostname` | The end-user's device. |

### Identity and access

| Query field | Raw JSON (under `log`) |
|---|---|
| `ag_user` | `distinguished_name_user` → `username` |
| `ag_ou` | `distinguished_name_ou` |
| `entitlement` | `rule_name` |
| `ag_action` | `action` |
| `ag_status` | `status` |
| `ag_reason` | `reason` → `drop_reason` |
| `auth_type` | `authentication_type` |
| `session` | `session_id` |
| `exec_ms` | `execution_ms` |

### Network

| Query field | Raw JSON (under `log`) |
|---|---|
| `dest_ip` | `destination_ip` |
| `dest_port` | `destination_port` |
| `ag_protocol` | `protocol` |
| `ag_bytes` | `packet_size` |
| `ag_client_ip` | `client_ip` |
| `ag_command` | `command` |

### Geo — attached by the LogForwarder, not by AppGate

| Query field | Raw JSON (under `log`) |
|---|---|
| `ag_lat` / `ag_lon` | `geoip.location.lat` / `.lon` |
| `ag_country` | `geoip.country_name` |
| `ag_city` | `geoip.city_name` |
| `ag_region` | `geoip.region_name` |

> The `geoip` block is rich — city, region, postal code, timezone, continent. Only these five are
> aliased; the rest are reachable under their flattened names (`geoip.postal_code`, and so on).
>
> AppGate also emits a *second*, thinner geo block at `log.system_claims.geo_ip` (country code,
> state code, lat/lon). Ignore it — `geoip` is a superset and is what every panel reads.

### Device posture — from the client's `device_claims` block

| Query field | Raw JSON (under `log.device_claims`) |
|---|---|
| `device_os` | `os.name` |
| `device_platform` | `os.platform` |
| `device_family` | `os.family` |
| `device_client_type` | `clientType` |
| `device_profile` | `profileName` |
| `device_firewall` | `isFirewallEnabled` |
| `device_user_admin` | `isUserAdmin` |
| `device_stig` | `stig` |
| `client_version` | `clientVersion` → `log.client_version` |

### IdP claims

| Query field | Raw JSON (under `log`) |
|---|---|
| `first_name` / `last_name` | `user_claims.firstName` / `.lastName` |
| `idp` | `user_claims.ag.identityProviderName` |
| `risk_score` | `scripted_user_claims.riskScore` |
| `clearance` | `scripted_user_claims.clearance` |
| `device_trust` | `scripted_user_claims.deviceTrust` |
| `geo_location` | `scripted_user_claims.geoLocation` |
| `home_base` | `scripted_user_claims.homeBase` |
| `user_role` | `scripted_user_claims.role` |
| `mfa_provided` | `scripted_user_claims.mfaProvided` |
| `ia_training` | `scripted_user_claims.iaTrainingCompleted` |

> **`user_claims` vs `scripted_user_claims` ride on different events.** SAML claims arrive on
> `authentication_succeeded`; UCS-resolved attributes arrive on `authorization_succeeded`. A query
> reading only one event type sees only one set. Collective Health doesn't use these columns — the
> User Investigation dashboard does, and discovers them with `transpose` rather than hardcoding.
>
> In this collective the `scripted_user_claims` block also carries **`synthetic: true`**. See
> [Known issue 6](#6-demo-caution--the-person-named-identities-are-synthetic-load-gen).

### Controller admin messages

| Query field | Raw JSON (under `log.admin_message`) |
|---|---|
| `msg_level` | `level` |
| `msg_category` | `category` |
| `msg_text` | `message` |
| `msg_source` | `source` |

### Array fields — the `{}` suffix, then `mvexpand`

**Splunk's JSON extraction names an array field with a trailing `{}`.** The raw key
`log.policy_names` extracts as the *field* `log.policy_names{}`, which after `rename log.* AS *`
becomes `policy_names{}` — so a panel searching `policy_names=*` matches nothing and renders **blank
with no error**. Stage 3 of the macro (`rename "*{}" AS "*"`) strips the suffix so queries use the
clean name. It is a no-op when the suffix isn't present, so don't remove it as redundant.

Arrays of *objects* are not covered and don't need to be: `log.dns_settings` extracts as
`log.dns_settings{}.domain`, which does not **end** in `{}` and is left alone. No panel reads one.

Once named cleanly they are multivalue. To count per element you must `mvexpand` first, or `stats`
treats the whole array as one opaque value.

| Query field | Event type |
|---|---|
| `policy_names` | `authorization_succeeded` |
| `entitlement_names` | `authorization_succeeded` |
| `site_names` | `authorization_succeeded` |
| `successful_condition_names` | `entitlement_token_evaluated` |
| `successful_entitlement_names` | `entitlement_token_evaluated` |
| `admin_role_names` | `admin_authorization_succeeded` |

**These are the fields that make policy and entitlement-assignment analysis possible with no API
poller.** Only `entitlement_names` is still used on the dashboard, by **Assigned vs Actually Used**;
the rest belonged to the [removed panels](#removed-panels).

---

## Filter tokens — which panels honor which

Six inputs: a time picker plus five **text** boxes defaulting to `*` and accepting wildcards
(`gw-*`, `*loadgen*`). Text rather than dropdowns because Studio's `input.dropdown` chain-expression
schema fails silently — the README explains the history, and
[Known issue 7](#7-static-dropdowns-are-already-stale) records the static-list alternative being tried
and abandoned.

`$user_a$` / `$user_b$` are the Compare tab's own pair, separate from `$user$`. They are resolved to
concrete identities at search time by `appgate_pair` / `appgate_slot`, so `*` means "pick for me" here
rather than "all" — see [Self-seeding identity selection](#7-static-dropdowns-are-already-stale) and
the macro comments.

**Every panel inherits `$global_time$`** via the dashboard's `defaults.dataSources` block. That is
the only universal filter.

| Token | Panels that honor it |
|---|---|
| `$appliance$` | Appliance Liveness · Users per Appliance · Event Volume by Appliance |
| `$user$` | Authentication Failures · Client Activity Map · Top Resources Accessed · Admin & Configuration Activity |
| `$entitlement$` | Entitlements by Actual Use · Traffic · Entitlement Access Outcomes · Top Resources Accessed |

**Twelve panels deliberately ignore all three.** The six At-a-Glance tiles, Controller Messages,
Assigned vs Actually Used, Authentication Outcomes, SSH Attack Surface, Session & Tunnel Churn, and
Active Users Over Time are collective-wide by design — they answer "what is the collective doing,"
and scoping them to one appliance or user would make them lie. If you type a filter and a panel
doesn't move, check this table before assuming it's broken.

---

## Panel reference

### At a Glance

Six `splunk.singlevalue` tiles. None honor any filter token.

---

#### 1. Active Users · `ds_tile_users`

```
`appgate_lf` | search ag_user=* | stats dc(ag_user) as v
```

- **Source:** any daemon, any event carrying an identity — mostly `cz-controllerd` (auth) and `cz-vpnd` (`ip_access`).
- **Raw fields:** `log.distinguished_name_user` → `log.username`
- **Via macro:** `ag_user`
- **Filters:** none
- **Blank means:** no identity-bearing events at all. If this is 0 the whole dashboard is empty; work the README's diagnostic ladder.

> Counts **distinct users seen in events**, not concurrent sessions. Service accounts and load-gen
> identities inflate it. A true concurrent-session count needs the poller.

---

#### 2. Appliances Reporting · `ds_tile_appliances`

```
`appgate_lf` | search appliance=* | stats dc(appliance) as v
```

- **Source:** every event — `hostname` is top-level and universal.
- **Raw fields:** `hostname` (top level)
- **Via macro:** `appliance`, computed pre-rename
- **Filters:** none
- **Blank means:** nothing is reaching `index=appgate`.

> Appliances that **emitted at least one event** in the window — not appliances that exist. A dead
> appliance silently drops out. That's the intent; **Appliance Liveness** makes it visible.

---

#### 3. Sessions Established · `ds_tile_sessions`

```
`appgate_lf` | search event_type IN (session_created, session_reconnected) | timechart span=1h count as v
```

- **Source:** `cz-sessiond` → `session_created`, `session_reconnected`
- **Raw fields:** `log.event_type`
- **Via macro:** pass-through
- **Filters:** none
- **Blank means:** `cz-sessiond` isn't forwarding, or nobody connected in the window.

> Rendered as a sparkline, so it reads as trend plus total. Counts **establishment events**, not live
> sessions — a user who reconnects five times counts five times.

---

#### 4. Access Denials · `ds_tile_denials`

```
`appgate_lf` | search event_type=ip_access (ag_action=drop OR ag_action=reject) | stats count as v
```

- **Source:** `cz-vpnd` → `ip_access`
- **Raw fields:** `log.event_type`, `log.action`
- **Via macro:** `ag_action` from `action`
- **Filters:** none
- **Blank means:** no `ip_access` traffic, or nothing was denied.

> `drop` and `reject` are counted together. They differ operationally — drop is silent, reject sends
> an RST — and **Traffic** breaks them apart. Zero here with nonzero traffic is normal and healthy.

---

#### 5. Auth Failures · `ds_tile_authfail`

```
`appgate_lf` | search event_type=authentication_failed | stats count as v
```

- **Source:** `cz-controllerd` → `authentication_failed`
- **Raw fields:** `log.event_type`
- **Via macro:** pass-through
- **Filters:** none
- **Blank means:** no failed logins, or `cz-controllerd` isn't forwarding. Cross-check with tile 6 — both come from auth-adjacent paths.

---

#### 6. Appliance SSH Failures · `ds_tile_ssh`

```
`appgate_lf` | search event_type=ssh_access_failed | stats count as v
```

- **Source:** `cz-appliance` → `ssh_access_failed`
- **Raw fields:** `log.event_type`
- **Via macro:** pass-through
- **Filters:** none
- **Blank means:** no SSH attempts against appliance management interfaces.

> **This is not AppGate access control.** It's failed SSH against the appliance OS itself — largely
> unsolicited internet brute-force. Nonzero is expected on a public appliance and is the hook for
> **Appliance SSH Attack Surface**. Emitted by `cz-appliance`, *not* `sshd` — the CrowdStrike parser
> assumed `sshd` and found nothing.

---

### Appliance Health

No polled gauges exist in this feed. Health is inferred from **whether an appliance is still
talking**. CPU, disk, RAM, and NIC throughput live only in the Controller API's
`/admin/stats/appliances` and are absent from every audit event.

---

#### 7. Appliance Liveness · `ds_appliance_liveness`

```
`appgate_lf` | search appliance="$appliance$"
| stats max(_time) as last_event, count as events, dc(daemon) as daemon_count,
        values(daemon) as subsystems, dc(ag_user) as users by appliance
| eval silent_min=round((now()-last_event)/60, 1)
| eval last_event=strftime(last_event, "%F %T")
| sort - silent_min
```

- **Source:** every daemon, every event type. Deliberately unfiltered by event type — *any* event proves liveness.
- **Raw fields:** `hostname` (top level), `daemon` (top level), `_time`, `log.distinguished_name_user`
- **Via macro:** `appliance`, `ag_user`; `daemon` passes through
- **Filters:** `$appliance$`
- **Blank means:** nothing in `index=appgate` in the window.

> **The core health panel.** Sorted by silence descending, so the most worrying row is always on top.
> Amber past 15 min, red past 60.
>
> **Silence is the failure signal.** This catches a partitioned or wedged appliance faster than a
> status poll, because it needs no cooperation from the appliance.
>
> **False alarm to rule out first:** if an appliance you know is healthy shows 60+ minutes silent,
> suspect timestamp extraction, not the appliance. These events carry two timestamps — a
> timezone-less syslog `date` ("Aug  6 15:00:31") that appears *first* in the raw string, and an
> unambiguous ISO-8601 `timestamp`. Splunk tends to grab `date` and can land events hours off, which
> inflates `silent_min` for everything at once. **Every appliance suddenly stale = a clock problem.
> One appliance stale = a real appliance problem.** Diagnostic step 5 in the README settles it.
>
> `Subsystems` lists which daemons that appliance runs — a controller shows `cz-controllerd`, a
> gateway shows `cz-vpnd`. A gateway that drops from 2 daemons to 1 is partially degraded, which the
> `Daemons` count surfaces before total silence does.

---

#### 8. Users per Appliance · `ds_users_per_appliance`

```
`appgate_lf` | search ag_user=* appliance="$appliance$"
| stats dc(ag_user) as users by appliance | sort - users
```

- **Source:** any identity-bearing event, per appliance.
- **Raw fields:** `hostname` (top level), `log.distinguished_name_user` → `log.username`
- **Via macro:** `appliance`, `ag_user`
- **Filters:** `$appliance$`
- **Blank means:** events are landing but carry no identity — likely only `ssh_access_failed` or appliance-internal events.

> Load distribution across gateways. Sums to **more** than the Active Users tile — a user traversing
> two gateways counts on both.

---

#### 9. Event Volume by Appliance · `ds_event_volume`

```
`appgate_lf` | search appliance="$appliance$" | timechart span=1h count by appliance
```

- **Source:** every daemon and event type.
- **Raw fields:** `hostname` (top level), `_time`
- **Via macro:** `appliance`
- **Filters:** `$appliance$`
- **Blank means:** nothing in the index.

> **Appliance Liveness as a shape.** A line dropping to the floor is an appliance that went quiet —
> and unlike the table, it shows you *when* and whether it recovered. A line that never starts is an
> appliance that was already dead when the window opened; the table catches that case, the chart
> doesn't. Use both.

---

#### 10. Controller Messages · `ds_controller_messages`

```
`appgate_lf` | search event_type=admin_message_posted | sort - _time
| table _time, msg_level, msg_category, msg_source, msg_text
```

- **Source:** `cz-controllerd` → `admin_message_posted`
- **Raw fields:** `log.admin_message.level`, `.category`, `.source`, `.message`
- **Via macro:** `msg_level`, `msg_category`, `msg_source`, `msg_text`
- **Filters:** none
- **Blank means:** the controller has posted no operational messages — common and not a problem.

> The controller's own operational log: license warnings, certificate expiry, cluster events. The one
> panel where AppGate tells you about itself in prose rather than in inferred metrics. Worth a glance
> before a demo — an expiring-license banner here explains oddities elsewhere.

---

### Entitlements

Two panels: what actually carried traffic, and how that compares to what was granted.

> **Three panels were removed from this section** — Top Policies, Entitlements by Assignment, and
> Conditions Satisfied. See [Removed panels](#removed-panels) for their queries and the conditions
> under which they'd be worth restoring.

---

#### 11. Entitlements — by Actual Use · `ds_ent_used`

```
`appgate_lf` | search event_type=ip_access entitlement="$entitlement$"
| stats count as accesses, dc(ag_user) as users, dc(dest_ip) as resources by entitlement
| sort - accesses
```

- **Source:** `cz-vpnd` → `ip_access`
- **Raw fields:** `log.rule_name`, `log.destination_ip`, `log.distinguished_name_user`
- **Via macro:** `entitlement` from `rule_name`; `dest_ip` from `destination_ip`; `ag_user`
- **Filters:** `$entitlement$`
- **Blank means:** no `ip_access` events — check that `cz-vpnd` is forwarding.

> Entitlements that actually **carried traffic**, read from the gateway rather than the controller.
> `entitlement` maps from `rule_name`, the entitlement rule that permitted the packet.
>
> This panel reads a **scalar** field, which is why it kept working while its array-field neighbours
> went blank — that contrast is what localized the fault to array naming rather than to the macro,
> the index, or the feed.

---

#### 12. Assigned vs Actually Used · `ds_drift`

```
`appgate_lf` | search event_type=authorization_succeeded entitlement_names=*
| mvexpand entitlement_names | stats dc(ag_user) as assigned by entitlement_names
| rename entitlement_names as ent
| append [ search `appgate_lf` | search event_type=ip_access
           | stats dc(ag_user) as used by entitlement | rename entitlement as ent ]
| stats max(assigned) as assigned, max(used) as used by ent
| fillnull value=0 assigned used
| eval gap=if(assigned-used>0, assigned-used, 0) | sort - gap
```

- **Source:** **two** — `cz-controllerd` → `authorization_succeeded` (assigned) joined to `cz-vpnd` → `ip_access` (used).
- **Raw fields:** `log.entitlement_names` (array), `log.rule_name`, `log.distinguished_name_user`
- **Via macro:** `entitlement_names` passes through stage 3; `entitlement` from `rule_name`; `ag_user`
- **Filters:** none — collective-wide
- **Blank means:** an all-zero **Assigned** column with **Used** populated is the array-naming fault, not real data — the subsearch reads `entitlement` (scalar, works) while the outer search reads `entitlement_names` (array). It renders half-populated, which reads as "every entitlement is fully used," the exact inverse of the truth. **Verify Assigned is non-zero before demoing this panel.**

> **The headline panel, and the reason this section survived the cull.** Entitlements granted to
> users who never exercised them — the least-privilege gap, computed entirely from the audit stream
> with no poller.
>
> It internally does what the removed **Entitlements by Assignment** panel did, then subtracts actual
> use — so the assignment data is still on the dashboard, expressed as a gap rather than a raw list.
>
> Joins the two sides on entitlement name. The `stats max(...) by ent` after `append` is a merge
> idiom, not an aggregation — `append` produces disjoint rows (assigned-only or used-only) and `max`
> collapses each pair to one row.
>
> **Expect a large gap here.** A single sampled user carried 17 entitlement names under 5 policies,
> spanning Acquisitions, HR, and Enterprise services — plus "01 - Lunch Menu - Hanscom AFB". Broad
> assignment against narrow actual use is exactly the story this panel exists to tell.
>
> **What it cannot see:** entitlements nobody was granted during the window, and disabled
> entitlements. The window bounds the universe. A complete catalog needs the poller's
> `entitlement_catalog`.

---

### Resource Access

All three read `cz-vpnd` → `ip_access` and all three honor `$entitlement$`.

---

#### 13. Traffic — Allowed vs Dropped vs Rejected · `ds_traffic`

```
`appgate_lf` | search event_type=ip_access entitlement="$entitlement$"
| timechart span=1h count by ag_action
```

- **Source:** `cz-vpnd` → `ip_access`
- **Raw fields:** `log.action`, `log.rule_name`, `_time`
- **Via macro:** `ag_action` from `action`; `entitlement` from `rule_name`
- **Filters:** `$entitlement$`
- **Blank means:** `cz-vpnd` isn't forwarding.

> Enforcement over time, split three ways where the Access Denials tile merges two. A **drop/reject
> spike with flat allow** is a policy or posture change biting; **both rising together** is just more
> traffic.

---

#### 14. Entitlement Access Outcomes · `ds_ent_outcomes`

```
`appgate_lf` | search event_type=ip_access entitlement="$entitlement$"
| stats count as total, count(eval(ag_action="allow")) as allowed,
        count(eval(ag_action="drop" OR ag_action="reject")) as denied by entitlement
| eval deny_pct=if(total>0, round(denied*100/total, 1), 0) | sort - denied
```

- **Source:** `cz-vpnd` → `ip_access`
- **Raw fields:** `log.action`, `log.rule_name`
- **Via macro:** `ag_action`, `entitlement`
- **Filters:** `$entitlement$`
- **Blank means:** `cz-vpnd` isn't forwarding.

> **Deny rate per entitlement — a demo panel worth dwelling on.** A high Deny % is diagnosable: a
> misconfigured entitlement, an over-narrow resource definition, or a condition failing unexpectedly.
> Unlike a raw denial count, it's normalized, so a low-traffic entitlement denying everything ranks
> as loudly as a busy one.
>
> Sorted by absolute denials, not percentage — so a 100%-deny entitlement with 3 requests sits below
> a 20%-deny entitlement with 10,000. Sort the Deny % column to invert that.
>
> With **Conditions Satisfied** removed, this is now the dashboard's only window onto condition
> behaviour — a condition failing unexpectedly shows up here as an unexplained deny rate.

---

#### 15. Top Resources Accessed · `ds_top_resources`

```
`appgate_lf` | search event_type=ip_access entitlement="$entitlement$" ag_user="$user$"
| eval dest_port=coalesce(dest_port, "--")
| stats count as accesses, dc(ag_user) as users by dest_ip, dest_port, ag_protocol
| sort - accesses
```

- **Source:** `cz-vpnd` → `ip_access`
- **Raw fields:** `log.destination_ip`, `log.destination_port`, `log.protocol`, `log.rule_name`, `log.distinguished_name_user`
- **Via macro:** `dest_ip`, `dest_port`, `ag_protocol`, `entitlement`, `ag_user`
- **Filters:** `$entitlement$` **and** `$user$` — the only panel honoring two
- **Blank means:** `cz-vpnd` isn't forwarding.

> What's actually being reached, by IP/port/protocol. The `coalesce(dest_port, "--")` is deliberate:
> ICMP and some protocols carry no port, and without it those rows vanish from `stats ... by`.
>
> **IPs, not hostnames.** The gateway logs what it routes. Name resolution would need the poller's
> per-entitlement host detail.

---

### Authentication & Attack Surface

---

#### 16. Authentication Outcomes · `ds_auth_outcomes`

```
`appgate_lf` | search event_type IN (authentication_succeeded, authentication_failed)
| timechart span=1h count by event_type
```

- **Source:** `cz-controllerd` → `authentication_succeeded`, `authentication_failed`
- **Raw fields:** `log.event_type`, `_time`
- **Via macro:** pass-through
- **Filters:** none
- **Blank means:** `cz-controllerd` isn't forwarding.

> Success and failure side by side, so the **ratio** is legible. Failures alone can't distinguish a
> brute-force attempt from a busy morning; this can.
>
> **Authentication only** — proving identity. Authorization (what you then qualify for) is a separate
> event, and with the policy panels removed it now reaches the dashboard only through **Assigned vs
> Actually Used**.

---

#### 17. Authentication Failures · `ds_auth_failures`

```
`appgate_lf` | search event_type=authentication_failed ag_user="$user$" | sort - _time
| table _time, ag_user, ag_reason, ag_client_ip, ag_country, device_hostname, appliance
```

- **Source:** `cz-controllerd` → `authentication_failed`
- **Raw fields:** `log.distinguished_name_user`, `log.reason` → `log.drop_reason`, `log.client_ip`, `log.geoip.country_name`, `log.hostname`, `hostname` (top level)
- **Via macro:** `ag_user`, `ag_reason`, `ag_client_ip`, `ag_country`, `device_hostname`, `appliance`
- **Filters:** `$user$`
- **Blank means:** no failed logins in the window.

> Raw event detail — who, why, from where, on what device. **This panel shows both hostnames at once**
> (`Device` = the client, `Appliance` = the gateway), which is the clearest illustration of why the
> macro must not coalesce them.
>
> The natural place to attach a drilldown into the User Investigation dashboard; the README has the
> `eventHandlers` snippet.

---

#### 18. Appliance SSH Attack Surface · `ds_ssh_attacks`

```
`appgate_lf` | search event_type=ssh_access_failed
| stats count as attempts, dc(ag_client_ip) as source_ips, values(ag_user) as accounts,
        latest(_time) as last by ag_country, appliance
| eval last=strftime(last, "%F %T") | sort - attempts
```

- **Source:** `cz-appliance` → `ssh_access_failed`
- **Raw fields:** `log.client_ip`, `log.geoip.country_name`, `log.username`, `hostname` (top level), `_time`
- **Via macro:** `ag_client_ip`, `ag_country`, `ag_user` from `username`, `appliance`
- **Filters:** none — collective-wide
- **Blank means:** no SSH attempts. Genuinely good, but a quiet panel on stage.

> **A panel to build the demo around.** Geolocated brute-force against appliance management
> interfaces, grouped by origin country. Concrete, external, unambiguous — it makes the case against
> exposing appliance SSH better than any slide.
>
> `ag_user` here resolves via the `username` fallback, not `distinguished_name_user` — these are
> attempted OS logins (`root`, `admin`, `ubuntu`) with no AppGate identity behind them. That's why
> **Targeted Accounts** reads as a dictionary list.
>
> Geo works because the sources are internet-routable, so the LogForwarder attaches a full `geoip`
> block.

---

#### 19. Session & Tunnel Churn · `ds_session_churn`

```
`appgate_lf` | search event_type IN (session_created, session_reconnected, session_removed,
                                     tunnel_connected, tunnel_established, tunnel_closed)
| timechart span=1h count by event_type
```

- **Source:** **two** — `cz-sessiond` → `session_*`, and `cz-vpnd` → `tunnel_*`
- **Raw fields:** `log.event_type`, `_time`
- **Via macro:** pass-through
- **Filters:** none
- **Blank means:** neither daemon is forwarding.

> Connection stability. Steady low churn is healthy; a **reconnect/close spike with no matching
> create spike** is an unstable path, not new users.
>
> **The event names here were corrected against live data.** The CrowdStrike LogScale parser implied
> `tunnel_up` / `tunnel_down`; AppGate 6.7 emits `tunnel_connected`, `tunnel_established`, and
> `tunnel_closed`. Querying the assumed names returned nothing, with no error. Re-verify after any
> AppGate upgrade with `` `appgate_lf` | stats count by daemon, event_type ``.

---

#### 20. Active Users Over Time · `ds_active_users`

```
`appgate_lf` | search ag_user=* | timechart span=1h dc(ag_user) as Users
```

- **Source:** any identity-bearing event.
- **Raw fields:** `log.distinguished_name_user` → `log.username`, `_time`
- **Via macro:** `ag_user`
- **Filters:** none
- **Blank means:** no identity-bearing events.

> The Active Users tile as a trend. **Buckets don't sum to the tile** — `dc` is computed per bucket,
> and a user active all day counts once per hour here but once total in the tile. That's correct
> behavior for a distinct count, and worth pre-empting if someone asks on stage.

---

#### 21. Client Activity Map · `ds_map`

```
`appgate_lf` | search ag_lat=* ag_user="$user$" | geostats latfield=ag_lat longfield=ag_lon count
```

- **Source:** any event the LogForwarder geo-enriched — mainly `ip_access`, `authorization_succeeded`, and `ssh_access_failed`.
- **Raw fields:** `log.geoip.location.lat`, `log.geoip.location.lon`
- **Via macro:** `ag_lat`, `ag_lon`
- **Filters:** `$user$`
- **Blank means:** no `geoip` block — clients are on non-routable addresses that can't be geolocated.

> **Enrichment is the LogForwarder's, not AppGate's.** AppGate logs the IP; the LogForwarder resolves
> it. Purely internal RFC-1918 clients produce no `geoip` block and an empty map — no query fixes
> that.
>
> In this demo environment clients connect from public AWS addresses, so the map populates. An
> earlier README note predicted the opposite from a single internal-only sample event and was wrong.
> Confirm before a demo with:
> ```
> `appgate_lf` | search event_type=ip_access | stats count(ag_lat) as with_geo, count as total
> ```
>
> **Mixes legitimate users with SSH attackers**, since both carry geo. Unexpected countries are more
> likely tile 6's brute-force traffic than a traveling employee — check **SSH Attack Surface** before
> drawing conclusions.

---

### Admin Audit

---

#### 22. Admin & Configuration Activity · `ds_admin_activity`

```
`appgate_lf` | search event_type IN (entities_listed, on_boarded_devices_listed, entity_created,
                                     entity_updated, entity_deleted, admin_authorization_succeeded,
                                     admin_authorization_failed) ag_user="$user$"
| sort - _time
| table _time, event_type, entity_type, entity_name, ag_user, ag_client_ip, exec_ms, appliance
```

- **Source:** `cz-controllerd` → the seven event types above.
- **Raw fields:** `log.event_type`, `log.entity_type`, `log.entity_name`, `log.distinguished_name_user`, `log.client_ip`, `log.execution_ms`, `hostname` (top level)
- **Via macro:** `ag_user`, `ag_client_ip`, `exec_ms` from `execution_ms`, `appliance`; `entity_type` / `entity_name` pass through
- **Filters:** `$user$`
- **Blank means:** no admin activity at all — unusual, since API reads alone normally populate it.

> **Reads and writes in one table, deliberately.** `entity_created` / `_updated` / `_deleted` appear
> only when an admin actually changes something, which in a stable collective is rarely. What this
> environment emits constantly is `entities_listed` and `on_boarded_devices_listed` — API *reads*,
> largely from the CrowdStrike poller's service account. Including them keeps the panel alive; the
> **Event** column tells you which kind you're looking at.
>
> `Object Type` is exposed so you can see which entity kinds appear and narrow from there.
>
> **`exec_ms` is a controller performance signal** riding along free. Climbing execution times under
> steady volume are a controller under pressure — the closest thing to a resource metric available
> without the poller.
>
> This is the section the CrowdStrike original **declared but never rendered**: its "Security &
> Compliance" section had no `widgetIds`, orphaning four widgets including Admin Configuration
> Activity. Reproduced here intentionally.

---

## Removed panels

Three panels were cut from the Policies/Entitlements section. Their queries are preserved here so
restoring one is a copy-paste rather than a re-derivation. All three read **array** fields, so any
restoration depends on macro stage 3 working — confirm with
[Known issue 1](#1-array-field-panels-were-blank--macro-fix-applied-pending-confirmation) first.

### Top Policies — was `ds_top_policies`

```
`appgate_lf` | search event_type=authorization_succeeded policy_names=*
| mvexpand policy_names
| stats dc(ag_user) as users, count as grants, latest(_time) as last by policy_names
| eval last=strftime(last, "%F %T") | sort - users
| rename policy_names as Policy, users as Users, grants as Authorizations, last as "Last Granted"
```

- **Source:** `cz-controllerd` → `authorization_succeeded`
- **Raw fields:** `log.policy_names` (array), `log.distinguished_name_user`, `_time`
- Panel type was `splunk.table`, `count: 25`, `dataOverlayMode: heatmap`

> Policies ranked by distinct users granted. The data is genuinely present — a sampled event carried
> `["Enterprise DNS Access", "Enterprise DNS Resolution", "Enterprise Services Access",
> "Human Resources Access Policy", "Test Policy"]`.
>
> **Restore when** you want the policy layer visible in its own right. Note that policy names appear
> nowhere else in the LogForwarder stream, so nothing corroborates them — and **Assigned vs Actually
> Used** already exposes the over-provisioning story one level down, at the entitlement.
>
> **Authorizations counts events, Users counts people.** A high ratio means frequent re-authorization
> (short token lifetimes), not broad access.

### Entitlements — by Assignment — was `ds_ent_assigned`

```
`appgate_lf` | search event_type=authorization_succeeded entitlement_names=*
| mvexpand entitlement_names
| stats dc(ag_user) as users, dc(site_names) as sites by entitlement_names
| sort - users
| rename entitlement_names as Entitlement, users as "Users Assigned", sites as Sites
```

- **Source:** `cz-controllerd` → `authorization_succeeded`
- **Raw fields:** `log.entitlement_names` (array), `log.site_names` (array), `log.distinguished_name_user`
- Panel type was `splunk.table`, `count: 25`, `dataOverlayMode: heatmap`

> Who **qualifies** for what — the left-hand side of the least-privilege comparison. **Assigned vs
> Actually Used** computes the same figure internally as its `Assigned` column, which is why this one
> was the easiest of the three to drop.
>
> **If you restore it, change `dc(site_names)` to `values(site_names)`.** This collective has a single
> site (`["AWS"]`), so the count reads 1 on every row forever — a column of identical numbers. The
> name is at least informative and degrades gracefully if a second site appears.
>
> **`site_names` is also why there's no Site filter.** It exists only on `authorization_succeeded`, so
> a Site input would filter this panel and nothing else. An earlier README claimed `site_names` didn't
> exist at all; that was wrong.

### Conditions Satisfied — was `ds_conditions`

```
`appgate_lf` | search event_type=entitlement_token_evaluated successful_condition_names=*
| mvexpand successful_condition_names
| stats dc(ag_user) as users, count as evaluations by successful_condition_names
| sort - users
| rename successful_condition_names as Condition, users as Users, evaluations as Evaluations
```

- **Source:** `cz-sessiond` → `entitlement_token_evaluated`
- **Raw fields:** `log.successful_condition_names` (array), `log.distinguished_name_user`
- Panel type was `splunk.table`, `count: 25`, `dataOverlayMode: heatmap`

> Which access Conditions users actually satisfy — posture checks, MFA, network location. Zero-trust
> policy evaluation made visible, and conceptually the most interesting of the three.
>
> **This is the one whose source event is unconfirmed.** Unlike the other two, no sampled event has
> proven `entitlement_token_evaluated` is emitted by this collective. Check before restoring:
> ```
> `appgate_lf` | stats count by daemon, event_type | sort - count
> ```
> If the event type is absent, the array fix cannot help — the panel has no data source at all.
>
> **Records only *successful* conditions.** There is no `failed_condition_names` in this stream, so it
> could never show which condition blocked someone. That negative signal comes indirectly from
> **Entitlement Access Outcomes**' deny rate.
>
> **Restore when** the CrowdStrike integration's three quarantine Condition scripts are bound — they'd
> appear here by name, which is the panel that would make EDR-driven ZTNA enforcement visible in
> Splunk.

---

## Event-type → panel index

Reverse lookup. Run this to see what your collective actually emits:

```
`appgate_lf` | stats count by daemon, event_type | sort - count
```

| Daemon | Event type | Panels |
|---|---|---|
| *(any)* | *(any)* | Appliances Reporting · Appliance Liveness · Event Volume by Appliance |
| `cz-controllerd` | `authentication_succeeded` | Authentication Outcomes |
| `cz-controllerd` | `authentication_failed` | Auth Failures tile · Authentication Outcomes · Authentication Failures |
| `cz-controllerd` | `authorization_succeeded` | Assigned vs Actually Used · Client Activity Map |
| `cz-controllerd` | `admin_message_posted` | Controller Messages |
| `cz-controllerd` | `entities_listed`, `on_boarded_devices_listed`, `entity_*`, `admin_authorization_*` | Admin & Configuration Activity |
| `cz-vpnd` | `ip_access` | Access Denials tile · Entitlements by Actual Use · Assigned vs Actually Used · Traffic · Entitlement Access Outcomes · Top Resources · Client Activity Map |
| `cz-vpnd` | `tunnel_connected`, `tunnel_established`, `tunnel_closed` | Session & Tunnel Churn |
| `cz-sessiond` | `session_created`, `session_reconnected` | Sessions Established tile · Session & Tunnel Churn |
| `cz-sessiond` | `session_removed` | Session & Tunnel Churn |
| `cz-sessiond` | `entitlement_token_evaluated` | *(none — Conditions Satisfied removed)* |
| `cz-appliance` | `ssh_access_failed` | SSH Failures tile · SSH Attack Surface · Client Activity Map |

**Identity-bearing events** (any of the above carrying `distinguished_name_user` or `username`) feed
Active Users, Users per Appliance, and Active Users Over Time.

### Event types the CrowdStrike parser assumed that do not exist here

Panels depending on these were **removed**, not left blank.

| Assumed | Reality in AppGate 6.7 |
|---|---|
| `tunnel_up` / `tunnel_down` | `tunnel_connected` / `tunnel_established` / `tunnel_closed` |
| `appliance_status_changed` | Not emitted — Status Transitions and Last Known State dropped |
| `rule_monitor_health_change` | Not emitted — Access Rule Health dropped |
| `audit_drop` | Not emitted — telemetry-loss tile dropped |
| `remedy*` | Not emitted — Posture Remediation dropped |
| daemon `cz-configd` | Does not exist here |
| daemon `sshd` | Real source is `cz-appliance` |

---

## Known issues

### 1. Array-field panels were blank — macro fix applied, pending confirmation

**Affected:** the three [removed panels](#removed-panels), plus the Assigned column of **Assigned vs
Actually Used** — the one still on the dashboard.

All four depend on **JSON array** fields (`policy_names`, `entitlement_names`, `site_names`,
`successful_condition_names`). Every non-array panel works, which localized the fault to array
handling rather than to the macro, the index, or the feed.

**"Not emitted" is ruled out** for `authorization_succeeded`. A live event from this collective
(2026-08-06, `controller.sdpfederal.com`, `cz-controllerd`) carries all three arrays populated:

```json
"policy_names": ["Enterprise DNS Access", "Enterprise DNS Resolution",
                 "Enterprise Services Access", "Human Resources Access Policy", "Test Policy"],
"entitlement_names": ["01 - Lunch Menu - Hanscom AFB", "Acquisitions Application Access", ...17 total],
"site_names": ["AWS"]
```

**Cause — Splunk's `{}` array suffix.** Splunk's JSON extraction names array fields with a trailing
`{}`, so the raw `log.policy_names` array extracts as the field **`log.policy_names{}`**. The macro's
`rename log.* AS *` yielded `policy_names{}`, so `search policy_names=*` matched nothing and the panel
rendered empty **with no error** — the same silent-blank class of failure the sourcetype-alias problem
produced.

**Fix applied:** stage 3 of `appgate_lf` is now `rename "*{}" AS "*"`, which strips the suffix from
every array field at once. One macro edit, no dashboard changes, and it covers any future panel using
these fields. It is a **no-op if your extraction doesn't add the suffix**, so it was safe to apply
before confirming.

**Confirm** — rebuild and re-upload the add-on with **Upgrade app** ticked so the macro reloads, then:

```
`appgate_lf` | search event_type=authorization_succeeded
| head 1 | table policy_names, entitlement_names, site_names
```

All three columns populated (multivalue) means fixed — and **Assigned vs Actually Used** should now
show a non-zero Assigned column. Still empty means the suffix wasn't the cause; fall back to naming
the real fields directly:

```
index=appgate log.event_type=authorization_succeeded
| head 1 | fields log.* | transpose 1
| rename column as Field, "row 1" as Value | sort Field
```

### 1a. `entitlement_token_evaluated` is unverified

The removed **Conditions Satisfied** panel is the one whose **event type has not been confirmed to
exist** — the sample event above is `authorization_succeeded`, which does not carry
`successful_condition_names`. If you restore that panel, verify the event type first; the macro fix
helps only if `cz-sessiond` actually emits it.

```
`appgate_lf` | stats count by daemon, event_type | sort - count
```

### 2. `README.md` Option C ships a macro definition that reintroduces a fixed bug

The hand-install path in the main README defines:

```
... | eval appliance=coalesce('log.hostname', hostname, 'log.log_source', log_source) | ...
```

`macros.conf` exists specifically to document why this is wrong: `log.hostname` is the **client
device**, so this fills the appliance column of Appliance Liveness with client device names — wrong
output that looks plausible. It also omits `device_hostname` entirely, breaking the Device column of
Authentication Failures.

Option C additionally names the OU alias `ag_user_ou`, while the shipped macro uses **`ag_ou`**, and
predates stage 3 entirely — so a hand-installed macro will still show blank array panels even after
the add-on is fixed.

**Use the definition in `TA-appgate-demo/default/macros.conf` as the source of truth**, not the
README's Option C block. Option C needs updating.

### 3. `README.md` points at the wrong User Investigation file — RESOLVED

The Dashboard 2 section said to import `dashboards/appgate_user_investigation.json` and described 22
panels. That file is a **2-panel stub** (header + Identity Summary) whose own description reads
"panels will be added incrementally." The complete dashboard was
**`appgate_user_investigation_full.json`** (33 layout items).

The stub also carried a mangled dropdown value: label `loadgen-scale-probe_7tae3` against value
`loadgen-scale_loadgen-scale_loadgen-scale-probe_7tae3`, which matches nothing.

> **Resolved.** Both files moved to `dashboards/archive/`, and the README now documents Investigate
> User as **tab 2 of `appgate_collective_health.json`** rather than a separate import. There is one
> dashboard to paste. The mangled value was not carried into the new dropdowns.

### 4. Panel-count drift in comments

`macros.conf` says "all 23 panels". The consolidated three-tab dashboard now has **31** data panels
plus 11 markdown blocks (42 visualizations): 22 on Health Overview, 2 on Investigate User, 7 on
Compare Users. Cosmetic, but the comment has never matched.

### 5. A restored Assignment panel should not use `dc(site_names)`

`site_names` on the sample `authorization_succeeded` is `["AWS"]` — this collective has a **single
site**. `dc(site_names)` would return 1 for every row, forever. Use `values(site_names)` instead. This
only matters if you restore Entitlements by Assignment.

### 6. Demo caution — the person-named identities are synthetic load-gen

The sampled user `dwight-eisenhower` looks like an interactive user and is not. Its `device_claims`
say `"clientType": "headless"`, `"profileName": "fed-loadgen-local"`, and its `scripted_user_claims`
carry **`"synthetic": true`** alongside `clearance: top-secret`, `homeBase: Pentagon`,
`riskScore: 53`, `role: supervisor`.

Two consequences:

- The claims values are **generated, not real IdP output**. Fine to demo, but don't describe them as
  live attributes from a federated IdP if anyone asks precisely.
- The main README advises picking "a real interactive user" for the User Investigation claims panels
  because service accounts carry no claims. In this collective that advice inverts — the load-gen
  identities are the ones *with* a full `scripted_user_claims` block. Filter on
  `device_client_type != headless` if you need a genuinely interactive session, or accept the
  synthetic ones knowingly.

### 7. Static dropdowns go stale — MITIGATED, NOT SOLVED

`dwight-eisenhower` did not appear in the hardcoded user list in the archived
`appgate_user_investigation.json` (`chuck-yeager`, `sally-ride`, `benjamin-franklin`, `admin`,
`crowdstrike-api-poller`, `loadgen-scale-probe_7tae3`). The collective grew identities the static list
didn't know about.

That's the tradeoff the main README names — static `items` can't track new users — showing up in
practice rather than in theory. It was previously the argument for keeping text inputs on Collective
Health.

> **Round two: static dropdowns were shipped anyway,** on the argument that click-to-filter made the
> lists a convenience rather than the authority. The residual risk was written up honestly at the time
> and never went away: the lists were hand-maintained, built from event samples recorded in this repo
> rather than from a live query, and three pre-flight regeneration searches were carried in the README
> and DEMO-BRIEF as the mitigation. A pre-flight step that must be run by hand against every new
> environment is not a mitigation, it is a deferred failure.
>
> **Round three — RESOLVED. The dropdowns are gone and so is every hardcoded value.** All five inputs
> are `input.text` defaulting to `*`. There is no list to maintain, nothing to regenerate, and the
> dashboard no longer contains a single username, appliance hostname, or entitlement name. The three
> things the static lists were bought with are each replaced:
>
> - **Value discovery** → the tables, which were already doing it. **Users in Window**, **Appliance
>   Liveness**, and **Assigned vs Actually Used** ignore their own filters specifically so they list
>   everything, and all three are click-to-filter. A user who appeared five minutes ago is clickable.
> - **Wildcard presets** → typed instead of picked. `*`, `*loadgen*`, `10.10.*`, `Acquisitions*` all
>   still work; they are just no longer menu items that assert a particular environment.
> - **Populated-on-load for the Compare and Investigate tabs** → the `appgate_top_user`,
>   `appgate_pair`, and `appgate_slot` macros, which resolve `*` to the busiest identity (or two) in
>   the window at search time. This is what previously required naming two real users in the input
>   defaults, and it was the last hardcoded thing standing.
>
> **What the fix cost.** The head-to-head panels can no longer put the compared names in their titles,
> because the tokens now usually hold `*` rather than a name. The names appear as the two column
> headers of the head-to-head table instead, and the surrounding markdown points at them. Also gone is
> the cosmetic mismatch where a drilldown set a token to a value absent from `items` and the dropdown
> showed no selection — a text box always shows what it is filtering on.
