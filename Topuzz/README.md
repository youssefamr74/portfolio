# Topuzz — Family Jigsaw Puzzle App (with live co-op)

> Portfolio notes for the `topuzz` project. Source of truth is `Documents/topuzz/app` (React
> Native/Expo client) and `Documents/topuzz/backend` (FastAPI service), plus the project's own
> `topuzz-app-kickoff-spec.md`.

## What it does

Topuzz recreates the feeling of a physical jigsaw puzzle spread out on a table — pick a
photo, get it cut into pieces, drag pieces around a large pannable/zoomable board, sort them
into "boxes," and finish the picture. It's explicitly designed for non-technical/older users
(the spec calls out large tap targets, no jargon, no forced tutorial, undo-friendly
interactions) as well as puzzle enthusiasts.

Concretely, from the code:

- **Puzzle board rendering** (`app/src/rendering/`) — a custom canvas renderer built on
  `@shopify/react-native-skia`, not a stack of individual RN `<Image>`/`<View>` components per
  piece. `jigsawPath.ts` procedurally generates jigsaw-shaped tab/socket edges per cell using a
  deterministic hash so that two adjacent pieces always agree on whether their shared edge is a
  tab or a socket. `PuzzleBoard.tsx` records the board as a Skia `Picture` and pans/zooms it
  with `react-native-gesture-handler` + `react-native-reanimated`.
- **Puzzle organization** — "boxes" to sort loose pieces into, an auto-sort hook
  (`useAutoSort.ts`), a magnifying "reveal mode," a piece-ranking utility — the kind of tooling
  a real physical-puzzle solver actually wants (per the kickoff spec's design constraints).
- **Live multiplayer** — `backend/app/routers/ws.py` + `backend/app/services/rooms.py`
  implement a per-process WebSocket "room" system: multiple users can join the same puzzle
  session, see who else is online (presence), and see each other's piece placements and box
  changes in real time. The mobile side (`app/src/hooks/useCollabSession.ts`) auto-reconnects
  on drops but treats explicit server-sent close codes (auth failure, kicked, session ended) as
  terminal — no pointless retry loop.
- **Accounts & entitlements** — Google Sign-In + JWT auth, and a Pro tier gated via RevenueCat
  purchases (`backend/app/services/entitlements.py`).
- **Deep linking** — `topuzz://join/<code>` lets a host share a session code that opens
  straight into the join flow.

## Tech stack

**Mobile app** (`app/`)
- React Native 0.86 + Expo (~57), TypeScript
- `@shopify/react-native-skia` for the puzzle board canvas
- `react-native-gesture-handler` + `react-native-reanimated` for pan/zoom/drag
- `@react-navigation/native-stack`, `expo-secure-store`, `@react-native-google-signin/google-signin`

**Backend** (`backend/`)
- FastAPI + SQLAlchemy + Alembic migrations, Postgres
- Plain FastAPI `WebSocket` for real-time multiplayer (no external message broker)
- JWT auth (`security.py`), Google Sign-In verification
- Deployed via a `Procfile` (Railway-style)

## Architecture

```
┌───────────────────────────────┐
│   topuzz mobile app (Expo)     │
│                                │
│  Skia canvas renderer          │
│  (jigsawPath + PuzzleBoard)    │
│                                │
│  useCollabSession (WS client)  │
└───────────┬─────────┬─────────┘
            │ REST     │ WebSocket (topuzz://join/<code>)
            ▼          ▼
┌─────────────────────────────────────────┐
│              topuzz backend (FastAPI)     │
│                                            │
│  routers: auth, puzzles, boxes,           │
│           categories, progress,           │
│           sessions, purchases, ws          │
│                                            │
│  services: rooms.py (in-memory WS         │
│  registry, per session_id → per user_id   │
│  → set of live sockets)                   │
│  services: puzzle_state, entitlements     │
└───────────┬─────────────────┬─────────────┘
            │                 │
            ▼                 ▼
   ┌─────────────────┐  ┌───────────────────────┐
   │ Postgres          │  │ Google Sign-In / JWT   │
   │ (users, puzzles,  │  │ RevenueCat (Pro tier)  │
   │ boxes, sessions,  │  └───────────────────────┘
   │ progress,         │
   │ purchases)        │
   └─────────────────┘
```

## Interesting technical decisions worth calling out

- **Skia `Picture` recording instead of per-piece React components.** A jigsaw puzzle can have
  hundreds of pieces; mounting that many RN views would be a performance problem. Instead the
  board is recorded once as a Skia `Picture` and redrawn on a single canvas, with pan/zoom
  handled by reanimated shared values rather than React re-renders.
- **Deterministic, hash-seeded piece edges.** `jigsawPath.ts` derives each shared edge's
  tab/socket direction from a hash of `(row, col, axis)`. Both neighboring pieces compute the
  same hash independently and therefore always agree — no need to precompute or store an edge
  map.
- **Multiplayer chosen to be "just enough," not over-engineered.** The code comment in
  `rooms.py` is explicit about this: a plain in-process WebSocket dict was chosen *because*
  the current use case (≤4 peers, single backend instance) doesn't need Redis/pub-sub — with a
  documented note about what would need to change if the backend ever scaled horizontally.
  That's a deliberate, reasoned trade-off rather than an oversight.
- **Terminal vs. transient WebSocket closes.** The mobile client distinguishes server close
  codes that mean "stop trying" (4401/4403/4404/4409/4410 — auth failure, kicked, session
  ended) from an ordinary dropped connection, which it retries automatically.

## Running it locally

**Backend**
```bash
cd backend
python -m venv .venv && source .venv/bin/activate   # or .venv\Scripts\activate on Windows
pip install -r requirements.txt
cp .env.example .env   # fill in real values — see backend.env.example in this folder
alembic upgrade head
uvicorn app.main:app --reload
```

**Mobile app**
```bash
cd app
npm install
cp .env.example .env   # fill in real values — see app.env.example in this folder
npx expo start
```

**Puzzle image tiling script** (`scripts/tile_image.py`) is a separate helper used to prepare
source images for puzzles (uploads to Supabase Storage per the kickoff spec) — see
`scripts.env.example` for the config it expects.

## Configuration

See `app.env.example`, `backend.env.example`, and `scripts.env.example` in this folder for
every environment variable the real `.env` files define (names only, placeholder values). The
real `.env` files and the `.jks` signing keystores were **not** copied into this portfolio
folder. Note: the real `topuzz/scripts/.env` is currently committed in the actual project's
git history with a live Supabase service-role key — that needs rotating/scrubbing in the real
repo independently of this portfolio folder.
