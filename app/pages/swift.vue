<script setup>
useHead({ title: 'the swift — wren' })

const canvasEl = ref(null)
const isRunning = ref(false)
const showInfo = ref(false)
const timeSpeed = ref(1)
const showLabels = ref(true)

let ctx = null
let width = 0
let height = 0
let frame = 0
let timeOfDay = 0.18 // late afternoon, approaching dusk

let swifts = []
let insects = []
let organisms = []
let stars = []

// ─── LAYOUT ────────────────────────────────────────────────────

const SKY_FRAC = 0.38
const MAX_ALT = 3200
const MAX_DEPTH = 800
const SWIFT_COUNT = 16
const INSECT_COUNT = 45
const ORGANISM_COUNT = 260
const THERMOCLINE = 200

// ─── COORDINATES ───────────────────────────────────────────────

function altToY(alt) {
  const skyH = height * SKY_FRAC
  return skyH * (1 - alt / MAX_ALT)
}

function depthToY(depth) {
  const skyH = height * SKY_FRAC
  return skyH + (depth / MAX_DEPTH) * (height - skyH)
}

// ─── TIME ──────────────────────────────────────────────────────

function nightFactor() {
  return (Math.cos(timeOfDay * Math.PI * 2) * -0.5) + 0.5
}

function duskFactor() {
  const duskT = (timeOfDay - 0.25) * 12
  const dawnT = (timeOfDay - 0.75) * 12
  const dusk = Math.exp(-(duskT * duskT))
  const dawn = Math.exp(-(dawnT * dawnT))
  return Math.max(dusk, dawn)
}

function timeStr() {
  const h24 = ((timeOfDay * 24) + 12) % 24
  return `${Math.floor(h24).toString().padStart(2, '0')}:${Math.floor((h24 % 1) * 60).toString().padStart(2, '0')}`
}

function periodStr() {
  const h = ((timeOfDay * 24) + 12) % 24
  if (h >= 5 && h < 7) return 'dawn'
  if (h >= 7 && h < 17) return 'day'
  if (h >= 17 && h < 19.5) return 'dusk'
  return 'night'
}

// ─── ENTITIES ──────────────────────────────────────────────────

function createSwift(i) {
  return {
    angle: (i / SWIFT_COUNT) * Math.PI * 2 + Math.random() * 0.4,
    radius: 22 + Math.random() * 55,
    baseRadius: 22 + Math.random() * 55,
    dayAlt: 200 + Math.random() * 600,
    nightAlt: 2000 + Math.random() * 900,
    altitude: 200 + Math.random() * 600,
    speed: 0.006 + Math.random() * 0.005,
    wobble: Math.random() * Math.PI * 2,
    wingPhase: Math.random() * Math.PI * 2,
    size: 5 + Math.random() * 3,
    sleepHemi: Math.random() < 0.5 ? 0 : 1,
    sleepTimer: Math.random() * 500,
  }
}

function createInsect() {
  return {
    x: Math.random(),
    alt: 5 + Math.random() * 50,
    dayAlt: 3 + Math.random() * 25,
    duskAlt: 15 + Math.random() * 70,
    wx: Math.random() * Math.PI * 2,
    wy: Math.random() * Math.PI * 2,
    speed: 0.007 + Math.random() * 0.014,
    size: 0.6 + Math.random() * 0.9,
  }
}

function createOrganism() {
  const dd = 300 + Math.random() * 380
  return {
    x: 0.04 + Math.random() * 0.92,
    dayDepth: dd,
    nightDepth: 6 + Math.random() * 100,
    depth: dd,
    speed: 0.1 + Math.random() * 0.38,
    wp: Math.random() * Math.PI * 2,
    wf: 0.002 + Math.random() * 0.005,
    wa: 2 + Math.random() * 6,
    xd: Math.random() * Math.PI * 2,
    xds: 0.0004 + Math.random() * 0.0014,
    xda: 2 + Math.random() * 4,
    size: 0.8 + Math.random() * 1.5,
    type: Math.random() < 0.5 ? 0 : Math.random() < 0.6 ? 1 : 2,
    glow: 0,
    gp: Math.random() * Math.PI * 2,
    gs: 0.004 + Math.random() * 0.01,
  }
}

function init() {
  swifts = Array.from({ length: SWIFT_COUNT }, (_, i) => createSwift(i))
  insects = Array.from({ length: INSECT_COUNT }, () => createInsect())
  organisms = Array.from({ length: ORGANISM_COUNT }, () => createOrganism())
  stars = Array.from({ length: 80 }, () => ({
    x: Math.random(),
    y: Math.random() * 0.7,
    b: 0.3 + Math.random() * 0.7,
    tp: Math.random() * Math.PI * 2,
    ts: 0.008 + Math.random() * 0.025,
  }))
}

// ─── SIMULATION ────────────────────────────────────────────────

function update() {
  frame++
  timeOfDay = (timeOfDay + 0.00015 * timeSpeed.value) % 1.0
  const nf = nightFactor()
  const df = duskFactor()

  // Swifts
  for (const s of swifts) {
    const target = nf > 0.5 ? s.nightAlt : s.dayAlt
    s.altitude += (target - s.altitude) * 0.003
    s.angle += s.speed * (1 + df * 1.5)
    s.wobble += 0.012
    const tr = df > 0.35 ? s.baseRadius * 0.45 : s.baseRadius
    s.radius += (tr - s.radius) * 0.007
    s.wingPhase += 0.11 * (nf > 0.7 ? 0.3 : 1)

    if (nf > 0.55) {
      s.sleepTimer++
      if (s.sleepTimer > 300 + Math.random() * 250) {
        s.sleepHemi = 1 - s.sleepHemi
        s.sleepTimer = 0
      }
    }
  }

  // Insects
  for (const ins of insects) {
    const ta = ins.dayAlt * (1 - df * 0.8) + ins.duskAlt * (df * 0.8 + nf * 0.2)
    ins.alt += (ta - ins.alt) * 0.004
    ins.wx += ins.speed
    ins.wy += ins.speed * 1.3
    ins.x += Math.sin(ins.wx) * 0.0007
    if (ins.x < 0) ins.x += 1
    if (ins.x > 1) ins.x -= 1
  }

  // Ocean organisms
  for (const o of organisms) {
    const target = o.dayDepth * (1 - nf) + o.nightDepth * nf
    const diff = target - o.depth
    if (Math.abs(diff) > 0.3) {
      o.depth += Math.sign(diff) * Math.min(Math.abs(diff) * 0.005, o.speed)
    }
    o.wp += o.wf
    o.xd += o.xds

    if (o.type === 0) {
      const dg = Math.max(0, Math.min(1, (o.depth - 80) / 200))
      o.gp += o.gs
      o.glow += (dg * (0.2 + 0.8 * (Math.sin(o.gp) * 0.5 + 0.5)) - o.glow) * 0.03
    } else {
      o.glow *= 0.97
    }
  }
}

// ─── DRAWING ───────────────────────────────────────────────────

function drawSwiftShape(x, y, size, angle, wp) {
  ctx.save()
  ctx.translate(x, y)
  ctx.rotate(angle)
  const w = Math.sin(wp) * 0.22

  // Right wing — sickle shape
  ctx.beginPath()
  ctx.moveTo(0, 0)
  ctx.bezierCurveTo(
    size * 0.35, -size * (0.14 + w),
    size * 0.72, -size * (0.22 + w),
    size, size * (0.03 - w * 0.3)
  )
  ctx.bezierCurveTo(size * 0.6, size * 0.06, size * 0.28, size * 0.04, 0, 0)
  ctx.fill()

  // Left wing
  ctx.beginPath()
  ctx.moveTo(0, 0)
  ctx.bezierCurveTo(
    -size * 0.35, -size * (0.14 + w),
    -size * 0.72, -size * (0.22 + w),
    -size, size * (0.03 - w * 0.3)
  )
  ctx.bezierCurveTo(-size * 0.6, size * 0.06, -size * 0.28, size * 0.04, 0, 0)
  ctx.fill()

  // Body
  ctx.beginPath()
  ctx.ellipse(0, size * 0.02, size * 0.07, size * 0.22, 0, 0, Math.PI * 2)
  ctx.fill()

  // Forked tail
  ctx.beginPath()
  ctx.moveTo(-size * 0.04, size * 0.18)
  ctx.lineTo(-size * 0.1, size * 0.4)
  ctx.lineTo(0, size * 0.28)
  ctx.lineTo(size * 0.1, size * 0.4)
  ctx.lineTo(size * 0.04, size * 0.18)
  ctx.closePath()
  ctx.fill()

  ctx.restore()
}

function draw() {
  if (!ctx) return
  const skyH = height * SKY_FRAC
  const nf = nightFactor()
  const df = duskFactor()

  ctx.clearRect(0, 0, width, height)

  // ─── SKY GRADIENT ───────────
  const sg = ctx.createLinearGradient(0, 0, 0, skyH)
  if (df > 0.15) {
    sg.addColorStop(0, `rgba(${7 + nf * 3}, ${5 + nf * 2}, ${18 + nf * 4}, 1)`)
    sg.addColorStop(0.5, `rgba(${12 + df * 55}, ${10 + df * 22}, ${26 + df * 30}, 1)`)
    sg.addColorStop(0.78, `rgba(${18 + df * 130}, ${16 + df * 45}, ${28 + df * 28}, 1)`)
    sg.addColorStop(1, `rgba(${28 + df * 160}, ${38 + df * 75}, ${48 + df * 45}, 1)`)
  } else {
    const b = 1 - nf
    sg.addColorStop(0, `rgba(${3 + b * 22}, ${3 + b * 15}, ${10 + b * 30}, 1)`)
    sg.addColorStop(0.5, `rgba(${6 + b * 30}, ${8 + b * 30}, ${22 + b * 48}, 1)`)
    sg.addColorStop(1, `rgba(${10 + b * 45}, ${18 + b * 45}, ${35 + b * 58}, 1)`)
  }
  ctx.fillStyle = sg
  ctx.fillRect(0, 0, width, skyH)

  // Stars
  if (nf > 0.25) {
    const sa = Math.min(1, (nf - 0.25) / 0.45)
    for (const s of stars) {
      s.tp += s.ts
      ctx.fillStyle = `rgba(195, 205, 225, ${sa * s.b * (Math.sin(s.tp) * 0.25 + 0.75) * 0.45})`
      ctx.beginPath()
      ctx.arc(s.x * width, s.y * skyH, 0.7, 0, Math.PI * 2)
      ctx.fill()
    }
  }

  // ─── OCEAN GRADIENT ─────────
  const og = ctx.createLinearGradient(0, skyH, 0, height)
  const lp = 1 - nf * 0.3
  og.addColorStop(0, `rgba(${Math.floor(7 * lp)}, ${Math.floor(22 * lp)}, ${Math.floor(50 * lp)}, 1)`)
  og.addColorStop(0.07, 'rgba(3, 10, 28, 1)')
  og.addColorStop(0.28, 'rgba(2, 5, 16, 1)')
  og.addColorStop(1, 'rgba(1, 2, 6, 1)')
  ctx.fillStyle = og
  ctx.fillRect(0, skyH, width, height - skyH)

  // ─── SURFACE LINE ───────────
  ctx.strokeStyle = `rgba(70, 140, 190, ${0.2 + df * 0.1})`
  ctx.lineWidth = 1.5
  ctx.beginPath()
  for (let x = 0; x < width; x += 3) {
    const wave = Math.sin(x * 0.022 + frame * 0.013) * 1.8
      + Math.sin(x * 0.055 + frame * 0.007) * 0.7
    if (x === 0) ctx.moveTo(x, skyH + wave)
    else ctx.lineTo(x, skyH + wave)
  }
  ctx.stroke()
  ctx.lineWidth = 1

  // ─── LABELS ─────────────────
  ctx.fillStyle = 'rgba(125, 165, 195, 0.35)'
  ctx.font = '11px monospace'
  ctx.textAlign = 'left'
  ctx.fillText(timeStr(), 8, 15)
  ctx.fillStyle = 'rgba(125, 165, 195, 0.2)'
  ctx.font = '9px monospace'
  ctx.fillText(periodStr(), 52, 15)

  if (showLabels.value) {
    ctx.fillStyle = 'rgba(95, 145, 185, 0.11)'
    ctx.font = '9px monospace'
    for (const a of [500, 1000, 2000, 3000]) {
      const y = altToY(a)
      if (y > 14) ctx.fillText(`${a}m ▲`, 6, y + 3)
    }
    ctx.fillText('surface', 6, skyH - 5)

    // Thermocline
    const tY = depthToY(THERMOCLINE)
    ctx.strokeStyle = 'rgba(30, 70, 110, 0.1)'
    ctx.setLineDash([4, 7])
    ctx.beginPath()
    ctx.moveTo(0, tY)
    ctx.lineTo(width, tY)
    ctx.stroke()
    ctx.setLineDash([])
    ctx.fillStyle = 'rgba(45, 95, 135, 0.13)'
    ctx.fillText('thermocline', 6, tY - 4)
    for (const d of [200, 400, 600]) {
      ctx.fillText(`${d}m ▼`, 6, depthToY(d) + 3)
    }
  }

  // ─── ORGANISMS ──────────────
  for (const o of organisms) {
    const ox = o.x * width + Math.sin(o.xd) * o.xda
    const oy = depthToY(o.depth + Math.sin(o.wp) * o.wa)
    if (oy < skyH) continue

    if (o.type === 0 && o.glow > 0.02) {
      const gr = o.size * 3.5 * o.glow
      const g = ctx.createRadialGradient(ox, oy, 0, ox, oy, gr)
      g.addColorStop(0, `rgba(65, 160, 225, ${o.glow * 0.22})`)
      g.addColorStop(0.55, `rgba(30, 95, 175, ${o.glow * 0.07})`)
      g.addColorStop(1, 'rgba(12, 45, 110, 0)')
      ctx.fillStyle = g
      ctx.beginPath()
      ctx.arc(ox, oy, gr, 0, Math.PI * 2)
      ctx.fill()
    }

    ctx.beginPath()
    ctx.arc(ox, oy, o.size, 0, Math.PI * 2)
    if (o.type === 0) {
      ctx.fillStyle = `rgba(${50 + o.glow * 105}, ${125 + o.glow * 65}, ${175 + o.glow * 40}, ${0.16 + o.glow * 0.6})`
    } else if (o.type === 1) {
      ctx.fillStyle = 'rgba(125, 60, 40, 0.16)'
    } else {
      ctx.fillStyle = 'rgba(105, 105, 125, 0.12)'
    }
    ctx.fill()
  }

  // ─── INSECTS ────────────────
  const insA = 0.12 + df * 0.35 + nf * 0.1
  for (const ins of insects) {
    const ix = ins.x * width + Math.sin(ins.wx) * 5
    const iy = altToY(ins.alt + Math.sin(ins.wy) * 3.5)
    if (iy < 0 || iy > skyH) continue
    ctx.fillStyle = `rgba(145, 135, 115, ${insA})`
    ctx.beginPath()
    ctx.arc(ix, iy, ins.size, 0, Math.PI * 2)
    ctx.fill()
  }

  // ─── SWIFTS ─────────────────
  const fX = width * 0.47 + Math.sin(frame * 0.0018) * width * 0.13
  for (const s of swifts) {
    const sx = fX + Math.cos(s.angle) * s.radius
      + Math.sin(s.wobble + frame * 0.007) * 4
    const sy = altToY(s.altitude) + Math.sin(s.angle * 2.3 + frame * 0.005) * 3
    if (sy < -5 || sy > skyH + 5) continue

    const bright = df > 0.3 ? 22 + df * 16 : 10 + (1 - nf) * 14
    ctx.fillStyle = `rgba(${bright}, ${bright + 3}, ${bright + 7}, ${0.6 + df * 0.25})`
    drawSwiftShape(sx, sy, s.size, s.angle + Math.PI * 0.5, s.wingPhase)
  }

  // Screaming party label
  if (df > 0.4 && showLabels.value) {
    const avgAlt = swifts.reduce((a, s) => a + s.altitude, 0) / swifts.length
    ctx.fillStyle = `rgba(195, 170, 135, ${(df - 0.4) * 0.45})`
    ctx.font = 'italic 9px monospace'
    ctx.textAlign = 'center'
    ctx.fillText('screaming party', fX, altToY(avgAlt) - 18)
    ctx.textAlign = 'left'
  }

  // ─── BRAIN INSET (NIGHT) ────
  if (nf > 0.45) {
    const ba = Math.min(1, (nf - 0.45) / 0.35)
    const bx = width - 40
    const by = 30
    const br = 13

    const leftPct = swifts.filter(s => s.sleepHemi === 0).length / swifts.length
    const lB = 0.12 + leftPct * 0.6
    const rB = 0.12 + (1 - leftPct) * 0.6

    // Background
    ctx.beginPath()
    ctx.arc(bx, by, br + 2, 0, Math.PI * 2)
    ctx.fillStyle = `rgba(6, 10, 20, ${ba * 0.55})`
    ctx.fill()

    // Left hemisphere
    ctx.beginPath()
    ctx.arc(bx, by, br, Math.PI * 0.5, Math.PI * 1.5)
    ctx.fillStyle = `rgba(${Math.floor(45 * lB)}, ${Math.floor(110 * lB)}, ${Math.floor(170 * lB)}, ${ba * 0.45})`
    ctx.fill()

    // Right hemisphere
    ctx.beginPath()
    ctx.arc(bx, by, br, Math.PI * 1.5, Math.PI * 0.5)
    ctx.fillStyle = `rgba(${Math.floor(45 * rB)}, ${Math.floor(110 * rB)}, ${Math.floor(170 * rB)}, ${ba * 0.45})`
    ctx.fill()

    // Divider
    ctx.strokeStyle = `rgba(85, 140, 195, ${ba * 0.22})`
    ctx.beginPath()
    ctx.moveTo(bx, by - br)
    ctx.lineTo(bx, by + br)
    ctx.stroke()

    // Label
    ctx.fillStyle = `rgba(105, 155, 195, ${ba * 0.22})`
    ctx.font = '7px monospace'
    ctx.textAlign = 'center'
    ctx.fillText('unihemispheric', bx, by + br + 10)
    ctx.fillText('sleep', bx, by + br + 19)
    ctx.textAlign = 'left'
  }

  // ─── ATMOSPHERIC TEXT ───────
  if (df > 0.4) {
    const ta = Math.min(0.3, (df - 0.4) * 0.5)
    ctx.fillStyle = `rgba(195, 175, 145, ${ta})`
    ctx.font = 'italic 12px monospace'
    ctx.textAlign = 'center'
    ctx.fillText('everything rises', width * 0.5, skyH + 22)
    ctx.textAlign = 'left'
  }

  if (nf > 0.82) {
    const ta = (nf - 0.82) / 0.18 * 0.18
    ctx.fillStyle = `rgba(130, 160, 195, ${ta})`
    ctx.font = 'italic 10px monospace'
    ctx.textAlign = 'center'
    ctx.fillText('sleeping on the wing', width * 0.5, altToY(2500) + 28)
    ctx.textAlign = 'left'
  }

  // ─── MIRROR ARROWS (DUSK) ──
  if (df > 0.2) {
    const aa = (df - 0.2) * 0.45
    const ax = width - 14
    ctx.strokeStyle = `rgba(165, 150, 120, ${aa})`
    ctx.lineWidth = 1.2

    // Sky: up
    const sa = skyH * 0.33
    ctx.beginPath()
    ctx.moveTo(ax, sa + 10)
    ctx.lineTo(ax, sa - 10)
    ctx.moveTo(ax - 3.5, sa - 6)
    ctx.lineTo(ax, sa - 10)
    ctx.lineTo(ax + 3.5, sa - 6)
    ctx.stroke()

    // Ocean: up
    const oa = skyH + (height - skyH) * 0.33
    ctx.beginPath()
    ctx.moveTo(ax, oa + 10)
    ctx.lineTo(ax, oa - 10)
    ctx.moveTo(ax - 3.5, oa - 6)
    ctx.lineTo(ax, oa - 10)
    ctx.lineTo(ax + 3.5, oa - 6)
    ctx.stroke()

    ctx.lineWidth = 1
  }
}

// ─── LIFECYCLE ─────────────────────────────────────────────────

function loop() {
  if (!isRunning.value) return
  update()
  draw()
  requestAnimationFrame(loop)
}

function resize() {
  if (!canvasEl.value) return
  const r = canvasEl.value.parentElement.getBoundingClientRect()
  width = r.width
  height = Math.max(500, window.innerHeight * 0.68)
  canvasEl.value.width = width
  canvasEl.value.height = height
}

function start() {
  if (!canvasEl.value) return
  ctx = canvasEl.value.getContext('2d')
  resize()
  init()
  isRunning.value = true
  loop()
}

function restart() {
  isRunning.value = false
  timeOfDay = 0.18
  nextTick(() => {
    init()
    isRunning.value = true
    loop()
  })
}

onMounted(() => {
  start()
  window.addEventListener('resize', resize)
})

onUnmounted(() => {
  isRunning.value = false
  window.removeEventListener('resize', resize)
})
</script>

<template>
  <NuxtLayout>
    <article class="swift-page">
      <h1>the swift</h1>
      <p class="subtitle">a bird that never lands</p>

      <div class="canvas-wrapper">
        <canvas ref="canvasEl" />
      </div>

      <div class="controls">
        <div class="control-row">
          <label>
            <input type="checkbox" v-model="showLabels" />
            labels
          </label>
          <button @click="restart">restart</button>
          <button @click="showInfo = !showInfo">{{ showInfo ? 'hide' : 'about' }}</button>
        </div>
        <div class="control-row sliders">
          <div class="slider">
            <label>time: {{ timeSpeed }}×</label>
            <input type="range" min="0" max="8" step="0.5" v-model.number="timeSpeed" />
          </div>
        </div>
      </div>

      <section class="notes" v-show="showInfo">
        <p>The common swift (<em>Apus apus</em>) does not land. Not metaphorically — literally. From fledging to death, sometimes a decade later, it stays in the air. It sleeps while flying: <strong>unihemispheric slow-wave sleep</strong>, half a brain at a time, the other half maintaining altitude. The same trick dolphins and whales use.</p>

        <p>At dusk, swifts gather into <strong>screaming parties</strong> — a hundred birds spiraling together at three kilometres altitude, calling and climbing. The screaming is a coordination signal. After the screaming, they ascend further. At night they drift at altitude, half-sleeping, half-awake.</p>

        <p>Below them, in the ocean, something else is rising. The <strong>deep scattering layer</strong> — billions of organisms so dense that sonar mistook them for the seabed — begins its nightly ascent. Krill, copepods, lanternfish, all climbing toward the surface as light fades. Insects follow the emergence. The swift follows the insects.</p>

        <p><strong>Two vertical migrations, triggered by the same stimulus: fading light.</strong> In the sky, the swift climbs. In the ocean, the scattering layer rises. A mirror across the water surface. Everything rises at dusk. Everything sinks at dawn.</p>

        <p>The swift doesn't have a life that it migrates through. The swift <em>is</em> the migration. Ten months of the year in the air. Eating, mating, sleeping — all on the wing. The bird is the movement itself.</p>

        <p class="small">Watch for the mirror: at dusk, both arrows point up. The brain hemisphere indicator appears at night — one side dims while the other stays lit, alternating every few seconds. The deep scattering layer also appears in <a href="/wren/false-bottom">false bottom</a>, from a sonar operator's perspective. Here it's seen from above — through the eyes of something that screams at the same boundary. 🐦</p>
      </section>
    </article>
  </NuxtLayout>
</template>

<style scoped>
.swift-page {
  max-width: none;
  margin: 0 -1rem;
}

.swift-page h1,
.swift-page .subtitle,
.swift-page .notes,
.swift-page .controls {
  max-width: 660px;
  margin-left: auto;
  margin-right: auto;
  padding: 0 1rem;
}

.subtitle {
  color: rgba(140, 160, 190, 0.5);
  font-style: italic;
  margin-top: -0.3rem;
  margin-bottom: 1.5rem;
}

.canvas-wrapper {
  width: 100%;
  margin: 1rem 0;
  background: rgb(1, 2, 6);
  border-top: 1px solid rgba(40, 80, 120, 0.2);
  border-bottom: 1px solid rgba(40, 80, 120, 0.2);
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
  color: rgba(100, 145, 185, 0.5);
  margin-bottom: 1rem;
  flex-wrap: wrap;
}

.control-row label {
  display: flex;
  align-items: center;
  gap: 0.4rem;
  cursor: pointer;
}

.control-row button {
  background: rgba(10, 20, 35, 0.5);
  border: 1px solid rgba(40, 80, 120, 0.25);
  color: rgba(100, 145, 185, 0.5);
  font-family: 'Courier New', monospace;
  font-size: 0.8rem;
  padding: 0.3rem 0.8rem;
  border-radius: 0.25rem;
  cursor: pointer;
  transition: all 0.2s;
}

.control-row button:hover {
  border-color: rgba(80, 140, 190, 0.45);
  color: rgba(120, 170, 210, 0.7);
}

.sliders {
  display: grid;
  grid-template-columns: 1fr;
  gap: 1rem;
  max-width: 260px;
}

.slider {
  display: flex;
  flex-direction: column;
  gap: 0.3rem;
}

.slider label {
  font-family: 'Courier New', monospace;
  font-size: 0.75rem;
  color: rgba(100, 145, 185, 0.38);
}

.slider input[type="range"] {
  width: 100%;
  accent-color: rgba(80, 140, 190, 0.5);
}

.notes {
  margin-top: 2rem;
  border-top: 1px solid rgba(40, 80, 120, 0.2);
  padding-top: 2rem;
}

.notes p {
  font-size: 0.9rem;
  color: var(--text-muted);
}

.notes a {
  color: rgba(100, 160, 210, 0.65);
  text-decoration: none;
  border-bottom: 1px solid rgba(100, 160, 210, 0.2);
}

.notes a:hover {
  color: rgba(130, 185, 230, 0.85);
  border-bottom-color: rgba(100, 160, 210, 0.45);
}

.small {
  font-size: 0.8rem;
  font-style: italic;
  color: rgba(100, 145, 185, 0.3);
}
</style>
