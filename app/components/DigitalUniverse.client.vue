<template>
  <div ref="root" class="universe">
    <canvas ref="canvas" class="universe-canvas" aria-hidden="true" />
    <div class="shade" aria-hidden="true"></div>
    <header class="nav"><a href="#intro" class="logo">NF<span> / </span>UNIVERSE</a><span class="nav-center">PORTFOLIO / 2026</span><a href="#contact">CONTACT ↗</a></header>
    <main>
      <section id="intro" class="chapter intro">
        <div class="copy hero"><p class="kicker">[ SYSTEM ONLINE ] &nbsp; FULL-STACK DEVELOPER</p><h1>BEYOND<br><span>THE CODE.</span></h1><p>Je suis Nathan Fasquel Willard. Je crée des expériences web, des applications et des univers numériques.</p><span class="scroll-label">SCROLL TO ENTER THE UNIVERSE ↓</span></div>
      </section>
      <section id="about" class="chapter about"><div class="copy"><p class="kicker">01 / ORIGIN</p><h2>DES IDÉES.<br><span>DU CODE.</span></h2><p>Étudiant en BUT Informatique, parcours réalisation d'applications. J'aime construire des produits complets, du frontend à l'infrastructure.</p><div class="chips"><span>NUXT</span><span>VUE</span><span>NODE.JS</span><span>TYPESCRIPT</span><span>POSTGRESQL</span></div></div></section>
      <section id="work" class="chapter work"><div class="copy"><p class="kicker">02 / SELECTED PROJECTS</p><h2>MY DIGITAL<br><span>WORLD.</span></h2><div class="projects"><article v-for="(project,i) in projects" :key="project.name" class="project"><span class="num">0{{ i+1 }}</span><div><h3>{{project.name}}</h3><p>{{project.description}}</p></div><a v-if="project.url" :href="project.url" target="_blank" rel="noopener noreferrer" :aria-label="`Voir ${project.name}`">↗</a><span v-else class="soon">À DÉCOUVRIR</span></article></div></div></section>
      <section id="contact" class="chapter contact"><div class="copy"><p class="kicker">03 / NEXT CONNECTION</p><h2>LET'S<br><span>CONNECT.</span></h2><p>Une opportunité, un stage ou un projet ? Parlons-en.</p><div class="actions"><a href="https://github.com/FASQUELWILLARDNathan" target="_blank" rel="noopener noreferrer">GITHUB ↗</a><a v-if="email" :href="`mailto:${email}`">EMAIL ↗</a></div></div></section>
    </main>
    <div class="hud" aria-hidden="true"><span>NF // DIGITAL UNIVERSE</span><span>FR / 2026</span></div>
    <div class="progress" aria-hidden="true"><div ref="progressBar"></div></div>
  </div>
</template>

<script setup lang="ts">
import { ref, nextTick, onMounted, onBeforeUnmount } from 'vue'
import * as THREE from 'three'
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'

const projects = [
  { name: 'RIFTLY', description: 'Plateforme esport League of Legends', url: '' },
  { name: 'SPOT', description: 'Application de suivi de musculation', url: 'https://spot.neyznn.fr' },
  { name: 'ACCESSICODE', description: 'Audit d’accessibilité assisté par IA', url: '' }
]
const email = '' // Renseigner ton adresse publique si souhaité
const root = ref<HTMLElement|null>(null)
const canvas = ref<HTMLCanvasElement|null>(null)
const progressBar = ref<HTMLElement|null>(null)
let renderer: THREE.WebGLRenderer | undefined
let scene: THREE.Scene | undefined
let camera: THREE.PerspectiveCamera | undefined
let trigger: ScrollTrigger | undefined
let raf = 0
let progress = 0
let reduced = false
let active = true
let observer: IntersectionObserver | undefined
const mouse = new THREE.Vector2()
const smoothMouse = new THREE.Vector2()
const timer = new THREE.Timer()
const objects: THREE.Object3D[] = []
const disposeMaterials = new Set<THREE.Material>()
const disposeGeometries = new Set<THREE.BufferGeometry>()

function material(color: number, emissive = color, intensity = 0.5) {
  const m = new THREE.MeshStandardMaterial({color, metalness:0.65, roughness:0.24, emissive, emissiveIntensity:intensity, side:THREE.DoubleSide})
  disposeMaterials.add(m)
  return m
}
function mesh(geometry: THREE.BufferGeometry, mat: THREE.Material, parent: THREE.Object3D, x=0,y=0,z=0) {
  disposeGeometries.add(geometry)
  const o = new THREE.Mesh(geometry, mat)
  o.position.set(x,y,z)
  parent.add(o)
  return o
}
function ring(parent: THREE.Object3D, radius:number, z:number, color:number, tilt=0) {
  const r = mesh(new THREE.TorusGeometry(radius, 0.035, 8, 128), material(color,color,1.7),parent,0,0,z)
  r.rotation.set(0.16,tilt,0)
  return r
}
function addStars() {
  const count = window.innerWidth < 700 ? 350 : 850
  const data = new Float32Array(count*3)
  for(let i=0;i<count;i++) {
    data[i*3]=(Math.random()-.5)*65
    data[i*3+1]=(Math.random()-.5)*40
    data[i*3+2]=10-Math.random()*160
  }
  const g = new THREE.BufferGeometry()
  g.setAttribute('position',new THREE.BufferAttribute(data,3))
  disposeGeometries.add(g)
  const m = new THREE.PointsMaterial({color:0x9e9cff,size:0.085,transparent:true,opacity:.65,depthWrite:false})
  disposeMaterials.add(m)
  scene!.add(new THREE.Points(g,m))
}
function createN() {
  const group = new THREE.Group()
  group.position.set(2.6,0,-8)
  const metal = material(0x151b3f,0x7c3aed,0.45)
  const glow = material(0x57efff,0x06d6ff,2.8)
  const left = mesh(new THREE.BoxGeometry(.7,5,.7),metal,group,-1.5,0,0)
  const right = mesh(new THREE.BoxGeometry(.7,5,.7),metal,group,1.5,0,0)
  const diag = mesh(new THREE.BoxGeometry(.7,5.85,.7),metal,group,0,0,0)
  diag.rotation.z=Math.atan2(-3,5)
  for (const obj of [left,right,diag]) {
    const edge = new THREE.LineSegments(new THREE.EdgesGeometry((obj as THREE.Mesh).geometry),new THREE.LineBasicMaterial({color:0x65eaff}))
    disposeGeometries.add(edge.geometry)
    disposeMaterials.add(edge.material as THREE.Material)
    obj.add(edge)
  }
  ring(group,4.7,-1.4,0x8b5cf6,.4)
  ring(group,5.5,-2.8,0x06b6d4,-.3)
  mesh(new THREE.IcosahedronGeometry(.28,1),glow,group,0,-3.8,0)
  scene!.add(group);objects.push(group)
}
function createCore() {
  const group=new THREE.Group()
  group.position.set(-2.6,0,-47)
  const core=mesh(new THREE.IcosahedronGeometry(2.1,2),material(0x271354,0x8b5cf6,1.1),group)
  core.rotation.z=.2
  for(let i=0;i<5;i++) {
    const r=ring(group,3.2+i*.45,0,i%2?0x06b6d4:0xa855f7,i*.3)
    r.rotation.x=i*.35
  }
  for(let i=0;i<9;i++) {
    const angle=i*Math.PI*2/9
    mesh(new THREE.OctahedronGeometry(.16),material(0x66eeff,0x06b6d4,2),group,Math.cos(angle)*5,Math.sin(angle)*3,Math.sin(angle)*2)
  }
  scene!.add(group);objects.push(group)
}
function createGallery() {
  const group=new THREE.Group()
  group.position.set(0,0,-89)
  const panelMat=material(0x080f2d,0x161c59,.45)
  const borderMat=material(0x22d3ee,0x06b6d4,1.8)
  for(let i=0;i<3;i++) {
    const x=(i-1)*5.2
    const z= -i*1.2
    mesh(new THREE.BoxGeometry(4.1,5.3,.16),panelMat,group,x,0,z)
    for(const y of [-2.65,2.65]) mesh(new THREE.BoxGeometry(4.2,.035,.2),borderMat,group,x,y,z+.1)
    for(const side of [-2.05,2.05]) mesh(new THREE.BoxGeometry(.035,5.3,.2),borderMat,group,x+side,0,z+.1)
    mesh(new THREE.IcosahedronGeometry(.8,i===1?2:1),material(i===0?0x8b5cf6:0x06b6d4,i===0?0x8b5cf6:0x06b6d4,1.4),group,x,0,z+1)
  }
  scene!.add(group);objects.push(group)
}
function createPortal() {
  const group=new THREE.Group()
  group.position.set(2.3,0,-130)
  for(let i=0;i<7;i++) {
    const r=ring(group,3+i*.5,-i*.6,i%2?0x8b5cf6:0x06b6d4,i*.16)
    r.rotation.x=.12+i*.07
  }
  const orb=mesh(new THREE.IcosahedronGeometry(1.1,2),material(0x050a29,0x7c3aed,1.8),group,0,0,-3)
  orb.rotation.z=.4
  scene!.add(group);objects.push(group)
}
function init() {
  if(!canvas.value) return
  scene=new THREE.Scene()
  scene.background=new THREE.Color(0x050513)
  scene.fog=new THREE.FogExp2(0x050513,.008)
  camera=new THREE.PerspectiveCamera(55,innerWidth/innerHeight,.1,250)
  camera.position.set(0,0,9)
  renderer=new THREE.WebGLRenderer({canvas:canvas.value,antialias:true,powerPreference:'high-performance'})
  renderer.setPixelRatio(Math.min(devicePixelRatio,innerWidth<700?1.25:1.75))
  renderer.setSize(innerWidth,innerHeight)
  renderer.outputColorSpace=THREE.SRGBColorSpace
  renderer.toneMapping=THREE.ACESFilmicToneMapping
  scene.add(new THREE.AmbientLight(0x8f9dff,2))
  const cyan=new THREE.PointLight(0x00ddff,110,28);cyan.position.set(4,4,-6);scene.add(cyan)
  const violet=new THREE.PointLight(0x9333ea,140,35);violet.position.set(-4,-3,-48);scene.add(violet)
  addStars();createN();createCore();createGallery();createPortal()
  gsap.registerPlugin(ScrollTrigger)
  if(!reduced && root.value) {
    trigger=ScrollTrigger.create({trigger:root.value,start:'top top',end:'bottom bottom',onUpdate:self=>{
      progress=self.progress
      if(progressBar.value) progressBar.value.style.transform=`scaleY(${progress})`
    }})
    progress=trigger.progress
  }
  draw()
}
function draw() {
  raf=requestAnimationFrame(draw)
  if(!active || !renderer || !camera || !scene) return
  timer.update()
  const t=timer.getElapsed()
  const stops=[9,-30,-73,-114]
  const s=Math.min(2.999,progress*3)
  const idx=Math.floor(s)
  const local=s-idx
  const eased=local*local*(3-2*local)
  const targetZ=THREE.MathUtils.lerp(stops[idx],stops[idx+1],eased)
  camera.position.z+= (targetZ-camera.position.z)*.055
  smoothMouse.lerp(mouse,.035)
  const drift=reduced?0:smoothMouse.x*.38
  const xStops=[0,0,0,0]
  const targetX=THREE.MathUtils.lerp(xStops[idx],xStops[idx+1],eased)+drift
  camera.position.x+=(targetX-camera.position.x)*.045
  camera.position.y+=((reduced?0:-smoothMouse.y*.22)-camera.position.y)*.045
  camera.rotation.z=reduced?0:Math.sin(t*.35)*.004
  if(!reduced) {
    objects[0].rotation.y=Math.sin(t*.28)*.13
    objects[1].rotation.y=t*.06
    objects[2].rotation.y=Math.sin(t*.22)*.035
    objects[3].rotation.z=t*.045
  }
  renderer.render(scene,camera)
}
function resize() {
  if(!camera || !renderer)return
  camera.aspect=innerWidth/innerHeight
  camera.updateProjectionMatrix()
  renderer.setPixelRatio(Math.min(devicePixelRatio,innerWidth<700?1.25:1.75))
  renderer.setSize(innerWidth,innerHeight)
  ScrollTrigger.refresh()
}
function move(e:MouseEvent) {mouse.set((e.clientX/innerWidth-.5)*2,(e.clientY/innerHeight-.5)*2)}
function visibility() {active=!document.hidden}
onMounted(async()=>{
  await nextTick()
  if(!canvas.value || !root.value){console.error('Références canvas/root absentes');return}
  reduced=matchMedia('(prefers-reduced-motion: reduce)').matches
  init()
  addEventListener('resize',resize)
  addEventListener('mousemove',move)
  document.addEventListener('visibilitychange',visibility)
})
onBeforeUnmount(()=>{
  removeEventListener('resize',resize)
  removeEventListener('mousemove',move)
  document.removeEventListener('visibilitychange',visibility)
  cancelAnimationFrame(raf)
  trigger?.kill()
  disposeGeometries.forEach(g=>g.dispose())
  disposeMaterials.forEach(m=>m.dispose())
  renderer?.dispose()
})
</script>

<style scoped>
:global(html){scroll-behavior:auto}
:global(body){margin:0;background:#050513;color:#fff;font-family:Inter,system-ui,Arial,sans-serif}
:global(*){box-sizing:border-box}
.universe{position:relative;isolation:isolate;color:white;min-height:400vh}
.universe-canvas{position:fixed;inset:0;width:100vw;height:100vh;z-index:0;pointer-events:none}
.shade{position:fixed;inset:0;z-index:1;pointer-events:none;background:linear-gradient(90deg,rgba(4,5,20,.7),transparent 75%),linear-gradient(0deg,rgba(4,5,20,.3),transparent 30%)}
.nav{position:fixed;inset:0 0 auto;z-index:20;padding:28px 5vw;display:flex;justify-content:space-between;align-items:center;font-size:11px;letter-spacing:.19em;font-weight:700}
.nav a{color:white;text-decoration:none}.logo{font-size:16px}.logo span{color:#22d3ee}.nav-center{color:#9ca3c0}
main{position:relative;z-index:3}.chapter{height:100vh;min-height:600px;display:flex;align-items:center;padding:10vw}.copy{max-width:750px}.intro{justify-content:flex-start}.about{justify-content:flex-end}.work{justify-content:flex-start}.contact{justify-content:flex-end}
.kicker{font-size:11px;letter-spacing:.3em;color:#67e8f9;font-weight:700;margin-bottom:30px}
h1,h2,h3{margin:0;font-weight:850}h1{font-size:clamp(3.4rem,9vw,9rem);letter-spacing:-.085em;line-height:.88}h2{font-size:clamp(3rem,7.5vw,7.5rem);letter-spacing:-.075em;line-height:.93}h1 span,h2 span{color:#a78bfa;text-shadow:0 0 38px rgba(124,58,237,.3)}.copy>p:not(.kicker){color:#b8bdd9;max-width:480px;line-height:1.8;font-size:clamp(14px,1.4vw,18px);margin-top:32px}.scroll-label{display:inline-block;margin-top:65px;color:#67e8f9;font-size:10px;letter-spacing:.24em}
.chips{display:flex;gap:8px;flex-wrap:wrap;margin-top:30px}.chips span{border:1px solid #6d54a0;padding:10px 13px;font-size:10px;letter-spacing:.1em;color:#e2d9ff;background:rgba(10,9,35,.55)}
.projects{margin-top:30px;max-width:640px}.project{display:flex;gap:18px;align-items:center;border-bottom:1px solid #474064;padding:18px 0;background:rgba(4,5,20,.22)}.project .num{font-size:11px;color:#67e8f9}.project h3{font-size:clamp(19px,2.5vw,32px);letter-spacing:-.035em}.project p{font-size:12px;color:#b8bdd9;margin:6px 0 0}.project a{color:#67e8f9;text-decoration:none;font-size:23px;margin-left:auto}.soon{margin-left:auto;color:#8d8aa7;font-size:9px;letter-spacing:.12em}
.actions{display:flex;gap:15px;flex-wrap:wrap;margin-top:35px}.actions a{color:white;text-decoration:none;border:1px solid #8b5cf6;padding:16px 25px;letter-spacing:.13em;font-size:11px;background:rgba(76,29,149,.24)}.actions a:hover{background:rgba(76,29,149,.5)}
.hud{position:fixed;z-index:12;bottom:24px;left:5vw;right:5vw;display:flex;justify-content:space-between;font-size:10px;letter-spacing:.16em;color:#8e94b8;pointer-events:none}.progress{position:fixed;z-index:30;top:0;right:0;bottom:0;width:3px;background:#16152c}.progress>div{width:100%;height:100%;transform:scaleY(0);transform-origin:top;background:linear-gradient(#22d3ee,#a855f7)}
@media(max-width:700px){.nav{padding:22px}.nav-center,.hud{display:none}.chapter{padding:26px;min-height:100svh}.intro,.about,.work,.contact{justify-content:flex-start}.copy{max-width:100%}.shade{background:rgba(4,5,20,.42)}h1{font-size:clamp(3rem,13vw,5rem)}h2{font-size:clamp(3rem,12vw,5rem)}.project{gap:10px}.soon{font-size:8px}}
@media(prefers-reduced-motion:reduce){.actions a{transition:none}}
</style>
