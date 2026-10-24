# Live deployment animation (ArcGIS Online dashboard embed)

A single web page that reads a hosted node-deployment layer **live**, and plays it back day by day: each new node flashes white-hot, cools through yellow and lime, and settles to the dashboard node green (#55FF00), while a date counter, running totals and a play/pause time scrubber (with a line of nodes per day that draws itself as time moves) track progress, and each new day's working areas "ping" on the map with sonar ripples. It loops forever, and re-queries the layer between loops, so it grows by one day every day with no one touching it.

- `index.html`: the animation (ArcGIS Maps SDK for JavaScript 5.1, no build step)
- `oauth-callback.html`: sign-in popup helper (only used if the layer stays private)

Preview without any data or sign-in: open `index.html?demo=1` (synthetic grid in the Chaco).

---

## How it stays live

1. On load, the page queries every feature from the layer (paged, 4 requests in parallel).
2. Each record's `local_time` becomes a fractional project day (day number + time of day), so nodes "light up" across the working day the way the crews actually laid them.
3. After each loop, if the data is older than `refreshMinutes` (20 min), the page re-queries before the next loop.
4. The page also reloads itself every `reloadHours` (12 h) so wall screens pick up code changes.

Your daily deployment notebook appends to the hosted layer → the next loop shows the new day. Nothing else to schedule.

## Design choices

| Choice | Why |
|---|---|
| Pan camera at one fixed zoom | Follows the crews' work day to day without zooming; `camera=static` keeps the whole survey in view |
| White → yellow → lime → light green "cooling" ramp | Makes the *new* stand out (avoids change blindness) while the history stays readable; light green matches the nodes on the dashboard map |
| Bloom on bright pixels only | Firefly glow on fresh nodes, crisp settled grid |
| Faint ghost of the full footprint | Viewer sees where the spread is heading |
| Time scrubber with a self-drawing daily line, month marks + glowing playhead | A temporal legend (Flourish-style): the eye always knows *when* it is; click to jump to any day |
| Sonar ripples where each day's crews start | Draws the eye to where work moved that day, without another chart |
| Hold at start and end | Time to read the final state before the loop restarts |
| Day boundary from `utc_time` converted to America/La_Paz | Bolivia calendar days, no UTC-midnight drift |

---

## Setup

### Step 1. Decide how the page gets the data

The service (`a30934`) is **secured** (it returns "Token Required"), so pick one:

**Option A: Public hosted view (zero clicks for viewers).**
Create a hosted feature layer *view* of `Bolivia_3D_Deployed_Nodes_Accepted`, limit its fields to `local_time`, share the view publicly, and put the view's URL in `layerUrl` (or pass `?layerUrl=`).
Node coordinates then become readable by anyone who has the URL, so confirm with the PM/client that this is acceptable before doing it.

**Option B: Keep it private, sign in once (recommended for client data).**
1. In ArcGIS Online, create an OAuth 2.0 developer credential / application item (Content → New item → Developer credentials).
2. Add redirect URLs for your hosted page, e.g. `https://<account>.github.io/explor-live-maps/` and `https://<account>.github.io/explor-live-maps/oauth-callback.html`.
3. Copy the **Client ID** and add it to the embed URL: `&appId=<CLIENT_ID>` (or set `appId` in `DEFAULTS`).

Viewers who can already see the layer click **Sign in to ArcGIS Online** once; the token is kept in the browser for about two weeks.

### Step 2. Host the two files (GitHub Pages)

1. Create a **public** GitHub repository, e.g. `explor-live-maps` (the code holds no secrets; an OAuth Client ID is public by design).
2. Upload `index.html` and `oauth-callback.html` to the repository root.
3. Settings → Pages → Deploy from branch → `main` / root.
4. After a minute the page is live at `https://<account>.github.io/explor-live-maps/`.

Any HTTPS web server works the same way; GitHub Pages is just the free option.

### Step 3. Embed in the dashboard

Edit the dashboard → add element → **Embedded content** → URL:

```
https://<account>.github.io/explor-live-maps/?project=bolivia3d
```

(Add `&appId=<CLIENT_ID>` for Option B.) Give the element a decent height; the HUD trims itself on small tiles (below 340 px tall the scrubber hides).

### Step 4. Check it once

Open the URL with `&debug=1` in a normal browser tab. The panel shows the date field used, the day basis, record count, and first/last dates. Confirm the first and last dates match what you expect from the field crews.

---

## URL parameters

Every setting can be overridden in the URL, e.g. `?project=bolivia3d&basemap=imagery&secondsPerDay=0.3`.

| Parameter | Default | Meaning |
|---|---|---|
| `project` | `bolivia3d` | Preset to use (see `PRESETS` in `index.html`) |
| `layerUrl` | preset | FeatureServer layer URL (include `/0`) |
| `appId` | empty | OAuth Client ID (Option B) |
| `dateField` | `utc_time` | Field that sequences the animation; converted to Bolivia local days (`timeZone`). Falls back to `local_time`, then the first date field |
| `tzMode` | `auto` | `wallclock` = stored value is local clock time; or an IANA zone such as `America/La_Paz` for true UTC fields |
| `nodeColor` | `55FF00` | Theme colour as hex (settled nodes, title, day counter, scrubber line), e.g. `7ED957`, to match the dashboard map |
| `brightness` | `0.85` | Satellite basemap brightness (1 = original, lower = darker) |
| `basemap` | `imagery` | `imagery` (dimmed satellite), `hillshade`, or `dark` |
| `labels` | `true` | Faint reference labels |
| `ghost` | `true` | Faint footprint of all nodes |
| `secondsPerDay` | `1.3` | Playback speed (higher = slower) |
| `minDuration` / `maxDuration` | `12` / `90` | Bounds on total loop length in seconds (keeps long projects watchable) |
| `holdStart` / `holdEnd` | `1.5` / `6` | Pauses before and after each run, in seconds |
| `refreshMinutes` | `20` | Re-query between loops when data is older than this |
| `reloadHours` | `12` | Full page reload interval |
| `nodeSize` / `flashSize` | `3.2` / `11` | Maximum marker sizes in px |
| `autoSize` | `true` | Shrinks markers on dense / small views so the line texture stays visible |
| `target` | `56588` | Planned total deployments for the progress ring (0 hides it) |
| `panX` | `0.06` | Nudges the survey right (+) or left (-), as a fraction of the free map width |
| `outlineUrl` | Ibibobo outline | Survey boundary drawn on the map and used for a steady frame (`""` = none) |
| `outlineSeconds` / `outlinePause` / `camMoveSeconds` | `2.5` / `1.2` / `1.8` | Opening: outline traces on the whole survey, pauses, then the camera moves in to day 1 |
| `scrim` | `0.55` | Darkness of the soft band behind the left text (`0` = off, `0.8` = strong) |
| `scaleKm` | `5` | Scale bar length in km (`0` = automatic) |
| `camera` | `pan` | `pan`: one fixed zoom, gliding from each day's work area to the next (zooms in once at loop start, out once at the end); `static`: whole survey, no motion |
| `camSecondsPerDay` / `camMaxDuration` | `1.8` / `100` | Playback speed and longest loop while the camera pans |
| `panZoom` | `2` | How much closer than the whole-survey view the pan camera sits |
| `padding` | `0.03` | Margin around the survey as a fraction of its size; raise (e.g. `0.15`) to zoom out |
| `rotation` | `0` | Map rotation in degrees, to square up the survey grid |
| `interactive` | `false` | Allow pan/zoom (off so scrolling the dashboard is not captured) |
| `demo` | `false` | Synthetic data, no sign-in |
| `debug` | `true` | Metadata block under the counters (source, records, dates); `?debug=0` hides it |

Use the round button (or Space) to pause and resume; click anywhere on the scrubber to jump to that day.

## Adding another project

Copy the `bolivia3d` block in `PRESETS`, give it a new key, change `layerUrl`, `eyebrow`, `title` (e.g. "Source acquisition") and `timeZone`, commit, and embed with `?project=<new key>`. Give each dataset its own theme with `&nodeColor=<hex>` (the title, day counter and scrubber follow it). One hosted page serves every project.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| "Could not load… token / not authorized" | Layer is private and no `appId` was given, or the signed-in account cannot see the layer |
| Sign-in popup blocked | Allow popups for the hosting site; the button must be clicked (not automatic) |
| Dates one day off | Field is a true UTC instant: use `&tzMode=America/La_Paz`, or switch to `&dateField=local_time` |
| Blank map, no basemap | Network blocks `services.arcgisonline.com`; nodes still animate |
| Very slow first load | Large layer; consider a view limited to the fields above |
