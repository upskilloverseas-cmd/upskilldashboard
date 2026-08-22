---
name: upskill-dashboard
description: Render a live operations dashboard for UpSkill Overseas — used whenever Bansal says "run the dashboard", "show me the dashboard", "give me an overview", or asks for today's snapshot across leads, tasks, calls, marketing spend, and active B2B applications. Always pull fresh data from Zoho CRM and Windsor.ai on every run rather than reusing a stale summary from earlier in the conversation — the whole point is a same-moment snapshot. Trigger proactively even if Bansal just says "run this" in a context that's clearly asking for the overall business overview rather than a single-campaign lead audit (use upskill-lead-audit for that instead).
---

# UpSkill Overseas — Operations Dashboard

A repeatable, on-demand workflow that pulls live data from Zoho CRM, Windsor.ai, and Gmail, combines it with a cached B2B-portal application status, and renders a single HTML dashboard as a Claude Artifact. No separate app, hosting, or login — this workflow runs inside the conversation each time Bansal asks for it.

## When to use this

- Bansal says "run the dashboard", "show me the dashboard", or "give me the overview"
- Asked for today's snapshot: leads today, tasks due today, calls made today, who's doing what (Harsh/Krisha)
- Asked how marketing/ads are performing (spend, cost per lead)
- Asked about active B2B application status

Not for a single-campaign deep audit (Australia batch, Dubai batch, etc.) — that's `upskill-lead-audit`.

## Workflow — do all of these steps on every run

### 1. Zoho CRM — leads, tasks, calls, deals

Use `Zoho CRM:executeCOQLQuery` for each of the following. Field names below are confirmed against this org's actual module schema (via `getFields`) — do not guess alternate casings.

**Leads created/modified today**, bucketed by source:
```sql
SELECT id, First_Name, Last_Name, Phone, Lead_Source, Lead_Status, Owner, Created_Time, Modified_Time
FROM Leads
WHERE Created_Time >= '<today 00:00, org timezone>'
ORDER BY Created_Time desc
```
(Note the `Next_Call_date` field from the lead-audit skill is lowercase "d" if you need to reference it here too.)

**Tasks due today**:
```sql
SELECT id, Subject, Due_Date, Status, Priority, Owner, What_Id, Who_Id
FROM Tasks
WHERE Due_Date = '<today, org timezone>' AND Status != 'Completed'
```

**Calls logged today**:
```sql
SELECT id, Subject, Call_Start_Time, Call_Duration, Call_Type, Call_Purpose, Call_Result, Owner, What_Id, Who_Id
FROM Calls
WHERE Call_Start_Time >= '<today 00:00, org timezone>'
ORDER BY Call_Start_Time desc
```

> **Verified quirk — `Owner` doesn't split by person, but Lead `Tag` does.** This org has exactly one Zoho CRM user (the shared admin login, `Upskill Overseas` / `aayushhishah@upskilloverseas.in` — confirmed via `getUsers`), so every Lead/Task/Call/Deal shows the same single `Owner` and COQL returns `Owner.name` as `null` regardless — don't use Owner for a per-person split. **Tasks and Calls have no Tag data either** (confirmed empty on both modules). But **Leads carry a real `Tag` field**, and `getTags` on the Leads module confirms live, populated tags per counselor: `HarshSIR` (294 leads), `krisha` (169 leads), plus `Hemangi` (402) and `Bhoomi` (371) — two more names that show up tagged but haven't been confirmed as team members to include; ask Bansal once rather than assuming. COQL filters on it directly: `WHERE Tag = 'HarshSIR'` / `WHERE Tag = 'krisha'` (lowercase, exactly as stored) works and can be combined with a `Modified_Time` window, e.g. leads *touched* today by each person:
```sql
SELECT COUNT(id) FROM Leads WHERE Modified_Time >= '<today 00:00, org timezone>' AND Tag = 'HarshSIR'
SELECT COUNT(id) FROM Leads WHERE Modified_Time >= '<today 00:00, org timezone>' AND Tag = 'krisha'
```
This is a real Harsh/Krisha split, sourced from Zoho — use it as the base for the ops-by-person panel. It only covers Leads, though, so it's "leads each person touched today," not tasks or calls done — that part still isn't in Zoho (see the next step).

**Deal/pipeline snapshot** (COQL has no `NOT IN` — use two `!=` clauses):
```sql
SELECT id, Deal_Name, Stage, Amount, Lead_Source, Campaign_Source, Owner, Closing_Date
FROM Deals
WHERE Stage != 'Closed Won' AND Stage != 'Closed Lost'
```
Aggregate into stage counts for a funnel view.

### 2. Windsor.ai — marketing/ad performance

Connected accounts confirmed via `get_connectors`:
- `facebook` — account id `5297469533670258` ("Upskill Overseas Education")
- `google_ads` — account id `128-750-7158` ("Upskill Overseas Education | Best Student Visa Consultant...")

Call `get_data` for both connectors, last 7 and last 30 days (`date_preset: "last_7dT"` / `"last_30dT"` — the trailing `T` includes today). Verified field IDs:
- `facebook`: `date, spend, clicks, impressions, actions_leadgen_grouped, actions_lead, cost_per_action_type_leadgen_grouped`
- `google_ads`: `date, spend, clicks, impressions, conversions`

**Verified quirk — lead tracking is currently near-zero on both platforms.** As of this writing, `actions_leadgen_grouped`/`actions_lead` return 0 every day on the Facebook account (spend is real, ~₹400-900/day, but no lead-form actions are being tracked — the campaign may be running as engagement/messaging rather than lead-gen, based on the raw `actions` field showing messaging/engagement events instead), and Google Ads `conversions` also returns 0 every day despite real spend/clicks. Don't silently compute cost-per-lead as spend/0 — show spend, clicks, and cost-per-click as the reliable numbers, and surface leads/CPL as "0 tracked" rather than hiding the panel or dividing by zero. If this changes on a future run (leads start showing non-zero), CPL = spend / leads per platform per window.

### 3. Gmail — inbox summary

Call `list_labels` for the unread/total counts on `INBOX` and `IMPORTANT` (verified working — confirmed a large backlog: thousands of unread messages, so lead with unread *change since last run* if tracked, not just the raw total, since the raw total will always look alarming). Use `search_threads` for anything more targeted (e.g. BitTRACK correspondence, per `b2b-portal-check.md`). `list_labels` errored transiently once during setup — retry once on failure; if it still fails, note "Gmail summary unavailable this run" rather than blocking the whole dashboard.

### 4. Active B2B-portal applications — read the cache, don't log in live

The B2B partner portals (KC Overseas, Crizac, SI/StudyIn, Leverage Edu, BitTRACK) have no API and no MCP connector, and reading them requires the "Claude in Chrome" browser-control tools (click by coordinate, screenshot, page text) that only exist in a **desktop Claude Code session** — this cloud/on-demand session does not have them, and there's no way to run that login flow unattended on a schedule here. So this piece is intentionally **manual**: Bansal runs the check himself from a desktop Claude Code session every couple of days, following `b2b-portal-check.md` in this same skill folder, and that session overwrites `data/b2b-applications.json` with the new snapshot.

On a dashboard run in *this* environment:
- Read that file.
- Show its `last_checked` timestamp prominently next to the panel — if it's stale (no update in the last ~4 days), flag that visibly rather than presenting it as current, and remind Bansal it's due for a manual check.
- Do **not** attempt a browser login here — this session doesn't have the tools for it.

### 5. Ops data outside Zoho — tasks/calls still need Bansal's input

The Lead `Tag` split above covers leads touched per person, but Tasks and Calls have no Tag data and Owner is useless (see above), so there's still no Zoho-native way to attribute today's tasks/calls, or any "processing" work, to Harsh vs Krisha specifically. Ask Bansal conversationally at the start of the run ("what's the task/call split between Harsh and Krisha today, and anything on the processing side outside Zoho I should fold in?") rather than presenting the Zoho-only tasks/calls totals as already broken down by person. If he has nothing to add, show the Zoho numbers as org-wide totals rather than guessing a split.

### 6. Render the dashboard

Follow the `dataviz` and `artifact-design` skills for the visual pass. Publish as a single HTML Artifact with:
- Stat tiles: leads today, tasks due today, calls today, unread emails
- Marketing panel: spend, cost-per-lead, leads by platform (Facebook vs Google Ads), 7-day and 30-day toggle
- Lead-source breakdown for today's leads
- Ops-by-owner panel: Harsh vs Krisha leads touched today from the Lead `Tag` split (step 1), plus tasks/calls split from what Bansal gives conversationally in step 5 — org-wide Zoho task/call totals shown alongside as context
- Deal pipeline funnel (stage counts)
- Active Applications panel: cached B2B status + last-checked timestamp

Redeploy to the same Artifact URL on every run (don't create a new artifact each time) once the first one exists for this workflow — track the URL in the conversation so repeat runs update it in place.

If the request is conversational rather than asking to "see" a dashboard, a plain-text summary of the same numbers is fine — read the room.

## Manual B2B portal check (separate from this workflow, desktop session only)

See `b2b-portal-check.md` in this folder for the per-portal login/navigation steps. Run it from a desktop Claude Code session with the Chrome extension attached, roughly every 2 days. It writes its results to `data/b2b-applications.json`, which this dashboard workflow then reads. Credentials are never stored in this repo — `b2b-portal-check.md` points to where they live instead.
