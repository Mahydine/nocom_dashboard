<script setup>
import { onMounted, onBeforeUnmount, watch, ref } from 'vue'
import L from 'leaflet'

/**
 * Props:
 *  - players: [{ name, x, y, img, lastSeen, trail: [{x,y}, ...] }]
 *  - query: string (filtre)
 *  - tileBaseUrl: e.g. "https://cloud.daporkchop.net/minecraft/2b2t/map/2021/256k/topdown"
 */
const props = defineProps({
  players: { type: Array, default: () => [] },
  query: { type: String, default: '' },
  tileBaseUrl: { type: String, required: true },
  minZoom: { type: Number, default: 0 },
  maxZoom: { type: Number, default: 7 },
  initialZoom: { type: Number, default: 3 },

  // Projection: 'topdown' ou 'isometric'
  projection: { type: String, default: 'topdown' },

  // Échelles
  // topdown: px/block au zoom max
  pxPerBlock: { type: Number, default: 1 },

  // isometric: screenX = (x - y) * isoScale
  //            screenY = (x + y) * isoScale * isoYScale
  isoScale: { type: Number, default: 0.35 },   // ~0.5 colle bien au 256k
  isoYScale: { type: Number, default: 0.35 },  // aplatissement vertical (2:1)

  // facteur d’échelle monde->pixels (tu peux l’ajuster à la main)
  pxPerBlock: { type: Number, default: 0.08 },

  // Compression visuelle (uniquement en isometric)
  isoViewScale: { type: Number, default: 0.10 }, // < 1 => plus rapproché
  // Longueur max des trails affichés en isometric (0 = illimité)
  isoTrailMax: { type: Number, default: 20 },

})

const mapEl = ref(null)
let map, tileLayer, markerLayer, trailSvgPane, trailSvg

// --- Mode déduit de l'URL des tuiles ---
const mode = ref(props.tileBaseUrl.includes('/isometric') ? 'isometric' : 'topdown')

// Watch: quand l’URL change, on bascule de mode et on redessine
watch(() => props.tileBaseUrl, (url) => {
  mode.value = url.includes('/isometric') ? 'isometric' : 'topdown'
  if (tileLayer) tileLayer.redraw()
  if (map) map.setView(map.getCenter(), map.getZoom(), { animate: false })
  drawPlayers(filteredPlayers())
})

function quadPathFromXYZ(x, y, z) {
  const path = []
  for (let i = z; i >= 1; i--) {
    const mask = 1 << (i - 1)
    const bx = (x & mask) ? 1 : 0   // 0 = gauche, 1 = droite
    const by = (y & mask) ? 1 : 0   // 0 = haut,   1 = bas   (Leaflet: y croît vers le bas)
    let digit
    if (by === 0 && bx === 0) digit = 1   // top-left
    else if (by === 0 && bx === 1) digit = 2 // top-right
    else if (by === 1 && bx === 0) digit = 3 // bottom-left
    else digit = 4                            // bottom-right
    path.push(digit)
  }
  return path.join('/') // ex: "1/2/4"
}

function buildTileUrl(coords) {
  const { x, y, z } = coords
  if (z < 1) return ''
  const n = 1 << z // 2^z tuiles par axe à ce niveau

  // hors du monde -> aucune tuile
  if (x < 0 || y < 0 || x >= n || y >= n) return ''

  const quadPath = quadPathFromXYZ(x, y, z)
  return `${props.tileBaseUrl}/tl/${quadPath}.png`
}


// Création d’un TileLayer custom (Leaflet n’accepte pas directement une fonction URL)
const CustomTileLayer = L.TileLayer.extend({
  getTileUrl(coords) {
    return buildTileUrl(coords)
  },
  createTile(coords, done) {
    const img = document.createElement('img')
    L.DomEvent.on(img, 'load', L.bind(this._tileOnLoad, this, done, img))
    L.DomEvent.on(img, 'error', L.bind(this._tileOnError, this, done, img))
    img.alt = ''
    img.decoding = 'async'
    img.loading = 'lazy'           // hint au navigateur
    img.fetchPriority = 'low'      // hint (support partiel, inoffensif)
    img.referrerPolicy = 'no-referrer'
    img.crossOrigin = 'anonymous'  // autorise le cache cross-origin
    img.src = this.getTileUrl(coords)
    return img
  }
})


function worldToPixelTopdown(p) {
  const s = props.pxPerBlock
  const xpx = 100 + p.x * s
  const ypx = -100 - p.y * s
  return { x: xpx, y: ypx }
}
function worldToPixelIso(p) {
  const s = props.isoScale
  const sy = props.isoYScale
  const k = props.isoViewScale // <— compression visuelle
  // isometric classique: rotation 45° + squash vertical, avec compression
  const wx = p.x * k
  const wy = p.y * k
  const xpx = 120 + (wx - wy) * s
  const ypx = -120 - (wx + wy) * s * sy
  return { x: xpx, y: ypx }
}
function worldToLatLng(p) {
  const px = mode.value === 'isometric'
    ? worldToPixelIso(p)
    : worldToPixelTopdown(p)
  // CRS.Simple: lat = y, lng = x
  return L.latLng(px.y, px.x)
}



function drawPlayers(filtered) {
  markerLayer.clearLayers()

  // Trails en SVG : on dessine dans un <svg> lié au pane
  // reset contenu
  trailSvg.innerHTML = ''

  filtered.forEach(p => {
    // --- trail (polyline SVG) ---
    if (Array.isArray(p.trail) && p.trail.length > 1) {
      const pts = p.trail.map(pt => worldToLatLng(pt))
        .map(ll => map.latLngToLayerPoint(ll))
      const d = pts.map((pt, i) => (i ? `L${pt.x},${pt.y}` : `M${pt.x},${pt.y}`)).join(' ')
      const glow = document.createElementNS('http://www.w3.org/2000/svg', 'path')
      glow.setAttribute('d', d)
      glow.setAttribute('fill', 'none')
      glow.setAttribute('stroke', 'rgba(255, 0, 0,0.18)') // emerald-500/20
      glow.setAttribute('stroke-width', '10')
      glow.setAttribute('stroke-linecap', 'round')
      glow.setAttribute('stroke-linejoin', 'round')
      trailSvg.appendChild(glow)

      const line = document.createElementNS('http://www.w3.org/2000/svg', 'path')
      line.setAttribute('d', d)
      line.setAttribute('fill', 'none')
      line.setAttribute('stroke', 'rgba(255, 0, 0,0.95)')
      line.setAttribute('stroke-width', '2.5')
      line.setAttribute('stroke-linecap', 'round')
      line.setAttribute('stroke-linejoin', 'round')
      trailSvg.appendChild(line)
    }

    // --- marker (avatar dans divIcon) ---
    const iconHtml = `
      <div class="relative group">
        <img src="${p.img}" alt="${p.name}"
             class="h-10 w-10 rounded-lg border border-zinc-700 shadow" />
        <div class="absolute -bottom-6 left-1/2 -translate-x-1/2 text-[10px]
                    bg-zinc-900/80 border border-zinc-800 px-1.5 py-0.5 rounded
                    text-white whitespace-nowrap">
          ${p.name}
        </div>
      </div>`
    const icon = L.divIcon({
      html: iconHtml,
      className: 'mc-marker', // on neutralise les styles par défaut
      iconSize: [40, 40],
      iconAnchor: [20, 20],
    })

    const m = L.marker(worldToLatLng(p), { icon })
    m.addTo(markerLayer)
    // simple popup
    m.bindPopup(`
      <div class="text-xs">
        <div class="font-medium mb-1">${p.name}</div>
        <div>X: <b>${p.x}</b> &nbsp; Y: <b>${p.y}</b></div>
        <div>Dernier ping: <b>${p.lastSeen || '—'}</b></div>
      </div>
    `)
  })
}

function filteredPlayers() {
  const q = (props.query || '').toLowerCase().trim()
  if (!q) return props.players
  return props.players.filter(p => p.name.toLowerCase().includes(q))
}

function redrawTrailsOnZoomOrMove() {
  // Reprojeter les points (latLngToLayerPoint change avec zoom/pan)
  drawPlayers(filteredPlayers())
}

onMounted(() => {
  // Carte
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

  tileLayer = new CustomTileLayer('', {
    tileSize: 256,
    noWrap: true,
    keepBuffer: 8,
    updateInterval: 200,
    updateWhenIdle: true,

    keepBuffer: 2,              // garde peu de tuiles hors-écran (réduit le débit)
    updateWhenIdle: true,       // ne rafraîchit qu'à la fin du pan
    updateWhenZooming: false,   // évite de spam aux zooms continus
    // ne demande pas au-delà de ce que le serveur a
  })
  tileLayer.addTo(map)

  // Pane pour SVG trails (dessiné par dessus les tuiles)
  trailSvgPane = map.createPane('trailsPane')
  trailSvgPane.style.zIndex = 450 // sous les markers (par défaut 600)
  trailSvg = L.svg({ pane: 'trailsPane' }).addTo(map)._container // récupère le <svg>

  // Calque markers
  markerLayer = L.layerGroup().addTo(map)

  // Premier rendu
  drawPlayers(filteredPlayers())

  // Recalcul sur pan/zoom (reprojection des trails)
  map.on('zoomend moveend', redrawTrailsOnZoomOrMove)
  // Crée un pane sous les tuiles (tilePane est ~z-index:200)
  const gridPane = map.createPane('gridPane')
  gridPane.style.zIndex = 180   // < 200 => derrière les tuiles
  gridPane.style.pointerEvents = 'none'

  // GridLayer qui dessine un fond sombre + lignes de grille, aligné monde
  const GridLayer = L.GridLayer.extend({
    createTile: function (coords) {
      const size = this.getTileSize()
      const canvas = document.createElement('canvas')
      canvas.width = size.x
      canvas.height = size.y
      const ctx = canvas.getContext('2d')

      // fond sombre
      ctx.fillStyle = '#191818'
      ctx.fillRect(0, 0, size.x, size.y)

      // grille (répétée proprement sans joints entre tuiles)
      const spacing = 38 // px entre lignes (ajuste comme tu veux)

      // origine monde (en px) du coin haut-gauche de cette tuile
      const worldX = coords.x * size.x
      const worldY = coords.y * size.y

      // décalage pour tomber pile sur le “quadrillage monde”
      let startX = - (worldX % spacing)
      let startY = - (worldY % spacing)

      ctx.strokeStyle = 'rgba(255,255,255,0.12)'
      ctx.lineWidth = 1

      // lignes verticales
      for (let x = startX; x <= size.x; x += spacing) {
        ctx.beginPath()
        ctx.moveTo(x + 0.5, 0)             // +0.5 pour une ligne nette
        ctx.lineTo(x + 0.5, size.y)
        ctx.stroke()
      }
      // lignes horizontales
      for (let y = startY; y <= size.y; y += spacing) {
        ctx.beginPath()
        ctx.moveTo(0, y + 0.5)
        ctx.lineTo(size.x, y + 0.5)
        ctx.stroke()
      }

      return canvas
    }
  })

  // ajoute la grille dans le pane sous-jacent
  const gridLayer = new GridLayer({
    pane: 'gridPane',
    tileSize: 256,
    updateWhenZooming: false,
    updateWhenIdle: true,
  })
  gridLayer.addTo(map)
})

onBeforeUnmount(() => {
  if (map) map.remove()
})

// Réagir aux changements de liste / filtre
watch(() => [props.players, props.query], () => {
  drawPlayers(filteredPlayers())
}, { deep: true })
</script>

<template>
  <div ref="mapEl" class="h-[70vh] md:h-[72vh] rounded-b-2xl overflow-hidden"></div>
</template>

<style>
/* Neutralise le style par défaut des markers Leaflet pour nos divIcon */
.mc-marker {
  background: transparent;
  border: none;
}

.leaflet-container {
  background-color: #0a0a0b;
  /* fond sombre */
  background-image:
    linear-gradient(to right, rgba(255, 255, 255, 0.06) 1px, transparent 1px),
    linear-gradient(to bottom, rgba(255, 255, 255, 0.06) 1px, transparent 1px);
  background-size: 40px 40px;
  /* taille des carreaux */
  background-position: center;
}


/* fond sombre derrière les tuiles si manque */
</style>
