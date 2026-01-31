<script setup>
import { RouterView } from 'vue-router'
</script>

<template>
  <div class="cyber-shell">
    <div class="cyber-grid-floor"></div>
    <div class="cyber-glow"></div>
    <RouterView v-slot="{ Component }">
      <Transition name="fade" mode="out-in">
        <component :is="Component" />
      </Transition>
    </RouterView>
  </div>
</template>

<style>
@import url('https://fonts.googleapis.com/css2?family=Outfit:wght@300;500;700;900&display=swap');
:root { --bg: #030712; --surface: #111827; --surface-highlight: #1f2937; --primary: #8b5cf6; --primary-glow: rgba(139, 92, 246, 0.5); --accent: #06b6d4; --success: #10b981; --danger: #ef4444; --text: #f3f4f6; --text-dim: #9ca3af; --border: rgba(255,255,255,0.08); }
* { box-sizing: border-box; margin: 0; padding: 0; outline: none; -webkit-tap-highlight-color: transparent; }
body { background: var(--bg); color: var(--text); font-family: 'Outfit', sans-serif; overflow: hidden; height: 100vh; }
.cyber-shell { width: 100vw; height: 100vh; position: relative; overflow: hidden; }
.cyber-grid-floor { position: absolute; bottom: 0; left: -50%; width: 200%; height: 50%; background: linear-gradient(transparent 0%, var(--primary-glow) 100%); transform: perspective(500px) rotateX(60deg); opacity: 0.1; pointer-events: none; z-index: 0; background-image: linear-gradient(var(--bg) 2px, transparent 2px), linear-gradient(90deg, var(--bg) 2px, transparent 2px); background-size: 40px 40px; }
.cyber-glow { position: absolute; top: -20%; right: -10%; width: 600px; height: 600px; background: radial-gradient(circle, var(--primary) 0%, transparent 70%); opacity: 0.15; filter: blur(80px); pointer-events: none; z-index: 0; }
.fade-enter-active, .fade-leave-active { transition: opacity 0.3s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>