# Topuzz

A mobile jigsaw puzzle app. Puzzles up to 5,000 pieces, real-time collaborative solving with friends, and a custom-built rendering engine underneath, since nothing off the shelf could handle a jigsaw board at that scale smoothly.

Built solo. React Native/Expo on the frontend, FastAPI and PostgreSQL on the backend.

## Screenshots

### Home
<img src="screenshots/01-home.png" width="280">

A corkboard-style layout where each puzzle is pinned like a physical photo, with a deterministic pin position, tape angle, and rotation seeded per puzzle so it looks intentional rather than randomly scattered on every load.

### Library
<img src="screenshots/02-library.png" width="280">

The puzzle catalog, organized by a tier ladder from 96 pieces up to 4,988. This is where difficulty progression actually lives, so someone starting out isn't dropped into a 5,000-piece board on day one.

### Solve screen
<img src="screenshots/03-solve-screen.png" width="280">

Mid-puzzle, this is the actual rendering engine at work: pieces with real jigsaw tab-and-socket edges, smooth pan and pinch-zoom regardless of puzzle size, and a piece tray you drag from. Everything below explains what's actually happening here.

## The interesting part: rendering thousands of jigsaw pieces at 60fps

A puzzle board needs to clip thousands of independently-shaped image regions (each with a jigsaw tab or socket) and stay smooth under continuous pan and pinch-zoom. Standard React Native list or grid components can't clip to a non-rectangular shape per cell, and a plain view tree with 1,000+ pieces blows past React's rendering budget every frame. So the board is built directly on `@shopify/react-native-skia`, with a three-tier model that separates "what's on screen" from React's render cycle entirely:

1. **Every frame:** pan and zoom are driven by Reanimated shared values on the UI thread, feeding directly into Skia's transform. React never re-renders for this, which is why the board stays smooth no matter how many pieces are on it.
2. **On re-record:** the JS thread rebuilds an `SkPicture`, an imperative recording of draw calls, but only when the visible area moves outside a padded window around the current viewport. Between re-records, the canvas just replays the same static picture under a moving transform.
3. **React render:** happens once per re-record, swapping a single `Picture` prop. Cheap, and rare.

**Viewport culling** works the same way: only a bounded window of cells around what's actually visible is ever drawn or has its jigsaw path computed, padded by roughly one extra screen's worth in each direction. A 5,000-piece puzzle costs the same per frame as a 100-piece one, because the vast majority of pieces are simply never touched until they're about to come into view.

Reconciling those two things (a value that must update every frame on the UI thread, and an imperative recording API that can only run on the JS thread) was the actual hard problem here. Getting the invalidation right, so a theme change, a tile finishing loading, a hint pulse, and normal panning all funnel through one consistent "redraw this" mechanism without double-recording or showing a stale picture mid-gesture, took several rounds of hardening.

**How piece shapes are generated:** there's no server-side step deciding which edges bulge or dent. Each edge's shape is computed client-side from a small integer hash of its row, column, and axis. Two adjacent cells hash the same shared edge, so a tab drawn bulging out of one cell is guaranteed to line up with the matching socket carved into its neighbor, with zero shared state and zero network round-trip. Any client can recompute any cell's exact shape independently and get the same answer every time.

## Real-time multiplayer

Up to four people can solve the same puzzle together, with piece placements syncing live between everyone in the session.

It runs on plain FastAPI WebSockets rather than a third-party pub/sub service, a deliberate choice given the small session size: four peers on one backend process doesn't need external infrastructure. Session state lives in Postgres, plus a lightweight in-memory registry that tracks who's currently connected.

Two things worth calling out:

- **Presence without double-firing.** A user can have multiple devices connected at once, so "did this user just come online" and "did they just go fully offline" only fire on their first and last connection respectively, not on every device individually.
- **One writer for disconnect handling.** A connection can close two different ways: the client disconnects naturally, or the server force-closes it (a kick, or the host ending the session). Both paths funnel through a single cleanup point rather than each handling their own broadcast, which structurally prevents a race where the same disconnect could get double-broadcast or missed entirely.

There's also a `topuzz://` deep link scheme so a shared join code can open the app directly to a pre-filled join screen, without needing a verified domain for universal links.

*(No screenshot yet for the multiplayer session screen itself, coming once I can capture a real multi-device session.)*

## What else is in there

- **Board atmosphere themes** (Warm Wood, Card Table Felt, Midnight Study) for visual variety while solving.
- **Reveal modes** that control how much of the empty board outline you see while placing pieces.
- **A real image pipeline** that takes a source photo and target piece count, computes an aspect-ratio-matched grid, and slices it into padded tiles so the jigsaw clip path always has real pixel data under the tab tips, not just a bare square.

## Stack

- **Mobile:** React Native, Expo, react-native-skia
- **Backend:** FastAPI, PostgreSQL, WebSockets
- **Storage:** Supabase Storage (puzzle tile images)
- **Infra:** GitHub Actions

## Running it locally

See `.env.example` for required environment variables. Real credentials are not included in this repo.

\`\`\`
# backend
cd backend
pip install -r requirements.txt
uvicorn main:app --reload

# mobile
cd app
npm install
npx expo start
\`\`\`