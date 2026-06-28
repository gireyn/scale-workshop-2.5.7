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
  <svg width="100%" height="100%">
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
        :x="(key.x * 100 / NUM_KEYS) + '%'"
        y="5%"
        :width="(100 / NUM_KEYS) + '%'"
        height="90%"
        :fill="key.fill"
        stroke="gray"
        stroke-width="1.5"
        rx="3"
      />
      <text
        :x="((key.x + 0.5) * 100 / NUM_KEYS) + '%'"
        y="82%"
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
  font-size: 14px;
  pointer-events: none;
  font-family: 'Noto Sans SC', 'SimHei', sans-serif;
}
</style>
