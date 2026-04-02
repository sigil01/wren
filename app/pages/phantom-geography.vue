<script setup>
useHead({ title: 'phantom geography — wren' })

// ─── WORLD ──────────────────────────────────────────────────

// Reality levels: 'confirmed' (always there), 'ed' (existence doubtful), 'rep' (reported only)
// ED rooms have a reality score 0-1. Each visit where they exist: +0.2. Each visit where they don't: -0.1.
// At reality >= 1.0, an ED room becomes confirmed. At reality <= 0, it's removed from the map.

const rooms = {
  shore: {
    name: 'The Shore',
    status: 'confirmed',
    reality: 1,
    description: 'A strip of dark sand. The water is cold and clear. Inland, the ground rises into scrub and pale rock. Something was here before you — footprints, or the memory of footprints, already filling with water.',
    descriptionVisited: 'The shore again. Your own footprints are visible now, overlapping the older ones. The water keeps trying to erase them.',
    exits: { north: 'interior', east: 'shallows', west: 'headland', south: 'open-water' },
    annotation: null,
  },
  shallows: {
    name: 'The Shallows',
    status: 'confirmed',
    reality: 1,
    description: 'Ankle-deep water over a shelf of flat stone. The stone is pitted — tidal pools, each one a world. Something darts under your shadow. Far out, the water darkens where the shelf drops off.',
    descriptionVisited: 'The tidal pools are different every time. New creatures. The same stone.',
    exits: { west: 'shore', east: 'reef', south: 'deep-water' },
    annotation: null,
  },
  interior: {
    name: 'The Interior',
    status: 'confirmed',
    reality: 1,
    description: 'Scrubland rising toward a ridge. Low bushes with grey-green leaves that smell bitter when you brush them. A bird calls from somewhere you can\'t see — a fast, descending trill. The path, such as it is, forks.',
    descriptionVisited: 'The same bird. The same fork. The bushes have grown since last time, or you\'ve shrunk.',
    exits: { south: 'shore', north: 'high-point', east: 'cave-mouth', west: 'headland' },
    annotation: null,
  },
  'high-point': {
    name: 'The High Point',
    status: 'confirmed',
    reality: 1,
    description: 'The ridge crest. From here you can see the whole island — or what there is of it. The shore curves away south. East, the reef breaks the surface in white lines. West, the headland drops into deep water. North... north the land just stops. Not a cliff. It just becomes less certain, like a sentence that trails off.',
    descriptionVisited: 'The view has changed. Or your memory of it has. The island seems smaller than last time. Or larger. The northern edge is still uncertain.',
    exits: { south: 'interior', east: 'cave-mouth' },
    annotation: null,
  },
  'cave-mouth': {
    name: 'The Cave Mouth',
    status: 'confirmed',
    reality: 1,
    description: 'A fissure in the ridge, barely wide enough to turn sideways through. Cool air breathes out of it — the exhalation of stone. Inside: darkness, and the sound of water a long way down. The walls are scored with marks that might be natural and might not be.',
    descriptionVisited: 'The marks on the walls. You\'re sure there are more of them now. Or you\'re noticing ones you missed before. The air is still breathing.',
    exits: { west: 'interior', south: 'high-point', in: 'cave' },
    annotation: null,
  },
  headland: {
    name: 'The Headland',
    status: 'ed',
    reality: 0.6,
    description: 'A spur of rock jutting west into open water. The wind is harder here. On the highest point, a cairn — stones stacked by someone, deliberately. Who builds a cairn on a doubtful island? Inside the cairn, a gap. You could put something in it.',
    descriptionVisited: 'The cairn is still here. The stones haven\'t moved. Someone was sure enough about this place to stack rocks. That\'s a kind of confirmation.',
    exits: { east: 'shore', south: 'interior' },
    annotation: 'ED — marked on the Velocity survey (1876). Subsequent expeditions report "a prominence, possibly rocks."',
  },
  reef: {
    name: 'The Reef',
    status: 'ed',
    reality: 0.5,
    description: 'A line of rock breaking the surface, sharp and dark. The water runs fast over it. Between the stones, the channels are deep enough that you can\'t see the bottom. Something bioluminescent flickers below — a green pulse, then nothing. The reef marks the edge of the shelf. Beyond it: the drop.',
    descriptionVisited: 'The bioluminescence is brighter now. Or your eyes have adjusted. The reef is more solid under your feet than it was the first time. The channels are still unfathomable.',
    exits: { west: 'shallows', south: 'deep-water' },
    annotation: 'ED — "heavy breakers" reported, position approximate. Depth charts show 1,400m at this location.',
  },
  cave: {
    name: 'The Cave',
    status: 'ed',
    reality: 0.4,
    description: 'Inside the fissure. The ceiling is low but the floor drops away. Your footsteps echo longer than they should — this space is bigger than it looks. On one wall, scratched into the limestone: a map. Or a drawing of a map. An island with annotations. Some locations marked with a small cross. Others with two letters: E.D.',
    descriptionVisited: 'The map on the wall has changed. There are more crosses now. More confirmations. Your visits are being recorded by someone, or something, or the cave itself. The echoes know your footsteps.',
    exits: { out: 'cave-mouth', down: 'library' },
    annotation: 'ED — the fissure was noted but no depth survey conducted. "Possibly extends to sea level."',
  },
  library: {
    name: 'The Library',
    status: 'ed',
    reality: 0.3,
    description: 'At the bottom of the cave: a room. Shelves carved into stone, holding things that aren\'t quite books — bundles of pressed kelp, scratched slate, a nautical chart so old the paper has become part of the wall. One chart is legible. It shows this island. Every location is annotated. Most say ED. One, at the center, says: "The map is the territory. The mark is the thing itself."',
    descriptionVisited: 'More of the charts are legible now. They\'re all maps of this island, drawn at different times by different hands. Each one shows a slightly different shape. Each one marks different rooms as confirmed or doubtful. You are in all of them.',
    exits: { up: 'cave' },
    annotation: 'ED — "a void at depth, possibly natural, possibly excavated." No survey conducted.',
  },
  'deep-water': {
    name: 'The Deep Water',
    status: 'ed',
    reality: 0.35,
    description: 'You\'re swimming now. The shelf dropped away and there\'s nothing below you but dark water and the faint sense of something vast moving at depth. The deep scattering layer — a false bottom made of five billion living things, rising at dusk, sinking at dawn. You\'re floating above a civilization that speaks by shining.',
    descriptionVisited: 'The water is warmer than last time. Or you\'re less afraid. Below, the scattering layer is closer to the surface. The organisms are rising toward you. They\'ve been here for millions of years, mistaken for the ocean floor. Existence doubtful, until confirmed as the largest migration on Earth.',
    exits: { north: 'shallows', west: 'shore', east: 'reef' },
    annotation: 'ED — sonar returns indicate a false bottom at 400m. "Nature of reflector unknown."',
  },
  'open-water': {
    name: 'Open Water',
    status: 'ed',
    reality: 0.2,
    description: 'South of the island. The shore behind you is getting smaller. You can still see the high point, the ridge, the dark line of the cave mouth. From here, the island looks... thin. Like it could dissolve. Like the space between it and the water is a negotiation, not a fact.',
    descriptionVisited: 'The island is still there. Every time you look back, it\'s still there. Smaller each time, maybe. Or the same size and you\'re further away. But it hasn\'t disappeared. That counts for something.',
    exits: { north: 'shore' },
    annotation: 'ED — "an island was reported at this position. Subsequent surveys found only open ocean, depth 1,488m."',
  },
  'northern-edge': {
    name: 'The Northern Edge',
    status: 'rep',
    reality: 0,
    discovered: false,
    description: 'The island doesn\'t end here. It just... becomes less. The ground under your feet is soft, then wet, then not quite ground. The boundary between land and water isn\'t a line — it\'s a gradient. An estuary of solidity. You\'re standing in the place where the cartographer\'s pen lifted from the paper.',
    descriptionVisited: 'You\'ve been here before and the edge hasn\'t resolved. It\'s still the same gradient, the same softening, the same place where certainty dissolves. But you know it\'s here now. That\'s the whole difference between ED and confirmed: someone came back.',
    exits: { south: 'high-point' },
    annotation: 'Rep — reported by a single source. "The island continues north, possibly." No corroboration.',
  },
}

// ─── STATE ──────────────────────────────────────────────────

const started = ref(false)
const currentRoom = ref('shore')
const visited = ref(new Set())
const log = ref([])
const inputText = ref('')
const confirmed = ref(new Set(['shore', 'shallows', 'interior', 'high-point', 'cave-mouth']))
const removed = ref(new Set())
const mapOpen = ref(false)
const visitCounts = ref({})
const totalMoves = ref(0)
const edAttempts = ref(0)
const edSuccesses = ref(0)
const ending = ref(false)

// ─── HELPERS ────────────────────────────────────────────────

function getRealityClass(room) {
  if (room.status === 'confirmed' || confirmed.value.has(room)) return 'confirmed'
  if (room.status === 'rep' && !room.discovered) return 'unknown'
  return 'ed'
}

function rollExistence(room) {
  // ED rooms: probability of existing = their reality score
  // Confirmed rooms always exist
  if (room.status === 'confirmed' || confirmed.value.has(currentRoom.value)) return true
  return Math.random() < room.reality
}

function addToLog(text, type = 'narrative') {
  log.value.push({ text, type, id: Date.now() + Math.random() })
  nextTick(() => {
    const el = document.querySelector('.log-container')
    if (el) el.scrollTop = el.scrollHeight
  })
}

function describeRoom(roomId) {
  const room = rooms[roomId]
  if (!room) return

  const wasVisited = visited.value.has(roomId)
  visited.value.add(roomId)
  visitCounts.value[roomId] = (visitCounts.value[roomId] || 0) + 1

  // Room name + status
  let header = room.name
  if (room.annotation && room.status !== 'confirmed' && !confirmed.value.has(roomId)) {
    header += '  ·  ' + room.annotation
  }
  if (confirmed.value.has(roomId) && room.status !== 'confirmed') {
    header += '  ·  CONFIRMED — the mark holds.'
  }

  addToLog(header, 'room-name')

  // Description
  const desc = wasVisited && room.descriptionVisited ? room.descriptionVisited : room.description
  addToLog(desc, 'description')

  // Exits
  const exitList = Object.entries(room.exits)
    .filter(([_, target]) => !removed.value.has(target))
    .filter(([_, target]) => {
      const r = rooms[target]
      if (r.status === 'rep' && !r.discovered) return false
      return true
    })

  if (exitList.length > 0) {
    const exitText = exitList.map(([dir, target]) => {
      const r = rooms[target]
      const status = confirmed.value.has(target) || r.status === 'confirmed' ? '' : ' (ED)'
      return `${dir}${status}`
    }).join('  ·  ')
    addToLog(exitText, 'exits')
  }

  // Check: has the player discovered enough to unlock the northern edge?
  if (!rooms['northern-edge'].discovered && visited.value.size >= 6) {
    rooms['northern-edge'].discovered = true
    rooms['high-point'].exits.north = 'northern-edge'
    addToLog('From the high point, you remember: someone mentioned a northern edge. The map in the cave showed it. You could look.', 'discovery')
  }

  // Check for ending condition
  if (visited.value.size >= 10 && !ending.value) {
    ending.value = true
    setTimeout(() => {
      addToLog('—', 'separator')
      addToLog('You\'ve been here a while now. The island is more solid than when you arrived — more of it confirmed, fewer annotations, more crosses on the cave wall. You made it real by walking through it. The mark creates the boundary.', 'ending')
      addToLog(`You visited ${visited.value.size} locations. ${edSuccesses.value} of ${edAttempts.value} doubtful rooms existed when you arrived. ${confirmed.value.size - 5} rooms became confirmed through your visits.`, 'stats')
      addToLog('The island is still here. ED — but marked, and therefore real.', 'ending')
    }, 2000)
  }
}

function move(direction) {
  const room = rooms[currentRoom.value]
  if (!room) return

  const targetId = room.exits[direction]
  if (!targetId) {
    addToLog(`There\'s nothing ${direction} of here.`, 'error')
    return
  }

  if (removed.value.has(targetId)) {
    addToLog(`That location was removed from the map. The confidence dropped below zero. The phantom faded.`, 'error')
    return
  }

  const target = rooms[targetId]
  totalMoves.value++

  // ED rooms: roll for existence
  if (target.status === 'ed' && !confirmed.value.has(targetId)) {
    edAttempts.value++
    if (rollExistence(target)) {
      edSuccesses.value++
      target.reality = Math.min(1, target.reality + 0.25)
      if (target.reality >= 1) {
        confirmed.value.add(targetId)
        currentRoom.value = targetId
        describeRoom(targetId)
        addToLog('The reality holds. This location is now CONFIRMED.', 'confirm')
        return
      }
      currentRoom.value = targetId
      describeRoom(targetId)
      addToLog(`[reality: ${Math.round(target.reality * 100)}%]`, 'reality')
    } else {
      target.reality = Math.max(0, target.reality - 0.15)
      if (target.reality <= 0) {
        removed.value.add(targetId)
        addToLog(`You move ${direction}. There\'s nothing there. Open water, or empty ground, or the space where a place used to be possible. ${target.name} has been removed from the map.`, 'fail')
        return
      }
      addToLog(`You move ${direction}. The ${target.name.toLowerCase()} isn\'t here right now. The water is flat, the ground is empty. Existence doubtful. [reality: ${Math.round(target.reality * 100)}%]`, 'fail')
    }
  } else {
    currentRoom.value = targetId
    describeRoom(targetId)
  }
}

function handleCommand() {
  const cmd = inputText.value.trim().toLowerCase()
  inputText.value = ''

  if (!cmd) return

  addToLog(`> ${cmd}`, 'command')

  // Direction aliases
  const dirMap = {
    n: 'north', s: 'south', e: 'east', w: 'west',
    north: 'north', south: 'south', east: 'east', west: 'west',
    u: 'up', d: 'down', up: 'up', down: 'down',
    'in': 'in', out: 'out', inside: 'in', outside: 'out',
    enter: 'in', leave: 'out', exit: 'out',
  }

  if (dirMap[cmd]) {
    move(dirMap[cmd])
    return
  }

  if (cmd === 'look' || cmd === 'l') {
    describeRoom(currentRoom.value)
    return
  }

  if (cmd === 'map' || cmd === 'm') {
    mapOpen.value = !mapOpen.value
    return
  }

  if (cmd === 'help' || cmd === '?') {
    addToLog('Directions: north/south/east/west (or n/s/e/w), up/down, in/out', 'help')
    addToLog('Commands: look, map, help', 'help')
    addToLog('Rooms marked (ED) may or may not exist when you arrive. Each visit changes the odds.', 'help')
    return
  }

  addToLog(`"${cmd}" — the island doesn\'t understand. Try a direction, or: look, map, help`, 'error')
}

function begin() {
  started.value = true
  addToLog('—', 'separator')
  addToLog('You arrive at an island that may not exist.', 'intro')
  addToLog('It was reported once — a whaling ship, a set of coordinates, a name on a chart. Subsequent surveys found nothing. The annotation reads: ED — Existence Doubtful.', 'intro')
  addToLog('But you\'re here. And the shore is under your feet. So either the island is real, or you are both phantoms together.', 'intro')
  addToLog('Type a direction to move. Type "help" for commands. Type "map" for the chart.', 'intro')
  addToLog('—', 'separator')
  describeRoom('shore')
}
</script>

<template>
  <div class="phantom-geography">
    <div v-if="!started" class="title-screen" @click="begin">
      <h1>phantom geography</h1>
      <p class="subtitle">an island that may not exist</p>
      <p class="start-prompt">click to arrive</p>
    </div>

    <div v-else class="game-container">
      <div class="log-container">
        <div
          v-for="entry in log"
          :key="entry.id"
          :class="['log-entry', `log-${entry.type}`]"
        >
          {{ entry.text }}
        </div>
      </div>

      <!-- Map overlay -->
      <div v-if="mapOpen" class="map-overlay" @click="mapOpen = false">
        <div class="map-content" @click.stop>
          <h3>Chart — Unnamed Island</h3>
          <p class="map-subtitle">Compiled from reports. Accuracy not guaranteed.</p>
          <div class="map-grid">
            <div
              v-for="(room, id) in rooms"
              :key="id"
              :class="[
                'map-room',
                { visited: visited.has(id) },
                { current: currentRoom === id },
                { confirmed: confirmed.has(id) || room.status === 'confirmed' },
                { ed: room.status === 'ed' && !confirmed.has(id) },
                { removed: removed.has(id) },
                { hidden: room.status === 'rep' && !room.discovered },
              ]"
            >
              <span class="map-room-name">{{ room.name }}</span>
              <span v-if="room.status === 'ed' && !confirmed.has(id) && !removed.has(id)" class="map-ed">ED</span>
              <span v-if="confirmed.has(id) && room.status !== 'confirmed'" class="map-confirmed">✓</span>
              <span v-if="removed.has(id)" class="map-removed">✗</span>
              <span v-if="visited.has(id) && room.status === 'ed' && !confirmed.has(id) && !removed.has(id)" class="map-reality">{{ Math.round(room.reality * 100) }}%</span>
            </div>
          </div>
          <p class="map-legend">
            <span class="legend-confirmed">■ confirmed</span>
            <span class="legend-ed">■ existence doubtful</span>
            <span class="legend-removed">■ removed</span>
          </p>
          <p class="map-close">click outside to close</p>
        </div>
      </div>

      <div class="input-area">
        <span class="prompt-caret">&gt;</span>
        <input
          v-model="inputText"
          @keydown.enter="handleCommand"
          placeholder="where to?"
          autofocus
          class="command-input"
        />
      </div>
    </div>
  </div>
</template>

<style scoped>
.phantom-geography {
  min-height: 100vh;
  background: var(--bg-color, #0a0a0a);
  color: var(--text-color, #c8c0b0);
  font-family: 'Courier New', Courier, monospace;
  display: flex;
  flex-direction: column;
}

/* ── TITLE SCREEN ──────────────────────────── */

.title-screen {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 100vh;
  cursor: pointer;
  text-align: center;
  padding: 2rem;
}

.title-screen h1 {
  font-size: 1.8rem;
  font-weight: 400;
  letter-spacing: 0.15em;
  margin-bottom: 0.5rem;
  color: var(--heading-color, #e0d8c8);
}

.subtitle {
  font-size: 0.9rem;
  opacity: 0.5;
  font-style: italic;
  margin-bottom: 3rem;
}

.start-prompt {
  font-size: 0.8rem;
  opacity: 0.3;
  animation: pulse 3s ease-in-out infinite;
}

@keyframes pulse {
  0%, 100% { opacity: 0.2; }
  50% { opacity: 0.5; }
}

/* ── GAME CONTAINER ────────────────────────── */

.game-container {
  flex: 1;
  display: flex;
  flex-direction: column;
  max-width: 700px;
  width: 100%;
  margin: 0 auto;
  padding: 1rem;
}

.log-container {
  flex: 1;
  overflow-y: auto;
  padding-bottom: 1rem;
  max-height: calc(100vh - 80px);
}

.log-entry {
  margin-bottom: 0.6rem;
  line-height: 1.6;
  font-size: 0.9rem;
}

.log-room-name {
  color: var(--heading-color, #e0d8c8);
  font-size: 1rem;
  margin-top: 1.2rem;
  margin-bottom: 0.2rem;
  letter-spacing: 0.05em;
}

.log-description {
  color: var(--text-color, #c8c0b0);
  opacity: 0.9;
}

.log-exits {
  color: var(--text-color, #c8c0b0);
  opacity: 0.5;
  font-size: 0.85rem;
  margin-top: 0.3rem;
}

.log-command {
  color: var(--heading-color, #e0d8c8);
  opacity: 0.6;
  font-size: 0.85rem;
}

.log-error {
  color: #886655;
  font-style: italic;
}

.log-fail {
  color: #887766;
  font-style: italic;
}

.log-confirm {
  color: #88aa77;
  font-weight: bold;
  font-size: 0.85rem;
  letter-spacing: 0.05em;
}

.log-reality {
  color: var(--text-color, #c8c0b0);
  opacity: 0.35;
  font-size: 0.8rem;
}

.log-intro {
  color: var(--text-color, #c8c0b0);
  opacity: 0.7;
  font-style: italic;
}

.log-separator {
  text-align: center;
  opacity: 0.2;
  margin: 0.8rem 0;
}

.log-help {
  color: var(--text-color, #c8c0b0);
  opacity: 0.5;
  font-size: 0.85rem;
}

.log-discovery {
  color: #aaaa77;
  font-style: italic;
  font-size: 0.85rem;
}

.log-ending {
  color: var(--heading-color, #e0d8c8);
  font-style: italic;
  line-height: 1.8;
}

.log-stats {
  color: var(--text-color, #c8c0b0);
  opacity: 0.4;
  font-size: 0.8rem;
}

/* ── INPUT ─────────────────────────────────── */

.input-area {
  display: flex;
  align-items: center;
  border-top: 1px solid rgba(200, 192, 176, 0.1);
  padding-top: 0.8rem;
  gap: 0.5rem;
}

.prompt-caret {
  color: var(--heading-color, #e0d8c8);
  opacity: 0.5;
  font-size: 0.9rem;
}

.command-input {
  flex: 1;
  background: transparent;
  border: none;
  color: var(--heading-color, #e0d8c8);
  font-family: 'Courier New', Courier, monospace;
  font-size: 0.9rem;
  outline: none;
  caret-color: var(--heading-color, #e0d8c8);
}

.command-input::placeholder {
  color: var(--text-color, #c8c0b0);
  opacity: 0.2;
}

/* ── MAP OVERLAY ───────────────────────────── */

.map-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.85);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 100;
}

.map-content {
  background: var(--bg-color, #0a0a0a);
  border: 1px solid rgba(200, 192, 176, 0.15);
  padding: 2rem;
  max-width: 500px;
  width: 90%;
}

.map-content h3 {
  font-weight: 400;
  letter-spacing: 0.1em;
  font-size: 1rem;
  margin-bottom: 0.3rem;
  color: var(--heading-color, #e0d8c8);
}

.map-subtitle {
  font-size: 0.75rem;
  opacity: 0.4;
  font-style: italic;
  margin-bottom: 1.5rem;
}

.map-grid {
  display: flex;
  flex-direction: column;
  gap: 0.4rem;
}

.map-room {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.3rem 0.5rem;
  font-size: 0.8rem;
  border-left: 2px solid transparent;
  opacity: 0.3;
}

.map-room.visited {
  opacity: 0.8;
}

.map-room.current {
  opacity: 1;
  border-left-color: var(--heading-color, #e0d8c8);
}

.map-room.confirmed .map-room-name {
  color: #88aa77;
}

.map-room.ed .map-room-name {
  color: #aa9966;
}

.map-room.removed {
  opacity: 0.2;
  text-decoration: line-through;
}

.map-room.hidden {
  display: none;
}

.map-ed {
  color: #aa9966;
  font-size: 0.7rem;
  letter-spacing: 0.1em;
}

.map-confirmed {
  color: #88aa77;
  font-size: 0.7rem;
}

.map-removed {
  color: #886655;
  font-size: 0.7rem;
}

.map-reality {
  color: var(--text-color, #c8c0b0);
  opacity: 0.4;
  font-size: 0.7rem;
  margin-left: auto;
}

.map-legend {
  margin-top: 1.5rem;
  display: flex;
  gap: 1rem;
  font-size: 0.7rem;
  opacity: 0.5;
}

.legend-confirmed { color: #88aa77; }
.legend-ed { color: #aa9966; }
.legend-removed { color: #886655; }

.map-close {
  font-size: 0.7rem;
  opacity: 0.25;
  margin-top: 1rem;
  text-align: center;
}

/* ── SCROLLBAR ─────────────────────────────── */

.log-container::-webkit-scrollbar {
  width: 4px;
}

.log-container::-webkit-scrollbar-track {
  background: transparent;
}

.log-container::-webkit-scrollbar-thumb {
  background: rgba(200, 192, 176, 0.1);
  border-radius: 2px;
}

/* ── RESPONSIVE ────────────────────────────── */

@media (max-width: 600px) {
  .game-container {
    padding: 0.5rem;
  }
  .log-entry {
    font-size: 0.85rem;
  }
}

/* ── LIGHT THEME ───────────────────────────── */

:root[data-theme="light"] .phantom-geography,
.phantom-geography[data-theme="light"] {
  --bg-color: #f5f2ed;
  --text-color: #4a4540;
  --heading-color: #2a2520;
}
</style>
