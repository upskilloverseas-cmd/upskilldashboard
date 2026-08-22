# B2B Partner Portal Check — run from a desktop Claude Code session

Run this roughly every 2 days from a **desktop Claude Code session with the "Claude in Chrome" extension attached** (this only works there — the cloud/on-demand dashboard session has no browser-control tools). After finishing, update `data/b2b-applications.json` in this repo with the results and a fresh `last_checked` timestamp, then commit and push so the on-demand dashboard picks it up.

What Bansal actually wants out of this: not just a count, but **each active application's current stage** (and whether it's a Visitor Visa application specifically) so the dashboard can show who's at what stage, not just "N applications." Capture per-student/per-application detail, not a portal-level total — see the JSON shape below. If a portal's dashboard only shows a rolled-up number with no per-application breakdown, note that limitation rather than inventing stage detail that isn't there.

Credentials for every portal below live in `Partner Login Details.docx` (Downloads) — check there first, since passwords can change without notice. Nothing in this file or in this repo should ever contain an actual password.

## Portals

1. **KC Overseas (Krishna Consultants) — coursefinder.ai**
   Login via the "Login to coursefinder.ai" link, lands on `/dashboard/partner`. Active applications: check the sidebar (same place "Commission Structure" lives) — exact nav label not yet pinned down.

2. **Crizac — crizac.com**
   `https://www.crizac.com/authentication/login` → "AGENT LOGIN" → `/agent/dashboard`. Check nav for an "Applications" section — not yet mapped in detail.

3. **SI / StudyIn — si-applications.com**
   `https://si-applications.com/auth/login`. **As of 29 Jul 2026 this account is locked** for inactivity — needs reactivation via SI support (contacts in `Partner Login Details.docx`) before it can be checked again. Once unlocked, look for an "Applications" report alongside the commission report.

4. **Leverage Edu — StudentOps360**
   `https://upskillsovwerseas.studentops360.io/` → email → "Generate OTP" → **the OTP arrives by email in seconds** (Gmail search for subject "OTP for StudentOps360 Login" from `communications@studentops360.io` — needs Gmail search tool access in the same session) → enter code → Login. Post-login `/dashboard` shows "Recent Enrolments" and a conversion funnel directly — fastest read on active activity.

5. **BitTRACK — Gmail only, no portal**
   All-email relationship. Search `upskilloverseas@gmail.com` for BitTRACK's sending domains (`ukapp@bittrack.com`, `ukadmissions@bittrack.com`, `ausadmissions@bittrack.com`, `canada.marketing@bittrack.com`) or by student name / `BitTRACK CRM:<ref>` if known. This one *can* run from the cloud dashboard session too, since it's pure Gmail search — no browser needed.

## General gotchas

- No persistent login sessions — expect to log into every portal fresh each check.
- React-controlled login forms: click the input by pixel coordinate (not just the accessibility ref) before typing, then zoom-screenshot to confirm the text landed before submitting.
- First screenshot attempt occasionally times out (~30s) — retry once.
- Google Sheets referenced from these portals (commission/deadline sheets) render on canvas — `get_page_text` won't capture them; click a cell, Ctrl+F, and screenshot instead.

## After checking

Update `data/b2b-applications.json`, one entry per active application (not one entry per portal):
```json
{
  "last_checked": "<ISO timestamp>",
  "portals_checked": ["<which portals were actually reachable this pass>"],
  "applications": [
    {
      "student": "<name or ref as shown in the portal>",
      "portal": "<KC Overseas | Crizac | SI/StudyIn | Leverage Edu | BitTRACK>",
      "stage": "<whatever stage label the portal itself uses — don't normalize it away>",
      "is_visitor_visa": false,
      "last_updated": "<date shown in the portal, if any>"
    }
  ],
  "note": "<call out anything locked/unreachable this pass, e.g. SI still locked, or a portal that only exposes a total with no per-application stage>"
}
```
Commit and push so the next `upskill-dashboard` run in the cloud session picks up the fresh snapshot. If Bansal shares an update directly in a dashboard-run conversation instead of via this file, that's a valid second input path too (see `SKILL.md` step 4) — it doesn't have to go through this file every time.
