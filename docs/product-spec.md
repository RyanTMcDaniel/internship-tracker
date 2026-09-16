# Product Spec: Internship Tracker

## 30-Second Pitch

Internship Tracker is a hosted spreadsheet for internship applications. Paste a
job listing URL, and the app extracts the company, role, salary, and locations
into a draft. You fix whatever it got wrong, accept it, and it becomes a row you
track through status buckets from Saved to Offer. Every user signs in and only
ever sees their own applications.

## Problem

Students applying to dozens or hundreds of internships track them in personal
spreadsheets that fall apart. Every listing gets copy-pasted by hand, columns
drift out of consistency, nobody records when a status changed, deadlines get
missed, and there is no easy way to see response rates or where applications
are stalling.

## Users

Primary users are me and a small group of friends applying to software
internships. Secondary audience is engineers and recruiters reviewing the
codebase. This is not built for recruiters, career centers, or companies.

## Core Workflow

1. Sign in.
2. Paste a job listing URL.
3. Review the extracted draft and correct any wrong or missing fields.
4. Accept the draft, which creates an application row.
5. Manage applications in a spreadsheet view with bucket tabs, sorting,
   filtering, and inline edits.
6. Move applications through statuses, with each change recorded as an event.
7. Check an analytics summary of totals, response rate, and upcoming deadlines.

## V1 Scope

- Authentication through Supabase Auth, with all data private to each user.
- Manual application creation without a URL.
- Best-effort URL extraction into a reviewable draft. Extracted data is never
  saved as an application without user review.
- Bucket tabs: All, Saved, Applied, OA, Interview, Offer, Rejected, Archived.
- Fields: company, role, category, locations, salary range and period, deadline,
  status, source URL, notes, and last updated date.
- Sorting and filtering by status, deadline, category, and location.
- Status transitions that write an event history entry.
- Per-user analytics: total applications, counts by status, response rate,
  upcoming deadlines, and applications by category.
- In-app follow-up reminders for stale applications.

## Non-Goals For V1

- Job discovery or search. Users bring their own listings.
- Auto-applying or filling out application forms.
- Email, calendar, or LinkedIn integrations.
- Browser extension or native mobile app.
- Resume or cover letter generation.
- Sharing applications between users or team workspaces.
- Email or push notifications.
- Guaranteed extraction from every job board.

## Constraints And Known Limitations

- Many job boards render listings with JavaScript, require login, or restrict
  scraping. Extraction is best-effort and will fail on some sites. The app does
  not bypass logins, CAPTCHAs, or bot protection.
- Fetches only happen when a user submits a URL, use a descriptive user agent,
  are rate limited, and are cached to avoid repeat requests.
- Built for small scale on free or low-cost hosting: tens of users, not
  thousands.

## Success Criteria

- A friend can sign up on the hosted site, paste a listing URL, review and accept
  the draft, edit the row, and see their own analytics.
- Two users can never read or modify each other's data, proven by tests.
- I use it for my own internship applications instead of a spreadsheet.

## Decisions

- **Deleting:** Applications in Saved are hard deleted. Applications past Saved
  cannot be deleted and must be moved to Archived instead. The API rejects
  deletes on non-Saved applications.
- **Archived:** Archived is a status. The previous status is recovered from
  event history, and un-archiving restores it.
- **Rejected vs Archived:** Rejected means the company declined. Archived means
  the user withdrew or stopped pursuing it. They are never combined.
- **Category:** Fixed list (SWE, Data, PM, Hardware, Other) plus an optional
  free-text label. The scraper only assigns from the fixed list. Analytics group
  by the fixed category.
- **Status transitions:** Rejected can move back to any active status.
- **Deadlines:** Date only, stored as DATE with no time zone.
- **Salary:** Store min, max, currency, and period (hour, month, year, unknown)
  exactly as listed. Sorting uses a computed hourly equivalent (annual / 2080,
  monthly / 173). Unknown periods sort last. No annualizing.
- **Response rate:** Applications that ever reached OA, Interview, Offer, or
  Rejected, divided by applications that ever reached Applied.
- **Positive rate:** Applications that ever reached OA, Interview, or Offer,
  divided by applications that ever reached Applied.
- **Analytics source:** Rates are computed from event history, not current
  status, so archived applications still count.
