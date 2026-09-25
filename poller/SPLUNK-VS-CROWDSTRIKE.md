# Splunk vs. CrowdStrike — What Each Build Can Actually Do

A capability comparison between the **AppGate Splunk demo** in this repo and the **Appgate ZTNA
Insights for CrowdStrike** bundle in `appgate-crowdstrike-integration-main`.

Every claim about the CrowdStrike side was verified against the shipped bundle — dashboard YAML,
Foundry manifest, api-integration spec, and poller config — not against its README.

**Contents**

- [The honest headline](#the-honest-headline)
- [Why the difference exists — it's the data sources](#why-the-difference-exists--its-the-data-sources)
- [Capability matrix](#capability-matrix)
- [Only CrowdStrike can do this](#only-crowdstrike-can-do-this)
- [Only the Splunk build does this](#only-the-splunk-build-does-this)
- [Cost to deploy](#cost-to-deploy)
- [Defects in the reference build](#defects-in-the-reference-build)
- [Which one for which job](#which-one-for-which-job)
- [Closing the gaps](#closing-the-gaps)
- [Verified counts](#verified-counts)

---

## The honest headline

**These are not competing products, and neither is a subset of the other.**

The CrowdStrike bundle is a four-component control plane: two data feeds, a Foundry app with 63
function handlers, a Flask receiver running on a controller, and three access Conditions. It can
**change access** — a detection quarantines a user and their network access flips mid-session with no
re-authentication.

The Splunk build is one add-on and one dashboard, fed by a log stream the collective already emits.
It cannot change anything. It is strictly read-only telemetry.

The interesting part is what happens when you compare them on **analysis** rather than on
capability count:

> The CrowdStrike build has **more data available** — it polls the Controller API for the complete
> entitlement catalog — and does **less with it analytically**. There is no least-privilege gap panel,
> no per-entitlement deny rate, and no user-vs-user comparison anywhere in its dashboards. The Splunk
> build derives all three from the audit stream alone.

That's the line worth having ready: CrowdStrike buys **breadth and enforcement** with real
infrastructure. Splunk buys **depth of analysis** with none.

---

## Why the difference exists — it's the data sources

Almost every capability difference traces back to one architectural fact.

| Build | Feeds | What that unlocks |
|---|---|---|
| **Splunk (this repo)** | LogForwarder only → `index=appgate` | Everything AppGate *audits*: who authenticated, what was authorized, what traffic flowed, what was denied |
| **CrowdStrike** | LogForwarder → `identity_repo`<br>**+ Poller → `snapshot_repo`**<br>+ optional DPI repo | The above, **plus** anything the Controller API *reports*: resource utilization, site health, live session counts, device inventory, the full entitlement catalog |

The poller is the whole difference on the read side. It is a standalone Python service that hits
eight Controller API endpoints:

```
/admin/stats/appliances              <- CPU, disk, RAM, NIC
/admin/sites/status                  <- site health
/admin/stats/active-sessions/dashboard  <- live concurrent sessions
/admin/stats/active-sessions/dn      <- client version / platform per user
/admin/on-boarded-devices            <- registered devices, licensed users
/admin/appliances/                   <- appliance inventory
/admin/session-info/                 <- session detail
/admin/discovered-apps               <- app discovery
```

**Anything in that list is structurally impossible in the current Splunk build** — not a query we
haven't written, but data that does not exist in any audit event. Conversely, everything the Splunk
build does is available to CrowdStrike too; it simply wasn't built there.

> The Splunk add-on already ships the `appgate_api`, `appgate_latest_snapshot`, and
> `appgate_latest_catalog` macros and an `[appgate:api]` props stanza. The poller emits Splunk HEC
> natively with no code changes. The upgrade path is open — see [Closing the gaps](#closing-the-gaps).

---

## Capability matrix

Legend: ● full · ◐ partial or indirect · ○ absent

### Appliance and collective health

| Capability | Splunk | CrowdStrike | Note |
|---|:--:|:--:|---|
| Appliance liveness / staleness detection | ● | ○ | Splunk-only health model — silence as the failure signal |
| Event volume per appliance | ● | ◐ | CS has *Sessions by Appliance*, not all-event volume |
| CPU / disk / RAM / NIC utilization | ○ | ● | Poller `/admin/stats/appliances` |
| Site health | ○ | ● | Poller `/admin/sites/status` |
| Health by appliance (polled status) | ○ | ● | Poller |
| Controller operational messages | ● | ○ | `admin_message_posted` |
| Controller execution time as a load signal | ● | ○ | `execution_ms`, free with the audit event |
| Ingest rate / telemetry-loss monitoring | ○ | ◐ | CS widget exists but **never renders** (orphaned) |

### Access, policy, entitlement

| Capability | Splunk | CrowdStrike | Note |
|---|:--:|:--:|---|
| Traffic allowed vs dropped vs rejected | ● | ● | Both |
| Top resources accessed | ● | ● | Both |
| Resource access by protocol | ◐ | ● | Splunk shows protocol as a column, not its own panel |
| **Deny rate per entitlement** | ● | ○ | Normalized — Splunk-only |
| **Assigned vs actually used (least-privilege gap)** | ● | ○ | **Splunk-only, and the headline capability** |
| Entitlement assignment per user | ● | ◐ | CS surfaces it in the Identity Profile, not as an analysis |
| Complete entitlement catalog | ○ | ● | Poller. Splunk sees only what appeared in the window |
| Disabled / never-granted entitlements | ○ | ● | Poller catalog only |
| Per-entitlement host/port/CIDR definitions | ○ | ● | Poller catalog only |
| Conditions satisfied | ○ | ◐ | Removed from Splunk — event type unconfirmed; CS widget is orphaned |

### Identity and device

| Capability | Splunk | CrowdStrike | Note |
|---|:--:|:--:|---|
| Per-user identity summary | ● | ● | Both |
| IdP claims | ● | ● | Splunk discovers schema; CS hardcodes claim names |
| UCS / scripted risk claims | ● | ● | Both |
| Device posture from `device_claims` | ● | ● | Both |
| **CrowdStrike AID cross-system pivot** | ○ | ● | Links an AppGate identity to the Falcon host record |
| **ZTA (Zero Trust Assessment) score** | ○ | ● | Requires CrowdStrike sensor data — not an AppGate field |
| Registered devices / licensed users | ○ | ● | Poller |
| Client version / platform distribution | ○ | ● | Poller |
| **User-vs-user comparison** | ● | ○ | **Splunk-only** |

### Authentication and attack surface

| Capability | Splunk | CrowdStrike | Note |
|---|:--:|:--:|---|
| Authentication outcomes and trend | ● | ● | Both |
| Authentication failure detail | ● | ● | Both |
| Auth failure *rate* (normalized) | ● | ○ | Splunk-only |
| **Appliance SSH attack surface (geolocated)** | ● | ○ | Splunk-only; CS parser assumed daemon `sshd`, which doesn't exist |
| Client activity map | ● | ● | Both |
| Session and tunnel churn | ● | ◐ | CS has session counts; Splunk has churn with corrected event names |
| Live concurrent session count | ○ | ● | Poller |

### Response and enforcement

| Capability | Splunk | CrowdStrike | Note |
|---|:--:|:--:|---|
| Quarantine / unquarantine user | ○ | ● | Fusion SOAR action → receiver |
| Blacklist / unblacklist user | ○ | ● | Fusion SOAR action → receiver |
| Revoke session | ○ | ● | Fusion SOAR action → receiver |
| **Mid-session access change, no re-auth** | ○ | ● | The bundle's signature capability |
| Auto-quarantine on detection | ○ | ● | Fusion workflow, Detection trigger → Quarantine |
| Policy / entitlement / condition authoring | ○ | ● | With apply, history, and rollback |
| Inline in the detection workflow | ○ | ● | Panel extension renders in the NG-SIEM detection slide-out |
| Grant forensic access | ○ | ● | Third Condition script |

### Layer 7

| Capability | Splunk | CrowdStrike | Note |
|---|:--:|:--:|---|
| DPI / L7 traffic, JA4, shadow-AI detection | ○ | ◐ | **Not included in either.** CS ships the tab disabled; needs a separate Suricata + nDPI engagement |

---

## Only CrowdStrike can do this

Ranked by how hard they are to argue with.

**1. It changes access.** Five verified Fusion SOAR actions (`/fusion/quarantine`, `/unquarantine`,
`/blacklist`, `/unblacklist`, `/revoke`) POST to a receiver running on the controller. A CrowdStrike
detection can quarantine a user and their network access flips **mid-session with no
re-authentication**. Nothing in Splunk approaches this, and nothing in Splunk should — it's a
different class of system.

**2. It lives where the analyst already is.** The panel extension renders inside the NG-SIEM detection
slide-out. The analyst never context-switches. A Splunk dashboard is a place you go; this is a thing
that appears.

**3. It correlates to the endpoint.** The CrowdStrike AID is the cross-system pivot: an AppGate
identity links to the Falcon host record, and ZTA scores come along. AppGate's audit stream has no
concept of either.

**4. It can author policy.** The Foundry manifest carries handlers for listing, applying, and rolling
back conditions, entitlements, and policies, plus audit history and topology. That is a management
surface, not a dashboard.

**5. It sees the full catalog.** The poller ships the complete entitlement and policy inventory, so it
can answer "what exists" rather than only "what was exercised in this window."

**Worth knowing about the enforcement mechanism.** The bundle's own README documents that its
Conditions call the receiver via `httpGet` at evaluation time, and **fail open on any error** — so a
gateway that can't reach the receiver silently disables enforcement, with quarantine appearing to work
while doing nothing. They hit real AWS same-VPC hairpin and gateway DNS failures. It's a candid
writeup, and it's a genuine sharp edge in a security control. Read `ENFORCEMENT-MODELS.md` before
promising the capability.

---

## Only the Splunk build does this

Not a consolation list — these are analyses that don't exist anywhere in the CrowdStrike bundle
despite it having strictly more data.

**1. Assigned vs Actually Used — the least-privilege gap.** Joins entitlements granted (controller,
`authorization_succeeded`) against entitlements that carried traffic (gateway, `ip_access`). The
delta is over-provisioning. The CrowdStrike dashboards have no equivalent panel at any level.

**2. Compare Users.** Head-to-head differential across two identities — entitlement, policy, and
actual resource reach, each tagged `both` / `A only` / `B only`, plus a cohort matrix ranking every
user on the same measures. This is the access-review question, and CrowdStrike has no per-user
comparison anywhere.

**3. Deny rate per entitlement.** Normalized, so a low-traffic entitlement denying everything ranks
as loudly as a busy one. CrowdStrike counts denials; it doesn't rate them.

**4. Liveness as the health model.** Because no polled gauges were available, health was redefined as
"has this appliance stopped talking." That's arguably a *better* signal for the failure case that
matters — it needs no cooperation from the appliance, so it catches a partitioned or wedged appliance
faster than a status poll.

**5. Appliance SSH attack surface.** Geolocated brute-force against appliance management interfaces,
by origin country and targeted account. CrowdStrike's parser expected daemon `sshd`; the real source
is `cz-appliance`, so the reference build finds nothing.

**6. Schema-agnostic claims.** The CrowdStrike Identity Profile hardcodes `user_claims.Branch`,
`.Citizenship`, `.clearance`, `.riskScore`, `.IA_Training_Status` — and its own comments admit these
are example claims from one federal demo schema. On any other tenant that renders a table of empty
rows. Splunk uses `transpose` to show whatever the IdP actually emits, so it works anywhere and
doubles as a discovery tool.

**7. It renders everything it declares.** See [Defects](#defects-in-the-reference-build).

---

## Cost to deploy

The starkest difference, and usually the one that decides the conversation.

| | Splunk | CrowdStrike |
|---|---|---|
| **Time** | Minutes | ~half a day |
| **Components** | 1 add-on + 1 dashboard | 4 (data plane, Foundry app, controller customization, access policy) |
| **Services to run** | None | Flask receiver on a controller + poller |
| **Network prerequisites** | None | **Public DNS name + CA-signed TLS cert** — the Condition sandbox rejects self-signed and localhost |
| **Build toolchain** | None | `foundry` CLI, Node + pnpm, python + pyyaml/requests/falconpy, zip |
| **Tenant prerequisites** | A Splunk index | Falcon tenant with Foundry **and** NG-SIEM; OAuth client with **NGSIEM Write** (not just Read — a search is a POST), ZTA Read, Hosts Read, Alerts Read, Dashboards Read |
| **AppGate prerequisites** | LogForwarder → Splunk, JSON extraction on | A free customization slot on a controller; 3 Condition scripts pasted; entitlements and policies created |
| **Changes your collective?** | No — read-only | Yes — runs code on a controller, binds Conditions to entitlements |
| **Rollback** | Delete an app | Uninstall app, remove customization, unbind Conditions |

**The trap on the CrowdStrike side:** the receiver secret can **only** be entered at the app install
wizard. There is no API and no post-install reconfigure surface. Click past that step and the only
recovery is uninstall and reinstall.

---

## Defects in the reference build

Verified directly against `AppgateZTNADashboard.yaml`, not taken from documentation.

**1. A whole section never renders.** `Security & Compliance` is declared with a title and an order but
**no `widgetIds` key at all**. Four widgets are orphaned — declared in the `widgets:` block, referenced
by no section, so they never appear:

- Admin Configuration Activity
- Polled Endpoints
- Access Denied — Failed Policy Conditions
- Ingest Rate

31 widgets are declared; 27 are placed. **13% of the dashboard is invisible.** The Splunk build
deliberately renders Admin & Configuration Activity, which is one of the four.

**2. Undefined section order.** `Remote Access Activity` and `Security & Compliance` both declare
`order: 3`, so display order between them is undefined.

**3. `Distinct Resources Today` is ambiguous, not simply wrong.** Its query is:

```
#Vendor = appgate | groupBy(destination.ip, destination.port) | count()
```

This counts distinct **ip+port pairs**, so one host exposing five ports counts as five "resources."
Whether that's a bug depends on your definition of a resource — but it inflates against the natural
reading of "distinct resources," and it isn't stated on the panel.

> Correction worth noting: this repo's main `README.md` describes that query as counting *groups
> rather than distinct resources*. That's not quite right — counting groups **is** counting distinct
> pairs. The real issue is the ip+port granularity, not the aggregation.

**4. Assumed event types that AppGate 6.7 doesn't emit.** The LogScale parser implies
`tunnel_up`/`tunnel_down`, `appliance_status_changed`, `rule_monitor_health_change`, `audit_drop`,
`remedy*`, and daemons `cz-configd` and `sshd`. None exist in this collective. Panels built on them
return nothing, **with no error** — indistinguishable from "no activity." The Splunk build removed
those panels rather than shipping silent blanks.

---

## Which one for which job

| If the question is… | Use |
|---|---|
| "Who has access they never use?" | **Splunk** — the only build that answers it |
| "Why does A see something B can't?" | **Splunk** — Compare Users |
| "Is an appliance down right now?" | **Splunk** — liveness beats a status poll for detection speed |
| "Is an appliance out of disk?" | **CrowdStrike** — or add the poller to Splunk |
| "How many users are connected right now?" | **CrowdStrike** — or add the poller |
| "This host has a detection — cut its access." | **CrowdStrike**, and only CrowdStrike |
| "What's attacking our appliances?" | **Splunk** — SSH attack surface |
| "Show me everything about this one user." | Either — CrowdStrike wins if you're already in a detection |
| "We need something running this week." | **Splunk** — minutes vs. half a day plus DNS and certs |
| "We don't run CrowdStrike." | **Splunk** — CrowdStrike requires a Falcon tenant with Foundry and NG-SIEM |

**The one-liner if someone asks why both exist:**

> Same telemetry, two different jobs. CrowdStrike is for the analyst mid-incident who needs to act.
> Splunk is for the access review, the health check, and the question of who's over-provisioned — and
> it runs anywhere, with nothing to stand up.

---

## Closing the gaps

Ranked by effort against payoff. All of these are genuinely available.

**1. Stand up the poller — closes most of the read-side gap.**
`02-source/appgate-api-poller/` is standalone Python and **already emits Splunk HEC with no code
changes**. The Splunk add-on already ships the `appgate_api`, `appgate_latest_snapshot`, and
`appgate_latest_catalog` macros plus an `[appgate:api]` props stanza. This unlocks resource
utilization, site health, live session counts, registered devices, licensed users, and client version
distribution — the entire *Clients Fleet* section of the CrowdStrike dashboard.

> **One trap.** `entitlement_grant` and `entitlement_catalog` are **full re-ships on a heartbeat, not
> deltas.** Any query must reduce to the newest sweep — that's what the snapshot macros do — or counts
> multiply by the number of sweeps in the window. Roughly 1440× over 24h at a 60s cadence.

**2. Two portable panels, no new data.** *Claims at a Glance* and *Top Users by Resource Access*
exist in the CrowdStrike dashboard and are feedable from the LogForwarder stream today. Small.

**3. Restore the removed panels** if the array fix confirms — Top Policies and Entitlements by
Assignment. Queries are preserved verbatim in `COLLECTIVE-HEALTH-PANEL-REFERENCE.md`.

**Not closable in Splunk, at any effort:**

- **Response actions** — need the receiver and Conditions. A different architecture, not a query.
- **ZTA scores** — CrowdStrike sensor data. The AppGate-native substitute is a composite posture score
  built from `device_claims`: disk encryption, firewall enabled, not-local-admin, Azure-joined,
  Intune-enrolled. Worth building; it isn't the same number.
- **AID pivot** — requires Falcon host records.
- **Inline detection panel** — requires Foundry.
- **DPI / L7** — needs a passive Suricata + nDPI sensor, which is not part of either package.

---

## Verified counts

Measured, not quoted.

| | Splunk | CrowdStrike |
|---|---|---|
| Dashboards | 1 (3 tabs) | 2 (ZTNA + Identity Profile) |
| Data panels | **31** | **27 rendered** (31 declared, 4 orphaned) |
| Sections / tabs | 3 tabs | 4 sections, one empty |
| Data sources | 31 searches, 1 feed | 2 feeds (+1 optional DPI) |
| Filter inputs | 5 | Hostname + time |
| Foundry function handlers | — | 63 |
| SOAR response actions | 0 | 5 |
| Controller API endpoints polled | 0 | 8 |

**Splunk panels by tab:** Health Overview 22 · Investigate User 2 · Compare Users 7.

**CrowdStrike sections:** Collective Health 10 · Remote Access Activity 9 · Clients Fleet 8 ·
Security & Compliance 0.
