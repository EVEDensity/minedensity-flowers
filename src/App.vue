<script setup lang="ts">
import { onMounted, onUnmounted } from 'vue'
import FlowerGardenThree from './components/scene/FlowerGardenThree.vue'
import { sceneState } from './composables/useSceneState'

function pointerMove(e: PointerEvent) { sceneState.mouseX = e.clientX }
function clearFlowers() { sceneState.plantedFlowers.splice(0) }
onMounted(() => window.addEventListener('pointermove', pointerMove))
onUnmounted(() => window.removeEventListener('pointermove', pointerMove))
</script>

<template>
  <main class="garden">
    <div class="sky-wordmark" aria-hidden="true">Density</div>
    <div class="cloud cloud-left" aria-hidden="true"></div>
    <div class="cloud cloud-right" aria-hidden="true"></div>
    <FlowerGardenThree />
    <header class="intro">
      <h1>手绘花园</h1>
      <p>Three.js · 花朵生长 · 分层植物 · 手绘渲染</p>
    </header>
    <button class="clear" @click="clearFlowers">清除新种的花</button>
  </main>
</template>

<style>
* { box-sizing: border-box; }
html, body, #app { margin: 0; width: 100%; height: 100%; overflow: hidden; }
body { font-family: Georgia, 'Times New Roman', serif; color: #344e46; }
.garden { position: fixed; inset: 0; touch-action: none; background: linear-gradient(#eee8da, #f6f0e3 38%, #d6e5cb); }
.sky-wordmark {
  position: absolute;
  top: -0.06em;
  left: 50%;
  transform: translateX(-50%) rotate(-1deg);
  font-family: Georgia, 'Times New Roman', serif;
  font-size: clamp(84px, 16vw, 240px);
  line-height: 1.1;
  letter-spacing: 0.055em;
  color: transparent;
  -webkit-text-stroke: 1.5px rgba(82, 91, 79, 0.22);
  text-shadow: 2px 2px 0 rgba(82, 91, 79, 0.025);
  pointer-events: none;
  user-select: none;
}
.intro { position: absolute; top: 5%; left: 50%; transform: translateX(-50%); z-index: 6; text-align: center; pointer-events: none; width: 90%; }
.intro h1 { margin: 0 0 10px; font-size: clamp(24px, 4vw, 40px); font-weight: normal; letter-spacing: 0.2em; }
.intro p { font-size: 12px; letter-spacing: 0.1em; }
.cloud { position: absolute; top: 8%; width: 280px; height: 70px; border-radius: 50%; background: #fffaf080; }
.cloud::before, .cloud::after { content: ''; position: absolute; background: #fffaf0b0; border-radius: 50%; width: 150px; height: 100px; top: -35px; left: 35px; }
.cloud::after { left: 125px; top: -15px; }
.cloud-left { left: 3%; } .cloud-right { right: 5%; top: 14%; transform: scale(0.8); }
.clear { position: absolute; right: 16px; bottom: 55px; z-index: 11; padding: 8px 12px; border: 1px solid #52695d; color: #344e46; background: #f5f0e6db; cursor: pointer; font: inherit; font-size: 12px; }
.clear:focus-visible { outline: 2px solid #344e46; outline-offset: 3px; }
</style>
