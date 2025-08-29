<template>
  <section class="rounded-2xl border border-zinc-800 bg-zinc-900/40">
    <!-- Header / toolbar -->
    <header class="flex items-center gap-2 p-3 border-b border-zinc-800">
      <div class="px-2 py-1 text-[10px] uppercase tracking-wider rounded bg-emerald-500/15 text-emerald-400 border border-emerald-500/30">
        Console
      </div>
      <div class="text-xs text-zinc-400">NoCom I/O monitor</div>
      <div class="ml-auto flex items-center gap-2">
        <button @click="togglePause"
                class="h-8 px-3 rounded-lg border border-zinc-700 text-xs"
                :class="paused ? 'bg-zinc-800 text-zinc-300' : 'bg-emerald-600 text-white'">
          {{ paused ? 'Reprendre' : 'Pause bots' }}
        </button>
        <button @click="clearLogs" class="h-8 px-3 rounded-lg border border-zinc-700 text-xs hover:bg-zinc-800">Clear</button>
        <label class="flex items-center gap-2 text-xs text-zinc-400">
          <input type="checkbox" v-model="autoScroll" class="accent-emerald-500">
          Auto-scroll
        </label>
      </div>
    </header>

    <!-- Log area -->
    <div ref="logEl"
         class="font-mono text-[12px] leading-relaxed px-3 py-2 overflow-auto bg-[#0a0f0d]"
         :style="{ height: height + 'px' }">
      <template v-for="item in logs" :key="item.id">
        <div class="whitespace-pre-wrap">
          <span class="text-zinc-500">[{{ item.time }}]</span>
          <span class="mx-2 rounded px-1 py-0.5"
                :class="badgeClass(item.level)">
            {{ item.level }}
          </span>
          <span class="text-emerald-300">{{ item.src }}</span>
          <span class="text-zinc-400"> › </span>
          <span class="text-zinc-200">{{ item.text }}</span>
        </div>
      </template>
    </div>

    <!-- Prompt -->
    <form class="flex items-center gap-2 p-3 border-t border-zinc-800" @submit.prevent="submitCmd">
      <span class="font-mono text-[12px] text-emerald-400">nocom@node-{{ nodeId }}&gt;</span>
      <input v-model="cmd"
             type="text"
             :placeholder="placeholder"
             class="flex-1 bg-zinc-950 border border-zinc-800 rounded-md px-3 py-2 text-sm text-zinc-200 focus:outline-none focus:ring-2 focus:ring-emerald-500/40"
             autocomplete="off" />
      <button class="h-9 px-3 rounded-md bg-emerald-600 text-white text-sm border border-emerald-500/60 hover:bg-emerald-500">
        Exécuter
      </button>
    </form>
  </section>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, nextTick } from 'vue'

/** Props */
const props = defineProps({
  height: { type: Number, default: 420 },          // hauteur en px
  minDelay: { type: Number, default: 900 },        // min ms entre messages bots
  maxDelay: { type: Number, default: 3200 },       // max ms
  showTimestamps: { type: Boolean, default: true } // (garde pour évolutions)
})

/** State */
const logs = ref([]) // {id, time, level, src, text}
const cmd = ref('')
const autoScroll = ref(true)
const paused = ref(false)
const nodeId = Math.floor(1 + Math.random() * 8)   // 1..8
const logEl = ref(null)
let botTimer = null
let idCounter = 0

const placeholder = 'help | bots | stats | scan 123 -456 | where Notch | clear | pause/resume'

/** Utils */
function nowStr() {
  const d = new Date()
  const h = String(d.getHours()).padStart(2, '0')
  const m = String(d.getMinutes()).padStart(2, '0')
  const s = String(d.getSeconds()).padStart(2, '0')
  return `${h}:${m}:${s}`
}
function rnd(min, max) { return Math.floor(min + Math.random() * (max - min + 1)) }
function rndChoice(arr) { return arr[Math.floor(Math.random()*arr.length)] }
function fmtInt(n) { return new Intl.NumberFormat('fr-FR').format(n) }

function pushLog({ level = 'INFO', src = 'system', text }) {
  logs.value.push({ id: ++idCounter, time: nowStr(), level, src, text })
  // cap mémoire
  if (logs.value.length > 800) logs.value.splice(0, logs.value.length - 800)
  if (autoScroll.value) nextTick(() => {
    const el = logEl.value
    if (el) el.scrollTop = el.scrollHeight
  })
}

function badgeClass(level) {
  return {
    'INFO':  'bg-emerald-500/15 text-emerald-300 border border-emerald-500/30',
    'WARN':  'bg-amber-500/15 text-amber-300 border border-amber-500/30',
    'ALERT': 'bg-rose-500/15 text-rose-300 border border-rose-500/30',
  }[level] || 'bg-zinc-700/20 text-zinc-300 border border-zinc-600/40'
}

/** Bots generator */
function scheduleNextBot() {
  clearTimeout(botTimer)
  const delay = rnd(props.minDelay, props.maxDelay)
  botTimer = setTimeout(() => {
    if (!paused.value) pushLog(genBotEvent())
    scheduleNextBot()
  }, delay)
}

function genBotEvent() {
  const botId = String(rnd(1, 20)).padStart(2, '0')
  const src = `bot_${botId}`

  // scénarios plausibles
  const scenarios = [
    () => {
      const x = rnd(-50000, 50000), z = rnd(-50000, 50000)
      const p = rndChoice(['Steve','Alex','Notch','jeb_','Dinnerbone','popbob'])
      return { level: 'INFO', text: `joueur détecté: ${p} @ X=${x} Z=${z} · lat=${(Math.random()*120).toFixed(1)}ms` }
    },
    () => {
      const n = fmtInt(rnd(12, 180))
      return { level: 'WARN', text: `burst de paquets mouvement: ${n}/s · heuristique anti-noise appliquée` }
    },
    () => {
      const c = fmtInt(rnd(8, 240))
      const x = rnd(-30000, 30000), z = rnd(-30000, 30000)
      return { level: 'INFO', text: `chunk loads anormaux près de X=${x} Z=${z} · fenêtre 2m · ${c} événements` }
    },
    () => {
      const d = (Math.random()*60+20).toFixed(1)
      return { level: 'WARN', text: `bruit redstone ↑ ${d} dB · filtre passe-bande engagé` }
    },
    () => {
      const ch = fmtInt(rnd(40, 1200))
      return { level: 'INFO', text: `diff chests @ region - mise à jour inventaire: +${ch} conteneurs` }
    },
    () => {
      const x = rnd(-40000, 40000), z = rnd(-40000, 40000)
      return { level: 'ALERT', text: `activation portail nether triangulée · overworld ≈ X=${x} Z=${z}` }
    },
    () => {
      const n = rnd(2,5)
      return { level: 'INFO', text: `transfert shulker détecté · ${n} entités conteneurs en série` }
    },
    () => {
      const km = (Math.random()*60+5).toFixed(1)
      return { level: 'INFO', text: `piste Elytra confirmée · corrélation directionnelle ${km} km` }
    },
    () => {
      const p = rndChoice(['rail','beacon','portal','elytra'])
      const conf = (Math.random()*0.2+0.75).toFixed(2)
      return { level: 'INFO', text: `signal "${p}" → confiance=${conf}` }
    },
    () => {
      const ms = rnd(40, 160)
      return { level: 'WARN', text: `latence cible ↑ ${ms}ms · bascule sur nœud relais` }
    },
  ]
  const chosen = rndChoice(scenarios)()
  return { level: chosen.level, src, text: chosen.text }
}

/** Commands */
function clearLogs() { logs.value = [] }
function togglePause() { paused.value = !paused.value }

async function submitCmd() {
  const raw = cmd.value.trim()
  if (!raw) return
  cmd.value = ''
  pushLog({ src: 'user', level: 'INFO', text: `> ${raw}` })

  // faux chargement
  const ticket = Math.random().toString(36).slice(2, 7)
  pushLog({ src: 'system', level: 'INFO', text: `job ${ticket} • exécution…` })

  // petite latence simulée
  await new Promise(r => setTimeout(r, 600 + Math.random()*800))

  const [c, ...args] = raw.split(/\s+/)
  switch ((c || '').toLowerCase()) {
    case 'help':
      pushLog({ src: 'system', level: 'INFO', text: 'cmds: help, bots, stats, scan <x> <z>, where <player>, pause, resume, clear' })
      break
    case 'bots': {
      const active = rnd(12, 20)
      pushLog({ src: 'system', level: 'INFO', text: `${active}/20 bots actifs · débit ≈ ${fmtInt(rnd(1200,4200))} msg/h` })
      break
    }
    case 'stats':
      pushLog({ src: 'system', level: 'INFO', text: `paquets: ${fmtInt(8400000)} · chunks profilés: ${fmtInt(42130)} · coords valides: 1823 · faux positifs: 1.6%` })
      break
    case 'scan': {
      const x = Number(args[0]), z = Number(args[1])
      if (Number.isFinite(x) && Number.isFinite(z)) {
        pushLog({ src: 'scanner', level: 'INFO', text: `analyse X=${x} Z=${z}… fenêtres 90s` })
        await new Promise(r => setTimeout(r, 700 + Math.random()*1000))
        const found = rnd(0, 3)
        pushLog({ src: 'scanner', level: found ? 'INFO' : 'WARN', text: found ? `candidats: ${found} · confiance max ${(0.72+Math.random()*0.23).toFixed(2)}` : 'aucun signal exploitable' })
      } else {
        pushLog({ src: 'system', level: 'WARN', text: 'usage: scan <x> <z>' })
      }
      break
    }
    case 'where': {
      const p = args[0] || 'Steve'
      const x = rnd(-50000, 50000), z = rnd(-50000, 50000)
      pushLog({ src: 'resolver', level: 'INFO', text: `${p} ≈ X=${x} Z=${z} · corrélation 0.${rnd(70,95)}` })
      break
    }
    case 'pause':
      paused.value = true; pushLog({ src: 'system', level: 'INFO', text: 'bots: pause' }); break
    case 'resume':
      paused.value = false; scheduleNextBot(); pushLog({ src: 'system', level: 'INFO', text: 'bots: reprise' }); break
    case 'clear':
      clearLogs(); break
    default:
      pushLog({ src: 'system', level: 'WARN', text: `commande inconnue: "${raw}" (help)` })
  }
}

onMounted(() => {
  pushLog({ src: 'system', level: 'INFO', text: `nœud ${nodeId} initialisé · ${rnd(8,16)} watchers` })
  pushLog({ src: 'system', level: 'INFO', text: 'tape "help" pour la liste des commandes' })
  scheduleNextBot()
})

onBeforeUnmount(() => { clearTimeout(botTimer) })
</script>
