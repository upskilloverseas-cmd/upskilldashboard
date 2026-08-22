---
name: upskill-dashboard
description: Render a live operations dashboard for UpSkill Overseas — used whenever Bansal says "run the dashboard", "show me the dashboard", "give me an overview", or asks for today's snapshot across leads, tasks, calls, marketing spend, and active B2B applications. Always pull fresh data from Zoho CRM and Windsor.ai on every run rather than reusing a stale summary from earlier in the conversation — the whole point is a same-moment snapshot. Trigger proactively even if Bansal just says "run this" in a context that's clearly asking for the overall business overview rather than a single-campaign lead audit (use upskill-lead-audit for that instead).
---

# UpSkill Overseas — Operations Dashboard

A repeatable, on-demand workflow that pulls live data from Zoho CRM, Windsor.ai, and Gmail, combines it with a cached B2B-portal application status, and renders a single HTML dashboard as a Claude Artifact. No separate app, hosting, or login — this workflow runs inside the conversation each time Bansal asks for it.

## When to use this

- Bansal says "run the dashboard", "show me the dashboard", or "give me the overview"
- Asked for today's snapshot: leads today, calls made today, Harsh/Krisha's open backlog to close — and the rolling 120-day view behind those numbers
- Asked how marketing/ads are performing (spend, cost per lead, whether ad-platform leads are actually reaching Zoho)
- Asked about active B2B application / Visitor Visa status

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

**"Tasks" — verified: this is not Zoho's Tasks module, and it's not "due today" either. It's each person's backlog of leads still to close.** The Zoho Tasks module has exactly one record in the entire org, ever (a single "Follow-up for Ielts" task from Jan 2025, still Not Started) — confirmed via `SELECT COUNT(id) FROM Tasks WHERE Status != 'Completed'` (=1). Bansal's own definition: "tasks" = the number of calls/leads Harsh and Krisha each still have to close — i.e. their open-lead backlog by `Tag`, not a due-today count. A lead counts as closed (no longer "to do") once its `Lead_Status` is one of `Not Interested`, `Lost Lead`, `Junk Lead`, or `Converted` (full picklist confirmed via `getFields`; `Not Qualified` is ambiguous — treat as still-open unless Bansal says otherwise, don't assume it's closed). Compute per person as total tagged minus closed-count, since a 3-condition `!=` filter isn't possible (see the COQL quirk below) — as of this writing:
```sql
SELECT COUNT(id) FROM Leads WHERE Tag = 'HarshSIR'                          -- 294 total
SELECT COUNT(id) FROM Leads WHERE Tag = 'HarshSIR' AND Lead_Status = 'Not Interested'  -- 90
SELECT COUNT(id) FROM Leads WHERE Tag = 'HarshSIR' AND Lead_Status = 'Lost Lead'       -- 127
SELECT COUNT(id) FROM Leads WHERE Tag = 'HarshSIR' AND Lead_Status = 'Junk Lead'       -- 0
SELECT COUNT(id) FROM Leads WHERE Tag = 'HarshSIR' AND Lead_Status = 'Converted'       -- 0
-- Harsh backlog = 294 - (90+127+0+0) = 77 open leads to close
```
Same pattern for `krisha` (169 total, 95 Not Interested, 25 Lost Lead, 0 Junk, 2 Converted → **47 open**). Re-run all of these live each time rather than trusting these numbers — they're this session's snapshot, not fixed constants. Separately, a Lead's `Next_Call_date` (lowercase "d", same field the lead-audit skill uses) is still worth surfacing as "follow-ups due today" alongside the backlog total — it's a due-date cut of the same backlog, not a replacement for it. `SELECT COUNT(id) FROM Leads WHERE Next_Call_date < '<today>'` (384 as of this writing) is the raw overdue count, not yet filtered by status.

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

### 3. Leads created (ad platform) vs. leads reached Zoho — reconciliation

Bansal wants this tracked: does everything the ad platforms report as a lead actually land in Zoho as a Lead record? Compare Windsor.ai's platform-reported lead count (step 2) against Zoho's own count for that source over the same 120-day window. **Verified quirk — the Zoho `Lead_Source` value for Facebook is `'Meta Ads'`, not `'Facebook'`** (`Lead_Source = 'Facebook'` silently returns 0 — it's not that there are no Facebook leads, it's the wrong string):
```sql
SELECT COUNT(id) FROM Leads WHERE Lead_Source = 'Meta Ads' AND Created_Time >= '<120 days ago>'    -- 10
SELECT COUNT(id) FROM Leads WHERE Lead_Source = 'Google Ads' AND Created_Time >= '<120 days ago>'   -- 20
```
As of this writing, over the same 120-day window: **Facebook** — Windsor reports 15 leads tracked, Zoho shows 10 as `Meta Ads` (a real gap: ~5 leads/33% not showing up as Zoho leads, or arriving with a different/blank source). **Google Ads** — Windsor reports only ~5.3 tracked conversions, but Zoho shows 20 leads sourced as `Google Ads` — the *opposite* problem: Google's own conversion tracking is significantly under-counting real leads that are demonstrably reaching Zoho, which likely also means Google Ads isn't optimizing against the real lead volume. Report both directions, not just "leaks" — a platform under-reporting its own conversions is as worth flagging as leads dropping before Zoho. Re-run live each time; these are this session's numbers, not fixed facts.

### 4. Gmail — inbox summary

Call `list_labels` for the unread/total counts on `INBOX` and `IMPORTANT` (verified working — confirmed a large backlog: thousands of unread messages, so lead with unread *change since last run* if tracked, not just the raw total, since the raw total will always look alarming). Use `search_threads` for anything more targeted (e.g. BitTRACK correspondence, per `b2b-portal-check.md`). `list_labels` errored transiently once during setup — retry once on failure; if it still fails, note "Gmail summary unavailable this run" rather than blocking the whole dashboard.

### 5. Active B2B-portal applications and Visitor Visa pipeline — two input paths, not just one

What Bansal wants here: **number of active applications, the Visitor Visa pipeline specifically, and which stage each application is at** — not just a count. This comes from two places, and a dashboard run should use whichever is freshest:

1. **The cached portal-check file**, `data/b2b-applications.json`, written every couple of days by a manual desktop-session login pass (see below). The B2B partner portals (KC Overseas, Crizac, SI/StudyIn, Leverage Edu, BitTRACK) have no API and no MCP connector, and reading them requires the "Claude in Chrome" browser-control tools (click by coordinate, screenshot, page text) that only exist in a **desktop Claude Code session** — this cloud/on-demand session doesn't have them, so it can't log in live here.
2. **Whatever Bansal shares directly in the conversation** when he runs the dashboard — he said some of this information will just be given here rather than always coming from a portal login. Treat a direct chat update as at least as current as the cached file (probably more current), and fold it in rather than overwriting it silently — if it conflicts with the cache, the conversational input wins, but say so.

On a dashboard run:
- Read `data/b2b-applications.json` for the cached baseline (per-application stage, not just a total, once the cache actually has that shape — see the note in `b2b-portal-check.md`).
- Ask/incorporate anything Bansal gives directly this run, especially Visitor Visa status.
- Show the cache's `last_checked` timestamp — if stale (no update in ~4 days) and nothing was given conversationally either, flag that visibly rather than presenting old data as current.
- Do **not** attempt a browser login here — this session doesn't have the tools for it.

### 6. Ops data outside Zoho — tasks/calls still need Bansal's input

The Lead `Tag` split above covers each person's lead backlog, but Tasks and Calls have no Tag data and Owner is useless (see above), so there's still no Zoho-native way to attribute today's calls, or any "processing" work, to Harsh vs Krisha specifically. Ask Bansal conversationally at the start of the run ("anything on calls/processing outside the lead backlog I should fold in for Harsh and Krisha today?") rather than presenting the Zoho-only call totals as already broken down by person. If he has nothing to add, show the Zoho numbers as org-wide totals rather than guessing a split.

### 7. Render the dashboard

Follow the `dataviz` and `artifact-design` skills for the visual pass. Publish as a single HTML Artifact with:
- Stat tiles: leads today, combined open backlog to close (Harsh + Krisha), calls today, unread emails
- Marketing panel: spend, cost-per-lead, leads by platform (Facebook vs Google Ads) over the 120-day window
- Leads-created-vs-reached-Zoho reconciliation (step 3) — call out both directions (leads not reaching Zoho, and platforms under-reporting conversions that did reach Zoho)
- Lead-source breakdown over the 120-day window (today's count shown as context, not the headline — see the short-window quirk above)
- Ops-by-owner panel: Harsh vs Krisha open-lead backlog (their "tasks") as the headline, follow-ups due today as a sub-figure, both from the Lead `Tag` split (step 1) — plus whatever call/processing split Bansal gives conversationally in step 6, since Zoho has no Tag data on Tasks/Calls
- Deal pipeline funnel (stage counts)
- Active Applications / Visitor Visa panel (step 5): per-application stage where available, cached `last_checked` timestamp, plus anything Bansal shared directly this run

Redeploy to the same Artifact URL on every run (don't create a new artifact each time) once the first one exists for this workflow — track the URL in the conversation so repeat runs update it in place.

If the request is conversational rather than asking to "see" a dashboard, a plain-text summary of the same numbers is fine — read the room.

## Manual B2B portal check (separate from this workflow, desktop session only)

See `b2b-portal-check.md` in this folder for the per-portal login/navigation steps. Run it from a desktop Claude Code session with the Chrome extension attached, roughly every 2 days. It writes its results to `data/b2b-applications.json`, which this dashboard workflow then reads. Credentials are never stored in this repo — `b2b-portal-check.md` points to where they live instead.
