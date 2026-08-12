# Screenshot / Visual Checklist — Jōb

Capture these yourself (I'm not taking screenshots). File naming suggestion in brackets.

## App screens (from job-mobile/screens/)

1. **Home screen** [`01-home.png`] — the main dashboard (`HomeScreen.js`). This is the first
   thing a viewer sees; pick a state with a few applications visible so it doesn't look empty.
2. **Applications list** [`02-applications-list.png`] — `ApplicationsScreen.js`, showing
   several applications with different statuses (Applied / In Review / Interview / Offer /
   Rejected) so the status system is visible at a glance.
3. **Application detail** [`03-application-detail.png`] — `ApplicationDetailScreen.js` for a
   single application, ideally one with an interview date or status history so it shows off
   the timeline aspect, not just a bare record.
4. **Insights / analytics screen** [`04-insights.png`] — `InsightsScreen.js`. This is your
   differentiator vs. a plain tracker — capture it once there's enough data to show a real
   chart/summary, not a "no data yet" empty state.
5. **Job AI / companion screen** [`05-job-ai.png`] — `JobAIScreen.js`, showing an actual AI
   response or companion message, not a blank input box.
6. **Weekly review screen** [`06-weekly-review.png`] — `WeeklyReviewScreen.js`.
7. **Goals screen** [`07-goals.png`] — `GoalsScreen.js`, showing a streak or progress state.
8. **Onboarding — one representative slide** [`08-onboarding.png`] — pick the AI or
   auto-sync slide (`AISlide.js` / `AutoSyncSlide.js`) since those explain the product's core
   pitch better than the welcome slide.
9. **Sync moment screen** [`09-sync-moment.png`] — `SyncMomentScreen.js`, the screen shown
   right after a Gmail sync finds something new — good evidence the Gmail integration is real
   and working.

## Diagrams to sketch (not screenshots)

10. **Classification pipeline diagram** [`10-classifier-pipeline.png`] — a simple flow:
    `Gmail message → quick_filter (heuristic, free) → [skip] or → Claude Haiku (classify) →
    sync_service (dedupe/match) → Postgres`. This is the most interesting technical piece of
    the backend and isn't visible in any UI screenshot, so it needs its own diagram.
11. **(Optional) status state machine** — Applied → In Review → Interview → Offer/Rejected,
    since `sync_service.py` explicitly enforces valid transitions.

## Notes

- Use a test/seed account with realistic-looking (but fake) company names and applications —
  don't screenshot your real personal job search data if you'd rather not share it publicly.
- Crop mobile screenshots to just the phone frame; keep resolution consistent across all of
  them (e.g. all from the same simulator/device).
