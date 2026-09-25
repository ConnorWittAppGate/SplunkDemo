# Demo Brief — AppGate ZTNA in Splunk

A presenter's guide to the three-tab **AppGate Collective Health** dashboard: what to say, what to
click, what you'll be asked, and what not to claim.

Technical detail lives in `COLLECTIVE-HEALTH-PANEL-REFERENCE.md`. This file is what you read the
morning of.

**Contents**

- [The pitch](#the-pitch)
- [Pre-flight — run this 30 minutes before](#pre-flight--run-this-30-minutes-before)
- [The narrative arc](#the-narrative-arc)
- [Tab 1 — Health Overview](#tab-1--health-overview)
- [Tab 2 — Investigate User](#tab-2--investigate-user)
- [Tab 3 — Compare Users](#tab-3--compare-users)
- [The three panels to dwell on](#the-three-panels-to-dwell-on)
- [Questions you will get](#questions-you-will-get)
- [Do not claim](#do-not-claim)
- [If something breaks on stage](#if-something-breaks-on-stage)

---

## The pitch

**One sentence:**

> Every panel here comes from the AppGate audit stream that your collective is already emitting —
> no API poller, no agent, no extra services. One log forwarder into Splunk and you can see who has
> access to what, who actually used it, and what's attacking the appliances.

**The 30-second version:**

> AppGate's LogForwarder ships a JSON audit stream. We land it in one Splunk index, define one search
> macro, and paste in one dashboard. From that alone you get appliance health, the least-privilege
> gap — entitlements people were granted and never touched — full per-user investigation, and
> side-by-side access comparison between any two identities. The install is an app upload and a copy-
> paste, and the whole thing is telemetry, so it's read-only against your collective.

**The differentiator, if someone pushes:** the least-privilege gap is normally an entitlement-catalog
export cross-referenced by hand against traffic logs. Here it's a panel, live, from data you already
have.

---

## Pre-flight — run this 30 minutes before

Not optional. Two of these have bitten this dashboard already.

### 1. Does the macro resolve, and did the array fix take?

```
`appgate_lf` | search event_type=authorization_succeeded
| head 1 | table policy_names, entitlement_names, site_names
```

- **Populated (multivalue):** good. Your least-privilege panels will have real numbers.
- **Empty:** the `rename "*{}" AS "*"` fix hasn't loaded. Re-upload `TA-appgate-demo.tar.gz`
  with **Upgrade app** ticked and restart. **Until this passes, do not present Assigned vs Actually
  Used** — see [If something breaks](#if-something-breaks-on-stage).
- **"macro cannot be found":** the add-on isn't installed or isn't shared globally.

### 2. What does the collective actually emit right now?

```
`appgate_lf` | stats count by daemon, event_type | sort - count
```

Expect `cz-controllerd`, `cz-vpnd`, `cz-sessiond`, `cz-appliance`. Any panel whose event type is
absent goes blank with no error — know before your audience does.

### 3. Is the clock right?

Open Tab 1 and look at **Appliance Liveness**. If *every* appliance shows 60+ minutes silent, that's
timestamp extraction, not dead appliances — these events carry a timezone-less syslog `date` that
Splunk sometimes grabs instead of the ISO-8601 `timestamp`. One appliance stale is real; all of them
stale is a clock problem, and every time-based panel is shifted.

### 4. See who the dashboard will pick for you

```
`appgate_lf` | search ag_user=* | stats count as events by ag_user | sort - events
```

**Nothing is configured with names any more.** Every filter is a wildcard text box defaulting to `*`,
and the two tabs that need a specific identity seed themselves: Identity Summary profiles the busiest
identity in the window, and Tab 3's head-to-head compares the top two. This search shows you, before
you open the dashboard, exactly who those are.

The one thing worth checking: are the **top two genuinely different enough** to make the differential
tables interesting? If they're two near-identical service accounts, plan to pin a better pair — type a
name into **User A**, or click one in the cohort tables, and leave **User B** at `*` to have the
busiest other identity fill in.

If you also want to sanity-check the appliance and entitlement values you'll be typing into filters:

```
`appgate_lf` | search appliance=* | stats count by appliance | sort appliance
`appgate_lf` | search event_type=ip_access | stats count by entitlement | sort entitlement
```

You don't have to — Appliance Liveness and Assigned vs Actually Used list all of them on screen, and
both are click-to-filter.

### 5. Set the time range

24h is the safe default — enough volume that nothing looks empty. Narrow to 1h only if you want to
show responsiveness, and check nothing goes blank first.

---

## The narrative arc

Three tabs, three questions, in the order a security team actually asks them.

| Tab | The question | The payoff |
|---|---|---|
| **Health Overview** | Is the collective healthy, and what's happening across it? | Liveness without polling; the privilege gap; the attack surface |
| **Investigate User** | Who is this one person and what did they do? | Identity, claims, device, footprint in one view |
| **Compare Users** | Why does this person have access that person doesn't? | The access-review answer, side by side |

**Say the arc out loud when you start.** "I'm going to go collective, then one person, then two people
against each other." It makes the tabs feel designed rather than accumulated.

---

## Tab 1 — Health Overview

### Open on the tiles

Six numbers. Don't narrate all six — call out two:

- **Appliances Reporting** — "these are appliances that emitted at least one event, not appliances
  that exist. An appliance that dies drops out of this number, which is the point."
- **Appliance SSH Failures** — "that's not AppGate access control. That's failed SSH against the
  appliance operating system. We'll come back to it."

### Appliance Health — lead with the reframe

This is where you say the thing that makes the whole approach make sense:

> There are no CPU or memory gauges here, and that's deliberate. Those numbers only exist in the
> Controller API. What we have instead is better for the failure case that actually matters: **an
> appliance that stops talking.** Liveness tracks the gap since each appliance's last event of any
> kind. A partitioned or wedged appliance shows up here faster than a status poll would, because it
> needs no cooperation from the appliance to detect.

Point at the **Silent (min)** column, amber past 15, red past 60. Then **Event Volume by Appliance**:
"same signal as a shape — a line dropping to the floor is an appliance that went quiet, and it shows
you when."

**Subsystems column** is a nice aside if you have time: a gateway running two daemons that drops to
one is partially degraded before it goes fully silent.

### Entitlements — this is the money section

Two panels. Set them up as controller vs gateway:

> **Actual Use** is read from the gateway — what genuinely carried traffic. **Assigned vs Actually
> Used** joins that against what the controller granted. Left side is permission, right side is
> behaviour, and the gap between them is over-provisioning.

Then the line to land:

> These are entitlements somebody was granted and never touched. No catalog export, no spreadsheet,
> no poller — this is the audit stream telling you where least privilege isn't being met.

### Resource Access — one panel worth pausing on

**Entitlement Access Outcomes.** Deny % per entitlement:

> A high deny rate is diagnosable. It's a misconfigured entitlement, a resource definition that's too
> narrow, or a condition failing in a way nobody noticed. And because it's a percentage, a
> low-traffic entitlement denying everything ranks as loudly as a busy one.

### Attack Surface — end tab 1 here

**Appliance SSH Attack Surface** is the most concrete panel on the dashboard. Geolocated brute-force
against appliance management interfaces, grouped by origin country, with the targeted account names
visible.

> These are real, unsolicited, from the public internet, right now. The account names are a dictionary
> list — root, admin, ubuntu. This is the argument against exposing appliance SSH, and it makes it
> better than a slide does.

---

## Tab 2 — Investigate User

Deliberately small — two panels. Don't apologize for that; frame it:

> This is the pivot, not a report. You find the identity, you get the summary, and if you need the
> full forensic timeline that's a deeper view.

### The flow

1. **Users in Window** lists every identity in the range. It ignores the User filter on purpose so
   it's always complete.
2. **Click the row.** Identity Summary loads that user immediately — no typing, no copy-paste.
3. **Identity Summary** transposes into field/value rows — org unit, IdP, auth type, risk claims,
   device, client version, country, plus footprint counts and first/last seen.

**Click, don't type.** Every table on this dashboard is click-to-filter. It reads as a tool rather than
a form, and it removes the one thing most likely to go wrong live — typing a username from memory.
The text boxes are still there if you prefer to pre-set a value, and they take wildcards.

**The design point worth making:** rows with no value are dropped. Nothing is hardcoded to a claim
schema, so this works against any tenant's IdP rather than only one that happens to emit the fields
we guessed.

**The click carries across tabs.** Whoever you click here is loaded as User A when you move to
Compare. Say that as you switch — it makes the three tabs feel like one tool.

---

## Tab 3 — Compare Users

Two layers. Lead with the cohort, because it needs no setup and it's immediately legible.

### Cohort — no input required

**User Comparison Matrix** — every identity scored on the same measures. Then do this:

> Sort by Deny %. Sort by Client IPs. Sort by Countries.

Sorting live is the demo. It turns a table into an outlier finder, and it's the fastest way to show
the audience they'd find something in their own data.

**Privilege Gap by User** and the **Assigned vs Used** bar chart are the per-user version of the tab-1
drift panel. The chart is the one that reads across a room — a long amber bar next to a short green
one is over-provisioning you can see without reading numbers.

### Head to head

**This half is already populated when you arrive** — with **User A** and **User B** left at `*` it
seeds itself with the two busiest identities in the window. These are dedicated tokens, separate from
the collective User filter, so it can never be blank because a wildcard was left in that box, and
nothing here is tied to a name that only exists in one environment.

To change the pair without typing: click a row in **User Comparison Matrix** to set User A, and a row
in **Privilege Gap by User** (or a bar in the chart next to it) to set User B. Pin one and leave the
other at `*` and it refills with the busiest other identity. **Who A and B currently are is shown by
the two column headers of the head-to-head table** — worth pointing at once, because the differential
tables beside it say `User A only` / `User B only` in that same order.

The demo move: sort the matrix by Deny %, click the worst offender to make them User A, then click your
most over-provisioned identity in Privilege Gap to make them User B. Two clicks and you're comparing
the two most interesting identities in the collective.

> **Policy Differential first.** A single policy difference usually explains a long list of
> entitlement differences — so read the cause before the symptoms.

Then **Entitlement Differential** (`both` / `User A only` / `User B only`), then **Resource Reach
Differential**.

The closing line for the whole demo:

> Assignment tells you what they're allowed to do. Reach tells you what they did. Having both, for
> two people, side by side, is the access review — and it came out of a log stream you're already
> producing.

---

## The three panels to dwell on

If you only have ten minutes, these three:

| Panel | Tab | Why |
|---|---|---|
| **Assigned vs Actually Used** | Health Overview | The least-privilege gap, from telemetry alone. The thing nobody expects to be free. |
| **Appliance SSH Attack Surface** | Health Overview | Concrete, external, happening now. Impossible to argue with. |
| **Entitlement / Policy Differential** | Compare Users | Answers a real access-review question that a per-user view structurally cannot. |

---

## Questions you will get

**"Why no CPU, memory, or disk?"**
> They don't exist in the audit stream — only in the Controller API's appliance stats endpoint. We
> deliberately built this feed-only so there's nothing to stand up. If you want resource gauges,
> site health, licensed users, and client version distribution, there's a poller that adds them, and
> the macros for it already ship in the add-on.

**"Is this real-time?"**
> It's as real-time as your LogForwarder and Splunk indexing. Liveness is measured in minutes since
> last event, so it's near-live, not streaming.

**"Can it take action — quarantine, revoke?"**
> Not this. This is telemetry, read-only. Response actions exist in the CrowdStrike integration,
> where a detection can flip a user's network access mid-session with no re-authentication. Different
> deliverable, same telemetry foundation.

**"How long to deploy?"**
> Upload an app, paste a dashboard. The prerequisite is the LogForwarder pointed at Splunk with JSON
> extraction on. Minutes, not a project.

**"What if our event types are different?"**
> That's why there's one diagnostic search that lists exactly what your collective emits. We already
> hit this — the reference build assumed event names that AppGate 6.7 doesn't use, and those panels
> were removed rather than left silently blank.

**"Does it work with a different identity provider?"**
> Yes. The claims panels discover whatever attributes your IdP emits rather than assuming a schema.
> That was a deliberate departure from the reference build, which hardcoded one customer's claim set.

**"Why is that user called Lunch Menu?"**
> Good eye. That's a real entitlement name in this collective — "01 - Lunch Menu - Hanscom AFB" sits
> in the same grant as the Acquisitions Database. It's a genuinely useful illustration: broad grants
> accumulate trivial entitlements alongside sensitive ones, and the gap panel is how you find them.

**"Can we see who's currently connected?"**
> Not a live concurrent count — that's a poller metric. What you see is session establishment and
> tunnel events, which is a rate rather than a gauge. Worth being precise about.

---

## Do not claim

Getting caught overstating costs more than the feature was worth.

- **Don't call the demo identities real users.** `dwight-eisenhower` and its peers are load-generated:
  `clientType: headless`, `profileName: fed-loadgen-local`, and the claims block literally carries
  `synthetic: true`. The clearance and risk-score values are generated. If someone asks whether that's
  live IdP output, say no — it's a load generator with a realistic claim shape.
- **Don't say "all appliance health."** Say liveness and event flow. Resource utilization is genuinely
  absent.
- **Don't promise policy names come from the catalog.** They come from authorization events, so you
  see policies that were exercised in the window — not a complete inventory. Entitlements nobody was
  granted, and disabled entitlements, are invisible here.
- **Don't present Conditions data.** That panel was removed; `entitlement_token_evaluated` is not
  confirmed present in this collective.
- **Don't over-read the Client Activity Map.** It mixes legitimate users with SSH attackers, since
  both carry geo. An unexpected country is more likely brute-force traffic than a travelling employee.

---

## If something breaks on stage

**Assigned column is all zeros while Used has numbers.**
The array fix didn't load. This is the one failure that actively misleads — it reads as "every
entitlement is fully used," the opposite of the truth. Don't explain it, move past it:
> "Let me show you the per-user version of this instead."
Go to Tab 3, **Privilege Gap by User**. If that's also zeroed, use **Entitlements by Actual Use** and
talk about the gap conceptually.

**A panel is blank.**
Blank means the event type isn't in the window — it isn't an error. Say so plainly and move:
> "Nothing in that category during this window, which in this case is good news."
Then widen the time range if it matters.

**Every appliance shows as silent.**
Clock problem, not an outage. Don't debug it live:
> "Timestamp extraction on this instance — the appliances are up."
Move to the entitlement section, which is time-range-relative and unaffected.

**Head-to-head panels are empty.**
Not the wildcard trap it used to be — at `*` / `*` these panels pick the two busiest identities in the
window for themselves, so empty means either no identities in the range at all (widen it) or you pinned
a name that has no events. Clear **User A** / **User B** back to `*` and they'll refill.

**Everything is empty.**
Check the time range first, then check whether a stray click left a filter set — clicking a table row
sets that filter. **Reset by typing `*` back into the box.** The User filter carries across all three
tabs, which is usually the feature and occasionally the trap.

**Clicking a row does nothing.**
Your Studio version rejected the `eventHandlers` key. Nothing is broken — every panel still renders,
and you can type the value into the box instead. Do that and don't mention it.

---

## One-line reminders

- Silence is the health signal, not a gauge.
- Assignment is the controller; use is the gateway; the gap is the finding.
- SSH failures are the appliance OS, not AppGate access control.
- Click table rows, don't type — every table is a filter.
- Reset a filter by typing `*` back into its box.
- The User filter follows you across all three tabs.
- Sorting the comparison matrix, then clicking the outlier, is the demo.
- Tab 3 arrives populated — you never have to fill a box on stage.
