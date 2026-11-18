<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'

const props = withDefaults(defineProps<{
  /** Nombre de rebonds : la balle double à chaque rebond (2^bounces balles au final) */
  bounces?: number
  /** Texte révélé à la fin */
  text?: string
  /** Couleur de la balle d'origine (et du flash) */
  color?: string
  /** Diamètre de la balle d'origine (px) */
  size?: number
  /** Hauteur de la scène (px) */
  height?: number
  /** Hauteur du "sol" depuis le bas de la scène (px) */
  floor?: number
  /** Durée de la première chute (s) */
  fallTime?: number
  /** Chaque rebond atteint ce ratio de la hauteur du précédent (0-1) */
  decay?: number
  /** Durée de la sortie en roulant (s) */
  rollTime?: number
  /** Délai avant le départ (s) */
  delay?: number
  /** Nombre maximum de balles à l'écran */
  maxBalls?: number
  /** Vitesse de changement de couleur des autres balles (degrés de teinte par seconde) */
  colorSpeed?: number
  /** Durée du flash et de l'apparition du texte (s) */
  flashTime?: number
  /** Taille de police du texte (valeur CSS) */
  fontSize?: string
  /** Police du texte (valeur CSS) */
  fontFamily?: string
}>(), {
  bounces: 4,
  text: 'Contratchap',
  color: '#2563eb',
  size: 56,
  height: 420,
  floor: 90,
  fallTime: 0.75,
  decay: 0.58,
  rollTime: 1.6,
  delay: 0.3,
  maxBalls: 32,
  colorSpeed: 120,
  flashTime: 1.1,
  fontSize: 'min(13vw, 8rem)',
  fontFamily: 'inherit',
})

const emit = defineEmits<{ (e: 'done'): void }>()

// ───────── Balles ─────────
const nBounces = computed(() => Math.min(8, Math.max(0, Math.round(props.bounces))))
const ballCount = computed(() => Math.max(1, Math.min(props.maxBalls, 2 ** nBounces.value)))
const ballIdx = computed(() => Array.from({ length: ballCount.value }, (_, i) => i))

/** Génération de la balle i : 0 = balle d'origine, 1 = née au 1er impact, etc. */
const genOf = (i: number) => (i === 0 ? 0 : 32 - Math.clz32(i))
const sizeOf = (i: number) => props.size * Math.max(0.55, 1 - 0.1 * genOf(i))

/** Suite de van der Corput : répartit les balles uniformément sur la largeur (0, .5, .25, .75, ...) */
function vdc(i: number) {
  let f = 0
  let d = 0.5
  while (i > 0) {
    if (i & 1) f += d
    i >>= 1
    d /= 2
  }
  return f
}

const clamp = (v: number, a = 0, b = 1) => Math.min(b, Math.max(a, v))
const easeOutBack = (u: number) => {
  const c1 = 1.70158
  const c3 = c1 + 1
  return 1 + c3 * Math.pow(u - 1, 3) + c1 * Math.pow(u - 1, 2)
}
const easeOutCubic = (u: number) => 1 - Math.pow(1 - u, 3)

// ───────── Références DOM ─────────
const root = ref<HTMLElement>()
const label = ref<HTMLElement>()
const flash = ref<HTMLElement>()
const ring = ref<HTMLElement>()
const moverEls: (HTMLElement | null)[] = []
const rollerEls: (HTMLElement | null)[] = []
const shadowEls: (HTMLElement | null)[] = []

let raf = 0
let observer: IntersectionObserver | null = null
let isPlaying = false

const reduceMotion = () =>
  typeof window !== 'undefined' && window.matchMedia?.('(prefers-reduced-motion: reduce)').matches

function colorOf(i: number, t: number) {
  if (i === 0) return props.color
  return `hsl(${(i * 137.5 + t * props.colorSpeed) % 360} 85% 55%)`
}

function hideBall(i: number) {
  const m = moverEls[i]
  const s = shadowEls[i]
  if (m) m.style.opacity = '0'
  if (s) s.style.opacity = '0'
}

function hideAllBalls() {
  for (let i = 0; i < ballCount.value; i++) hideBall(i)
}

function setBall(
  i: number, x: number, y: number, squash: number, angle: number,
  pop: number, color: string, shadowK: number,
) {
  const m = moverEls[i]
  const r = rollerEls[i]
  const s = shadowEls[i]
  if (!m || !r || !s) return
  const half = sizeOf(i) / 2
  m.style.opacity = '1'
  m.style.transform =
    `translate3d(${x - half}px, ${-y}px, 0) scale(${pop * (1 + squash)}, ${pop * (1 - squash)})`
  r.style.transform = `rotate(${angle}rad)`
  r.style.background = color
  s.style.transform = `translate3d(${x - half}px, 0, 0) scale(${shadowK * pop})`
  s.style.opacity = String(0.35 * shadowK * pop)
}

function setText(opacity: number, scale: number, blur: number, glow: number) {
  const el = label.value
  if (!el) return
  el.style.opacity = String(opacity)
  el.style.transform = `scale(${scale})`
  el.style.filter = blur > 0.05 ? `blur(${blur}px)` : 'none'
  el.style.textShadow = glow > 0.5 ? `0 0 ${glow}px ${props.color}` : 'none'
}

function setFlash(opacity: number, scale: number, ringOpacity: number, ringScale: number) {
  if (flash.value) {
    flash.value.style.opacity = String(opacity)
    flash.value.style.transform = `scale(${scale})`
  }
  if (ring.value) {
    ring.value.style.opacity = String(ringOpacity)
    ring.value.style.transform = `scale(${ringScale})`
  }
}

function reset() {
  cancelAnimationFrame(raf)
  isPlaying = false
  if (!root.value) return
  const H = Math.max(40, root.value.clientHeight - floorSpace())
  const W = root.value.clientWidth
  hideAllBalls()
  setBall(0, W * 0.1, H, 0, 0, 1, props.color, 0.4)
  setText(0, 1.18, 10, 0)
  setFlash(0, 0.4, 0, 0.2)
}

function showFinal() {
  cancelAnimationFrame(raf)
  isPlaying = false
  hideAllBalls()
  setText(1, 1, 0, 0)
  setFlash(0, 1, 0, 1)
}

/** Espace occupé sous et au-dessus de la balle d'origine pour calculer la hauteur de chute */
const floorSpace = () => props.floor + props.size + 16

function play() {
  if (!root.value) return
  if (reduceMotion()) { showFinal(); emit('done'); return }

  cancelAnimationFrame(raf)
  isPlaying = true

  const W = root.value.clientWidth
  const H = Math.max(40, root.value.clientHeight - floorSpace())
  const N = ballCount.value
  const n = nBounces.value

  // Planning : une chute depuis le haut, puis n rebonds de plus en plus bas
  type Seg = { kind: 'fall' | 'bounce'; apex: number; dur: number }
  const segs: Seg[] = [{ kind: 'fall', apex: H, dur: props.fallTime }]
  for (let k = 1; k <= n; k++) {
    const apex = H * Math.pow(props.decay, k)
    segs.push({ kind: 'bounce', apex, dur: 2 * props.fallTime * Math.sqrt(apex / H) })
  }
  // ends[k] = instant du k-ième impact au sol (le 0 est la fin de la première chute)
  const ends: number[] = []
  segs.reduce((acc, g) => { ends.push(acc + g.dur); return acc + g.dur }, 0)
  const bounceTotal = ends[ends.length - 1]

  const targetX = (i: number) => W * (0.1 + 0.8 * vdc(i))

  /** Position horizontale : une nouvelle balle naît sous sa parente puis glisse vers sa place */
  function ballX(i: number, t: number): number {
    if (i === 0) return targetX(0)
    const g = genOf(i)
    const ts = ends[g - 1]
    const parent = i - (1 << (g - 1))
    const startX = ballX(parent, ts)
    const u = clamp((t - ts) / segs[g].dur)
    const s = u * u * (3 - 2 * u)
    return startX + (targetX(i) - startX) * s
  }

  // Sortie : même accélération pour toutes, la balle la plus à gauche sort en rollTime
  const accel = (2 * (W + props.size - targetX(0))) / (props.rollTime * props.rollTime)
  const rollEnd = bounceTotal + props.rollTime
  const t3 = rollEnd + 0.12          // début du flash
  const tEnd = t3 + props.flashTime
  const PEAK = 0.12                  // moment du pic de flash (fraction de flashTime)

  let start = 0
  let ballsHidden = false

  const tick = (now: number) => {
    if (!start) start = now + props.delay * 1000
    const t = Math.max(0, (now - start) / 1000)

    if (t < bounceTotal) {
      // Phase 1 : rebonds, la population double à chaque impact
      let acc = 0
      let seg = segs[0]
      for (const g of segs) { seg = g; if (t < acc + g.dur) break; acc += g.dur }
      const u = Math.min(1, (t - acc) / seg.dur)
      const y = seg.kind === 'fall' ? seg.apex * (1 - u * u) : seg.apex * 4 * u * (1 - u)
      const shadowK = 1 - Math.min(1, y / H) * 0.6

      for (let i = 0; i < N; i++) {
        const g = genOf(i)
        const ts = g === 0 ? 0 : ends[g - 1]
        if (t < ts) { hideBall(i); continue }
        const pop = g === 0 ? 1 : easeOutBack(clamp((t - ts) / 0.2))
        const half = sizeOf(i) / 2
        const squash = Math.max(0, 1 - y / (half * 0.8)) * 0.22
        setBall(i, ballX(i, t), y, squash, 0, pop, colorOf(i, t), shadowK)
      }
    } else if (t < rollEnd) {
      // Phase 2 : toutes les balles roulent vers la droite et quittent l'écran
      const tau = t - bounceTotal
      const travelled = 0.5 * accel * tau * tau
      for (let i = 0; i < N; i++) {
        const half = sizeOf(i) / 2
        setBall(i, targetX(i) + travelled, 0, 0, travelled / half, 1, colorOf(i, t), 1)
      }
    } else if (t < t3) {
      // Courte pause : l'écran est vide
      if (!ballsHidden) { hideAllBalls(); ballsHidden = true }
    } else {
      // Phase 3 : flash, puis le texte apparaît
      if (!ballsHidden) { hideAllBalls(); ballsHidden = true }
      const u = clamp((t - t3) / props.flashTime)
      const flashOp = u < PEAK ? u / PEAK : Math.pow(1 - (u - PEAK) / (1 - PEAK), 2)
      setFlash(flashOp, 0.4 + 0.9 * easeOutCubic(u), u < PEAK ? 0 : (1 - u) * 0.9, 0.2 + 2.6 * easeOutCubic(u))

      if (u < PEAK) {
        setText(0, 1.18, 10, 0)
      } else {
        const s = easeOutCubic(clamp((u - PEAK) / 0.55))
        setText(1, 1 + 0.18 * (1 - s), 10 * (1 - s), 60 * (1 - s))
      }
    }

    if (t < tEnd) {
      raf = requestAnimationFrame(tick)
    } else {
      showFinal()
      emit('done')
    }
  }
  raf = requestAnimationFrame(tick)
}

onMounted(() => {
  reset()
  // Slidev garde les slides montées : on joue l'animation quand la slide devient visible
  observer = new IntersectionObserver(([entry]) => {
    if (entry.isIntersecting && !isPlaying) play()
    else if (!entry.isIntersecting) reset()
  }, { threshold: 0.6 })
  if (root.value) observer.observe(root.value)
})

onBeforeUnmount(() => {
  cancelAnimationFrame(raf)
  observer?.disconnect()
})

defineExpose({ play, reset })
</script>

<template>
  <div
    ref="root"
    class="cc-stage"
    :style="{ height: `${height}px`, '--floor': `${floor}px`, '--dot': color }"
  >
    <div class="cc-title" :style="{ fontSize, fontFamily }">
      <span ref="flash" class="cc-flash" aria-hidden="true" />
      <span ref="ring" class="cc-ring" aria-hidden="true" />
      <span ref="label" class="cc-text">{{ text }}</span>
    </div>

    <template v-for="i in ballIdx" :key="i">
      <div
        :ref="(el) => { shadowEls[i] = el as HTMLElement }"
        class="cc-shadow"
        :style="{ width: `${sizeOf(i)}px` }"
        aria-hidden="true"
      />
      <div
        :ref="(el) => { moverEls[i] = el as HTMLElement }"
        class="cc-ball"
        :style="{ width: `${sizeOf(i)}px`, height: `${sizeOf(i)}px` }"
        aria-hidden="true"
      >
        <div :ref="(el) => { rollerEls[i] = el as HTMLElement }" class="cc-roller">
          <span class="cc-glint" />
        </div>
      </div>
    </template>
  </div>
</template>

<style scoped>
.cc-stage {
  position: relative;
  width: 100%;
  /* Le layout "center" de Slidev est une grille qui s'adapte à son contenu :
     sans largeur minimale, la scène s'effondre à 0 px et rien n'est visible.
     Le canevas Slidev fait 980 px par défaut. */
  min-width: 900px;
  overflow-x: clip;
  overflow-y: visible;
}

.cc-title {
  position: absolute;
  inset-inline: 0;
  bottom: calc(var(--floor) - 0.16em);
  text-align: center;
  line-height: 1;
}

.cc-text {
  display: inline-block;
  font-weight: 800;
  letter-spacing: -0.03em;
  white-space: nowrap;
  opacity: 0;
  will-change: transform, filter, opacity;
}

/* Flash centré sur le texte */
.cc-flash {
  position: absolute;
  left: 50%;
  top: 50%;
  width: 800px;
  height: 800px;
  margin: -400px 0 0 -400px;
  border-radius: 50%;
  background: radial-gradient(
    circle closest-side,
    #fff 0%,
    color-mix(in srgb, var(--dot) 60%, transparent) 40%,
    transparent 100%
  );
  opacity: 0;
  pointer-events: none;
}

.cc-ring {
  position: absolute;
  left: 50%;
  top: 50%;
  width: 300px;
  height: 300px;
  margin: -150px 0 0 -150px;
  border: 3px solid var(--dot);
  border-radius: 50%;
  opacity: 0;
  pointer-events: none;
}

.cc-ball {
  position: absolute;
  left: 0;
  bottom: var(--floor);
  transform-origin: 50% 100%;
  opacity: 0;
  will-change: transform;
}

.cc-roller {
  position: relative;
  width: 100%;
  height: 100%;
  border-radius: 50%;
  background: var(--dot);
  will-change: transform;
}

/* Petit repère clair : sans lui, on ne voit pas la balle rouler */
.cc-glint {
  position: absolute;
  top: 14%;
  left: 50%;
  width: 18%;
  height: 18%;
  margin-left: -9%;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.85);
}

.cc-shadow {
  position: absolute;
  left: 0;
  bottom: calc(var(--floor) - 5px);
  height: 10px;
  border-radius: 50%;
  background: currentColor;
  opacity: 0;
  filter: blur(3px);
}
</style>