<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'

const props = withDefaults(defineProps<{
  /** Texte animé en vague */
  text?: string
  /** Hauteur de la scène (px) */
  height?: number
  /** Amplitude verticale de la vague (px) */
  amplitude?: number
  /** Vitesse de propagation de la vague (rad/s) */
  speed?: number
  /** Déphasage entre deux lettres consécutives (rad) */
  step?: number
  /** Durée de l'entrée en vague (s) */
  enterTime?: number
  /** Taille de police (valeur CSS) */
  fontSize?: string
}>(), {
  text: 'Sécurisez juridiquement vos business',
  height: 380,
  amplitude: 16,
  speed: 3.4,
  step: 0.42,
  enterTime: 1.15,
  fontSize: 'clamp(1.9rem, 4.2vw, 3.35rem)',
})

const emit = defineEmits<{ (e: 'done'): void }>()

const letters = computed(() =>
  Array.from(props.text).map((ch, index) => ({
    char: ch === ' ' ? '\u00A0' : ch,
    isSpace: ch === ' ',
    index,
  })),
)

const clamp = (v: number, a = 0, b = 1) => Math.min(b, Math.max(a, v))
const easeOutCubic = (u: number) => 1 - Math.pow(1 - u, 3)

const root = ref<HTMLElement>()
const charEls: (HTMLElement | null)[] = []

let raf = 0
let observer: IntersectionObserver | null = null
let isPlaying = false

const reduceMotion = () =>
  typeof window !== 'undefined' && window.matchMedia?.('(prefers-reduced-motion: reduce)').matches

function colorFor(phase: number) {
  // Dégradé fluide entre le bleu (#0454b8) et le vert (#54a50a) Contratchap
  const mix = 0.5 + 0.5 * Math.sin(phase)
  const r = Math.round(4 + (84 - 4) * mix)
  const g = Math.round(84 + (165 - 84) * mix)
  const b = Math.round(184 + (10 - 184) * mix)
  return `rgb(${r}, ${g}, ${b})`
}

function reset() {
  cancelAnimationFrame(raf)
  isPlaying = false
  for (const el of charEls) {
    if (!el) continue
    el.style.opacity = '0'
    el.style.transform = 'translate3d(0, 32px, 0)'
  }
}

function showFinal() {
  cancelAnimationFrame(raf)
  isPlaying = false
  const total = Math.max(1, letters.value.length - 1)
  letters.value.forEach((item, i) => {
    const el = charEls[i]
    if (!el) return
    el.style.opacity = '1'
    el.style.transform = 'translate3d(0, 0, 0)'
    if (!item.isSpace) {
      el.style.color = colorFor((i / total) * Math.PI * 1.5)
    }
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

  let start = 0
  let doneEmitted = false
  const total = Math.max(1, letters.value.length)

  const tick = (now: number) => {
    if (!start) start = now
    const t = (now - start) / 1000

    if (!doneEmitted && t >= props.enterTime) {
      doneEmitted = true
      emit('done')
    }

    for (let i = 0; i < total; i++) {
      const el = charEls[i]
      if (!el) continue

      // Apparition progressive lettre par lettre
      const delay = (i / total) * (props.enterTime * 0.65)
      const enterU = easeOutCubic(clamp((t - delay) / 0.4))

      const phase = t * props.speed - i * props.step
      const waveY = -Math.sin(phase) * props.amplitude * enterU
      const waveRot = -Math.cos(phase) * 5.5 * enterU
      const waveScale = 1 + 0.05 * Math.sin(phase) * enterU
      const dropY = 34 * (1 - enterU)

      el.style.opacity = enterU.toFixed(3)
      el.style.transform =
        `translate3d(0, ${(dropY + waveY).toFixed(2)}px, 0) ` +
        `rotate(${waveRot.toFixed(2)}deg) scale(${waveScale.toFixed(3)})`
      el.style.color = colorFor(phase * 0.65)
    }

    raf = requestAnimationFrame(tick)
  }

  raf = requestAnimationFrame(tick)
}

onMounted(() => {
  reset()
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
    class="cc-wave-stage"
    :style="{ height: `${height}px` }"
    @click="play"
  >
    <p class="cc-wave-text" :style="{ fontSize }" :aria-label="text">
      <span
        v-for="item in letters"
        :key="item.index"
        :ref="(el) => { charEls[item.index] = el as HTMLElement }"
        class="cc-wave-char"
        aria-hidden="true"
      >{{ item.char }}</span>
    </p>
  </div>
</template>

<style scoped>
.cc-wave-stage {
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

.cc-wave-text {
  margin: 0;
  font-weight: 800;
  letter-spacing: -0.02em;
  line-height: 1.3;
  white-space: nowrap;
}

.cc-wave-char {
  display: inline-block;
  will-change: transform, opacity, color;
  transform-origin: center 70%;
}
</style>

