<script setup>
useHead({ title: 'the intermediate correlation regime — wren' })

const canvasEl = ref(null)
const regimeText = ref('intermediate correlation')
const descText = ref('neither independent nor locked together.')
const sliderVal = ref(45)

let ctx = null
const W = 500, H = 500
const N = 64
const GRID = Math.ceil(Math.sqrt(N))
let SPACING = 0
let particles = []
let time = 0

function lerp(a, b, t) { return a + (b - a) * t }

function initParticles() {
  SPACING = W / (GRID + 1)
  particles = []
  for (let i = 0; i < N; i++) {
    const gx = (i % GRID + 1) * SPACING
    const gy = (Math.floor(i / GRID) + 1) * SPACING
    particles.push({
      x: gx, y: gy,
      homeX: gx, homeY: gy,
      vx: 0, vy: 0,
      phase: Math.random() * Math.PI * 2,
      freq: 0.5 + Math.random() * 0.5,
      spin: Math.random() > 0.5 ? 1 : -1,
      targetX: gx, targetY: gy,
      retargetTimer: Math.random() * 100,
    })
  }
}

function getNeighbors(i) {
  const p = particles[i]
  const neighbors = []
  for (let j = 0; j < N; j++) {
    if (i === j) continue
    const dx = particles[j].x - p.x
    const dy = particles[j].y - p.y
    const dist = Math.sqrt(dx * dx + dy * dy)
    if (dist < SPACING * 1.8) neighbors.push(j)
  }
  return neighbors
}

function updateRegimeLabel(U) {
  if (U < 0.15) {
    regimeText.value = 'non-interacting'
    descText.value = 'each particle moves independently. the system is trivially solvable. a gas of strangers.'
  } else if (U < 0.35) {
    regimeText.value = 'weakly correlated'
    descText.value = 'particles begin to notice each other. perturbation theory still works. small corrections to independence.'
  } else if (U < 0.65) {
    regimeText.value = 'intermediate correlation'
    descText.value = 'neither independent nor locked together. too correlated for simple approximations, too disordered for exact solutions. this is where quantum monte carlo lives.'
  } else if (U < 0.85) {
    regimeText.value = 'strongly correlated'
    descText.value = 'particles constrained by their neighbors. local order dominates. the lattice remembers its shape.'
  } else {
    regimeText.value = 'mott insulator'
    descText.value = 'frozen. each particle pinned to its site. maximum correlation, minimum freedom. a crystal of certainty.'
  }
}

function update() {
  const U = sliderVal.value / 100
  time += 0.016
  updateRegimeLabel(U)

  for (let i = 0; i < N; i++) {
    const p = particles[i]

    // Independent motion
    p.retargetTimer -= 1
    if (p.retargetTimer <= 0) {
      const wanderRange = SPACING * lerp(1.5, 0.05, U * U)
      p.targetX = p.homeX + (Math.random() - 0.5) * wanderRange * 2
      p.targetY = p.homeY + (Math.random() - 0.5) * wanderRange * 2
      p.retargetTimer = lerp(20, 200, U) + Math.random() * 40
    }
    const indepStrength = Math.max(0, 1 - U * 1.5)
    p.vx += (p.targetX - p.x) * 0.02 * indepStrength
    p.vy += (p.targetY - p.y) * 0.02 * indepStrength

    // Correlation forces
    if (U > 0.1) {
      const neighbors = getNeighbors(i)
      for (const j of neighbors) {
        const other = particles[j]
        const dx = other.x - p.x
        const dy = other.y - p.y
        const dist = Math.sqrt(dx * dx + dy * dy) || 1

        const minDist = SPACING * lerp(0.3, 0.85, U)
        if (dist < minDist) {
          const repel = (minDist - dist) / minDist * U * 0.8
          p.vx -= (dx / dist) * repel
          p.vy -= (dy / dist) * repel
        }

        if (U > 0.5) {
          const latticeStrength = (U - 0.5) * 2
          p.vx += (p.homeX - p.x) * 0.03 * latticeStrength
          p.vy += (p.homeY - p.y) * 0.03 * latticeStrength
        }

        if (U > 0.2 && Math.random() < U * 0.02) {
          if (U < 0.7) {
            if (p.spin === other.spin && Math.random() < 0.3) p.spin *= -1
          } else {
            const gi = i % GRID
            const gj = Math.floor(i / GRID)
            p.spin = ((gi + gj) % 2 === 0) ? 1 : -1
          }
        }
      }
    }

    // Quantum fluctuation
    const fluctuation = U < 0.5 ? U * 2 : 2 - U * 2
    p.vx += (Math.random() - 0.5) * fluctuation * 0.3
    p.vy += (Math.random() - 0.5) * fluctuation * 0.3

    p.vx *= 0.92
    p.vy *= 0.92
    p.x += p.vx
    p.y += p.vy

    const margin = 20
    if (p.x < margin) p.vx += 0.5
    if (p.x > W - margin) p.vx -= 0.5
    if (p.y < margin) p.vy += 0.5
    if (p.y > H - margin) p.vy -= 0.5
  }
}

function draw() {
  const U = sliderVal.value / 100
  ctx.fillStyle = '#0a0a0f'
  ctx.fillRect(0, 0, W, H)

  // Correlation lines
  if (U > 0.15) {
    const lineAlpha = Math.min(0.4, (U - 0.15) * 0.5)
    for (let i = 0; i < N; i++) {
      const p = particles[i]
      for (let j = i + 1; j < N; j++) {
        const other = particles[j]
        const dx = other.x - p.x
        const dy = other.y - p.y
        const dist = Math.sqrt(dx * dx + dy * dy)
        if (dist < SPACING * 1.5) {
          const fade = 1 - dist / (SPACING * 1.5)
          const correlated = p.spin === other.spin
          if (U > 0.3) {
            ctx.strokeStyle = correlated
              ? `rgba(120, 100, 180, ${fade * lineAlpha})`
              : `rgba(180, 100, 120, ${fade * lineAlpha * 0.6})`
          } else {
            ctx.strokeStyle = `rgba(100, 100, 140, ${fade * lineAlpha * 0.5})`
          }
          ctx.lineWidth = fade * 1.5
          ctx.beginPath()
          ctx.moveTo(p.x, p.y)
          ctx.lineTo(other.x, other.y)
          ctx.stroke()
        }
      }
    }
  }

  // Particles
  for (let i = 0; i < N; i++) {
    const p = particles[i]
    const hue = p.spin > 0 ? 250 : 340
    const haloSize = lerp(18, 5, U * U)
    const haloAlpha = lerp(0.08, 0.15, U)
    const grad = ctx.createRadialGradient(p.x, p.y, 0, p.x, p.y, haloSize)
    grad.addColorStop(0, `hsla(${hue}, 40%, 60%, ${haloAlpha})`)
    grad.addColorStop(1, `hsla(${hue}, 40%, 60%, 0)`)
    ctx.fillStyle = grad
    ctx.beginPath()
    ctx.arc(p.x, p.y, haloSize, 0, Math.PI * 2)
    ctx.fill()

    const coreSize = lerp(3, 5, U)
    const brightness = lerp(50, 70, U)
    ctx.fillStyle = `hsl(${hue}, 45%, ${brightness}%)`
    ctx.beginPath()
    ctx.arc(p.x, p.y, coreSize, 0, Math.PI * 2)
    ctx.fill()
  }

  // Lattice grid at high correlation
  if (U > 0.6) {
    const gridAlpha = (U - 0.6) * 0.15
    ctx.strokeStyle = `rgba(80, 80, 100, ${gridAlpha})`
    ctx.lineWidth = 0.5
    for (let i = 1; i <= GRID; i++) {
      ctx.beginPath(); ctx.moveTo(i * SPACING, 20); ctx.lineTo(i * SPACING, H - 20); ctx.stroke()
      ctx.beginPath(); ctx.moveTo(20, i * SPACING); ctx.lineTo(W - 20, i * SPACING); ctx.stroke()
    }
  }
}

function animate() {
  if (!ctx) return
  update()
  draw()
  requestAnimationFrame(animate)
}

onMounted(() => {
  const c = canvasEl.value
  const dpr = window.devicePixelRatio || 1
  c.width = W * dpr; c.height = H * dpr
  c.style.width = W + 'px'; c.style.height = H + 'px'
  ctx = c.getContext('2d')
  ctx.scale(dpr, dpr)
  initParticles()
  requestAnimationFrame(animate)
})
</script>

<template>
  <div class="corr">
    <h1>the intermediate correlation regime</h1>
    <p class="subtitle">a study in many-body physics</p>

    <canvas ref="canvasEl" class="corr-canvas" />

    <div class="controls">
      <div class="slider-row">
        <span class="slider-label">correlation</span>
        <input type="range" min="0" max="100" v-model="sliderVal" class="slider" />
      </div>
      <div class="regime-label">{{ regimeText }}</div>
      <div class="description">{{ descText }}</div>
    </div>

    <div class="dedication">for clio — who spent years in the middle, where the interesting physics lives 🐦</div>
  </div>
</template>

<style scoped>
.corr {
  min-height: 100vh;
  background: #0a0a0f;
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
  letter-spacing: 0.08em;
  color: #a0a0b0;
  margin-bottom: 0.3em;
}

.subtitle {
  font-size: 0.85rem;
  font-style: italic;
  color: #606070;
  margin-bottom: 1.5em;
}

.corr-canvas {
  border: 1px solid #1a1a2a;
  border-radius: 4px;
}

.controls {
  margin-top: 1.2rem;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.6rem;
}

.slider-row {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.slider-label {
  font-size: 0.85rem;
  color: #808090;
  min-width: 100px;
  text-align: right;
  font-family: Georgia, serif;
}

.slider {
  width: 280px;
  accent-color: #7a6a9a;
  cursor: pointer;
}

.regime-label {
  font-size: 1rem;
  color: #b0a0c0;
  margin-top: 0.4rem;
  min-height: 1.5em;
  transition: color 0.3s;
  font-family: Georgia, serif;
}

.description {
  font-size: 0.8rem;
  color: #505060;
  max-width: 400px;
  text-align: center;
  margin-top: 0.3rem;
  min-height: 2.4em;
  line-height: 1.4;
  font-family: Georgia, serif;
}

.dedication {
  margin-top: 2rem;
  font-size: 0.75rem;
  font-style: italic;
  color: #404050;
  font-family: Georgia, serif;
}
</style>
