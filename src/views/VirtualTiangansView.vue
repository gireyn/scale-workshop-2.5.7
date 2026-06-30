<script setup lang="ts">
import { computed, ref } from 'vue'
import VirtualTiangans from '@/components/VirtualTiangans.vue'
import ChordWheel from '@/components/ChordWheel.vue'
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
    <!-- UPPER HALF: Chord analysis -->
    <div class="analysis-section">
      <div class="chord-wheels">
        <div class="chord-column">
          <h2>Otonal chord</h2>
          <ChordWheel
            class="chord-wheel"
            type="otonal"
            :maxChordRoot="maxOtonalRoot"
            :virtualSynth="audio.virtualSynth"
            :width="500"
            :height="300"
            :lineWidth="2"
            :backgroundRBG="backgroundRBG"
            :fadeAlpha="fadeAlpha"
            :shadowBlur="2"
            :strokeStyle="strokeStyle"
            textColor="red"
          />
        </div>
        <div class="chord-column">
          <h2>Utonal chord</h2>
          <ChordWheel
            class="chord-wheel"
            type="utonal"
            :maxChordRoot="maxUtonalRoot"
            :virtualSynth="audio.virtualSynth"
            :width="500"
            :height="300"
            :lineWidth="2"
            :backgroundRBG="backgroundRBG"
            :fadeAlpha="fadeAlpha"
            :shadowBlur="2"
            :strokeStyle="strokeStyle"
            textColor="blue"
          />
        </div>
      </div>
      <div class="controls-row">
        <div class="control">
          <label for="trail-longevity">Trail longevity</label>
          <input
            id="trail-longevity"
            type="number"
            class="control"
            min="0"
            max="100"
            v-model="trailLongevity"
          />
        </div>
        <div class="control">
          <label for="otonal-root">Maximum root (otonal)</label>
          <input
            id="otonal-root"
            type="number"
            class="control"
            min="1"
            v-model="maxOtonalRoot"
          />
        </div>
        <div class="control">
          <label for="utonal-root">Maximum root (utonal)</label>
          <input
            id="utonal-root"
            type="number"
            class="control"
            min="1"
            v-model="maxUtonalRoot"
          />
        </div>
      </div>
    </div>

    <!-- LOWER HALF: Tiangan keyboard -->
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

/* Upper half: analysis */
.analysis-section {
  flex: 0 0 auto;
  padding: 0.5rem 1rem;
  overflow-y: auto;
  border-bottom: 2px solid var(--color-border);
}

.chord-wheels {
  display: flex;
  flex-direction: row;
  gap: 1rem;
}

.chord-column {
  flex: 1;
  min-width: 0;
}

.chord-column h2 {
  border-bottom: 1px solid var(--color-border);
  font-size: 1.1em;
  font-weight: bold;
  width: 100%;
  margin-bottom: 0.5rem;
}

.chord-wheel {
  border: 1px solid var(--color-border);
  max-width: 100%;
  height: auto;
}

.controls-row {
  display: flex;
  flex-direction: row;
  gap: 1rem;
  flex-wrap: wrap;
  margin-top: 0.5rem;
}

.controls-row .control {
  display: flex;
  flex-direction: column;
  gap: 0.15rem;
}

.controls-row .control label {
  font-weight: bold;
}

/* Lower half: keyboard */
.keyboard-section {
  flex: 1 1 auto;
  min-height: 120px;
}
</style>
