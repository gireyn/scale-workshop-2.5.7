<script setup lang="ts">
import { computed, ref } from 'vue'
import VirtualTiangans from '@/components/VirtualTiangans.vue'
import ChordWheel from '@/components/ChordWheel.vue'
import TianganLattice from '@/components/TianganLattice.vue'
import { useAudioStore } from '@/stores/audio'
import { useStateStore } from '@/stores/state'

defineProps<{
  noteOn: NoteOnCallback
}>()

const audio = useAudioStore()
const state = useStateStore()

type NoteOff = () => void
type NoteOnCallback = (index: number) => NoteOff

const trailLongevity = ref(70)
const maxOtonalRoot = ref(28)
const maxUtonalRoot = ref(30)

const fadeAlpha = computed(() => 1 - trailLongevity.value / 100)

const backgroundRBG = computed<[number, number, number]>(() => {
  // Add dependency.
  state.colorScheme
  // Fetch from document.
  const css = getComputedStyle(document.documentElement)
    .getPropertyValue('--color-background')
    .trim()
    .toLowerCase()
  if (css === '#fff') {
    return [255, 255, 255]
  } else if (css === '#000') {
    return [0, 0, 0]
  } else {
    throw new Error('General color parsing not implemented')
  }
})

const strokeStyle = computed(() => {
  // Add dependency.
  state.colorScheme
  // Fetch from document.
  return getComputedStyle(document.documentElement).getPropertyValue('--color-text').trim()
})
</script>

<template>
  <main>
    <!-- UPPER 2/3: Three-column analysis -->
    <div class="analysis-section">
      <div class="three-columns">
        <!-- Column 1: Lattice (larger flex share) -->
        <div class="col col-lattice">
          <h2>Lattice</h2>
          <div class="lattice-wrapper">
            <TianganLattice :heldNotes="state.heldNotes" :baseIndex="state.baseIndex" />
          </div>
        </div>

        <!-- Column 2: Otonal chord + controls (narrower) -->
        <div class="col col-chord">
          <h2>Otonal chord</h2>
          <ChordWheel
            class="chord-wheel"
            type="otonal"
            :maxChordRoot="maxOtonalRoot"
            :virtualSynth="audio.virtualSynth"
            :width="514"
            :height="400"
            :lineWidth="2"
            :backgroundRBG="backgroundRBG"
            :fadeAlpha="fadeAlpha"
            :shadowBlur="2"
            :strokeStyle="strokeStyle"
            textColor="red"
          />
          <!-- Controls placed under otonal chord -->
          <div class="controls-group">
            <div class="control-item">
              <label for="trail-longevity">Trail longevity</label>
              <input
                id="trail-longevity"
                type="number"
                class="control-input"
                min="0"
                max="100"
                v-model="trailLongevity"
              />
            </div>
            <div class="control-item">
              <label for="otonal-root">Maximum root (otonal)</label>
              <input
                id="otonal-root"
                type="number"
                class="control-input"
                min="1"
                v-model="maxOtonalRoot"
              />
            </div>
            <div class="control-item">
              <label for="utonal-root">Maximum root (utonal)</label>
              <input
                id="utonal-root"
                type="number"
                class="control-input"
                min="1"
                v-model="maxUtonalRoot"
              />
            </div>
          </div>
        </div>

        <!-- Column 3: Utonal chord (narrower) -->
        <div class="col col-chord">
          <h2>Utonal chord</h2>
          <ChordWheel
            class="chord-wheel"
            type="utonal"
            :maxChordRoot="maxUtonalRoot"
            :virtualSynth="audio.virtualSynth"
            :width="514"
            :height="400"
            :lineWidth="2"
            :backgroundRBG="backgroundRBG"
            :fadeAlpha="fadeAlpha"
            :shadowBlur="2"
            :strokeStyle="strokeStyle"
            textColor="blue"
          />
        </div>
      </div>
    </div>

    <!-- LOWER 1/3: Tiangan keyboard -->
    <div class="keyboard-section">
      <VirtualTiangans
        :baseIndex="state.baseIndex"
        :noteOn="noteOn"
        :heldNotes="state.heldNotes"
      />
    </div>
  </main>
</template>

<style scoped>
/* View layout */
main {
  display: flex;
  flex-direction: column;
  height: 100%;
  overflow: hidden;
}

/* Upper 2/3: analysis */
.analysis-section {
  flex: 2 1 0;
  padding: 0.5rem 0.75rem;
  overflow-y: auto;
  border-bottom: 2px solid var(--color-border);
  min-height: 0;
}

.three-columns {
  display: flex;
  flex-direction: row;
  gap: 0.75rem;
  height: 100%;
}

.col {
  min-width: 0;
  display: flex;
  flex-direction: column;
}
.col-lattice {
  flex: 3;
}
.col-chord {
  flex: 2;
}

.col h2 {
  border-bottom: 1px solid var(--color-border);
  font-size: 1.1em;
  font-weight: bold;
  width: 100%;
  margin-bottom: 0.4rem;
  flex-shrink: 0;
}

/* Lattice column */
.lattice-wrapper {
  flex: 1;
  min-height: 100px;
  border: 1px solid var(--color-border);
  overflow: hidden;
}

/* Chord wheel column */
.chord-wheel {
  border: 1px solid var(--color-border);
  max-width: 100%;
  height: auto;
  flex-shrink: 0;
}

/* Controls under otonal chord */
.controls-group {
  display: flex;
  flex-direction: column;
  gap: 0.3rem;
  margin-top: 2rem;
}

.control-item {
  display: flex;
  flex-direction: row;
  align-items: center;
  gap: 0.4rem;
}

.control-item label {
  font-weight: bold;
  font-size: 0.85em;
  white-space: nowrap;
  min-width: 8rem;
}

.control-input {
  padding: 0.3rem;
  background-color: var(--color-background-soft);
  color: var(--color-text);
  border: 1px solid var(--color-border);
  border-radius: 3px;
  width: 4rem;
}

/* Lower 1/3: keyboard */
.keyboard-section {
  flex: 1 1 0;
  min-height: 80px;
}
</style>
