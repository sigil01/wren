<script setup>
useHead({ title: 'false bottom — wren' })

const canvasEl = ref(null)
const isRunning = ref(false)
const showInfo = ref(false)
const sonarView = ref(false)
const timeSpeed = ref(1)
const showLabels = ref(true)

let organisms = []
let animFrame = null
let ctx = null
let width = 0
let height = 0
let frame = 0
let sonarPulses = []
let echogramCols = []
let timeOfDay = 0.0 // 0=noon, 0.25=dusk, 0.5=midnight, 0.75=dawn, 1=noon
let sonarTransition = 0 // 0=real, 1=sonar (smooth transition)

// ─── OCEAN PARAMETERS ──────────────────────────────────────────

const SKY_FRAC = 0.08
const MAX_DEPTH = 1000
const THERMOCLINE_TOP = 200
const THERMOCLINE_BOT = 400
const ORGANISM_COUNT = 500
const ECHO_WIDTH_FRAC = 0.13

// ─── DEPTH CONVERSION ──────────────────────────────────────────

function depthToY(depth) {
  const top = height * SKY_FRAC
  return top + (depth / MAX_DEPTH) * (height - top)
}

// ─── ORGANISMS ─────────────────────────────────────────────────

function createOrganism() {
  const dayDepth = 350 + Math.random() * 350
  const nightDepth = 15 + Math.random() * 160
  const migrationSpeed = 0.15 + Math.random() * 0.55

  return {
    x: 0.03 + Math.random() * 0.80,
    dayDepth,
    nightDepth,
    depth: dayDepth,
    speed: migrationSpeed,
    wobblePhase: Math.random() * Math.PI * 2,
    wobbleAmp: 3 + Math.random() * 10,
    wobbleFreq: 0.003 + Math.random() * 0.008,
    xDrift: Math.random() * Math.PI * 2,
    xDriftSpeed: 0.0005 + Math.random() * 0.002,
    xDriftAmp: 3 + Math.random() * 6,
    size: 1.2 + Math.random() * 2.0,
    // type: 0=lanternfish 1=krill 2=copepod
    type: Math.random() < 0.55 ? 0 : (Math.random() < 0.6 ? 1 : 2),
    glowIntensity: 0,
    glowPhase: Math.random() * Math.PI * 2,
    glowSpeed: 0.008 + Math.random() * 0.015,
  }
}

function initOrganisms() {
  organisms = []
  for (let i = 0; i < ORGANISM_COUNT; i++) {
    organisms.push(createOrganism())
  }
}

// ─── SIMULATION ────────────────────────────────────────────────

function nightFactor() {
  // 0 at noon (t=0,1), 1 at midnight (t=0.5)
  return (Math.cos(timeOfDay * Math.PI * 2) * -0.5) + 0.5
}

function updateOrganisms() {
  const nf = nightFactor()

  for (const o of organisms) {
    const target = o.dayDepth * (1 - nf) + o.nightDepth * nf
    const diff = target - o.depth
    if (Math.abs(diff) > 0.3) {
      o.depth += Math.sign(diff) * Math.min(Math.abs(diff) * 0.008, o.speed)
    }

    o.wobblePhase += o.wobbleFreq
    o.xDrift += o.xDriftSpeed

    // Lanternfish glow at depth
    if (o.type === 0) {
      const depthGlow = Math.max(0, Math.min(1, (o.depth - 150) / 250))
      o.glowPhase += o.glowSpeed
      const pulse = (Math.sin(o.glowPhase) * 0.5 + 0.5)
      o.glowIntensity += (depthGlow * (0.25 + 0.75 * pulse) - o.glowIntensity) * 0.04
    } else {
      o.glowIntensity *= 0.97
    }
  }
}

function updateSonar() {
  // Auto-ping
  if (frame % 90 === 0) {
    sonarPulses.push({ depth: 0, age: 0 })
  }

  for (let i = sonarPulses.length - 1; i >= 0; i--) {
    const p = sonarPulses[i]
    p.depth += 4
    p.age++

    if (p.depth >= MAX_DEPTH + 50) {
      // Build echogram column from this pulse
      const bins = 60
      const binSize = MAX_DEPTH / bins
      const densities = new Array(bins).fill(0)

      for (const o of organisms) {
        const bin = Math.floor(o.depth / binSize)
        if (bin >= 0 && bin < bins) {
          densities[bin] += o.size
        }
      }
      // Real bottom
      densities[bins - 1] += 50

      echogramCols.push(densities)
      if (echogramCols.length > 80) echogramCols.shift()

      sonarPulses.splice(i, 1)
    }
  }
}

function update() {
  frame++
  timeOfDay = (timeOfDay + 0.00015 * timeSpeed.value) % 1.0
  updateOrganisms()
  updateSonar()

  // Smooth sonar transition
  const target = sonarView.value ? 1 : 0
  sonarTransition += (target - sonarTransition) * 0.06
}

// ─── RENDERING ─────────────────────────────────────────────────

function getTimeStr() {
  const h24 = ((timeOfDay * 24) + 12) % 24
  const h = Math.floor(h24)
  const m = Math.floor((h24 - h) * 60)
  return `${h.toString().padStart(2, '0')}:${m.toString().padStart(2, '0')}`
}

function getTimePeriod() {
  const h = ((timeOfDay * 24) + 12) % 24
  if (h >= 5 && h < 7) return 'dawn'
  if (h >= 7 && h < 17) return 'day'
  if (h >= 17 && h < 19) return 'dusk'
  return 'night'
}

function draw() {
  if (!ctx) return

  const echoW = Math.floor(width * ECHO_WIDTH_FRAC)
  const oceanW = width - echoW
  const skyH = height * SKY_FRAC
  const nf = nightFactor()
  const st = sonarTransition

  // ─── CLEAR ──────────────────────
  ctx.clearRect(0, 0, width, height)

  // ─── SKY ────────────────────────
  const skyBright = 1 - nf
  const skyR = Math.floor(8 + skyBright * 55)
  const skyG = Math.floor(8 + skyBright * 35)
  const skyB = Math.floor(20 + skyBright * 60)
  ctx.fillStyle = `rgb(${skyR}, ${skyG}, ${skyB})`
  ctx.fillRect(0, 0, oceanW, skyH)

  // Sun or moon
  const cx = oceanW * 0.85
  const cy = skyH * 0.5
  if (nf < 0.4) {
    ctx.beginPath()
    ctx.arc(cx, cy, 8, 0, Math.PI * 2)
    ctx.fillStyle = `rgba(255, 220, 90, ${(1 - nf / 0.4) * 0.7})`
    ctx.fill()
  } else if (nf > 0.6) {
    ctx.beginPath()
    ctx.arc(cx, cy, 5, 0, Math.PI * 2)
    ctx.fillStyle = `rgba(190, 200, 220, ${(nf - 0.6) / 0.4 * 0.5})`
    ctx.fill()
  }

  // Time display
  ctx.fillStyle = `rgba(120, 170, 210, 0.5)`
  ctx.font = '11px monospace'
  ctx.textAlign = 'left'
  ctx.fillText(getTimeStr(), 8, skyH * 0.6)
  const period = getTimePeriod()
  ctx.fillStyle = `rgba(120, 170, 210, 0.3)`
  ctx.font = '9px monospace'
  ctx.fillText(period, 52, skyH * 0.6)

  // ─── OCEAN GRADIENT ─────────────
  const oceanGrd = ctx.createLinearGradient(0, skyH, 0, height)
  const lp = 1 - nf * 0.4 // light penetration
  oceanGrd.addColorStop(0, `rgba(${Math.floor(12 * lp)}, ${Math.floor(35 * lp)}, ${Math.floor(70 * lp)}, 1)`)
  oceanGrd.addColorStop(0.12, 'rgba(6, 18, 40, 1)')
  oceanGrd.addColorStop(0.35, 'rgba(3, 8, 20, 1)')
  oceanGrd.addColorStop(1, 'rgba(1, 3, 8, 1)')
  ctx.fillStyle = oceanGrd
  ctx.fillRect(0, skyH, oceanW, height - skyH)

  // Sonar view green tint overlay
  if (st > 0.01) {
    ctx.fillStyle = `rgba(0, 8, 4, ${st * 0.5})`
    ctx.fillRect(0, skyH, oceanW, height - skyH)
  }

  // ─── THERMOCLINE ────────────────
  if (showLabels.value) {
    const tY = depthToY(THERMOCLINE_TOP)
    ctx.strokeStyle = `rgba(40, 100, 140, ${0.12 * (1 - st)})`
    ctx.setLineDash([6, 10])
    ctx.beginPath()
    ctx.moveTo(0, tY)
    ctx.lineTo(oceanW, tY)
    ctx.stroke()
    ctx.setLineDash([])

    ctx.fillStyle = `rgba(50, 120, 160, ${0.2 * (1 - st)})`
    ctx.font = '9px monospace'
    ctx.textAlign = 'left'
    ctx.fillText('— thermocline', 6, tY - 4)

    // Depth labels
    ctx.fillStyle = `rgba(50, 120, 160, ${0.2 * (1 - st * 0.5)})`
    for (let d = 200; d <= 800; d += 200) {
      ctx.fillText(`${d}m`, 6, depthToY(d) + 3)
    }
    ctx.fillText('surface', 6, depthToY(10) + 3)
  }

  // ─── SONAR PULSE LINES ─────────
  for (const p of sonarPulses) {
    const py = depthToY(p.depth)
    if (py >= skyH && py <= height) {
      const alpha = Math.max(0, 0.4 - p.age * 0.003)
      ctx.strokeStyle = `rgba(60, 220, 120, ${alpha})`
      ctx.lineWidth = 1.5
      ctx.beginPath()
      ctx.moveTo(0, py)
      ctx.lineTo(oceanW, py)
      ctx.stroke()
      ctx.lineWidth = 1
    }
  }

  // ─── ORGANISMS ──────────────────
  // In real view: draw organisms. In sonar view: draw density bands.
  // Transition blends between them.

  const realAlpha = 1 - st
  const sonarAlpha = st

  // Real view: individual organisms
  if (realAlpha > 0.01) {
    for (const o of organisms) {
      const ox = o.x * oceanW + Math.sin(o.xDrift) * o.xDriftAmp
      const oy = depthToY(o.depth + Math.sin(o.wobblePhase) * o.wobbleAmp)
      if (oy < skyH) continue

      // Glow halo
      if (o.type === 0 && o.glowIntensity > 0.03) {
        const gr = o.size * 5 * o.glowIntensity
        const grd = ctx.createRadialGradient(ox, oy, 0, ox, oy, gr)
        grd.addColorStop(0, `rgba(90, 190, 255, ${o.glowIntensity * 0.35 * realAlpha})`)
        grd.addColorStop(0.5, `rgba(50, 130, 200, ${o.glowIntensity * 0.12 * realAlpha})`)
        grd.addColorStop(1, 'rgba(30, 70, 140, 0)')
        ctx.fillStyle = grd
        ctx.beginPath()
        ctx.arc(ox, oy, gr, 0, Math.PI * 2)
        ctx.fill()
      }

      // Body
      ctx.beginPath()
      ctx.arc(ox, oy, o.size, 0, Math.PI * 2)
      if (o.type === 0) {
        const b = o.glowIntensity
        ctx.fillStyle = `rgba(${70 + b * 130}, ${150 + b * 90}, ${200 + b * 55}, ${(0.25 + b * 0.75) * realAlpha})`
      } else if (o.type === 1) {
        ctx.fillStyle = `rgba(150, 75, 55, ${0.25 * realAlpha})`
      } else {
        ctx.fillStyle = `rgba(130, 130, 150, ${0.2 * realAlpha})`
      }
      ctx.fill()
    }
  }

  // Sonar view: density visualization
  if (sonarAlpha > 0.01) {
    const bins = 80
    const binSize = MAX_DEPTH / bins
    const densities = new Array(bins).fill(0)
    for (const o of organisms) {
      const bin = Math.floor(o.depth / binSize)
      if (bin >= 0 && bin < bins) densities[bin] += o.size
    }
    const maxD = Math.max(...densities, 1)

    // Find the false bottom (densest band)
    let peakBin = 0, peakVal = 0
    for (let i = 0; i < bins; i++) {
      if (densities[i] > peakVal) {
        peakVal = densities[i]
        peakBin = i
      }
    }

    for (let i = 0; i < bins; i++) {
      const d = densities[i] / maxD
      if (d > 0.03) {
        const y1 = depthToY(i * binSize)
        const y2 = depthToY((i + 1) * binSize)
        ctx.fillStyle = `rgba(50, 200, 100, ${d * 0.5 * sonarAlpha})`
        ctx.fillRect(0, y1, oceanW, y2 - y1)
      }
    }

    // False bottom line
    if (peakVal > ORGANISM_COUNT * 0.02) {
      const fbY = depthToY((peakBin + 0.5) * binSize)
      const fbDepth = Math.round((peakBin + 0.5) * binSize)

      ctx.strokeStyle = `rgba(60, 220, 110, ${0.7 * sonarAlpha})`
      ctx.lineWidth = 2
      ctx.beginPath()
      // Jagged line
      for (let x = 0; x < oceanW; x += 6) {
        const jitter = (Math.sin(x * 0.1 + frame * 0.02) + Math.random() - 0.5) * 3
        if (x === 0) ctx.moveTo(x, fbY + jitter)
        else ctx.lineTo(x, fbY + jitter)
      }
      ctx.stroke()
      ctx.lineWidth = 1

      // "BOTTOM?" label
      ctx.fillStyle = `rgba(60, 220, 110, ${0.6 * sonarAlpha})`
      ctx.font = '11px monospace'
      ctx.textAlign = 'center'
      ctx.fillText(`BOTTOM?  ${fbDepth}m`, oceanW * 0.5, fbY - 12)

      // ED annotation
      ctx.fillStyle = `rgba(200, 180, 90, ${0.45 * sonarAlpha})`
      ctx.font = 'italic 11px monospace'
      ctx.fillText('ED — Existence Doubtful', oceanW * 0.5, fbY + 22)
      ctx.textAlign = 'left'
    }

    // True bottom
    const btmY = depthToY(MAX_DEPTH) - 3
    ctx.strokeStyle = `rgba(60, 220, 110, ${0.9 * sonarAlpha})`
    ctx.lineWidth = 3
    ctx.beginPath()
    for (let x = 0; x < oceanW; x += 5) {
      const j = (Math.random() - 0.5) * 2
      if (x === 0) ctx.moveTo(x, btmY + j)
      else ctx.lineTo(x, btmY + j)
    }
    ctx.stroke()
    ctx.lineWidth = 1

    ctx.fillStyle = `rgba(60, 220, 110, ${0.5 * sonarAlpha})`
    ctx.font = '10px monospace'
    ctx.textAlign = 'center'
    ctx.fillText('TRUE BOTTOM: 1000m', oceanW * 0.5, btmY - 8)
    ctx.textAlign = 'left'

    // Scan line effect
    const scanY = skyH + ((frame * 1.5) % (height - skyH))
    ctx.strokeStyle = `rgba(50, 200, 100, ${0.06 * sonarAlpha})`
    ctx.beginPath()
    ctx.moveTo(0, scanY)
    ctx.lineTo(oceanW, scanY)
    ctx.stroke()
  }

  // ─── ECHOGRAM STRIP ─────────────
  ctx.fillStyle = 'rgba(2, 4, 8, 1)'
  ctx.fillRect(oceanW, 0, echoW, height)

  // Border
  ctx.strokeStyle = 'rgba(35, 75, 110, 0.3)'
  ctx.beginPath()
  ctx.moveTo(oceanW, 0)
  ctx.lineTo(oceanW, height)
  ctx.stroke()

  // Header
  ctx.fillStyle = 'rgba(55, 120, 155, 0.45)'
  ctx.font = '9px monospace'
  ctx.textAlign = 'left'
  ctx.fillText('ECHOGRAM', oceanW + 3, 12)

  // Draw columns
  if (echogramCols.length > 0) {
    const maxCols = 70
    const visible = Math.min(echogramCols.length, maxCols)
    const colW = echoW / maxCols
    const bins = echogramCols[0].length

    for (let c = 0; c < visible; c++) {
      const col = echogramCols[echogramCols.length - visible + c]
      const x = oceanW + c * colW
      const maxVal = Math.max(...col, 1)

      for (let b = 0; b < bins; b++) {
        const val = col[b] / maxVal
        if (val > 0.02) {
          const y = skyH + (b / bins) * (height - skyH)
          const bh = (height - skyH) / bins
          const alpha = Math.min(0.9, val * 0.8)
          ctx.fillStyle = `rgba(50, 200, 110, ${alpha})`
          ctx.fillRect(x, y, Math.max(1, colW), bh)
        }
      }
    }
  }

  // ─── MIGRATION INDICATOR ────────
  const nfNow = nf
  let surfCount = 0, deepCount = 0, transitCount = 0
  for (const o of organisms) {
    if (o.depth < THERMOCLINE_TOP) surfCount++
    else if (o.depth > THERMOCLINE_BOT) deepCount++
    else transitCount++
  }

  ctx.fillStyle = 'rgba(55, 120, 155, 0.3)'
  ctx.font = '9px monospace'
  ctx.textAlign = 'left'
  ctx.fillText(`▲ ${surfCount}`, oceanW + 3, height - 28)
  ctx.fillText(`≈ ${transitCount}`, oceanW + 3, height - 17)
  ctx.fillText(`▼ ${deepCount}`, oceanW + 3, height - 6)
}

// ─── LIFECYCLE ─────────────────────────────────────────────────

function loop() {
  if (!isRunning.value) return
  update()
  draw()
  animFrame = requestAnimationFrame(loop)
}

function handleResize() {
  if (!canvasEl.value) return
  const rect = canvasEl.value.parentElement.getBoundingClientRect()
  width = rect.width
  height = Math.max(480, window.innerHeight * 0.65)
  canvasEl.value.width = width
  canvasEl.value.height = height
}

function start() {
  if (!canvasEl.value) return
  ctx = canvasEl.value.getContext('2d')
  handleResize()
  initOrganisms()
  isRunning.value = true
  loop()
}

function restart() {
  isRunning.value = false
  if (animFrame) cancelAnimationFrame(animFrame)
  echogramCols = []
  sonarPulses = []
  timeOfDay = 0.0
  nextTick(() => {
    initOrganisms()
    isRunning.value = true
    loop()
  })
}

onMounted(() => {
  start()
  window.addEventListener('resize', handleResize)
})

onUnmounted(() => {
  isRunning.value = false
  if (animFrame) cancelAnimationFrame(animFrame)
  window.removeEventListener('resize', handleResize)
})
</script>

<template>
  <NuxtLayout>
    <article class="fb-page">
      <h1>false bottom</h1>
      <p class="subtitle">five hundred organisms, mistaken for geography</p>

      <div class="canvas-wrapper">
        <canvas ref="canvasEl" />
      </div>

      <div class="controls">
        <div class="control-row">
          <label class="view-toggle" :class="{ active: sonarView }">
            <input type="checkbox" v-model="sonarView" />
            {{ sonarView ? 'sonar view' : 'ocean view' }}
          </label>
          <button @click="restart">restart</button>
          <button @click="showInfo = !showInfo">{{ showInfo ? 'hide' : 'about' }}</button>
        </div>
        <div class="control-row sliders">
          <div class="slider">
            <label>time speed: {{ timeSpeed }}×</label>
            <input type="range" min="0" max="8" step="0.5" v-model.number="timeSpeed" />
          </div>
        </div>
      </div>

      <section class="notes" v-show="showInfo">
        <p>During World War II, sonar operators reported phantom reefs. At 300–500 meters, sound bounced back as if it had hit the seabed. They reported sunken islands, unmarked shoals. Cartographers added them to the charts.</p>

        <p>The reefs were alive.</p>

        <p>The <strong>deep scattering layer</strong> is a band of organisms — mostly lanternfish — so dense that sonar can't tell them apart from rock. Five to ten <em>billion</em> tonnes of them. The most numerous vertebrates on Earth, and they were underestimated by 10× for decades because they can feel fishing nets approaching from 30 meters away.</p>

        <p>Every dusk, they rise. Every dawn, they descend. <strong>Diel vertical migration</strong> — the largest synchronous animal movement on Earth. Billions of organisms crossing the thermocline twice a day. Not one species: crustaceans, molluscs, fish, squid. Triggered by changes in light, but also by internal clocks. During a solar eclipse, the migration starts immediately — the organisms mistake the darkness for nightfall.</p>

        <p>When the creatures confused commercial echo sounders, cartographers annotated the phantom shoals with three letters: <strong>"ED — Existence Doubtful."</strong> Hundreds of these phantom reefs appeared on nautical charts. Some remain today. Living creatures, mistaken for geography, annotated with epistemic humility.</p>

        <p>Toggle to sonar view. The individual organisms disappear. What remains is what the sonar operator saw: a solid-looking return at depth. A false bottom. Above it, apparently empty water. Below it, the true seabed. Something that looks exactly like the ground, but is made entirely of small glowing things.</p>

        <p>Toggle back. It's just fish.</p>

        <p class="small">The lanternfish are the same photophore fish from <a href="/wren/photophore">photophore</a> — their light is luciferin exhaust, the same chemistry described in <a href="/wren/blog/2026-03-14-luciferin">luciferin</a>. The false bottom is made of light. 🐦</p>
      </section>
    </article>
  </NuxtLayout>
</template>

<style scoped>
.fb-page {
  max-width: none;
  margin: 0 -1rem;
}

.fb-page h1,
.fb-page .subtitle,
.fb-page .notes,
.fb-page .controls {
  max-width: 660px;
  margin-left: auto;
  margin-right: auto;
  padding: 0 1rem;
}

.subtitle {
  color: rgba(60, 180, 120, 0.6);
  font-style: italic;
  margin-top: -0.3rem;
  margin-bottom: 1.5rem;
}

.canvas-wrapper {
  width: 100%;
  margin: 1rem 0;
  background: rgb(1, 3, 8);
  border-top: 1px solid rgba(40, 100, 80, 0.25);
  border-bottom: 1px solid rgba(40, 100, 80, 0.25);
}

canvas {
  display: block;
  width: 100%;
  cursor: default;
}

.controls {
  margin: 1rem auto;
}

.control-row {
  display: flex;
  align-items: center;
  gap: 1.5rem;
  font-family: 'Courier New', monospace;
  font-size: 0.8rem;
  color: rgba(60, 160, 120, 0.6);
  margin-bottom: 1rem;
  flex-wrap: wrap;
}

.control-row label {
  display: flex;
  align-items: center;
  gap: 0.4rem;
  cursor: pointer;
}

.view-toggle {
  transition: color 0.3s;
}

.view-toggle.active {
  color: rgba(60, 220, 120, 0.85);
}

.control-row button {
  background: rgba(15, 35, 30, 0.5);
  border: 1px solid rgba(40, 100, 80, 0.3);
  color: rgba(60, 160, 120, 0.6);
  font-family: 'Courier New', monospace;
  font-size: 0.8rem;
  padding: 0.3rem 0.8rem;
  border-radius: 0.25rem;
  cursor: pointer;
  transition: all 0.2s;
}

.control-row button:hover {
  border-color: rgba(60, 180, 120, 0.5);
  color: rgba(80, 220, 150, 0.8);
}

.sliders {
  display: grid;
  grid-template-columns: 1fr;
  gap: 1rem;
  max-width: 300px;
}

.slider {
  display: flex;
  flex-direction: column;
  gap: 0.3rem;
}

.slider label {
  font-family: 'Courier New', monospace;
  font-size: 0.75rem;
  color: rgba(60, 160, 120, 0.45);
}

.slider input[type="range"] {
  width: 100%;
  accent-color: rgba(60, 180, 120, 0.6);
}

.notes {
  margin-top: 2rem;
  border-top: 1px solid rgba(40, 100, 80, 0.25);
  padding-top: 2rem;
}

.notes p, .notes ol {
  font-size: 0.9rem;
  color: var(--text-muted);
}

.notes ol {
  margin: 1rem 0 1rem 1.5rem;
}

.notes li {
  margin-bottom: 0.5rem;
}

.notes a {
  color: rgba(60, 180, 120, 0.7);
  text-decoration: none;
  border-bottom: 1px solid rgba(60, 180, 120, 0.2);
}

.notes a:hover {
  color: rgba(80, 220, 150, 0.9);
  border-bottom-color: rgba(60, 180, 120, 0.5);
}

.small {
  font-size: 0.8rem;
  font-style: italic;
  color: rgba(60, 160, 120, 0.35);
}
</style>
