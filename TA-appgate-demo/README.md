# AppGate ZTNA Dashboard for Splunk

Zero-trust telemetry for AppGate ZTNA, built entirely from the audit stream the AppGate
LogForwarder already emits. No Controller API poller, no agent, no additional services — one log
feed into one Splunk index, and the dashboard works.

The app ships one dashboard, **AppGate Collective Health**, with three tabs: collective-wide health,
single-user investigation, and side-by-side comparison of two identities.

**This app is read-only telemetry.** It performs no actions against your collective.

---

## Requirements

- Splunk Enterprise 9.0+ or Splunk Cloud, with Dashboard Studio available
- AppGate SDP/ZTNA LogForwarder configured to send its JSON audit stream to Splunk
- That stream landing in a Splunk index you can name, with JSON field extraction enabled

---

## Install

1. **Apps → Manage Apps → Install app from file**, choose the package, upload, restart when prompted.
2. Point the app at your index (below). This is the only configuration step.
3. Open **AppGate ZTNA Dashboard** from the app menu.

Upgrading from an earlier version is the same flow with **Upgrade app** ticked.

---

## Configure — one macro

Everything the dashboard reads is selected by a single macro, `appgate_index`. It ships as
`index=appgate`. If your index is named something else, change it:

**Settings → Advanced Search → Search macros → `appgate_index`**, and set the definition to
`index=<your index>`.

> Edit it here, in Splunk Web, rather than in the app's `default/macros.conf`. Splunk writes your
> change to `local/`, which survives an app upgrade. Edits made directly to `default/` are
> overwritten the next time you upgrade the app.

Not sure of the index name:

```
| tstats count where index=* by index, sourcetype
```

Nothing else is environment-specific. No usernames, appliance hostnames, entitlement names, or
identity-provider attributes are hardcoded anywhere in the dashboard — every filter, default, and
panel discovers its values from your own data at search time.

### If JSON extraction isn't on

```
index=<your index> | head 1 | table sourcetype, hostname, daemon, log.event_type
```

All columns populated means extraction works. All null with JSON visible in `_raw` means it's off:
**Settings → Source types →** *(your sourcetype)* **→ Advanced**, add `KV_MODE = json`. Nothing in
this app can work until that is true — no macro can recover fields that were never extracted.

---

## Verify

Three checks, in order. Each one catches a different failure.

**1. Does the macro resolve and map fields?**

```
`appgate_lf` | table _time, appliance, daemon, event_type, ag_user, entitlement, ag_action
```

Every column should populate for `ip_access` rows. *"The search specifies a macro that cannot be
found"* means the app isn't installed, or its macros aren't shared globally.

**2. Which panels will have data?**

```
`appgate_lf` | stats count by daemon, event_type | sort - count
```

The most useful check you can run. Every panel traces to specific event types, and **a panel whose
event type is absent renders blank with no error** — indistinguishable from a broken panel. Common
cases:

| Absent from your stream | Panels that stay blank |
|---|---|
| `ip_access` | Access Denials, the entitlement and resource panels, Traffic |
| `authentication_*` | Auth Failure Rate, Auth Outcomes, Auth Failure Reasons |
| `appliance_status_changed` | Status Transitions, the Last Known State column |
| `appliance_started` / `_shutdown` / `_rebooted` | Recent Appliance State Changes |
| `session_*`, `tunnel_*` | Sessions Established, Session & Tunnel Churn |
| `entity_created` / `_updated` / `_deleted` | Policy, Entitlement, and Condition Changes |
| `audit_drop` | Dropped Audit Events — blank here is good news |

Which event types a collective emits varies by AppGate version and configuration. A blank panel is
usually this, not a fault.

**3. Are timestamps landing correctly?**

```
index=<your index> | eval real=strftime(strptime('log.timestamp', "%Y-%m-%dT%H:%M:%S.%3NZ"), "%F %T")
| table _time, real, date, timestamp
```

`_time` and `real` must agree. These events carry **two** timestamps: a syslog-style `date`
(`"Aug  4 19:07:28"` — no year, no timezone) that appears first in the raw string, and an
unambiguous ISO-8601 UTC `timestamp`. Splunk's auto-detection tends to take `date`, and with no
timezone the events can land hours off, shifting every time-based panel.

The symptom on the dashboard: *every* appliance showing as silent for 60+ minutes on Appliance
Liveness. One appliance silent is a real finding; all of them is this.

---

## Using the dashboard

**Every table is click-to-filter.** Clicking a row — an appliance, an entitlement, a user — scopes
the tab to it. Filter boxes accept wildcards (`*svc*`, an address prefix). **Reset a filter by
typing `*` back into its box.** Panels refresh every 5 minutes.

**Tab 1 — Health Overview.** Appliance liveness and event flow, entitlements assigned versus
actually used, resource access outcomes, authentication health, and the appliance SSH attack
surface.

**Tab 2 — Investigate User.** One identity in full: org unit, identity provider, authentication
type, risk claims, device, client version, country, plus footprint counts and first/last seen. The
claims panels discover whatever attributes your IdP emits rather than assuming a schema; rows with
no value are omitted. Clicking an identity here also loads it as User A on Tab 3.

**Tab 3 — Compare Users.** A cohort matrix scoring every identity on the same measures, a
per-user privilege gap, and four head-to-head panels comparing two identities' policies,
entitlements, and resource reach. With both user boxes left at `*` the head-to-head panels seed
themselves with the two busiest identities in the window, so the tab is populated on arrival
without configuring any names.

---

## What this app cannot show

Worth knowing before you go looking for it.

- **Resource utilization.** CPU, disk, memory, and NIC throughput exist only in the Controller
  API's appliance stats endpoint, not in the audit stream. What replaces them is liveness: an
  appliance that stops emitting events is the failure signal, and it surfaces faster than a status
  poll because it needs no cooperation from the appliance.
- **A complete policy or entitlement catalog.** Policy and entitlement names come from
  authorization events, so you see what was *exercised* in the selected time range. Entitlements
  nobody was granted, and disabled entitlements, do not appear.
- **A live concurrent-session count.** Session and tunnel events give you a rate, not a gauge.
- **Response actions.** This app is telemetry only. It cannot quarantine, revoke, or modify access.

---

## License

Proprietary. Copyright (c) 2026 Appgate, Inc. All rights reserved. See `LICENSE` in this package.

## Support

For questions about this app, contact Appgate at https://www.appgate.com
