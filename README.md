# Air Traffic Conflict Detector

You are building a 2D top-down air traffic control simulator in the browser. Controllers add aircraft to the airspace, watch them move in simulated time, and receive automated conflict alerts with resolution suggestions. Everything runs client-side — no server, no database.

## Stack

- **Frontend**: Pure React, port **3000**
- **Animations**: GSAP for all animated effects (aircraft rings, conflict pulses, etc.)
- **Persistence**: `localStorage` only — no backend

## Coordinate System & Units

The airspace is a **1000 × 1000 nautical-mile (nm) square**. The origin `(0, 0)` is the south-west corner. North is the **+Y** direction; East is **+X**. Altitudes are in **feet**; flight levels (FL) are altitude in hundreds of feet (e.g. 35,000 ft = FL350). Speeds are in **knots** (nautical miles per hour). Headings follow aviation convention: **0° = North, 90° = East**, clockwise.

---

## Airspace Canvas

Render a radar-style top-down view of the airspace. The display area represents the full 1000 × 1000 nm square.

- Container: `data-testid="radar"`
- Each aircraft is drawn as a **filled triangle** pointing along its current heading.
- Alongside every aircraft, a text label shows: `{callsign} FL{flightlevel} {speed}kt` — for example, `AAL1 FL350 480kt`.
- Each aircraft element: `data-testid="aircraft-{callsign}"`
- When the simulation runs, aircraft positions update every animation frame. The position update rule is:

  ```
  x_new = x + speed * sin(heading_rad) * dt / 3600
  y_new = y + speed * cos(heading_rad) * dt / 3600
  ```

  where `dt` is the elapsed real-world seconds since the last frame, and the division by 3600 converts knots (nm/hr) to nm/s.

---

## Aircraft Form

A form for entering new aircraft into the airspace. `data-testid="add-aircraft-form"`

Fields:
- **Callsign** — alphanumeric identifier (e.g. `AAL1`, `UAL2`)
- **X position** — east-west coordinate in nm
- **Y position** — north-south coordinate in nm
- **Altitude** — in feet (e.g. `35000`)
- **Heading** — degrees (0–359), clockwise from North
- **Ground speed** — knots

On submit, the aircraft is placed at its specified coordinates at the current simulation time and immediately appears on the radar.

---

## Simulation Clock

A persistent clock tracking elapsed simulated seconds since the simulation began.

- Display element: `data-testid="sim-clock"` — shows elapsed time in seconds
- `data-testid="play-btn"` — starts the simulation running in real time
- `data-testid="pause-btn"` — freezes the simulation (aircraft stop moving, clock stops)
- `data-testid="step-btn"` — advances simulation by exactly **10 seconds** (aircraft positions jump forward, clock increments by 10)
- `data-testid="clear-all-btn"` — removes **all** aircraft from the airspace and resets the sim clock to `t = 0`

When playing, the simulation clock advances continuously in real time and aircraft positions update each animation frame.

---

## Conflict Detection

The system continuously evaluates all aircraft pairs for predicted conflicts. This evaluation runs at every simulation tick and whenever an aircraft is added or modified.

### Definition of a Conflict

Two aircraft are **in conflict** if both of the following are true:

1. **Vertical separation** is less than **1000 ft** — i.e. `|alt_A - alt_B| < 1000`. Since aircraft do not climb or descend, this check is static.
2. **Minimum horizontal separation** during the **look-ahead window** (the next **10 minutes / 600 seconds** of simulated time) falls below **5 nm**.

### Closest-Point-of-Approach (CPA) Calculation

The horizontal CPA must be computed **analytically** from the current relative position and relative velocity vectors. Do not approximate by stepping through time at discrete intervals — the math must find the exact geometric minimum distance between the two linear trajectories.

- If the two aircraft are flying parallel courses (zero relative velocity), their separation never changes — there is no CPA to evaluate, so no conflict is raised.
- If the aircraft are diverging (relative velocity is pointed away from each other), they will not converge — no conflict.
- If the time of minimum separation falls **outside** `[0, 600]` seconds from the current moment, no conflict is raised even if the minimum distance is small.

---

## Conflict Panel

`data-testid="conflicts"` — a list of all active predicted conflicts, sorted ascending by **time-to-CPA**.

Each conflict entry:
- `data-testid="conflict-{callsignA}-{callsignB}"` where the two callsigns are in **alphabetical order**
- Displayed information per entry:
  - Both callsigns
  - Time to CPA, in seconds, **rounded to 1 decimal place**
  - Minimum horizontal separation at CPA, in nm, **to 2 decimal places**
  - Closure rate in knots (the rate at which horizontal distance is decreasing)

The panel updates live on every simulation tick and on any aircraft state change.

---

## Visual Conflict Indicators

- Aircraft **not** involved in any conflict: rendered with a **steady gray ring** around the triangle
- Aircraft **in conflict**: GSAP animates a **pulsing red ring** around the triangle
- For each conflicting pair, a **dashed red line** is drawn between the two aircraft on the radar

All animations use GSAP.

---

## Resolution Suggestions

Clicking a conflict entry opens a resolution panel for that pair.

`data-testid="resolve-{callsignA}-{callsignB}"` — the clickable conflict entry that opens the resolution view

The system finds the **minimum heading change for aircraft A** (alphabetically first callsign) that would push the predicted minimum horizontal separation to **≥ 5 nm** within the look-ahead window, while leaving aircraft B's heading unchanged.

Resolution search:
- Try heading changes of `+1°, -1°, +2°, -2°, ...` up to `±90°`
- Pick the **smallest absolute value** that resolves the conflict
- If both `+k°` and `-k°` resolve it, prefer the **right turn** (`+k°`) per aviation convention
- If no turn within `±90°` resolves the conflict: display `Suggest: descend {callsignA}`
- Otherwise display: `Suggest: turn {callsignA} {direction} {n}°` where direction is `left` or `right`

Resolution suggestion display: `data-testid="resolve-suggestion"`

`data-testid="apply-resolve-btn"` — applies the suggested heading change immediately to aircraft A. After applying, the conflict panel updates.

---

## Aircraft Editing

Clicking an aircraft on the radar **selects** it.

- The selected aircraft is shown in `data-testid="selected-aircraft"`
- An editing panel exposes inputs to modify:
  - **Heading** (degrees)
  - **Ground speed** (knots)
- Changes apply live: the triangle rotates to the new heading instantly, and the conflict panel re-evaluates

---

## Persistence

On every state change (aircraft added, edited, sim time advanced, aircraft removed), save the full airspace state and current sim time to `localStorage`. On page load, restore all aircraft at their saved positions and resume the sim clock from the saved time.

After a page reload, the radar, aircraft labels, and sim clock must all reflect the previously saved state.

---

## `data-testid` Reference

### Radar & Aircraft

| Element | `data-testid` |
|---|---|
| Airspace canvas container | `radar` |
| Each aircraft | `aircraft-{callsign}` |

### Aircraft Form

| Element | `data-testid` |
|---|---|
| Add aircraft form | `add-aircraft-form` |

### Simulation Controls

| Element | `data-testid` |
|---|---|
| Elapsed time display | `sim-clock` |
| Play button | `play-btn` |
| Pause button | `pause-btn` |
| Step (+10s) button | `step-btn` |
| Clear all aircraft & reset clock | `clear-all-btn` |

### Conflict Panel

| Element | `data-testid` |
|---|---|
| Conflict list container | `conflicts` |
| Individual conflict entry | `conflict-{callsignA}-{callsignB}` (alphabetical) |
| Resolution trigger (same as entry) | `resolve-{callsignA}-{callsignB}` |
| Resolution suggestion text | `resolve-suggestion` |
| Apply resolution button | `apply-resolve-btn` |

### Aircraft Editor

| Element | `data-testid` |
|---|---|
| Selected aircraft panel | `selected-aircraft` |