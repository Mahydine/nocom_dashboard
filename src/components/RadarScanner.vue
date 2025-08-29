<template>
    <div class="rounded-2xl border border-zinc-800 bg-zinc-900/40 p-4 w-max">
        <div class="text-sm text-zinc-400 mb-2">Scanner radar</div>

        <!-- conteneur carré responsive -->
        <div ref="containerEl" class="relative rounded-xl overflow-hidden bg-[#0a120e] mx-auto"
            :style="{ width: props.size + 'px', height: props.size + 'px' }">
            <canvas ref="canvasEl" class="absolute inset-0 w-full h-full"></canvas>

            <!-- léger vignettage pour style CRT -->
            <div class="pointer-events-none absolute inset-0"
                style="background: radial-gradient(ellipse at center, rgba(0,0,0,0) 50%, rgba(0,0,0,.45) 100%);"></div>
        </div>
    </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'

/**
 * Props (optionnelles)
 * - rpm: rotations par minute de la barre (par défaut 12 ≈ 5 s / tour)
 * - blipMeanMs: moyenne (ms) entre blips (poisson) — plus petit => plus de blips
 * - color: couleur principale (barre & blips)
 * - gridColor: couleur de la grille
 */
const props = defineProps({
    rpm: { type: Number, default: 12 },
    blipMeanMs: { type: Number, default: 2300 },
    color: { type: String, default: '#34d399' },     // emerald-400
    gridColor: { type: String, default: 'rgba(52,211,153,0.22)' },
    size: { type: Number, default: 240 }, // taille du carré en px
})

const containerEl = ref(null)
const canvasEl = ref(null)

let ctx, rafId, ro, lastTs = 0
let angle = 0 // radians
let blips = [] // { r:0..1, theta:rad, birth:ms, life:ms, size: px }
let nextBlipTimer = null
let steveImg, steveReady = false

function expRand(meanMs) {
    // tirage loi exponentielle (poisson process inter-arrival)
    return -Math.log(1 - Math.random()) * meanMs
}

function scheduleNextBlip() {
    clearTimeout(nextBlipTimer)
    nextBlipTimer = setTimeout(() => {
        spawnBlipNear(angle)
        scheduleNextBlip()
    }, expRand(props.blipMeanMs))
}

function spawnBlipNear(refAngle) {
    // un blip pas trop loin de la barre
    const theta = refAngle + (Math.random() - 0.5) * 0.35 // ± ~20°
    // densité un peu plus forte au milieu
    const r = Math.pow(Math.random(), 0.6) // bias vers l'extérieur (0..1)
    const life = 1200 + Math.random() * 1200
    const size = 3 + Math.random() * 2
    blips.push({ r, theta, birth: performance.now(), life, size })
    // garde la file courte
    if (blips.length > 120) blips = blips.slice(-120)
}

function draw(ts) {
    const dpr = window.devicePixelRatio || 1
    const w = canvasEl.value.width / dpr
    const h = canvasEl.value.height / dpr
    const cx = w / 2
    const cy = h / 2
    const radius = Math.min(w, h) * 0.48

    // temps
    if (!lastTs) lastTs = ts
    const dt = (ts - lastTs) / 1000
    lastTs = ts

    // avance la barre (rpm -> rad/s)
    const omega = (props.rpm * 2 * Math.PI) / 60
    angle = (angle + omega * dt) % (Math.PI * 2)

    // clear
    ctx.clearRect(0, 0, w, h)

    // fond
    ctx.fillStyle = '#0a120e'
    ctx.fillRect(0, 0, w, h)

    ctx.save()
    ctx.translate(cx, cy)

    // grille: cercles
    ctx.strokeStyle = props.gridColor
    ctx.lineWidth = 1
    const rings = 5
    for (let i = 1; i <= rings; i++) {
        ctx.beginPath()
        ctx.arc(0, 0, (i / rings) * radius, 0, Math.PI * 2)
        ctx.stroke()
    }
    // grille: rayons
    ctx.save()
    ctx.globalAlpha = 0.7
    const spokes = 12
    for (let i = 0; i < spokes; i++) {
        const a = (i / spokes) * Math.PI * 2
        ctx.beginPath()
        ctx.moveTo(0, 0)
        ctx.lineTo(Math.cos(a) * radius, Math.sin(a) * radius)
        ctx.stroke()
    }
    ctx.restore()

    // sweep (arc avec dégradé)
    const sweepWidth = Math.PI / 26 // ~7°
    const grad = ctx.createRadialGradient(0, 0, radius * 0.0, 0, 0, radius)
    grad.addColorStop(0, 'rgba(52,211,153,0.0)')
    grad.addColorStop(1, 'rgba(52,211,153,0.28)')
    ctx.fillStyle = grad
    ctx.beginPath()
    ctx.moveTo(0, 0)
    ctx.arc(0, 0, radius, angle - sweepWidth, angle)
    ctx.closePath()
    ctx.fill()

    // rayon principal (ligne brillante)
    ctx.save()
    ctx.shadowColor = props.color
    ctx.shadowBlur = 12
    ctx.strokeStyle = props.color
    ctx.lineWidth = 2
    ctx.beginPath()
    ctx.moveTo(0, 0)
    ctx.lineTo(Math.cos(angle) * radius, Math.sin(angle) * radius)
    ctx.stroke()
    ctx.restore()

    // blips (points + anneau d’onde)
    const now = performance.now()
    blips = blips.filter(b => now - b.birth < b.life)
    for (const b of blips) {
        const t = (now - b.birth) / b.life // 0..1
        const a = 1 - t // fade out
        const rpx = b.r * radius
        const x = Math.cos(b.theta) * rpx
        const y = Math.sin(b.theta) * rpx

        // avatar Steve (fallback: point si pas encore chargé)
        ctx.save()
        ctx.globalAlpha = Math.max(0, Math.min(1, a * 1.0))
        ctx.shadowColor = props.color
        ctx.shadowBlur = 8
        if (steveReady) {
            const s = 16 + b.size * 2.5 // taille du sprite
            // option pixel-art propre
            const prevSmoothing = ctx.imageSmoothingEnabled
            ctx.imageSmoothingEnabled = false
            ctx.drawImage(steveImg, x - s / 2, y - s / 2, s, s)
            ctx.imageSmoothingEnabled = prevSmoothing
        } else {
            // fallback temporaire: petit point
            ctx.fillStyle = props.color
            ctx.beginPath()
            ctx.arc(x, y, b.size + 2, 0, Math.PI * 2)
            ctx.fill()
        }
        ctx.restore()

        // anneau qui s’étend
        ctx.save()
        const ringR = b.size + t * 18
        ctx.globalAlpha = Math.max(0, 0.6 - t)
        ctx.strokeStyle = props.color
        ctx.lineWidth = 1
        ctx.beginPath()
        ctx.arc(x, y, ringR, 0, Math.PI * 2)
        ctx.stroke()
        ctx.restore()
    }

    // masque bord (cercle)
    ctx.beginPath()
    ctx.arc(0, 0, radius, 0, Math.PI * 2)
    ctx.clip()

    ctx.restore()

    rafId = requestAnimationFrame(draw)
}

function resize() {
    const dpr = window.devicePixelRatio || 1
    const rect = containerEl.value.getBoundingClientRect()
    const size = Math.min(rect.width, rect.height)
    canvasEl.value.width = Math.floor(size * dpr)
    canvasEl.value.height = Math.floor(size * dpr)
    // on laisse le canvas remplir via CSS (w-full h-full)
    if (ctx) ctx.setTransform(1, 0, 0, 1, 0, 0) // reset
}

onMounted(() => {
    ctx = canvasEl.value.getContext('2d', { alpha: true })
    resize()
    // observer taille pour rester sharp
    ro = new ResizeObserver(() => resize())
    ro.observe(containerEl.value)

    // charge la tête de Steve
    steveImg = new Image()
    steveImg.crossOrigin = 'anonymous'
    steveImg.src = "https://minotar.net/helm/Steve/48.png"
    steveImg.onload = () => { steveReady = true }

    // lancer anim + blips
    rafId = requestAnimationFrame(draw)
    scheduleNextBlip()
})

onBeforeUnmount(() => {
    cancelAnimationFrame(rafId)
    clearTimeout(nextBlipTimer)
    if (ro) ro.disconnect()
})
</script>

<style scoped>
/* optionnel : scanlines très subtiles (rajoute du grain) */
</style>
