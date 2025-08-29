<template>
  <section class="rounded-2xl border border-zinc-800 bg-zinc-900/40">
    <!-- Toolbar -->
    <div class="flex flex-col sm:flex-row sm:items-center gap-3 p-3 border-b border-zinc-800">
      <div class="px-2 py-1 text-[10px] uppercase tracking-wider rounded bg-emerald-500/15 text-emerald-400 border border-emerald-500/30">
        Bases potentielles (fictif)
      </div>

      <div class="flex-1"></div>

      <div class="flex items-center gap-2">
        <input v-model="q" type="search" placeholder="Rechercher (nom, tag, signal)…"
               class="w-64 md:w-80 h-10 rounded-xl bg-zinc-950 border border-zinc-800 px-4 text-sm focus:outline-none focus:ring-2 focus:ring-emerald-500/50 focus:border-emerald-500"/>

        <select v-model="sortBy" class="h-10 rounded-xl bg-zinc-950 border border-zinc-800 px-3 text-sm">
          <option value="chests">Coffres estimés</option>
          <option value="trust">Fiabilité</option>
          <option value="activity">Activité</option>
          <option value="distance">Distance spawn</option>
          <option value="density">Densité (coffres/chunk)</option>
        </select>
      </div>
    </div>

    <!-- List -->
    <div class="divide-y divide-zinc-800">
      <article v-for="b in filtered" :key="b.id" class="p-4 hover:bg-zinc-900/40 transition">
        <div class="flex flex-col md:flex-row md:items-center gap-3">
          <!-- Title + coords -->
          <div class="flex-1 min-w-0">
            <div class="flex items-center gap-2">
              <h3 class="font-semibold truncate">
                {{ b.name }}
              </h3>
              <span class="text-[10px] px-1.5 py-0.5 rounded border border-zinc-700 text-zinc-300"
                    :class="badgeTone(b)">
                {{ b.category }}
              </span>
            </div>

            <div class="mt-1 flex flex-wrap items-center gap-x-3 gap-y-1 text-xs text-zinc-400">
              <button class="inline-flex items-center gap-1 hover:text-zinc-200"
                      @click="copyCoords(b)">
                <span class="font-mono text-zinc-200">X {{ fmtInt(b.coords.x) }}</span>
                <span class="font-mono text-zinc-200">Z {{ fmtInt(b.coords.z) }}</span>
                <span class="text-[10px] px-1 py-0.5 rounded bg-zinc-800/60 border border-zinc-700">Copier</span>
              </button>
              <span>Dist. spawn: <b class="text-zinc-200">{{ fmtKm(b.distance_k) }} km</b></span>
              <span>Dernier signal: <b class="text-zinc-200">{{ b.last_seen }}</b></span>
            </div>
          </div>

          <!-- KPI row -->
          <div class="grid grid-cols-2 md:grid-cols-5 gap-3 w-full md:w-auto">
            <div class="rounded-xl border border-zinc-800 bg-zinc-950/50 p-3 text-xs">
              <div class="text-zinc-400">Coffres</div>
              <div class="text-lg font-semibold">{{ fmtInt(b.chests_est) }}</div>
              <div class="mt-1 text-[11px] text-zinc-500">±{{ Math.round(b.chests_est * 0.12) }}</div>
            </div>

            <div class="rounded-xl border border-zinc-800 bg-zinc-950/50 p-3 text-xs">
              <div class="text-zinc-400">Densité</div>
              <div class="text-lg font-semibold">{{ b.density.toFixed(2) }}</div>
              <div class="mt-1 text-[11px] text-zinc-500">coffres/chunk</div>
            </div>

            <div class="rounded-xl border border-zinc-800 bg-zinc-950/50 p-3 text-xs">
              <div class="text-zinc-400">Fiabilité</div>
              <div class="mt-1 h-2 rounded bg-zinc-800 overflow-hidden">
                <div class="h-full" :style="{ width: (b.trust*100).toFixed(0)+'%', background: trustColor(b.trust) }"></div>
              </div>
              <div class="mt-1 text-[11px] text-zinc-500">{{ (b.trust*100).toFixed(0) }}%</div>
            </div>

            <div class="rounded-xl border border-zinc-800 bg-zinc-950/50 p-3 text-xs">
              <div class="text-zinc-400">Activité</div>
              <div class="h-10">
                <svg viewBox="0 0 100 30" preserveAspectRatio="none" class="w-full h-full">
                  <path :d="sparkPath(b.activity)" fill="none" stroke="rgba(16,185,129,0.9)" stroke-width="2"/>
                  <path :d="sparkFill(b.activity)" fill="rgba(16,185,129,0.15)"/>
                </svg>
              </div>
              <div class="mt-1 text-[11px] text-zinc-500">{{ b.activity[b.activity.length-1] }} pings/h</div>
            </div>

            <div class="rounded-xl border border-zinc-800 bg-zinc-950/50 p-3 text-xs">
              <div class="text-zinc-400">Signals</div>
              <div class="mt-1 flex flex-wrap gap-1">
                <span v-for="s in b.signals" :key="s"
                      class="text-[10px] px-1.5 py-0.5 rounded border"
                      :class="signalTone(s)">
                  {{ s }}
                </span>
              </div>
            </div>
          </div>
        </div>

        <!-- Meta row -->
        <div class="mt-3 grid sm:grid-cols-2 lg:grid-cols-4 gap-2 text-xs text-zinc-400">
          <div>Stash détectés: <b class="text-zinc-200">{{ b.stashes }}</b></div>
          <div>Joueurs vus (7j): <b class="text-zinc-200">{{ b.sightings_7d }}</b></div>
          <div>Portails récents: <b class="text-zinc-200">{{ b.portals_7d }}</b></div>
          <div>Redstone bruit: <b class="text-zinc-200">{{ b.redstone_noise }}</b> dB</div>
        </div>
      </article>
    </div>
  </section>
</template>

<script setup>
import { ref, computed } from 'vue'

// ---- Mock réaliste (10 bases)
const baseData = [
  mkBase(1, 'Obsidian Library',  18240,  -950,  16200, 2.41,  8.6, 'il y a 18 min',  14.2, ['chunk loads','portal','elytra'], 'Hold',  5, 12, 37),
  mkBase(2, 'Winterhold',       -33120,  22110, 21500, 3.02,  0.92, 'il y a 42 min', 36.8, ['chunk loads','shulker','beacon'], 'Active', 7, 18, 45),
  mkBase(3, 'SkyVault',          5200,  -48000, 11800, 1.75,  0.84, 'il y a 7 min',  48.1, ['elytra','portal'], 'Hidden', 3, 7, 22),
  mkBase(4, 'Tetra Spire',      -9050,   10400,  8200, 0.98,  0.78, 'il y a 3 h',    12.3, ['chunk loads','rail'], 'Ruins', 2, 2, 9),
  mkBase(5, 'Nether Yard',       6400,    6400, 15400, 2.89,  0.88, 'il y a 25 min',  9.1, ['portal','elytra','beacon'], 'Active', 6, 15, 31),
  mkBase(6, 'Quartz Garden',   -22000,  -22000,  9600, 1.34,  0.81, 'il y a 1 h',    31.7, ['shulker','chunk loads'], 'Hold', 4, 6, 18),
  mkBase(7, 'Bedrock Museum',    9800,   12900,  7400, 0.76,  0.87, 'il y a 11 min', 21.9, ['chunk loads','elytra'], 'Hidden', 2, 5, 14),
  mkBase(8, 'Lag Factory',     -41200,  -40200, 32800, 4.42,  0.75, 'il y a 2 min',  58.4, ['chunk loads','redstone','portal'], 'Critical', 12, 27, 63),
  mkBase(9, 'Ametrine Cove',    27100,  -33150,  6800, 0.69,  0.79, 'il y a 4 h',     6.3, ['elytra'], 'Hold', 1, 1, 6),
  mkBase(10,'The Archive',     -1500,     2600, 18950, 3.35,  0.91, 'il y a 33 min', 17.2, ['chunk loads','portal','shulker'], 'Active', 8, 19, 40),
]

// ---- État UI
const q = ref('')
const sortBy = ref('chests')

// ---- Dérivés
const filtered = computed(() => {
  const norm = (s) => s.toLowerCase()
  const hits = baseData.filter(b => {
    const hay = [
      b.name, b.category,
      b.signals.join(' '),
      b.coords.x, b.coords.z,
    ].join(' ').toString().toLowerCase()
    return hay.includes(norm(q.value))
  })

  const sorters = {
    chests:   (a,b) => b.chests_est - a.chests_est,
    trust:    (a,b) => b.trust - a.trust,
    activity: (a,b) => avg(b.activity) - avg(a.activity),
    distance: (a,b) => b.distance_k - a.distance_k,
    density:  (a,b) => b.density - a.density,
  }
  return hits.sort(sorters[sortBy.value])
})

// ---- Helpers & formatters
function mkBase(id, name, x, z, chests, density, trust, last_seen, dist_k, signals, category, stashes, portals7, sightings7) {
  return {
    id, name,
    coords: { x, z },
    chests_est: chests,
    density,               // coffres / chunk
    trust,                 // 0..1
    last_seen,             // texte
    distance_k: dist_k,    // km depuis spawn
    signals,               // tags
    category,              // Active / Hidden / Hold / Critical / Ruins
    stashes: stashes,
    portals_7d: portals7,
    sightings_7d: sightings7,
    redstone_noise: (Math.random()*8 + 36).toFixed(1),

    // série d’activité pings/h sur 24h
    activity: mkActivitySeries(),
  }
}

function mkActivitySeries() {
  let base = 6 + Math.random()*6
  const arr = []
  for (let i=0;i<24;i++){
    base += (Math.random()-0.45)*1.8
    arr.push(Math.max(0, Math.round(base + Math.random()*2)))
  }
  return arr
}

function avg(a){ return a.reduce((s,v)=>s+v,0)/a.length }

function fmtInt(n) {
  return new Intl.NumberFormat('fr-FR').format(n)
}
function fmtKm(n) {
  return (Math.round(n*10)/10).toLocaleString('fr-FR', { minimumFractionDigits: 1, maximumFractionDigits: 1 })
}

function trustColor(t){
  // gradient rouge -> jaune -> vert
  const r = t < 0.5 ? 255 : Math.round(255*(1-(t-0.5)*2))
  const g = t < 0.5 ? Math.round(255*(t*2)) : 255
  return `rgb(${r},${g},120)`
}

function badgeTone(b){
  const tones = {
    Active: 'border-emerald-500/40 text-emerald-300',
    Hidden: 'border-sky-500/40 text-sky-300',
    Hold: 'border-zinc-600 text-zinc-300',
    Critical: 'border-rose-500/40 text-rose-300',
    Ruins: 'border-amber-500/40 text-amber-300',
  }
  return tones[b.category] || 'border-zinc-600 text-zinc-300'
}

function signalTone(s){
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

function copyCoords(b){
  const text = `X ${b.coords.x} Z ${b.coords.z}`
  navigator.clipboard?.writeText(text)
}

function sparkPath(series){
  const n = series.length
  const max = Math.max(...series, 1)
  const W = 100, H = 30
  const step = W/(n-1)
  let d = ''
  series.forEach((v,i)=>{
    const x = i*step
    const y = H - (v/max)*H
    d += (i===0?`M${x},${y}`:` L${x},${y}`)
  })
  return d
}
function sparkFill(series){
  const n = series.length
  const max = Math.max(...series, 1)
  const W = 100, H = 30
  const step = W/(n-1)
  let d = ''
  series.forEach((v,i)=>{
    const x = i*step
    const y = H - (v/max)*H
    d += (i===0?`M${x},${y}`:` L${x},${y}`)
  })
  d += ` L100,30 L0,30 Z`
  return d
}
</script>
