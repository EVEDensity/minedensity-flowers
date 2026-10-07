<template>
  <canvas ref="canvas" class="flower-garden" aria-hidden="true" />
  <p v-if="webglError" class="garden-error" role="alert">无法启动 WebGL。请启用浏览器硬件加速，或换用支持 WebGL 的浏览器。</p>
  <div v-if="sceneState.threeFlowersReady && sceneState.phase === 'interactive'" class="garden-hint">
    <span>↑</span> 按住草地向上拖动，让花生长 <i>·</i> 也可以轻点种花
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, onActivated, onDeactivated } from 'vue'
import * as THREE from 'three'
import { sceneState, isInGroundZone, type FlowerData } from '@/composables/useSceneState'
import { plantFlower } from '@/composables/useFlowers'
import { shootPoint, foldedPetal } from '@/composables/flowerGrowth'
import { createGardenWorld, terrainHeight } from './gardenWorld'
import { illustrationMaterial } from './illustrationMaterial'

const emit = defineEmits<{ planted: [] }>()
const canvas = ref<HTMLCanvasElement | null>(null)
const webglError = ref(false)
const ink = '#293e3d'
const palettes = [
  ['#829bbd', '#9aaec9', '#b4c3d5', '#cad2d9'],
  ['#bd8584', '#ce9992', '#ddb0a2', '#e8c2ae'],
  ['#c5af79', '#d7bf8d', '#e5cea1', '#eee0bd'],
]
const scene = new THREE.Scene()
const camera = new THREE.PerspectiveCamera(46, 1, 5, 5000)
camera.position.set(0, 680, 1250)
camera.lookAt(0, 120, -180)
const world = createGardenWorld(scene)
const raycaster = new THREE.Raycaster()
function groundPoint(x: number, y: number) {
  camera.updateMatrixWorld(); world.ground.updateMatrixWorld()
  raycaster.setFromCamera(new THREE.Vector2(x * 2 - 1, 1 - y * 2), camera)
  const groundHit = raycaster.intersectObject(world.ground)[0]
  if (!groundHit) return
  world.plantingObstacles.forEach(object => object.updateWorldMatrix(true, false))
  const obstruction = raycaster.intersectObjects(world.plantingObstacles, false)[0]
  if (obstruction && obstruction.distance < groundHit.distance) return
  return groundHit.point.clone()
}
let renderer: THREE.WebGLRenderer | undefined
let raf = 0
let previous = 0
let active = false
let width = 1
let height = 1
const reducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)')
const smooth = (v: number) => { const x = THREE.MathUtils.clamp(v, 0, 1); return x * x * (3 - 2 * x) }

// Shared geometry is allocated once; each blossom only owns transforms.
const geometries: THREE.BufferGeometry[] = []
const materials: THREE.Material[] = []
function keep<T extends THREE.BufferGeometry>(geometry: T): T { geometries.push(geometry); return geometry }
function material(color: string) {
  const m = illustrationMaterial(color, true)
  materials.push(m)
  return m
}
function solid(color: string) {
  const m = illustrationMaterial(color)
  materials.push(m); return m
}
const stemMaterial = solid('#79a18c')
const stemInk = solid(ink)
const leafMaterial = material('#7da995')
const centerMaterial = solid('#e9b891')
const tipMaterial = solid('#edcc76')
const stemGeometry = keep(new THREE.CylinderGeometry(1, 1, 1, 7))
const sphereGeometry = keep(new THREE.SphereGeometry(1, 12, 8))

function blade(length: number, breadth: number, curve: number, variant = 0, leaf = false) {
  const vertices: number[] = [], indices: number[] = [], colors: number[] = [], edge: THREE.Vector3[] = []
  const steps = 24
  for (let i = 0; i <= steps; i++) {
    const t = i / steps
    // Broad, rounded petal tips and narrow roots, not diamond-shaped blades.
    const profile = leaf ? Math.pow(Math.sin(Math.PI * t), 0.8) : Math.pow(Math.sin(Math.PI * t), 0.36) * (0.22 + 0.78 * Math.pow(t, 0.38))
    const w = profile * breadth * (1 + 0.15 * Math.sin(t * 13 + variant)) + 0.12
    const drift = Math.sin(t * Math.PI * 0.9) * Math.sin(variant * 2.3) * 7 + t * t * Math.cos(variant) * 7
    const y = t * length
    const z = curve * Math.sin(Math.PI * t) + Math.pow(t, 4) * (variant % 2 ? 18 : -19)
    const twist = Math.sin(t * Math.PI) * Math.sin(variant * 1.7) * 7
    vertices.push(drift - w, y, z - twist, drift, y, z + Math.sin(Math.PI * t) * 2, drift + w, y, z + twist)
    for (let j = 0; j < 3; j++) {
      const wash = 0.79 + t * 0.18 + Math.sin(t * 11 + variant) * 0.018 + (j === 1 ? 0.02 : 0)
      colors.push(wash, wash, wash)
    }
    edge.push(new THREE.Vector3(drift - w, y, z - twist))
    if (i < steps) for (let j = 0; j < 2; j++) {
      const a = i * 3 + j
      indices.push(a, a + 3, a + 1, a + 1, a + 3, a + 4)
    }
  }
  for (let i = steps; i >= 0; i--) {
    const k = i * 9
    edge.push(new THREE.Vector3(vertices[k + 6], vertices[k + 7], vertices[k + 8]))
  }
  const mesh = keep(new THREE.BufferGeometry())
  mesh.setAttribute('position', new THREE.Float32BufferAttribute(vertices, 3))
  mesh.setAttribute('color', new THREE.Float32BufferAttribute(colors, 3))
  mesh.setIndex(indices); mesh.computeVertexNormals()
  const outline = keep(new THREE.TubeGeometry(new THREE.CatmullRomCurve3(edge, true), 64, leaf ? 0.62 : 0.65, 3, true))
  const veins: THREE.Vector3[] = []
  for (let i = 3; i < steps - 3; i++) {
    const k = i * 9
    const next = (i + 1) * 9
    veins.push(new THREE.Vector3(vertices[k + 3], vertices[k + 4], vertices[k + 5] + 0.35), new THREE.Vector3(vertices[next + 3], vertices[next + 4], vertices[next + 5] + 0.35))
    if (leaf && i % 4 === 0) {
      for (const side of [0, 6]) veins.push(new THREE.Vector3(vertices[k + 3], vertices[k + 4], vertices[k + 5] + 0.4), new THREE.Vector3(vertices[next + side] * 0.8, vertices[next + side + 1], vertices[next + side + 2] + 0.4))
    }
  }
  const veinGeometry = keep(new THREE.BufferGeometry().setFromPoints(veins))
  if (!leaf) for (const geometry of [mesh, outline, veinGeometry]) {
    const positions = geometry.getAttribute('position')
    const folded: number[] = []
    for (let i = 0; i < positions.count; i++) folded.push(...foldedPetal(positions.getX(i), positions.getY(i), positions.getZ(i), length))
    geometry.morphAttributes.position = [new THREE.Float32BufferAttribute(folded, 3)]
    const foldedGeometry = new THREE.BufferGeometry()
    foldedGeometry.setAttribute('position', new THREE.Float32BufferAttribute(folded, 3))
    if (geometry.index) foldedGeometry.setIndex(geometry.index.clone())
    foldedGeometry.computeVertexNormals()
    geometry.morphAttributes.normal = [foldedGeometry.getAttribute('normal').clone()]
    foldedGeometry.dispose()
  }
  return { mesh, outline, veins: veinGeometry }
}
const petalShapes = Array.from({ length: 13 }, (_, i) => blade(78 + (i % 7) * 4, 7 + (i % 4) * 1.7, 4 + i * 1.2, i))
const leafShape = blade(49, 13, 4, 2, true)
const veinMaterial = new THREE.LineBasicMaterial({ color: '#53665e', transparent: true, opacity: 0.34 })
materials.push(veinMaterial)
function outlined(shape: ReturnType<typeof blade>, mat: THREE.Material) {
  const g = new THREE.Group()
  g.add(new THREE.Mesh(shape.mesh, mat), new THREE.Mesh(shape.outline, stemInk), new THREE.LineSegments(shape.veins, veinMaterial))
  g.traverse(object => { if (object instanceof THREE.Mesh) { object.castShadow = true; object.receiveShadow = true } })
  return g
}
const flowerMaterials = palettes.map(colors => colors.map(material))
// Near-camera leaves share the flower renderer, so they can actually occlude
// blossoms and move independently of the distant painted valley.
const foreground = new THREE.Group()
const foregroundLeaves: { mesh: THREE.Group; side: number; index: number; angle: number }[] = []
const foregroundMaterial = material('#657f70')
for (const side of [-1, 1]) for (let i = 0; i < 5; i++) {
  const mesh = outlined(leafShape, foregroundMaterial)
  const angle = side * (0.28 + i * 0.19)
  foreground.add(mesh); foregroundLeaves.push({ mesh, side, index: i, angle })
}
// Foreground placement is now world-space, rather than a screen-space overlay.
scene.add(foreground)
interface Rig {
  group: THREE.Group; head: THREE.Group; petals: { pivot: THREE.Group; surfaces: (THREE.Mesh | THREE.LineSegments)[]; ring: number; offset: number; curl: number }[]
  stems: THREE.Group[]; leaves: THREE.Group[]; shadow: THREE.Mesh; core: THREE.Group; tuft: THREE.Group; bud: THREE.Group
  stemHeight: number; bloomAge: number; targetBend?: number
  age: number; grow: number; target: number; bend: number; seed: number; tilt: number
  x: number; y: number; size: number; headSize: number; source?: FlowerData; demo: boolean
  fadingMaterials: { material: THREE.Material; opacity: number }[]
  root?: THREE.Vector3
}
const rigs: Rig[] = []
const shadowMaterial = new THREE.MeshBasicMaterial({ color: '#4b7163', transparent: true, opacity: 0.16, depthWrite: false })
materials.push(shadowMaterial)
const shadowShape = new THREE.Shape()
for (let i = 0; i <= 144; i++) {
  const angle = i / 144 * Math.PI * 2
  const radius = 0.78 + Math.sin(angle * 18) * 0.15 + Math.sin(angle * 7) * 0.05
  if (i === 0) shadowShape.moveTo(Math.cos(angle) * radius, Math.sin(angle) * radius)
  else shadowShape.lineTo(Math.cos(angle) * radius, Math.sin(angle) * radius)
}
const shadowGeometry = keep(new THREE.ShapeGeometry(shadowShape))
const filamentShapes = Array.from({ length: 9 }, (_, i) => {
  const lean = (i - 4) * 4.5
  const tall = 29 + (i % 4) * 7
  const path = new THREE.CubicBezierCurve3(new THREE.Vector3(0, 0, 0), new THREE.Vector3(lean * 0.3, 4, 12), new THREE.Vector3(lean * 1.2, tall * 0.6, 24), new THREE.Vector3(lean, tall, 31 + i % 3 * 5))
  return { outer: keep(new THREE.TubeGeometry(path, 18, 0.95, 4, false)), inner: keep(new THREE.TubeGeometry(path, 18, 0.45, 4, false)), tip: path.getPoint(1) }
})
const grassShapes = Array.from({ length: 6 }, (_, i) => {
  const path = new THREE.CubicBezierCurve3(new THREE.Vector3(0, 0, 0), new THREE.Vector3((i - 3) * 2, 13, 2), new THREE.Vector3((i - 3) * 5, 20 + i % 3 * 6, 3), new THREE.Vector3((i - 3) * 7, 5 + i % 2 * 9, 4))
  return keep(new THREE.TubeGeometry(path, 10, 0.8, 3, false))
})
function createRig(x: number, y: number, color: number, demo = false, source?: FlowerData) {
  const group = new THREE.Group(), head = new THREE.Group(), core = new THREE.Group()
  group.add(head)
  const petals: Rig['petals'] = []
  for (let ring = 0; ring < 4; ring++) {
    const count = [25, 21, 16, 11][ring]
    for (let i = 0; i < count; i++) {
      const radial = new THREE.Group(), pivot = new THREE.Group()
      radial.rotation.z = i / count * Math.PI * 2 + ring * 0.31 + (Math.random() - 0.5) * 0.075
      const petal = outlined(petalShapes[(i + ring * 3) % petalShapes.length], flowerMaterials[color][ring])
      const scale = [1, 0.82, 0.59, 0.35][ring] * (0.78 + Math.random() * 0.38)
      petal.scale.set(scale * (0.85 + Math.random() * 0.3), scale, scale)
      pivot.position.y = 5 + ring * 0.8
      pivot.position.z = ring * 3.8
      pivot.add(petal); radial.add(pivot); head.add(radial)
      petals.push({ pivot, surfaces: petal.children as (THREE.Mesh | THREE.LineSegments)[], ring, offset: Math.random() * 0.18, curl: (Math.random() - 0.5) * 0.24 })
    }
  }
  const center = new THREE.Mesh(sphereGeometry, centerMaterial)
  center.scale.set(15, 12, 8); core.add(center)
  const centerCap = new THREE.Mesh(sphereGeometry, tipMaterial)
  centerCap.scale.set(10, 7, 4); centerCap.position.set(-1, 3, 6); core.add(centerCap)
  for (let i = 0; i < 9; i++) {
    const stamen = new THREE.Group()
    const shape = filamentShapes[i]
    const filament = new THREE.Mesh(shape.outer, stemInk)
    const filamentFill = new THREE.Mesh(shape.inner, tipMaterial)
    filamentFill.position.z = 0.75
    const tipOutline = new THREE.Mesh(sphereGeometry, stemInk)
    tipOutline.scale.set(2.5, 4.9, 2.5); tipOutline.position.copy(shape.tip)
    const tip = new THREE.Mesh(sphereGeometry, tipMaterial)
    tip.scale.set(1.8, 4.1, 1.8); tip.position.copy(shape.tip); tip.position.z += 1.4
    stamen.add(filament, filamentFill, tipOutline, tip)
    stamen.rotation.z = (Math.random() - 0.5) * 0.35
    stamen.position.set((i - 4) * 0.8, 0, 5)
    core.add(stamen)
  }
  core.position.z = 11; head.add(core)
  // A compact calyx conceals the folded petals until the stem has matured.
  const bud = new THREE.Group()
  for (let i = 0; i < 5; i++) {
    const sepal = outlined(leafShape, leafMaterial)
    sepal.rotation.set(1.05, 0, i / 5 * Math.PI * 2)
    sepal.scale.set(0.22, 0.25, 0.22)
    bud.add(sepal)
  }
  head.add(bud)
  const stems = Array.from({ length: 20 }, () => {
    const segment = new THREE.Group()
    const outline = new THREE.Mesh(stemGeometry, stemInk)
    const fill = new THREE.Mesh(stemGeometry, stemMaterial)
    fill.scale.set(0.65, 1.01, 0.65); fill.position.z = 0.62
    segment.add(outline, fill)
    group.add(segment); return segment
  })
  const leaves = Array.from({ length: 6 }, () => {
    const leaf = outlined(leafShape, leafMaterial); group.add(leaf); return leaf
  })
  const shadow = new THREE.Mesh(shadowGeometry, shadowMaterial)
  shadow.position.z = -35; group.add(shadow)
  const tuft = new THREE.Group()
  grassShapes.forEach(shape => tuft.add(new THREE.Mesh(shape, stemInk)))
  tuft.position.z = 12; group.add(tuft)
  const rig: Rig = { group, head, petals, stems, leaves, shadow, core, tuft, bud,
    stemHeight: demo ? 230 : 0, bloomAge: demo ? 4 : 0,
    age: demo ? 3.5 : 0, grow: demo ? 1 : 0, target: 230, bend: (Math.random() - 0.5) * 55,
    seed: Math.random() * 6.28, tilt: -0.55 - Math.random() * 0.65, x, y, size: 1, headSize: 0.9 + Math.random() * 0.18, source, demo, fadingMaterials: [] }
  rigs.push(rig); scene.add(group)
  group.traverse(object => { if (object instanceof THREE.Mesh) { object.castShadow = true; object.receiveShadow = true } })
  shadow.visible = false; shadow.castShadow = false
  return rig
}
const axis = new THREE.Vector3(0, 1, 0)
const point = new THREE.Vector3(), last = new THREE.Vector3(), delta = new THREE.Vector3()
function stemPoint(r: Rig, t: number, time: number, target: THREE.Vector3) {
  const sway = reducedMotion.matches ? 0 : Math.sin(time * 1.3 + r.seed) * 3
  const p = shootPoint(t, r.stemHeight, (r.bend + sway) * r.grow, smooth(r.stemHeight / 160))
  return target.set(p.x, p.y, 0)
}
let drag: { rig: Rig; id: number; startX: number; startY: number; moved: boolean } | undefined
function updateRig(r: Rig, dt: number, time: number) {
  r.age += dt
  if (r.targetBend !== undefined) r.bend += (r.targetBend - r.bend) * (1 - Math.exp(-dt * 6))
  const held = drag?.rig === r
  const desiredHeight = held ? r.target : r.target * smooth((r.age - 0.12) / 1.65)
  // Height is continuous across pointer release; changing a target cannot teleport the flower.
  const step = (desiredHeight - r.stemHeight) * (1 - Math.exp(-dt * 5))
  r.stemHeight += THREE.MathUtils.clamp(step, 0, dt * 210)
  if (r.demo || reducedMotion.matches) r.stemHeight = r.target
  r.grow = THREE.MathUtils.clamp(r.stemHeight / Math.max(r.target, 1), 0, 1)
  if (r.grow > 0.85 && (!held || r.stemHeight > 110)) r.bloomAge += dt
  const open = reducedMotion.matches ? 1 : smooth((r.bloomAge - 0.25) / 1.6)
  r.size = 0.9
  if (!r.root) r.root = groundPoint(r.x, r.y)
  if (!r.root) return
  r.group.position.copy(r.root)
  r.group.scale.setScalar(r.size)
  // Fade pigment instead of shrinking the whole rooted plant into the ground.
  const opacity = r.source?.opacity ?? 1
  if (opacity < 0.999 && !r.fadingMaterials.length) {
    const clones = new Map<THREE.Material, THREE.Material>()
    r.group.traverse(object => {
      if (!(object instanceof THREE.Mesh || object instanceof THREE.LineSegments)) return
      const original = object.material as THREE.Material
      let clone = clones.get(original)
      if (!clone) {
        clone = original.clone(); clone.onBeforeCompile = original.onBeforeCompile
        clone.transparent = true; clones.set(original, clone)
        r.fadingMaterials.push({ material: clone, opacity: original.opacity })
      }
      object.material = clone
    })
  }
  r.fadingMaterials.forEach(entry => { entry.material.opacity = entry.opacity * opacity })
  stemPoint(r, 0, time, last)
  r.stems.forEach((stem, i) => {
    stemPoint(r, (i + 1) / r.stems.length, time, point)
    delta.subVectors(point, last)
    stem.position.copy(last).addScaledVector(delta, 0.5)
    stem.scale.set(2.5 - i * 0.085, Math.max(0.001, delta.length()), 2.5 - i * 0.085)
    stem.quaternion.setFromUnitVectors(axis, delta.normalize())
    last.copy(point)
  })
  r.leaves.forEach((leaf, i) => {
    const t = 0.16 + i * 0.115 + Math.sin(i * 3 + r.seed) * 0.025
    // Nodes stay at their own heights rather than sliding up with a stretching stem.
    const nodeHeight = (r.demo ? r.target : 240) * t
    const reached = smooth((r.stemHeight - nodeHeight) / 42)
    stemPoint(r, Math.min(1, nodeHeight / Math.max(r.stemHeight, 1)), time, leaf.position)
    const side = i % 2 ? 1 : -1
    leaf.rotation.set(0.2 + Math.sin(i + r.seed) * 0.45, side * 0.35, side * (1.03 + Math.sin(i * 2 + r.seed) * 0.24 + 0.35 * (1 - open)))
    leaf.scale.setScalar(reached * (0.82 + Math.sin(i + r.seed) * 0.18))
  })
  stemPoint(r, 1, time, r.head.position)
  const settleTime = Math.max(0, r.bloomAge - 1.8)
  const settle = reducedMotion.matches ? 0 : Math.sin(settleTime * 5) * Math.exp(-settleTime * 2.8) * 0.06
  stemPoint(r, 0.97, time, point)
  delta.subVectors(r.head.position, point)
  const tipAngle = -Math.atan2(delta.x, delta.y)
  r.head.rotation.set(-Math.PI / 2 * (1 - open) + r.tilt * open, Math.sin(r.seed) * 0.12 * open, tipAngle * (1 - open) + r.bend / 300 * open + settle)
  r.head.scale.setScalar(smooth(r.stemHeight / 22) * r.headSize)
  r.bud.position.z = -3
  r.bud.scale.setScalar(1)
  r.petals.forEach(p => {
    const progress = reducedMotion.matches ? 1 : smooth((r.bloomAge - 0.28 - p.ring * 0.17 - p.offset) / 1.35)
    p.pivot.visible = true
    for (const surface of p.surfaces) if (surface.morphTargetInfluences) surface.morphTargetInfluences[0] = 1 - progress
    const flutter = reducedMotion.matches ? 0 : Math.sin(time * 1.5 + p.offset * 35 + r.seed) * 0.018 * progress
    p.pivot.rotation.x = ([-0.10, 0.16, 0.40, 0.72][p.ring] + p.curl) * progress + flutter
    p.pivot.position.y = 1.5 + (3.5 + p.ring * 0.8) * progress
    p.pivot.position.z = p.ring * 1.2 + p.ring * 2.6 * progress
  })
  r.core.scale.setScalar(smooth((open - 0.25) / 0.7))
  const shadowGrowth = smooth(r.stemHeight / 150) * (0.18 + open * 0.82)
  r.shadow.scale.set(68 * shadowGrowth * r.headSize, 21 * shadowGrowth, 1)
  r.shadow.position.set(-32 + r.bend * 0.3, -4, -35)
  r.shadow.rotation.z = -0.12
  r.tuft.scale.setScalar(0.4 + r.grow * 0.45)
}
function resize() {
  width = window.innerWidth; height = window.innerHeight
  camera.aspect = width / height; camera.updateProjectionMatrix()
  renderer?.setSize(width, height)
}
function frame(now: number) {
  if (!active || !renderer) return
  const dt = Math.min((now - previous) / 1000 || 0.016, 0.05); previous = now
  if (document.hidden) { raf = requestAnimationFrame(frame); return }
  for (const f of sceneState.plantedFlowers) {
    if (!rigs.some(r => r.source === f)) createRig(f.x / width, f.y / height, sceneState.totalPlanted % 3, false, f)
  }
  for (let i = rigs.length - 1; i >= 0; i--) {
    const r = rigs[i]
    if (r.source && !sceneState.plantedFlowers.includes(r.source)) { scene.remove(r.group); r.fadingMaterials.forEach(e => e.material.dispose()); rigs.splice(i, 1); continue }
    updateRig(r, dt, now / 1000)
  }
  const mouse = reducedMotion.matches ? 0 : sceneState.mouseX / Math.max(width, 1) - 0.5
  camera.position.x += (mouse * 42 - camera.position.x) * (1 - Math.exp(-dt * 3))
  camera.lookAt(0, 120, -180)
  foregroundLeaves.forEach(({ mesh, side, index, angle }) => {
    const x = side * (480 + index * 48), z = 600 + index * 35
    mesh.position.set(x, terrainHeight(x, z), z)
    mesh.rotation.set(0.12, side * 0.3, angle + (reducedMotion.matches ? 0 : Math.sin(now / 2300 + index) * 0.025))
    mesh.scale.set(2.4, 3.5 - index * 0.32, 1)
  })
  scene.visible = sceneState.phase !== 'entry'
  renderer.render(scene, camera)
  raf = requestAnimationFrame(frame)
}
function isControl(target: EventTarget | null) {
  return target instanceof Element && !!target.closest('button, a, input, textarea, select, [role="button"]')
}
function down(e: PointerEvent) {
  if (!active || !sceneState.threeFlowersReady || sceneState.phase !== 'interactive' || e.button !== 0 || drag || isControl(e.target) || !isInGroundZone(e.clientY, height)) return
  const root = groundPoint(e.clientX / width, e.clientY / height)
  if (!root) return
  plantFlower(e.clientX, e.clientY)
  const source = sceneState.plantedFlowers[sceneState.plantedFlowers.length - 1]
  const rig = createRig(e.clientX / width, e.clientY / height, sceneState.totalPlanted % 3, false, source)
  rig.target = 38
  rig.root = root
  drag = { rig, id: e.pointerId, startX: e.clientX, startY: e.clientY, moved: false }
  e.preventDefault()
}
function move(e: PointerEvent) {
  if (!drag || drag.id !== e.pointerId) return
  const dy = drag.startY - e.clientY
  if (Math.hypot(e.clientX - drag.startX, dy) > 8) drag.moved = true
  const root = drag.rig.root
  if (root) {
    raycaster.setFromCamera(new THREE.Vector2(e.clientX / width * 2 - 1, 1 - e.clientY / height * 2), camera)
    const dragPlane = new THREE.Plane(new THREE.Vector3(0, 0, 1), -root.z)
    const tip = raycaster.ray.intersectPlane(dragPlane, new THREE.Vector3())
    if (tip) {
      drag.rig.target = Math.max(drag.rig.target, THREE.MathUtils.clamp((tip.y - root.y) / drag.rig.size, 38, 330))
      drag.rig.targetBend = THREE.MathUtils.clamp((tip.x - root.x) / drag.rig.size, -90, 90)
    }
  }
  e.preventDefault()
}
function release(e?: PointerEvent) {
  if (!drag || (e && e.pointerId !== drag.id)) return
  if (!drag.moved) drag.rig.target = 190 + Math.random() * 70
  else drag.rig.age = Math.max(drag.rig.age, 1.77)
  drag = undefined
  emit('planted')
}
function start() {
  if (!renderer || active) return
  active = true; previous = 0; resize(); raf = requestAnimationFrame(frame)
  window.addEventListener('pointerdown', down)
  window.addEventListener('pointermove', move, { passive: false })
  window.addEventListener('pointerup', release)
  window.addEventListener('pointercancel', release)
  window.addEventListener('blur', blur)
}
function blur() { release() }
function stop() {
  release(); active = false; cancelAnimationFrame(raf)
  window.removeEventListener('pointerdown', down); window.removeEventListener('pointermove', move)
  window.removeEventListener('pointerup', release); window.removeEventListener('pointercancel', release)
  window.removeEventListener('blur', blur)
}
onMounted(() => {
  try {
    renderer = new THREE.WebGLRenderer({ canvas: canvas.value!, alpha: true, antialias: true })
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
    renderer.setClearColor(0x000000, 0)
    renderer.shadowMap.enabled = true
    renderer.shadowMap.type = THREE.PCFShadowMap
    renderer.toneMapping = THREE.ACESFilmicToneMapping
    renderer.toneMappingExposure = 1.05
    resize()
    const rear = createRig(0.43, 0.66, 2, true)
    rear.target = 270; rear.headSize = 0.76; rear.tilt = -0.55; rear.bend = -12
    const left = createRig(0.35, 0.81, 1, true)
    left.target = 160; left.headSize = 0.98; left.tilt = -1.1; left.bend = -16
    const middle = createRig(0.51, 0.81, 0, true)
    middle.target = 277; middle.headSize = 1.18; middle.tilt = -0.65; middle.bend = -15
    const right = createRig(0.61, 0.71, 0, true)
    right.target = 303; right.headSize = 1.05; right.tilt = -0.6; right.bend = -30
    const side = createRig(0.68, 0.83, 1, true)
    side.target = 207; side.headSize = 0.96; side.tilt = -1.23; side.bend = 20
    sceneState.threeFlowersReady = true
    window.addEventListener('resize', resize)
    start()
  } catch (error) { webglError.value = true; console.warn('WebGL garden unavailable.', error) }
})
onActivated(start)
onDeactivated(stop)
onUnmounted(() => {
  stop(); window.removeEventListener('resize', resize)
  sceneState.threeFlowersReady = false
  rigs.forEach(r => r.fadingMaterials.forEach(e => e.material.dispose()))
  world.dispose()
  geometries.forEach(g => g.dispose()); materials.forEach(m => m.dispose()); renderer?.dispose()
})
</script>

<style scoped>
.flower-garden { position: fixed; inset: 0; z-index: 4; pointer-events: none; }
.garden-error { position: fixed; top: 40%; left: 10%; width: 80%; z-index: 12; text-align: center; }
.garden-hint { position: fixed; bottom: 20px; left: 50%; transform: translateX(-50%); z-index: 10; pointer-events: none; white-space: nowrap; color: #e9e5d4; background: #293e3dbd; padding: 6px 14px; border-radius: 20px; font-size: 11px; letter-spacing: 0.08em; }
.garden-hint span { display: inline-block; margin-right: 8px; font-size: 18px; }
.garden-hint i { margin: 0 10px; font-style: normal; opacity: 0.5; }
@media (max-width: 640px) { .garden-hint { font-size: 10px; letter-spacing: 0; bottom: 13px; } }
</style>
