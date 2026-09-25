# Session Transcript - AppGate Splunk Demo

Claude Code session `81b1370d-47b4-4152-8b87-ca330d9e82f1`. User prompts and assistant replies in full; tool calls listed by name (outputs omitted for readability). Generated from the raw session log.

---

## Turn 1 - User

do you remeber this project

**Claude:**

I don't have any memory of it yet — let me check the project.

> _tools: Read_

> _tools: PowerShell_

**Claude:**

No memory file exists yet, so I have no stored recollection. Let me read what's here.

> _tools: Read_

**Claude:**

No — I have no stored memory of prior sessions on this project (the memory index is empty). But the README is thorough, so here's what's here:

**AppGate SDP → Splunk Dashboard Studio demo** — a port of the AppGate ZTNA telemetry dashboard from the CrowdStrike NG-SIEM version, re-authored in SPL. Collective Health scope, telemetry only, no response actions. Fed entirely by the AppGate LogForwarder into `index=appgate`; the Controller API poller is retained in `poller/config.yaml` but unused.

Two artifacts actually leave the directory:
- `TA-appgate-demo.tar.gz` → uploaded to Splunk (contains the one object that matters, the `appgate_lf` macro, exported `system`-wide)
- `dashboards/appgate_collective_health.json` (26 panels) and `dashboards/appgate_user_investigation.json` (22 panels) → pasted into Studio's `</>` editor

Design decisions the README is emphatic about:
- **All field mapping lives in the `appgate_lf` macro**, not `props.conf` — sourcetype-keyed aliases caused silent zero-row panels
- **Text inputs, not dropdowns** — Studio's `input.dropdown` chain-expression schema fails silently; static `items` is the fallback if you want real dropdowns on stage
- **Liveness-as-health** — an appliance that stops emitting is the failure signal, since CPU/disk/RAM only exist in the Controller API
- Panels for event types AppGate 6.7 doesn't actually emit (`audit_drop`, `remedy*`, `appliance_status_changed`, `rule_monitor_health_change`) were **removed**, not left blank

The demo narrative hangs on **Assigned vs Actually Used** (least-privilege gap) and **Appliance SSH Attack Surface** (geolocated brute force against appliance IPs).

Some sections have visible edit history — corrections where earlier claims about geo enrichment and `site_names` were wrong, and the verification checklist is misnumbered (1, then 4–7).

What do you want to do with it?

## Turn 2 - User

let me open you in a different folder

**Claude:**

Sounds good. Two things worth knowing before you do:

- This session's context won't carry over — a new folder means a fresh start, and memory is scoped per-project directory anyway.
- If the new folder is the `appgate-crowdstrike-integration-main` repo (or wherever else), just point me at it and say what you want; I'll read what's there.

## Turn 3 - User

that is the new folder you can yead it

> _tools: PowerShell_

> _tools: PowerShell_

**Claude:**

Found it: `C:\Users\connor.witt\Downloads\appgate-crowdstrike-integration-main`. Let me survey it.

> _tools: PowerShell_

> _tools: PowerShell_

> _tools: Read_

**Claude:**

Read it. Note the actual root is nested one level: `...\appgate-crowdstrike-integration-main\appgate-crowdstrike-integration-main\`.

**Appgate ZTNA Insights for CrowdStrike** — a full bidirectional integration, much bigger in scope than the Splunk demo. Where the Splunk port is telemetry-only, this one closes the loop: Falcon detection → quarantine → user's network access flips mid-session with no re-auth.

**Four components:**
1. **CrowdStrike data plane** — 2 HEC connectors → 2 NG-SIEM repos, 2 parsers. `identity_repo` fed by the LogForwarder (same `cz-logd` stream your Splunk `appgate_lf` macro consumes); `snapshot_repo` fed by the poller.
2. **Foundry app** — standalone Insights page, detection-panel extension, one `identity-enrichment` function, and 5 Fusion SOAR actions (Quarantine/Unquarantine/Blacklist/Unblacklist/Revoke).
3. **Controller customization** — Flask "Fusion receiver" on HTTPS :9443, plus the poller.
4. **Access policy** — 3 Condition scripts that `httpGet` the receiver at evaluation time to read live quarantine state.

**Deployment model:** fill one `env-profile.yaml` → `preflight.py` (blocks on failed connectivity) → `render_profile.py` (fans values into manifest/JSX/conditions, with a leak-gate for leftover placeholders) → build/deploy → verify. Ships only `@@PLACEHOLDER@@` templates.

**Two things stand out:**

- The README contains an explicit **ask to Appgate product/dev**: there's no native way to push a dynamic external signal into a Condition evaluation, so the integration works around it with gateway-side `httpGet`. The workaround **fails open silently** on any reachability error — quarantine appears to work but enforces nothing. They hit real AWS same-VPC hairpin and gateway DNS failures. That's a candid writeup of a sharp edge in a security control.
- The `identity-profile` page's `IdentityPanel.jsx` is **693 KB**, and the poller's `fusion_receiver.py` is **170 KB** — both are almost certainly generated or heavily inlined rather than hand-maintained.

The Splunk demo's README references `02-source/appgate-api-poller/` here as the optional upgrade path for appliance CPU/disk/RAM gauges.

What do you want to do — port more of this to Splunk, review something specific, or work on the integration itself?

## Turn 4 - User

I am working on making the splunk dashboard based on the crowdstrike demo

**Claude:**

Let me establish where things actually stand — what the CrowdStrike dashboards contain vs. what you've already ported.

> _tools: PowerShell_

> _tools: PowerShell_

> _tools: PowerShell_

> _tools: Read_

**Claude:**

Here's the state of play, and there's one thing you'll want to know first.

## The user-investigation dashboard has two files, and the README points at the wrong one

| File | Panels | Modified |
|---|---|---|
| `appgate_user_investigation_full.json` | 33 layout items | 2:41 PM |
| `appgate_user_investigation.json` | **2** (header + Identity Summary) | 3:24 PM ← newest |
| `appgate_collective_health.json` | 32 | 2:39 PM |

The README's Dashboard 2 section tells you to import `appgate_user_investigation.json` and describes 22 panels — but that file is a stub whose own description says *"panels will be added incrementally."* Importing per the runbook gives you a nearly empty dashboard. The real one is `_full.json`.

The stub is also the **most recently modified file**, so it looks like it was a deliberate experiment — it uses the static dropdown the README recommends as the reliable route, populated with real identities (`chuck-yeager`, `sally-ride`, `benjamin-franklin`, `loadgen-scale-probe_7tae3`). Three problems in it if you meant to keep going that direction:

- The loadgen dropdown entry has a mangled value — label `loadgen-scale-probe_7tae3` but value `loadgen-scale_loadgen-scale_loadgen-scale-probe_7tae3` (tripled prefix), so selecting it matches nothing.
- Its query uses `ag_ou`, but the `appgate_lf` macro defines `ag_user_ou`. Also `session`, `idp`, `auth_type`, `device_os`, `client_version` — none are mapped in the macro.
- It hardcodes `clearance`, `risk_score`, `home_base`, `ia_training` — exactly the federal-demo claims the README argues against hardcoding, and which `_full.json` solves with `transpose`.

## The gap vs. the CrowdStrike dashboard

The CS `AppgateZTNADashboard.yaml` has 4 sections. Your Collective Health covers **Remote Access Activity**, **Collective Health**, and **Security & Compliance** (the section that's broken in the original). What's unported is essentially all of **Clients Fleet** — Resource Utilization by Appliance, Health by Site/Appliance, Registered Devices, Licensed Users, Clients by Version/Platform, macOS/Windows Devices, Live Session Count, Sites, Appliances — and every one of those is poller-fed, not LogForwarder-fed. So the gap isn't SPL you haven't written; it's the poller you haven't stood up.

Genuinely portable and currently missing: **Claims at a Glance** and **Top Users by Resource Access**.

Where do you want to go?

> _tools: AskUserQuestion_

**Claude:**

What would you like to clarify?

## Turn 5 - User

I need to give you the log data in splunk first give me a search to get the log data from multiple daemoens. I want to work on the overall health dashboard first to remove the blank pannels labeled top policies, entitlements, conditions satisfied. Additionally, I would like a seperate readme file that explains in depth where the data is being collected for each panel in the collective health dashboard.

**Claude:**

Good — that's the right order. Let me look at exactly how those three panels are written before I hand you a search.

> _tools: PowerShell_

**Claude:**

Got it. Here's the plan: searches first (I need your output to fix the three panels), and I'll write the provenance README now since it doesn't depend on your data.

## The likely cause, so you know what we're testing for

All three blank panels — plus the "Assigned" half of **Assigned vs Actually Used** — depend on **JSON array** fields:

| Panel | Event | Field |
|---|---|---|
| Top Policies | `authorization_succeeded` | `policy_names` |
| Entitlements by Assignment | `authorization_succeeded` | `entitlement_names`, `site_names` |
| Conditions Satisfied | `entitlement_token_evaluated` | `successful_condition_names` |

Splunk's JSON extraction names array fields with a **`{}` suffix** — `log.policy_names{}`, not `log.policy_names`. The macro's `rename log.* AS *` then yields `policy_names{}`, so `search policy_names=*` matches nothing and the panel goes blank with no error. That's the same silent-blank failure mode the README already documents for sourcetype aliases. The other candidate is simply that these event types aren't emitted.

## Run these three

**1 — Daemon × event type census** (this is the "multiple daemons" search):

```
index=appgate
| eval daemon=coalesce(daemon, "(none)"), event_type=coalesce('log.event_type', "(none)")
| stats count as events, dc(host) as hosts, earliest(_time) as first, latest(_time) as last by daemon, event_type
| eval first=strftime(first, "%F %T"), last=strftime(last, "%F %T")
| sort - events
```

**2 — Real field names on the three events** (run once per event type, swap the value):

```
index=appgate log.event_type=authorization_succeeded
| head 1
| fields log.*
| transpose 1 | rename column as Field, "row 1" as Value
| sort Field
```

Repeat with `entitlement_token_evaluated`. If either returns nothing, that event type isn't in your data and we take a different route entirely.

**3 — The brace check** (one line, settles the hypothesis outright):

```
index=appgate log.event_type=authorization_succeeded
| stats count as events,
        count('log.policy_names') as plain_policy,
        count('log.policy_names{}') as braced_policy,
        count('log.entitlement_names') as plain_ent,
        count('log.entitlement_names{}') as braced_ent,
        count('log.site_names') as plain_site,
        count('log.site_names{}') as braced_site
```

If the `braced_*` columns are populated and the `plain_*` ones are zero, the fix is a one-line macro change and I can apply it immediately.

If you'd rather just paste me one raw event, that works too and tells me more: `index=appgate log.event_type=authorization_succeeded | head 1 | table _raw`.

Now let me write the provenance README while you run those.

> _tools: PowerShell_

> _tools: Read_

**Claude:**

Before the README — I need to correct something I told you earlier. I based my critique of the user-investigation stub on the main README's Option C macro text, and **that text is stale**. The actual `macros.conf` defines `ag_ou` (not `ag_user_ou`), plus `session`, `idp`, `auth_type`, `device_os`, `client_version`, `clearance`, `risk_score`, `home_base`, and `ia_training` — every field I flagged is real. The stub's query is fine; only the mangled `loadgen-scale` dropdown value is an actual bug.

Worse, Option C in the README also tells you to define `appliance=coalesce('log.hostname', hostname, ...)` — and `macros.conf` lines 22-28 exist specifically to warn that this silently fills the appliance column with **client device names**. Anyone installing by hand gets the bug the macro was fixed to remove. I'll note that in the new file.

> _tools: Write_

**Claude:**

Written to `COLLECTIVE-HEALTH-PANEL-REFERENCE.md` — all 25 data panels, each with source daemon → event type, raw JSON field → macro alias → query field, which filter tokens apply, and what blank means for that specific panel. Plus a field-mapping table for the whole macro, a reverse event-type → panel index, and the four known issues.

**Three things I found while writing it:**

1. **The blank panels are localized.** Every panel that reads an array field is blank; every panel that doesn't, works. Notably **Entitlements by Actual Use** works and **Entitlements by Assignment** doesn't — same section, same dashboard, and the difference is precisely that one reads `entitlement` (scalar) and the other reads `entitlement_names` (array). That's strong evidence the fault is array handling, not the macro, index, or feed.

2. **Assigned vs Actually Used is worse than blank.** Its subsearch reads `entitlement` (working) while the outer search reads `entitlement_names` (broken), so it renders with Used populated and Assigned at zero — which reads as "every entitlement is fully used," the exact opposite of the least-privilege gap it's meant to show. Since the README calls this the headline demo panel, don't show it until we've fixed the others.

3. **The `macros.conf` comment at lines 35-42 asserts the arrays are "usable directly after the rename."** Given the blanks, that claim is unverified — it may be aspirational rather than tested.

**If the `{}` hypothesis holds, the fix is one macro edit, not four query edits** — add `rename "policy_names{}" AS policy_names` (and the other three) inside `appgate_lf`, and all four panels light up with no dashboard changes. That also means the fix carries to any future panel using those fields.

Run search #3 (the brace check) and paste me the output — that single line settles it. If it comes back with `events` at 0, the event types aren't there and we'll need a different approach for that section, which I'd rather know before writing SPL against it.

## Turn 6 - User

{"version":2,"date":"Aug  6 15:00:31","timestamp":"2026-08-06T15:00:31.568Z","hostname":"controller.sdpfederal.com","daemon":"cz-controllerd","log":{"claims_token_id":"1cb90a43-9ad6-431d-a43b-61989e2b03c5","client_ip":"50.18.241.71","collective_id":"9ce0adf3-7803-48d2-ae81-5ed0f3ad2294","collective_name":"SDP Federal","device_claims":{"Notes_Running":false,"PPModeEnabled":false,"clientIPs":["10.10.10.34"],"clientType":"headless","clientVersion":"6.7.0-18925-release","currentUser":"appgate","deviceScriptsDisabled":false,"dnsSuffixes":["us-west-1.compute.internal"],"gfe":false,"isAlwaysOn":false,"isFirewallEnabled":false,"isMultiUser":false,"isUserAdmin":false,"macAddresses":["06640C8202B3"],"os":{"family":"linux","hostname":"ip-10-10-10-34","name":"Amazon Linux 2023.11.20260514","parent":"fedora","platform":"x64","type":"desktop","version":"2023"},"profileHostnames":["sdp-controllers.sdpfederal.com"],"profileName":"fed-loadgen-local","spaKey":"fed-loadgen-local","stig":false},"distinguished_name":"CN=8678fa7aee31e181637f2a2b3b54c09b,CN=dwight-eisenhower,OU=local","distinguished_name_device_id":"8678fa7aee31e181637f2a2b3b54c09b","distinguished_name_ou":"local","distinguished_name_user":"dwight-eisenhower","dns_settings":[{"domain":"demo.sdpfederal.com","servers":["10.10.0.2"]}],"entitlement_names":["01 - Lunch Menu - Hanscom AFB","Acquisitions Application Access","Acquisitions Application Server","Acquisitions Database","Acquisitions Management Console","Acquisitions Network","Enterprise DNS","Enterprise DNS Resolver","Enterprise Email","Enterprise NTP","Enterprise SSO","Enterprise Wiki","Human Resources Application Access","Human Resources Application Server","Human Resources Database","Human Resources Management Console","Human Resources Network"],"entitlement_token_ids":["853329db-e7d9-45e0-bb3f-fbab953556b6"],"event_type":"authorization_succeeded","execution_ms":79,"geoip":{"ip":"50.18.241.71","time_zone":"America/Los_Angeles","continent_code":"NA","city_name":"San Jose","country_name":"United States","country_code2":"US","dma_code":807,"country_code3":"US","region_code":"CA","region_name":"California","postal_code":"95141","location":{"lon":-121.8916,"lat":37.3388},"latitude":37.3388,"longitude":-121.8916,"cordinates":[-121.8916,37.3388]},"id":"cfa47a8f-0006-4517-a6a9-409170f65fe9","policy_names":["Enterprise DNS Access","Enterprise DNS Resolution","Enterprise Services Access","Human Resources Access Policy","Test Policy"],"pool_v4_ip":"192.168.101.55","scripted_user_claims":{"clearance":"top-secret","deviceTrust":"trusted","geoLocation":"OCONUS","homeBase":"Pentagon","iaTrainingCompleted":true,"mfaProvided":true,"riskScore":53,"role":"supervisor","synthetic":true},"site_names":["AWS"],"system_claims":{"client_src_ip":"50.18.241.71","geo_ip":{"continent_code":"NA","country_code":"US","latitude":37.3388,"longitude":-121.8916,"state_code":"CA"}},"tamper_proofing":false,"timestamp":"2026-08-06T15:00:31.567831495Z","version":24}}

**Claude:**

That's exactly what I needed — and it settles the important half. The event type exists and carries all three arrays:

- `log.policy_names` — 5 policies
- `log.entitlement_names` — **17 entitlements** for this one user
- `log.site_names` — `["AWS"]`

So "not emitted" is ruled out. The fields are there and the panels are blank anyway, which leaves the naming hypothesis. Let me make the macro robust to it.

> _tools: Edit_

> _tools: Edit_

> _tools: Edit_

**Claude:**

Macro updated — stage 3 `rename "*{}" AS "*"` strips Splunk's array suffix. It's a no-op if your extraction doesn't add the suffix, so it's safe regardless of which way the check goes. Now updating the reference doc.

> _tools: Edit_

> _tools: Edit_

> _tools: Edit_

**Claude:**

Done. Here's where things stand.

## The fix

`appgate_lf` now has a stage 3:

```
index=appgate
| eval appliance=hostname, device_hostname=coalesce(...)
| rename log.* AS *
| rename "*{}" AS "*"        ← new
| eval ag_user=..., ...
```

Your event confirms the arrays are present and populated, so "not emitted" is out. That leaves the field-naming problem: Splunk extracts a JSON array as `log.policy_names{}`, the flatten turns it into `policy_names{}`, and `search policy_names=*` matches nothing — blank, no error. One rename strips the suffix from every array field at once, so **Top Policies, Entitlements by Assignment, and the Assigned column of the drift panel** should all populate with no dashboard edits. It's a no-op if the suffix isn't there, which is why I applied it before you'd confirmed.

Rebuild and re-upload with **Upgrade app** ticked:

```powershell
Set-Location C:\Users\connor.witt\appgate-splunk-demo
Remove-Item TA-appgate-demo.tar.gz -Force -ErrorAction SilentlyContinue
tar -czf TA-appgate-demo.tar.gz TA-appgate-demo
```

Then verify:

```
`appgate_lf` | search event_type=authorization_succeeded | head 1 | table policy_names, entitlement_names, site_names
```

## One of the three panels isn't fixed yet

**Conditions Satisfied** reads `entitlement_token_evaluated` / `successful_condition_names`. Your sample is `authorization_succeeded`, which doesn't carry that field — so I still don't know whether `cz-sessiond` emits that event type at all. If it doesn't, no macro change can help and the panel needs a different source or removal. Run the census to settle it:

```
index=appgate
| eval daemon=coalesce(daemon, "(none)"), event_type=coalesce('log.event_type', "(none)")
| stats count as events, earliest(_time) as first, latest(_time) as last by daemon, event_type
| eval first=strftime(first, "%F %T"), last=strftime(last, "%F %T")
| sort - events
```

## Three things your event revealed

**`site_names` is `["AWS"]` — a single site.** So `dc(site_names)` in Entitlements by Assignment returns 1 on every row, permanently. It's a column of identical numbers. Worth switching to `values(site_names)` to show the name instead — say the word and I'll change it.

**Your drift panel is going to be striking.** This one user is assigned **17 entitlements** under 5 policies, including "01 - Lunch Menu - Hanscom AFB" alongside Acquisitions Database and HR Management Console. Once Assigned populates, the least-privilege gap should be large and very legible on stage.

**`dwight-eisenhower` is synthetic.** `clientType: headless`, `profileName: fed-loadgen-local`, and `scripted_user_claims.synthetic: true` — with `clearance: top-secret`, `homeBase: Pentagon`, `riskScore: 53`. It's load-gen wearing a person's name. Fine to demo, but don't present those claims as live IdP output. It also inverts the main README's advice to "pick a real interactive user" for the claims panels — here the synthetic identities are the ones carrying full claims. And it isn't in the stub dashboard's hardcoded dropdown, which is the static-list staleness problem showing up for real.

All of this is written into `COLLECTIVE-HEALTH-PANEL-REFERENCE.md` (issues 1, 1a, 5, 6, 7).

## Turn 7 - User

lets remove those panels from the dashboard

**Claude:**

Removing them. Let me get the exact layout geometry first so the reflow is clean.

> _tools: PowerShell_

> _tools: Read_

**Claude:**

Now the edits — removing the three panels and reflowing the section.

> _tools: Edit_

> _tools: Edit_

> _tools: Edit_

> _tools: Edit_

**Claude:**

Now the layout reflow — the Policies row collapses to two panels and everything below shifts up 310px.

> _tools: Edit_

**Claude:**

Now the canvas height and the two text blocks that reference the removed panels.

> _tools: Edit_

> _tools: Edit_

> _tools: Edit_

**Claude:**

Now validating the JSON and confirming no dangling references:

> _tools: PowerShell_

**Claude:**

Layout validates clean — no overlaps, no orphaned references, consistent 10px gutters, canvas trimmed to fit. Now updating the reference doc to match.

> _tools: Grep_

> _tools: Grep_

> _tools: Read_

**Claude:**

Now replacing the whole Policies section with the trimmed two-panel version.

> _tools: Edit_

> _tools: Edit_

> _tools: Edit_

> _tools: Edit_

**Claude:**

Now renumbering the remaining panels (16–25 → 13–22):

> _tools: PowerShell_

**Claude:**

The write mangled the file's UTF-8 encoding and the renumber produced empty numbers. Fixing both:

> _tools: PowerShell_

**Claude:**

The reversal lost the em-dashes and arrows (CP1252 vs Latin-1 mismatch), so transformation can't recover them. Rewriting the file cleanly instead:

> _tools: Write_

> _tools: PowerShell_

> _tools: Write_

> _tools: PowerShell_

> _tools: PowerShell_

> _tools: Grep_

**Claude:**

Done. Tarball rebuilt (4,920 bytes) with the updated macro.

## What changed

**`dashboards/appgate_collective_health.json`** — three panels removed, section reflowed:

| Removed | Its dataSource |
|---|---|
| Top Policies | `ds_top_policies` |
| Entitlements — by Assignment | `ds_ent_assigned` |
| Conditions Satisfied | `ds_conditions` |

The section went from 5 panels across 2 rows to **2 panels on 1 row** — Entitlements by Actual Use and Assigned vs Actually Used, each 710px wide. Everything below shifted up 310px and the canvas trimmed 2950 → 2640. Validated: JSON parses, no overlaps, no orphaned visualizations or dataSources, consistent 10px gutters.

Also renamed the section header from "Policies, Entitlements & Conditions" to **"Entitlements"**, since it no longer covers the other two, and updated the dashboard description.

**I kept Assigned vs Actually Used.** It reads the same `entitlement_names` array as the panel I removed, so it's the one remaining panel exposed to the array-naming bug — but it's the README's designated headline panel, and it *contains* the assignment data as its `Assigned` column. Removing it would have taken the least-privilege story off the dashboard entirely. Say the word if you want it gone too.

**`COLLECTIVE-HEALTH-PANEL-REFERENCE.md`** — rewritten. 22 panels renumbered 1–22, and a new **Removed panels** section preserves all three queries verbatim with their panel config and the specific condition under which each is worth restoring. Conditions Satisfied is flagged separately since its source event `entitlement_token_evaluated` is still unconfirmed — if that event type doesn't exist, restoring it can't work regardless of the macro fix.

## One caveat on the panel you're keeping

Until you upload the rebuilt add-on and confirm, **Assigned vs Actually Used may still render misleadingly** — Used populated, Assigned all zeros, which reads as "every entitlement is fully used." That's the inverse of the truth. After the upload:

```
`appgate_lf` | search event_type=authorization_succeeded | head 1 | table policy_names, entitlement_names, site_names
```

Populated multivalue columns means the fix took and the Assigned column will be real.

The main `README.md` still describes the old 26-panel inventory and the "Policies, Entitlements & Conditions" section. Want me to bring it in line?

## Turn 8 - User

{
    "title": "AppGate Collective Health",
    "description": "Zero-trust collective telemetry from the AppGate LogForwarder audit stream: appliance liveness, entitlement assignment vs actual use, resource access, authentication health, and appliance attack surface. All queries verified against live event shapes.",
    "inputs": {
        "input_appliance": {
            "options": {
                "defaultValue": "*",
                "token": "appliance"
            },
            "title": "Appliance  (* = all, wildcards ok)",
            "type": "input.text"
        },
        "input_entitlement": {
            "options": {
                "defaultValue": "*",
                "token": "entitlement"
            },
            "title": "Entitlement  (* = all, wildcards ok)",
            "type": "input.text"
        },
        "input_time": {
            "options": {
                "defaultValue": "-24h,now",
                "token": "global_time"
            },
            "title": "Time Range",
            "type": "input.timerange"
        },
        "input_user": {
            "options": {
                "defaultValue": "*",
                "token": "user"
            },
            "title": "User  (* = all, wildcards ok)",
            "type": "input.text"
        }
    },
    "defaults": {
        "dataSources": {
            "ds.search": {
                "options": {
                    "queryParameters": {
                        "earliest": "$global_time.earliest$",
                        "latest": "$global_time.latest$"
                    }
                }
            }
        }
    },
    "visualizations": {
        "sec_access": {
            "options": {
                "markdown": "## Resource Access"
            },
            "type": "splunk.markdown"
        },
        "sec_appliance": {
            "options": {
                "markdown": "## Appliance Health\nHealth is inferred from the audit stream: an appliance that stops emitting events is the primary failure signal. **Silent (min)** is the gap since its last event of any kind — amber past 15 minutes, red past 60."
            },
            "type": "splunk.markdown"
        },
        "sec_audit": {
            "options": {
                "markdown": "## Admin Audit"
            },
            "type": "splunk.markdown"
        },
        "sec_glance": {
            "options": {
                "markdown": "## At a Glance"
            },
            "type": "splunk.markdown"
        },
        "sec_policy": {
            "options": {
                "markdown": "## Entitlements\n**Actual Use** comes from `rule_name` on **ip_access** (the gateway) — what actually carried traffic. **Assigned vs Actually Used** joins that against `entitlement_names` on **authorization_succeeded** (the controller) to expose the least-privilege gap, straight from the audit stream with no API poller."
            },
            "type": "splunk.markdown"
        },
        "sec_security": {
            "options": {
                "markdown": "## Authentication & Attack Surface"
            },
            "type": "splunk.markdown"
        },
        "viz_active_users": {
            "dataSources": {
                "primary": "ds_active_users"
            },
            "options": {
                "legendDisplay": "off",
                "yAxisTitleText": "Distinct Users"
            },
            "title": "Active Users Over Time",
            "type": "splunk.line"
        },
        "viz_admin_activity": {
            "dataSources": {
                "primary": "ds_admin_activity"
            },
            "description": "Admin authorizations and object reads/changes, with controller execution time. Object Type shows which entity kinds your collective emits.",
            "options": {
                "adminEventColorConfig": [
                    {
                        "match": "entity_deleted",
                        "value": "#D41F1F"
                    },
                    {
                        "match": "admin_authorization_failed",
                        "value": "#D41F1F"
                    },
                    {
                        "match": "entity_updated",
                        "value": "#CBA700"
                    },
                    {
                        "match": "entity_created",
                        "value": "#118832"
                    }
                ],
                "columnFormat": {
                    "Event": {
                        "textColor": "> table | seriesByName('Event') | matchValue(adminEventColorConfig)"
                    }
                },
                "count": 50
            },
            "title": "Admin & Configuration Activity",
            "type": "splunk.table"
        },
        "viz_appliance_liveness": {
            "dataSources": {
                "primary": "ds_appliance_liveness"
            },
            "description": "Appliance identity comes from the event's top-level hostname -- gateways report an IP, controllers an FQDN. A silent appliance is the clearest health signal the audit stream offers.",
            "options": {
                "columnFormat": {
                    "Silent (min)": {
                        "textColor": "> table | seriesByName('Silent (min)') | rangeValue(silentColorConfig)"
                    }
                },
                "count": 25,
                "silentColorConfig": [
                    {
                        "to": 15,
                        "value": "#118832"
                    },
                    {
                        "from": 15,
                        "to": 60,
                        "value": "#CBA700"
                    },
                    {
                        "from": 60,
                        "value": "#D41F1F"
                    }
                ]
            },
            "title": "Appliance Liveness",
            "type": "splunk.table"
        },
        "viz_auth_failures": {
            "dataSources": {
                "primary": "ds_auth_failures"
            },
            "description": "Includes the client device hostname and geolocated client IP.",
            "options": {
                "count": 50
            },
            "title": "Authentication Failures",
            "type": "splunk.table"
        },
        "viz_auth_outcomes": {
            "dataSources": {
                "primary": "ds_auth_outcomes"
            },
            "options": {
                "fieldColors": {
                    "authentication_failed": "#D41F1F",
                    "authentication_succeeded": "#118832"
                },
                "legendDisplay": "bottom",
                "stackMode": "stacked"
            },
            "title": "Authentication Outcomes",
            "type": "splunk.column"
        },
        "viz_controller_messages": {
            "dataSources": {
                "primary": "ds_controller_messages"
            },
            "description": "Controller-posted warnings and notices -- deprecated API use, version issues, operational advisories.",
            "options": {
                "columnFormat": {
                    "Level": {
                        "textColor": "> table | seriesByName('Level') | matchValue(levelColorConfig)"
                    }
                },
                "count": 25,
                "levelColorConfig": [
                    {
                        "match": "Error",
                        "value": "#D41F1F"
                    },
                    {
                        "match": "Warning",
                        "value": "#CBA700"
                    },
                    {
                        "match": "Information",
                        "value": "#118832"
                    }
                ]
            },
            "title": "Controller Messages",
            "type": "splunk.table"
        },
        "viz_drift": {
            "dataSources": {
                "primary": "ds_drift"
            },
            "description": "The least-privilege gap: entitlements granted to users who never exercised them. High values are over-provisioning. This is the panel to build the demo narrative around.",
            "options": {
                "columnFormat": {
                    "Assigned, Never Used": {
                        "textColor": "> table | seriesByName('Assigned, Never Used') | rangeValue(gapColorConfig)"
                    }
                },
                "count": 30,
                "gapColorConfig": [
                    {
                        "to": 1,
                        "value": "#118832"
                    },
                    {
                        "from": 1,
                        "to": 5,
                        "value": "#CBA700"
                    },
                    {
                        "from": 5,
                        "value": "#D41F1F"
                    }
                ]
            },
            "title": "Assigned vs Actually Used",
            "type": "splunk.table"
        },
        "viz_ent_outcomes": {
            "dataSources": {
                "primary": "ds_ent_outcomes"
            },
            "description": "A high Deny % usually means a misconfigured entitlement or a condition failing unexpectedly.",
            "options": {
                "columnFormat": {
                    "Deny %": {
                        "textColor": "> table | seriesByName('Deny %') | rangeValue(denyColorConfig)"
                    }
                },
                "count": 25,
                "denyColorConfig": [
                    {
                        "to": 5,
                        "value": "#118832"
                    },
                    {
                        "from": 5,
                        "to": 20,
                        "value": "#CBA700"
                    },
                    {
                        "from": 20,
                        "value": "#D41F1F"
                    }
                ]
            },
            "title": "Entitlement Access Outcomes",
            "type": "splunk.table"
        },
        "viz_ent_used": {
            "dataSources": {
                "primary": "ds_ent_used"
            },
            "description": "Ranked by traffic that actually traversed the entitlement.",
            "options": {
                "count": 25,
                "dataOverlayMode": "heatmap"
            },
            "title": "Entitlements — by Actual Use",
            "type": "splunk.table"
        },
        "viz_event_volume": {
            "dataSources": {
                "primary": "ds_event_volume"
            },
            "description": "A line dropping to zero is an appliance that stopped reporting.",
            "options": {
                "legendDisplay": "right",
                "yAxisTitleText": "Events"
            },
            "title": "Event Volume by Appliance",
            "type": "splunk.line"
        },
        "viz_header": {
            "options": {
                "fontSize": "default",
                "markdown": "# AppGate SDP — Collective Health\nCollective telemetry from the **LogForwarder** audit stream. Filters accept wildcards; `*` means all. Value lists live in the tables themselves — Appliance Liveness for appliances, the entitlement tables for entitlements."
            },
            "type": "splunk.markdown"
        },
        "viz_map": {
            "dataSources": {
                "primary": "ds_map"
            },
            "description": "Geolocated client source IPs. Populated because clients connect from routable addresses.",
            "options": {},
            "title": "Client Activity Map",
            "type": "splunk.map"
        },
        "viz_session_churn": {
            "dataSources": {
                "primary": "ds_session_churn"
            },
            "description": "session_created / _reconnected / _removed plus tunnel_connected / _established / _closed.",
            "options": {
                "legendDisplay": "bottom",
                "stackMode": "stacked"
            },
            "title": "Session & Tunnel Churn",
            "type": "splunk.column"
        },
        "viz_ssh_attacks": {
            "dataSources": {
                "primary": "ds_ssh_attacks"
            },
            "description": "Failed SSH logins against appliances, grouped by origin country and target. Unexpected countries against a root account are internet background brute-force -- a strong talking point for why the appliance should not expose SSH.",
            "options": {
                "count": 25,
                "dataOverlayMode": "heatmap"
            },
            "title": "Appliance SSH Attack Surface",
            "type": "splunk.table"
        },
        "viz_tile_appliances": {
            "dataSources": {
                "primary": "ds_tile_appliances"
            },
            "options": {
                "majorValue": "> primary | seriesByName('v') | lastPoint()",
                "shouldAbbreviateValues": false
            },
            "title": "Appliances Reporting",
            "type": "splunk.singlevalue"
        },
        "viz_tile_authfail": {
            "dataSources": {
                "primary": "ds_tile_authfail"
            },
            "options": {
                "authFailConfig": [
                    {
                        "to": 1,
                        "value": "#118832"
                    },
                    {
                        "from": 1,
                        "to": 25,
                        "value": "#CBA700"
                    },
                    {
                        "from": 25,
                        "value": "#D41F1F"
                    }
                ],
                "majorColor": "> majorValue | rangeValue(authFailConfig)",
                "majorValue": "> primary | seriesByName('v') | lastPoint()",
                "shouldAbbreviateValues": false
            },
            "title": "Auth Failures",
            "type": "splunk.singlevalue"
        },
        "viz_tile_denials": {
            "dataSources": {
                "primary": "ds_tile_denials"
            },
            "options": {
                "majorColor": "> majorValue | rangeValue(zeroGoodConfig)",
                "majorValue": "> primary | seriesByName('v') | lastPoint()",
                "shouldAbbreviateValues": false,
                "zeroGoodConfig": [
                    {
                        "to": 1,
                        "value": "#118832"
                    },
                    {
                        "from": 1,
                        "value": "#D41F1F"
                    }
                ]
            },
            "title": "Access Denials",
            "type": "splunk.singlevalue"
        },
        "viz_tile_sessions": {
            "dataSources": {
                "primary": "ds_tile_sessions"
            },
            "description": "session_created plus session_reconnected -- a rate, not a concurrent count.",
            "options": {
                "majorValue": "> primary | seriesByName('v') | lastPoint()",
                "shouldAbbreviateValues": false,
                "sparklineValues": "> primary | seriesByName('v')",
                "trendDisplay": "absolute",
                "trendValue": "> primary | seriesByName('v') | delta(-1)"
            },
            "title": "Sessions Established",
            "type": "splunk.singlevalue"
        },
        "viz_tile_ssh": {
            "dataSources": {
                "primary": "ds_tile_ssh"
            },
            "description": "Failed SSH logins against appliances -- internet-facing brute-force attempts.",
            "options": {
                "majorColor": "> majorValue | rangeValue(sshConfig)",
                "majorValue": "> primary | seriesByName('v') | lastPoint()",
                "shouldAbbreviateValues": false,
                "sshConfig": [
                    {
                        "to": 1,
                        "value": "#118832"
                    },
                    {
                        "from": 1,
                        "value": "#D41F1F"
                    }
                ]
            },
            "title": "Appliance SSH Failures",
            "type": "splunk.singlevalue"
        },
        "viz_tile_users": {
            "dataSources": {
                "primary": "ds_tile_users"
            },
            "options": {
                "majorValue": "> primary | seriesByName('v') | lastPoint()",
                "shouldAbbreviateValues": false,
                "unit": "users",
                "unitPosition": "after"
            },
            "title": "Active Users",
            "type": "splunk.singlevalue"
        },
        "viz_top_resources": {
            "dataSources": {
                "primary": "ds_top_resources"
            },
            "options": {
                "columnFormat": {
                    "Port": {
                        "textColor": "> table | seriesByName('Port') | matchValue(portColorConfig)"
                    }
                },
                "count": 25,
                "portColorConfig": [
                    {
                        "match": "445",
                        "value": "#D41F1F"
                    },
                    {
                        "match": "3389",
                        "value": "#D41F1F"
                    },
                    {
                        "match": "22",
                        "value": "#CBA700"
                    }
                ]
            },
            "title": "Top Resources Accessed",
            "type": "splunk.table"
        },
        "viz_traffic": {
            "dataSources": {
                "primary": "ds_traffic"
            },
            "options": {
                "fieldColors": {
                    "allow": "#118832",
                    "drop": "#CBA700",
                    "reject": "#D41F1F"
                },
                "legendDisplay": "bottom",
                "stackMode": "stacked"
            },
            "title": "Traffic — Allowed vs Dropped vs Rejected",
            "type": "splunk.area"
        },
        "viz_users_per_appliance": {
            "dataSources": {
                "primary": "ds_users_per_appliance"
            },
            "options": {
                "legendDisplay": "off",
                "xAxisTitleText": "Distinct Users",
                "yAxisTitleText": "Appliance"
            },
            "title": "Users per Appliance",
            "type": "splunk.bar"
        }
    },
    "dataSources": {
        "ds_active_users": {
            "name": "Active users over time",
            "options": {
                "query": "`appgate_lf` | search ag_user=* | timechart span=1h dc(ag_user) as Users"
            },
            "type": "ds.search"
        },
        "ds_admin_activity": {
            "name": "Admin and configuration activity",
            "options": {
                "query": "`appgate_lf` | search event_type IN (entities_listed, on_boarded_devices_listed, entity_created, entity_updated, entity_deleted, admin_authorization_succeeded, admin_authorization_failed) ag_user=\"$user$\" | sort - _time | table _time, event_type, entity_type, entity_name, ag_user, ag_client_ip, exec_ms, appliance | rename event_type as Event, entity_type as \"Object Type\", entity_name as Object, ag_user as Admin, ag_client_ip as \"Client IP\", exec_ms as \"ms\", appliance as Appliance"
            },
            "type": "ds.search"
        },
        "ds_appliance_liveness": {
            "name": "Appliance liveness",
            "options": {
                "query": "`appgate_lf` | search appliance=\"$appliance$\" | stats max(_time) as last_event, count as events, dc(daemon) as daemon_count, values(daemon) as subsystems, dc(ag_user) as users by appliance | eval silent_min=round((now()-last_event)/60, 1) | eval last_event=strftime(last_event, \"%F %T\") | sort - silent_min | rename appliance as Appliance, last_event as \"Last Event\", silent_min as \"Silent (min)\", events as Events, users as Users, daemon_count as Daemons, subsystems as Subsystems"
            },
            "type": "ds.search"
        },
        "ds_auth_failures": {
            "name": "Authentication failures",
            "options": {
                "query": "`appgate_lf` | search event_type=authentication_failed ag_user=\"$user$\" | sort - _time | table _time, ag_user, ag_reason, ag_client_ip, ag_country, device_hostname, appliance | rename ag_user as User, ag_reason as Reason, ag_client_ip as \"Client IP\", ag_country as Country, device_hostname as Device, appliance as Appliance"
            },
            "type": "ds.search"
        },
        "ds_auth_outcomes": {
            "name": "Authentication outcomes over time",
            "options": {
                "query": "`appgate_lf` | search event_type IN (authentication_succeeded, authentication_failed) | timechart span=1h count by event_type"
            },
            "type": "ds.search"
        },
        "ds_controller_messages": {
            "name": "Controller messages",
            "options": {
                "query": "`appgate_lf` | search event_type=admin_message_posted | sort - _time | table _time, msg_level, msg_category, msg_source, msg_text | rename msg_level as Level, msg_category as Category, msg_source as Source, msg_text as Message"
            },
            "type": "ds.search"
        },
        "ds_drift": {
            "name": "Assigned vs actually used",
            "options": {
                "query": "`appgate_lf` | search event_type=authorization_succeeded entitlement_names=* | mvexpand entitlement_names | stats dc(ag_user) as assigned by entitlement_names | rename entitlement_names as ent | append [ search `appgate_lf` | search event_type=ip_access | stats dc(ag_user) as used by entitlement | rename entitlement as ent ] | stats max(assigned) as assigned, max(used) as used by ent | fillnull value=0 assigned used | eval gap=if(assigned-used>0, assigned-used, 0) | sort - gap | rename ent as Entitlement, assigned as Assigned, used as Used, gap as \"Assigned, Never Used\""
            },
            "type": "ds.search"
        },
        "ds_ent_outcomes": {
            "name": "Entitlement access outcomes",
            "options": {
                "query": "`appgate_lf` | search event_type=ip_access entitlement=\"$entitlement$\" | stats count as total, count(eval(ag_action=\"allow\")) as allowed, count(eval(ag_action=\"drop\" OR ag_action=\"reject\")) as denied by entitlement | eval deny_pct=if(total>0, round(denied*100/total, 1), 0) | sort - denied | rename entitlement as Entitlement, total as Requests, allowed as Allowed, denied as Denied, deny_pct as \"Deny %\""
            },
            "type": "ds.search"
        },
        "ds_ent_used": {
            "name": "Entitlements by actual use",
            "options": {
                "query": "`appgate_lf` | search event_type=ip_access entitlement=\"$entitlement$\" | stats count as accesses, dc(ag_user) as users, dc(dest_ip) as resources by entitlement | sort - accesses | rename entitlement as Entitlement, accesses as Accesses, users as Users, resources as Resources"
            },
            "type": "ds.search"
        },
        "ds_event_volume": {
            "name": "Event volume by appliance",
            "options": {
                "query": "`appgate_lf` | search appliance=\"$appliance$\" | timechart span=1h count by appliance"
            },
            "type": "ds.search"
        },
        "ds_map": {
            "name": "Client activity map",
            "options": {
                "query": "`appgate_lf` | search ag_lat=* ag_user=\"$user$\" | geostats latfield=ag_lat longfield=ag_lon count"
            },
            "type": "ds.search"
        },
        "ds_session_churn": {
            "name": "Session and tunnel churn",
            "options": {
                "query": "`appgate_lf` | search event_type IN (session_created, session_reconnected, session_removed, tunnel_connected, tunnel_established, tunnel_closed) | timechart span=1h count by event_type"
            },
            "type": "ds.search"
        },
        "ds_ssh_attacks": {
            "name": "Appliance SSH access failures by origin",
            "options": {
                "query": "`appgate_lf` | search event_type=ssh_access_failed | stats count as attempts, dc(ag_client_ip) as source_ips, values(ag_user) as accounts, latest(_time) as last by ag_country, appliance | eval last=strftime(last, \"%F %T\") | sort - attempts | rename ag_country as Country, appliance as Appliance, attempts as Attempts, source_ips as \"Source IPs\", accounts as \"Targeted Accounts\", last as Latest"
            },
            "type": "ds.search"
        },
        "ds_tile_appliances": {
            "name": "Tile: appliances reporting",
            "options": {
                "query": "`appgate_lf` | search appliance=* | stats dc(appliance) as v"
            },
            "type": "ds.search"
        },
        "ds_tile_authfail": {
            "name": "Tile: authentication failures",
            "options": {
                "query": "`appgate_lf` | search event_type=authentication_failed | stats count as v"
            },
            "type": "ds.search"
        },
        "ds_tile_denials": {
            "name": "Tile: access denials",
            "options": {
                "query": "`appgate_lf` | search event_type=ip_access (ag_action=drop OR ag_action=reject) | stats count as v"
            },
            "type": "ds.search"
        },
        "ds_tile_sessions": {
            "name": "Tile: sessions established",
            "options": {
                "query": "`appgate_lf` | search event_type IN (session_created, session_reconnected) | timechart span=1h count as v"
            },
            "type": "ds.search"
        },
        "ds_tile_ssh": {
            "name": "Tile: appliance SSH access failures",
            "options": {
                "query": "`appgate_lf` | search event_type=ssh_access_failed | stats count as v"
            },
            "type": "ds.search"
        },
        "ds_tile_users": {
            "name": "Tile: active users",
            "options": {
                "query": "`appgate_lf` | search ag_user=* | stats dc(ag_user) as v"
            },
            "type": "ds.search"
        },
        "ds_top_resources": {
            "name": "Top resources accessed",
            "options": {
                "query": "`appgate_lf` | search event_type=ip_access entitlement=\"$entitlement$\" ag_user=\"$user$\" | eval dest_port=coalesce(dest_port, \"--\") | stats count as accesses, dc(ag_user) as users by dest_ip, dest_port, ag_protocol | sort - accesses | rename dest_ip as Destination, dest_port as Port, ag_protocol as Protocol, accesses as Accesses, users as Users"
            },
            "type": "ds.search"
        },
        "ds_traffic": {
            "name": "Traffic allowed vs denied",
            "options": {
                "query": "`appgate_lf` | search event_type=ip_access entitlement=\"$entitlement$\" | timechart span=1h count by ag_action"
            },
            "type": "ds.search"
        },
        "ds_users_per_appliance": {
            "name": "Users per appliance",
            "options": {
                "query": "`appgate_lf` | search ag_user=* appliance=\"$appliance$\" | stats dc(ag_user) as users by appliance | sort - users | rename appliance as Appliance, users as Users"
            },
            "type": "ds.search"
        }
    },
    "layout": {
        "globalInputs": [
            "input_time",
            "input_appliance",
            "input_user",
            "input_entitlement"
        ],
        "layoutDefinitions": {
            "layout_1": {
                "options": {
                    "display": "auto-scale",
                    "height": 2640,
                    "width": 1440
                },
                "structure": [
                    {
                        "item": "viz_header",
                        "position": {
                            "h": 70,
                            "w": 1440,
                            "x": 0,
                            "y": 0
                        },
                        "type": "block"
                    },
                    {
                        "item": "sec_glance",
                        "position": {
                            "h": 36,
                            "w": 1440,
                            "x": 0,
                            "y": 80
                        },
                        "type": "block"
                    },
                    {
                        "item": "viz_tile_users",
                        "position": {
                            "h": 140,
                            "w": 230,
                            "x": 0,
                            "y": 126
                        },
                        "type": "block"
                    },
                    {
                        "item": "viz_tile_appliances",
                        "position": {
                            "h": 140,
                            "w": 230,
                            "x": 240,
                            "y": 126
                        },
                        "type": "block"
                    },
                    {
                        "item": "viz_tile_sessions",
                        "position": {
                            "h": 140,
                            "w": 230,
                            "x": 480,
                            "y": 126
                        },
                        "type": "block"
                    },
                    {
                        "item": "viz_tile_denials",
                        "position": {
                            "h": 140,
                            "w": 230,
                            "x": 720,
                            "y": 126
                        },
                        "type": "block"
                    },
                    {
                        "item": "viz_tile_authfail",
                        "position": {
                            "h": 140,
                            "w": 230,
                            "x": 960,
                            "y": 126
                        },
                        "type": "block"
                    },
                    {
                        "item": "viz_tile_ssh",
                        "position": {
                            "h": 140,
                            "w": 230,
                            "x": 1200,
                            "y": 126
                        },
                        "type": "block"
                    },
                    {
                        "item": "sec_appliance",
                        "position": {
                            "h": 56,
                            "w": 1440,
                            "x": 0,
                            "y": 276
                        },
                        "type": "block"
                    },
                    {
                        "item": "viz_appliance_liveness",
                        "position": {
                            "h": 300,
                            "w": 950,
                            "x": 0,
                            "y": 342
                        },
                        "type": "block"
                    },
                    {
                        "item": "viz_users_per_appliance",
                        "position": {
                            "h": 300,
                            "w": 470,
                            "x": 960,
                            "y": 342
                        },
                        "type": "block"
                    },
                    {
                        "item": "viz_event_volume",
                        "position": {
                            "h": 280,
                            "w": 710,
                            "x": 0,
                            "y": 652
                        },
                        "type": "block"
                    },
                    {
                        "item": "viz_controller_messages",
                        "position": {
                            "h": 280,
                            "w": 710,
                            "x": 720,
                            "y": 652
                        },
                        "type": "block"
                    },
                    {
                        "item": "sec_policy",
                        "position": {
                            "h": 56,
                            "w": 1440,
                            "x": 0,
                            "y": 942
                        },
                        "type": "block"
                    },
                    {
                        "item": "viz_ent_used",
                        "position": {
                            "h": 300,
                            "w": 710,
                            "x": 0,
                            "y": 1008
                        },
                        "type": "block"
                    },
                    {
                        "item": "viz_drift",
                        "position": {
                            "h": 300,
                            "w": 710,
                            "x": 720,
                            "y": 1008
                        },
                        "type": "block"
                    },
                    {
                        "item": "sec_access",
                        "position": {
                            "h": 36,
                            "w": 1440,
                            "x": 0,
                            "y": 1318
                        },
                        "type": "block"
                    },
                    {
                        "item": "viz_traffic",
                        "position": {
                            "h": 280,
                            "w": 470,
                            "x": 0,
                            "y": 1364
                        },
                        "type": "block"
                    },
                    {
                        "item": "viz_ent_outcomes",
                        "position": {
                            "h": 280,
                            "w": 470,
                            "x": 480,
                            "y": 1364
                        },
                        "type": "block"
                    },
                    {
                        "item": "viz_top_resources",
                        "position": {
                            "h": 280,
                            "w": 470,
                            "x": 960,
                            "y": 1364
                        },
                        "type": "block"
                    },
                    {
                        "item": "sec_security",
                        "position": {
                            "h": 36,
                            "w": 1440,
                            "x": 0,
                            "y": 1654
                        },
                        "type": "block"
                    },
                    {
                        "item": "viz_auth_outcomes",
                        "position": {
                            "h": 280,
                            "w": 470,
                            "x": 0,
                            "y": 1700
                        },
                        "type": "block"
                    },
                    {
                        "item": "viz_auth_failures",
                        "position": {
                            "h": 280,
                            "w": 470,
                            "x": 480,
                            "y": 1700
                        },
                        "type": "block"
                    },
                    {
                        "item": "viz_ssh_attacks",
                        "position": {
                            "h": 280,
                            "w": 470,
                            "x": 960,
                            "y": 1700
                        },
                        "type": "block"
                    },
                    {
                        "item": "viz_session_churn",
                        "position": {
                            "h": 280,
                            "w": 470,
                            "x": 0,
                            "y": 1990
                        },
                        "type": "block"
                    },
                    {
                        "item": "viz_active_users",
                        "position": {
                            "h": 280,
                            "w": 470,
                            "x": 480,
                            "y": 1990
                        },
                        "type": "block"
                    },
                    {
                        "item": "viz_map",
                        "position": {
                            "h": 280,
                            "w": 470,
                            "x": 960,
                            "y": 1990
                        },
                        "type": "block"
                    },
                    {
                        "item": "sec_audit",
                        "position": {
                            "h": 36,
                            "w": 1440,
                            "x": 0,
                            "y": 2280
                        },
                        "type": "block"
                    },
                    {
                        "item": "viz_admin_activity",
                        "position": {
                            "h": 300,
                            "w": 1440,
                            "x": 0,
                            "y": 2326
                        },
                        "type": "block"
                    }
                ],
                "type": "absolute"
            },
            "layout_DiiilFMC": {
                "options": {
                    "width": 1440
                },
                "structure": [],
                "type": "grid"
            }
        },
        "tabs": {
            "items": [
                {
                    "label": "Health Overview",
                    "layoutId": "layout_1"
                },
                {
                    "label": "Investigate User",
                    "layoutId": "layout_DiiilFMC"
                }
            ]
        }
    }
} I added a tab that I want to put the investigate user dashboard in. Can u integrate this for me

**Claude:**

Splunk gave you back a **tabbed** layout — `layoutDefinitions` + `tabs.items`. That directly falsifies the README's claim that "Dashboard Studio has no tabs," which is why the two dashboards were split in the first place. Let me look at what needs merging.

> _tools: Read_

**Claude:**

Merging. There are **four ID collisions** between the two dashboards I need to resolve first — `viz_header`, `sec_access`, `viz_auth_outcomes`, and `ds_auth_outcomes`. The last one is the dangerous one: both dashboards define it with *different* queries (the collective version has no user filter), so a naive merge would silently make one tab wrong. I'm namespacing everything from the investigation dashboard with `ui_`.

> _tools: Write_

> _tools: PowerShell_

**Claude:**

Merged and validated. Paste `dashboards/appgate_collective_health.json` into the source editor.

**Validation:** 62 visualizations, 48 dataSources, 62 layout items across both tabs — no duplicate placements, no orphaned panels or dataSources, no overlaps on either tab, both within canvas bounds.

| Tab | Layout | Panels | Height |
|---|---|---|---|
| Health Overview | `layout_1` | 29 | 2640 |
| Investigate User | `layout_DiiilFMC` | 33 | 3100 |

## The collision that would have bitten you

`ds_auth_outcomes` existed in **both** dashboards with different queries — the collective version has no user filter, the per-user version has `ag_user="$user$"`. A straight merge would have kept one and silently applied it to both tabs, so either the Health tab would filter to a single user or the Investigate tab would show collective-wide auth for everyone. Same story for `viz_auth_outcomes`, `viz_header`, and `sec_access`.

Everything from the investigation dashboard is now prefixed `ui_`, so ownership is unambiguous and future merges won't collide.

## Layout change

Your `layout_DiiilFMC` was `type: "grid"` with an empty structure. I changed it to `absolute` to carry over the investigation dashboard's pixel positions intact — the 4-across claims row and 3-across footprint row don't survive a grid conversion.

## Filters

`globalInputs` are shared across tabs, and the token names already lined up — both dashboards used `user` and `global_time`. So **typing a username on the Health tab carries it to the Investigate tab**, which makes the drill-through work without the `eventHandlers` URL hack the README describes.

Appliance and Entitlement inputs still render on tab 2 but nothing there reads them (25 panels use `$user$`, zero use the other two). I noted that in the tab-2 header text so it doesn't read as a broken filter.

## Worth knowing

**The README's claim that "Dashboard Studio has no tabs" is wrong** — your own export proves it, and that claim is the stated reason the two dashboards were split in the first place. The split is no longer necessary.

Consequently `dashboards/appgate_user_investigation_full.json` and the 2-panel `appgate_user_investigation.json` stub are now both redundant. I left them in place rather than deleting.

Two things I did **not** change, flag rather than fix:

- **`ui_viz_conditions`** reads `entitlement_token_evaluated` — still unconfirmed in your collective. It's the same panel I removed from Health Overview. I kept it on the user tab since you only asked about the overview, but added a description noting a blank may mean the event type is absent. Same for `ui_viz_policies` and the two entitlement panels, which depend on the array fix.
- **The dashboard title** is still "AppGate Collective Health," which now undersells a two-tab dashboard. Renaming is your call since it changes the saved dashboard's name.

Want me to update the README and reference doc to reflect the tabbed structure and retract the no-tabs claim?

## Turn 9 - User

I wanted the file that is not the full version to be integrated

**Claude:**

Understood — you want the 2-panel stub, not `_full`. Rewriting tab 2 with just the header and Identity Summary.

> _tools: Write_

> _tools: PowerShell_

**Claude:**

Tab 2 now holds the stub instead of `_full`. Validated: 32 visualizations, 24 dataSources, no orphans, no overlaps.

| Tab | Panels | Height |
|---|---|---|
| Health Overview | 29 | 2640 |
| Investigate User | 3 | 840 |

## Two judgment calls you should know about

**I dropped the stub's static dropdown.** It defined `input_user` as a dropdown on token `user` — but that token is now a *shared global input* driving four panels on tab 1. Converting it to a dropdown would replace the Health tab's wildcard-capable User text box with a fixed 6-name list. On top of that, the list is already stale (no `dwight-eisenhower`) and carries the mangled `loadgen-scale_loadgen-scale_loadgen-scale-probe_7tae3` value. So the shared text input stayed.

**That left no way to discover usernames, so I added Users in Window.** It's the one panel I pulled from `_full` — without it you'd have a text box and nothing telling you what to type, which is exactly the gap the static dropdown was filling. It ignores the User filter so it always lists everyone, and you copy a name across. That's a third panel the stub didn't have; say so if you'd rather have the bare two.

Layout is the stub's original geometry — 80px header, then the two tables side by side at 710px each.

## If you'd rather have the real dropdown

It's viable, just needs a current list. Run:

```
`appgate_lf` | search ag_user=* | stats count by ag_user | sort ag_user
```

Then I'll swap `input_user` to `input.dropdown` with static `items` built from that output. The tradeoff is the Health tab loses wildcard filtering (`*loadgen*` stops working), and the list goes stale again the next time an identity appears — which is what just happened with `dwight-eisenhower`.

`appgate_user_investigation_full.json` is untouched on disk if you want to pull more panels in later.

## Turn 10 - User

how can isave this chat

**Claude:**

It's already saved — Claude Code writes every session to disk automatically. Let me show you where.

> _tools: PowerShell_

**Claude:**

This session is already on disk:

```
C:\Users\connor.witt\.claude\projects\C--users-connor-witt-appgate-splunk-demo\
  81b1370d-47b4-4152-8b87-ca330d9e82f1.jsonl   (1.38 MB, updating live)
```

Claude Code writes the full transcript automatically — every message and tool call. Nothing to do to preserve it.

**To pick it back up:** `/resume` in this project directory lists past sessions and lets you continue one with full context. `claude --resume` does the same from a fresh terminal. Note it's keyed to the working directory — if you start Claude Code in the CrowdStrike folder, this session won't appear in that list.

**For a readable copy:** `/export` writes the conversation out to a file or clipboard.

One thing worth saying: the transcript is a record of *how* we got here, but the actual deliverables are already committed to the project and don't depend on it —

- `dashboards/appgate_collective_health.json` — the two-tab dashboard
- `TA-appgate-demo/default/macros.conf` — the `rename "*{}" AS "*"` array fix
- `TA-appgate-demo.tar.gz` — rebuilt add-on
- `COLLECTIVE-HEALTH-PANEL-REFERENCE.md` — the provenance doc, including the removed panels' queries and the seven known issues

That last file is deliberately the durable version of this conversation's findings. If you want the reasoning preserved beyond the transcript, that's where it lives.

The project isn't a git repo, so if you want real history I'd suggest `git init` and a first commit — that would make the dashboard changes reviewable and revertible, which matters more than the chat log. Want me to set that up?

## Turn 11 - User

can we work on a tab that will allow us to compare users

**Claude:**

Good idea — that's the panel type the CrowdStrike original never had. Let me design it around the data we've confirmed.

> _tools: Edit_

> _tools: Edit_

> _tools: Edit_

> _tools: Edit_

> _tools: Edit_

> _tools: PowerShell_

**Claude:**

Third tab added and validated: 42 visualizations, 31 dataSources, no orphans, no duplicate placements, no overlaps on any tab.

| Tab | Panels | Height |
|---|---|---|
| Health Overview | 29 | 2640 |
| Investigate User | 3 | 840 |
| **Compare Users** | 10 | 1830 |

## Design — two layers, because "compare" means two different things

**Cohort (needs no input, useful the moment the tab loads):**

- **User Comparison Matrix** — every identity, same 13 measures: events, auth OK/fail, Fail %, requests, denials, Deny %, entitlements used, resources, client IPs, countries, appliances, last seen. Fail % and Deny % are colored, and both are *normalized* so a low-volume user failing everything ranks as loudly as a busy one.
- **Privilege Gap by User** — assigned vs used vs never-used per identity, sorted by widest gap. This is the per-user version of the drift panel; the top row is your most over-provisioned account.
- **Assigned vs Used — Top 15 Gaps** — the same thing as a grouped bar chart, amber Assigned against green Used.

**Head-to-head (A vs B):**

- **User A vs B** — one column per user, one row per measure. Uses `transpose 0 header_field=ag_user`, which pivots usernames into column headers so the two identities line up for direct reading. Covers risk claims, device posture (firewall, local admin, STIG), and activity totals.
- **Entitlement Differential** — every entitlement tagged `both` / `User A only` / `User B only`. This is the access-review question a per-user view can't answer.
- **Policy Differential** — the same one level up. A single policy difference usually explains a long entitlement difference, so read this one first.
- **Resource Reach Differential** — what each user *actually reached*, by destination and entitlement. Assignment says what they may do; this says what they did.

## Input handling

I added **one** input — `Compare With / User B` (token `user_b`). The existing **User** box doubles as User A.

That was deliberate: the `user` token is already shared across tabs, so a user you're investigating on tab 2 is already loaded as User A when you switch to Compare. Adding a dedicated `user_a` would have made five user-ish boxes and broken that carry-over.

**One limitation, stated in the tab header:** head-to-head panels use `eval` exact matching (`count(eval(ag_user="$user$"))`), so they're blank while either box is `*`. Wildcards don't apply there. The cohort panels above them have no such constraint, which is why they're at the top.

## Dependency

Privilege Gap, Entitlement Differential, and Policy Differential all read the `entitlement_names` / `policy_names` arrays — so **they depend on the `rename "*{}" AS "*"` macro fix**, which is still unconfirmed on your instance. The Comparison Matrix and Resource Reach Differential read scalars only and will work regardless. If the gap panels come back empty while the matrix populates, that's the array fix, not these queries.

The docs are now behind — `COLLECTIVE-HEALTH-PANEL-REFERENCE.md` describes a single-tab, 22-panel dashboard. Want me to bring it up to date with all three tabs?

## Turn 12 - User

I have to present this can you write a summary doc for me so I can present effectively

**Claude:**

Presenting changes what matters — you need the story, the failure modes, and honest answers to hard questions. Let me write that.

> _tools: Write_

**Claude:**

Written to `DEMO-BRIEF.md` — a presenter's brief, not another technical reference. It's structured so you can read the pre-flight the morning of and skim the rest.

**The most important part is the pre-flight**, and one item there is a genuine presentation risk:

> `appgate_lf` | search event_type=authorization_succeeded | head 1 | table policy_names, entitlement_names, site_names

If those columns come back empty, the macro fix hasn't loaded, and **Assigned vs Actually Used will render with Used populated and Assigned at zero**. That doesn't look broken — it reads as "every entitlement is fully used," which is the exact opposite of the point you're making, in front of an audience, on your headline panel. Run that search before you present. The brief has a recovery path if it fails mid-demo.

**What else is in there:**

- **The pitch** — one sentence, a 30-second version, and the differentiator if someone pushes (the least-privilege gap is normally a manual catalog-export cross-reference; here it's a live panel).
- **A three-act arc** matching the three tabs — collective, one person, two people against each other. Worth saying out loud up front; it makes the tabs feel designed rather than accumulated.
- **Tab-by-tab with actual language to use**, including the reframe that makes the whole approach cohere: *there are no CPU gauges and that's deliberate — an appliance that stops talking is the failure signal that matters, and it's detectable without the appliance's cooperation.*
- **Nine likely questions with honest answers** — including "why no CPU/memory," "is this real-time," "can it quarantine," and what to say about the `01 - Lunch Menu - Hanscom AFB` entitlement if someone spots it (it's a gift — broad grants accumulate trivial entitlements next to sensitive ones, which is exactly what the gap panel finds).
- **A "do not claim" section.** The one to internalize: `dwight-eisenhower` and the other person-named identities are load-generated — `clientType: headless`, and the claims block literally carries `synthetic: true`. The top-secret clearance and Pentagon home base are generated values. If anyone asks whether that's live IdP output, the answer is no.
- **Five stage-failure recoveries** with the line to say and where to pivot.

Two suggestions for the demo itself: **sort the User Comparison Matrix live** — turning a table into an outlier finder in real time is more persuasive than any static view. And on tab 3, read **Policy Differential before Entitlement Differential**, since one policy difference usually explains a long list of entitlement differences.

## Turn 13 - User

make me a doc that gives detailed comparison on what is possible in splunk versus cloudstrike demo

**Claude:**

Let me verify the CrowdStrike side properly rather than relying on the README's characterization of it.

> _tools: PowerShell_

> _tools: Read_

> _tools: Read_

> _tools: PowerShell_

**Claude:**

All three reference-build bugs confirmed independently. Let me check the poller and the second CrowdStrike dashboard.

> _tools: PowerShell_

> _tools: PowerShell_

> _tools: PowerShell_

**Claude:**

Everything verified against the actual bundle. Writing the comparison.

> _tools: Write_

**Claude:**

Written to `SPLUNK-VS-CROWDSTRIKE.md`. I verified every CrowdStrike claim against the shipped bundle — dashboard YAML, Foundry manifest, the api-integration spec, and the poller config — rather than trusting its README.

**The framing that makes this presentable:** these aren't competing builds, and neither is a subset of the other. But the comparison isn't one-directional either —

> CrowdStrike has **more data available** (it polls the Controller API for the full entitlement catalog) and does **less with it analytically**. There is no least-privilege gap panel, no per-entitlement deny rate, and no user-vs-user comparison anywhere in its dashboards.

So: CrowdStrike buys breadth and enforcement with real infrastructure. Splunk buys depth of analysis with none.

**What only CrowdStrike can do** — 5 verified SOAR actions (`/fusion/quarantine`, `/unquarantine`, `/blacklist`, `/unblacklist`, `/revoke`) that flip access mid-session with no re-auth; the panel extension that renders inside the NG-SIEM detection slide-out; the AID pivot to Falcon host records; ZTA scores; and 63 Foundry function handlers including policy/entitlement authoring with rollback. That last one is well beyond what its README advertises.

**What only Splunk does** — the least-privilege gap, Compare Users, normalized deny rate, liveness-as-health, SSH attack surface, and schema-agnostic claims. On that last point: CrowdStrike's Identity Profile hardcodes `user_claims.Branch`, `.Citizenship`, `.clearance` — its own comments admit those are example claims from one federal demo schema, so on any other tenant it renders empty rows.

**Three verified defects in the reference build.** The `Security & Compliance` section is declared with a title and an order but **no `widgetIds` key at all** — 31 widgets declared, 27 placed, so 4 never render: Admin Configuration Activity, Polled Endpoints, Access Denied — Failed Policy Conditions, and Ingest Rate. 13% of that dashboard is invisible. Two sections also both claim `order: 3`.

**One correction to your own README.** It says `Distinct Resources Today` used `groupBy(...) | count()` which "counts groups, not distinct resources." That's not quite right — counting groups *is* counting distinct pairs. The real issue is that it groups by `destination.ip, destination.port`, so one host exposing five ports counts as five resources. Ambiguous granularity, not a broken aggregation. I've noted the correction in the new doc; the README still has the original wording.

The doc closes with a **"which one for which job"** table and a one-liner for when someone asks why both exist.

## Turn 14 - User

i need to save this chat

**Claude:**

Last time I pointed you at commands. Let me just produce the file directly.

> _tools: PowerShell_

> _tools: PowerShell_

> _tools: PowerShell_

