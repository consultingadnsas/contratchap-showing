<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref, watch } from 'vue'

const props = withDefaults(defineProps<{
  /** Texte affiché (les points de suspension sont ajoutés automatiquement) */
  text?: string
  /** Couleur de fond */
  color?: string
  /** Durée d'un cycle de points de suspension (s) */
  cycle?: number
  /** Taille de police (valeur CSS) */
  fontSize?: string
  /** Nombre de marches */
  steps?: number
  /** Hauteur d'une marche (px) */
  stepHeight?: number
  /** Largeur d'une marche (px) */
  stepWidth?: number
  /** Hauteur du sol (px) */
  groundHeight?: number
  /** Couleur des marches */
  stepColor?: string
  /** Hauteur totale du personnage, cheveux compris (px) */
  personHeight?: number
  /** Couleur du costume */
  suitColor?: string
  /** Couleur de la cravate */
  tieColor?: string
  /** Couleur de peau */
  skinColor?: string
  /** Couleur des cheveux */
  hairColor?: string
  /** Couleur de la mallette */
  caseColor?: string
}>(), {
  text: 'Les défis juridiques',
  color: '#2563eb',
  cycle: 1.8,
  fontSize: '4.5rem',
  steps: 6,
  stepHeight: 28,
  stepWidth: 82,
  groundHeight: 22,
  stepColor: '#ffffff',
  personHeight: 92,
  suitColor: '#0b1f6b',
  tieColor: '#ef4444',
  skinColor: '#f4c7a1',
  hairColor: '#1b2437',
  caseColor: '#92400e',
})

const root = ref<HTMLElement>()
const active = ref(false)
const W = ref(980)

let observer: IntersectionObserver | null = null
let resizeObserver: ResizeObserver | null = null
let raf = 0

// ───────── Pose du personnage ─────────
type Pose = {
  x: number
  y: number
  lean: number      // Inclinaison du buste (degrés, positif = vers l'avant)
  legL: number      // Angle de hanche, jambe arrière (négatif = vers l'avant)
  legR: number      // Angle de hanche, jambe avant
  armL: number      // Angle d'épaule, bras arrière
  armR: number      // Angle d'épaule, bras avant (tient la mallette)
  bob: number       // Mouvement vertical du corps
  stumble: number   // 0 → 1 : intensité du choc
  dir?: 1 | -1      // Sens du regard : 1 = vers la droite, -1 = vers la gauche
  fatigue?: number  // 0 → 1 : essoufflement
}
type CharState = Required<Pose> & { opacity: number }
type Seg = { dur: number; at: (u: number) => Pose }

const withDefaults_ = (p: Pose, opacity = 1): CharState => ({ dir: 1, fatigue: 0, ...p, opacity })

const charState = ref<CharState>(
  withDefaults_({ x: 0, y: 0, lean: 0, legL: 0, legR: 0, armL: 0, armR: 0, bob: 0, stumble: 0 }, 0),
)

// ───────── Articulations : genoux et coudes déduits des angles ─────────
// Une jambe qui part vers l'avant se plie au genou (le pied se lève), une jambe arrière se tend.
const kneeOf = (a: number) => (a < 0 ? Math.min(90, -a * 1.15) : a * 0.3)
// Un bras qui part vers l'avant se plie au coude, un bras arrière reste presque droit.
const elbowOf = (a: number) => (a < 0 ? a * 0.9 : -a * 0.15)

const legs = computed(() => {
  const s = charState.value
  return [
    { hip: s.legL, back: true },
    { hip: s.legR, back: false },
  ].map((l) => {
    const knee = kneeOf(l.hip)
    // Le pied reste en partie à plat : contre-rotation de la cheville
    return { ...l, knee, foot: -(l.hip + knee) * 0.55 }
  })
})

const arms = computed(() => {
  const s = charState.value
  const mk = (a: number) => {
    const elbow = elbowOf(a)
    // La mallette pend verticalement avec un léger effet de balancier
    return { a, elbow, hold: -(s.lean + a + elbow) * 0.85 }
  }
  return { back: mk(s.armL), front: mk(s.armR) }
})

const face = computed(() => {
  const s = charState.value
  if (s.stumble > 0.35) return 'ouch'
  if (s.fatigue > 0.5) return 'tired'
  if (s.lean >= 15) return 'effort'
  return 'neutral'
})

// ───────── Géométrie de l'escalier ─────────
const RUNWAY = 180

const nSteps = computed(() => Math.max(1, Math.round(props.steps)))
const stairs = computed(() => {
  const total = RUNWAY + nSteps.value * props.stepWidth
  const L = (W.value - total) / 2
  return {
    L,
    list: Array.from({ length: nSteps.value }, (_, k) => ({
      left: L + RUNWAY + k * props.stepWidth,
      height: props.groundHeight + (k + 1) * props.stepHeight,
    })),
  }
})

// ───────── Animation procédurale du personnage ─────────
const clamp01 = (u: number) => Math.min(1, Math.max(0, u))
const easeInOutCubic = (u: number) => (u < 0.5 ? 4 * u * u * u : 1 - Math.pow(-2 * u + 2, 3) / 2)
const easeOutQuad = (u: number) => 1 - (1 - u) * (1 - u)
const easeInQuad = (u: number) => u * u

function buildAttempt(): Seg[] {
  const { L } = stairs.value
  const n = nSteps.value
  const w = props.stepWidth
  const h = props.stepHeight

  const surf = (j: number) => props.groundHeight + j * h
  const left = (j: number) => L + RUNWAY + (j - 1) * w
  const center = (j: number) => left(j) + w * 0.45
  const launch = (j: number) => left(j) - 18
  const xStart = L + 40

  // Le personnage échoue à la marche f (entre 2 et n)
  const f = Math.min(n, 2 + Math.floor(Math.random() * Math.max(1, n - 1)))

  let stridePhase = 0

  // 1. Posture d'attente / respiration
  const idle = (dur: number, x: number, y: number, tired = false): Seg => ({
    dur,
    at: (u) => {
      const breath = Math.sin(u * Math.PI * (tired ? 4 : 2))
      return {
        x,
        y,
        lean: tired ? 14 + breath * 3 : 2 + breath * 1.5,
        legL: tired ? -8 : 0,
        legR: tired ? 10 : 0,
        armL: tired ? 12 : -4,
        armR: tired ? 15 : 4,
        bob: breath * (tired ? 1.8 : 0.8),
        stumble: 0,
        dir: 1,
        fatigue: tired ? 1 : 0,
      }
    },
  })

  // 2. Marche déterminée sur le plat ou sur une marche
  const walk = (x0: number, x1: number, y: number, dur: number, ease: (u: number) => number): Seg => {
    const startPhase = stridePhase
    const dist = Math.abs(x1 - x0)
    const dir: 1 | -1 = x1 >= x0 ? 1 : -1
    const cycles = Math.max(1, dist / 38) * Math.PI * 2
    stridePhase += cycles

    return {
      dur,
      at: (u) => {
        const e = ease(u)
        const phase = startPhase + cycles * e
        // L'amplitude s'annule aux extrémités : pas de saut de pose entre deux segments
        const env = Math.min(1, 3.2 * Math.sin(Math.PI * u))
        const swing = Math.sin(phase) * env
        return {
          x: x0 + (x1 - x0) * e,
          y,
          lean: 8,
          legL: swing * 28,
          legR: -swing * 28,
          armL: -swing * 24,
          armR: swing * 24,
          bob: Math.abs(Math.cos(phase)) * 2.5 * env,
          stumble: 0,
          dir,
        }
      },
    }
  }

  // 3. Enjambée / montée d'une marche avec effort
  const stepUp = (x0: number, y0: number, x1: number, y1: number, dur: number): Seg => {
    const isLeftLead = Math.sin(stridePhase) >= 0
    stridePhase += Math.PI
    return {
      dur,
      at: (u) => {
        const e = easeInOutCubic(u)
        const lift = Math.sin(u * Math.PI)
        // La jambe de tête revient sous la hanche en fin de mouvement : le pied se pose à plat sur la marche
        const leadAngle = -48 * Math.sin(u * Math.PI)
        const trailAngle = 22 * (1 - e)

        return {
          x: x0 + (x1 - x0) * e,
          y: y0 + (y1 - y0) * e + lift * 4,
          lean: 16 + lift * 6,
          legL: isLeftLead ? leadAngle : trailAngle,
          legR: isLeftLead ? trailAngle : leadAngle,
          armL: isLeftLead ? 32 * lift : -28 * lift,
          armR: isLeftLead ? -28 * lift : 32 * lift,
          bob: lift * 2,
          stumble: 0,
          dir: 1,
        }
      },
    }
  }

  // 4. Trébuchement contre la marche f (le défi bloquant)
  const tripAndRecover = (x0: number, y0: number): Seg[] => {
    const hitX = left(f) - 10
    const backX = launch(f) - 8
    return [
      // Il lève le pied mais heurte la contremarche
      {
        dur: 0.28,
        at: (u) => {
          const e = easeOutQuad(u)
          return {
            x: x0 + (hitX - x0) * e,
            y: y0 + Math.sin(u * Math.PI) * 5,
            lean: 18 + e * 14,
            legL: -38 * e,
            legR: 18 * e,
            armL: -35 * e,
            armR: 42 * e,
            bob: -e * 3,
            stumble: clamp01((u - 0.55) / 0.25),
            dir: 1,
          }
        },
      },
      // Recul et déséquilibre
      {
        dur: 0.42,
        at: (u) => {
          const e = easeOutQuad(u)
          const wobble = Math.sin(u * Math.PI * 3) * (1 - u) * 12
          return {
            x: hitX + (backX - hitX) * e,
            y: y0,
            lean: 32 * (1 - e) - 12 * Math.sin(u * Math.PI) + wobble * 0.4,
            legL: -38 * (1 - e) + 15 * e,
            legR: 18 * (1 - e) - 12 * e,
            armL: -35 * (1 - e) + wobble,
            armR: 42 * (1 - e) - wobble,
            bob: -3 * (1 - e),
            stumble: 1 - u,
            dir: 1,
          }
        },
      },
    ]
  }

  // 5. Descente prudente marche par marche
  const stepDown = (x0: number, y0: number, x1: number, y1: number, dur: number): Seg => {
    const isLeftLead = Math.sin(stridePhase) >= 0
    stridePhase += Math.PI
    return {
      dur,
      at: (u) => {
        const e = easeInQuad(u)
        const lift = Math.sin(u * Math.PI)
        return {
          x: x0 + (x1 - x0) * easeOutQuad(u),
          y: y0 + (y1 - y0) * e,
          lean: -4 + lift * 4,
          legL: isLeftLead ? 24 * lift : -12 * (1 - e),
          legR: isLeftLead ? -12 * (1 - e) : 24 * lift,
          armL: 16 * lift,
          armR: -16 * lift,
          bob: lift * 1.5,
          stumble: 0,
          dir: -1,
        }
      },
    }
  }

  const segs: Seg[] = []

  // Départ : il ajuste sa posture
  segs.push(idle(0.4, xStart, surf(0)))

  // Marche vers l'escalier
  segs.push(walk(xStart, launch(1), surf(0), (launch(1) - xStart) / 145, easeInOutCubic))

  // Ascension des marches 1 à f-1
  for (let j = 1; j < f; j++) {
    segs.push(stepUp(launch(j), surf(j - 1), center(j), surf(j), 0.46))
    segs.push(walk(center(j), launch(j + 1), surf(j), 0.24, easeOutQuad))
  }

  // Tentative ratée à la marche f
  segs.push(...tripAndRecover(launch(f), surf(f - 1)))

  // Petite pause dépitée / essoufflée avant de redescendre
  segs.push(idle(0.55, launch(f) - 8, surf(f - 1), true))

  // Descente marche par marche (il fait demi-tour)
  let curX = launch(f) - 8
  for (let k = f - 1; k >= 1; k--) {
    const edge = left(k) + 12
    segs.push(walk(curX, edge, surf(k), 0.25, easeInQuad))
    const landX = k - 1 >= 1 ? center(k - 1) : left(1) - 24
    segs.push(stepDown(edge, surf(k), landX, surf(k - 1), 0.36))
    curX = landX
  }

  // Retour au point de départ pour reprendre son élan
  segs.push(walk(curX, xStart, surf(0), Math.max(0.55, Math.abs(curX - xStart) / 140), easeInOutCubic))
  segs.push(idle(0.65, xStart, surf(0)))

  return segs
}

const reduceMotion = () =>
  typeof window !== 'undefined' && window.matchMedia?.('(prefers-reduced-motion: reduce)').matches

function startLoop() {
  stopLoop()
  if (root.value) W.value = root.value.clientWidth || 980

  let segs = buildAttempt()
  let total = segs.reduce((s, g) => s + g.dur, 0)
  charState.value = withDefaults_(segs[0].at(0))

  if (reduceMotion()) return

  let t0 = 0
  const tick = (now: number) => {
    if (!t0) t0 = now + 500
    let t = (now - t0) / 1000
    if (t >= total) {
      t0 = now - (t - total) * 1000
      segs = buildAttempt()
      total = segs.reduce((s, g) => s + g.dur, 0)
      t = (now - t0) / 1000
    }
    if (t >= 0) {
      let acc = 0
      for (const g of segs) {
        if (t < acc + g.dur) {
          charState.value = withDefaults_(g.at((t - acc) / g.dur))
          break
        }
        acc += g.dur
      }
    }
    raf = requestAnimationFrame(tick)
  }
  raf = requestAnimationFrame(tick)
}

function stopLoop() {
  cancelAnimationFrame(raf)
}

onMounted(() => {
  observer = new IntersectionObserver(([entry]) => {
    active.value = entry.isIntersecting
  }, { threshold: 0.6 })

  resizeObserver = new ResizeObserver(() => {
    if (root.value) W.value = root.value.clientWidth || 980
  })

  if (root.value) {
    observer.observe(root.value)
    resizeObserver.observe(root.value)
  }
})

watch(active, (on) => {
  if (on) startLoop()
  else {
    stopLoop()
    charState.value.opacity = 0
  }
})

onBeforeUnmount(() => {
  stopLoop()
  observer?.disconnect()
  resizeObserver?.disconnect()
})
</script>

<template>
  <div
    ref="root"
    class="dj-bg"
    :class="{ 'dj-on': active }"
    :style="{
      '--bg': color,
      '--cycle': `${cycle}s`,
      '--step': stepColor,
      '--suit': suitColor,
      '--tie': tieColor,
      '--skin': skinColor,
      '--hair': hairColor,
      '--case': caseColor,
      fontSize,
    }"
  >
    <h1 class="dj-title" :aria-label="`${text}...`">
      <span>{{ text }}</span><span class="dj-dots" aria-hidden="true">
        <i>.</i><i>.</i><i>.</i>
      </span>
    </h1>

    <div class="dj-stairs" aria-hidden="true">
      <div class="dj-ground" :style="{ height: `${groundHeight}px` }" />
      <div
        v-for="(s, i) in stairs.list"
        :key="i"
        class="dj-step"
        :style="{ left: `${s.left}px`, width: `${stepWidth}px`, height: `${s.height}px` }"
      />

      <!-- Avocat en costume, entièrement articulé (SVG) -->
      <div
        class="dj-person"
        :style="{
          height: `${personHeight}px`,
          width: `${personHeight * 0.769}px`,
          opacity: charState.opacity,
          transform: `translate3d(${charState.x - personHeight * 0.3845}px, ${-charState.y}px, 0)`,
        }"
      >
        <div class="dj-flip" :style="{ transform: `scaleX(${charState.dir})` }">
          <svg viewBox="0 14 80 104" width="100%" height="100%" overflow="visible">
            <!-- Ombre portée -->
            <ellipse cx="40" cy="117.6" rx="19" ry="2.6" fill="rgba(0,0,0,0.2)" />

            <g :transform="`translate(0 ${-charState.bob})`">
              <!-- ───── Jambes : cuisse + tibia + chaussure ───── -->
              <g
                v-for="(leg, i) in legs"
                :key="`leg${i}`"
                :transform="`translate(40 72) rotate(${leg.hip})`"
              >
                <rect x="-5.5" y="-3" width="11" height="22.5" rx="5.5" :class="leg.back ? 'c-pants-back' : 'c-pants'" />
                <g :transform="`translate(0 19) rotate(${leg.knee})`">
                  <rect x="-5" y="-3.5" width="10" height="22.5" rx="5" :class="leg.back ? 'c-pants-back' : 'c-pants'" />
                  <g :transform="`rotate(${leg.foot} 0 19)`">
                    <path
                      d="M -5.6 17.5 H 4.2 C 5.5 20.6 11.5 21 13.6 24.6 C 14.4 26 13.6 27 12.4 27 H -5.6 Z"
                      class="c-shoe"
                    />
                    <rect x="-5.8" y="25" width="19.6" height="2" rx="1" class="c-sole" />
                  </g>
                </g>
              </g>

              <!-- ───── Buste (s'incline avec l'effort) ───── -->
              <g :transform="`translate(40 72) rotate(${charState.lean})`">
                <!-- Bras arrière -->
                <g :transform="`translate(0 -25) rotate(${arms.back.a})`">
                  <rect x="-3.8" y="-3" width="7.6" height="15" rx="3.8" class="c-suit-back" />
                  <g :transform="`translate(0 11) rotate(${arms.back.elbow})`">
                    <rect x="-3.4" y="-3" width="6.8" height="14.5" rx="3.4" class="c-suit-back" />
                    <rect x="-3.6" y="8.6" width="7.2" height="2.4" rx="1" fill="#f8fafc" opacity="0.85" />
                    <circle cx="0" cy="13.2" r="3.3" class="c-skin-2" />
                  </g>
                </g>

                <!-- Cou -->
                <rect x="-3.2" y="-37" width="6.4" height="9" rx="2" class="c-skin-2" />

                <!-- Veste -->
                <path
                  d="M -9.5 -27 Q -9.5 -30.5 -6 -31 L 6 -31 Q 9.5 -30.5 9.5 -27 L 11 5 Q 11 8 8.5 8 L -8.5 8 Q -11 8 -11 5 Z"
                  class="c-suit"
                />
                <!-- Chemise -->
                <polygon points="-4.4,-31 4.4,-31 0,-12" fill="#f8fafc" />
                <!-- Cravate : garde une verticale approximative malgré l'inclinaison -->
                <g :transform="`rotate(${-charState.lean * 0.55} 0 -29)`">
                  <path d="M -2.3 -30.5 L 2.3 -30.5 L 1.8 -27.5 L 3.4 -14 L 0 -9.5 L -3.4 -14 L -1.8 -27.5 Z" class="c-tie" />
                  <path d="M 0 -27 L 0 -11" stroke="rgba(0,0,0,0.18)" stroke-width="0.8" fill="none" />
                </g>
                <!-- Revers -->
                <path d="M -4.4 -31 L -8.4 -17 L -1.6 -10.5 L -5 -17.4 Z" class="c-lapel" />
                <path d="M 4.4 -31 L 8.4 -17 L 1.6 -10.5 L 5 -17.4 Z" class="c-lapel" />
                <!-- Pochette et bouton -->
                <path d="M 4.8 -16.4 L 7.8 -16.8 L 7.8 -14.6 L 4.8 -14.6 Z" fill="#f8fafc" />
                <circle cx="0" cy="-3" r="0.9" fill="rgba(255,255,255,0.4)" />

                <!-- ───── Tête (reste tournée vers l'avant) ───── -->
                <g :transform="`rotate(${-charState.lean * 0.4} 0 -33)`">
                  <ellipse cx="-8" cy="-42.5" rx="2.2" ry="3" class="c-skin-2" />
                  <ellipse cx="1" cy="-43" rx="10" ry="11" class="c-skin" />
                  <circle cx="6.8" cy="-39.2" r="2" fill="rgba(239,68,68,0.16)" />
                  <!-- Nez -->
                  <path d="M 10.6 -42.6 Q 12.9 -40.6 10.8 -39.6" class="c-line c-line-skin" />

                  <!-- Cheveux -->
                  <path
                    d="M -9.5 -44 C -11.5 -54.5 -2 -58.5 6.5 -56 C 11.5 -54.5 12 -50 11 -47.5 C 8.5 -50.5 3 -51 -2 -49.5 C -5.5 -48.5 -7.5 -46.5 -9.5 -44 Z"
                    class="c-hair"
                  />
                  <path d="M -3.5 -54.6 C 0 -56.6 5 -56.2 8 -54.2" stroke="rgba(255,255,255,0.22)" stroke-width="1" fill="none" stroke-linecap="round" />

                  <!-- Expression : neutre -->
                  <template v-if="face === 'neutral'">
                    <circle cx="6.2" cy="-43.4" r="1.4" fill="#1e293b" />
                    <circle cx="6.7" cy="-43.9" r="0.45" fill="#fff" />
                    <path d="M 3.4 -47.4 Q 6.2 -48.8 9 -47.6" class="c-line" />
                    <path d="M 4.8 -37.6 Q 7.2 -35.6 9.6 -37.8" class="c-line" />
                  </template>

                  <!-- Expression : effort (sourcils froncés, dents serrées) -->
                  <template v-else-if="face === 'effort'">
                    <circle cx="6.4" cy="-43.2" r="1.3" fill="#1e293b" />
                    <path d="M 3.2 -48.6 L 9 -46.4" class="c-line c-line-thick" />
                    <rect x="4.6" y="-38.8" width="5.4" height="2.6" rx="1" fill="#fff" stroke="#1e293b" stroke-width="0.6" />
                    <path d="M 7.3 -38.8 V -36.2" stroke="#1e293b" stroke-width="0.5" />
                  </template>

                  <!-- Expression : aïe ! -->
                  <template v-else-if="face === 'ouch'">
                    <circle cx="6.2" cy="-43.4" r="2.3" fill="#fff" stroke="#1e293b" stroke-width="0.6" />
                    <circle cx="6.6" cy="-43.2" r="0.9" fill="#1e293b" />
                    <path d="M 3 -48.6 Q 6.2 -50.4 9.4 -48.8" class="c-line" />
                    <ellipse cx="7.2" cy="-36.6" rx="1.9" ry="2.4" fill="#7f1d1d" />
                    <!-- Étoiles du choc -->
                    <path class="dj-twinkle" transform="translate(-10 -58)" d="M 0 -3 L 0.9 -0.9 L 3 0 L 0.9 0.9 L 0 3 L -0.9 0.9 L -3 0 L -0.9 -0.9 Z" fill="#fde047" />
                    <path class="dj-twinkle dj-twinkle-2" transform="translate(13 -57)" d="M 0 -2.4 L 0.7 -0.7 L 2.4 0 L 0.7 0.7 L 0 2.4 L -0.7 0.7 L -2.4 0 L -0.7 -0.7 Z" fill="#fde047" />
                  </template>

                  <!-- Expression : épuisé (paupière mi-close, goutte de sueur) -->
                  <template v-else>
                    <path d="M 4.2 -43.6 Q 6.3 -41.9 8.4 -43.6" class="c-line" />
                    <path d="M 3.4 -46.8 Q 6 -47.6 8.8 -45.8" class="c-line" />
                    <path d="M 4.6 -37 Q 6 -38.4 7.4 -37 T 10 -37" class="c-line" />
                    <path
                      class="dj-sweat"
                      transform="translate(-6 -52)"
                      d="M 0 -3.5 C 2 -0.8 2.6 0.6 2.6 1.6 A 2.6 2.6 0 0 1 -2.6 1.6 C -2.6 0.6 -2 -0.8 0 -3.5 Z"
                      fill="#bae6fd"
                    />
                  </template>

                  <!-- Lunettes -->
                  <circle cx="6.2" cy="-43.4" r="3.9" fill="rgba(255,255,255,0.14)" stroke="#0f172a" stroke-width="0.9" />
                  <path d="M 2.3 -43.6 L -7.2 -42.6" stroke="#0f172a" stroke-width="0.9" stroke-linecap="round" />
                </g>

                <!-- Bras avant + mallette -->
                <g :transform="`translate(0 -25) rotate(${arms.front.a})`">
                  <rect x="-3.8" y="-3" width="7.6" height="15" rx="3.8" class="c-suit" />
                  <g :transform="`translate(0 11) rotate(${arms.front.elbow})`">
                    <rect x="-3.4" y="-3" width="6.8" height="14.5" rx="3.4" class="c-suit" />
                    <rect x="-3.6" y="8.6" width="7.2" height="2.4" rx="1" fill="#f8fafc" />
                    <g :transform="`rotate(${arms.front.hold} 0 13.2)`">
                      <!-- Poignée -->
                      <path d="M -3.6 18 V 15.6 Q -3.6 13.6 -1.8 13.6 H 1.8 Q 3.6 13.6 3.6 15.6 V 18" fill="none" stroke="#451a03" stroke-width="1.4" />
                      <!-- Corps de la mallette -->
                      <rect x="-9" y="17.5" width="18" height="12" rx="2" class="c-case" />
                      <rect x="-9" y="22" width="18" height="1" fill="rgba(0,0,0,0.25)" />
                      <rect x="-1.4" y="21" width="2.8" height="3" rx="0.6" fill="#fcd34d" />
                      <!-- Main -->
                      <circle cx="0" cy="13.2" r="3.3" class="c-skin" />
                    </g>
                  </g>
                  <circle cx="0" cy="-0.5" r="4.4" class="c-suit" />
                </g>
              </g>
            </g>

            <!-- Éclats d'impact au niveau du pied -->
            <g :opacity="charState.stumble" transform="translate(59 105)">
              <line x1="0" y1="0" x2="6" y2="-5" stroke="#fde047" stroke-width="2" stroke-linecap="round" />
              <line x1="1" y1="4" x2="8" y2="4" stroke="#fde047" stroke-width="2" stroke-linecap="round" />
              <line x1="-1" y1="-4" x2="2" y2="-9" stroke="#fde047" stroke-width="2" stroke-linecap="round" />
            </g>
          </svg>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.dj-bg {
  position: absolute;
  inset: 0;
  z-index: 10;
  display: flex;
  align-items: center;
  justify-content: center;
  padding-bottom: 22%;
  box-sizing: border-box;
  overflow: hidden;
  color: #fff;
  background:
    radial-gradient(circle at 50% 35%, color-mix(in srgb, var(--bg) 82%, #fff) 0%, transparent 65%),
    var(--bg);
}

.dj-title {
  margin: 0;
  color: #fff;
  font-size: inherit;
  font-weight: 700;
  letter-spacing: -0.02em;
  line-height: 1.1;
  white-space: nowrap;
  opacity: 0;
}

.dj-dots i {
  font-style: normal;
  opacity: 0;
  display: inline-block;
}

/* Escalier */
.dj-stairs {
  position: absolute;
  inset: 0;
  pointer-events: none;
  opacity: 0;
}

.dj-ground {
  position: absolute;
  left: 0;
  right: 0;
  bottom: 0;
  background: var(--step);
}

.dj-step {
  position: absolute;
  bottom: 0;
  background: var(--step);
}

.dj-person {
  position: absolute;
  left: 0;
  bottom: 0;
  will-change: transform;
}

/* Demi-tour : le personnage se retourne en douceur */
.dj-flip {
  width: 100%;
  height: 100%;
  transform-origin: 50% 50%;
  transition: transform 0.2s ease;
}

/* ───── Palette du personnage (nuances dérivées des couleurs passées en props) ───── */
.c-suit,
.c-suit-back,
.c-pants,
.c-pants-back,
.c-lapel {
  stroke: rgba(255, 255, 255, 0.16);
  stroke-width: 0.7;
}
.c-suit       { fill: var(--suit); }
.c-suit-back  { fill: color-mix(in srgb, var(--suit) 68%, #000); }
.c-pants      { fill: color-mix(in srgb, var(--suit) 86%, #000); }
.c-pants-back { fill: color-mix(in srgb, var(--suit) 62%, #000); }
.c-lapel      { fill: color-mix(in srgb, var(--suit) 80%, #fff); }
.c-tie        { fill: var(--tie); }
.c-skin       { fill: var(--skin); }
.c-skin-2     { fill: color-mix(in srgb, var(--skin) 84%, #b45309); }
.c-hair       { fill: var(--hair); }
.c-shoe       { fill: color-mix(in srgb, var(--suit) 30%, #000); }
.c-sole       { fill: #cbd5e1; }
.c-case       { fill: var(--case); stroke: #451a03; stroke-width: 0.7; }

.c-line {
  fill: none;
  stroke: #1e293b;
  stroke-width: 1.1;
  stroke-linecap: round;
  stroke-linejoin: round;
}
.c-line-thick { stroke-width: 1.5; }
.c-line-skin  { stroke: color-mix(in srgb, var(--skin) 70%, #92400e); stroke-width: 0.9; }

/* Détails animés du visage */
.dj-twinkle {
  transform-box: fill-box;
  transform-origin: center;
  animation: dj-twinkle 0.5s ease-in-out infinite alternate;
}
.dj-twinkle-2 { animation-delay: 0.25s; }

.dj-sweat {
  transform-box: fill-box;
  animation: dj-sweat 0.9s ease-in infinite;
}

@keyframes dj-twinkle {
  from { opacity: 0.5; scale: 0.7; }
  to   { opacity: 1;   scale: 1.15; }
}

@keyframes dj-sweat {
  0%   { opacity: 0; translate: 0 -2px; }
  30%  { opacity: 1; }
  100% { opacity: 0; translate: 0 9px; }
}

/* Entrée fluide du texte et de l'escalier */
.dj-on .dj-title {
  animation: dj-in 0.85s cubic-bezier(0.22, 1, 0.36, 1) forwards;
}

.dj-on .dj-stairs {
  animation: dj-fade 0.9s 0.25s cubic-bezier(0.22, 1, 0.36, 1) forwards;
}

.dj-on .dj-dots i {
  animation: var(--cycle) ease-in-out infinite;
}
.dj-on .dj-dots i:nth-child(1) { animation-name: dj-d1; }
.dj-on .dj-dots i:nth-child(2) { animation-name: dj-d2; }
.dj-on .dj-dots i:nth-child(3) { animation-name: dj-d3; }

@keyframes dj-in {
  from { opacity: 0; transform: translateY(20px); }
  to   { opacity: 1; transform: none; }
}

@keyframes dj-fade {
  from { opacity: 0; }
  to   { opacity: 1; }
}

@keyframes dj-d1 {
  0%, 15% { opacity: 0; transform: translateY(2px); }
  25%, 85% { opacity: 1; transform: translateY(0); }
  95%, 100% { opacity: 0; }
}
@keyframes dj-d2 {
  0%, 40% { opacity: 0; transform: translateY(2px); }
  50%, 85% { opacity: 1; transform: translateY(0); }
  95%, 100% { opacity: 0; }
}
@keyframes dj-d3 {
  0%, 65% { opacity: 0; transform: translateY(2px); }
  75%, 85% { opacity: 1; transform: translateY(0); }
  95%, 100% { opacity: 0; }
}
</style>