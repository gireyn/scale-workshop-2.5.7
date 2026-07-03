<script setup lang="ts">
import { LEFT_MOUSE_BTN } from '@/constants'
import { computed, onMounted, onUnmounted, ref } from 'vue'

type NoteOff = () => void
type NoteOnCallback = (index: number) => NoteOff

const props = defineProps<{
  baseIndex: number
  noteOn: NoteOnCallback
  heldNotes: Map<number, number>
}>()

const NUM_KEYS = 41
const KEY_SPACING = 100 // SVG viewBox units per key
const KEY_WIDTH = 85    // compressed key width (15 units gap)
const KEY_TOP = 700     // key top y in viewBox
const KEY_HEIGHT = 250  // key height
const KEY_BOTTOM = KEY_TOP + KEY_HEIGHT
const GRID_AREA_TOP = 180  // top of grid visualization area

const TIANGAN = [
  { char: '甲', color: 'rgb(89, 195, 181)' },
  { char: '乙', color: 'rgb(178, 225, 217)' },
  { char: '丙', color: 'rgb(229, 152, 125)' },
  { char: '丁', color: 'rgb(244, 204, 190)' },
  { char: '戊', color: 'rgb(175, 181, 104)' },
  { char: '己', color: 'rgb(214, 218, 180)' },
  { char: '庚', color: 'rgb(209, 151, 206)' },
  { char: '辛', color: 'rgb(233, 203, 231)' },
  { char: '壬', color: 'rgb(132, 177, 237)' },
  { char: '癸', color: 'rgb(193, 216, 247)' },
]

function parseRgb(color: string) {
  return color.match(/\d+/g)!.map(Number)
}

const keys = computed(() => {
  const result = []
  for (let i = 0; i < NUM_KEYS; i++) {
    const index = props.baseIndex + i - 10
    const tg = TIANGAN[i % TIANGAN.length]
    const active = (props.heldNotes.get(index) || 0) > 0
    const rgb = parseRgb(tg.color)
    const fill = active
      ? `rgb(${Math.round(rgb[0] / 2)}, ${Math.round(rgb[1] / 2)}, ${Math.round(rgb[2] / 2)})`
      : tg.color
    const textColor = active ? 'white' : 'black'
    result.push({ index, char: tg.char, fill, textColor, x: i })
  }
  return result
})

// --- Gap relationship data ---
// Sequence of gap types between consecutive Tiangan characters
const GAP_SEQUENCE = [7, 9, 7, 7, 5, 7, 7, 9, 7, 7]

interface GapRenderInfo {
  gapType: number
  centerX: number
  isLast: boolean
  // Grid dots: [cx, cy][] relative to grid origin (top-left of grid)
  gridDots: [number, number][]
  // Grid lines forming the grid cells
  gridLines: { x1: number; y1: number; x2: number; y2: number }[]
  // Vector line: start and end points
  vecX1: number
  vecY1: number
  vecX2: number
  vecY2: number
  // Colors
  color: string
  darkColor: string
  // Grid dimensions for centering
  gridPixelW: number
  gridPixelH: number
}

// Unit edge sizes
const UNIT_7_9 = 28  // for type 7 and 9
const UNIT_5 = 14     // half for type 5

// Colors per type
const COLORS: Record<number, { color: string; dark: string }> = {
  7: { color: '#4CAF50', dark: '#1B5E20' },
  9: { color: '#2196F3', dark: '#0D47A1' },
  5: { color: '#FF9800', dark: '#BF360C' },
}

function buildGapInfo(gapType: number, isLast: boolean): Omit<GapRenderInfo, 'centerX'> {
  const unit = gapType === 5 ? UNIT_5 : UNIT_7_9
  const { color, dark } = COLORS[gapType]
  let gridW: number, gridH: number
  let vecStartUser: [number, number], vecEndUser: [number, number]

  if (gapType === 7) {
    // 1×1 square grid, vector (-1,-1) from upper-right to lower-left
    gridW = 1
    gridH = 1
    // User coords: upper-right=(1,0), lower-left=(0,-1)
    vecStartUser = [1, 0]
    vecEndUser = [0, -1]
  } else if (gapType === 9) {
    // 2×3 rectangle grid, vector (2,3)
    gridW = 2
    gridH = 3
    vecStartUser = [0, 0]
    vecEndUser = [2, 3]
  } else {
    // 3×1 rectangle grid, vector (3,1)
    gridW = 3
    gridH = 1
    vecStartUser = [0, 0]
    vecEndUser = [3, 1]
  }

  // Generate grid dots and lines
  const gridDots: [number, number][] = []
  const gridLines: { x1: number; y1: number; x2: number; y2: number }[] = []
  for (let gx = 0; gx <= gridW; gx++) {
    for (let gy = 0; gy <= gridH; gy++) {
      gridDots.push([gx * unit, gy * unit])
    }
  }
  // Horizontal grid lines
  for (let gy = 0; gy <= gridH; gy++) {
    gridLines.push({ x1: 0, y1: gy * unit, x2: gridW * unit, y2: gy * unit })
  }
  // Vertical grid lines
  for (let gx = 0; gx <= gridW; gx++) {
    gridLines.push({ x1: gx * unit, y1: 0, x2: gx * unit, y2: gridH * unit })
  }

  // Convert user coords to SVG coords for the vector
  // User: +x=right, +y=up → SVG: +x=right, +y=down (flip y)
  const vecX1 = vecStartUser[0] * unit
  const vecY1 = -vecStartUser[1] * unit
  const vecX2 = vecEndUser[0] * unit
  const vecY2 = -vecEndUser[1] * unit

  return {
    gapType,
    isLast,
    gridDots,
    gridLines,
    vecX1, vecY1, vecX2, vecY2,
    color,
    darkColor: dark,
    gridPixelW: gridW * unit,
    gridPixelH: gridH * unit,
  }
}

const baseGapInfo: Record<number, Omit<GapRenderInfo, 'centerX'>> = {
  7: buildGapInfo(7, false),
  9: buildGapInfo(9, false),
  5: buildGapInfo(5, false),
}
// Special last gap info for 癸→甲 (type 7)
const lastGapInfo: Omit<GapRenderInfo, 'centerX'> = {
  ...buildGapInfo(7, false),
  isLast: true,
}

const gaps = computed<GapRenderInfo[]>(() => {
  const result: GapRenderInfo[] = []
  for (let i = 0; i < NUM_KEYS - 1; i++) {
    const seqIdx = i % 10
    const gapType = GAP_SEQUENCE[seqIdx]
    const isLast = seqIdx === 9 // 癸→甲 transition
    const base = isLast ? lastGapInfo : baseGapInfo[gapType]
    const centerX = (i + 1) * KEY_SPACING
    result.push({ ...base, centerX, gapType, isLast })
  }
  return result
})

const noteOffs: Map<number, NoteOff> = new Map()
const isMousePressed = ref(false)

function start(index: number) {
  noteOffs.set(index, props.noteOn(index))
}

function end(index: number) {
  if (noteOffs.has(index)) {
    noteOffs.get(index)!()
    noteOffs.delete(index)
  }
}

function onTouchStart(event: TouchEvent, index: number) {
  event.preventDefault()
  end(index)
  start(index)
}

function onTouchEnd(event: TouchEvent, index: number) {
  event.preventDefault()
  end(index)
}

function onMouseDown(event: MouseEvent, index: number) {
  if (event.button !== LEFT_MOUSE_BTN) return
  event.preventDefault()
  isMousePressed.value = true
  start(index)
}

function onMouseUp(event: MouseEvent, index: number) {
  if (event.button !== LEFT_MOUSE_BTN) return
  event.preventDefault()
  isMousePressed.value = false
  end(index)
}

function onMouseEnter(event: MouseEvent, index: number) {
  if (!isMousePressed.value) return
  event.preventDefault()
  start(index)
}

function onMouseLeave(event: MouseEvent, index: number) {
  if (!isMousePressed.value) return
  event.preventDefault()
  end(index)
}

function windowMouseUp(event: MouseEvent) {
  if (event.button === LEFT_MOUSE_BTN) {
    isMousePressed.value = false
  }
}

onMounted(() => {
  window.addEventListener('mouseup', windowMouseUp)
})

onUnmounted(() => {
  noteOffs.forEach((off) => {
    if (off !== null) off()
  })
  window.removeEventListener('mouseup', windowMouseUp)
})
</script>

<template>
  <svg viewBox="0 0 4100 1000" width="100%" height="100%">
    <defs>
      <!-- Arrow marker for gap vectors -->
      <marker
        id="gap-arrow"
        markerWidth="6"
        markerHeight="6"
        refX="5"
        refY="3"
        orient="auto"
        markerUnits="userSpaceOnUse"
      >
        <polygon points="0 0, 6 3, 0 6" fill="currentColor" />
      </marker>
    </defs>

    <!-- Gap visualizations (grids + vectors + text) -->
    <g v-for="(gap, gi) of gaps" :key="'gap-' + gi">
      <!-- Grid area centered horizontally on the gap -->
      <g
        :transform="`translate(${gap.centerX - gap.gridPixelW / 2}, ${GRID_AREA_TOP})`"
      >
        <!-- Grid lines -->
        <line
          v-for="(line, li) of gap.gridLines"
          :key="'gl-' + li"
          :x1="line.x1"
          :y1="line.y1"
          :x2="line.x2"
          :y2="line.y2"
          :stroke="gap.color"
          stroke-width="1"
          opacity="0.6"
        />
        <!-- Grid dots -->
        <circle
          v-for="(dot, di) of gap.gridDots"
          :key="'gd-' + di"
          :cx="dot[0]"
          :cy="dot[1]"
          r="2.5"
          :fill="gap.color"
        />
        <!-- Vector line with arrow -->
        <line
          :x1="gap.vecX1"
          :y1="gap.vecY1"
          :x2="gap.vecX2"
          :y2="gap.vecY2"
          :stroke="gap.darkColor"
          stroke-width="1.5"
          marker-end="url(#gap-arrow)"
          :style="{ color: gap.darkColor }"
        />
      </g>

      <!-- Text centered above the grid -->
      <text
        :x="gap.centerX"
        :y="GRID_AREA_TOP - 10"
        text-anchor="middle"
        class="gap-text"
        :fill="gap.darkColor"
      >
        <template v-if="gap.isLast">
          <tspan>{{ gap.gapType + '\\' }}</tspan><tspan class="gap-text-extra">186ed6</tspan>
        </template>
        <template v-else>
          {{ gap.gapType + '\\' }}
        </template>
      </text>
    </g>

    <!-- Keyboard keys -->
    <g
      v-for="(key, i) of keys"
      :key="i"
      @touchstart="onTouchStart($event, key.index)"
      @touchend="onTouchEnd($event, key.index)"
      @touchcancel="onTouchEnd($event, key.index)"
      @mousedown="onMouseDown($event, key.index)"
      @mouseup="onMouseUp($event, key.index)"
      @mouseenter="onMouseEnter($event, key.index)"
      @mouseleave="onMouseLeave($event, key.index)"
      class="key-group"
    >
      <rect
        :x="key.x * KEY_SPACING + (KEY_SPACING - KEY_WIDTH) / 2"
        :y="KEY_TOP"
        :width="KEY_WIDTH"
        :height="KEY_HEIGHT"
        :fill="key.fill"
        stroke="gray"
        stroke-width="1.5"
        rx="3"
      />
      <text
        :x="key.x * KEY_SPACING + KEY_SPACING / 2"
        :y="KEY_TOP + KEY_HEIGHT * 0.78"
        text-anchor="middle"
        :fill="key.textColor"
        class="key-char"
      >{{ key.char }}</text>
    </g>
  </svg>
</template>

<style scoped>
svg {
  user-select: none;
}
.key-group {
  cursor: pointer;
}
.key-char {
  font-size: 36px;
  pointer-events: none;
  font-family: 'Noto Sans SC', 'SimHei', sans-serif;
}
.gap-text {
  font-family: monospace;
  font-size: 18px;
  font-weight: bold;
}
.gap-text-extra {
  font-size: 14px;
}
</style>
