<template>
  <div class="app">

    <!-- Address Search Panel -->
    <div class="search-panel" :class="{ collapsed: searchCollapsed }">
      <button class="search-toggle" @click="searchCollapsed = !searchCollapsed">
        {{ searchCollapsed ? '🔍 Cerca' : '✕' }}
      </button>

      <div class="search-body" v-show="!searchCollapsed">
        <div class="search-field">
          <label>Da</label>
          <div class="search-input-wrap">
            <input
              v-model="originQuery"
              placeholder="La tua posizione o indirizzo…"
              @input="onOriginInput"
              @focus="activeField = 'origin'"
              autocomplete="off"
            />
            <button
              v-if="originQuery"
              class="clear-btn"
              @click="originQuery = ''; originSuggestions = []"
            >✕</button>
            <button
              v-if="speechSupported"
              class="mic-btn"
              @click="startVoiceInput('origin')"
            >{{ isListening ? '⏹' : '🎤' }}</button>
          </div>
          <ul v-if="activeField === 'origin' && originSuggestions.length" class="suggestions">
            <li
              v-for="s in originSuggestions"
              :key="s.place_id"
              @mousedown.prevent="selectOrigin(s)"
            >
              {{ s.display_name }}
            </li>
          </ul>
        </div>

        <div class="search-field">
          <label>A</label>
          <div class="search-input-wrap">
            <input
              v-model="destQuery"
              placeholder="Indirizzo di destinazione…"
              @input="onDestInput"
              @focus="activeField = 'dest'"
              autocomplete="off"
            />
            <button
              v-if="destQuery"
              class="clear-btn"
              @click="destQuery = ''; destSuggestions = []"
            >✕</button>
            <button
              v-if="speechSupported"
              class="mic-btn"
              @mousedown.prevent="startVoiceInput('dest')"
            >{{ isListening ? '⏹' : '🎤' }}</button>
          </div>
          <ul v-if="activeField === 'dest' && destSuggestions.length" class="suggestions">
            <li
              v-for="s in destSuggestions"
              :key="s.place_id"
              @mousedown.prevent="selectDest(s)"
            >
              {{ s.display_name }}
            </li>
          </ul>
        </div>

        <button
          class="btn primary wide go-btn"
          :disabled="!searchOrigin || !searchDest"
          @click="routeFromSearch"
        >
          Indicazioni
        </button>
      </div>
    </div>

    <!-- Map -->
    <div ref="mapContainer" class="map"></div>

    <!-- Top navigation bar -->
    <div v-if="routeInfo" class="nav-top" :class="{ 'nav-below-search': searchCollapsed }">
      <div class="nav-top-inner">
        <div class="nav-arrow">{{ overlayIcon }}</div>
        <div class="nav-meta">
          <span class="nav-time">{{ durationText }}</span>
          <span class="nav-distance">{{ distanceText }}</span>
        </div>
      </div>
    </div>
    <!-- Cancel route button -->
<button
  v-if="routeInfo && !showRoutePreview"
  class="cancel-route-btn"
  :class="{ 'nav-below-search': searchCollapsed }"
  @click="cancelRoute()"
>
  ✕ Annulla percorso
</button>
<div v-if="overlayVisible" class="overlay">
  <div class="overlay-card">
    <div 
      class="overlay-handle-zone"
      @touchstart="onDragStart"
      @touchmove.prevent="onDragMove"
      @touchend="onDragEnd"
      @mousedown="onHandleMouseDown"
    >
    <div class="overlay-handle"></div>
  </div>
    <button class="overlay-close-btn" @click="dismissOverlay()">✕</button>

    <Transition v-if="overlayStage === 0" name="fade">
      <div class="overlay-stage stage-0">
        <div class="stage0-pin">📍</div>
        <div class="stage0-text">Qui puoi scegliere</div>
      </div>
    </Transition>

    <Transition v-else-if="overlayStage === 1" name="slide-up">
      <div class="overlay-stage stage-1">
        <div class="overlay-subtitle">Percorso a piedi</div>
        <div class="overlay-main">
          <div class="overlay-icon">{{ overlayIcon }}</div>
          <div class="overlay-direction">{{ overlayDirection }}</div>
        </div>
      </div>
    </Transition>

    <Transition v-else-if="overlayStage === 2" name="slide-up">
      <div class="overlay-stage stage-1">
        <div class="overlay-subtitle">Percorso a piedi</div>
        <div class="overlay-main">
          <div class="overlay-icon">{{ overlayIcon }}</div>
          <div class="overlay-direction">{{ overlayDirection }}</div>
        </div>
        <div class="overlay-actions">
          <button class="btn primary wide" @click="followUser = true; dismissOverlay()">
            Seguo questa strada
          </button>
          <button class="btn text-only" @click="followUser = false; dismissOverlay()">
            Guardo la mappa
          </button>
        </div>
      </div>
    </Transition>

  </div>
</div>
<!-- Reroute overlay -->
    <div v-if="rerouteOverlayVisible" class="overlay">
      <div class="overlay-card">
        <div 
          class="overlay-handle-zone"
          @touchstart="onDragStart"
          @touchmove.prevent="onDragMoveReroute"
          @touchend="onDragEnd"
          @mousedown="onHandleMouseDownReroute"
        >
          <div class="overlay-handle"></div>
        </div>
        <button class="overlay-close-btn" @click="dismissRerouteOverlay()">✕</button>

        <Transition v-if="rerouteOverlayStage === 0" name="fade">
          <div class="overlay-stage stage-0">
            <img src="/src/assets/mascot.png" style="width:100px;height:100px;object-fit:contain;display:block;margin:0 auto 12px;" />
          </div>
        </Transition>

        <Transition v-else-if="rerouteOverlayStage === 1" name="slide-up">
          <div class="overlay-stage stage-1">
            <img src="/src/assets/mascot.png" style="width:100px;height:100px;object-fit:contain;display:block;margin:0 auto 12px;" />
            <div class="overlay-direction" style="font-size: 18px;">Va tutto bene.</div>
            <div class="overlay-subtitle" style="font-size: 15px; color: #3b3b3b;">Sembra che il percorso sia cambiato.</div>
          </div>
        </Transition>

        <Transition v-else-if="rerouteOverlayStage === 2" name="slide-up">
          <div class="overlay-stage stage-1">
            <img src="/src/assets/mascot.png" style="width:100px;height:100px;object-fit:contain;display:block;margin:0 auto 12px;" />
            <div class="overlay-direction" style="font-size: 18px;">Va tutto bene.</div>
            <div class="overlay-subtitle" style="font-size: 15px; color: #3b3b3b; margin-bottom: 16px;">Sembra che il percorso sia cambiato.</div>
            <div class="overlay-actions">
              <button class="btn primary wide" style="background: #1f6fe5;" @click="fetchAltRoute(); rerouteOverlayStage = 3">
                Mostra percorso alternativo
              </button>
              <button class="btn text-only" @click="keepOriginalRoute()">
                Continua sul percorso attuale
              </button>
            </div>
          </div>
        </Transition>

        <Transition v-else-if="rerouteOverlayStage === 3" name="slide-up">
          <div class="overlay-stage stage-1">
            <img src="/src/assets/mascot.png" style="width:100px;height:100px;object-fit:contain;display:block;margin:0 auto 12px;" />
            <div class="overlay-direction" style="font-size: 18px;">Percorso alternativo</div>
            <div class="overlay-subtitle" style="font-size: 15px; color: #3b3b3b; margin-bottom: 16px;">Vuoi usare questo percorso?</div>
            <div class="overlay-actions">
              <button class="btn primary wide" style="background: #1f6fe5;" @click="useAltRoute()">
                Usa questo percorso
              </button>
              <button class="btn text-only" @click="keepOriginalRoute()">
                Torna al percorso originale
              </button>
            </div>
          </div>
        </Transition>

      </div>
    </div>

    <!-- Arrival state -->
    <div v-if="hasArrived" class="arrival-screen">
      <div class="arrival-header">
        <span class="arrival-check">✓</span>
        <span class="arrival-title">Sei arrivato</span>
      </div>
      <div class="arrival-body">
        <p class="arrival-subtitle">Termine navigazione</p>
        <div class="arrival-actions">
          <button class="btn text-only arrival-share">Condividi viaggio</button>
          <button class="btn primary arrival-exit" @click="exitNavigation()">
            Esci
          </button>
        </div>
      </div>
    </div>

    <div v-if="gpsError" class="floating-error">
      {{ gpsError }}
    </div>

    <div v-if="!userLocation" class="floating-hint">
      In attesa del GPS…
    </div>

    <div v-if="userLocation && !dest" class="floating-hint">
      Tocca la mappa per scegliere una destinazione a piedi.
    </div>
    <!-- Route Preview -->
<div v-if="showRoutePreview" class="route-preview-screen">
  <div class="route-preview-header">
    <div class="route-preview-meta">
      <span class="route-preview-time">{{ durationText }}</span>
      <span class="route-preview-dist">{{ distanceText }}</span>
    </div>
    <p class="route-preview-subtitle">Percorso a piedi</p>
  </div>

  <div class="route-preview-steps">
    <div
      v-for="step in routeSteps"
      :key="step.index"
      class="route-step"
    >
      <div class="route-step-icon">{{ step.icon }}</div>
      <div class="route-step-label">{{ step.direction }}</div>
    </div>
  </div>

  <div class="route-preview-footer">
    <button class="btn primary wide route-start-btn" @click="startNavigation()">
      Inizia
    </button>
  </div>
</div>
<!-- Recenter button -->
<button
  v-if="userLocation && routeInfo"
  class="recenter-btn"
  @click="recenterMap()"
>
  ◎
</button>
</div>
</template>

<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref, computed } from 'vue'
import maplibregl, { type Map, type GeoJSONSource } from 'maplibre-gl'
import 'maplibre-gl/dist/maplibre-gl.css'
import type { Feature, FeatureCollection, LineString, Point } from 'geojson'

// ─── Types ────────────────────────────────────────────────────────────────────

type LngLat = { lng: number; lat: number }
type CoordinatePair = [number, number]
type RouteFeature = Feature<LineString, {}>
type PointFeature = Feature<Point, {}>
type RouteFeatureCollection = FeatureCollection<LineString, {}>
type PointFeatureCollection = FeatureCollection<Point, {}>

type DecisionNode = {
  coords: CoordinatePair
  modifier: string
  triggered: boolean
}

type NominatimResult = {
  place_id: number
  display_name: string
  lat: string
  lon: string
}

// ─── Map setup ────────────────────────────────────────────────────────────────

const mapContainer = ref<HTMLDivElement | null>(null)
let map: Map | null = null
let watchId: number | null = null

const SRC_ROUTE = 'route-src'
const LYR_ROUTE_CASE = 'route-case'
const LYR_ROUTE = 'route-line'
const SRC_USER = 'user-src'
const LYR_USER_OUT = 'user-out'
const LYR_USER_IN = 'user-in'
const SRC_DEST = 'dest-src'
const LYR_DEST = 'dest-layer'

const SRC_ALT = 'alt-route-src'
const LYR_ALT = 'alt-route-line'
const LYR_ALT_CASE = 'alt-route-case'

const SRC_ARROW = 'arrow-src'
const LYR_ARROW = 'arrow-layer'

const SRC_ROUTE_ARROWS = 'route-arrows-src'
const LYR_ROUTE_ARROWS = 'route-arrows-layer'

const isListening = ref(false)
const speechSupported = typeof window !== 'undefined' && 'webkitSpeechRecognition' in window

// ─── Navigation state ─────────────────────────────────────────────────────────

const userLocation = ref<LngLat | null>(null)
const dest = ref<LngLat | null>(null)
const routeInfo = ref<{ distance: number; duration: number } | null>(null)
const gpsError = ref<string | null>(null)
const followUser = ref(true)
const hasArrived = ref(false)
const overlayDirection = ref('Continua')
let maneuverModifiersCache: string[] = []
let decisionNodes: DecisionNode[] = []
const routeCoords = ref<{ lng: number; lat: number }[]>([])

const originalRouteCoords = ref<CoordinatePair[]>([])
const altRouteCoords = ref<CoordinatePair[]>([])
const showingAltRoute = ref(false)

// ─── Adaptive overlay state ───────────────────────────────────────────────────

const overlayVisible = ref(false)
const overlayStage = ref<0 | 1 | 2 | null>(null)
let stageTimers: ReturnType<typeof setTimeout>[] = []

// Legacy ref kept so existing logic that checks overlay.value.nodeIdx still works
const overlay = ref({ visible: false, nodeIdx: null as number | null })

// ─── Search panel state ───────────────────────────────────────────────────────

const searchCollapsed = ref(false)
const activeField = ref<'origin' | 'dest' | null>(null)
const originQuery = ref('')
const destQuery = ref('')
const originSuggestions = ref<NominatimResult[]>([])
const destSuggestions = ref<NominatimResult[]>([])
const searchOrigin = ref<LngLat | null>(null)
const searchDest = ref<LngLat | null>(null)

// ─── Hesitation detection state ───────────────────────────────────────────────

const PROXIMITY_RADIUS_M = 30
const SPEED_SAMPLES: number[] = []
const SPEED_WINDOW = 5
let lastCheckTime = 0
let lastCheckPos: LngLat | null = null
let stopStartTime: number | null = null
const STOP_THRESHOLD_MS = 4000

const navigationStartTime = ref<number | null>(null)

// ─── Reroute state ────────────────────────────────────────────────────────────

let lastReroute: LngLat | null = null
let lastRerouteTime = 0
const isOffRoute = ref(false)

const showRoutePreview = ref(false)

// ─── Computed ─────────────────────────────────────────────────────────────────

const durationText = computed(() => {
  if (!routeInfo.value) return ''
  return `${Math.round(routeInfo.value.duration / 60)} min`
})

const distanceText = computed(() => {
  if (!routeInfo.value) return ''
  const m = Math.round(routeInfo.value.distance)
  return m < 1000 ? `${m} m` : `${(m / 1000).toFixed(1)} km`
})

const overlayIcon = computed(() => {
  const m = maneuverModifiersCache[overlay.value.nodeIdx ?? 0]
  switch (m) {
    case 'right':        return '↱'
    case 'left':         return '↰'
    case 'straight':     return '↑'
    case 'slight right': return '↗'
    case 'slight left':  return '↖'
    default:             return '↑'
  }
})

const routeSteps = computed(() => {
  return decisionNodes.map((node, i) => ({
    index: i,
    direction: getDirectionLabel(i),
    icon: (() => {
      switch (maneuverModifiersCache[i]) {
        case 'right': return '↱'
        case 'left': return '↰'
        case 'straight': return '↑'
        case 'slight right': return '↗'
        case 'slight left': return '↖'
        default: return '↑'
      }
    })()
  }))
})

// ─── GeoJSON helpers ──────────────────────────────────────────────────────────

function emptyLineCollection(): RouteFeatureCollection {
  return { type: 'FeatureCollection', features: [] }
}

function emptyPointCollection(): PointFeatureCollection {
  return { type: 'FeatureCollection', features: [] }
}

function makeLineFeature(coords: number[][]): RouteFeature {
  return { type: 'Feature', properties: {}, geometry: { type: 'LineString', coordinates: coords } }
}

function makePointFeature(lng: number, lat: number): PointFeature {
  return { type: 'Feature', properties: {}, geometry: { type: 'Point', coordinates: [lng, lat] } }
}

// ─── Distance ─────────────────────────────────────────────────────────────────

function distanceMeters(a: LngLat, b: LngLat): number {
  const R = 6371000
  const toRad = (d: number) => (d * Math.PI) / 180
  const dLat = toRad(b.lat - a.lat)
  const dLng = toRad(b.lng - a.lng)
  const lat1 = toRad(a.lat)
  const lat2 = toRad(b.lat)
  const sin1 = Math.sin(dLat / 2)
  const sin2 = Math.sin(dLng / 2)
  const h = sin1 * sin1 + Math.cos(lat1) * Math.cos(lat2) * sin2 * sin2
  return R * 2 * Math.atan2(Math.sqrt(h), Math.sqrt(1 - h))
}

async function fetchRoute(a: LngLat, b: LngLat) {
  const url = `https://router.project-osrm.org/route/v1/foot/${a.lng},${a.lat};${b.lng},${b.lat}?steps=true&geometries=geojson&overview=full`
  const res = await fetch(url)
  if (!res.ok) throw new Error(`Routing failed: ${res.status}`)

  const data = await res.json()
  const route = data.routes?.[0]
  if (!route) throw new Error('No route returned')

  routeInfo.value = {
    distance: route.distance,
    duration: route.distance / 1.4  // 1.4 m/s = average walking speed (~5 km/h)
  }

  const coords: CoordinatePair[] = route.geometry.coordinates
  routeCoords.value = coords.map(c => ({ lng: c[0], lat: c[1] }))
  const src = map?.getSource(SRC_ROUTE) as GeoJSONSource | undefined
  src?.setData(makeLineFeature(coords))
  const steps = route.legs?.[0]?.steps ?? []
  maneuverModifiersCache = steps.map((s: any) => s.maneuver?.modifier ?? '')
  console.log('First coord:', coords[0], 'Last coord:', coords[coords.length - 1])
  const arrowsSrc = map?.getSource(SRC_ROUTE_ARROWS) as GeoJSONSource | undefined
  arrowsSrc?.setData(makeLineFeature(coords))
  decisionNodes = steps.map((s: any) => ({
    coords: s.maneuver.location as CoordinatePair,
    modifier: s.maneuver?.modifier ?? '',
    triggered: false
  }))

  const first = decisionNodes.findIndex(n => n.modifier !== '')
  const idx = first >= 0 ? first : 0
  overlay.value.nodeIdx = idx
  overlayDirection.value = getDirectionLabel(idx)
  updateArrow()
}

function recenterMap() {
  followUser.value = true
  if (userLocation.value && map) {
    map.easeTo({ center: [userLocation.value.lng, userLocation.value.lat], zoom: 17, duration: 800 })
  }
}

function cancelRoute() {
  dest.value = null
  routeInfo.value = null
  decisionNodes = []
  maneuverModifiersCache = []
  dismissOverlay()
  dismissRerouteOverlay()
  hasArrived.value = false
  showRoutePreview.value = false
  const routeSrc = map?.getSource(SRC_ROUTE) as GeoJSONSource | undefined
  routeSrc?.setData(emptyLineCollection())
  const arrowsSrc = map?.getSource(SRC_ROUTE_ARROWS) as GeoJSONSource | undefined
  arrowsSrc?.setData(emptyLineCollection())
  const destSrc = map?.getSource(SRC_DEST) as GeoJSONSource | undefined
  destSrc?.setData(emptyPointCollection())
  updateArrow()
}

function getDirectionLabel(i: number): string {
  const m = maneuverModifiersCache[i]
  switch (m) {
    case 'right':        return 'Continua a destra'
    case 'left':         return 'Continua a sinistra'
    case 'straight':     return 'Continua dritto'
    case 'slight right': return 'Leggermente a destra'
    case 'slight left':  return 'Leggermente a sinistra'
    default:             return 'Continua'
  }
}

function checkIfOffRoute(): boolean {
  if (!userLocation.value || decisionNodes.length === 0) return false
  // Check distance to nearest decision node or route point
  const nearAnyNode = decisionNodes.some(n => {
    const d = distanceMeters(userLocation.value!, { lng: n.coords[0], lat: n.coords[1] })
    return d < 50
  })
  return !nearAnyNode
}
function checkArrival() {
  if (!userLocation.value || !dest.value || hasArrived.value) return
  if (!routeInfo.value) return
  const dist = distanceMeters(userLocation.value, dest.value)
  if (dist < 30) {
    hasArrived.value = true
    dismissOverlay()
    dismissRerouteOverlay()
  }
}
function exitNavigation() {
  hasArrived.value = false
  dest.value = null
  routeInfo.value = null
}

function startNavigation() {
  navigationStartTime.value = Date.now()
  showRoutePreview.value = false
  // Zoom into first decision node
  const firstNode = decisionNodes.find(n => n.modifier !== '')
  if (firstNode && map) {
    map.flyTo({
      center: [firstNode.coords[0], firstNode.coords[1]],
      zoom: 18,
      duration: 800
    })
  }
}
// ─── Drag to dismiss ──────────────────────────────────────────────────────────
let dragStartY = 0
let isDragging = false

function getClientY(e: TouchEvent | MouseEvent): number | null {
  if ('touches' in e) {
    const touch = e.touches[0]
    return touch ? touch.clientY : null
  }
  return e.clientY
}

function onDragStart(e: TouchEvent | MouseEvent) {
  const y = getClientY(e)
  if (y === null) return
  dragStartY = y
  isDragging = true
}

function onDragMove(e: TouchEvent | MouseEvent) {
  if (!isDragging) return
  const y = getClientY(e)
  if (y === null) return
  const deltaY = y - dragStartY
  if (deltaY > 80) {
    isDragging = false
    dismissOverlay()
  }
}

function onDragMoveReroute(e: TouchEvent | MouseEvent) {
  if (!isDragging) return
  const y = getClientY(e)
  if (y === null) return
  const deltaY = y - dragStartY
  if (deltaY > 80) {
    isDragging = false
    dismissRerouteOverlay()
  }
}

function onDragEnd() {
  isDragging = false
  window.removeEventListener('mousemove', onDragMove)
  window.removeEventListener('mousemove', onDragMoveReroute)
  window.removeEventListener('mouseup', onDragEnd)
}

function onHandleMouseDown(e: MouseEvent) {
  onDragStart(e)
  window.addEventListener('mousemove', onDragMove)
  window.addEventListener('mouseup', onDragEnd)
}

function onHandleMouseDownReroute(e: MouseEvent) {
  onDragStart(e)
  window.addEventListener('mousemove', onDragMoveReroute)
  window.addEventListener('mouseup', onDragEnd)
}
// ─── Rerouting ────────────────────────────────────────────────────────────────


async function reroute() {
  if (!userLocation.value || !dest.value) return
  const now = Date.now()
  if (lastReroute) {
    const moved = distanceMeters(userLocation.value, lastReroute)
    if (moved < 20) return
  }
  if (now - lastRerouteTime < 5000) return
  // Don't check off-route for first 10 seconds after starting
  if (navigationStartTime.value && Date.now() - navigationStartTime.value < 10000) return
  lastReroute = { ...userLocation.value }
  lastRerouteTime = now

  // Check if user is off route before rerouting
  const offRoute = checkIfOffRoute()
  if (offRoute && !isOffRoute.value) {
    isOffRoute.value = true
    triggerRerouteOverlay()
  }

  await fetchRoute(userLocation.value, dest.value)
  
  if (isOffRoute.value) {
    isOffRoute.value = false
  }
}

function clearAltRoute() {
  const src = map?.getSource(SRC_ALT) as GeoJSONSource | undefined
  src?.setData(emptyLineCollection())
  altRouteCoords.value = []
  showingAltRoute.value = false
}

async function fetchAltRoute() {
  if (!userLocation.value || !dest.value) return

  // Add a slight offset waypoint to force OSRM to find a different path
  const midLng = (userLocation.value.lng + dest.value.lng) / 2 + 0.003
  const midLat = (userLocation.value.lat + dest.value.lat) / 2 + 0.003

  const url = `https://router.project-osrm.org/route/v1/foot/${userLocation.value.lng},${userLocation.value.lat};${midLng},${midLat};${dest.value.lng},${dest.value.lat}?steps=true&geometries=geojson&overview=full`
  const res = await fetch(url)
  if (!res.ok) return
  const data = await res.json()

  const route = data.routes?.[0]
  if (!route) return

  altRouteCoords.value = route.geometry.coordinates
  const src = map?.getSource(SRC_ALT) as GeoJSONSource | undefined
  src?.setData(makeLineFeature(altRouteCoords.value))
  showingAltRoute.value = true
}

function useAltRoute() {
  if (!altRouteCoords.value.length) return
  // Swap alt route to main route
  const src = map?.getSource(SRC_ROUTE) as GeoJSONSource | undefined
  src?.setData(makeLineFeature(altRouteCoords.value))
  clearAltRoute()
  dismissRerouteOverlay()
}

function keepOriginalRoute() {
  clearAltRoute()
  dismissRerouteOverlay()
}

// ─── Map dot updates ──────────────────────────────────────────────────────────

function updateUserDot() {
  if (!map || !userLocation.value) return
  const src = map.getSource(SRC_USER) as GeoJSONSource | undefined
  src?.setData(makePointFeature(userLocation.value.lng, userLocation.value.lat))
  if (followUser.value) {
    map.easeTo({ center: [userLocation.value.lng, userLocation.value.lat], duration: 800 })
  }
}

function updateDestinationDot() {
  if (!map) return
  const src = map.getSource(SRC_DEST) as GeoJSONSource | undefined
  if (!dest.value) {
    src?.setData(emptyPointCollection())
    return
  }
  src?.setData(makePointFeature(dest.value.lng, dest.value.lat))
}

function findNearestRouteIndex(user: LngLat): number {
  let minDist = Infinity
  let nearestIdx = 0
  for (let i = 0; i < routeCoords.value.length; i++) {
    const point = routeCoords.value[i]
    if (!point) continue
    const d = distanceMeters(user, point)
    if (d < minDist) {
      minDist = d
      nearestIdx = i
    }
  }
  return nearestIdx
}

function calculateBearing(from: LngLat, to: LngLat): number {
  const lat1 = from.lat * Math.PI / 180
  const lat2 = to.lat * Math.PI / 180
  const dLng = (to.lng - from.lng) * Math.PI / 180

  const y = Math.sin(dLng) * Math.cos(lat2)
  const x = Math.cos(lat1) * Math.sin(lat2) - Math.sin(lat1) * Math.cos(lat2) * Math.cos(dLng)
  const bearing = Math.atan2(y, x) * 180 / Math.PI

  return (bearing + 360) % 360
}

function updateArrow() {
  if (!map) return
  const nextNode = decisionNodes.find(n => !n.triggered && n.modifier !== '')
  if (!nextNode || !userLocation.value || routeCoords.value.length < 2) {
    const src = map.getSource(SRC_ARROW) as GeoJSONSource | undefined
    src?.setData(emptyPointCollection())
    return
  }

  const user = userLocation.value
  const nearestIdx = findNearestRouteIndex(user)

  // Look a few points ahead on the route to get a stable direction
  const lookAheadIdx = Math.min(nearestIdx + 5, routeCoords.value.length - 1)
  const target = routeCoords.value[lookAheadIdx]
  if (!target) return

  const bearing = calculateBearing(user, target)

  const feature: PointFeature = {
    type: 'Feature',
    properties: { bearing },
    geometry: { type: 'Point', coordinates: [user.lng, user.lat] }
  }
  const src = map.getSource(SRC_ARROW) as GeoJSONSource | undefined
  src?.setData({ type: 'FeatureCollection', features: [feature] })
}


// ─── Adaptive overlay ─────────────────────────────────────────────────────────

function triggerOverlayForNode(node: DecisionNode) {
  node.triggered = true
  stopStartTime = null

  const nodeIdx = decisionNodes.indexOf(node)
  overlay.value.nodeIdx = nodeIdx
  overlayDirection.value = getDirectionLabel(nodeIdx)

  stageTimers.forEach(clearTimeout)
  stageTimers = []

  overlayStage.value = 0
  overlayVisible.value = true

  stageTimers.push(setTimeout(() => { overlayStage.value = 1 }, 1200))
  stageTimers.push(setTimeout(() => { overlayStage.value = 2 }, 2800))
}

function dismissOverlay() {
  stageTimers.forEach(clearTimeout)
  stageTimers = []
  overlayStage.value = null
  overlayVisible.value = false
  overlay.value.visible = false
}


// ─── Reroute overlay ──────────────────────────────────────────────────────────

const rerouteOverlayVisible = ref(false)
const rerouteOverlayStage = ref<0 | 1 | 2 | 3 | null>(null)
let rerouteTimers: ReturnType<typeof setTimeout>[] = []

function triggerRerouteOverlay() {
  rerouteTimers.forEach(clearTimeout)
  rerouteTimers = []

  rerouteOverlayStage.value = 0
  rerouteOverlayVisible.value = true

  rerouteTimers.push(setTimeout(() => { rerouteOverlayStage.value = 1 }, 1200))
  rerouteTimers.push(setTimeout(() => { rerouteOverlayStage.value = 2 }, 2800))
}

function dismissRerouteOverlay() {
  rerouteTimers.forEach(clearTimeout)
  rerouteTimers = []
  rerouteOverlayStage.value = null
  rerouteOverlayVisible.value = false
  followUser.value = true  // ← add this
  if (userLocation.value && map) {
    map.easeTo({ 
      center: [userLocation.value.lng, userLocation.value.lat], 
      zoom: 17,
      duration: 800 
    })
  }
}

function resetOverlayIfWalking(speed: number) {
  if (!overlayVisible.value || overlayStage.value === null) return
  if (speed > 0.8) {
    stageTimers.forEach(clearTimeout)
    stageTimers = []
    overlayStage.value = 0
    stageTimers.push(setTimeout(() => { overlayStage.value = 1 }, 1200))
    stageTimers.push(setTimeout(() => { overlayStage.value = 2 }, 2800))
  }
}

// ─── Hesitation detection ─────────────────────────────────────────────────────

function checkDecisionNodeProximity() {
  if (!userLocation.value || decisionNodes.length === 0) return

  const now = Date.now()
  const user = userLocation.value

  // Rolling speed estimate
  if (lastCheckPos && lastCheckTime) {
    const elapsed = (now - lastCheckTime) / 1000
    const moved = distanceMeters(user, lastCheckPos)
    const speed = elapsed > 0 ? moved / elapsed : 0
    SPEED_SAMPLES.push(speed)
    if (SPEED_SAMPLES.length > SPEED_WINDOW) SPEED_SAMPLES.shift()
  }

  lastCheckPos = { ...user }
  lastCheckTime = now

  const avgSpeed = SPEED_SAMPLES.length
    ? SPEED_SAMPLES.reduce((a, b) => a + b, 0) / SPEED_SAMPLES.length
    : null

  // Reset overlay if user picks up pace
  if (avgSpeed !== null) resetOverlayIfWalking(avgSpeed)

  // Check each untriggered decision node
  for (const node of decisionNodes) {
    if (node.triggered) continue
    if (node.modifier === '') continue

    const dist = distanceMeters(user, { lng: node.coords[0], lat: node.coords[1] })
    if (dist > PROXIMITY_RADIUS_M) continue

    const isStopped = avgSpeed !== null && avgSpeed < 0.1
    const isSlow    = avgSpeed !== null && avgSpeed < 0.5

    if (isStopped) {
      if (stopStartTime === null) stopStartTime = now
      if (now - stopStartTime >= STOP_THRESHOLD_MS) triggerOverlayForNode(node)
    } else if (isSlow) {
      stopStartTime = null
      triggerOverlayForNode(node)
    } else {
      stopStartTime = null
    }

    break
  }

  
  const nextNode = decisionNodes.find(n => !n.triggered && n.modifier !== '')
  if (nextNode) {
    const idx = decisionNodes.indexOf(nextNode)
    overlay.value.nodeIdx = idx
    overlayDirection.value = getDirectionLabel(idx)
  }
  updateArrow() 
}

// ─── Search / geocoding ───────────────────────────────────────────────────────

let originDebounce: ReturnType<typeof setTimeout> | null = null
let destDebounce:   ReturnType<typeof setTimeout> | null = null

function startVoiceInput(field: 'origin' | 'dest') {
  if (!speechSupported) return
  const SpeechRecognition = (window as any).webkitSpeechRecognition
  const recognition = new SpeechRecognition()
  recognition.lang = 'it-IT'
  recognition.continuous = false
  recognition.interimResults = false

  isListening.value = true

  recognition.onresult = async (event: any) => {
    const transcript = event.results[0][0].transcript
    isListening.value = false
    if (field === 'origin') {
      originQuery.value = transcript
      originSuggestions.value = await geocode(transcript)
    } else {
      destQuery.value = transcript
      destSuggestions.value = await geocode(transcript)
    }
  }

  recognition.onerror = () => { isListening.value = false }
  recognition.onend = () => { isListening.value = false }
  recognition.start()
}
async function geocode(query: string): Promise<NominatimResult[]> {
  if (query.trim().length < 3) return []

  // Milan bounding box: west, south, east, north
  // Covers Milan city and its immediate surroundings
  const milanViewbox = '9.04,45.39,9.28,45.54'

  const params = new URLSearchParams({
    format: 'json',
    q: query,
    limit: '5',
    countrycodes: 'it',   // only Italian addresses
    viewbox: milanViewbox, // prefer results inside this box
    bounded: '0',          // 0 = also fall back to rest of Italy if nothing matches locally
                           // change to '1' to strictly limit to Milan area only
  })

  const url = `https://nominatim.openstreetmap.org/search?${params}`
  const res = await fetch(url, { headers: { 'Accept-Language': 'it' } })
  return res.ok ? res.json() : []
}

function onOriginInput() {
  searchOrigin.value = null
  if (originDebounce) clearTimeout(originDebounce)
  originDebounce = setTimeout(async () => {
    originSuggestions.value = await geocode(originQuery.value)
  }, 350)
}

function onDestInput() {
  searchDest.value = null
  if (destDebounce) clearTimeout(destDebounce)
  destDebounce = setTimeout(async () => {
    destSuggestions.value = await geocode(destQuery.value)
  }, 350)
}

function selectOrigin(s: NominatimResult) {
  originQuery.value = s.display_name
  searchOrigin.value = { lng: parseFloat(s.lon), lat: parseFloat(s.lat) }
  originSuggestions.value = []
  activeField.value = null
}

function selectDest(s: NominatimResult) {
  destQuery.value = s.display_name
  searchDest.value = { lng: parseFloat(s.lon), lat: parseFloat(s.lat) }
  destSuggestions.value = []
  activeField.value = null
}

async function routeFromSearch() {
  if (!searchOrigin.value || !searchDest.value) return
  userLocation.value = searchOrigin.value
  updateUserDot()
  dest.value = searchDest.value
  updateDestinationDot()
  await fetchRoute(searchOrigin.value, searchDest.value)
  map?.fitBounds(
    [
      [Math.min(searchOrigin.value.lng, searchDest.value.lng),
       Math.min(searchOrigin.value.lat, searchDest.value.lat)],
      [Math.max(searchOrigin.value.lng, searchDest.value.lng),
       Math.max(searchOrigin.value.lat, searchDest.value.lat)]
    ],
    { padding: 60, duration: 800 }
  )
  showRoutePreview.value = true
  searchCollapsed.value = true
}
// ─── Debug shortcuts ──────────────────────────────────────────────────────────

let debugKeyHandler: ((e: KeyboardEvent) => void) | undefined

function setupDebugShortcuts() {
  debugKeyHandler = (e: KeyboardEvent) => {
    if (e.key === 'h' || e.key === 'H') {
      // Ignore shortcuts while typing in an input field
      const target = e.target as HTMLElement
    if (target.tagName === 'INPUT' || target.tagName === 'TEXTAREA') return
      const node = decisionNodes.find(n => !n.triggered && n.modifier !== '')
      if (!node) {
        console.warn('[Debug] No untriggered decision nodes available')
        return
      }
      console.log('[Debug] Simulating hesitation at node:', node)
      triggerOverlayForNode(node)
    }
    if (e.key === 'r' || e.key === 'R') {
    // Ignore shortcuts while typing in an input field
      const target = e.target as HTMLElement
      if (target.tagName === 'INPUT' || target.tagName === 'TEXTAREA') return
      decisionNodes.forEach(n => (n.triggered = false))
      dismissOverlay()
      console.log('[Debug] All decision nodes reset')
    }
    if (e.key === 'v' || e.key === 'V') {
      // Ignore shortcuts while typing in an input field
      const target = e.target as HTMLElement
      if (target.tagName === 'INPUT' || target.tagName === 'TEXTAREA') return
      triggerRerouteOverlay()
    }
    if (e.key === 'a' || e.key === 'A') {
      // Ignore shortcuts while typing in an input field
      const target = e.target as HTMLElement
      if (target.tagName === 'INPUT' || target.tagName === 'TEXTAREA') return
      console.log('[Debug] Arrival triggered')
      hasArrived.value = true
      dismissOverlay()
      dismissRerouteOverlay()
    }
  }
  window.addEventListener('keydown', debugKeyHandler!)
}

// ─── Lifecycle ────────────────────────────────────────────────────────────────

onMounted(() => {
  searchCollapsed.value = window.innerWidth < 480
  setupDebugShortcuts()

  if (!mapContainer.value) return

  map = new maplibregl.Map({
    container: mapContainer.value,
    style: {
      version: 8,
      sources: {
        osm: {
          type: 'raster',
          tiles: [
            'https://a.tile.openstreetmap.org/{z}/{x}/{y}.png',
            'https://b.tile.openstreetmap.org/{z}/{x}/{y}.png',
            'https://c.tile.openstreetmap.org/{z}/{x}/{y}.png'
          ],
          tileSize: 256
        }
      },
      layers: [{ id: 'osm', type: 'raster', source: 'osm' }]
    },
   
    center: [9.1900, 45.4642],
    zoom: 14
  })

  map.on('load', () => {
    map?.addSource(SRC_ROUTE, { type: 'geojson', data: emptyLineCollection() })
    map?.addLayer({
      id: LYR_ROUTE_CASE, type: 'line', source: SRC_ROUTE,
      paint: { 'line-color': '#ffffff', 'line-width': 10 },
      layout: { 'line-cap': 'round', 'line-join': 'round' }
    })
    map?.addLayer({
      id: LYR_ROUTE, type: 'line', source: SRC_ROUTE,
      paint: { 'line-color': '#2563eb', 'line-width': 10 },
      layout: { 'line-cap': 'round', 'line-join': 'round' }
    })

    map?.addSource(SRC_USER, { type: 'geojson', data: emptyPointCollection() })
    map?.addLayer({
      id: LYR_USER_OUT, type: 'circle', source: SRC_USER,
      paint: { 'circle-radius': 10, 'circle-color': '#ffffff' }
    })
    map?.addLayer({
      id: LYR_USER_IN, type: 'circle', source: SRC_USER,
      paint: { 'circle-radius': 6, 'circle-color': '#2563eb' }
    })

    map?.addSource(SRC_DEST, { type: 'geojson', data: emptyPointCollection() })
    map?.addLayer({
      id: LYR_DEST, type: 'circle', source: SRC_DEST,
      paint: { 'circle-radius': 8, 'circle-color': '#16a34a' }
    })
    const SRC_ALT = 'alt-route-src'
    const LYR_ALT = 'alt-route-line'
    const LYR_ALT_CASE = 'alt-route-case'

    map?.addSource(SRC_ALT, { type: 'geojson', data: emptyLineCollection() })
    map?.addLayer({
      id: LYR_ALT_CASE, type: 'line', source: SRC_ALT,
      paint: { 'line-color': '#ffffff', 'line-width': 10 },
      layout: { 'line-cap': 'round', 'line-join': 'round' }
    })
    map?.addLayer({
      id: LYR_ALT, type: 'line', source: SRC_ALT,
      paint: { 'line-color': '#9ca3af', 'line-width': 6 },
      layout: { 'line-cap': 'round', 'line-join': 'round' }
    })
    // Small white chevron arrows along the route
    const chevronCanvas = document.createElement('canvas')
    chevronCanvas.width = 16
    chevronCanvas.height = 16
    const chevCtx = chevronCanvas.getContext('2d')!
    chevCtx.strokeStyle = '#ffffff'
    chevCtx.lineWidth = 2.5
    chevCtx.lineCap = 'round'
    chevCtx.lineJoin = 'round'
    chevCtx.beginPath()
    chevCtx.moveTo(4, 11)
    chevCtx.lineTo(8, 5)
    chevCtx.lineTo(12, 11)
    chevCtx.stroke()

    const chevronImg = new Image()
    chevronImg.src = chevronCanvas.toDataURL()
    chevronImg.onload = () => {
      map?.addImage('chevron-arrow', chevronImg)
      map?.addSource(SRC_ROUTE_ARROWS, { type: 'geojson', data: emptyLineCollection() })
      map?.addLayer({
        id: LYR_ROUTE_ARROWS,
        type: 'symbol',
        source: SRC_ROUTE_ARROWS,
        layout: {
          'symbol-placement': 'line',
          'symbol-spacing': 50,
          'icon-image': 'chevron-arrow',
          'icon-size': 1,
          'icon-rotate': 90,
          'icon-rotation-alignment': 'map',
          'icon-pitch-alignment': 'map',
          'icon-allow-overlap': true,
          'icon-ignore-placement': true
        }
      })
    }
        // Create arrow image programmatically
    const arrowSize = 80
    const arrowCanvas = document.createElement('canvas')
    arrowCanvas.width = arrowSize
    arrowCanvas.height = arrowSize
    const ctx = arrowCanvas.getContext('2d')!
    ctx.fillStyle = '#2563eb'
    ctx.beginPath()
    ctx.moveTo(arrowSize / 2, 4)
    ctx.lineTo(arrowSize - 4, arrowSize - 4)
    ctx.lineTo(arrowSize / 2, arrowSize - 16)
    ctx.lineTo(4, arrowSize - 4)
    ctx.closePath()
    ctx.fill()

    const arrowImage = new Image()
    arrowImage.src = arrowCanvas.toDataURL()
    arrowImage.onload = () => {
      map?.addImage('nav-arrow', arrowImage)
      map?.addSource(SRC_ARROW, { type: 'geojson', data: emptyPointCollection() })
      map?.addLayer({
        id: LYR_ARROW,
        type: 'symbol',
        source: SRC_ARROW,
        layout: {
          'icon-image': 'nav-arrow',
          'icon-size': 0.6,
          'icon-rotate': ['get', 'bearing'],
          'icon-rotation-alignment': 'map',
          'icon-allow-overlap': true
        }
      })
    }
  })

  map.on('click', async (e) => {
  if (!userLocation.value) return
  if (routeInfo.value) return

  const lng = e.lngLat.lng
  const lat = e.lngLat.lat

  dest.value = { lng, lat }
  updateDestinationDot()

  // Reverse geocode the tapped point
  try {
    const reverseUrl = `https://nominatim.openstreetmap.org/reverse?format=json&lat=${lat}&lon=${lng}&zoom=18`
    const res = await fetch(reverseUrl, { headers: { 'Accept-Language': 'it' } })
    if (res.ok) {
      const data = await res.json()
      destQuery.value = data.display_name ?? `${lat.toFixed(5)}, ${lng.toFixed(5)}`
      searchDest.value = { lng, lat }
    }
  } catch {}

  // Auto-fill origin with current GPS position
  try {
    const revOrigin = `https://nominatim.openstreetmap.org/reverse?format=json&lat=${userLocation.value.lat}&lon=${userLocation.value.lng}&zoom=18`
    const resOrigin = await fetch(revOrigin, { headers: { 'Accept-Language': 'it' } })
    if (resOrigin.ok) {
      const dataOrigin = await resOrigin.json()
      originQuery.value = dataOrigin.display_name ?? 'Posizione attuale'
      searchOrigin.value = { ...userLocation.value }
    }
  } catch {}

  searchCollapsed.value = false
  await fetchRoute(userLocation.value, dest.value)
})

  if (navigator.geolocation) {
    watchId = navigator.geolocation.watchPosition(
      async (pos) => {
        userLocation.value = { lng: pos.coords.longitude, lat: pos.coords.latitude }
        updateUserDot()
        checkDecisionNodeProximity()
        checkArrival()
        if (dest.value) await reroute()
      },
      () => { gpsError.value = 'Impossibile ottenere la posizione GPS' },
      { enableHighAccuracy: true }
    )
  }
})

onBeforeUnmount(() => {
  if (watchId !== null) navigator.geolocation.clearWatch(watchId)
  if (debugKeyHandler) window.removeEventListener('keydown', debugKeyHandler)
  stageTimers.forEach(clearTimeout)
  rerouteTimers.forEach(clearTimeout)
  map?.remove()
})
</script>

<style scoped>
.app {
  position: relative;
  height: 100vh;
  width: 100%;
  overflow: hidden;
  font-family: Inter, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  background: #d8d8d8;
  padding-top: env(safe-area-inset-top);
  box-sizing: border-box;
}
.map {
  position: absolute;
  inset: 0;
}

/* ── Search Panel ─────────────────────────────────────────────── */
.search-panel {
  position: absolute;
  top: max(16px, env(safe-area-inset-top));
  left: 16px;
  z-index: 40;
  width: min(340px, calc(100vw - 32px));
  background: rgba(255, 255, 255, 0.97);
  border-radius: 20px;
  box-shadow: 0 12px 36px rgba(0, 0, 0, 0.18);
  backdrop-filter: blur(8px);
  overflow: visible;
  transition: width 0.2s ease;
}

.search-toggle {
  display: block;
  width: 100%;
  padding: 12px 16px;
  background: transparent;
  border: none;
  text-align: left;
  font-size: 14px;
  font-weight: 600;
  color: #1d1d1f;
  cursor: pointer;
  border-radius: 20px;
}

.search-body {
  padding: 0 14px 14px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.search-field {
  display: flex;
  flex-direction: column;
  gap: 4px;
  position: relative;
}

.search-field label {
  font-size: 11px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: #8a8a8a;
  padding-left: 2px;
}

.search-input-wrap {
  position: relative;
  display: flex;
  align-items: center;
}

.search-input-wrap input {
  width: 100%;
  padding: 10px 60px 10px 12px;
  border: 1.5px solid #e0e0e0;
  border-radius: 12px;
  font-size: 14px;
  color: #1d1d1f;
  background: #f7f7f7;
  outline: none;
  transition: border-color 0.15s;
  box-sizing: border-box;
}

.search-input-wrap input:focus {
  border-color: #2563eb;
  background: #fff;
}
.mic-btn {
  position: absolute;
  right: 30px;
  background: none;
  border: none;
  font-size: 14px;
  cursor: pointer;
  padding: 2px 4px;
  color: #555;
}
.clear-btn {
  position: absolute;
  right: 8px;
  background: none;
  border: none;
  color: #aaa;
  font-size: 12px;
  cursor: pointer;
  padding: 2px 4px;
}

.suggestions {
  position: absolute;
  top: calc(100% + 4px);
  left: 0;
  right: 0;
  background: #fff;
  border: 1px solid #e0e0e0;
  border-radius: 12px;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.12);
  list-style: none;
  margin: 0;
  padding: 4px 0;
  z-index: 50;
  max-height: 200px;
  overflow-y: auto;
}

.suggestions li {
  padding: 9px 14px;
  font-size: 13px;
  color: #1d1d1f;
  cursor: pointer;
  line-height: 1.4;
  border-bottom: 1px solid #f0f0f0;
}
.recenter-btn {
  position: absolute;
  bottom: 100px;
  right: 16px;
  z-index: 25;
  width: 44px;
  height: 44px;
  border-radius: 50%;
  background: white;
  border: none;
  font-size: 20px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
}

.recenter-btn:active {
  transform: scale(0.95);
}

.cancel-route-btn {
  position: absolute;
  top: 88px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 25;
  background: white;
  border: 1.5px solid #e0e0e0;
  border-radius: 999px;
  padding: 8px 16px;
  font-size: 13px;
  font-weight: 600;
  color: #e53e3e;
  cursor: pointer;
  box-shadow: 0 4px 12px rgba(0,0,0,0.12);
  white-space: nowrap;
}
.cancel-route-btn.nav-below-search {
  top: 160px; /* 88px base + 72px, same shift as .nav-top.nav-below-search */
}

.suggestions li:last-child { border-bottom: none; }
.suggestions li:hover { background: #f0f5ff; }

.go-btn { margin-top: 4px; }
.go-btn:disabled { opacity: 0.4; cursor: not-allowed; }

/* ── Top Nav Bar ──────────────────────────────────────────────── */
.nav-top {
  position: absolute;
  top: max(16px, env(safe-area-inset-top));
  left: 16px;
  right: 16px;
  z-index: 20;
  display: flex;
  justify-content: center;
  pointer-events: none;
  transition: top 0.2s ease;
}

.nav-top.nav-below-search {
  top: 88px;
}

.nav-top-inner {
  min-width: 220px;
  max-width: 340px;
  width: fit-content;
  background: #1f6fe5;
  color: #ffffff;
  border-radius: 18px;
  padding: 12px 16px;
  display: flex;
  align-items: center;
  gap: 12px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.22);
  pointer-events: auto;
}

.nav-arrow   { font-size: 22px; line-height: 1; font-weight: 700; }
.nav-meta    { display: flex; align-items: baseline; gap: 10px; white-space: nowrap; }
.nav-time    { font-size: 18px; font-weight: 700; }
.nav-distance{ font-size: 14px; opacity: 0.95; }


/* ── Overlay bottom sheet ─────────────────────────────────────── */
.overlay {
  position: absolute;
  left: 16px;
  right: 16px;
  bottom: 20px;
  z-index: 30;
  display: flex;
  justify-content: center;
  pointer-events: none;
}

.overlay-card {
  width: min(100%, 360px);
  background: #ffffff;
  border-radius: 28px;
  padding: 14px 18px 18px;
  box-shadow: 0 18px 40px rgba(0, 0, 0, 0.2), 0 4px 10px rgba(0, 0, 0, 0.08);
  pointer-events: auto;
  animation: sheetUp 0.25s ease;
  min-height: 120px;
  position: relative;
}

@keyframes sheetUp {
  from { opacity: 0; transform: translateY(22px); }
  to   { opacity: 1; transform: translateY(0); }
}

.overlay-handle {
  width: 42px;
  height: 5px;
  border-radius: 999px;
  background: #d1d1d6;
  margin: 0 auto 14px;
}

/* Stage 0 */
.stage-0 {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  padding: 8px 0 16px;
}

.stage0-pin {
  font-size: 36px;
  animation: pulse-pin 1s ease-in-out infinite alternate;
}

@keyframes pulse-pin {
  from { transform: translateY(0px);  opacity: 0.8; }
  to   { transform: translateY(-6px); opacity: 1;   }
}

.stage0-text {
  font-size: 18px;
  font-weight: 600;
  color: #1d1d1f;
  text-align: center;
}

/* Stage 1 */
.overlay-subtitle {
  text-align: center;
  font-size: 13px;
  color: #8a8a8a;
  margin-bottom: 14px;
}

.overlay-main {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
  margin-bottom: 18px;
}

.overlay-icon      { font-size: 34px; color: #2f8f1f; line-height: 1; }
.overlay-direction { font-size: 24px; line-height: 1.2; font-weight: 700; text-align: center; color: #1d1d1f; }

/* Stage 2 */
.overlay-actions { display: flex; flex-direction: column; gap: 10px; }

/* ── Buttons ──────────────────────────────────────────────────── */
.btn {
  appearance: none;
  border: none;
  outline: none;
  cursor: pointer;
  transition: transform 0.15s ease, opacity 0.15s ease, background 0.15s ease;
}

.btn:active { transform: scale(0.98); }

.btn.primary {
  background: #2f9e1f;
  color: #ffffff;
  font-weight: 700;
  font-size: 16px;
  border-radius: 14px;
  padding: 14px 18px;
}

.btn.primary:hover { background: #27851a; }
.btn.wide { width: 100%; }

.btn.text-only {
  background: transparent;
  color: #5f6368;
  font-size: 14px;
  font-weight: 600;
  padding: 4px 0 0;
}

/* ── Transitions ──────────────────────────────────────────────── */
.fade-enter-active, .fade-leave-active { transition: opacity 0.4s ease; }
.fade-enter-from,   .fade-leave-to     { opacity: 0; }

.slide-up-enter-active { transition: all 0.35s cubic-bezier(0.34, 1.56, 0.64, 1); }
.slide-up-enter-from   { opacity: 0; transform: translateY(16px); }

/* ── Floating messages ────────────────────────────────────────── */
.floating-error {
  position: absolute;
  left: 16px; right: 16px;
  bottom: 210px;
  z-index: 25;
  margin: 0 auto;
  width: min(100%, 360px);
  padding: 10px 12px;
  border-radius: 12px;
  background: #fdecec;
  color: #8b1e1e;
  font-size: 13px;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.12);
}

.floating-hint {
  position: absolute;
  left: 16px; right: 16px;
  bottom: 20px;
  z-index: 15;
  margin: 0 auto;
  width: min(100%, 360px);
  padding: 12px 14px;
  border-radius: 16px;
  background: rgba(255, 255, 255, 0.92);
  color: #3b3b3b;
  font-size: 13px;
  line-height: 1.4;
  box-shadow: 0 10px 26px rgba(0, 0, 0, 0.1);
  backdrop-filter: blur(6px);
}

/* ── Small screens ────────────────────────────────────────────── */
@media (max-width: 480px) {
  .nav-top        { top: 12px; left: 12px; right: 12px; }
  .nav-top.nav-below-search { top: 88px; }
  .overlay        { left: 12px; right: 12px; bottom: 14px; }
  .overlay-direction { font-size: 22px; }
  .floating-error { left: 12px; right: 12px; }
  .floating-hint  { left: 12px; right: 12px; bottom: 14px; }
}
/* ── Arrival screen ───────────────────────────────────────────── */
.arrival-screen {
  position: absolute;
  inset: 0;
  z-index: 100;
  display: flex;
  flex-direction: column;
}

.arrival-header {
  background: #16a34a;
  color: white;
  padding: 20px 16px;
  display: flex;
  align-items: center;
  gap: 12px;
}

.arrival-body {
  flex: 1;
  background: transparent !important;
  display: flex !important;
  flex-direction: column !important;
  justify-content: flex-end !important;
  padding: 0 !important;
}

.arrival-subtitle {
  font-size: 18px !important;
  color: #1d1d1f !important;
  margin: 0 !important;
  font-weight: 600 !important;
  padding: 16px 16px 0 !important;
  background: white;
}
.arrival-actions {
  display: flex !important;
  flex-direction: row !important;
  gap: 12px !important;
  width: 100% !important;
  padding: 12px 16px 24px !important;
  background: white;
  align-items: center !important;
  justify-content: center !important;
}
.arrival-share {
  color: #3b3b3b !important;
  font-size: 15px !important;
  border: 1.5px solid #e0e0e0 !important;
  border-radius: 999px !important;
  padding: 10px 18px !important;
}

.arrival-exit {
  padding: 10px 28px !important;
  border-radius: 999px !important;
}
.overlay-close-btn {
  position: absolute;
  top: 12px;
  right: 14px;
  background: none;
  border: none;
  font-size: 16px;
  color: #aaa;
  cursor: pointer;
  padding: 4px 8px;
  line-height: 1;
}

.overlay-close-btn:hover {
  color: #555;
}
/* ── Route Preview ────────────────────────────────────────────── */
.route-preview-screen {
  position: absolute;
  inset: 0;
  z-index: 90;
  display: flex;
  flex-direction: column;
  background: #ffffff;
}

.route-preview-header {
  background: #1f6fe5;
  color: white;
  padding: 20px 16px 16px;
}

.route-preview-meta {
  display: flex;
  align-items: baseline;
  gap: 10px;
}

.route-preview-time {
  font-size: 28px;
  font-weight: 700;
}

.route-preview-dist {
  font-size: 16px;
  opacity: 0.9;
}

.route-preview-subtitle {
  margin: 4px 0 0;
  font-size: 14px;
  opacity: 0.85;
}

.route-preview-steps {
  flex: 1;
  overflow-y: auto;
  padding: 8px 0;
}

.route-step {
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 14px 20px;
  border-bottom: 1px solid #f0f0f0;
}

.route-step-icon {
  font-size: 22px;
  width: 32px;
  text-align: center;
  flex-shrink: 0;
}

.route-step-label {
  font-size: 15px;
  color: #1d1d1f;
}

.route-preview-footer {
  padding: 16px;
  border-top: 1px solid #f0f0f0;
}

.route-start-btn {
  font-size: 17px;
  padding: 16px;
  border-radius: 16px;
}
</style>
