# VehicleNet — hackathon prototype (Android app + backend + dashboard)

Advisory-only vehicle-to-cloud-to-vehicle safety prototype. Cars and bikes
share anonymous location/speed/heading; nearby participants get short,
structured alerts. **It never touches brakes, steering, throttle, or any
vehicle control system.**

This repo combines all three pieces of the original hackathon stack into one
project:

```
Android app  --(writes/reads)-->  Firebase Realtime DB  <--(reads)--  backend
                                                                          |
                                                                    Socket.io
                                                                          |
                                                                     dashboard
```

- **`app/`** — Kotlin + Jetpack Compose Android app (the phone client)
- **`backend/`** — Node.js/TypeScript service that runs the same nearby-vehicle
  logic centrally and broadcasts it over Socket.io
- **`frontend/`** — plain HTML/CSS/JS dashboard for watching the whole system
  live (e.g. on a laptop while judges watch the phone demo)

Everything mirrors the Android/Kotlin source 1:1, so behavior stays
consistent across all three surfaces:

| Concept | Android (Kotlin) | Backend (TypeScript) |
|---|---|---|
| Data model | `app/src/main/java/.../model/VehicleUpdate.kt`, `model/EventType.kt` | `backend/src/types.ts` |
| Distance/bearing math | `app/src/main/java/.../logic/GeoUtils.kt` | `backend/src/geoUtils.ts` |
| Nearby-vehicle rules | `app/src/main/java/.../logic/NearbyVehicleEngine.kt` | `backend/src/nearbyVehicleEngine.ts` |
| Zero-setup demo | `app/src/main/java/.../data/DemoModeSimulator.kt` | `backend/src/demoModeSimulator.ts` |

## What's built

- Kotlin + Jetpack Compose Android app (single module, `app/`)
- GPS tracking via `FusedLocationProviderClient`, one update/second (`data/LocationTracker.kt`)
- Firebase Realtime Database sync + anonymous auth (`data/FirebaseVehicleRepository.kt`)
- Rotating anonymous vehicle ID, no name/phone/permanent ID (`data/VehicleIdProvider.kt`)
- Nearby-vehicle engine: Haversine distance, 500 m radius, 5 s staleness cutoff,
  hard-braking inference from a sharp speed drop + similar heading (`logic/NearbyVehicleEngine.kt`)
- Structured help requests only — `BREAKDOWN`, `MEDICAL`, `ACCIDENT`, `CHARGING_HELP`,
  `HAZARD` — no free-text chat while driving (`model/EventType.kt`)
- "I can help" is only enabled once the receiving driver has been stationary for
  a few seconds, per the driver-distraction requirement
- Demo Mode: 8 simulated vehicles circling a fixed point, one of which fires a
  hazard event ~20s in, so the whole flow is demoable with a single phone and
  zero setup (`data/DemoModeSimulator.kt`)
- Built-in radar view (`ui/RadarView.kt`) — a Compose Canvas showing you at the
  center and nearby vehicles as dots by bearing/distance, colored red when
  they're alerting. This needs no API key, so it's the safe fallback for judging.
- A TypeScript backend (`backend/`) that runs the same nearby-vehicle logic
  centrally for *every* vehicle at once, not just one "self" like the phone app
- An HTML/CSS/JS dashboard (`frontend/`) that visualizes the backend's live
  feed with its own radar, table, and instrument readouts

## What you need to add (the "GPS and some other things" part)

1. **A physical device or emulator with location** — real GPS hardware can't
   be tested from here. Run on an actual Android phone (or an emulator with a
   simulated location route) with location services turned on.
2. **A Firebase project**:
   - Create one at the Firebase console, add an Android app with package name
     `com.hackathon.vehiclenet`, download `google-services.json`, and drop it
     into `app/google-services.json`.
   - Enable **Realtime Database** (start in test mode for the hackathon).
   - Enable **Authentication → Anonymous** sign-in.
   - Suggested rules once you're past pure test mode:
     ```json
     {
       "rules": {
         "vehicles": { ".read": "auth != null", ".write": "auth != null" },
         "helpResponses": { ".read": "auth != null", ".write": "auth != null" }
       }
     }
     ```
   - The backend needs the same project's service account key if you want it
     in `firebase` mode — see `backend/.env.example`.
3. **A real map (optional)** — swap `RadarView` for Google Maps Compose or
   OpenStreetMap once you have a Maps API key. The radar view already has the
   distance/bearing math you'll need for markers.

## Running the Android app

1. Open this folder in Android Studio. Let Gradle sync; if it suggests a
   newer Android Gradle Plugin/Kotlin version, accept it — the versions
   pinned here are a reasonable Sept-2026 baseline but Android Studio's own
   suggestions will be more current.
2. Grant location permission when prompted.
3. **Fastest path to a demo:** tap **Demo Mode** — no Firebase, no second
   device, no GPS needed. Watch a hazard alert appear on the radar ~20s in.
4. **Real two-phone test:** on both phones, pick Car/Bike, tap **Start
   Drive**, and drive/walk them within 500m of each other. Confirm a hard-brake
   or Request Help event on one phone shows up on the other within a few
   seconds.

## Running the backend

Node.js + Express + Socket.io. Computes, once a second, which alerts are
relevant to *every* vehicle, and broadcasts the full vehicle list + alert map
to any connected dashboard.

Two data sources, picked by `SOURCE_MODE` in `.env`:
- **`demo`** (default) — runs its own internal simulator, identical in spirit
  to the Android app's Demo Mode. No Firebase, no credentials, works
  immediately.
- **`firebase`** — reads the same `vehicles` node the Android app writes to,
  via the Firebase Admin SDK. Needs a service account key (see `backend/.env.example`).

```bash
cd backend
npm install
cp .env.example .env      # defaults to SOURCE_MODE=demo, no edits needed to try it
npm run dev
```

It listens on `http://localhost:4000`. Check `http://localhost:4000/health`
to confirm it's alive, and `http://localhost:4000/vehicles` to see the raw
current snapshot.

REST endpoints, in case you want another client to push data in without
going through Firebase:
- `GET /vehicles` — current snapshot
- `GET /vehicles/:id/alerts` — alerts currently relevant to one vehicle
- `POST /vehicles` — upsert one vehicle record (JSON body matching `VehicleUpdate`)
- `DELETE /vehicles/:id` — remove a vehicle (e.g. on "Stop Drive")

Socket.io events it emits every second: `vehicles:update` (array) and
`alerts:update` (object keyed by vehicleId).

## Running the frontend dashboard

Plain HTML/CSS/JS, no build step. Open `frontend/index.html` directly in a
browser — it connects to `window.BACKEND_URL` (defaults to
`http://localhost:4000`, editable inline in `index.html` if you deploy the
backend elsewhere).

Shows:
- A live radar (same distance/bearing math as the Android radar view),
  plotting every vehicle relative to the group's center point, colored by
  severity — teal for normal, amber for hazards/braking/help requests, red
  for medical/accident.
- A live table of every vehicle: type, speed, status, seconds since last update.
- Two instrument-style readouts up top: active vehicle count and active alert count.

This is the "Hackathon dashboard" piece from the original spec — meant to be
left open on a laptop or projected screen while judges watch the Android
demo run.

## Fastest path to seeing all three pieces together

1. `cd backend && npm install && npm run dev` — starts in demo mode, no setup.
2. Open `frontend/index.html` in a browser — you'll see the same simulated
   vehicles the backend is generating, live.
3. Separately, the Android app's own **Demo Mode** button still works exactly
   as before, fully offline — it doesn't talk to this backend. To see the
   *same* vehicles on your phone and the dashboard, run the Android app in
   **Start Drive** mode against a real Firebase project, and set the backend's
   `SOURCE_MODE=firebase` pointing at that same project.

## Known gaps / natural next steps

**Android app:**
- `observeHelpResponses()` in the repository already streams how many vehicles
  offered to help, but it isn't wired into the UI yet — cheap to add as a small
  counter under the alert banner.
- Only one alert is shown at a time (the first in the list) — fine for a
  three-minute demo, but you'll want a short queue or list for anything longer.
- Radar view assumes "up" is compass north, not direction of travel — swap in
  a real map when you have a key if judges expect map-north orientation.
- Real radar, cameras, and CAN-bus signals are deliberately out of scope: those
  need OEM-level permissions and belong in a documented "future work" slide,
  not this prototype.

**Backend / frontend:**
- No authentication on the backend's REST/Socket.io endpoints — fine for a
  judged demo on a local network, not for anything public.
- In demo mode, backend state resets on restart (in-memory only).
- The dashboard has no "self", so its radar centers on the group's centroid
  rather than any one vehicle's point of view — a deliberate difference from
  the Android radar, which centers on you.
