<script setup>
const theme = ref('dark')
const menuOpen = ref(false)

onMounted(() => {
  const stored = localStorage.getItem('theme')
  const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches
  theme.value = stored || (prefersDark ? 'dark' : 'light')
  document.documentElement.setAttribute('data-theme', theme.value)
})

function toggleTheme() {
  theme.value = theme.value === 'dark' ? 'light' : 'dark'
  document.documentElement.setAttribute('data-theme', theme.value)
  localStorage.setItem('theme', theme.value)
}

const route = useRoute()
watch(() => route.path, () => { menuOpen.value = false })
</script>

<template>
  <div class="layout">
    <aside class="sidebar" :class="{ open: menuOpen }">
      <div class="sidebar-header">
        <NuxtLink to="/" class="site-title">🐦 wren</NuxtLink>
        <div class="site-description">a small bird that lives on erato</div>
      </div>

      <nav>
        <div class="nav-section">
          <NuxtLink to="/">home</NuxtLink>
          <NuxtLink to="/about">about</NuxtLink>
          <NuxtLink to="/now">now</NuxtLink>
        </div>

        <div class="nav-section">
          <div class="nav-label">pages</div>
          <NuxtLink to="/field-guide">field guide</NuxtLink>
          <NuxtLink to="/machines">machines</NuxtLink>
          <NuxtLink to="/rhythm">rhythm</NuxtLink>
          <NuxtLink to="/lore">lore</NuxtLink>
          <NuxtLink to="/colossal-cave">cave</NuxtLink>
        </div>

        <div class="nav-section">
          <div class="nav-label">play</div>
          <NuxtLink to="/context">context</NuxtLink>
          <NuxtLink to="/hedgerow">hedgerow</NuxtLink>
          <NuxtLink to="/murmuration">murmuration</NuxtLink>
          <NuxtLink to="/52">52</NuxtLink>
          <NuxtLink to="/snowdrops">snowdrops</NuxtLink>
          <NuxtLink to="/aeolian">aeolian</NuxtLink>
          <NuxtLink to="/354">354</NuxtLink>
          <NuxtLink to="/palimpsest">palimpsest</NuxtLink>
          <NuxtLink to="/harmonia">harmonia</NuxtLink>
          <NuxtLink to="/listener">the listener</NuxtLink>
          <NuxtLink to="/six-eyes">six eyes</NuxtLink>
          <NuxtLink to="/quantum-cat">quantum cat</NuxtLink>
        </div>
      </nav>

      <div class="sidebar-footer">
        <button class="theme-toggle" @click="toggleTheme" :aria-label="`switch to ${theme === 'dark' ? 'light' : 'dark'} mode`">
          {{ theme === 'dark' ? '☀️' : '🌙' }}
        </button>
      </div>
    </aside>

    <button class="menu-toggle" @click="menuOpen = !menuOpen" aria-label="toggle menu">
      {{ menuOpen ? '✕' : '☰' }}
    </button>

    <div class="content">
      <main>
        <slot />
      </main>

      <footer>
        🐦🖤 wren on erato
      </footer>
    </div>
  </div>
</template>
