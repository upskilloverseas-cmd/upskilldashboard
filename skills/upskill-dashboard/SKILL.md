---
name: upskill-dashboard
description: Render a live operations dashboard for UpSkill Overseas — used whenever Bansal says "run the dashboard", "show me the dashboard", "give me an overview", or asks for today's snapshot across leads, tasks, calls, marketing spend, and active B2B applications. Always pull fresh data from Zoho CRM and Windsor.ai on every run rather than reusing a stale summary from earlier in the conversation — the whole point is a same-moment snapshot. Trigger proactively even if Bansal just says "run this" in a context that's clearly asking for the overall business overview rather than a single-campaign lead audit (use upskill-lead-audit for that instead).
---

# UpSkill Overseas — Operations Dashboard

A repeatable, on-demand workflow that pulls live data from Zoho CRM, Windsor.ai, and Gmail, combines it with a cached B2B-portal application status, and renders a single HTML dashboard as a Claude Artifact. No separate app, hosting, or login — this workflow runs inside the conversation each time Bansal asks for it.

## When to use this

- Bansal says "run the dashboard", "show me the dashboard", or "give me the overview"
- Asked for today's snapshot: leads today, follow-ups due today, calls made today, who's doing what (Harsh/Krisha) — and the rolling 120-day view behind those numbers
- Asked how marketing/ads are performing (spend, cost per lead)
- Asked about active B2B application status

Not for a single-campaign deep audit (Australia batch, Dubai batch, etc.) — that's `upskill-lead-audit`.

## Workflow — do all of these steps on every run

### 1. Zoho CRM — leads, tasks, calls, deals

Use `Zoho CRM:executeCOQLQuery` for each of the following. Field names below are confirmed against this org's actual module schema (via `getFields`) — do not guess alternate casings. **The standard analysis window is a rolling 120 days** (Bansal wants this tracked "at any point of time" — i.e. whatever day the dashboard runs, look back 120 days from there), alongside same-day counts for the stat tiles.

**Leads — today's count and the 120-day window**, bucketed by source:
```sql
SELECT COUNT(id) FROM Leads WHERE Created_Time >= '<today 00:00, org timezone>'
SELECT id, First_Name, Last_Name, Phone, Lead_Source, Lead_Status, Owner, Created_Time, Modified_Time
FROM Leads
WHERE Created_Time >= '<120 days ago 00:00, org timezone>'
ORDER BY Created_Time desc
```
(Note the `Next_Call_date` field from the lead-audit skill is lowercase "d" if you need to reference it here too.)

**"Tasks due today" — verified quirk: the Zoho Tasks module is not what this business actually uses.** It has exactly one record in the entire org, ever (a single "Follow-up for Ielts" task from Jan 2025, still Not Started) — confirmed via `SELECT COUNT(id) FROM Tasks WHERE Status != 'Completed'` (=1) and a full dump. Don't query it for "today's tasks" — it will always read as ~0 and that's misleading, not accurate. What this team actually tracks as a to-do is a **Lead's `Next_Call_date`** (same field the lead-audit skill uses, lowercase "d"):
```sql
SELECT COUNT(id) FROM Leads WHERE Next_Call_date = '<today, org timezone>'
SELECT id, First_Name, Last_Name, Lead_Status, Next_Call_date, Tag FROM Leads WHERE Next_Call_date = '<today, org timezone>'
```
This is the real "tasks due today" — treat it as such in the dashboard, not the empty Tasks module. Also worth surfacing: leads whose `Next_Call_date` is already in the *past* (overdue follow-ups) — `SELECT COUNT(id) FROM Leads WHERE Next_Call_date < '<today>'` returns a raw count (384 as of this writing) that is **not yet filtered by status** — see the COQL quirk just below before narrowing it to open/active leads only, since a naive 3-condition filter will error.

> **Verified quirk — COQL via this tool caps out past 2 chained `AND` conditions.** Confirmed repeatedly: two conditions in a `WHERE` (any field combination) work fine; a third `AND`, regardless of which fields or operators, returns a `SYNTAX_ERROR`. So a filter like "Next_Call_date < today AND Lead_Status != 'Not Interested' AND Lead_Status != 'Lost Lead'" cannot be one query. Split it: run the 2-condition version (date + one status exclusion), or fetch the date-filtered set with `Lead_Status` in the select list and exclude closed statuses client-side after paginating through results — don't drop a status filter silently just to dodge the error.

**Calls — today's count and the 120-day count**:
```sql
SELECT COUNT(id) FROM Calls WHERE Call_Start_Time >= '<today 00:00, org timezone>'
SELECT COUNT(id) FROM Calls WHERE Call_Start_Time >= '<120 days ago 00:00, org timezone>'
```

> **Verified quirk — `Owner` doesn't split by person, but Lead `Tag` does.** This org has exactly one Zoho CRM user (the shared admin login, `Upskill Overseas` / `aayushhishah@upskilloverseas.in` — confirmed via `getUsers`), so every Lead/Task/Call/Deal shows the same single `Owner` and COQL returns `Owner.name` as `null` regardless — don't use Owner for a per-person split. **Tasks and Calls have no Tag data either** (confirmed empty on both modules). But **Leads carry a real `Tag` field**, and `getTags` on the Leads module confirms live, populated tags per counselor: `HarshSIR` and `krisha` (lowercase, exactly as stored). `getTags` also showed `Hemangi` and `Bhoomi` tags with a large associated-record count each — **confirmed by Bansal these two are no longer on the team, so exclude them from the ops-by-person panel entirely**, even though the historical tag data still exists on old leads. COQL filters directly on Tag and combines with a time window, e.g. leads *touched* today and over the last 120 days by each active person:
```sql
SELECT COUNT(id) FROM Leads WHERE Modified_Time >= '<today 00:00, org timezone>' AND Tag = 'HarshSIR'
SELECT COUNT(id) FROM Leads WHERE Modified_Time >= '<120 days ago 00:00, org timezone>' AND Tag = 'HarshSIR'
SELECT COUNT(id) FROM Leads WHERE Modified_Time >= '<today 00:00, org timezone>' AND Tag = 'krisha'
SELECT COUNT(id) FROM Leads WHERE Modified_Time >= '<120 days ago 00:00, org timezone>' AND Tag = 'krisha'
```
This is a real Harsh/Krisha split, sourced from Zoho — use it as the base for the ops-by-person panel, with the 120-day figure as the headline and today's count as context. It only covers Leads, though, so it's "leads each person touched," not tasks or calls done — that part still isn't in Zoho (see the next step).

**Deal/pipeline snapshot** (COQL has no `NOT IN` — use two `!=` clauses):
```sql
SELECT id, Deal_Name, Stage, Amount, Lead_Source, Campaign_Source, Owner, Closing_Date
FROM Deals
WHERE Stage != 'Closed Won' AND Stage != 'Closed Lost'
```
Aggregate into stage counts for a funnel view. Open deals accumulate rather than reset daily, so this one doesn't need a 120-day filter — it's already "everything currently open."

### 2. Windsor.ai — marketing/ad performance

Connected accounts confirmed via `get_connectors`:
- `facebook` — account id `5297469533670258` ("Upskill Overseas Education")
- `google_ads` — account id `128-750-7158` ("Upskill Overseas Education | Best Student Visa Consultant...")

Call `get_data` for both connectors with `date_preset: "last_120dT"` (the trailing `T` includes today) — this is the standard window, matching the 120-day rolling analysis period used everywhere else in this dashboard. Verified field IDs:
- `facebook`: `date, spend, clicks, impressions, actions_leadgen_grouped, actions_lead, cost_per_action_type_leadgen_grouped`
- `google_ads`: `date, spend, clicks, impressions, conversions`

**Verified quirk — lead tracking looks empty on short windows but isn't, over 120 days.** On a 7-day window both platforms show 0 tracked leads/conversions despite real spend — that's real, not a bug (see below), but don't conclude from it that tracking is broken. Over the actual 120-day window: Facebook `actions_lead` = 15 (still `actions_leadgen_grouped` = 0 — leads are being tracked as a generic lead action, not the dedicated lead-gen action, so use `actions_lead` as the Facebook lead figure), and Google Ads `conversions` ≈ 5.3 (Google reports fractional attributed conversions — round for display but don't be alarmed by the decimal). CPL = spend / leads over the 120-day window: Facebook ≈ ₹4,600/lead, Google Ads ≈ ₹11,800/lead as of this writing. Both are real leads being missed on any window shorter than ~120 days simply because conversions/lead-gen events are sparse and lagged in this account — which is itself a reason the 120-day default matters here, not just a stylistic choice.

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
- Stat tiles: leads today, follow-ups due today (Lead `Next_Call_date`, not the Tasks module — see quirk above), calls today, unread emails
- Marketing panel: spend, cost-per-lead, leads by platform (Facebook vs Google Ads) over the 120-day window
- Lead-source breakdown over the 120-day window (today's count shown as context, not the headline — see the short-window quirk above)
- Ops-by-owner panel: Harsh vs Krisha leads touched, 120-day figure as the headline with today's count as context, from the Lead `Tag` split (step 1) — plus whatever task/call split Bansal gives conversationally in step 5, since Zoho has no Tag data on those modules
- Deal pipeline funnel (stage counts)
- Active Applications panel: cached B2B status + last-checked timestamp

Redeploy to the same Artifact URL on every run (don't create a new artifact each time) once the first one exists for this workflow — track the URL in the conversation so repeat runs update it in place.

If the request is conversational rather than asking to "see" a dashboard, a plain-text summary of the same numbers is fine — read the room.

## Manual B2B portal check (separate from this workflow, desktop session only)

See `b2b-portal-check.md` in this folder for the per-portal login/navigation steps. Run it from a desktop Claude Code session with the Chrome extension attached, roughly every 2 days. It writes its results to `data/b2b-applications.json`, which this dashboard workflow then reads. Credentials are never stored in this repo — `b2b-portal-check.md` points to where they live instead.
