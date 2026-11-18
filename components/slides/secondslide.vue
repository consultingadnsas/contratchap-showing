<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'

const props = withDefaults(defineProps<{
  /** Titre principal de la slide */
  title?: string
}>(), {
  title: 'Défis juridiques des PME',
})

/**
 * Étapes :
 * 0 -> Carte 1 en grand (50% - Les PME meurent au bout de 3 ans)
 * 1 -> Carte 1 rangée en bas, Carte 2 en grand (Jusqu'à 30 millions - Litiges coûteux)
 * 2 -> Cartes 1 & 2 rangées en bas, Carte 3 en grand (Coût juridique élevé)
 * 3 -> Les 3 cartes sont rangées côte à côte
 */
const step = ref(0)
const isVisible = ref(false)
const root = ref<HTMLElement>()

const pctValue = ref(0)
const millionValue = ref(0)

let raf = 0
let observer: IntersectionObserver | null = null

const clamp = (v: number, a = 0, b = 1) => Math.min(b, Math.max(a, v))
const easeOutCubic = (u: number) => 1 - Math.pow(1 - u, 3)

function animateMetric(targetStep: number) {
  cancelAnimationFrame(raf)
  const duration = 1.05
  let start = 0

  if (targetStep === 0) pctValue.value = 0
  if (targetStep === 1) millionValue.value = 0

  const tick = (now: number) => {
    if (!start) start = now + 120
    const t = Math.max(0, (now - start) / 1000)
    const u = easeOutCubic(clamp(t / duration))

    if (targetStep === 0) {
      pctValue.value = Math.round(50 * u)
    } else if (targetStep === 1) {
      millionValue.value = Math.round(30 * u)
    }

    if (t < duration) {
      raf = requestAnimationFrame(tick)
    } else {
      if (targetStep >= 0) pctValue.value = 50
      if (targetStep >= 1) millionValue.value = 30
    }
  }

  raf = requestAnimationFrame(tick)
}

function goToStep(next: number) {
  step.value = next
  if (next >= 0) pctValue.value = 50
  if (next >= 1) millionValue.value = 30
  if (next === 0 || next === 1) {
    animateMetric(next)
  }
}

function nextStep() {
  if (step.value < 3) {
    goToStep(step.value + 1)
  } else {
    goToStep(0)
  }
}

function onCardClick(index: number, e: MouseEvent) {
  e.stopPropagation()
  if (step.value === index) {
    // Cliquer sur la carte en grand la range et affiche la suivante
    nextStep()
  } else if (index < step.value) {
    // Cliquer sur une carte déjà rangée la ré-affiche en grand
    goToStep(index)
  }
}

function cardState(index: number): 'active' | 'docked' | 'waiting' {
  if (step.value === index) return 'active'
  if (index < step.value) return 'docked'
  return 'waiting'
}

const actionLabel = computed(() => {
  if (step.value === 0) return 'Ranger & afficher le 2e défi'
  if (step.value === 1) return 'Ranger & afficher le 3e défi'
  if (step.value === 2) return 'Ranger la carte (vue finale)'
  return 'Revoir pas à pas'
})

function reset() {
  cancelAnimationFrame(raf)
  isVisible.value = false
  step.value = 0
  pctValue.value = 0
  millionValue.value = 0
}

function startSlide() {
  isVisible.value = true
  goToStep(0)
}

onMounted(() => {
  reset()
  observer = new IntersectionObserver(([entry]) => {
    if (entry.isIntersecting && !isVisible.value) startSlide()
    else if (!entry.isIntersecting) reset()
  }, { threshold: 0.45 })
  if (root.value) observer.observe(root.value)
})

onBeforeUnmount(() => {
  cancelAnimationFrame(raf)
  observer?.disconnect()
})

defineExpose({ nextStep, goToStep, reset })
</script>

<template>
  <div
    ref="root"
    class="cc-second-slide"
    :class="[`step-${step}`, { 'is-visible': isVisible, 'all-docked': step === 3 }]"
  >
    <!-- En-tête -->
    <header class="cc-header">
      <h2 class="cc-title">{{ props.title }}</h2>

      <!-- Indicateur d'étapes + bouton d'action pour ranger / passer au suivant -->
      <div class="cc-controls">
        <div class="cc-dots" role="tablist" aria-label="Étapes des défis">
          <button
            v-for="idx in [0, 1, 2, 3]"
            :key="idx"
            type="button"
            class="cc-dot"
            :class="{ 'is-current': step === idx, 'is-done': idx < step }"
            @click.stop="goToStep(idx)"
          >
            {{ idx < 3 ? idx + 1 : '✓' }}
          </button>
        </div>

        <button type="button" class="cc-action-btn" @click.stop="nextStep">
          <span>{{ actionLabel }}</span>
          <svg viewBox="0 0 20 20" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round">
            <path v-if="step < 3" d="M5 10h10M11 6l4 4-4 4" />
            <path v-else d="M3 10a7 7 0 1 0 2-4.9M3 2v3.5h3.5" />
          </svg>
        </button>
      </div>
    </header>

    <!-- Scène contenant la carte en grand (au centre) et les cartes rangées (en bas) -->
    <div class="cc-stage" @click="nextStep">
      <!-- ───────── CARTE 1 : Mortalité des PME (50%) ───────── -->
      <article
        class="cc-card cc-card--danger"
        :class="`is-${cardState(0)}`"
        style="--slot: 0"
        @click="onCardClick(0, $event)"
      >
        <div class="cc-icon-wrap cc-icon-wrap--danger">
          <span class="cc-icon-pulse cc-icon-pulse--danger" />
          <span class="cc-icon-ring cc-icon-ring--danger" />
          <svg class="cc-svg-icon cc-svg-icon--trend" viewBox="0 0 28 28" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round" stroke-linejoin="round">
            <polyline class="cc-trend-line" points="3 7 11 15 16 10 25 20" />
            <polyline class="cc-trend-head" points="18 20 25 20 25 13" />
          </svg>
        </div>

        <span class="cc-card-badge cc-card-badge--danger">Mortalité des PME</span>
        <div class="cc-big-value cc-big-value--danger">{{ pctValue }}%</div>
        <p class="cc-card-phrase">des PME meurent au bout de 3 ans</p>

        <span v-if="cardState(0) === 'active'" class="cc-click-hint">
          Cliquer pour ranger ↓
        </span>
      </article>

      <!-- ───────── CARTE 2 : Litiges coûteux (Jusqu'à 30 millions) ───────── -->
      <article
        class="cc-card cc-card--blue"
        :class="`is-${cardState(1)}`"
        style="--slot: 1"
        @click="onCardClick(1, $event)"
      >
        <div class="cc-icon-wrap cc-icon-wrap--blue">
          <span class="cc-icon-pulse cc-icon-pulse--blue" />
          <span class="cc-icon-ring cc-icon-ring--blue" />
          <svg class="cc-svg-icon cc-svg-icon--gavel" viewBox="0 0 28 28" fill="none" stroke="currentColor" stroke-width="2.3" stroke-linecap="round" stroke-linejoin="round">
            <g class="cc-gavel-hammer">
              <path d="m16 15-8.5 8.5a2.12 2.12 0 1 1-3-3L13 12" />
              <path d="m18 18 6-6" />
              <path d="m10 10 6-6" />
              <path d="m11 9 8 8" />
              <path d="m23 13-8-8" />
            </g>
            <line class="cc-gavel-base" x1="14" y1="24" x2="25" y2="24" />
          </svg>
        </div>

        <span class="cc-card-badge cc-card-badge--blue">Litiges coûteux</span>
        <div class="cc-big-value cc-big-value--blue">Jusqu'à {{ millionValue }} millions</div>
        <p class="cc-card-phrase">de pertes lors d'un litige juridique</p>

        <span v-if="cardState(1) === 'active'" class="cc-click-hint">
          Cliquer pour ranger ↓
        </span>
      </article>

      <!-- ───────── CARTE 3 : Coûts juridiques élevés ───────── -->
      <article
        class="cc-card cc-card--green"
        :class="`is-${cardState(2)}`"
        style="--slot: 2"
        @click="onCardClick(2, $event)"
      >
        <div class="cc-icon-wrap cc-icon-wrap--green">
          <span class="cc-icon-pulse cc-icon-pulse--green" />
          <span class="cc-icon-ring cc-icon-ring--green" />
          <svg class="cc-svg-icon cc-svg-icon--coins" viewBox="0 0 28 28" fill="none" stroke="currentColor" stroke-width="2.3" stroke-linecap="round" stroke-linejoin="round">
            <g class="cc-coin-front">
              <circle cx="11" cy="11" r="7" />
              <path d="M10 8.5h2v5" />
            </g>
            <g class="cc-coin-back">
              <path d="M20.5 12.5a7 7 0 1 1-8 8" />
              <path d="m19 16 1 1-3 3" />
            </g>
          </svg>
        </div>

        <span class="cc-card-badge cc-card-badge--green">Accès au droit</span>
        <div class="cc-big-value cc-big-value--green">Coût juridique élevé</div>
        <p class="cc-card-phrase">des frais de conseil hors de portée des PME</p>

        <span v-if="cardState(2) === 'active'" class="cc-click-hint">
          Cliquer pour ranger ↓
        </span>
      </article>
    </div>
  </div>
</template>

<style scoped>
.cc-second-slide {
  display: flex;
  flex-direction: column;
  width: 100%;
  min-width: 860px;
  max-width: 980px;
  height: 445px;
  margin: 0 auto;
  padding: 0.35rem 1.5rem;
  user-select: none;
}

/* ───────── En-tête & Contrôles ───────── */
.cc-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 0.65rem;
  opacity: 0;
  transform: translateY(-12px);
  transition: opacity 0.5s ease, transform 0.5s ease;
}

.is-visible .cc-header {
  opacity: 1;
  transform: translateY(0);
}

.cc-title {
  margin: 0;
  font-size: clamp(1.75rem, 3vw, 2.3rem);
  font-weight: 900;
  letter-spacing: -0.025em;
  background: linear-gradient(135deg, #0454b8 0%, #1880ec 55%, #54a50a 100%);
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
}

.cc-controls {
  display: flex;
  align-items: center;
  gap: 0.65rem;
}

.cc-dots {
  display: flex;
  align-items: center;
  gap: 0.35rem;
}

.cc-dot {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  border: 1.5px solid #cbd5e1;
  background: #ffffff;
  color: #64748b;
  font-size: 0.76rem;
  font-weight: 800;
  cursor: pointer;
  transition: all 0.25s ease;
}

.cc-dot.is-current {
  border-color: #0454b8;
  background: #0454b8;
  color: #ffffff;
  transform: scale(1.1);
  box-shadow: 0 4px 12px rgba(4, 84, 184, 0.28);
}

.cc-dot.is-done {
  border-color: #54a50a;
  background: rgba(84, 165, 10, 0.12);
  color: #3f7d07;
}

.cc-action-btn {
  display: inline-flex;
  align-items: center;
  gap: 0.45rem;
  padding: 0.45rem 0.95rem;
  border-radius: 999px;
  border: none;
  background: linear-gradient(135deg, #0454b8 0%, #1880ec 100%);
  color: #ffffff;
  font-size: 0.8rem;
  font-weight: 700;
  cursor: pointer;
  box-shadow: 0 6px 18px rgba(4, 84, 184, 0.25);
  transition: transform 0.2s ease, box-shadow 0.2s ease;
}

.cc-action-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 10px 22px rgba(4, 84, 184, 0.34);
}

.cc-action-btn:active {
  transform: translateY(0) scale(0.97);
}

/* ───────── Scène principale ───────── */
.cc-stage {
  position: relative;
  flex: 1;
  width: 100%;
  cursor: pointer;
}

/* ───────── Carte (états : waiting -> active (en grand) -> docked (rangée en bas)) ───────── */
.cc-card {
  position: absolute;
  left: 50%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  text-align: center;
  background: #ffffff;
  border-radius: 1.35rem;
  border: 2px solid #e2e8f0;
  transition:
    top 0.65s cubic-bezier(0.22, 1, 0.36, 1),
    left 0.65s cubic-bezier(0.22, 1, 0.36, 1),
    width 0.65s cubic-bezier(0.22, 1, 0.36, 1),
    height 0.65s cubic-bezier(0.22, 1, 0.36, 1),
    padding 0.65s cubic-bezier(0.22, 1, 0.36, 1),
    transform 0.65s cubic-bezier(0.22, 1, 0.36, 1),
    opacity 0.45s ease,
    box-shadow 0.45s ease,
    border-color 0.45s ease;
}

/* État 1 : En attente (pas encore affichée) */
.cc-card.is-waiting {
  top: 20px;
  width: 560px;
  height: 250px;
  padding: 1.8rem 2rem;
  opacity: 0;
  pointer-events: none;
  transform: translate3d(-50%, 35px, 0) scale(0.86);
  z-index: 1;
}

/* État 2 : Affichée EN GRAND au centre */
.cc-card.is-active {
  top: 6px;
  left: 50%;
  width: 580px;
  height: 252px;
  padding: 1.6rem 2.2rem;
  opacity: 1;
  transform: translate3d(-50%, 0, 0) scale(1);
  z-index: 10;
}

.cc-card--danger.is-active {
  border-color: rgba(225, 29, 72, 0.38);
  box-shadow: 0 22px 48px -12px rgba(225, 29, 72, 0.22);
}

.cc-card--blue.is-active {
  border-color: rgba(4, 84, 184, 0.38);
  box-shadow: 0 22px 48px -12px rgba(4, 84, 184, 0.22);
}

.cc-card--green.is-active {
  border-color: rgba(84, 165, 10, 0.42);
  box-shadow: 0 22px 48px -12px rgba(84, 165, 10, 0.24);
}

/* État 3 : RANGÉE EN BAS pendant qu'une autre carte est en grand */
.cc-card.is-docked {
  top: 274px;
  left: calc(50% + (var(--slot) - 1) * 305px);
  width: 285px;
  height: 112px;
  padding: 0.65rem 0.9rem;
  opacity: 0.95;
  transform: translate3d(-50%, 0, 0) scale(1);
  box-shadow: 0 8px 20px -6px rgba(15, 23, 42, 0.1);
  z-index: 5;
}

.cc-card.is-docked:hover {
  opacity: 1;
  transform: translate3d(-50%, -4px, 0) scale(1.02);
}

/* État 4 : Quand les 3 cartes sont toutes rangées (vue d'ensemble finale) */
.all-docked .cc-card.is-docked {
  top: 52px;
  left: calc(50% + (var(--slot) - 1) * 308px);
  width: 290px;
  height: 270px;
  padding: 1.5rem 1.25rem;
  opacity: 1;
  box-shadow: 0 16px 36px -10px rgba(15, 23, 42, 0.12);
}

/* ───────── Icônes & Effets animés ───────── */
.cc-icon-wrap {
  position: relative;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 68px;
  height: 68px;
  border-radius: 1.1rem;
  margin-bottom: 0.65rem;
  transition: all 0.5s ease;
}

.cc-card.is-docked:not(.all-docked *) .cc-icon-wrap {
  width: 34px;
  height: 34px;
  border-radius: 0.6rem;
  margin-bottom: 0.2rem;
}

.all-docked .cc-card.is-docked .cc-icon-wrap {
  width: 58px;
  height: 58px;
  margin-bottom: 0.75rem;
}

.cc-icon-wrap--danger {
  color: #e11d48;
  background: rgba(225, 29, 72, 0.1);
}

.cc-icon-wrap--blue {
  color: #0454b8;
  background: rgba(4, 84, 184, 0.1);
}

.cc-icon-wrap--green {
  color: #54a50a;
  background: rgba(84, 165, 10, 0.12);
}

.cc-svg-icon {
  width: 36px;
  height: 36px;
  z-index: 2;
  transition: width 0.5s ease, height 0.5s ease;
}

.cc-card.is-docked .cc-svg-icon {
  width: 20px;
  height: 20px;
}

.all-docked .cc-card.is-docked .cc-svg-icon {
  width: 30px;
  height: 30px;
}

/* Halo pulsant et onde autour de l'icône */
.cc-icon-pulse,
.cc-icon-ring {
  position: absolute;
  inset: 0;
  border-radius: inherit;
  pointer-events: none;
}

.cc-icon-pulse--danger {
  background: rgba(225, 29, 72, 0.22);
  animation: cc-pulse-glow 2s ease-in-out infinite;
}

.cc-icon-ring--danger {
  border: 2px solid rgba(225, 29, 72, 0.45);
  animation: cc-ring-expand 2s cubic-bezier(0.16, 1, 0.3, 1) infinite;
}

.cc-icon-pulse--blue {
  background: rgba(4, 84, 184, 0.22);
  animation: cc-pulse-glow 2s ease-in-out infinite;
}

.cc-icon-ring--blue {
  border: 2px solid rgba(4, 84, 184, 0.45);
  animation: cc-ring-expand 2s cubic-bezier(0.16, 1, 0.3, 1) infinite;
}

.cc-icon-pulse--green {
  background: rgba(84, 165, 10, 0.22);
  animation: cc-pulse-glow 2s ease-in-out infinite;
}

.cc-icon-ring--green {
  border: 2px solid rgba(84, 165, 10, 0.45);
  animation: cc-ring-expand 2s cubic-bezier(0.16, 1, 0.3, 1) infinite;
}

/* Animation spécifique Icône 1 : Courbe de chute */
.cc-svg-icon--trend {
  animation: cc-trend-float 1.8s ease-in-out infinite;
}

.cc-trend-line {
  stroke-dasharray: 40;
  stroke-dashoffset: 0;
  animation: cc-draw-line 1.8s ease-in-out infinite;
}

/* Animation spécifique Icône 2 : Marteau de justice qui frappe */
.cc-gavel-hammer {
  transform-origin: 10px 20px;
  animation: cc-gavel-strike 1.4s cubic-bezier(0.36, 0, 0.66, 1) infinite;
}

/* Animation spécifique Icône 3 : Pièces qui lévitent et tournent légèrement */
.cc-svg-icon--coins {
  animation: cc-coins-bounce 1.8s ease-in-out infinite;
}

@keyframes cc-pulse-glow {
  0%, 100% { transform: scale(1); opacity: 0.35; }
  50% { transform: scale(1.18); opacity: 0.7; }
}

@keyframes cc-ring-expand {
  0% { transform: scale(0.95); opacity: 0.85; }
  100% { transform: scale(1.55); opacity: 0; }
}

@keyframes cc-trend-float {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(3px) translateX(1px); }
}

@keyframes cc-draw-line {
  0% { stroke-dashoffset: 40; }
  45%, 100% { stroke-dashoffset: 0; }
}

@keyframes cc-gavel-strike {
  0%, 100% { transform: rotate(0deg); }
  35% { transform: rotate(-22deg); }
  55% { transform: rotate(10deg); }
  70% { transform: rotate(0deg); }
}

@keyframes cc-coins-bounce {
  0%, 100% { transform: translateY(0) rotate(0deg); }
  50% { transform: translateY(-4px) rotate(6deg); }
}

/* ───────── Typographie de la carte ───────── */
.cc-card-badge {
  display: inline-block;
  padding: 0.18rem 0.7rem;
  margin-bottom: 0.3rem;
  border-radius: 999px;
  font-size: 0.75rem;
  font-weight: 800;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  transition: all 0.45s ease;
}

.cc-card.is-docked .cc-card-badge {
  display: none;
}

.all-docked .cc-card.is-docked .cc-card-badge {
  display: inline-block;
  margin-bottom: 0.4rem;
}

.cc-card-badge--danger {
  color: #be123c;
  background: rgba(225, 29, 72, 0.1);
}

.cc-card-badge--blue {
  color: #0454b8;
  background: rgba(4, 84, 184, 0.1);
}

.cc-card-badge--green {
  color: #3f7d07;
  background: rgba(84, 165, 10, 0.13);
}

.cc-big-value {
  font-size: 3.1rem;
  font-weight: 900;
  line-height: 1.05;
  letter-spacing: -0.035em;
  margin-bottom: 0.3rem;
  transition: font-size 0.5s cubic-bezier(0.22, 1, 0.36, 1);
}

.cc-big-value--danger {
  color: #e11d48;
}

.cc-big-value--blue {
  color: #0454b8;
  font-size: 2.55rem;
}

.cc-big-value--green {
  color: #54a50a;
  font-size: 2.35rem;
}

/* Taille réduite quand la carte est rangée en bas */
.cc-card.is-docked .cc-big-value {
  font-size: 1.25rem;
  margin-bottom: 0.1rem;
}

.all-docked .cc-card.is-docked .cc-big-value--danger {
  font-size: 2.35rem;
  margin-bottom: 0.35rem;
}

.all-docked .cc-card.is-docked .cc-big-value--blue,
.all-docked .cc-card.is-docked .cc-big-value--green {
  font-size: 1.65rem;
  margin-bottom: 0.35rem;
}

.cc-card-phrase {
  margin: 0;
  font-size: 1.18rem;
  font-weight: 600;
  color: #334155;
  line-height: 1.3;
  transition: font-size 0.5s ease;
}

.cc-card.is-docked .cc-card-phrase {
  font-size: 0.76rem;
  line-height: 1.2;
}

.all-docked .cc-card.is-docked .cc-card-phrase {
  font-size: 0.95rem;
  line-height: 1.35;
}

.cc-click-hint {
  margin-top: 0.65rem;
  font-size: 0.74rem;
  font-weight: 700;
  color: #94a3b8;
  letter-spacing: 0.03em;
}
</style>
