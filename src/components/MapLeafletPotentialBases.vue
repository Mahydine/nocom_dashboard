<script setup>
import { onMounted, onBeforeUnmount, watch, ref, computed } from 'vue'
import L from 'leaflet'

/**
 * Props:
 *  - zones: [{
 *      id, name, x, y,
 *      size?: number,          // côté en blocs (si carré)
 *      w?: number, h?: number, // largeur/hauteur en blocs (option)
 *      trust: number,          // 0..1
 *      lastSeen: string,       // ex: "il y a 12 min"
 *      visitors: [{ name, img }], // derniers joueurs vus
 *      signals: string[],      // ex: ['chunk loads','portal']
 *      note?: string           // optionnel
 *    }]
 *  - query: string (filtre: nom, tags, visiteurs)
 *  - tileBaseUrl: e.g. "https://cloud.daporkchop.net/minecraft/2b2t/map/2021/256k/topdown"
 */
const props = defineProps({
  zones: { type: Array, default: () => [] },
  query: { type: String, default: '' },
  tileBaseUrl: { type: String, required: true },
  minZoom: { type: Number, default: 1 },
  maxZoom: { type: Number, default: 8 },
  initialZoom: { type: Number, default: 2 },

  // Échelles (topdown)
  pxPerBlock: { type: Number, default: 1 },

  // Isometric
  isoScale: { type: Number, default: 0.35 },
  isoYScale: { type: Number, default: 0.35 },
  isoViewScale: { type: Number, default: 0.10 },
})

const mapEl = ref(null)
let map, tileLayer
let zonesLayer, labelsLayer

// --- Mode déduit de l'URL des tuiles ---
const mode = ref(props.tileBaseUrl.includes('/isometric') ? 'isometric' : 'topdown')

// Watch: quand l’URL change, on bascule et on redessine
watch(() => props.tileBaseUrl, (url) => {
  mode.value = url.includes('/isometric') ? 'isometric' : 'topdown'
  if (tileLayer) tileLayer.redraw()
  if (map) map.setView(map.getCenter(), map.getZoom(), { animate: false })
  drawZones(filteredZones.value)
})

// ---- Quadkey 1..4
function quadPathFromXYZ(x, y, z) {
  const path = []
  for (let i = z; i >= 1; i--) {
    const mask = 1 << (i - 1)
    const bx = (x & mask) ? 1 : 0
    const by = (y & mask) ? 1 : 0
    path.push(by === 0 ? (bx === 0 ? 1 : 2) : (bx === 0 ? 3 : 4))
  }
  return path.join('/')
}

function buildTileUrl(coords) {
  const { x, y, z } = coords
  if (z < 1) return ''
  const n = 1 << z
  if (x < 0 || y < 0 || x >= n || y >= n) return ''
  return `${props.tileBaseUrl}/tl/${quadPathFromXYZ(x, y, z)}.png`
}

// ---- Custom tile layer
const CustomTileLayer = L.TileLayer.extend({
  getTileUrl(coords) { return buildTileUrl(coords) },
  createTile(coords, done) {
    const img = document.createElement('img')
    L.DomEvent.on(img, 'load', L.bind(this._tileOnLoad, this, done, img))
    L.DomEvent.on(img, 'error', L.bind(this._tileOnError, this, done, img))
    img.alt = ''
    img.decoding = 'async'
    img.loading = 'lazy'
    img.fetchPriority = 'low'
    img.referrerPolicy = 'no-referrer'
    img.crossOrigin = 'anonymous'
    img.src = this.getTileUrl(coords)
    return img
  }
})

// ---- Projections
function worldToPixelTopdown(p) {
  const s = props.pxPerBlock
  return { x: 100 + p.x * s, y: -100 - p.y * s }
}
function worldToPixelIso(p) {
  const s = props.isoScale, sy = props.isoYScale, k = props.isoViewScale
  const wx = p.x * k, wy = p.y * k
  return { x: 120 + (wx - wy) * s, y: -120 - (wx + wy) * s * sy }
}
function worldToLatLng(p) {
  const px = mode.value === 'isometric' ? worldToPixelIso(p) : worldToPixelTopdown(p)
  return L.latLng(px.y, px.x) // CRS.Simple: lat=y, lng=x
}

// ---- Données filtrées
const filteredZones = computed(() => {
  const q = (props.query || '').toLowerCase().trim()
  if (!q) return props.zones
  return props.zones.filter(z => {
    const hay = [
      z.name,
      ...(z.signals || []),
      ...(z.visitors || []).map(v => v.name),
      z.x, z.y
    ].join(' ').toLowerCase()
    return hay.includes(q)
  })
})

// ---- Styles & helpers
function trustColor(t) {
  // rouge -> jaune -> vert
  const r = t < 0.5 ? 255 : Math.round(255 * (1 - (t - 0.5) * 2))
  const g = t < 0.5 ? Math.round(255 * (t * 2)) : 255
  return `rgb(${r},${g},120)`
}
function zoneStyle(z, hover = false) {
  const col = trustColor(z.trust ?? 0.5)
  return {
    color: col,
    weight: hover ? 3 : 2,
    opacity: hover ? 0.95 : 0.75,
    fillColor: col,
    fillOpacity: hover ? 0.18 : 0.12,
    pane: 'zonesPane',
  }
}
function badgeTone(s) {
  const map = {
    'chunk loads': 'border-emerald-500/40 text-emerald-300',
    'portal': 'border-fuchsia-500/40 text-fuchsia-300',
    'elytra': 'border-sky-500/40 text-sky-300',
    'shulker': 'border-amber-500/40 text-amber-300',
    'beacon': 'border-cyan-500/40 text-cyan-300',
    'redstone': 'border-rose-500/40 text-rose-300',
    'rail': 'border-indigo-500/40 text-indigo-300',
  }
  return map[s] || 'border-zinc-600 text-zinc-300'
}

function zoomToPoly(poly, pad = 0.18) {
  const bounds = poly.getBounds().pad(pad) // marge autour
  map.flyToBounds(bounds, {
    duration: 0.6,
    easeLinearity: 0.25,
    maxZoom: Math.min(map.getMaxZoom(), (props.maxZoom ?? 8))
  })
}


// ---- Dessin des zones
function drawZones(list) {
  zonesLayer.clearLayers()
  labelsLayer.clearLayers()

  list.forEach(z => {
    // dimensions
    const w = z.size ?? z.w ?? 256
    const h = z.size ?? z.h ?? w

    // coins monde
    const minX = z.x - w / 2, maxX = z.x + w / 2
    const minY = z.y - h / 2, maxY = z.y + h / 2

    // coins projetés (ordre TL, TR, BR, BL)
    const tl = worldToLatLng({ x: minX, y: maxY })
    const tr = worldToLatLng({ x: maxX, y: maxY })
    const br = worldToLatLng({ x: maxX, y: minY })
    const bl = worldToLatLng({ x: minX, y: minY })

    // polygon (rect topdown, losange iso)
    const poly = L.polygon([tl, tr, br, bl], zoneStyle(z)).addTo(zonesLayer)

    // hover style
    poly.on('mouseover', () => poly.setStyle(zoneStyle(z, true)))
    poly.on('mouseout', () => poly.setStyle(zoneStyle(z, false)))

    // popup détail
    const trustPct = Math.round((z.trust ?? 0.5) * 100)
    const popupHtml = `
      <div class="text-xs min-w-[220px]">
        <div class="font-medium mb-1">${z.name || 'Zone suspecte'}</div>
        <div class="mb-1 text-[11px] text-zinc-400">X <b class="text-zinc-200">${Math.round(z.x)}</b> · Z <b class="text-zinc-200">${Math.round(z.y)}</b> — ${w}×${h} blocs</div>
        <div class="mb-2">
          <div class="text-zinc-400 mb-1">Fiabilité</div>
          <div class="h-2 rounded bg-zinc-800 overflow-hidden">
            <div class="h-full" style="width:${trustPct}%; background:${trustColor(z.trust ?? 0.5)}"></div>
          </div>
          <div class="mt-1 text-[11px] text-zinc-500">${trustPct}%</div>
        </div>
        ${(z.signals || []).length ? `
          <div class="mb-2 flex flex-wrap gap-1">
            ${(z.signals || []).map(s => `<span class="text-[10px] px-1.5 py-0.5 rounded border ${badgeTone(s)}">${s}</span>`).join('')}
          </div>` : ''}
        ${Array.isArray(z.visitors) && z.visitors.length ? `
          <div class="flex items-center gap-1">
            ${(z.visitors || []).slice(0, 5).map(v => `<img src="${v.img}" alt="${v.name}" class="h-6 w-6 rounded border border-zinc-700" />`).join('')}
            <span class="text-[11px] text-zinc-400 ml-1">Derniers visiteurs</span>
          </div>` : ''}
        ${z.lastSeen ? `<div class="mt-1 text-[11px] text-zinc-500">Dernier signal: <b class="text-zinc-300">${z.lastSeen}</b></div>` : ''}
        ${z.note ? `<div class="mt-2 text-[11px] text-zinc-400">${z.note}</div>` : ''}
      </div>
    `
    poly.bindPopup(popupHtml)

    poly.on('click', () => zoomToPoly(poly))

    // Zoom si on clique sur la popup (n’importe où dans son contenu)
    poly.on('popupopen', (e) => {
      const el = e.popup.getElement()
      if (el) el.addEventListener('click', () => zoomToPoly(poly), { once: true })
    })

    // label/marker au centre avec avatars & trust
    const center = worldToLatLng({ x: z.x, y: z.y })
    const avatarRow = (z.visitors || []).slice(0, 3).map((v, i) =>
      `<img src="${v.img}" alt="${v.name}" class="h-5 w-5 rounded border border-zinc-700 -ml-${i ? '2' : ''}" />`
    ).join('')
    const labelHtml = `
      <div class="rounded-lg bg-zinc-950/80 border border-zinc-800 px-2.5 py-2 text-[11px] text-zinc-300 shadow">
        <div class="flex items-center gap-2">
          <div class="font-medium text-zinc-100 truncate max-w-[140px]">${z.name || 'Zone'}</div>
          <div class="flex -space-x-2">${avatarRow}</div>
        </div>
        <div class="mt-1 h-1.5 rounded bg-zinc-800 overflow-hidden">
          <div class="h-full" style="width:${trustPct}%; background:${trustColor(z.trust ?? 0.5)}"></div>
        </div>
      </div>
    `
    const labelIcon = L.divIcon({
      html: labelHtml, className: 'zone-label', iconSize: null
    })
    const lab = L.marker(center, {
      icon: labelIcon,
      pane: 'labelsPane',
      interactive: true,
      keyboard: false,
      riseOnHover: true,
    }).addTo(labelsLayer)
    // ⬇️ Zoom au clic sur le label
    lab.on('click', () => zoomToPoly(poly))
    // (optionnel) accessibilité clavier
    lab.on('add', () => {
      const el = lab.getElement()
      if (el) {
        el.setAttribute('tabindex', '0')
        el.setAttribute('role', 'button')
        el.setAttribute('aria-label', `Zoom sur ${z.name || 'Zone'}`)
        el.addEventListener('keydown', (e) => {
          if (e.key === 'Enter' || e.key === ' ') zoomToPoly(poly)
        })
      }
    })
  })
}

// ---- Redraw util
function redraw() { drawZones(filteredZones.value) }

// ---- Lifecycle
onMounted(() => {
  map = L.map(mapEl.value, {
    crs: L.CRS.Simple,
    minZoom: props.minZoom ?? 1,
    maxZoom: props.maxZoom ?? 8,
    center: [0, 0],
    zoom: props.initialZoom ?? 2,
    zoomControl: false,
    attributionControl: false,
    zoomDelta: 1,
    wheelDebounceTime: 200,
  })
  L.control.zoom({ position: 'topright' }).addTo(map)

  tileLayer = new CustomTileLayer('', {
    tileSize: 256,
    noWrap: true,
    keepBuffer: 5,
    updateInterval: 200,
    updateWhenIdle: true,
    updateWhenZooming: false,
  }).addTo(map)

  // Pane zones (polygones) et labels (au-dessus)
  const zonesPane = map.createPane('zonesPane'); zonesPane.style.zIndex = 360
  const labelsPane = map.createPane('labelsPane'); labelsPane.style.zIndex = 560

  zonesLayer = L.layerGroup([], { pane: 'zonesPane' }).addTo(map)
  labelsLayer = L.layerGroup([], { pane: 'labelsPane' }).addTo(map)

  // Grille derrière les tuiles
  const gridPane = map.createPane('gridPane')
  gridPane.style.zIndex = 180
  gridPane.style.pointerEvents = 'none'
  const GridLayer = L.GridLayer.extend({
    createTile: function (coords) {
      const size = this.getTileSize()
      const canvas = document.createElement('canvas')
      canvas.width = size.x; canvas.height = size.y
      const ctx = canvas.getContext('2d')
      ctx.fillStyle = '#191818'; ctx.fillRect(0, 0, size.x, size.y)
      const spacing = 38
      const worldX = coords.x * size.x, worldY = coords.y * size.y
      let startX = - (worldX % spacing), startY = - (worldY % spacing)
      ctx.strokeStyle = 'rgba(255,255,255,0.12)'; ctx.lineWidth = 1
      for (let x = startX; x <= size.x; x += spacing) { ctx.beginPath(); ctx.moveTo(x + 0.5, 0); ctx.lineTo(x + 0.5, size.y); ctx.stroke() }
      for (let y = startY; y <= size.y; y += spacing) { ctx.beginPath(); ctx.moveTo(0, y + 0.5); ctx.lineTo(size.x, y + 0.5); ctx.stroke() }
      return canvas
    }
  })
  new GridLayer({ pane: 'gridPane', tileSize: 256, updateWhenIdle: true }).addTo(map)

  // premier rendu
  redraw()

  // sur pan/zoom, on garde les labels/zonings synchronisés (pas besoin de reprojeter, Leaflet gère)
  map.on('zoomend moveend', () => { })
})

onBeforeUnmount(() => { if (map) map.remove() })

// Réagir aux changements
watch(() => [props.zones, props.query], () => redraw(), { deep: true })
</script>

<template>
  <div ref="mapEl" class="h-[70vh] md:h-[72vh] rounded-b-2xl overflow-hidden"></div>
</template>

<style>
.zone-label {
  background: transparent;
  border: none;
}

.leaflet-container {
  background-color: #0a0a0b;
  background-image:
    linear-gradient(to right, rgba(255, 255, 255, 0.06) 1px, transparent 1px),
    linear-gradient(to bottom, rgba(255, 255, 255, 0.06) 1px, transparent 1px);
  background-size: 40px 40px;
  background-position: center;
}
</style>
