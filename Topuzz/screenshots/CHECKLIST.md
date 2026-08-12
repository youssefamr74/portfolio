# Screenshot / Visual Checklist — Topuzz

Capture these yourself (I'm not taking screenshots). File naming suggestion in brackets.

## App screens (from app/src/screens/)

1. **Home screen** [`01-home.png`] — `HomeScreen.tsx`, ideally with a pinned photo / an
   in-progress puzzle visible (the "corkboard" home design is a nice visual hook).
2. **Library screen** [`02-library.png`] — `LibraryScreen.tsx`, showing multiple puzzles so
   the browsing experience is clear.
3. **Solve screen — mid-puzzle** [`03-solve-midway.png`] — `SolveScreen.tsx` with the puzzle
   roughly half-assembled. This is the single most important screenshot in the whole
   portfolio for this project — it's the one place the Skia rendering actually shows.
4. **Solve screen — piece tray / box sorting** [`04-piece-organize.png`] — capture
   `OrganizePiecesModal.tsx` / `BoxPickerModal.tsx` in use, showing pieces sorted into boxes.
5. **Multiplayer session — presence bar** [`05-multiplayer-presence.png`] — `PresenceBar.tsx`
   showing more than one participant online in the same session. This is the proof the
   real-time collab feature works, not just exists in code.
6. **Join session modal** [`06-join-session.png`] — `JoinSessionModal.tsx`, showing the
   session-code entry flow (and ties back to the `topuzz://join/<code>` deep link).
7. **Profile / paywall screen** [`07-profile-or-paywall.png`] — `ProfileScreen.tsx` or
   `PaywallScreen.tsx` showing the Pro-tier upsell.
8. **Onboarding — one representative screen** [`08-onboarding.png`] — pick a screen that
   shows the "large tap targets / plain language" design constraints from the kickoff spec in
   action, since that's a differentiator worth demonstrating visually.

## Motion — worth more than a static screenshot here

9. **Screen recording or GIF of dragging/placing a piece** [`09-drag-piece-demo.gif`] — the
   Skia canvas + gesture-handler interaction is the whole technical point of this project; a
   short (5–10s) recording of smooth pan/zoom/drag will sell it far better than any still image.
10. **(Optional) screen recording of a live 2-player session** [`10-multiplayer-demo.gif`] —
    two devices/simulators side by side, one placing a piece and the other seeing it update in
    real time. Strong evidence for the WebSocket sync claim.

## Diagrams to sketch (not screenshots)

11. **Rendering pipeline diagram** [`11-rendering-pipeline.png`] — `source image → tiled into
    grid → jigsawPath.ts generates tab/socket edges per cell → Skia Picture recording →
    Canvas + pan/zoom via reanimated`.
12. **(Optional) multiplayer room diagram** — `client WS connect → rooms.py registers
    (session_id, user_id) → broadcast to other sockets in same room → client applies event`.

## Notes

- Use one of the sample/placeholder puzzle images already in `app/assets`, not a personal
  family photo, if you want to keep the portfolio fully public-safe.
- Keep all screenshots from the same device/simulator and orientation for visual consistency.
