# RideIQ — Two-Wheeler Driving Behavior Analytics

> **Live simulation prototype** for real-time two-wheeler riding behavior scoring, built for **Varroc Eureka · PS3**.

---

## Overview

RideIQ is a browser-based simulation dashboard that models how telematics data from a two-wheeler (speed, acceleration jerk, lean angle) can be processed into a real-time **Driving Behavior Score**. The prototype demonstrates the full analytics pipeline — from raw sensor inputs to scored breakdowns, waveform visualizations, event logging, and downloadable PDF reports — all running entirely in the browser with no backend required.

---

## Features

### Ride Scenarios
Five pre-configured auto-simulation profiles that drive the physics engine:

| Scenario | Description | Risk Level | Speed Limit |
|---|---|---|---|
| 🛵 Smooth City Ride | Gradual acceleration, gentle braking | Low | 60 km/h |
| ⚡ Aggressive Riding | Hard braking, sharp corners, high speed | High | 50 km/h |
| 🛣️ Highway Cruise | High sustained speed, minimal events | Medium | 100 km/h |
| 🌧️ Rain / Wet Road | Slippery surface, sudden lateral slides | High | 40 km/h |
| 📦 Delivery Rider | Frequent stops, lane weaving, mixed jerk | Medium | 50 km/h |

### Manual Override
Three sliders let you directly control the sensor inputs in real time:
- **Speed** — 0 to 120 km/h
- **Acceleration Jerk (dA/dt)** — −25 to +25 m/s³
- **Lean Angle (θ)** — −50° to +50°

Activating any slider disables the auto-scenario and gives you full manual control of the simulation.

### Live Behavior Score
A circular arc dial updates continuously with:
- **Score (0–100)** — exponentially smoothed from the weighted penalty model
- **Grade label** — EXCELLENT / GOOD / AVERAGE / POOR / DANGEROUS
- **Color coding** — green (≥80), amber (60–79), red (<60)

### Score Breakdown
Five weighted components contribute to the overall score:

| Component | Weight | Penalty Trigger |
|---|---|---|
| Braking | 25% | Jerk < −15 m/s³ |
| Acceleration | 20% | Jerk > +12 m/s³ |
| Cornering | 25% | Lean angle > 35° |
| Speed | 20% | Exceeds scenario speed limit |
| Stability | 10% | Sustained jerk > 5 m/s³ |

Score formula: `Score = 100 − Σ(penaltyᵢ × wᵢ)`, smoothed with exponential decay (`α = 0.15`).

### Live Sensor Panel
Right panel displays real-time readings with color-coded progress bars:
- GPS Speed (km/h)
- Jerk — rate of change of acceleration (m/s³)
- Lean Angle θ (degrees)

### Event Flash Alerts
On-canvas overlay that fires for detected events:
- ⚠ HARSH BRAKE
- ⚠ SHARP CORNER
- ⚠ HARD ACCEL
- ⚠ OVER SPEED

### Event Log
Scrollable live log at the bottom of the screen. Each entry captures:
- Timestamp (MM:SS)
- Event type (color-coded)
- Dot indicator

### Waveform Charts
Two canvas-drawn real-time charts in the bottom bar:
- **Jerk Waveform** — oscilloscope-style with brake/accel threshold lines
- **Score Trend** — color-shifting line chart of score over time (last 300 samples)

### Day / Night Theme Toggle
Header toggle switches between:
- **Night mode** — dark navy UI, neon green accents, glowing rim lights and underglow on the bike, starfield sky, city skyline
- **Day mode** — clean white/gray UI, muted accents, bright sky, clouds, treeline, natural road

Both themes are implemented via CSS variable overrides on `body.day` — no JS style manipulation needed beyond class toggling. Canvas scenes are fully re-rendered for each theme.

### PDF Report Download
Click **Download Report (PDF)** at any point during or after a simulation. Generates a 2-page A4 PDF using [jsPDF](https://github.com/parallax/jsPDF):

**Page 1 — Simulation Summary**
- Score hero card (final score, grade, avg score, min score, duration, event count)
- Peak Speed / Max Jerk / Max Lean metric cards
- Score breakdown with per-component bar charts
- Event summary count tiles (Braking / Accel / Cornering / Speed)
- Scenario detail row (name, risk, speed limit, mode)
- Score trend sparkline from the live history buffer

**Page 2 — Event Log Detail**
- Full chronological event table (Time · Event · Speed · Score · Jerk · Lean)
- Color-coded rows by event severity
- Contextual riding recommendations based on event frequency
- Auto-generated filename: `RideIQ_Report_<scenario>_<YYYYMMDD>_<HHMM>.pdf`

---

## Getting Started

No build step or server is required. The entire prototype is a single self-contained HTML file.

```bash
# Clone or download the file
# Then just open it in any modern browser:
open rideiq_enhanced.html
```

Or drag and drop `rideiq_enhanced.html` into Chrome, Firefox, Edge, or Safari.

### Requirements
- A modern browser with Canvas 2D API support (Chrome 90+, Firefox 88+, Safari 14+, Edge 90+)
- Internet connection on first load (for Google Fonts and jsPDF CDN)
- No npm, no build tools, no dependencies to install

---

## Usage

1. **Select a scenario** from the left panel — the simulation starts paused
2. **Click "Start Simulation"** — the road scene animates, score updates in real time
3. **Watch the score dial**, waveforms, and event log as driving behavior evolves
4. **Switch scenarios** at any time — score and log reset automatically
5. **Use manual sliders** to override sensor values and explore edge cases
6. **Toggle Day / Night** with the header switch for different visual contexts
7. **Click "Download Report (PDF)"** after running for a few seconds to export the full analytics report

---

## File Structure

```
rideiq_enhanced.html     # Complete standalone prototype (single file)
README.md                # This document
```

All CSS, JavaScript, canvas rendering, and PDF generation live inside `rideiq_enhanced.html`. External resources loaded via CDN:

| Resource | Source | Purpose |
|---|---|---|
| Space Grotesk | Google Fonts | Body / UI typography |
| Bebas Neue | Google Fonts | Display / score numbers |
| JetBrains Mono | Google Fonts | Monospace data values |
| jsPDF 2.5.1 | cdnjs.cloudflare.com | Client-side PDF generation |

---

## Architecture

### Simulation Engine
The physics simulation runs inside `requestAnimationFrame` at ~60 fps. Each tick:

1. Target speed is computed as a sine wave within the scenario's speed range
2. Jerk and lean values are perturbed stochastically — burst events fire at scenario-specific probabilities (`jerkBurst`, `leanBurst`)
3. Values are clamped to physical limits (jerk: ±25 m/s³, lean: ±50°)
4. `calcScore()` computes per-component penalties and exponentially smooths the overall score
5. `updateUI()` pushes all values to the DOM and canvas
6. `logEvent()` records structured events and updates report tracking state

### Scoring Model
```
brakePen  = clamp(25 × (|jerk| − 15) / 10,  0, 25)   if jerk < −15
accelPen  = clamp(20 × (jerk − 12) / 13,     0, 20)   if jerk > +12
cornerPen = clamp(25 × (|lean| − 35) / 15,   0, 25)   if |lean| > 35
speedPen  = clamp(20 × (speed − limit) / 30, 0, 20)   if speed > limit
stabPen   = clamp(10 × (|jerk| − 5) / 20,    0, 10)   if |jerk| > 5

raw   = (25 − brakePen) + (20 − accelPen) + (25 − cornerPen)
      + (20 − speedPen) + (10 − stabPen)
score = score × 0.85 + raw × 0.15    ← exponential smoothing
```

### Canvas Rendering
Two separate road drawing functions share the same bike rendering logic:
- `drawRoadNight()` — dark asphalt, starfield, city skyline, neon wheel rims, underglow
- `drawRoadDay()` — bright sky, clouds, sun, treeline, natural asphalt

Theme state (`isDay`) is read at render time, so switching themes is instantaneous even mid-simulation.

### PDF Generation
`downloadReport()` uses jsPDF's UMD bundle loaded from cdnjs. The report is built programmatically using low-level `doc.rect()`, `doc.text()`, `doc.line()` calls — no HTML-to-PDF conversion. All layout coordinates are computed in millimetres on A4 (210 × 297 mm).

---

## Scoring Reference

| Score Range | Grade | Color |
|---|---|---|
| 90 – 100 | EXCELLENT | Green |
| 75 – 89 | GOOD | Green |
| 60 – 74 | AVERAGE | Amber |
| 40 – 59 | POOR | Red |
| 0 – 39 | DANGEROUS | Red |

---

## Known Limitations

- **Single-session only** — no data persistence between page reloads
- **Simulated data** — sensor values are stochastically generated, not from real hardware
- **PDF requires CDN** — jsPDF loads from cdnjs; an offline build would need the library bundled locally
- **Fixed viewport** — layout is optimised for 1280×800 and wider; mobile layout is not supported in this prototype version
- **Event log cap** — the on-screen log shows the last 30 entries; the PDF report shows up to 55

---

## Roadmap / Next Steps

- [ ] Real Bluetooth / OBD-II sensor integration via Web Bluetooth API
- [ ] Trip history with persistent storage (IndexedDB)
- [ ] Multi-rider comparison view
- [ ] GPS map overlay with event markers
- [ ] Insurance scoring integration and gamification (badges, streaks)
- [ ] Mobile-responsive layout for tablet/phone dashboard use

---

## Credits

Developed as part of the **Varroc Eureka PS3** initiative.

Built with vanilla HTML, CSS, and JavaScript — no frameworks.
PDF generation powered by [jsPDF](https://github.com/parallax/jsPDF) (MIT License).
Typography by [Google Fonts](https://fonts.google.com).

---

*RideIQ — Making every ride smarter, safer, and more insightful.*
