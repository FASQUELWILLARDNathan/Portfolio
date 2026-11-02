<template>
  <div ref="root" class="basement">
    <canvas ref="canvasEl" class="world" aria-label="Scène 3D d'une cave avec ordinateur rétro" />
    <div class="hud"><span>THE TERMINAL / 1998</span><span>{{ Math.round(progress * 100) }}% CONNECTION</span></div>
    <div class="intro" :style="{ opacity: Math.max(0, 1 - progress * 4) }">
      <h1>THE TERMINAL<span>_</span></h1><p>AN ABANDONED CONNECTION</p><small>SCROLL TO APPROACH ↓</small>
    </div>
    <div v-if="progress > .65 && progress < .98" class="hint">ESTABLISHING CONNECTION...</div>
    <div class="crt-controls" :class="{ active: terminalActive }" :inert="!terminalActive">
      <div class="crt-controls-label"><span>● NATHAN.OS — CONNECTED</span><button type="button" @click="leaveTerminal">← RETURN</button></div>
      <form class="crt-input-row" @submit.prevent="runCommand">
        <span>~$</span><input ref="inputEl" v-model="command" aria-label="Commande du terminal CRT" autocomplete="off" autocapitalize="off" spellcheck="false" placeholder="Type help..." :tabindex="terminalActive ? 0 : -1" />
        <button type="submit">ENTER ↵</button>
      </form>
      <nav class="crt-shortcuts" aria-label="Commandes rapides"><button v-for="c in shortcuts" :key="c" type="button" @click="execute(c)">{{ c }}</button></nav>
    </div>
    <div class="progress-track" aria-hidden="true"><div :style="{ transform: `scaleY(${progress})` }" /></div>
  </div>
</template>

<script setup lang="ts">
import { ref, nextTick, onMounted, onBeforeUnmount, watch } from 'vue'
import * as THREE from 'three'
import { ScrollTrigger } from 'gsap/ScrollTrigger'
import gsap from 'gsap'
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js'

const root = ref<HTMLElement | null>(null)
const canvasEl = ref<HTMLCanvasElement | null>(null)
const inputEl = ref<HTMLInputElement | null>(null)
const progress = ref(0)
const terminalActive = ref(false)
const command = ref('')
const shortcuts = ['help', 'whoami', 'projects', 'skills', 'contact', 'clear'] as const
const lines = ref<string[]>([
  'NATHAN.OS [Version 1.0.98]', 'Copyright (c) 1998-2026', '',
  'Connection established. Type "help" to begin.', ''
])

let scene: THREE.Scene | undefined
let camera: THREE.PerspectiveCamera | undefined
let renderer: THREE.WebGLRenderer | undefined
let screenCanvas: HTMLCanvasElement | undefined
let screenTexture: THREE.CanvasTexture | undefined
let trigger: ScrollTrigger | undefined
let frame = 0
let reduced = false
let destroyed = false
let lastCRT = 0
const clock = new THREE.Clock()
const ownedGeometries = new Set<THREE.BufferGeometry>()
const ownedMaterials = new Set<THREE.Material>()
const ownedTextures = new Set<THREE.Texture>()
const from = new THREE.Vector3(2.2, 1.4, 4.9)
const seat = new THREE.Vector3(.35, .65, -.15)
const close = new THREE.Vector3(0, 1.25, -.65)
const lookFrom = new THREE.Vector3(0, .75, -3.5)
const lookClose = new THREE.Vector3(0, .65, -3.41)
const target = new THREE.Vector3()
const look = new THREE.Vector3()
const smooth = (v: number) => v * v * (3 - 2 * v)
const clamp01 = (v: number) => Math.max(0, Math.min(1, v))

function material(options: THREE.MeshStandardMaterialParameters) {
  const m = new THREE.MeshStandardMaterial(options)
  ownedMaterials.add(m)
  return m
}
function box(w: number, h: number, d: number, x: number, y: number, z: number, m: THREE.Material) {
  const g = new THREE.BoxGeometry(w, h, d)
  ownedGeometries.add(g)
  const mesh = new THREE.Mesh(g, m)
  mesh.position.set(x, y, z)
  scene!.add(mesh)
  return mesh
}
function cylinder(radius: number, length: number, x: number, y: number, z: number, m: THREE.Material, axis: 'x' | 'y' | 'z' = 'y') {
  const g = new THREE.CylinderGeometry(radius, radius, length, 10)
  ownedGeometries.add(g)
  const mesh = new THREE.Mesh(g, m)
  mesh.position.set(x, y, z)
  if (axis === 'x') mesh.rotation.z = Math.PI / 2
  if (axis === 'z') mesh.rotation.x = Math.PI / 2
  scene!.add(mesh)
}
function cable(points: number[][], m: THREE.Material, radius = .025) {
  const curve = new THREE.CatmullRomCurve3(points.map(p => new THREE.Vector3(p[0]!, p[1]!, p[2]!)))
  const g = new THREE.TubeGeometry(curve, 28, radius, 6, false)
  ownedGeometries.add(g)
  scene!.add(new THREE.Mesh(g, m))
}
function concreteTexture(base: string, seed: number) {
  const c = document.createElement('canvas'); c.width = c.height = 256
  const ctx = c.getContext('2d')!
  ctx.fillStyle = base; ctx.fillRect(0, 0, 256, 256)
  let s = seed >>> 0
  const rand = () => { s = (1664525 * s + 1013904223) >>> 0; return s / 4294967296 }
  for (let i = 0; i < 14000; i++) {
    const value = rand() > .5 ? 255 : 0
    ctx.fillStyle = `rgba(${value},${value},${value},${.012 + rand() * .055})`
    ctx.fillRect(rand() * 256, rand() * 256, 1 + rand() * 5, 1 + rand() * 5)
  }
  for (let i = 0; i < 17; i++) {
    ctx.strokeStyle = `rgba(4,7,6,${.05 + rand() * .14})`
    ctx.lineWidth = .5 + rand() * 2
    ctx.beginPath(); let x = rand() * 256, y = rand() * 256; ctx.moveTo(x, y)
    for (let j = 0; j < 5; j++) { x += rand() * 22 - 11; y += rand() * 25 - 4; ctx.lineTo(x, y) }
    ctx.stroke()
  }
  const texture = new THREE.CanvasTexture(c)
  texture.colorSpace = THREE.SRGBColorSpace
  texture.wrapS = texture.wrapT = THREE.RepeatWrapping
  texture.repeat.set(3, 2)
  texture.anisotropy = 4
  ownedTextures.add(texture)
  return texture
}
function makeRoom() {
  const concrete = material({ map: concreteTexture('#252b2b', 13), color: 0x9a9e9a, roughness: 1 })
  const floor = material({ map: concreteTexture('#242726', 29), color: 0x9b9b93, roughness: .96 })
  const rust = material({ color: 0x2b2723, metalness: .35, roughness: .85 })
  const metal = material({ color: 0x252c2c, metalness: .55, roughness: .65 })
  const black = material({ color: 0x090d0b, roughness: .8 })
  const damp = material({ color: 0x111d19, roughness: 1, transparent: true, opacity: .72 })
  box(15, .25, 17, 0, -2.4, -1, floor)
  box(15, .3, 17, 0, 5.6, -1, concrete)
  box(.3, 8, 17, -7.5, 1.5, -1, concrete)
  box(.3, 8, 17, 7.5, 1.5, -1, concrete)
  box(15, 8, .3, 0, 1.5, -9.5, concrete)
  for (let i = 0; i < 7; i++) box(.025, .035, 17, -6.7 + i * 2.2, -2.26, -1, rust)
  for (let i = 0; i < 4; i++) box(.12, .12, 17, -6.9 + i * .27, 4.5, -1, metal)
  for (let i = 0; i < 12; i++) {
    const patch = box(.5 + (i % 4) * .34, .025, .7 + (i % 3) * .38, -5.5 + (i * 2.11) % 11, -2.255, -7 + (i * 1.47) % 11, damp)
    patch.rotation.y = i * .47
  }
  for (const x of [-6.7, 6.7]) {
    cylinder(.085, 13, x, 3.9, -1, metal, 'z')
    for (let z = -7; z < 7; z += 2.5) cylinder(.12, .11, x, 3.9, z, rust, 'z')
  }
  cylinder(.11, 12, 0, 4.75, -8.95, metal, 'x')
  box(2.5, .18, .5, -1.6, 4.9, -3.4, material({ color: 0x4a504a, metalness: .6, roughness: .65 }))
  box(2.2, .045, .19, -1.6, 4.78, -3.4, material({ color: 0xb9d7bb, emissive: 0xc6e7d0, emissiveIntensity: 1.6 }))
  const cardboard = material({ color: 0x4d3b2b, roughness: 1 })
  for (const [x, y, z] of [[-5.7, -.77, -6.8], [-5.5, .85, -6.8], [-5.8, 2.4, -6.8], [4.6, -1.85, -6.1]]) box(.7, .55, .57, x!, y!, z!, cardboard)
  for (let i = 0; i < 6; i++) box(.07, .2, .08, -6.4 + i * .22, .52, -6.7, black)
  const wire = material({ color: 0x080c0b, roughness: .96 })
  cable([[.5,.2,-4.7],[1,-.2,-4.8],[1.5,-1.7,-4.5],[2,-2.23,-3.7],[2.9,-2.23,-2.8]], wire, .032)
  cable([[-.7,.3,-4.8],[-1.3,-.4,-4.9],[-1.6,-2.2,-4.4],[-2.6,-2.23,-3.4]], wire, .026)
  box(3.8, .16, 1.65, 0, -.15, -3.9, rust)
  for (const x of [-1.65, 1.65]) for (const z of [-4.55, -3.25]) box(.13, 2.2, .13, x, -1.3, z, metal)
  box(1.15, .16, 1.03, 0, -1.22, -1.55, black)
  box(1.15, 1.4, .17, 0, -.47, -.98, black)
  box(.12, 1.2, .12, 0, -1.9, -1.55, metal)
  for (const y of [-1.3, .3, 1.9]) box(2.3, .11, .8, -5.7, y, -6.8, rust)
  for (const x of [-6.7, -4.7]) box(.09, 4.3, .09, x, -.1, -6.8, metal)
  box(1.1, .9, 1.1, 4.8, -1.86, -5.8, rust)
  screenCanvas = document.createElement('canvas')
  screenCanvas.width = 512; screenCanvas.height = 320
  screenTexture = new THREE.CanvasTexture(screenCanvas)
  screenTexture.colorSpace = THREE.SRGBColorSpace
  const lights: Array<[number, number, number, number, number, number]> = [
    [0x29ff75, 5.5, 6, 0, 1.3, -2.95],
    [0xe4edda, 24, 14, -1.6, 4.3, -3.4],
    [0x9ca8a0, 11, 15, 4, 2, 1.5],
    [0xa17b54, 8, 10, -5, 1, -7]
  ]
  for (const [color, intensity, distance, x, y, z] of lights) {
    const light = new THREE.PointLight(color, intensity, distance, 2)
    light.position.set(x, y, z); scene!.add(light)
  }
  scene!.add(new THREE.AmbientLight(0x7c8b88, 1.05))
  const positions = new Float32Array(400 * 3)
  for (let i = 0; i < 400; i++) {
    positions[i * 3] = (Math.random() - .5) * 12
    positions[i * 3 + 1] = (Math.random() - .5) * 7 + 1
    positions[i * 3 + 2] = (Math.random() - .5) * 13 - 1
  }
  const pg = new THREE.BufferGeometry()
  pg.setAttribute('position', new THREE.BufferAttribute(positions, 3))
  ownedGeometries.add(pg)
  const pm = new THREE.PointsMaterial({ color: 0xa4b7a1, size: .025, transparent: true, opacity: .35 })
  ownedMaterials.add(pm)
  scene!.add(new THREE.Points(pg, pm))
}

function wrapLines(ctx: CanvasRenderingContext2D, source: string[], maxWidth: number) {
  const result: string[] = []
  for (const line of source) {
    if (!line) { result.push(''); continue }
    let current = ''
    for (const word of line.split(' ')) {
      const next = current ? `${current} ${word}` : word
      if (ctx.measureText(next).width > maxWidth && current) { result.push(current); current = word }
      else current = next
    }
    result.push(current)
  }
  return result
}
function updateCRT(blink = true) {
  if (!screenCanvas || !screenTexture) return
  const ctx = screenCanvas.getContext('2d')
  if (!ctx) return
  const w = screenCanvas.width, h = screenCanvas.height
  ctx.fillStyle = '#001606'; ctx.fillRect(0, 0, w, h)
  const glow = ctx.createRadialGradient(w / 2, h / 2, 10, w / 2, h / 2, w * .65)
  glow.addColorStop(0, 'rgba(0,80,20,.14)'); glow.addColorStop(1, 'rgba(0,0,0,0)')
  ctx.fillStyle = glow; ctx.fillRect(0, 0, w, h)
  ctx.textBaseline = 'top'; ctx.font = '18px monospace'
  ctx.fillStyle = '#5aff83'; ctx.shadowColor = '#00ff55'; ctx.shadowBlur = 5
  const lineHeight = 25, maxWidth = w - 48
  const visible = wrapLines(ctx, lines.value, maxWidth).slice(-Math.floor((h - 75) / lineHeight))
  visible.forEach((line, index) => ctx.fillText(line, 24, 22 + index * lineHeight))
  const prompt = `nathan@portfolio:~$ ${command.value}`
  ctx.fillStyle = '#a0ffb5'
  ctx.fillText(prompt, 24, h - 40, maxWidth)
  if (blink) {
    const cursorX = Math.min(24 + ctx.measureText(prompt).width + 3, w - 28)
    ctx.fillRect(cursorX, h - 40, 10, 19)
  }
  ctx.shadowBlur = 0
  ctx.fillStyle = 'rgba(0,0,0,.10)'
  for (let y = 0; y < h; y += 4) ctx.fillRect(0, y, w, 1)
  screenTexture.needsUpdate = true
}

const responses: Record<string, string[]> = {
  help: ['AVAILABLE COMMANDS:', 'whoami · projects · skills · contact · clear'],
  whoami: ['NATHAN FASQUEL WILLARD', 'Full-Stack Developer | BUT Informatique (3rd year)', 'Building web applications and interactive experiences.'],
  projects: ['RIFTDATA — League of Legends esports platform', 'SPOT — Fitness tracking PWA', 'ACCESSICODE — AI accessibility auditing', 'SAINT-PIERRE — Homework management platform'],
  skills: ['FRONTEND: Nuxt, Vue, TypeScript', 'BACKEND: Node.js, Express, Prisma', 'DATA: PostgreSQL', 'DEVOPS: Docker, Linux, Nginx'],
  contact: ['GITHUB: github.com/FASQUELWILLARDNathan', 'Open to software development opportunities.']
}
function execute(raw: string) {
  const c = raw.trim().toLowerCase()
  if (c === 'clear') lines.value = []
  else if (c) {
    lines.value.push(`nathan@portfolio:~$ ${raw.trim()}`)
    lines.value.push(...(responses[c] ?? [`Command not found: ${c}. Type help.`]), '')
    // Limit retained history to prevent unbounded growth.
    if (lines.value.length > 150) lines.value.splice(0, lines.value.length - 150)
  }
  command.value = ''
  updateCRT()
  if (terminalActive.value) nextTick(() => inputEl.value?.focus())
}
function runCommand() { execute(command.value) }
function leaveTerminal() { window.scrollTo({ top: 0, behavior: reduced ? 'instant' : 'smooth' }) }
function onKey(e: KeyboardEvent) { if (e.key === 'Escape' && terminalActive.value) leaveTerminal() }

function loadMonitor() {
  new GLTFLoader().load('/models/crt-monitor.glb', gltf => {
    if (destroyed || !scene || !screenTexture) {
      gltf.scene.traverse(obj => {
        if (!(obj instanceof THREE.Mesh)) return
        obj.geometry.dispose()
        const mats = Array.isArray(obj.material) ? obj.material : [obj.material]
        mats.forEach(m => m.dispose())
      })
      return
    }
    const monitor = gltf.scene
    monitor.scale.setScalar(.55)
    monitor.position.set(0, .05, -3.5)
    const screen = monitor.getObjectByName('CRT_Screen')
    if (screen instanceof THREE.Mesh) {
      screen.geometry.computeBoundingBox()
      const bounds = screen.geometry.boundingBox
      if (bounds) {
        const size = new THREE.Vector3(), center = new THREE.Vector3()
        bounds.getSize(size); bounds.getCenter(center)
        const geometry = new THREE.PlaneGeometry(1, 1)
        const displayMaterial = new THREE.MeshBasicMaterial({ map: screenTexture, toneMapped: false, side: THREE.DoubleSide, depthWrite: false })
        ownedGeometries.add(geometry); ownedMaterials.add(displayMaterial)
        const display = new THREE.Mesh(geometry, displayMaterial)
        display.position.set(center.x, center.y, bounds.max.z + .012)
        display.scale.set(size.x * .94, size.y * .94, 1)
        screen.add(display)
      }
    } else console.warn('CRT_Screen introuvable dans /models/crt-monitor.glb')
    scene.add(monitor)
  }, undefined, error => console.error('Chargement CRT impossible :', error))
}
function animate() {
  frame = requestAnimationFrame(animate)
  if (!renderer || !scene || !camera) return
  const dt = Math.min(clock.getDelta(), .05)
  const p = reduced ? 1 : progress.value
  const a = smooth(clamp01(p / .65))
  const b = smooth(clamp01((p - .65) / .35))
  target.copy(from).lerp(seat, a).lerp(close, b)
  camera.position.lerp(target, 1 - Math.exp(-5.3 * dt))
  look.copy(lookFrom).lerp(lookClose, b)
  camera.lookAt(look)
  const fov = 55 - 3 * b
  if (Math.abs(camera.fov - fov) > .001) { camera.fov = fov; camera.updateProjectionMatrix() }
  const now = performance.now()
  if (now - lastCRT >= 250) { updateCRT(Math.floor(now / 500) % 2 === 0); lastCRT = now }
  renderer.render(scene, camera)
}
function onResize() {
  if (!renderer || !camera || !canvasEl.value) return
  const width = canvasEl.value.clientWidth, height = canvasEl.value.clientHeight
  if (!width || !height) return
  camera.aspect = width / height; camera.updateProjectionMatrix()
  renderer.setSize(width, height, false)
}
function init() {
  if (!canvasEl.value) return
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0x050708)
  scene.fog = new THREE.FogExp2(0x090d0c, .012)
  camera = new THREE.PerspectiveCamera(55, 1, .08, 80)
  camera.position.copy(from)
  renderer = new THREE.WebGLRenderer({ canvas: canvasEl.value, antialias: true, powerPreference: 'high-performance' })
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 1.5))
  renderer.outputColorSpace = THREE.SRGBColorSpace
  renderer.toneMapping = THREE.ACESFilmicToneMapping
  renderer.toneMappingExposure = 1.8
  onResize(); makeRoom(); updateCRT(); loadMonitor(); clock.start(); animate()
}
watch(terminalActive, async active => { if (active) { await nextTick(); inputEl.value?.focus() } })
watch(command, () => updateCRT())
onMounted(async () => {
  await nextTick()
  if (!root.value || !canvasEl.value) return
  reduced = window.matchMedia('(prefers-reduced-motion: reduce)').matches
  if (reduced) { progress.value = 1; terminalActive.value = true }
  init()
  gsap.registerPlugin(ScrollTrigger)
  if (!reduced) {
    trigger = ScrollTrigger.create({
      trigger: root.value, start: 'top top', end: 'bottom bottom',
      onUpdate: self => { progress.value = self.progress; terminalActive.value = self.progress > .985 }
    })
    progress.value = trigger.progress
    terminalActive.value = trigger.progress > .985
  }
  window.addEventListener('resize', onResize)
  window.addEventListener('keydown', onKey)
})
onBeforeUnmount(() => {
  destroyed = true
  window.removeEventListener('resize', onResize)
  window.removeEventListener('keydown', onKey)
  trigger?.kill()
  cancelAnimationFrame(frame)
  if (scene) scene.traverse(obj => {
    if (!(obj instanceof THREE.Mesh)) return
    ownedGeometries.add(obj.geometry)
    const mats = Array.isArray(obj.material) ? obj.material : [obj.material]
    mats.forEach(m => {
      ownedMaterials.add(m)
      if ('map' in m && m.map instanceof THREE.Texture) ownedTextures.add(m.map)
    })
  })
  ownedGeometries.forEach(g => g.dispose())
  ownedMaterials.forEach(m => m.dispose())
  ownedTextures.forEach(t => t.dispose())
  screenTexture?.dispose()
  renderer?.dispose()
})
</script>

<style scoped>
:global(body) { margin: 0; background: #050708; font-family: ui-monospace, SFMono-Regular, Consolas, monospace; }
.basement { position: relative; height: 420vh; color: #c8e9d0; }
.world { position: fixed; inset: 0; width: 100%; height: 100%; z-index: 0; display: block; }
.hud { position: fixed; top: 0; left: 0; right: 0; display: flex; justify-content: space-between; padding: 24px 4vw; z-index: 3; font-size: 11px; letter-spacing: .18em; color: #80ac8a; pointer-events: none; }
.intro { position: absolute; top: 0; width: 100%; height: 100vh; z-index: 2; display: flex; flex-direction: column; align-items: center; justify-content: center; text-align: center; pointer-events: none; text-shadow: 0 2px 24px #000; }
h1 { font-size: clamp(3rem, 10vw, 9rem); letter-spacing: -.09em; margin: 0; font-weight: 900; color: #e2f3e4; }
h1 span { color: #6cff8e; animation: blink 1s steps(2) infinite; }
.intro p { letter-spacing: .45em; font-size: 12px; }
.intro small { margin-top: 16vh; letter-spacing: .25em; color: #8da999; }
.hint { position: fixed; bottom: 12%; width: 100%; text-align: center; z-index: 2; letter-spacing: .3em; font-size: 12px; color: #72ffa1; pointer-events: none; }
.crt-controls { position: fixed; bottom: 32px; left: 50%; transform: translateX(-50%) translateY(20px); width: min(92vw, 620px); box-sizing: border-box; z-index: 15; background: rgba(2, 12, 5, .94); border: 1px solid #286c3a; box-shadow: 0 0 30px #00ff3318; padding: 14px; opacity: 0; visibility: hidden; pointer-events: none; transition: opacity .3s, transform .3s, visibility .3s; }
.crt-controls.active { opacity: 1; visibility: visible; pointer-events: auto; transform: translateX(-50%) translateY(0); }
.crt-controls-label { display: flex; justify-content: space-between; align-items: center; gap: 10px; margin-bottom: 12px; color: #72ff99; font-size: 11px; letter-spacing: .1em; }
.crt-controls button { background: transparent; border: 1px solid #245d35; color: #72ff99; font: inherit; padding: 7px 10px; cursor: pointer; }
.crt-controls button:hover, .crt-controls button:focus-visible { background: #124b25; }
.crt-input-row { display: flex; align-items: center; gap: 12px; color: #72ff99; }
.crt-input-row input { flex: 1; min-width: 0; background: transparent; border: 0; outline: none; color: #a0ffb5; font: 14px monospace; caret-color: #72ff99; }
.crt-shortcuts { display: flex; gap: 6px; flex-wrap: wrap; margin-top: 12px; }
.crt-shortcuts button { font-size: 10px; }
.progress-track { position: fixed; right: 0; top: 0; width: 3px; height: 100vh; background: #112317; z-index: 20; }
.progress-track > div { height: 100%; background: #66ff8a; transform-origin: top; }
@keyframes blink { 50% { opacity: 0; } }
@media (max-width: 650px) { .hud { font-size: 9px; padding: 15px; } .intro p { letter-spacing: .15em; } .crt-controls { bottom: 12px; padding: 10px; } .crt-controls-label { font-size: 9px; } }
@media (prefers-reduced-motion: reduce) { h1 span { animation: none; } .crt-controls { transition: none; } }
</style>
