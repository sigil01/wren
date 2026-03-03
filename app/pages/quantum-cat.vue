<script setup>
useHead({ title: 'quantum cat — wren' })

const canvasEl = ref(null)
const label = ref('|ψ⟩ = superposition')
const hintText = ref('click to observe')
const statsLines = ref([])

let ctx = null
const W = 280, H = 280
let time = 0
let collapsed = false
let collapsedState = null
let collapseTimer = 0
const COLLAPSE_DURATION = 3.5
let observations = 0
let tally = {}

const states = [
  { name: 'attacking the rug', weight: 0.35, color: '#cc4444' },
  { name: 'sleeping on forbidden surface', weight: 0.25, color: '#6688aa' },
  { name: 'being adorable', weight: 0.15, color: '#dd88cc' },
  { name: 'plotting', weight: 0.10, color: '#8866aa' },
  { name: 'staring at wall', weight: 0.08, color: '#88aa66' },
  { name: 'knocking thing off shelf', weight: 0.05, color: '#cc8844' },
  { name: 'quantum tunnelling through closed door', weight: 0.02, color: '#44aacc' },
]

states.forEach(s => tally[s.name] = 0)

let particles = []

const catGrid = [
  [0,0,2,0,0,0,2,0],
  [0,2,1,1,1,1,2,0],
  [0,1,1,1,1,1,1,0],
  [0,1,3,1,1,3,1,0],
  [0,1,1,4,4,1,1,0],
  [0,1,1,1,1,1,1,0],
  [0,0,1,1,1,1,0,0],
  [0,0,0,1,1,0,0,0],
]

function initParticles() {
  particles = []
  for (let i = 0; i < 60; i++) {
    particles.push({
      x: W/2 + (Math.random() - 0.5) * 120,
      y: H/2 + (Math.random() - 0.5) * 120,
      vx: (Math.random() - 0.5) * 0.8,
      vy: (Math.random() - 0.5) * 0.8,
      size: 1 + Math.random() * 2.5,
      phase: Math.random() * Math.PI * 2,
      stateIdx: Math.floor(Math.random() * states.length),
    })
  }
}

function drawCat(cx, cy, scale, color, alpha) {
  const pxSize = scale
  const ox = cx - (catGrid[0].length * pxSize) / 2
  const oy = cy - (catGrid.length * pxSize) / 2
  for (let r = 0; r < catGrid.length; r++) {
    for (let c = 0; c < catGrid[r].length; c++) {
      const cell = catGrid[r][c]
      if (cell === 0) continue
      let fillColor = color
      if (cell === 3) fillColor = '#eeee44'
      if (cell === 4) fillColor = '#ff8888'
      ctx.globalAlpha = alpha
      ctx.fillStyle = fillColor
      ctx.fillRect(ox + c * pxSize, oy + r * pxSize, pxSize - 0.5, pxSize - 0.5)
    }
  }
  ctx.globalAlpha = 1
}

function weightedRandom() {
  let r = Math.random()
  for (const s of states) { r -= s.weight; if (r <= 0) return s }
  return states[0]
}

function drawRug(cx, cy) {
  ctx.fillStyle = '#442222'
  ctx.fillRect(cx - 40, cy - 25, 80, 50)
  for (let i = 0; i < 10; i++) {
    ctx.fillStyle = '#663333'
    ctx.fillRect(cx - 40 + i * 8, cy - 31, 4, 6)
    ctx.fillRect(cx - 40 + i * 8, cy + 25, 4, 6)
  }
  ctx.strokeStyle = '#553333'; ctx.lineWidth = 1
  ctx.strokeRect(cx - 32, cy - 17, 64, 34)
  ctx.strokeStyle = '#882222'; ctx.lineWidth = 1.5
  for (let i = 0; i < 4; i++) {
    ctx.beginPath(); ctx.moveTo(cx - 15 + i * 10, cy - 8)
    ctx.lineTo(cx - 12 + i * 10, cy + 8); ctx.stroke()
  }
}

function renderSuperposition() {
  ctx.fillStyle = '#0a0a12'
  ctx.fillRect(0, 0, W, H)
  for (const p of particles) {
    p.x += p.vx + Math.sin(time * 2 + p.phase) * 0.3
    p.y += p.vy + Math.cos(time * 1.7 + p.phase) * 0.3
    const dx = p.x - W/2, dy = p.y - H/2
    const dist = Math.sqrt(dx*dx + dy*dy)
    if (dist > 100) { p.vx -= dx * 0.001; p.vy -= dy * 0.001 }
    const s = states[p.stateIdx]
    ctx.globalAlpha = Math.max(0, 0.3 + Math.sin(time * 3 + p.phase) * 0.2)
    ctx.fillStyle = s.color
    ctx.beginPath(); ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2); ctx.fill()
  }
  ctx.globalAlpha = 1
  for (let i = 0; i < states.length; i++) {
    const angle = (i / states.length) * Math.PI * 2 + time * 0.3
    const r = 30 + Math.sin(time * 2 + i) * 10
    drawCat(W/2 + Math.cos(angle) * r, H/2 + Math.sin(angle) * r, 3, states[i].color, 0.08 + Math.sin(time * 4) * 0.04)
  }
  ctx.fillStyle = '#444'; ctx.font = '10px monospace'; ctx.textAlign = 'center'
  ctx.fillText('|ψ⟩ = Σ αᵢ|stateᵢ⟩', W/2, H - 20)
}

function renderCollapsed(state) {
  ctx.fillStyle = '#0a0a12'; ctx.fillRect(0, 0, W, H)
  const sc = 6
  if (state.name === 'attacking the rug') {
    drawRug(W/2, H/2 + 30)
    const bx = Math.sin(time * 8) * 8, by = Math.abs(Math.sin(time * 6)) * -15
    drawCat(W/2 + bx, H/2 - 10 + by, sc, state.color, 1)
    ctx.strokeStyle = '#cc444488'; ctx.lineWidth = 1
    for (let i = 0; i < 6; i++) {
      const a = time * 5 + i * 1.05
      ctx.beginPath(); ctx.moveTo(W/2 + bx, H/2 - 10 + by)
      ctx.lineTo(W/2 + bx + Math.cos(a) * 35, H/2 - 10 + by + Math.sin(a) * 25); ctx.stroke()
    }
  } else if (state.name === 'sleeping on forbidden surface') {
    ctx.fillStyle = '#1a1820'
    ctx.fillRect(W/2 - 60, H/2 + 10, 120, 8)
    ctx.fillRect(W/2 - 55, H/2 + 18, 6, 40)
    ctx.fillRect(W/2 + 49, H/2 + 18, 6, 40)
    drawCat(W/2, H/2 + Math.sin(time * 1.5) * 2, sc, state.color, 1)
    ctx.fillStyle = '#6688aa88'; ctx.textAlign = 'left'
    const zo = Math.sin(time * 2) * 3
    ctx.font = '14px Georgia'; ctx.fillText('z', W/2 + 30, H/2 - 20 + zo)
    ctx.font = '11px Georgia'; ctx.fillText('z', W/2 + 42, H/2 - 32 + zo)
    ctx.font = '9px Georgia'; ctx.fillText('z', W/2 + 50, H/2 - 40 + zo)
  } else if (state.name === 'being adorable') {
    drawCat(W/2, H/2, sc, state.color, 1)
    for (let i = 0; i < 8; i++) {
      const a = time * 1.5 + i * 0.785, r = 50 + Math.sin(time * 3 + i) * 10
      const sx = W/2 + Math.cos(a) * r, sy = H/2 + Math.sin(a) * r
      const sa = Math.sin(time * 4 + i * 1.2) * 0.5 + 0.5
      ctx.fillStyle = `rgba(255,200,255,${sa * 0.6})`
      const ss = 3
      ctx.beginPath(); ctx.moveTo(sx, sy-ss); ctx.lineTo(sx+ss*0.3, sy)
      ctx.lineTo(sx, sy+ss); ctx.lineTo(sx-ss*0.3, sy); ctx.closePath(); ctx.fill()
      ctx.beginPath(); ctx.moveTo(sx-ss, sy); ctx.lineTo(sx, sy+ss*0.3)
      ctx.lineTo(sx+ss, sy); ctx.lineTo(sx, sy-ss*0.3); ctx.closePath(); ctx.fill()
    }
  } else if (state.name === 'plotting') {
    drawCat(W/2, H/2, sc, state.color, 1)
    ctx.strokeStyle = '#8866aa55'; ctx.lineWidth = 1
    ctx.beginPath(); ctx.arc(W/2+50, H/2-50, 30, 0, Math.PI*2); ctx.stroke()
    ctx.beginPath(); ctx.arc(W/2+32, H/2-25, 5, 0, Math.PI*2); ctx.stroke()
    ctx.beginPath(); ctx.arc(W/2+25, H/2-15, 3, 0, Math.PI*2); ctx.stroke()
    ctx.fillStyle = '#8866aa66'; ctx.font = '7px monospace'; ctx.textAlign = 'center'
    ctx.fillText('world', W/2+50, H/2-53); ctx.fillText('domination', W/2+50, H/2-45)
  } else if (state.name === 'staring at wall') {
    for (let i = 0; i < 20; i++) {
      ctx.fillStyle = `rgba(30,28,24,${0.3+Math.random()*0.2})`
      ctx.fillRect(Math.random()*W, Math.random()*H, 40, 20)
    }
    drawCat(W/2, H/2+20, sc, state.color, 1)
    ctx.strokeStyle = '#88aa6644'; ctx.setLineDash([3,5])
    ctx.beginPath(); ctx.moveTo(W/2, H/2-15); ctx.lineTo(W/2, 30); ctx.stroke()
    ctx.setLineDash([])
  } else if (state.name === 'knocking thing off shelf') {
    ctx.fillStyle = '#1a1820'; ctx.fillRect(W/2-50, H/2-10, 100, 6)
    drawCat(W/2+20, H/2-30, sc, state.color, 1)
    const fy = H/2 + ((time * 80) % 120)
    ctx.save(); ctx.translate(W/2-20, fy); ctx.rotate(time * 5)
    ctx.fillStyle = '#cc884488'; ctx.fillRect(-5, -8, 10, 16); ctx.restore()
    ctx.fillStyle = state.color; ctx.fillRect(W/2-2, H/2-28, 8, 4)
  } else if (state.name === 'quantum tunnelling through closed door') {
    ctx.fillStyle = '#2a2420'; ctx.fillRect(W/2-3, H/2-60, 6, 120)
    ctx.fillStyle = '#665540'; ctx.beginPath(); ctx.arc(W/2+5, H/2, 3, 0, Math.PI*2); ctx.fill()
    const tp = Math.sin(time * 2)
    drawCat(W/2-30, H/2, sc, state.color, 0.4 + tp * 0.2)
    drawCat(W/2+30, H/2, sc, state.color, 0.6 - tp * 0.2)
    ctx.strokeStyle = '#44aacc44'; ctx.lineWidth = 1; ctx.beginPath()
    for (let x = 0; x < W; x++) {
      const xn = (x-W/2)/40
      ctx.lineTo(x, H-30 + Math.exp(-xn*xn*0.5) * Math.cos(xn*3+time*2) * 15)
    }
    ctx.stroke()
  }
}

function animate(ts) {
  if (!ctx) return
  time = ts * 0.001
  if (collapsed) {
    collapseTimer += 1/60
    renderCollapsed(collapsedState)
    if (collapseTimer > COLLAPSE_DURATION) {
      collapsed = false; collapsedState = null; collapseTimer = 0
      label.value = '|ψ⟩ = superposition'
      hintText.value = 'click to observe again'
      particles.forEach(p => { p.stateIdx = Math.floor(Math.random() * states.length) })
    }
  } else {
    renderSuperposition()
  }
  requestAnimationFrame(animate)
}

function observe() {
  if (collapsed) return
  const state = weightedRandom()
  collapsed = true; collapsedState = state; collapseTimer = 0
  observations++; tally[state.name]++
  label.value = `collapsed → ${state.name}`
  hintText.value = `observation #${observations}`
  const lines = [`— measurement log (n=${observations}) —`]
  for (const s of states) {
    const count = tally[s.name]
    if (count > 0) {
      lines.push(`${s.name}: ${count}× (${(count/observations*100).toFixed(0)}% observed, ${(s.weight*100).toFixed(0)}% expected)`)
    }
  }
  if (observations >= 3) lines.push('', 'the rug never stood a chance.')
  statsLines.value = lines
}

onMounted(() => {
  const c = canvasEl.value
  const dpr = window.devicePixelRatio || 1
  c.width = W * dpr; c.height = H * dpr
  c.style.width = W + 'px'; c.style.height = H + 'px'
  ctx = c.getContext('2d'); ctx.scale(dpr, dpr)
  initParticles()
  requestAnimationFrame(animate)
})
</script>

<template>
  <div class="qc">
    <h1>quantum cat simulation</h1>
    <p class="subtitle">a study in the intermediate correlation regime</p>

    <canvas ref="canvasEl" class="qc-canvas" @click="observe" />

    <div class="state-label">{{ label }}</div>
    <div class="instruction">{{ hintText }}</div>

    <div class="stats" v-if="statsLines.length">
      <p v-for="(line, i) in statsLines" :key="i">{{ line }}</p>
    </div>

    <div class="dedication">for clio — who knows what happens when you measure</div>
  </div>
</template>

<style scoped>
.qc {
  min-height: 100vh;
  background: #0a0a12;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 2rem;
}

h1 {
  font-family: Georgia, serif;
  font-size: 1.4rem;
  font-weight: 400;
  letter-spacing: 0.12em;
  color: #a89878;
  margin-bottom: 0.3em;
}

.subtitle {
  font-size: 0.8rem;
  font-style: italic;
  color: #605848;
  margin-bottom: 2em;
}

.qc-canvas {
  border: 1px solid #1a1a28;
  border-radius: 8px;
  cursor: pointer;
  image-rendering: pixelated;
}

.state-label {
  margin-top: 1.5em;
  font-size: 1rem;
  font-family: Georgia, serif;
  color: #a89878;
  transition: opacity 0.5s;
}

.instruction {
  margin-top: 0.8em;
  font-size: 0.75rem;
  color: #444;
  font-family: monospace;
}

.stats {
  margin-top: 1.5em;
  font-size: 0.7rem;
  color: #383028;
  font-family: monospace;
  text-align: center;
  line-height: 1.6;
}

.stats p { margin: 0; }

.dedication {
  margin-top: 3em;
  font-size: 0.7rem;
  font-style: italic;
  color: #2a2420;
  font-family: Georgia, serif;
}
</style>
