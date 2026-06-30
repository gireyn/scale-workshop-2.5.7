# Virtual Tiangans Redesign Plan

## Goal

Restructure the "Virtual Tiangans" page so the **upper half** displays otonal and utonal chord analysis (similar to the Analysis page), and the **lower half** displays the Tiangan keyboard.

## Default Values

| Parameter | Default Value |
|-----------|--------------|
| Trail longevity | 70 |
| Maximum root (otonal) | 28 |
| Maximum root (utonal) | 30 |

## Architecture Overview

```
┌─────────────────────────────────────────────────┐
│  VIRTUAL TIANGANS VIEW (full viewport)           │
│                                                   │
│  ┌──────────────── UPPER HALF ─────────────────┐ │
│  │                                              │ │
│  │  ┌────── Otonal Chord ──────┐ ┌── Controls ─┐│ │
│  │  │   [ChordWheel]           │ │ Trail: [70] ││ │
│  │  │   type="otonal"          │ │ Max Oto:[28]││ │
│  │  │                          │ │ Max Uto:[30]││ │
│  │  └──────────────────────────┘ └─────────────┘│ │
│  │                                              │ │
│  │  ┌────── Utonal Chord ──────┐                │ │
│  │  │   [ChordWheel]           │                │ │
│  │  │   type="utonal"          │                │ │
│  │  └──────────────────────────┘                │ │
│  └──────────────────────────────────────────────┘ │
│                                                   │
│  ┌──────────────── LOWER HALF ─────────────────┐ │
│  │                                              │ │
│  │  甲 乙 丙 丁 戊 己 庚 辛 壬 癸 ...          │ │
│  │  (VirtualTiangans SVG keyboard)              │ │
│  │                                              │ │
│  └──────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

## Files to Modify

### 1. `src/views/VirtualTiangansView.vue` (Major changes)

**Imports to add:**
- `ChordWheel` from `@/components/ChordWheel.vue`
- `computed` and `ref` from `vue`
- `useAudioStore` from `@/stores/audio`

**Reactive state to add:**
```typescript
const audio = useAudioStore()

const trailLongevity = ref(70)
const maxOtonalRoot = ref(28)
const maxUtonalRoot = ref(30)

const fadeAlpha = computed(() => 1 - trailLongevity.value / 100)

const backgroundRBG = computed<[number, number, number]>(() => {
  // Add dependency for reactivity
  state.colorScheme
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
  state.colorScheme
  return getComputedStyle(document.documentElement).getPropertyValue('--color-text').trim()
})
```

**Template restructure:**
```html
<template>
  <main>
    <!-- UPPER HALF: Chord analysis -->
    <div class="analysis-section">
      <h2>Otonal chord</h2>
      <div class="chord-wheels">
        <div class="chord-column">
          <ChordWheel
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
          <ChordWheel
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
      <!-- Controls section -->
      <div class="controls-row">
        <div class="control">
          <label for="trail-longevity">Trail longevity</label>
          <input id="trail-longevity" type="number" class="control" min="0" max="100" v-model="trailLongevity" />
        </div>
        <div class="control">
          <label for="otonal-root">Maximum root (otonal)</label>
          <input id="otonal-root" type="number" class="control" min="1" v-model="maxOtonalRoot" />
        </div>
        <div class="control">
          <label for="utonal-root">Maximum root (utonal)</label>
          <input id="utonal-root" type="number" class="control" min="1" v-model="maxUtonalRoot" />
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
```

**CSS layout:**
```css
main {
  display: flex;
  flex-direction: column;
  height: 100%;
  overflow: hidden;
}

.analysis-section {
  flex: 0 0 auto;  /* sizes to content */
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

.controls-row {
  display: flex;
  flex-direction: row;
  gap: 1rem;
  flex-wrap: wrap;
}

.keyboard-section {
  flex: 1 1 auto;  /* fills remaining space */
  min-height: 200px;
}
```

### 2. `src/components/VirtualTiangans.vue` (Minor adjustment)

The SVG currently uses `width="100%" height="100%"` which should naturally fill the available space. However, we may need to ensure it renders properly when placed inside the lower half. The component already sets `svg { user-select: none; }` which is fine.

No changes expected to the logic, but we should verify the SVG renders correctly in the constrained space.

## Data Flow

```
App.vue
  │
  │  Props: noteOn (callback)
  │
  ▼
VirtualTiangansView.vue
  │
  ├── Uses: audio.virtualSynth (from useAudioStore) ──► for ChordWheel components
  ├── Uses: state (from useStateStore) ──► for baseIndex, heldNotes, colorScheme
  │
  ├── UPPER HALF:
  │   ├── ChordWheel (otonal) ◄── otonalFundamental() from analysis.ts
  │   ├── ChordWheel (utonal) ◄── utonalFundamental() from analysis.ts
  │   └── Controls (trailLongevity, maxOtonalRoot, maxUtonalRoot)
  │
  └── LOWER HALF:
      └── VirtualTiangans ◄── noteOn, baseIndex, heldNotes
          └── SVG keyboard with 41 Tiangan keys
```

## Implementation Steps

1. Modify `src/views/VirtualTiangansView.vue` to:
   - Add imports for `ChordWheel`, `computed`, `ref`, `useAudioStore`
   - Add reactive refs with new default values
   - Add computed properties for `fadeAlpha`, `backgroundRBG`, `strokeStyle`
   - Restructure template to split into analysis-section and keyboard-section
   - Add appropriate scoped CSS

2. Verify `src/components/VirtualTiangans.vue` renders correctly in the lower half
   - The SVG's `width="100%" height="100%"` should adapt to the container

3. No routing changes needed — the router already points to `VirtualTiangansView`
