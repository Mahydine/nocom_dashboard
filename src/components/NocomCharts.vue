<template>
  <section class="grid lg:grid-cols-2 gap-4">
    <!-- Line: Detections par heure -->
    <div class="rounded-2xl border border-zinc-800 bg-zinc-900/40 p-4">
      <div class="text-sm text-zinc-400 mb-2">Taux de détection (24h)</div>
      <div class="h-56">
        <Line :data="lineData" :options="lineOpts" />
      </div>
    </div>

    <!-- Bar: volume par type de paquet -->
    <div class="rounded-2xl border border-zinc-800 bg-zinc-900/40 p-4">
      <div class="text-sm text-zinc-400 mb-2">Volume par type de paquet (24h)</div>
      <div class="h-56">
        <Bar :data="barData" :options="barOpts" />
      </div>
    </div>

    <!-- Doughnut: mix des sources -->
    <div class="rounded-2xl border border-zinc-800 bg-zinc-900/40 p-4">
      <div class="text-sm text-zinc-400 mb-2">Mix des sources</div>
      <div class="h-56">
        <Doughnut :data="donutData" :options="donutOpts" />
      </div>
    </div>

    <!-- Radar: qualité des signaux -->
    <div class="rounded-2xl border border-zinc-800 bg-zinc-900/40 p-4">
      <div class="text-sm text-zinc-400 mb-2">Qualité des signaux</div>
      <div class="h-56">
        <Radar :data="radarData" :options="radarOpts" />
      </div>
    </div>
  </section>
</template>

<script setup>
import { ref } from 'vue'
import { Line, Bar, Doughnut, Radar } from 'vue-chartjs'
import {
  Chart, LineElement, PointElement, LineController,
  CategoryScale, LinearScale, BarElement, BarController,
  ArcElement, RadialLinearScale, Filler, Tooltip, Legend,
} from 'chart.js'

Chart.register(
  LineElement, PointElement, LineController,
  CategoryScale, LinearScale,
  BarElement, BarController,
  ArcElement, RadialLinearScale,
  Filler, Tooltip, Legend
)

// Helpers
const hours = Array.from({ length: 24 }, (_, i) => (i < 10 ? `0${i}` : `${i}`))

// LINE — Détections/heure (un peu de bruit mais tendance crédible)
const lineSeries = (() => {
  let base = 8, arr = []
  for (let i = 0; i < 24; i++) {
    base += (Math.random() - 0.45) * 2.2
    arr.push(Math.max(2, Math.round(base + Math.random() * 3)))
  }
  return arr
})()

const lineData = ref({
  labels: hours,
  datasets: [{
    label: 'Détections',
    data: lineSeries,
    fill: true,
    tension: 0.35,
    borderWidth: 2,
    borderColor: 'rgba(16,185,129,0.9)',
    backgroundColor: 'rgba(16,185,129,0.15)',
    pointRadius: 0,
  }]
})
const commonGrid = {
  color: 'rgba(148,163,184,0.15)',
  borderDash: [4, 4]
}
const lineOpts = ref({
  responsive: true, maintainAspectRatio: false,
  plugins: { legend: { display: false }, tooltip: { mode: 'index', intersect: false } },
  scales: {
    x: { grid: commonGrid, ticks: { color: '#cbd5e1', maxTicksLimit: 8 } },
    y: { grid: commonGrid, ticks: { color: '#cbd5e1' }, suggestedMin: 0 }
  }
})

// BAR — Volume par type de paquet (24h)
const barData = ref({
  labels: ['KeepAlive', 'Move', 'Chunk', 'Login', 'Entity'],
  datasets: [{
    label: 'Paquets (k)',
    data: [320, 540, 210, 35, 110],
    backgroundColor: 'rgba(59,130,246,0.35)',
    borderColor: 'rgba(59,130,246,0.9)',
    borderWidth: 1
  }]
})
const barOpts = ref({
  responsive: true, maintainAspectRatio: false,
  plugins: { legend: { display: false } },
  scales: {
    x: { grid: { display: false }, ticks: { color: '#cbd5e1' } },
    y: { grid: commonGrid, ticks: { color: '#cbd5e1' }, suggestedMin: 0 }
  }
})

// DOUGHNUT — Mix des sources
const donutData = ref({
  labels: ['Sniff direct', 'Caches CDN', 'Logs proxy', 'Heuristiques'],
  datasets: [{
    data: [46, 28, 14, 12],
    backgroundColor: [
      'rgba(16,185,129,0.8)',
      'rgba(59,130,246,0.8)',
      'rgba(234,179,8,0.8)',
      'rgba(244,63,94,0.8)',
    ],
    borderColor: 'rgba(2,6,23,1)', // pour fond sombre
    borderWidth: 2
  }]
})
const donutOpts = ref({
  responsive: true, maintainAspectRatio: false,
  plugins: { legend: { position: 'bottom', labels: { color: '#cbd5e1' } } }
})

// RADAR — Qualité des signaux
const radarData = ref({
  labels: ['Latence', 'Précision', 'Couverture', 'Cohérence', 'Confiance'],
  datasets: [{
    label: 'Score',
    data: [72, 88, 64, 81, 90],
    backgroundColor: 'rgba(99,102,241,0.2)',
    borderColor: 'rgba(99,102,241,0.9)',
    borderWidth: 2,
    pointRadius: 2
  }]
})
const radarOpts = ref({
  responsive: true, maintainAspectRatio: false,
  plugins: { legend: { display: false } },
  scales: {
    r: {
      angleLines: { color: 'rgba(148,163,184,0.15)' },
      grid: { color: 'rgba(148,163,184,0.15)' },
      pointLabels: { color: '#cbd5e1' },
      ticks: { display: false, beginAtZero: true, max: 100 }
    }
  }
})
</script>
