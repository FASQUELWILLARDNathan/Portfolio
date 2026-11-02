
<template>
  <div ref="experienceElement" class="experience">
    <canvas ref="canvasElement" class="scene" aria-hidden="true" />

    <div class="vignette"></div>
    <div class="grain"></div>

    <header class="topbar">
      <a href="#home" class="brand">N<span>.</span></a>
      <span class="top-label">INTERACTIVE PORTFOLIO — 2026</span>
      <a href="#contact" class="contact-link">CONTACT ↗</a>
    </header>

    <main class="overlay">
      <section id="home" class="panel hero">
        <div class="hero-content">
          <div class="eyebrow">
            <span class="dot"></span>
            DEVELOPER / CREATIVE TECHNOLOGY
          </div>

          <h1>
            NATHAN
            <span>FASQUEL WILLARD</span>
          </h1>

          <p class="hero-description">
            Développeur Full-Stack.
            Je transforme des idées en expériences digitales.
          </p>

          <div class="hero-bottom">
            <span>SCROLL TO EXPLORE</span>
            <div class="scroll-line"></div>
            <span>01 — 04</span>
          </div>
        </div>
      </section>

      <section id="about" class="panel about">
        <div class="section-content">
          <div class="section-index">01 / ABOUT ME</div>
          <h2>
            CODE IS<br />
            <em>MY CANVAS.</em>
          </h2>
          <p>
            Étudiant en troisième année de BUT Informatique,
            spécialisé dans la réalisation d'applications.
            J'aime concevoir des solutions performantes,
            modernes et intuitives.
          </p>
          <div class="tags">
            <span>NUXT</span>
            <span>TYPESCRIPT</span>
            <span>NODE.JS</span>
            <span>POSTGRESQL</span>
          </div>
        </div>
      </section>

      <section id="projects" class="panel projects">
        <div class="section-content">
          <div class="section-index">02 / SELECTED WORK</div>
          <h2>
            DIGITAL<br />
            <em>CREATIONS.</em>
          </h2>

          <div class="project-list">
            <a href="#contact" class="project">
              <span class="project-number">01</span>
              <div>
                <h3>RIFTLY</h3>
                <p>League of Legends Esports Platform</p>
              </div>
              <span class="arrow">↗</span>
            </a>

            <a href="https://spot.neyznn.fr"
               target="_blank"
               rel="noopener noreferrer"
               class="project">
              <span class="project-number">02</span>
              <div>
                <h3>SPOT</h3>
                <p>Fitness Tracking Application</p>
              </div>
              <span class="arrow">↗</span>
            </a>

            <a href="#contact" class="project">
              <span class="project-number">03</span>
              <div>
                <h3>ACCESSICODE</h3>
                <p>AI-Powered Accessibility Audit</p>
              </div>
              <span class="arrow">↗</span>
            </a>
          </div>
        </div>
      </section>

      <section id="contact" class="panel contact">
        <div class="section-content">
          <div class="section-index">03 / CONTACT</div>
          <h2>
            LET'S MAKE<br />
            <em>IT REAL.</em>
          </h2>
          <p>
            Un projet, une collaboration ou une opportunité ?
            Construisons quelque chose ensemble.
          </p>
          <a
            class="cta"
            href="https://github.com/FASQUELWILLARDNathan"
            target="_blank"
            rel="noopener noreferrer"
          >
            EXPLORE MY GITHUB <span>↗</span>
          </a>
        </div>
      </section>
    </main>

    <div class="side-info">
      <span>FRANCE / FULL-STACK DEVELOPER</span>
      <span>© 2026</span>
    </div>

    <div class="progress">
      <div ref="progressElement" class="progress-fill"></div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, nextTick,onMounted, onBeforeUnmount } from 'vue'
import * as THREE from 'three'
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'

const canvasElement = ref<HTMLCanvasElement | null>(null)
const experienceElement = ref<HTMLElement | null>(null)
const progressElement = ref<HTMLElement | null>(null)

let renderer: THREE.WebGLRenderer | null = null
let scene: THREE.Scene
let camera: THREE.PerspectiveCamera
let tunnel: THREE.Group
let particles: THREE.Points
let scrollTrigger: ScrollTrigger | null = null
let frameId = 0

const timer = new THREE.Timer()
const mouse = new THREE.Vector2()
const smoothMouse = new THREE.Vector2()

let scrollProgress = 0
let reducedMotion = false

const TUNNEL_LENGTH = 130
const FRAME_COUNT = 34

function createFrame(z: number, index: number) {
  const group = new THREE.Group()

  const frameMaterial = new THREE.MeshStandardMaterial({
    color: index % 4 === 0 ? 0x528bff : 0x152a4c,
    emissive: index % 4 === 0 ? 0x2563eb : 0x0b1835,
    emissiveIntensity: index % 4 === 0 ? 3 : 1.2,
    metalness: 0.7,
    roughness: 0.25
  })

  const width = 12
  const height = 8
  const thickness = index % 4 === 0 ? 0.09 : 0.055

  const horizontal = new THREE.BoxGeometry(
    width, thickness, thickness
  )
  const vertical = new THREE.BoxGeometry(
    thickness, height, thickness
  )

  const top = new THREE.Mesh(horizontal, frameMaterial)
  top.position.y = height / 2

  const bottom = new THREE.Mesh(horizontal, frameMaterial)
  bottom.position.y = -height / 2

  const left = new THREE.Mesh(vertical, frameMaterial)
  left.position.x = -width / 2

  const right = new THREE.Mesh(vertical, frameMaterial)
  right.position.x = width / 2

  group.add(top, bottom, left, right)

  group.position.z = z

  // Légère variation architecturale
  group.rotation.z = Math.sin(index * 0.35) * 0.035

  return group
}

function createTunnel() {
  tunnel = new THREE.Group()

  for (let i = 0; i < FRAME_COUNT; i++) {
    const z = -i * (TUNNEL_LENGTH / FRAME_COUNT)
    tunnel.add(createFrame(z, i))
  }

  // Lignes lumineuses longitudinales
  const railMaterial = new THREE.MeshBasicMaterial({
    color: 0x1745a0
  })

  for (const x of [-6, 6]) {
    for (const y of [-4, 4]) {
      const rail = new THREE.Mesh(
        new THREE.BoxGeometry(0.025, 0.025, TUNNEL_LENGTH),
        railMaterial
      )
      rail.position.set(x, y, -TUNNEL_LENGTH / 2)
      tunnel.add(rail)
    }
  }

  scene.add(tunnel)
}

function createParticles() {
  const count = 700
  const positions = new Float32Array(count * 3)

  for (let i = 0; i < count; i++) {
    positions[i * 3] = (Math.random() - 0.5) * 22
    positions[i * 3 + 1] = (Math.random() - 0.5) * 16
    positions[i * 3 + 2] = -Math.random() * TUNNEL_LENGTH
  }

  const geometry = new THREE.BufferGeometry()
  geometry.setAttribute(
    'position',
    new THREE.BufferAttribute(positions, 3)
  )

  const material = new THREE.PointsMaterial({
    color: 0x91b8ff,
    size: 0.035,
    transparent: true,
    opacity: 0.6,
    depthWrite: false
  })

  particles = new THREE.Points(geometry, material)
  scene.add(particles)
}

function createEnvironment() {
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0x030509)
  scene.fog = new THREE.FogExp2(0x030509, 0.012)

  camera = new THREE.PerspectiveCamera(
    65,
    window.innerWidth / window.innerHeight,
    0.1,
    200
  )

  camera.position.set(0, 0, 9)

  renderer = new THREE.WebGLRenderer({
    canvas: canvasElement.value!,
    antialias: true,
    powerPreference: 'high-performance'
  })

  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.setPixelRatio(
    Math.min(window.devicePixelRatio, 1.75)
  )
  renderer.outputColorSpace = THREE.SRGBColorSpace
  renderer.toneMapping = THREE.ACESFilmicToneMapping

  const ambient = new THREE.AmbientLight(0x6386bb, 1.5)
  scene.add(ambient)

  const blueLight = new THREE.PointLight(
    0x2563eb, 120, 35
  )
  blueLight.position.set(0, 3, 3)
  scene.add(blueLight)

  const purpleLight = new THREE.PointLight(
    0x7040ff, 60, 25
  )
  purpleLight.position.set(-4, -2, -15)
  scene.add(purpleLight)

  createTunnel()
  createParticles()
}

function setupScroll() {
  if (reducedMotion) return

  gsap.registerPlugin(ScrollTrigger)

  scrollTrigger = ScrollTrigger.create({
    trigger: experienceElement.value!,
    start: 'top top',
    end: 'bottom bottom',
    onUpdate: self => {
      scrollProgress = self.progress

      if (progressElement.value) {
        progressElement.value.style.transform =
          `scaleY(${self.progress})`
      }
    }
  })
}

function onMouseMove(event: MouseEvent) {
  mouse.x = (event.clientX / window.innerWidth - 0.5) * 2
  mouse.y = (event.clientY / window.innerHeight - 0.5) * 2
}

function animate() {
  frameId = requestAnimationFrame(animate)

  timer.update()
  const elapsed = timer.getElapsed()

  const targetZ = 9 - scrollProgress * 125

  camera.position.z +=
    (targetZ - camera.position.z) * 0.045

  if (!reducedMotion) {
    smoothMouse.lerp(mouse, 0.025)

    camera.position.x +=
      (smoothMouse.x * 0.55 - camera.position.x) * 0.025

    camera.position.y +=
      (-smoothMouse.y * 0.3 - camera.position.y) * 0.025

    camera.rotation.z =
      Math.sin(elapsed * 0.2) * 0.007
  }

  // La caméra regarde toujours vers l'avant du tunnel
  camera.rotation.y = 0

  renderer?.render(scene, camera)
}

function onResize() {
  if (!renderer) return

  camera.aspect = window.innerWidth / window.innerHeight
  camera.updateProjectionMatrix()
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.setPixelRatio(
    Math.min(window.devicePixelRatio, 1.75)
  )
  ScrollTrigger.refresh()
}

onMounted(async () => {
  await nextTick()

  console.log('Canvas:', canvasElement.value)
  console.log('Experience:', experienceElement.value)

  if (!canvasElement.value || !experienceElement.value) {
    console.error('Impossible d’initialiser la scène 3D')
    return
  }

  reducedMotion = window.matchMedia(
    '(prefers-reduced-motion: reduce)'
  ).matches

  createEnvironment()
  setupScroll()
  animate()

  window.addEventListener('resize', onResize)
  window.addEventListener('mousemove', onMouseMove)

  console.log('SCÈNE 3D INITIALISÉE')
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', onResize)
  window.removeEventListener('mousemove', onMouseMove)

  cancelAnimationFrame(frameId)
  scrollTrigger?.kill()

  scene?.traverse(object => {
    if (object instanceof THREE.Mesh ||
        object instanceof THREE.Points) {
      object.geometry.dispose()

      const materials = Array.isArray(object.material)
        ? object.material
        : [object.material]

      materials.forEach(material => material.dispose())
    }
  })

  renderer?.dispose()
})
</script>

<style scoped>
:global(*) {
  box-sizing: border-box;
}

:global(html) {
  scroll-behavior: auto;
}

:global(body) {
  margin: 0;
  background: #030509;
  font-family: Inter, Arial, sans-serif;
}

.experience {
  position: relative;
  min-height: 400vh;
  background: transparent;
  color: #fff;
  overflow: visible;
  isolation: isolate;
}

.scene {
  position: fixed;
  inset: 0;
  display: block;
  width: 100vw;
  height: 100vh;
  z-index: 0;
  pointer-events: none;
}

.vignette {
  display: none;
}

.grain {
  position: fixed;
  inset: 0;
  z-index: 2;
  pointer-events: none;
  opacity: 0.035;
  background-image:
    url("data:image/svg+xml,%3Csvg viewBox='0 0 180 180' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='.9' numOctaves='3' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23n)' opacity='.5'/%3E%3C/svg%3E");
}

.topbar {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: 20;
  padding: 30px 5vw;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.brand {
  font-size: 30px;
  font-weight: 900;
  letter-spacing: -3px;
  color: white;
  text-decoration: none;
}

.brand span {
  color: #3b82f6;
}

.top-label,
.contact-link {
  font-size: 10px;
  letter-spacing: 0.22em;
  font-weight: 600;
}

.top-label {
  color: #777e90;
}

.contact-link {
  color: white;
  text-decoration: none;
}

.overlay {
  position: relative;
  z-index: 3;
}

.panel {
  min-height: 100vh;
  display: flex;
  align-items: center;
  padding: 10vw;
}

.hero {
  justify-content: center;
  text-align: center;
}

.hero-content {
  width: 100%;
}

.eyebrow,
.section-index {
  color: #60a5fa;
  font-size: 11px;
  font-weight: 700;
  letter-spacing: 0.3em;
  margin-bottom: 30px;
}

.dot {
  display: inline-block;
  width: 6px;
  height: 6px;
  background: #60a5fa;
  border-radius: 50%;
  margin-right: 12px;
  box-shadow: 0 0 15px #3b82f6;
}

h1,
h2,
h3 {
  margin: 0;
  font-weight: 800;
}

h1 {
  font-size: clamp(4rem, 11vw, 11rem);
  line-height: 0.85;
  letter-spacing: -0.09em;
}

h1 span {
  display: block;
  font-size: 0.42em;
  letter-spacing: -0.055em;
  margin-top: 20px;
  color: #a8b5d0;
}

.hero-description {
  margin: 45px auto 0;
  max-width: 550px;
  color: #aab3c4;
  font-size: 16px;
  line-height: 1.8;
}

.hero-bottom {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 25px;
  margin-top: 90px;
  font-size: 10px;
  letter-spacing: 0.25em;
  color: #8892a7;
}

.scroll-line {
  width: 70px;
  height: 1px;
  background: #3b82f6;
}

.section-content {
  width: 100%;
  max-width: 1000px;
}

h2 {
  font-size: clamp(3.5rem, 9vw, 9rem);
  line-height: 0.9;
  letter-spacing: -0.075em;
}

h2 em {
  font-style: normal;
  color: #638bd3;
}

.section-content > p {
  max-width: 510px;
  margin-top: 35px;
  color: #aab3c4;
  font-size: 17px;
  line-height: 1.9;
}

.tags {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  margin-top: 40px;
}

.tags span {
  border: 1px solid #34425c;
  padding: 12px 18px;
  font-size: 10px;
  letter-spacing: 0.15em;
  color: #b5c5e2;
}

.projects {
  justify-content: flex-end;
}

.project-list {
  margin-top: 50px;
  max-width: 850px;
}

.project {
  display: flex;
  align-items: center;
  gap: 25px;
  padding: 25px 0;
  border-bottom: 1px solid #34405a;
  color: white;
  text-decoration: none;
  transition: padding 0.3s ease;
}

.project:hover {
  padding-left: 15px;
}

.project-number {
  font-size: 11px;
  color: #6d88bb;
}

.project h3 {
  font-size: clamp(24px, 3vw, 42px);
  letter-spacing: -0.04em;
}

.project p {
  margin: 7px 0 0;
  color: #8d9bb1;
  font-size: 12px;
}

.arrow {
  margin-left: auto;
  font-size: 25px;
  color: #60a5fa;
}

.cta {
  display: inline-flex;
  align-items: center;
  gap: 40px;
  margin-top: 40px;
  padding: 20px 25px;
  border: 1px solid #5a7fc3;
  color: white;
  text-decoration: none;
  font-size: 11px;
  letter-spacing: 0.2em;
  transition: background 0.3s;
}

.cta:hover {
  background: #1d3b70;
}

.cta span {
  font-size: 20px;
}

.side-info {
  position: fixed;
  bottom: 30px;
  left: 5vw;
  right: 5vw;
  z-index: 10;
  display: flex;
  justify-content: space-between;
  font-size: 9px;
  letter-spacing: 0.2em;
  color: #78849c;
  pointer-events: none;
}

.progress {
  position: fixed;
  right: 0;
  top: 0;
  bottom: 0;
  width: 3px;
  z-index: 30;
  background: #101726;
}

.progress-fill {
  width: 100%;
  height: 100%;
  background: #3b82f6;
  transform: scaleY(0);
  transform-origin: top;
}

@media (max-width: 768px) {
  .topbar {
    padding: 20px;
  }

  .top-label {
    display: none;
  }

  .panel {
    padding: 25px;
  }

  h1 {
    font-size: clamp(3rem, 13vw, 6rem);
  }

  h2 {
    font-size: clamp(3rem, 12vw, 5rem);
  }

  .side-info {
    display: none;
  }

  .hero-description {
    font-size: 14px;
  }
}

@media (prefers-reduced-motion: reduce) {
  .project,
  .cta {
    transition: none;
  }
}
</style>
