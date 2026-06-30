<script setup lang="ts">
import { computed } from 'vue'
import { dot, kCombinations } from 'xen-dev-utils'
import { monzoEuclideanDistance } from '@/utils'

// Kraig Grady's coordinate system for primes up to 23
const horizontalCoords = [-23, 40, 0, 13, -14, -8, -5, 7, 20]
const verticalCoords = [-45, 0, -40, -11, -18, -4, -32, -25, -6]

// Tiangan colors matching VirtualTiangans.vue
const TIANGAN_COLORS = [
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

function darkenColor(color: string) {
  const rgb = parseRgb(color)
  return `rgb(${Math.round(rgb[0] / 2)}, ${Math.round(rgb[1] / 2)}, ${Math.round(rgb[2] / 2)})`
}

// The specific scale data as specified by the user
interface ScaleItem {
  char: string
  ratio: string
  monzo: number[]
}

// The Tiangan cycle: 甲→乙→丙→丁→戊→己→庚→辛→壬→癸→甲
// 1/1 is 甲, 2/1 is also 甲 (period), 16/15 is 乙
const scaleData: ScaleItem[] = [
  { char: '甲', ratio: '1/1', monzo: [0, 0, 0, 0, 0, 0, 0, 0, 0] },
  { char: '乙', ratio: '16/15', monzo: [4, -1, -1, 0, 0, 0, 0, 0, 0] },
  { char: '丙', ratio: '75/64', monzo: [-6, 1, 2, 0, 0, 0, 0, 0, 0] },
  { char: '丁', ratio: '5/4', monzo: [-2, 0, 1, 0, 0, 0, 0, 0, 0] },
  { char: '戊', ratio: '4/3', monzo: [2, -1, 0, 0, 0, 0, 0, 0, 0] },
  { char: '己', ratio: '45/32', monzo: [-5, 2, 1, 0, 0, 0, 0, 0, 0] },
  { char: '庚', ratio: '3/2', monzo: [-1, 1, 0, 0, 0, 0, 0, 0, 0] },
  { char: '辛', ratio: '8/5', monzo: [3, 0, -1, 0, 0, 0, 0, 0, 0] },
  { char: '壬', ratio: '225/128', monzo: [-7, 2, 2, 0, 0, 0, 0, 0, 0] },
  { char: '癸', ratio: '15/8', monzo: [-3, 1, 1, 0, 0, 0, 0, 0, 0] },
  { char: '甲', ratio: '2/1', monzo: [1, 0, 0, 0, 0, 0, 0, 0, 0] },
]

function getColorIndex(char: string): number {
  return TIANGAN_COLORS.findIndex((t) => t.char === char)
}

interface LatticeNode {
  x: number
  y: number
  char: string
  ratio: string
  color: string
  darkColor: string
  index: number
  isHeld: boolean
}

const props = defineProps<{
  heldNotes: Map<number, number>
  baseIndex: number
}>()

// Determine if a Tiangan character is currently pressed on the keyboard.
// The VirtualTiangans keyboard has 41 keys, positions 0-40.
// Key at position i has note index = baseIndex + i - 10
// Key at position i has char = TIANGAN[i % 10]
function isCharHeld(char: string): boolean {
  const colorIdx = getColorIndex(char)
  if (colorIdx < 0) return false
  // All keyboard positions that show this character
  for (let i = colorIdx; i < 41; i += 10) {
    const noteIndex = props.baseIndex + i - 10
    if ((props.heldNotes.get(noteIndex) || 0) > 0) {
      return true
    }
  }
  return false
}

const nodes = computed<LatticeNode[]>(() => {
  return scaleData.map((item, index) => {
    const vector = [...item.monzo]
    vector[0] = 0 // zero the equave (prime 2)
    // Shift right by 1/3 edge, down by 1/6 edge (grid edge = 40)
    const x = dot(horizontalCoords, vector) + 40 / 3
    const y = dot(verticalCoords, vector) + 40 / 6
    const colorIdx = getColorIndex(item.char)
    let color = '#888888'
    if (colorIdx >= 0) {
      color = TIANGAN_COLORS[colorIdx].color
    }
    const isHeld = isCharHeld(item.char)
    return {
      x,
      y,
      char: item.char,
      ratio: item.ratio,
      color,
      darkColor: darkenColor(color),
      index,
      isHeld,
    }
  })
})

const edges = computed(() => {
  const combinations = kCombinations(scaleData, 2)
  return combinations
    .map(([a, b]) => {
      const vectorA = [...a.monzo]
      vectorA[0] = 0
      const vectorB = [...b.monzo]
      vectorB[0] = 0
      return {
        distance: monzoEuclideanDistance(0, vectorA, vectorB),
        pair: [a.char || a.ratio, b.char || b.ratio]
      }
    })
    .filter(({ distance }) => distance === 1)
})

function findNode(identifier: string): LatticeNode | undefined {
  return nodes.value.find((n) => n.char === identifier || n.ratio === identifier)
}

const edgePairs = computed(() =>
  edges.value.map((e) => {
    const n1 = findNode(e.pair[0])
    const n2 = findNode(e.pair[1])
    return { x1: n1?.x ?? 0, y1: n1?.y ?? 0, x2: n2?.x ?? 0, y2: n2?.y ?? 0 }
  })
)

const viewBox = computed(() => {
  if (!nodes.value.length) return '0 0 100 100'
  const xs = nodes.value.map((n) => n.x)
  const ys = nodes.value.map((n) => n.y)
  const margin = 5  // Tight margin around nodes (left, top, bottom)
  const minX = Math.min(...xs) - margin
  const maxX = Math.max(...xs) + margin
  const minY = Math.min(...ys) - margin * 2
  const maxY = Math.max(...ys) + margin * 2.5
  const legendPadding = 57  // 2/3 of previous to match legend size reduction
  return `${minX} ${minY} ${maxX - minX + legendPadding} ${maxY - minY}`
})

// Legend configuration (2/3 of previous size)
// Origin shifted right by 12 and down by 12 (1 edge at previous size)
const LEGEND_ORIGIN_X = 142
const LEGEND_ORIGIN_Y = -48
const LEGEND_GRID = 8

interface LegendVec {
  dx: number
  dy: number
  label: string
  color: string
}
// Legend vectors in SVG coordinates (+y = DOWN in SVG, but user convention: +y = UP)
// User (dx, dy) → SVG: (dx, -dy)
const legendVectors: LegendVec[] = [
  { dx: 1, dy: 0, label: '3', color: '#e74c3c' },   // (1,0) right 1
  { dx: 0, dy: -1, label: '5', color: '#2980b9' },   // (0,1) up 1 → SVG dy=-1
  { dx: 2, dy: -2, label: '7', color: '#27ae60' },    // (2,2) right 2, up 2 → SVG dy=-2
  { dx: -1, dy: 3, label: '11', color: '#8e44ad' },   // (-1,-3) left 1, down 3 → SVG dy=3
]

// Legend grid dots in SVG coordinates.
// User convention: +x = right, +y = up. In SVG: +x = right, +y = down.
// Upper right 2x2: user coords x=[0,2], y=[2,0] (2 grids up from origin)
// → SVG: x=[0,2], y=[-2,0]
const upperRightDots: [number, number][] = []
for (let x = 0; x <= 2; x++) {
  for (let y = -2; y <= 0; y++) {
    upperRightDots.push([x, y])
  }
}

// Lower left 1x3: user coords x=[-1,0], y=[0,3] → SVG: x=[-1,0], y=[0,-3]
const lowerLeftDots: [number, number][] = []
for (let x = -1; x <= 0; x++) {
  for (let y = 0; y <= 3; y++) {
    lowerLeftDots.push([x, y])
  }
}
</script>

<template>
  <div class="lattice-container">
    <svg
      xmlns="http://www.w3.org/2000/svg"
      :viewBox="viewBox"
      preserveAspectRatio="xMidYMid meet"
      class="lattice-svg"
    >
      <!-- Edges -->
      <line
        v-for="(edge, i) of edgePairs"
        :key="'edge-' + i"
        class="edge"
        :x1="edge.x1"
        :y1="edge.y1"
        :x2="edge.x2"
        :y2="edge.y2"
      />

      <!-- Nodes -->
      <circle
        v-for="(node, i) of nodes"
        :key="'node-' + i"
        :cx="node.x"
        :cy="node.y"
        r="5"
        :fill="node.isHeld ? node.darkColor : node.color"
        stroke="#666"
        stroke-width="0.5"
      />

      <!-- Labels -->
      <text
        v-for="(node, i) of nodes"
        :key="'label-' + i"
        :x="node.x + 0"
        :y="node.y + 2.5"
        class="node-label"
      ><template v-if="node.char">{{ node.char }}</template></text>

      <!-- ===== LEGEND ===== -->
      <g class="legend">
        <!-- Grid lines: upper right 2×2 (2 grids up from origin) -->
        <line
          v-for="x in [0, 1, 2]"
          :key="'lgv-' + x"
          :x1="LEGEND_ORIGIN_X + x * LEGEND_GRID"
          :y1="LEGEND_ORIGIN_Y - 2 * LEGEND_GRID"
          :x2="LEGEND_ORIGIN_X + x * LEGEND_GRID"
          :y2="LEGEND_ORIGIN_Y"
          class="legend-grid"
        />
        <line
          v-for="y in [-2, -1, 0]"
          :key="'lgh-' + y"
          :x1="LEGEND_ORIGIN_X"
          :y1="LEGEND_ORIGIN_Y + y * LEGEND_GRID"
          :x2="LEGEND_ORIGIN_X + 2 * LEGEND_GRID"
          :y2="LEGEND_ORIGIN_Y + y * LEGEND_GRID"
          class="legend-grid"
        />

        <!-- Grid lines: lower left 1×3 (user: -x left, -y down → SVG: -x left, +y down) -->
        <line
          v-for="x in [-1, 0]"
          :key="'lgv2-' + x"
          :x1="LEGEND_ORIGIN_X + x * LEGEND_GRID"
          :y1="LEGEND_ORIGIN_Y"
          :x2="LEGEND_ORIGIN_X + x * LEGEND_GRID"
          :y2="LEGEND_ORIGIN_Y + 3 * LEGEND_GRID"
          class="legend-grid"
        />
        <line
          v-for="y in [0, 1, 2, 3]"
          :key="'lgh2-' + y"
          :x1="LEGEND_ORIGIN_X - LEGEND_GRID"
          :y1="LEGEND_ORIGIN_Y + y * LEGEND_GRID"
          :x2="LEGEND_ORIGIN_X"
          :y2="LEGEND_ORIGIN_Y + y * LEGEND_GRID"
          class="legend-grid"
        />

        <!-- Grid dots: upper right 2×2 -->
        <circle
          v-for="(pos, pi) of upperRightDots"
          :key="'gd-' + pi"
          :cx="LEGEND_ORIGIN_X + pos[0] * LEGEND_GRID"
          :cy="LEGEND_ORIGIN_Y + pos[1] * LEGEND_GRID"
          r="0.9"
          class="legend-dot"
        />

        <!-- Grid dots: lower left 1×3 -->
        <circle
          v-for="(pos, pi) of lowerLeftDots"
          :key="'gd2-' + pi"
          :cx="LEGEND_ORIGIN_X + pos[0] * LEGEND_GRID"
          :cy="LEGEND_ORIGIN_Y + pos[1] * LEGEND_GRID"
          r="0.9"
          class="legend-dot"
        />

        <!-- Arrow markers (defs) -->
        <defs>
          <marker
            v-for="(vec, vi) of legendVectors"
            :key="'mkr-' + vi"
            :id="'larr-' + vi"
            markerWidth="4"
            markerHeight="4"
            refX="4"
            refY="2"
            orient="auto"
            markerUnits="userSpaceOnUse"
          >
            <polygon :points="'0 0, 4 2, 0 4'" :fill="vec.color" />
          </marker>
        </defs>

        <!-- Legend arrow vectors -->
        <g v-for="(vec, vi) of legendVectors" :key="'vec-' + vi">
          <line
            :x1="LEGEND_ORIGIN_X"
            :y1="LEGEND_ORIGIN_Y"
            :x2="LEGEND_ORIGIN_X + vec.dx * LEGEND_GRID"
            :y2="LEGEND_ORIGIN_Y + vec.dy * LEGEND_GRID"
            :stroke="vec.color"
            stroke-width="0.9"
            :marker-end="'url(#larr-' + vi + ')'"
          />
          <text
            :x="LEGEND_ORIGIN_X + vec.dx * LEGEND_GRID + 2"
            :y="LEGEND_ORIGIN_Y + vec.dy * LEGEND_GRID - 0.75"
            class="legend-label legend-label-sm"
            :fill="vec.color"
          >
            {{ vec.label }}
          </text>
        </g>
      </g>
    </svg>
  </div>
</template>

<style scoped>
.lattice-container {
  width: 100%;
  height: 100%;
  overflow: hidden;
}
.lattice-svg {
  width: 100%;
  height: 100%;
  background-color: transparent;
}
.edge {
  stroke: var(--color-text);
  stroke-width: 0.5;
  opacity: 0.5;
}
.node-label {
  font-family: 'Noto Sans SC', 'SimHei', sans-serif;
  font-size: 6px;
  font-weight: bold;
  fill: #000;
  text-anchor: middle;
  stroke: var(--color-background);
  stroke-width: 0.3;
}
.legend-grid {
  stroke: #999;
  stroke-width: 0.3;
  opacity: 0.4;
}
.legend-dot {
  fill: #999;
  opacity: 0.5;
}
.legend-label {
  font-family: sans-serif;
  font-size: 6px;
  font-weight: bold;
}
.legend-label-sm {
  font-size: 4px;
}
.legend-label-sm {
  font-size: 4px;
}
</style>
