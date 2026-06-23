# Inclusive Maps — Adaptive Navigation Prototype

A mobile-first web application prototype developed as part of a course project at **Politecnico di Milano**. The project investigates how adaptive UI overlays can support users with intellectual disabilities during pedestrian navigation, by detecting hesitation at decision points and providing contextual assistance.

Built on open-source technologies as an alternative to Google Maps, using **Vue 3**, **MapLibre GL**, and **OpenStreetMap**-based services.

---

## Project Context

Navigation applications like Google Maps are optimised for route efficiency but assume stable cognitive capacity and high tolerance for uncertainty. This prototype explores targeted UI interventions at the moments where users struggle most: decision points, route changes, and arrival.

The system does not attempt to detect internal emotional states. Instead, it infers potential uncertainty through observable behavioural patterns — walking speed reduction and stationary pauses near intersections — and responds with a progressive, supportive overlay.

This prototype is a research artifact intended to test the feasibility of behaviour-triggered UI adaptation. It implements a transparent, rule-based mechanism without machine learning or biometric data.

---

## Features

### Normal Navigation
- Address search with autocomplete (Nominatim, biased to Milan)
- Pedestrian routing via OSRM
- Real-time GPS tracking with map following
- Top navigation bar showing direction, time, and distance

### Route Preview
- Step-by-step maneuver list before navigation begins
- Time and distance summary
- "Inizia" button zooms map to first decision point

### Hesitation Detection & Adaptive Overlay
Three-stage progressive overlay triggered when the user slows down or stops near a decision node:
- **Stage 0** — gentle prompt ("Qui puoi scegliere")
- **Stage 1** — direction revealed with arrow icon
- **Stage 2** — action buttons ("Seguo questa strada" / "Guardo la mappa")

Overlay activates only once per decision node to prevent repeated triggering.

### Off-Route Detection
When the user deviates from the route, a reassurance overlay appears:
- **Stage 0** — mascot character appears
- **Stage 1** — "Va tutto bene. Sembra che il percorso sia cambiato."
- **Stage 2** — "Mostra il nuovo percorso" / "Controllo la mappa" buttons

### Arrival Detection
When the user reaches within 30m of the destination:
- Full-screen arrival state with green header ("Sei arrivato")
- Map visible through transparent body
- "Condividi viaggio" and "Esci" buttons

---

## Technical Stack

| Layer | Technology |
|---|---|
| Framework | Vue 3 with `<script setup>` + TypeScript |
| Map rendering | MapLibre GL JS |
| Base map tiles | OpenStreetMap |
| Routing | OSRM (Open Source Routing Machine) — pedestrian profile |
| Geocoding | Nominatim (OpenStreetMap) |
| Build tool | Vite |

---

## System Architecture

The system operates through four conceptual layers:

1. **Base Navigation Layer** — route guidance, GPS positioning, map rendering
2. **Behavioural Monitoring Layer** — passively observes GPS position, movement speed, proximity to decision nodes, stop duration
3. **Uncertainty Inference Layer** — rule-based heuristic to determine whether observable behaviour near a decision node may indicate hesitation
4. **Adaptive UI Overlay Layer** — progressive bottom-sheet interface that frames the decision moment and offers a suggested action

### Hesitation Detection Logic

```
IF user is within 30m of a decision node:
  IF rolling average speed < 0.1 m/s for > 4 seconds → trigger overlay
  IF rolling average speed < 0.5 m/s → trigger overlay
  ELSE → continue normal navigation
```

Speed is computed from a rolling window of 5 GPS samples. Overlay activates once per node maximum.

### State Model

```
Normal Navigation → Hesitation Evaluation → Adaptive Support Active
                  ↑                                    |
                  └────────────────────────────────────┘
                         (dismiss or move past node)
```

---

## Getting Started

### Prerequisites
- Node.js 18+
- npm

### Installation

```bash
git clone https://github.com/YOUR_USERNAME/inclusive-maps.git
cd inclusive-maps
npm install
```

### Running locally

```bash
npm run dev
```

Open `http://localhost:5173` in your browser.

### Running on a mobile device (required for GPS)

Mobile browsers require HTTPS for geolocation. To test on a real device:

```bash
npm run dev
```

Vite will display a Network URL (e.g. `https://192.168.x.x:5174`). Make sure your phone is on the same WiFi network, open that URL in Safari, and accept the certificate warning.

---

## Debug Shortcuts (Desktop)

When testing on desktop without GPS, use keyboard shortcuts to simulate states:

| Key | Action |
|---|---|
| `H` | Simulate hesitation at next decision node |
| `R` | Reset all decision nodes |
| `V` | Trigger reroute overlay |
| `A` | Trigger arrival state |

---

## Configuration

Key parameters are defined as constants in `App.vue` and can be adjusted for testing:

| Parameter | Default | Description |
|---|---|---|
| `PROXIMITY_RADIUS_M` | 30m | Distance from decision node to begin hesitation evaluation |
| `STOP_THRESHOLD_MS` | 4000ms | Duration of stop before overlay triggers |
| `SPEED_WINDOW` | 5 samples | Rolling average window for speed estimation |
| Slow speed threshold | 0.5 m/s | Speed below which overlay triggers immediately |
| Stop speed threshold | 0.1 m/s | Speed considered stationary |
| Arrival radius | 30m | Distance from destination to trigger arrival state |

---

## Known Limitations

- Routing quality depends on the public OSRM API (`router.project-osrm.org`) and may not always return the most optimal pedestrian route
- Off-route detection uses a heuristic proximity check to decision nodes, which may produce false positives in areas with dense intersections
- GPS accuracy on mobile varies; thresholds may need calibration through user testing
- The prototype does not include cross-session personalization or long-term behavioural profiling by design

---

## Project Status

This is a **research prototype** (v1), not a production application. It is intended to make design hypotheses testable and generate transferable design principles for adaptive navigation support.

**Implemented:**
- ✅ Hesitation detection and adaptive overlay
- ✅ Off-route detection and reassurance overlay  
- ✅ Route preview with step list
- ✅ Arrival detection

**Not yet implemented:**
- ⬜ User testing and threshold calibration
- ⬜ Accessibility audit
- ⬜ Multi-language support beyond Italian

---

## Academic Context

**Institution:** Politecnico di Milano  
**Programme:** Geoinformatics Engineering  
**Supervisors:** Prof. Maristella Matera, Qi Ai  
**Student:** Zhanat Argimbayeva (11059833)
