<template>
  <div class="app">
    <header class="topbar">
      <div class="title">Inclusive Maps (Open-source stack)</div>
      <div class="hint">
        Click map: <b>Start</b> → <b>Destination</b> → route draws. Third click resets.
      </div>
      <div class="small" v-if="routeInfo">
        Distance: {{ (routeInfo.distance / 1000).toFixed(2) }} km · Duration: {{ (routeInfo.duration / 60).toFixed(0) }} min
      </div>
      <div v-if="routeError" class="error-banner" role="status" aria-live="polite">
        {{ routeError }}
      </div>
    </header>

    <div ref="mapContainer" class="map"></div>
  </div>
</template>

<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref } from 'vue'
import maplibregl, { type GeoJSONSource, type Map } from 'maplibre-gl'
import type { Feature, FeatureCollection, LineString, Point } from 'geojson'

type LngLat = { lng: number; lat: number }
type RouteInfo = { distance: number; duration: number }
type PointRole = 'start' | 'dest'
type PointFeature = Feature<Point, { role: PointRole }>
type PointFeatureCollection = FeatureCollection<Point, { role: PointRole }>
type RouteFeature = Feature<LineString, RouteInfo>
type DecisionNodeFeature = Feature<Point, { idx: number }>
type DecisionNodeCollection = FeatureCollection<Point, { idx: number }>
type CoordinatePair = [number, number]

interface OsrmStep {
  maneuver?: {
    location?: CoordinatePair
  }
}

interface OsrmRoute {
  geometry?: {
    coordinates?: number[][]
  }
  distance: number
  duration: number
  legs?: Array<{
    steps?: OsrmStep[]
  }>
}

interface OsrmRouteResponse {
  routes?: OsrmRoute[]
}

const mapContainer = ref<HTMLDivElement | null>(null)
let map: Map | null = null

const start = ref<LngLat | null>(null)
const dest = ref<LngLat | null>(null)
const routeInfo = ref<RouteInfo | null>(null)
const routeError = ref<string | null>(null)

const OSRM_BASE = 'https://router.project-osrm.org'
const PROFILE = 'driving' // reliable; later you can try 'walking' with your own OSRM instance

const SRC_POINTS = 'points-src'
const LYR_POINTS = 'points-layer'
const SRC_ROUTE = 'route-src'
const LYR_ROUTE = 'route-layer'
const SRC_NODES = 'decision-nodes-src'
const LYR_NODES = 'decision-nodes-layer'

function makePointsFC(): PointFeatureCollection {
  const features: PointFeature[] = []
  if (start.value) {
    features.push({
      type: 'Feature',
      properties: { role: 'start' },
      geometry: { type: 'Point', coordinates: [start.value.lng, start.value.lat] }
    })
  }
  if (dest.value) {
    features.push({
      type: 'Feature',
      properties: { role: 'dest' },
      geometry: { type: 'Point', coordinates: [dest.value.lng, dest.value.lat] }
    })
  }
  return { type: 'FeatureCollection', features }
}

function setPoints() {
  const src = map?.getSource(SRC_POINTS) as GeoJSONSource | undefined
  src?.setData(makePointsFC())
}

function clearRoute() {
  const src = map?.getSource(SRC_ROUTE) as GeoJSONSource | undefined
  const emptyRouteCollection: FeatureCollection<LineString> = { type: 'FeatureCollection', features: [] }
  src?.setData(emptyRouteCollection)
  routeInfo.value = null
}
function clearDecisionNodes() {
  const src = map?.getSource(SRC_NODES) as GeoJSONSource | undefined
  const emptyDecisionNodes: DecisionNodeCollection = { type: 'FeatureCollection', features: [] }
  src?.setData(emptyDecisionNodes)
}

function setDecisionNodes(points: CoordinatePair[]) {
  const features: DecisionNodeFeature[] = points.map((c, idx) => ({
    type: 'Feature',
    properties: { idx },
    geometry: { type: 'Point', coordinates: c }
  }))
  const src = map?.getSource(SRC_NODES) as GeoJSONSource | undefined
  const collection: DecisionNodeCollection = { type: 'FeatureCollection', features }
  src?.setData(collection)
}
function setRoute(feature: Feature<LineString>) {
  const src = map?.getSource(SRC_ROUTE) as GeoJSONSource | undefined
  const collection: FeatureCollection<LineString> = { type: 'FeatureCollection', features: [feature] }
  src?.setData(collection)
}

function fitToLineCoords(coords: [number, number][]) {
  let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity
  for (const [x, y] of coords) {
    minX = Math.min(minX, x)
    minY = Math.min(minY, y)
    maxX = Math.max(maxX, x)
    maxY = Math.max(maxY, y)
  }
  map?.fitBounds([[minX, minY], [maxX, maxY]], { padding: 70, duration: 600 })
}
async function fetchRoute(a: LngLat, b: LngLat) {
  const coords = `${a.lng},${a.lat};${b.lng},${b.lat}`
  const url = `${OSRM_BASE}/route/v1/${PROFILE}/${coords}?overview=full&geometries=geojson&steps=true`

  const res = await fetch(url)
  if (!res.ok) throw new Error(`OSRM error: ${res.status}`)
  const data: OsrmRouteResponse = await res.json()
  const r = data?.routes?.[0]
  if (!r?.geometry?.coordinates) throw new Error('No route returned')
  const routeCoords = r.geometry.coordinates.filter(
    (coord): coord is CoordinatePair => Array.isArray(coord) && coord.length === 2
  )
  if (!routeCoords.length) throw new Error('No route returned')

  routeInfo.value = { distance: r.distance, duration: r.duration }

  // Decision nodes = maneuver points (one per step)
  const steps = data?.routes?.[0]?.legs?.[0]?.steps ?? []
  const decisionNodes: CoordinatePair[] = steps
    .map((s) => s?.maneuver?.location)
    .filter((loc): loc is CoordinatePair => Array.isArray(loc) && loc.length === 2)

  const routeFeature: RouteFeature = {
    type: 'Feature',
    properties: { distance: r.distance, duration: r.duration },
    geometry: { type: 'LineString', coordinates: routeCoords }
  }

  return {
    routeFeature,
    decisionNodes,
    routeCoords
  }
}

function ensureLayers() {
  if (!map) return

  // Points source/layer
  if (!map.getSource(SRC_POINTS)) {
    map.addSource(SRC_POINTS, {
      type: 'geojson',
      data: makePointsFC()
    })
    map.addLayer({
      id: LYR_POINTS,
      type: 'circle',
      source: SRC_POINTS,
      paint: {
        'circle-radius': 8,
        'circle-stroke-width': 2
      }
    })
  }

  // Route source/layer
  if (!map.getSource(SRC_ROUTE)) {
    map.addSource(SRC_ROUTE, {
      type: 'geojson',
      data: { type: 'FeatureCollection', features: [] }
    })
    map.addLayer({
      id: LYR_ROUTE,
      type: 'line',
      source: SRC_ROUTE,
      paint: {
        'line-width': 6
      }
    })
  }
  // Decision nodes source/layer
if (!map.getSource(SRC_NODES)) {
  map.addSource(SRC_NODES, {
    type: 'geojson',
    data: { type: 'FeatureCollection', features: [] }
  })
  map.addLayer({
    id: LYR_NODES,
    type: 'circle',
    source: SRC_NODES,
    paint: {
      'circle-radius': 5,
      'circle-stroke-width': 2
    }
  })
}
}

function resetAll() {
  start.value = null
  dest.value = null
  routeError.value = null
  setPoints()
  clearRoute()
  clearDecisionNodes()
}

onMounted(() => {
  if (!mapContainer.value) return

  map = new maplibregl.Map({
    container: mapContainer.value,
    style: {
      version: 8,
      sources: {
        osm: {
          type: 'raster',
          tiles: ['https://a.tile.openstreetmap.org/{z}/{x}/{y}.png'],
          tileSize: 256,
          attribution: '© OpenStreetMap contributors'
        }
      },
      layers: [{ id: 'osm-layer', type: 'raster', source: 'osm' }]
    },
    center: [9.19, 45.46],
    zoom: 12
  })

  map.addControl(new maplibregl.NavigationControl(), 'top-right')

  map.on('load', () => {
    ensureLayers()
    setPoints()
    clearRoute()
    clearDecisionNodes()
  })

  map.on('click', async (e) => {
    const p: LngLat = { lng: e.lngLat.lng, lat: e.lngLat.lat }

    // Click flow: Start -> Destination -> Reset
    if (!start.value) {
      start.value = p
      setPoints()
      clearRoute()
      clearDecisionNodes()
      return
    }

    if (!dest.value) {
      dest.value = p
      setPoints()
      clearRoute()
      routeError.value = null

      try {
        const { routeFeature, decisionNodes, routeCoords } = await fetchRoute(start.value, dest.value)
        setRoute(routeFeature)
        setDecisionNodes(decisionNodes)
        fitToLineCoords(routeCoords)
      } catch (err: any) {
        console.error(err)
        routeError.value = 'Unable to calculate a route right now. Please try again.'
        clearRoute()
        clearDecisionNodes()
      }
      return
    }

    resetAll()
  })
})

onBeforeUnmount(() => {
  map?.remove()
  map = null
})
</script>

<style scoped>
.app {
  height: 100vh;
  display: flex;
  flex-direction: column;
  font-family: system-ui, -apple-system, sans-serif;
}
.topbar {
  padding: 10px 14px;
  border-bottom: 1px solid #e0e0e0;
  background: #ffffff;
}
.title {
  font-weight: 600;
  font-size: 16px;
}
.hint {
  font-size: 13px;
  color: #666;
  margin-top: 2px;
}
.small {
  font-size: 12px;
  color: #444;
  margin-top: 6px;
}
.error-banner {
  margin-top: 6px;
  padding: 6px 8px;
  font-size: 12px;
  color: #7a1f1f;
  background: #fce8e8;
  border: 1px solid #f2bcbc;
  border-radius: 4px;
}
.map {
  flex: 1;
}
</style>
