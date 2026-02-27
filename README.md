# Inclusive Maps

A Vue 3 + TypeScript app that renders an interactive OpenStreetMap basemap with MapLibre GL and computes routes through the public OSRM demo API.

## What it does

- Lets you click once to set a **start** point.
- Lets you click again to set a **destination** point.
- Requests a route from OSRM and draws it on the map.
- Displays route **distance** and **estimated duration**.
- Highlights maneuver/decision nodes from route steps.
- A third click resets start, destination, and route so you can begin again.

## Stack

- Vue 3 + Vite + TypeScript
- MapLibre GL JS
- OpenStreetMap raster tiles
- OSRM demo routing service (`https://router.project-osrm.org`)

## Development

```bash
npm install
npm run dev
```

Then open the local Vite URL shown in your terminal.
