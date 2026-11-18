<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from 'vue'

const props = withDefaults(defineProps<{
  /** Hauteur de la scène (px) */
  height?: number
  /** Délai avant le départ de l'animation (s) */
  delay?: number
  /** Durée de l'approche des deux mains (s) */
  approachTime?: number
  /** Durée de la poignée de main / serrage (s) */
  shakeTime?: number
  /** Nombre d'oscillations pendant la poignée de main */
  shakes?: number
  /** Durée du décalage des mains et de l'apparition de CONTRATCHAP (s) */
  shiftTime?: number
  /** Durée de la montée du logo et de l'apparition de la phrase (s) */
  liftTime?: number
  /** Durée de l'arrivée de la souris sur "clic" (s) */
  cursorTime?: number
  /** Durée du clic et de l'effet visuel (s) */
  clickTime?: number
  /** Conservé pour rétrocompatibilité avec les appels existants (:bounces="5") */
  bounces?: number
}>(), {
  height: 420,
  delay: 0.25,
  approachTime: 0.75,
  shakeTime: 0.95,
  shakes: 2.5,
  shiftTime: 0.9,
  liftTime: 0.65,
  cursorTime: 0.7,
  clickTime: 0.95,
})

const emit = defineEmits<{ (e: 'done'): void }>()

// ───────── Math & Easing ─────────
const clamp = (v: number, a = 0, b = 1) => Math.min(b, Math.max(a, v))
const easeOutCubic = (u: number) => 1 - Math.pow(1 - u, 3)
const easeInOutCubic = (u: number) =>
  u < 0.5 ? 4 * u * u * u : 1 - Math.pow(-2 * u + 2, 3) / 2
const easeOutBack = (u: number, overshoot = 1.45) => {
  const c3 = overshoot + 1
  return 1 + c3 * Math.pow(u - 1, 3) + overshoot * Math.pow(u - 1, 2)
}

// ───────── Références DOM / SVG ─────────
const root = ref<HTMLElement>()
const logoWrap = ref<HTMLElement>()
const handshakeGroup = ref<SVGGElement>()
const leftHandGroup = ref<SVGGElement>()
const rightHandGroup = ref<SVGGElement>()
const rightFingersGroup = ref<SVGGElement>()
const textGroup = ref<SVGGElement>()
const textClipRect = ref<SVGRectElement>()
const pulseRing = ref<SVGCircleElement>()
const pulseFlash = ref<SVGCircleElement>()

const taglineEl = ref<HTMLElement>()
const clicWordEl = ref<HTMLElement>()
const clicBgEl = ref<HTMLElement>()
const cursorEl = ref<HTMLElement>()
const clickRippleEl = ref<HTMLElement>()
const clickSparksEl = ref<HTMLElement>()

let raf = 0
let observer: IntersectionObserver | null = null
let isPlaying = false

// Coordonnées dans le viewBox SVG (0 0 1245 145)
const ICON_CX = 145
const ICON_CY = 72
const STAGE_CX = 622.5
const CENTER_OFFSET_X = STAGE_CX - ICON_CX // +477.5 px pour centrer les mains au départ

const reduceMotion = () =>
  typeof window !== 'undefined' && window.matchMedia?.('(prefers-reduced-motion: reduce)').matches

function applyLogoState(opts: {
  groupX: number
  groupY: number
  groupScale: number
  groupRotate: number
  leftX: number
  leftY: number
  leftRotate: number
  leftOpacity: number
  rightX: number
  rightY: number
  rightRotate: number
  rightOpacity: number
  fingersScale: number
  textX: number
  textOpacity: number
  textClipWidth: number
  flashOpacity: number
  flashScale: number
  ringOpacity: number
  ringScale: number
  logoLiftY: number
  logoLiftScale: number
}) {
  if (logoWrap.value) {
    logoWrap.value.style.transform =
      `translate3d(0, ${opts.logoLiftY.toFixed(2)}px, 0) scale(${opts.logoLiftScale.toFixed(4)})`
  }
  if (handshakeGroup.value) {
    handshakeGroup.value.setAttribute(
      'transform',
      `translate(${opts.groupX.toFixed(2)} ${opts.groupY.toFixed(2)}) ` +
      `translate(${ICON_CX} ${ICON_CY}) scale(${opts.groupScale.toFixed(4)}) rotate(${opts.groupRotate.toFixed(2)}) translate(${-ICON_CX} ${-ICON_CY})`,
    )
  }
  if (leftHandGroup.value) {
    leftHandGroup.value.style.opacity = opts.leftOpacity.toFixed(3)
    leftHandGroup.value.setAttribute(
      'transform',
      `translate(${opts.leftX.toFixed(2)} ${opts.leftY.toFixed(2)}) ` +
      `translate(45 55) rotate(${opts.leftRotate.toFixed(2)}) translate(-45 -55)`,
    )
  }
  if (rightHandGroup.value) {
    rightHandGroup.value.style.opacity = opts.rightOpacity.toFixed(3)
    rightHandGroup.value.setAttribute(
      'transform',
      `translate(${opts.rightX.toFixed(2)} ${opts.rightY.toFixed(2)}) ` +
      `translate(245 55) rotate(${opts.rightRotate.toFixed(2)}) translate(-245 -55)`,
    )
  }
  if (rightFingersGroup.value) {
    rightFingersGroup.value.setAttribute(
      'transform',
      `translate(96 110) scale(${opts.fingersScale.toFixed(3)}) translate(-96 -110)`,
    )
  }
  if (textGroup.value) {
    textGroup.value.style.opacity = opts.textOpacity.toFixed(3)
    textGroup.value.setAttribute('transform', `translate(${opts.textX.toFixed(2)} 0)`)
  }
  if (textClipRect.value) {
    textClipRect.value.setAttribute('width', Math.max(0, opts.textClipWidth).toFixed(1))
  }
  if (pulseFlash.value) {
    pulseFlash.value.style.opacity = opts.flashOpacity.toFixed(3)
    pulseFlash.value.setAttribute(
      'transform',
      `translate(${ICON_CX} ${ICON_CY}) scale(${opts.flashScale.toFixed(3)}) translate(${-ICON_CX} ${-ICON_CY})`,
    )
  }
  if (pulseRing.value) {
    pulseRing.value.style.opacity = opts.ringOpacity.toFixed(3)
    pulseRing.value.setAttribute(
      'transform',
      `translate(${ICON_CX} ${ICON_CY}) scale(${opts.ringScale.toFixed(3)}) translate(${-ICON_CX} ${-ICON_CY})`,
    )
  }
}

function applyTaglineState(opts: {
  taglineOpacity: number
  taglineY: number
  clicScale: number
  clicAccent: number
  cursorOpacity: number
  cursorX: number
  cursorY: number
  cursorScale: number
  cursorRotate: number
  rippleOpacity: number
  rippleScale: number
  sparksOpacity: number
  sparksScale: number
}) {
  if (taglineEl.value) {
    taglineEl.value.style.opacity = opts.taglineOpacity.toFixed(3)
    taglineEl.value.style.transform = `translate3d(0, ${opts.taglineY.toFixed(2)}px, 0)`
  }
  if (clicWordEl.value) {
    clicWordEl.value.style.transform = `scale(${opts.clicScale.toFixed(4)})`
    clicWordEl.value.style.color = opts.clicAccent > 0.5 ? '#ffffff' : '#0454b8'
  }
  if (clicBgEl.value) {
    clicBgEl.value.style.opacity = opts.clicAccent.toFixed(3)
    clicBgEl.value.style.transform = `scale(${(0.85 + 0.15 * opts.clicAccent).toFixed(3)})`
  }
  if (cursorEl.value) {
    cursorEl.value.style.opacity = opts.cursorOpacity.toFixed(3)
    cursorEl.value.style.transform =
      `translate3d(${opts.cursorX.toFixed(2)}px, ${opts.cursorY.toFixed(2)}px, 0) ` +
      `scale(${opts.cursorScale.toFixed(3)}) rotate(${opts.cursorRotate.toFixed(2)}deg)`
  }
  if (clickRippleEl.value) {
    clickRippleEl.value.style.opacity = opts.rippleOpacity.toFixed(3)
    clickRippleEl.value.style.transform = `translate(-50%, -50%) scale(${opts.rippleScale.toFixed(3)})`
  }
  if (clickSparksEl.value) {
    clickSparksEl.value.style.opacity = opts.sparksOpacity.toFixed(3)
    clickSparksEl.value.style.transform = `translate(-50%, -50%) scale(${opts.sparksScale.toFixed(3)})`
  }
}

function reset() {
  cancelAnimationFrame(raf)
  isPlaying = false
  applyLogoState({
    groupX: CENTER_OFFSET_X,
    groupY: 0,
    groupScale: 1.35,
    groupRotate: 0,
    leftX: -150,
    leftY: -18,
    leftRotate: -16,
    leftOpacity: 0,
    rightX: 150,
    rightY: -18,
    rightRotate: 16,
    rightOpacity: 0,
    fingersScale: 0.75,
    textX: -80,
    textOpacity: 0,
    textClipWidth: 0,
    flashOpacity: 0,
    flashScale: 0.3,
    ringOpacity: 0,
    ringScale: 0.4,
    logoLiftY: 0,
    logoLiftScale: 1,
  })
  applyTaglineState({
    taglineOpacity: 0,
    taglineY: 66,
    clicScale: 1,
    clicAccent: 0,
    cursorOpacity: 0,
    cursorX: 120,
    cursorY: 80,
    cursorScale: 1,
    cursorRotate: 8,
    rippleOpacity: 0,
    rippleScale: 0.2,
    sparksOpacity: 0,
    sparksScale: 0.4,
  })
}

function showFinal() {
  cancelAnimationFrame(raf)
  isPlaying = false
  applyLogoState({
    groupX: 0,
    groupY: 0,
    groupScale: 1,
    groupRotate: 0,
    leftX: 0,
    leftY: 0,
    leftRotate: 0,
    leftOpacity: 1,
    rightX: 0,
    rightY: 0,
    rightRotate: 0,
    rightOpacity: 1,
    fingersScale: 1,
    textX: 0,
    textOpacity: 1,
    textClipWidth: 960,
    flashOpacity: 0,
    flashScale: 1,
    ringOpacity: 0,
    ringScale: 1.5,
    logoLiftY: -42,
    logoLiftScale: 0.92,
  })
  applyTaglineState({
    taglineOpacity: 1,
    taglineY: 46,
    clicScale: 1,
    clicAccent: 1,
    cursorOpacity: 1,
    cursorX: 8,
    cursorY: 6,
    cursorScale: 1,
    cursorRotate: 0,
    rippleOpacity: 0,
    rippleScale: 1,
    sparksOpacity: 0,
    sparksScale: 1,
  })
}

function play() {
  if (!root.value) return
  if (reduceMotion()) {
    showFinal()
    emit('done')
    return
  }

  cancelAnimationFrame(raf)
  isPlaying = true

  const t1 = props.approachTime
  const t2 = t1 + props.shakeTime
  const t3 = t2 + props.shiftTime
  const t4 = t3 + props.liftTime
  const t5 = t4 + props.cursorTime
  const tEnd = t5 + props.clickTime

  let start = 0
  let doneEmitted = false

  const tick = (now: number) => {
    if (!start) start = now + props.delay * 1000
    const t = Math.max(0, (now - start) / 1000)

    if (t < t1) {
      // Phase 1 : Les deux mains apparaissent au centre et s'avancent pour se serrer la main
      const u = clamp(t / props.approachTime)
      const op = easeOutCubic(clamp(u / 0.45))
      const move = easeOutBack(u, 1.35)
      const fingerSnap = easeOutBack(clamp((u - 0.55) / 0.45), 1.8)

      applyLogoState({
        groupX: CENTER_OFFSET_X,
        groupY: 0,
        groupScale: 1.35,
        groupRotate: 0,
        leftX: -150 * (1 - move),
        leftY: -18 * (1 - move),
        leftRotate: -16 * (1 - move),
        leftOpacity: op,
        rightX: 150 * (1 - move),
        rightY: -18 * (1 - move),
        rightRotate: 16 * (1 - move),
        rightOpacity: op,
        fingersScale: 0.75 + 0.25 * fingerSnap,
        textX: -80,
        textOpacity: 0,
        textClipWidth: 0,
        flashOpacity: 0,
        flashScale: 0.3,
        ringOpacity: 0,
        ringScale: 0.4,
        logoLiftY: 0,
        logoLiftScale: 1,
      })
      applyTaglineState({
        taglineOpacity: 0,
        taglineY: 66,
        clicScale: 1,
        clicAccent: 0,
        cursorOpacity: 0,
        cursorX: 120,
        cursorY: 80,
        cursorScale: 1,
        cursorRotate: 8,
        rippleOpacity: 0,
        rippleScale: 0.2,
        sparksOpacity: 0,
        sparksScale: 0.4,
      })
    } else if (t < t2) {
      // Phase 2 : Poignée de main (oscillation + onde d'impact)
      const u = clamp((t - t1) / props.shakeTime)
      const damp = Math.pow(1 - u, 1.35)
      const wave = Math.sin(u * Math.PI * 2 * props.shakes)
      const shakeY = wave * 14 * damp
      const shakeRot = wave * 5.5 * damp
      const squeeze = 1 + 0.06 * Math.sin(u * Math.PI)

      const pulseU = clamp(u / 0.55)
      const flashOp = pulseU < 0.25 ? pulseU / 0.25 : Math.max(0, 1 - (pulseU - 0.25) / 0.75)
      const ringOp = Math.max(0, (1 - pulseU) * 0.75)

      applyLogoState({
        groupX: CENTER_OFFSET_X,
        groupY: shakeY,
        groupScale: 1.35 * squeeze,
        groupRotate: shakeRot,
        leftX: 0,
        leftY: 0,
        leftRotate: 0,
        leftOpacity: 1,
        rightX: 0,
        rightY: 0,
        rightRotate: 0,
        rightOpacity: 1,
        fingersScale: 1,
        textX: -80,
        textOpacity: 0,
        textClipWidth: 0,
        flashOpacity: flashOp * 0.55,
        flashScale: 0.5 + 1.1 * easeOutCubic(pulseU),
        ringOpacity: ringOp,
        ringScale: 0.6 + 1.35 * easeOutCubic(pulseU),
        logoLiftY: 0,
        logoLiftScale: 1,
      })
    } else if (t < t3) {
      // Phase 3 : Les mains se décalent vers la gauche et laissent place à "CONTRATCHAP"
      const u = clamp((t - t2) / props.shiftTime)
      const shift = easeInOutCubic(u)
      const textU = clamp((u - 0.12) / 0.88)
      const textEase = easeOutCubic(textU)

      applyLogoState({
        groupX: CENTER_OFFSET_X * (1 - shift),
        groupY: 0,
        groupScale: 1.35 - 0.35 * shift,
        groupRotate: 0,
        leftX: 0,
        leftY: 0,
        leftRotate: 0,
        leftOpacity: 1,
        rightX: 0,
        rightY: 0,
        rightRotate: 0,
        rightOpacity: 1,
        fingersScale: 1,
        textX: -65 * (1 - textEase),
        textOpacity: clamp(textU / 0.35),
        textClipWidth: 960 * textEase,
        flashOpacity: 0,
        flashScale: 1,
        ringOpacity: 0,
        ringScale: 1.5,
        logoLiftY: 0,
        logoLiftScale: 1,
      })
    } else if (t < t4) {
      // Phase 4 : Le logo monte légèrement et la phrase "Sécuriser votre business en un clic" apparaît
      const u = clamp((t - t3) / props.liftTime)
      const lift = easeInOutCubic(u)
      const tagU = clamp((u - 0.25) / 0.75)
      const tagEase = easeOutCubic(tagU)

      applyLogoState({
        groupX: 0,
        groupY: 0,
        groupScale: 1,
        groupRotate: 0,
        leftX: 0,
        leftY: 0,
        leftRotate: 0,
        leftOpacity: 1,
        rightX: 0,
        rightY: 0,
        rightRotate: 0,
        rightOpacity: 1,
        fingersScale: 1,
        textX: 0,
        textOpacity: 1,
        textClipWidth: 960,
        flashOpacity: 0,
        flashScale: 1,
        ringOpacity: 0,
        ringScale: 1.5,
        logoLiftY: -42 * lift,
        logoLiftScale: 1 - 0.08 * lift,
      })
      applyTaglineState({
        taglineOpacity: tagEase,
        taglineY: 66 - 20 * tagEase,
        clicScale: 1,
        clicAccent: 0,
        cursorOpacity: 0,
        cursorX: 120,
        cursorY: 80,
        cursorScale: 1,
        cursorRotate: 8,
        rippleOpacity: 0,
        rippleScale: 0.2,
        sparksOpacity: 0,
        sparksScale: 0.4,
      })
    } else if (t < t5) {
      // Phase 5 : La souris arrive et se positionne sur le mot "clic"
      const u = clamp((t - t4) / props.cursorTime)
      const move = easeInOutCubic(u)
      const op = easeOutCubic(clamp(u / 0.25))
      // Légère courbe naturelle dans la trajectoire
      const arcY = -18 * Math.sin(u * Math.PI)
      const hoverScale = 1 + 0.05 * easeOutCubic(clamp((u - 0.7) / 0.3))

      applyLogoState({
        groupX: 0,
        groupY: 0,
        groupScale: 1,
        groupRotate: 0,
        leftX: 0,
        leftY: 0,
        leftRotate: 0,
        leftOpacity: 1,
        rightX: 0,
        rightY: 0,
        rightRotate: 0,
        rightOpacity: 1,
        fingersScale: 1,
        textX: 0,
        textOpacity: 1,
        textClipWidth: 960,
        flashOpacity: 0,
        flashScale: 1,
        ringOpacity: 0,
        ringScale: 1.5,
        logoLiftY: -42,
        logoLiftScale: 0.92,
      })
      applyTaglineState({
        taglineOpacity: 1,
        taglineY: 46,
        clicScale: hoverScale,
        clicAccent: 0,
        cursorOpacity: op,
        cursorX: 120 * (1 - move),
        cursorY: 80 * (1 - move) + arcY,
        cursorScale: 1,
        cursorRotate: 8 * (1 - move),
        rippleOpacity: 0,
        rippleScale: 0.2,
        sparksOpacity: 0,
        sparksScale: 0.4,
      })
    } else if (t < tEnd) {
      // Phase 6 : Effet de clic sur "clic" (pression, onde de choc, accentuation)
      const u = clamp((t - t5) / props.clickTime)

      // Pression (0 -> 0.22) puis relâchement élastique (0.22 -> 0.58)
      let clicScale = 1
      let cursorScale = 1
      let cursorRot = 0
      if (u < 0.22) {
        const p = easeOutCubic(u / 0.22)
        clicScale = 1.05 - 0.16 * p // s'enfonce à 0.89
        cursorScale = 1 - 0.18 * p
        cursorRot = -6 * p
      } else if (u < 0.58) {
        const p = easeOutBack((u - 0.22) / 0.36, 1.8)
        clicScale = 0.89 + 0.11 * p
        cursorScale = 0.82 + 0.18 * p
        cursorRot = -6 * (1 - clamp((u - 0.22) / 0.36))
      }

      // Accentuation du mot "clic" dès l'impact
      const accent = u < 0.18 ? 0 : easeOutCubic(clamp((u - 0.18) / 0.22))

      // Onde de clic et étincelles à partir de l'impact (u = 0.20)
      const fxU = clamp((u - 0.2) / 0.65)
      const fxEase = easeOutCubic(fxU)
      const rippleOp = u < 0.2 ? 0 : Math.max(0, (1 - fxU) * 0.9)
      const sparksOp = u < 0.2 ? 0 : Math.max(0, (1 - fxU) * 0.95)

      applyTaglineState({
        taglineOpacity: 1,
        taglineY: 46,
        clicScale,
        clicAccent: accent,
        cursorOpacity: 1,
        cursorX: 0,
        cursorY: 0,
        cursorScale,
        cursorRotate: cursorRot,
        rippleOpacity: rippleOp,
        rippleScale: 0.25 + 1.55 * fxEase,
        sparksOpacity: sparksOp,
        sparksScale: 0.45 + 0.95 * fxEase,
      })
    } else {
      // Phase 7 : La souris reste, flotte autour de "clic" et clique toutes les 5 secondes
      if (!doneEmitted) {
        doneEmitted = true
        isPlaying = false
        emit('done')
      }
      const dt = t - tEnd
      const PERIOD = 5.0
      const cycle = dt % PERIOD

      // Enveloppe qui ramène doucement le curseur sur (0, 0) avant chaque clic
      let floatWeight = 1
      if (cycle >= 4.1 && cycle < 4.38) {
        floatWeight = 1 - easeInOutCubic((cycle - 4.1) / 0.28)
      } else if (cycle >= 4.38 && cycle < 4.82) {
        floatWeight = 0
      } else if (cycle >= 4.82) {
        floatWeight = easeInOutCubic((cycle - 4.82) / 0.18)
      }

      const floatX = 12 * Math.sin(dt * 1.7) * floatWeight
      const floatY = 7 * Math.sin(dt * 2.3) * floatWeight
      const floatRot = 4.5 * Math.sin(dt * 1.3) * floatWeight

      // Clic périodique dans la fenêtre [4.38s -> 4.98s]
      let clicScale = 1
      let cursorScale = 1
      let clickRot = 0
      let rippleOp = 0
      let rippleScale = 0.25
      let sparksOp = 0
      let sparksScale = 0.45

      if (cycle >= 4.38 && cycle < 4.98) {
        const cu = clamp((cycle - 4.38) / 0.6)
        if (cu < 0.22) {
          const p = easeOutCubic(cu / 0.22)
          clicScale = 1 - 0.11 * p
          cursorScale = 1 - 0.18 * p
          clickRot = -6 * p
        } else if (cu < 0.58) {
          const p = easeOutBack((cu - 0.22) / 0.36, 1.8)
          clicScale = 0.89 + 0.11 * p
          cursorScale = 0.82 + 0.18 * p
          clickRot = -6 * (1 - clamp((cu - 0.22) / 0.36))
        }

        if (cu >= 0.2) {
          const fxU = clamp((cu - 0.2) / 0.72)
          const fxEase = easeOutCubic(fxU)
          rippleOp = Math.max(0, (1 - fxU) * 0.9)
          rippleScale = 0.25 + 1.55 * fxEase
          sparksOp = Math.max(0, (1 - fxU) * 0.95)
          sparksScale = 0.45 + 0.95 * fxEase
        }
      }

      applyTaglineState({
        taglineOpacity: 1,
        taglineY: 46,
        clicScale,
        clicAccent: 1,
        cursorOpacity: 1,
        cursorX: floatX,
        cursorY: floatY,
        cursorScale,
        cursorRotate: floatRot + clickRot,
        rippleOpacity: rippleOp,
        rippleScale,
        sparksOpacity: sparksOp,
        sparksScale,
      })
    }

    raf = requestAnimationFrame(tick)
  }

  raf = requestAnimationFrame(tick)
}

onMounted(() => {
  reset()
  // Slidev garde les slides montées : on rejoue l'animation quand la slide devient visible
  observer = new IntersectionObserver(([entry]) => {
    if (entry.isIntersecting && !isPlaying) play()
    else if (!entry.isIntersecting) reset()
  }, { threshold: 0.5 })
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
    class="cc-intro-stage"
    :style="{ height: `${height}px` }"
    @click="play"
  >
    <!-- Logo vectoriel SVG (généré depuis assets/svg/LOGO_CONTRATCHAP.svg) -->
    <div ref="logoWrap" class="cc-logo-wrap">
      <svg
        class="cc-logo-svg"
        xmlns="http://www.w3.org/2000/svg"
        viewBox="0 0 1245 145"
        fill="none"
        role="img"
        aria-label="Logo Contratchap"
      >
        <defs>
          <linearGradient id="cc-green-grad" x1="0%" y1="0%" x2="0%" y2="100%">
            <stop offset="0%" stop-color="#9fcf31" />
            <stop offset="100%" stop-color="#6fa411" />
          </linearGradient>
          <linearGradient id="cc-blue-grad" x1="0%" y1="0%" x2="0%" y2="100%">
            <stop offset="0%" stop-color="#1880ec" />
            <stop offset="100%" stop-color="#0b58b9" />
          </linearGradient>
          <radialGradient id="cc-handshake-glow" cx="50%" cy="50%" r="50%">
            <stop offset="0%" stop-color="#1880ec" stop-opacity="0.45" />
            <stop offset="55%" stop-color="#84b80f" stop-opacity="0.2" />
            <stop offset="100%" stop-color="#84b80f" stop-opacity="0" />
          </radialGradient>
          <clipPath id="cc-text-reveal-clip">
            <rect ref="textClipRect" x="295" y="0" width="960" height="145" />
          </clipPath>
        </defs>

        <!-- Texte : CONTRATCHAP (révélé lorsque les mains se décalent) -->
        <g clip-path="url(#cc-text-reveal-clip)">
          <g ref="textGroup" id="logo-text" fill-rule="evenodd">
            <!-- CONTRAT (bleu) -->
            <g id="text-contrat" fill="#0454b8">
              <path d="M339.7,24.1L353.0,23.9L363.9,26.7L372.4,31.1L376.9,34.6L380.1,38.2L379.4,39.9L365.7,52.1L364.4,52.1L359.1,47.4L353.8,44.8L343.0,44.4L338.7,46.1L335.1,48.6L333.2,50.8L330.8,55.2L329.6,60.0L330.0,69.6L332.6,75.9L336.4,80.1L342.0,82.8L350.0,83.5L357.7,81.1L365.0,75.4L380.0,88.4L379.1,90.6L370.2,97.8L364.1,100.8L355.0,103.4L346.0,104.0L338.1,103.2L326.9,99.8L321.0,96.5L314.9,91.6L310.0,85.5L306.6,78.9L304.4,71.0L303.8,64.0L304.5,56.0L306.8,48.2L311.1,40.6L315.8,35.3L321.9,30.4L329.2,26.8Z M421.6,24.0L434.8,23.9L445.8,26.7L452.5,30.0L458.8,34.7L464.0,40.4L468.1,47.3L471.1,56.2L471.5,66.0L471.0,71.6L468.3,79.9L464.3,86.8L458.6,93.1L451.6,98.1L442.4,102.0L436.0,103.4L428.0,103.9L420.0,103.3L411.0,100.7L404.1,97.4L396.7,91.8L392.1,86.4L388.1,79.3L385.6,71.0L385.2,64.0L385.8,56.1L388.4,48.0L392.9,40.3L397.0,35.5L404.4,29.9L411.2,26.8ZM423.1,44.7L416.7,48.2L412.5,54.1L410.9,59.1L411.2,70.0L414.1,76.4L419.2,81.2L427.0,83.6L434.0,82.7L438.8,80.4L442.3,76.8L444.2,73.8L445.6,70.0L446.3,64.0L445.4,57.0L442.1,50.3L437.2,46.2L432.0,44.3Z M484.5,26.1L503.0,25.7L505.3,26.2L532.8,58.7L534.0,59.3L534.8,59.2L535.4,58.0L535.8,26.2L538.0,25.7L557.0,25.7L559.4,26.2L559.9,28.0L560.0,100.0L559.4,101.8L541.0,102.3L538.9,101.7L511.9,69.6L510.0,68.4L509.2,68.6L508.7,70.0L508.9,100.0L508.3,101.8L506.0,102.3L484.5,101.9L483.9,100.0L483.9,28.0Z M570.3,25.9L638.0,25.7L639.4,26.2L639.9,28.0L639.6,43.9L638.0,44.6L620.0,44.6L618.3,45.0L617.7,46.0L617.7,100.0L616.8,102.2L594.0,102.3L592.7,101.8L592.3,46.1L591.6,45.0L590.0,44.6L570.1,44.4L569.4,43.0L569.4,28.0Z M650.6,26.2L652.0,25.7L690.0,25.7L698.0,26.6L708.4,30.1L715.6,35.9L719.0,40.6L721.1,45.2L722.4,52.0L722.2,57.0L720.9,63.3L717.9,69.4L715.3,72.8L709.1,77.6L708.2,79.0L722.3,100.2L722.9,102.0L697.4,102.0L695.0,99.6L684.1,83.4L682.9,82.7L679.0,82.5L676.4,83.0L675.7,84.1L675.6,100.0L674.8,102.1L650.6,101.8L650.1,100.0L650.1,29.0ZM677.1,44.7L676.0,45.4L675.5,47.0L675.5,61.0L676.2,63.3L678.0,63.8L687.0,63.7L690.8,63.1L694.3,60.8L696.3,56.9L696.7,54.0L696.1,50.3L694.0,47.5L691.7,45.9L688.0,44.8Z M759.7,26.2L783.0,25.7L784.9,26.5L817.3,100.1L817.4,101.8L792.5,102.0L787.1,89.3L781.0,88.7L757.2,88.9L750.8,102.2L728.0,102.3L726.7,101.8L726.7,100.1ZM772.0,52.0L770.4,53.0L765.7,65.1L764.6,69.0L765.2,70.4L777.0,70.7L779.1,69.8L773.3,53.1Z M816.5,26.0L884.0,25.6L886.0,26.5L885.8,44.2L866.0,44.6L864.1,45.7L864.1,100.0L863.5,101.9L841.0,102.3L838.9,101.8L838.4,46.0L837.8,44.9L836.0,44.6L816.2,44.3L815.6,43.0L815.7,28.0Z" />
            </g>
            <!-- CHAP (vert) -->
            <g id="text-chap" fill="#54a50a">
              <path d="M928.5,24.0L941.8,23.9L952.8,26.8L960.0,30.5L965.7,34.9L968.5,38.0L967.7,40.1L953.8,52.2L947.3,47.1L942.0,44.6L932.0,44.3L927.3,45.9L924.3,47.9L921.9,50.4L919.7,54.2L918.1,60.0L918.5,70.0L921.2,76.2L925.2,80.3L930.7,82.9L936.0,83.6L941.4,83.0L946.5,81.0L952.5,76.0L953.9,75.5L968.4,88.2L967.9,90.4L959.8,97.2L953.0,100.6L943.0,103.5L935.0,103.9L927.0,103.2L917.9,100.8L909.2,96.2L903.2,91.4L898.8,85.8L895.3,79.1L892.9,70.9L892.4,64.0L892.9,56.2L895.5,48.0L899.9,40.3L905.1,34.5L911.4,29.9L918.1,26.7Z M979.5,26.2L1003.6,26.0L1004.3,28.0L1004.3,51.0L1005.0,52.6L1007.0,53.2L1026.0,53.2L1028.8,52.4L1029.4,51.0L1029.8,26.2L1032.0,25.7L1054.3,26.2L1054.4,101.8L1030.2,102.1L1029.4,100.0L1029.1,74.3L1027.0,73.4L1007.0,73.3L1004.7,74.2L1004.3,76.0L1004.0,101.6L1002.0,102.2L979.6,101.9L978.9,100.0L978.9,28.0Z M1095.7,26.2L1119.0,25.7L1120.9,26.5L1153.1,99.3L1153.4,101.8L1129.1,102.2L1127.8,100.8L1123.2,89.2L1121.0,88.6L1093.2,88.8L1092.0,90.5L1088.1,100.8L1086.8,102.1L1062.9,101.8L1062.9,100.2ZM1108.0,52.0L1106.5,53.1L1105.6,55.1L1100.8,68.2L1100.8,69.8L1103.0,70.7L1114.0,70.7L1115.2,69.0L1114.2,65.2L1109.5,53.1Z M1161.7,26.2L1163.0,25.7L1203.0,25.8L1213.8,27.9L1220.4,31.0L1225.5,35.0L1229.9,40.7L1234.1,51.1L1234.0,57.7L1228.9,69.3L1225.7,73.2L1221.4,76.9L1212.7,81.1L1207.0,82.4L1189.0,82.8L1187.0,83.4L1186.5,85.0L1186.5,100.0L1185.8,102.1L1162.2,102.2L1161.1,100.0L1161.1,28.0ZM1188.1,44.7L1187.0,45.5L1186.6,47.0L1186.6,61.0L1187.2,63.3L1198.0,63.7L1201.8,63.1L1205.4,60.8L1207.3,57.0L1207.6,54.0L1206.8,49.8L1205.0,47.4L1202.8,45.8L1198.9,44.8Z" />
            </g>
          </g>
        </g>

        <!-- Icône : Poignée de main -->
        <g ref="handshakeGroup" id="handshake">
          <!-- Halo / onde de contact lors du serrage de main -->
          <circle
            ref="pulseFlash"
            :cx="ICON_CX"
            :cy="ICON_CY"
            r="110"
            fill="url(#cc-handshake-glow)"
            style="opacity: 0; pointer-events: none"
          />
          <circle
            ref="pulseRing"
            :cx="ICON_CX"
            :cy="ICON_CY"
            r="85"
            stroke="#1880ec"
            stroke-width="2.5"
            fill="none"
            style="opacity: 0; pointer-events: none"
          />

          <!-- Main gauche (verte) -->
          <g ref="leftHandGroup" id="left-hand" fill="url(#cc-green-grad)" fill-rule="evenodd">
            <path id="left-cuff" d="M51.3,5.9L54.9,6.7L77.8,18.6L79.2,20.2L79.7,23.0L43.4,102.9L41.4,104.9L38.0,105.8L34.0,104.4L10.1,92.5L7.5,90.0L6.7,87.0L22.6,54.1L46.2,9.8L48.5,7.0ZM19.2,77.7L17.0,79.5L16.0,81.5L16.2,86.2L19.2,89.1L22.0,89.6L24.6,89.0L26.8,87.2L27.9,85.2L27.6,80.9L26.4,79.1L24.0,77.6L22.0,77.2Z" />
            <path id="left-palm" d="M109.5,15.0L116.2,15.0L106.4,26.0L102.7,32.2L101.8,35.1L101.2,39.0L101.8,43.9L103.0,46.5L106.6,50.0L112.3,52.0L118.9,51.3L125.6,47.1L134.8,37.2L136.6,36.0L143.0,35.3L149.0,36.6L154.6,39.9L198.5,78.0L203.6,82.9L205.4,86.0L205.6,91.0L203.0,95.5L199.0,97.8L194.0,97.5L189.7,94.8L165.0,73.5L163.0,72.7L161.7,73.2L160.7,75.0L161.3,77.2L185.4,98.2L187.2,102.0L187.1,105.7L185.3,108.8L182.7,111.1L179.9,112.2L176.5,112.0L172.7,109.8L152.8,92.8L151.0,92.2L149.2,92.7L148.5,94.0L149.2,96.3L166.8,111.7L168.3,115.0L168.3,118.0L167.2,120.8L164.8,123.4L161.3,124.9L158.7,125.0L155.1,123.5L138.3,109.1L137.0,108.5L135.2,108.7L134.3,111.0L135.0,112.6L147.9,123.6L149.9,127.7L149.9,130.3L148.2,133.7L145.4,136.0L142.6,137.0L137.6,136.0L133.1,132.4L132.6,131.0L133.5,128.0L133.7,124.0L132.5,120.1L127.9,115.7L121.4,113.9L120.0,108.4L115.9,103.6L111.0,101.5L105.1,101.4L102.4,95.1L99.6,91.9L95.0,89.4L87.5,88.8L85.0,82.5L82.6,79.9L79.5,78.0L73.0,77.2L69.7,78.1L65.0,81.0L64.1,79.3L63.5,75.0L64.8,71.2L81.1,35.6L85.7,27.2L89.1,23.5L93.2,20.8L101.0,17.4Z" />
          </g>

          <!-- Main droite (bleue) -->
          <g ref="rightHandGroup" id="right-hand" fill="url(#cc-blue-grad)" fill-rule="evenodd">
            <path id="right-palm" d="M145.4,9.0L152.0,8.8L161.8,10.9L168.8,13.9L182.6,22.0L188.1,24.2L204.9,26.5L228.2,77.2L229.4,80.9L228.0,81.3L224.0,80.8L210.0,78.4L205.7,76.9L157.8,35.6L152.9,32.7L147.8,30.8L140.2,29.9L133.4,32.0L124.8,41.4L117.8,46.1L114.0,47.0L111.0,46.4L108.5,45.0L106.3,41.0L106.9,35.3L111.5,28.0L117.6,21.1L122.1,17.6L130.7,13.1L137.1,10.8Z" />
            <path id="right-cuff" d="M237.3,5.0L241.6,5.9L245.1,10.3L272.0,62.4L283.2,86.0L282.6,89.1L280.8,91.2L251.9,105.1L248.5,104.0L246.6,101.9L210.0,22.0L210.7,19.2L212.8,17.1Z" />
            <g ref="rightFingersGroup" id="right-fingers">
              <path d="M73.3,82.0L77.0,82.5L80.0,84.4L82.0,87.4L82.6,91.0L81.5,93.9L76.2,101.7L73.9,103.4L70.0,104.3L66.3,103.1L64.1,101.3L62.5,98.0L62.5,95.0L67.9,85.3L70.5,83.0Z" />
              <path d="M88.7,94.1L92.0,93.7L97.1,96.4L99.3,100.1L99.4,104.0L91.2,116.8L87.5,119.0L84.1,119.2L79.1,116.4L77.6,113.9L77.0,110.3L78.5,106.1L84.6,97.1Z" />
              <path d="M104.7,107.1L109.0,106.2L112.9,107.6L116.0,111.5L116.5,116.0L115.3,118.8L108.6,129.1L105.8,131.2L102.0,131.9L98.7,130.9L96.0,128.6L94.6,126.0L94.4,122.0L95.9,118.4L101.5,110.0Z" />
              <path d="M120.4,120.0L123.0,119.4L126.7,120.9L128.8,123.8L129.3,127.0L128.0,130.5L124.2,135.8L121.9,137.4L119.1,138.2L114.6,135.8L113.1,133.2L112.8,131.1L116.1,124.7L118.6,121.2Z" />
            </g>
          </g>
        </g>
      </svg>
    </div>

    <!-- Phrase d'accroche avec accentuation et effet de clic sur "clic" -->
    <p ref="taglineEl" class="cc-tagline">
      <span>Sécuriser votre business en un </span>
      <span ref="clicWordEl" class="cc-clic-target">
        <span ref="clicBgEl" class="cc-clic-pill" aria-hidden="true" />
        <span class="cc-clic-label">clic</span>

        <!-- Onde de clic et étincelles centrées sur le mot "clic" -->
        <span ref="clickRippleEl" class="cc-click-ripple" aria-hidden="true" />
        <svg
          ref="clickSparksEl"
          class="cc-click-sparks"
          viewBox="0 0 64 64"
          fill="none"
          aria-hidden="true"
        >
          <line x1="32" y1="4" x2="32" y2="14" stroke="#6fa411" stroke-width="3" stroke-linecap="round" />
          <line x1="32" y1="50" x2="32" y2="60" stroke="#1880ec" stroke-width="3" stroke-linecap="round" />
          <line x1="4" y1="32" x2="14" y2="32" stroke="#1880ec" stroke-width="3" stroke-linecap="round" />
          <line x1="50" y1="32" x2="60" y2="32" stroke="#6fa411" stroke-width="3" stroke-linecap="round" />
          <line x1="12" y1="12" x2="19" y2="19" stroke="#6fa411" stroke-width="2.5" stroke-linecap="round" />
          <line x1="45" y1="45" x2="52" y2="52" stroke="#1880ec" stroke-width="2.5" stroke-linecap="round" />
          <line x1="52" y1="12" x2="45" y2="19" stroke="#1880ec" stroke-width="2.5" stroke-linecap="round" />
          <line x1="19" y1="45" x2="12" y2="52" stroke="#6fa411" stroke-width="2.5" stroke-linecap="round" />
        </svg>

        <!-- Curseur de souris venant cliquer sur "clic" -->
        <span ref="cursorEl" class="cc-mouse-cursor" aria-hidden="true">
          <svg viewBox="0 0 28 32" width="20" height="34" fill="none">
            <path
              d="M4 2.5L23.5 17.5L14.5 18.8L18.8 28.2L14.8 30L10.5 20.4L4 25.8V2.5Z"
              fill="#ffffff"
              stroke="#0f172a"
              stroke-width="1"
              stroke-linejoin="round"
            />
          </svg>
        </span>
      </span>
    </p>
  </div>
</template>

<style scoped>
.cc-intro-stage {
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
  min-width: 860px;
  padding: 0 2rem;
  overflow: hidden;
  user-select: none;
  cursor: pointer;
}

.cc-logo-wrap {
  will-change: transform;
  transform-origin: center center;
}

.cc-logo-svg {
  width: min(88vw, 840px);
  height: auto;
  overflow: visible;
  filter: drop-shadow(0 8px 24px rgba(4, 84, 184, 0.08));
}

.cc-tagline {
  position: absolute;
  margin: 0;
  font-size: clamp(1.45rem, 2.6vw, 2.05rem);
  font-weight: 600;
  letter-spacing: -0.015em;
  color: #334155;
  white-space: nowrap;
  opacity: 0;
  will-change: transform, opacity;
}

.cc-clic-target {
  position: relative;
  display: inline-block;
  padding: 0.08em 0.42em 0.12em;
  margin-left: 0.08em;
  font-weight: 800;
  color: #0454b8;
  border-radius: 0.42em;
  will-change: transform;
  transition: color 0.15s ease;
}

.cc-clic-pill {
  position: absolute;
  inset: 0;
  border-radius: inherit;
  background: linear-gradient(135deg, #0454b8 0%, #54a50a 100%);
  box-shadow: 0 6px 20px rgba(4, 84, 184, 0.28);
  opacity: 0;
  z-index: 0;
  pointer-events: none;
}

.cc-clic-label {
  position: relative;
  z-index: 1;
}

.cc-click-ripple {
  position: absolute;
  left: 52%;
  top: 54%;
  width: 68px;
  height: 68px;
  border-radius: 50%;
  border: 3px solid #1880ec;
  box-shadow: 0 0 14px rgba(84, 165, 10, 0.45);
  opacity: 0;
  pointer-events: none;
  z-index: 2;
}

.cc-click-sparks {
  position: absolute;
  left: 52%;
  top: 54%;
  width: 78px;
  height: 78px;
  opacity: 0;
  pointer-events: none;
  z-index: 2;
}

.cc-mouse-cursor {
  position: absolute;
  left: 52%;
  top: 54%;
  margin-left: -4px;
  margin-top: -3px;
  opacity: 0;
  pointer-events: none;
  z-index: 5;
  filter: drop-shadow(0 4px 8px rgba(15, 23, 42, 0.28));
  transform-origin: 4px 3px;
  will-change: transform, opacity;
  border: none;
}
</style>
