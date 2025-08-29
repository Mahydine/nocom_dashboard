<script setup>
import { ref, computed } from 'vue'
import MapLeaflet from './components/MapLeaflet.vue'
import MapLeafletPotentialBases from './components/MapLeafletPotentialBases.vue'
import StatsBoard from './components/StatsBoard.vue'
import RadarScanner from './components/RadarScanner.vue'
import NocomCharts from './components/NocomCharts.vue'
import PotentialBasesList from './components/PotentialBasesList.vue'

const stats = ref({ packets: 12345678, chunks: 4321, radius: 128, nodes: 6 })
const barData = ref(Array.from({ length: 24 }, () => Math.floor(20 + Math.random() * 80)))
const heatData = ref(Array.from({ length: 12 }, () => Math.floor(Math.random() * 100)))

const query = ref('')

function randn() { // gaussien ~ N(0,1)
  let u = 0, v = 0
  while (u === 0) u = Math.random()
  while (v === 0) v = Math.random()
  return Math.sqrt(-2 * Math.log(u)) * Math.cos(2 * Math.PI * v)
}

function length(v) { return Math.hypot(v.x, v.y) }
function normalize(v) { const L = length(v) || 1; return { x: v.x / L, y: v.y / L } }
function add(a, b) { return { x: a.x + b.x, y: a.y + b.y } }
function scale(v, s) { return { x: v.x * s, y: v.y * s } }

// Marche aléatoire biaisée (start -> target)
function generateBiasedWalk(start, target, {
  steps = 120,
  stepMean = 28,    // moyenne taille de pas (blocs)
  stepStd = 9,     // dispersion taille de pas
  noise = 0.30,  // bruit transversal
  momentum = 0.75,  // lissage directionnel
  attract = 0.12,  // force d’attraction vers target
  jitter = 0.08,  // petit zigzag longitudinal
} = {}) {
  const pts = [{ x: start.x, y: start.y }]
  let vel = { x: 0, y: 0 }

  for (let i = 1; i <= steps; i++) {
    const cur = pts[pts.length - 1]
    const toTarget = { x: target.x - cur.x, y: target.y - cur.y }
    const dirTarget = normalize(toTarget)

    // vitesse désirée = attirer vers cible + un peu de zigzag
    const desired = add(
      scale(dirTarget, stepMean * (1 + jitter * randn())),
      { x: noise * stepMean * randn(), y: noise * stepMean * randn() }
    )

    // mélange momentum / desired
    vel = add(scale(vel, momentum), scale(desired, (1 - momentum)))

    // légère correction d’attraction directe
    vel = add(vel, scale(dirTarget, attract * stepMean))

    // normalise la vitesse finale autour de stepMean +/- stepStd
    const speed = stepMean + stepStd * randn()
    const dir = normalize(vel)
    const step = scale(dir, Math.max(6, speed)) // borne minimale

    const nxt = add(cur, step)
    pts.push(nxt)

    // si on est proche de la cible, on “freine” la fin
    if (length(toTarget) < stepMean * 1.2) {
      // termine proprement en atteignant la cible
      pts.push({ x: target.x, y: target.y })
      break
    }
  }
  return pts
}

// Démo joueurs avec trail indépendants & réalistes
// exemple positions de départ “près du centre”
const players = ref([
  { name: 'Ant0c07', x: 420, y: 240, img: 'https://minotar.net/helm/Ant0c07/48', lastSeen: 'il y a 3 min' },
  { name: 'Vyber57', x: -500, y: 810, img: 'https://minotar.net/helm/Vyber57/48', lastSeen: 'il y a 9 min' },
  { name: 'Bilzos', x: 900, y: -500, img: 'https://minotar.net/helm/Bilzos/48', lastSeen: 'il y a 1 min' },
  { name: 'Xernoz', x: -900, y: -700, img: 'https://minotar.net/helm/Xernoz/48', lastSeen: 'il y a 1 min' },
  { name: 'popbob', x: 800, y: 800, img: 'https://minotar.net/helm/MazMahydine/48', lastSeen: 'il y a 1 min' },
  { name: 'Ibralux', x: 600, y: 0, img: 'https://minotar.net/helm/Dinnerbone/48', lastSeen: 'il y a 1 min' },
  { name: 'Assales', x: 700, y: -800, img: 'https://minotar.net/helm/Dinnerbone/48', lastSeen: 'il y a 1 min' },
  { name: 'Hachimi', x: 250, y: 420, img: 'https://minotar.net/helm/classic_steve/48', lastSeen: 'maintenant' },
])


// cible aléatoire dans un rayon raisonnable
const MIN_DIST = 3000
const MAX_DIST = 3000

players.value = players.value.map(p => {
  const dist = MIN_DIST + (MAX_DIST - MIN_DIST) * Math.random()
  const ang = Math.random() * Math.PI * 2
  const target = { x: p.x + Math.cos(ang) * dist, y: p.y + Math.sin(ang) * dist }

  const trail = generateBiasedWalk({ x: p.x, y: p.y }, target, {
    steps: 50,        // moins d’étapes
    stepMean: 22,     // pas plus courts (blocs)
    stepStd: 7,
    noise: 4.25,
    momentum: 0.8,
    attract: 0.12,
    jitter: 0.08,
  })
  const last = trail[trail.length - 1]
  return { ...p, trail }
})


// Base des tuiles (daporkchop 2b2t 2021 256k topdown)
const tileBaseUrl = 'https://cloud.daporkchop.net/minecraft/2b2t/map/2021/256k/topdown'
const tileBaseUrlIso = 'https://cloud.daporkchop.net/minecraft/2b2t/map/2021/256k/isometric'

// --- Toggle Topdown / Isometric ---
const mode = ref('topdown') // 'topdown' | 'isometric'
const currentTileBaseUrl = computed(() =>
  mode.value === 'isometric' ? tileBaseUrlIso : tileBaseUrl
)
</script>

<template>
  <div class="min-h-screen flex bg-zinc-950 text-zinc-100">
    <!-- Sidebar -->
    <aside class="hidden md:flex md:w-64 flex-col border-r border-zinc-800 bg-zinc-950/60">
      <div class="p-4 border-b border-zinc-800 flex items-center gap-2">
        <div class="h-9 w-9 rounded-lg bg-emerald-500/20 grid place-content-center">
          <span class="text-emerald-400 font-bold">N</span>
        </div>
        <div>
          <div class="text-sm text-zinc-400">NOCOM MOCK</div>
          <div class="text-xs text-zinc-500">2b2t telemetry</div>
        </div>
      </div>
      <nav class="p-2 text-sm space-y-1">
        <a href="#" class="block px-3 py-2 rounded-lg bg-zinc-800/50">Aperçu</a>
        <a href="#" class="block px-3 py-2 rounded-lg hover:bg-zinc-900/60">Carte</a>
        <a href="#" class="block px-3 py-2 rounded-lg hover:bg-zinc-900/60">Flux</a>
        <a href="#" class="block px-3 py-2 rounded-lg hover:bg-zinc-900/60">Alertes</a>
        <a href="#" class="block px-3 py-2 rounded-lg hover:bg-zinc-900/60">Paramètres</a>
      </nav>
      <div class="mt-auto p-4 text-xs text-zinc-500">Interface fictive inspirée de “nocom”.</div>
    </aside>

    <!-- Main -->
    <main class="flex-1">
      <!-- Header -->
      <header class="sticky top-0 z-10 backdrop-blur bg-zinc-950/70 border-b border-zinc-800">
        <div class="max-w-7xl mx-auto px-4 py-3 flex items-center justify-between">
          <div class="flex items-center gap-3">
            <button
              class="md:hidden inline-flex items-center justify-center h-9 w-9 rounded-lg bg-zinc-900 border border-zinc-800">≡</button>
            <h1 class="text-lg md:text-2xl font-semibold tracking-tight">NoCom Admin – Dashboard</h1>
          </div>
          <div class="text-xs md:text-sm text-zinc-400">Abonnez vous à maz</div>
        </div>

      </header>

      <div class="max-w-7xl mx-auto px-4 py-6 space-y-6">
        <!-- KPIs -->
        <section class="grid grid-cols-2 lg:grid-cols-4 gap-3 md:gap-4">
          <div class="rounded-2xl border border-zinc-800 bg-zinc-900/40 p-4">
            <div class="text-xs text-zinc-400">Packets observés</div>
            <div class="mt-2 text-2xl font-semibold">{{ stats.packets.toLocaleString('fr-FR') }}</div>
            <div class="mt-3 h-10 flex items-end gap-1" aria-hidden="true">
              <div v-for="(v, i) in barData" :key="i" class="w-1.5 rounded-t bg-emerald-500/70"
                :style="{ height: v + '%' }"></div>
            </div>
          </div>
          <div class="rounded-2xl border border-zinc-800 bg-zinc-900/40 p-4">
            <div class="text-xs text-zinc-400">Chunks chargés</div>
            <div class="mt-2 text-2xl font-semibold">{{ stats.chunks }}</div>
            <div class="mt-3 grid grid-cols-6 gap-1" aria-hidden="true">
              <div v-for="(v, i) in heatData" :key="i" class="aspect-square rounded bg-sky-500/20"
                :style="{ opacity: (0.3 + v / 120) }"></div>
            </div>
          </div>
          <div class="rounded-2xl border border-zinc-800 bg-zinc-900/40 p-4">
            <div class="text-xs text-zinc-400">Rayon de tracking</div>
            <div class="mt-2 text-2xl font-semibold">{{ stats.radius }} blocs</div>
            <p class="mt-2 text-xs text-zinc-400">Fenêtre active</p>
          </div>
          <div class="rounded-2xl border border-zinc-800 bg-zinc-900/40 p-4">
            <div class="text-xs text-zinc-400">Nœuds en ligne</div>
            <div class="mt-2 text-2xl font-semibold">{{ stats.nodes }}</div>
            <p class="mt-2 text-xs text-zinc-400">Région overworld</p>
          </div>
        </section>

        <section class="grid grid-cols-1 lg:grid-cols-2 gap-4">
          <!-- Colonne 1 : radar -->
          <div class="col-span-1 flex items-start justify-center">
            <RadarScanner :rpm="12" :size="500" :blip-mean-ms="800" />
          </div>

          <!-- Colonne 2 : carte (wrap avec une classe pour fixer la hauteur du canvas leaflet) -->
          <aside class="col-span-1 space-y-4 map-card">
            <section class="rounded-2xl border border-zinc-800 bg-zinc-900/40">
              <div class="flex flex-col lg:flex-row lg:items-center gap-3 p-3 border-b border-zinc-800">
                <div class="flex-1 flex items-center gap-2">
                  <div
                    class="px-2 py-1 text-[10px] uppercase tracking-wider rounded bg-emerald-500/15 text-emerald-400 border border-emerald-500/30">
                    NOCOM MOCK
                  </div>
                </div>

                <div class="inline-flex rounded-xl border border-zinc-800 bg-zinc-900 p-1">
                  <button class="px-3 h-8 rounded-lg text-xs"
                    :class="mode === 'topdown' ? 'bg-emerald-600 text-white' : 'text-zinc-300 hover:bg-zinc-800'"
                    @click="mode = 'topdown'">
                    Topdown
                  </button>
                  <button class="px-3 h-8 rounded-lg text-xs"
                    :class="mode === 'isometric' ? 'bg-emerald-600 text-white' : 'text-zinc-300 hover:bg-zinc-800'"
                    @click="mode = 'isometric'">
                    Isometric
                  </button>
                </div>

                <div class="flex items-center gap-2">
                  <input v-model="query" type="search" placeholder="Rechercher un joueur…"
                    class="w-64 md:w-80 h-10 rounded-xl bg-zinc-950 border border-zinc-800 px-4 text-sm focus:outline-none focus:ring-2 focus:ring-emerald-500/50 focus:border-emerald-500" />
                </div>
              </div>

              <MapLeafletPotentialBases :zones="[
                {
                  id: 'z1', name: 'Winterhold Perimeter', x: 0, y: -400, size: 50, trust: 0.86, lastSeen: 'il y a 18 min',
                  visitors: [{ name: 'Steve', img: 'https://minotar.net/helm/Steve/24' }, { name: 'Alex', img: 'https://minotar.net/helm/Alex/24' }],
                  signals: ['chunk loads', 'portal']
                },
                {
                  id: 'z2', name: 'Lag Factory', x: -100, y: -100, w: 100, h: 100, trust: 0.74, lastSeen: 'il y a 2 min',
                  visitors: [{ name: 'Dinnerbone', img: 'https://minotar.net/helm/Dinnerbone/24' }],
                  signals: ['redstone', 'chunk loads']
                },
                {
                  id: 'z3', name: 'SkyVault Hint', x: 900, y: -20, size: 36, trust: 0.81,
                  visitors: [{ name: 'jeb_', img: 'https://minotar.net/helm/jeb_/24' }],
                  signals: ['elytra']
                },
              ]" :query="query" :tile-base-url="currentTileBaseUrl" :min-zoom="1" :max-zoom="10" :initial-zoom="2"
                :px-per-block="1" :iso-scale="0.35" :iso-y-scale="0.35" :iso-view-scale="0.10" />
            </section>
          </aside>
        </section>

        <!-- === NOUVEAU : KPIs nocom === -->
        <StatsBoard />

        <NocomCharts />

        <PotentialBasesList />

        <!-- Map + Search -->
        <section class="rounded-2xl border border-zinc-800 bg-zinc-900/40">
          <div class="flex flex-col lg:flex-row lg:items-center gap-3 p-3 border-b border-zinc-800">
            <div class="flex-1 flex items-center gap-2">
              <div
                class="px-2 py-1 text-[10px] uppercase tracking-wider rounded bg-emerald-500/15 text-emerald-400 border border-emerald-500/30">
                NOCOM MOCK
              </div>
            </div>
            <div class="inline-flex rounded-xl border border-zinc-800 bg-zinc-900 p-1">
              <button class="px-3 h-8 rounded-lg text-xs"
                :class="mode === 'topdown' ? 'bg-emerald-600 text-white' : 'text-zinc-300 hover:bg-zinc-800'"
                @click="mode = 'topdown'">
                Topdown
              </button>
              <button class="px-3 h-8 rounded-lg text-xs"
                :class="mode === 'isometric' ? 'bg-emerald-600 text-white' : 'text-zinc-300 hover:bg-zinc-800'"
                @click="mode = 'isometric'">
                Isometric
              </button>
            </div>
            <div class="flex items-center gap-2">
              <input v-model="query" type="search" placeholder="Rechercher un joueur…"
                class="w-64 md:w-80 h-10 rounded-xl bg-zinc-950 border border-zinc-800 px-4 text-sm focus:outline-none focus:ring-2 focus:ring-emerald-500/50 focus:border-emerald-500" />
            </div>
          </div>


          <MapLeaflet :players="players" :query="query" :tile-base-url="currentTileBaseUrl" :min-zoom="1" :max-zoom="10"
            :initial-zoom="1" />

        </section>

      </div>
    </main>
  </div>
</template>

<style>
.bg-grid {
  background-image:
    linear-gradient(to right, rgba(255, 255, 255, 0.08) 1px, transparent 1px),
    linear-gradient(to bottom, rgba(255, 255, 255, 0.08) 1px, transparent 1px);
  background-size: 40px 40px;
  background-position: center;
}

@media (min-width: 1024px) {
  .map-card .leaflet-container {
    height: 494px !important;
  }
}
</style>
