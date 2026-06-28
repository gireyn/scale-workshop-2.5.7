<script setup lang="ts">
import VirtualTiangans from '@/components/VirtualTiangans.vue'
import { useStateStore } from '@/stores/state'

defineProps<{
  noteOn: NoteOnCallback
}>()

const state = useStateStore()

type NoteOff = () => void
type NoteOnCallback = (index: number) => NoteOff
</script>

<template>
  <main>
    <VirtualTiangans
      :baseIndex="state.baseIndex"
      :noteOn="noteOn"
      :heldNotes="state.heldNotes"
    ></VirtualTiangans>
  </main>
</template>
