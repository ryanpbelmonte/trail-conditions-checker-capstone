# E2E Walk — Server-side slice

**Role:** Ryan Belmonte (server-side)  
**Team:** Cache Kings  
**Scope:** OpenWeather integration and conditions routes — `/api/conditions`, `/trail-checker/results` — against a running Flask app (not pytest).

## 1. Definition

End-to-end for the server-side slice means: a real HTTP client hits the **deployed Flask app**, which calls **live OpenWeather** (geocoding, current weather, air pollution), and returns JSON or HTML that matches `CONTRACTS.md`. Boundaries exercised: **HTTP client → Flask routes → `weather_service.py` → OpenWeather APIs**. Postgres and saved-trail flows are out of scope for this slice (Nick/Liam + follow-up routes).

This walk intentionally uses **Python `requests`** inside the app container (same discipline as curl; the slim Docker image has no `curl` installed).

## 2. The walk

### Setup

**Step 1.** From repo root, ensure dependencies and env are configured:

```bash
cp .env.example .env
# Edit .env and set OPENWEATHER_API_KEY=your-key-here

docker compose up -d
```

**Step 2.** Confirm the Trail Checker app is the process on port 5000 (not another project). Hit a route only this repo implements:

```bash
docker compose exec app python -c "import requests; print(requests.get('http://127.0.0.1:5000/trail-checker').status_code)"
```

Expect `200`. If `404`, the wrong app may be bound to port 5000 on the host.

### JSON API (`/api/conditions`)

**Step 3.** Invalid input — short query:

```bash
docker compose exec app python -c "
import requests, json
r = requests.get('http://127.0.0.1:5000/api/conditions', params={'q': 'M'})
print(r.status_code, json.dumps(r.json()))
"
```

**Step 4.** Happy path — realistic outdoor location ( **must use real OpenWeather** ):

```bash
docker compose exec app python -c "
import requests, json
r = requests.get('http://127.0.0.1:5000/api/conditions', params={'q': 'Mount Rainier'})
print(r.status_code)
data = r.json()
print('ok:', data.get('ok'))
if data.get('ok'):
    d = data['data']
    print('resolved:', d.get('resolved_name'))
    print('weather keys:', list(d.get('weather', {}).keys()))
    print('aqi:', d.get('air_quality', {}).get('aqi'))
    print('recommendation:', d.get('recommendation'))
else:
    print(json.dumps(data, indent=2))
"
```

**Step 5.** Not found — nonsense location (real geocoder, empty or no match):

```bash
docker compose exec app python -c "
import requests, json
r = requests.get('http://127.0.0.1:5000/api/conditions', params={'q': 'zzzznotrealplace999'})
print(r.status_code, json.dumps(r.json()))
"
```

**Step 6.** Edge query — truthy-fixture check (pick something odd):

```bash
docker compose exec app python -c "
import requests, json
r = requests.get('http://127.0.0.1:5000/api/conditions', params={'q': 'Rainier'})
print(r.status_code)
print(json.dumps(r.json(), indent=2)[:800])
"
```

Document whether the first geocoding result is reasonable for hikers.

**Step 7.** Missing API key (error path):

```bash
docker compose run --rm --no-deps -e OPENWEATHER_API_KEY= app python -c "
import subprocess, time, requests, json
p = subprocess.Popen(['python', 'app.py'])
time.sleep(4)
r = requests.get('http://127.0.0.1:5000/api/conditions', params={'q': 'Mount Rainier'})
print(r.status_code, json.dumps(r.json()))
p.terminate()
"
```

Expect `503` / `external_api_unavailable`.

### HTML route (`/trail-checker/results`)

**Step 8.** Results page with real query (browser or requests):

```bash
docker compose exec app python -c "
import requests
r = requests.get('http://127.0.0.1:5000/trail-checker/results', params={'q': 'Mount Rainier'})
print('status', r.status_code)
print('weather-card', 'data-testid=\"weather-card\"' in r.text)
print('recommendation-badge', 'data-testid=\"recommendation-badge\"' in r.text)
"
```

With a valid API key, expect `200` and both `data-testid` markers present.

## 3. Pass criteria

- **Step 1:** `.env` contains `OPENWEATHER_API_KEY`; `docker compose up` succeeds.
- **Step 2:** `/trail-checker` returns `200` from the Trail Checker app.
- **Step 3:** Status `400`, JSON `ok: false`, `error.code == "invalid_input"`.
- **Step 4:** Status `200`, `ok: true`, `data.weather`, `data.air_quality`, and `data.recommendation` present; resolved name and coordinates look like Mount Rainier area (not empty or unrelated continent).
- **Step 5:** Status `404`, `error.code == "not_found"` (when API key is set and geocoder returns no results).
- **Step 6:** Response documented honestly — if ambiguous geocode result, note as finding.
- **Step 7:** Status `503`, `error.code == "external_api_unavailable"`.
- **Step 8:** Status `200`; HTML includes `data-testid="weather-card"` and `data-testid="recommendation-badge"`.

## 4. Execution log

Run date: 2026-05-21 (initial), **2026-05-21 re-run after key activation**  
Environment: EC2, Docker Compose, in-container `requests` via isolated `trail-checker-e2e-app` container (host port 5000 occupied by Week 5 assignment `week_5_506-app-1`; Trail Checker uses internal port 5000 only)

| Step | Result | Notes |
|------|--------|-------|
| 1 | PASS | `.env` contains a 32-character `OPENWEATHER_API_KEY`; key accepted by OpenWeather after activation window |
| 2 | PASS | `GET /trail-checker` → `200` |
| 3 | PASS | `400`, `invalid_input` for `q=M` |
| 4 | PASS | `200`, `ok: true`, `resolved_name: Mount Rainier`, `recommendation: caution`, `aqi: 3`. **Note:** geocoder returned lat/lon `38.94, -76.96` (Mid-Atlantic US), not WA Mount Rainier — see Finding 3 |
| 5 | PASS | `404`, `not_found` for `zzzznotrealplace999` |
| 6 | PASS | `200`, `q=Rainier` resolved to `Rainier, WA area` (`46.09, -122.94`) — different from step 4; see Finding 3 |
| 7 | PASS | (2026-05-20 prior run) Empty `OPENWEATHER_API_KEY` → `503` with clear message |
| 8 | PASS | `200`; `data-testid="weather-card"` and `data-testid="recommendation-badge"` present in HTML |

### Finding 1 — New OpenWeather account: dashboard shows Active, API still returns 401 (resolved)

**Symptom:** Initial run (~15 min after account creation): steps 4–6 returned `503` / OpenWeather **401 Invalid API key** even though the dashboard showed **Active**.

**Root cause:** OpenWeather [FAQ](https://openweathermap.org/faq#error401) — new API keys can take 10 minutes to 2 hours to activate after signup.

**Resolution:** Re-run after ~1 hour — geocode probe returned **200**, steps 4–6 and 8 **PASS** with live weather data.

**Lesson:** A non-empty `.env` entry and an **Active** dashboard status are not enough on day one — e2e must confirm the upstream accepts the key (step 4 is the truthy-fixture check).

### Finding 2 — Host port 5000 may not be Trail Checker

**Symptom:** `curl localhost:5000/api/conditions` on the host returned Flask 404 HTML (route not registered).

**Root cause:** Another process or older app bound to port 5000, or Compose app not running.

**Fix:** Use `docker compose up -d` for this repo, or verify with `/trail-checker` before e2e.

**Lesson:** E2E setup step should confirm app identity, not assume port 5000.

### Finding 3 — Ambiguous geocoding: "Mount Rainier" vs "Rainier"

**Symptom:** Step 4 (`q=Mount Rainier`) and step 6 (`q=Rainier`) both returned `200` but resolved to **different coordinates**.

**Data:**
- `Mount Rainier` → `38.94, -76.96` (Mid-Atlantic US, not the Washington volcano)
- `Rainier` → `46.09, -122.94` (Pacific Northwest, closer to the expected trailhead)

**Root cause:** Contract MVP uses the **first geocoding result** with no disambiguation UI (`CONTRACTS.md` §8).

**Fix:** Not required for Week 6. Future improvement: let users pick among geocoding matches, or prefer outdoor/trail-related result types.

**Lesson:** Live e2e caught a real product gap that mocked tests with fixed fixtures would miss — short or ambiguous place names may not resolve to the hiker's intended location.

## 5. Re-run checklist

Live OpenWeather steps completed 2026-05-21 after key activation. If re-running later:

1. Recreate the e2e app container so it reloads `.env`:
   ```bash
   docker rm -f trail-checker-e2e-app
   docker compose run -d --name trail-checker-e2e-app -e SECRET_KEY=e2e-test-secret-not-default app python app.py
   ```
2. Confirm geocode probe returns **200** before steps 4–8
3. Re-run steps 4, 5, 6, 8 from §2

**Note:** If host port 5000 is occupied (e.g. by a Week 5 assignment container), use the isolated `trail-checker-e2e-app` container above — it exercises internal port 5000 without conflicting on the host.

## 6. Per-role note

This file is the **server-side** contribution to the team `e2e.md`. Coordinator should link or merge sections from Liam (browser UI), Nick (Postgres/auth), and this doc for the whole-system walk.
