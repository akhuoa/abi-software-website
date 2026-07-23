<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

const SIZE = 10
const CELL_SIZE = 32

// Cell states: 0 = empty, 1 = filled, 2 = falling
const cells = ref<number[]>(Array(SIZE * SIZE).fill(0))

let intervals: ReturnType<typeof setInterval>[] = []
let timeouts: ReturnType<typeof setTimeout>[] = []

onMounted(() => {
  const heights = new Array(SIZE).fill(0)
  let remaining = SIZE * SIZE

  const drop = () => {
    if (remaining <= 0) return

    const available = heights
      .map((h, i) => ({ h, i }))
      .filter(({ h }) => h < SIZE)
    if (available.length === 0) return

    const { i: col } = available[Math.floor(Math.random() * available.length)]
    const targetRow = SIZE - 1 - heights[col]
    heights[col]++
    remaining--

    let row = 0
    cells.value[row * SIZE + col] = 2

    const interval = setInterval(() => {
      if (row < targetRow) {
        cells.value[row * SIZE + col] = 0
        row++
        cells.value[row * SIZE + col] = 2
      } else {
        // Landed
        cells.value[targetRow * SIZE + col] = 1
        clearInterval(interval)
        const t = setTimeout(drop, 30 + Math.random() * 50)
        timeouts.push(t)
      }
    }, 50)

    intervals.push(interval)
  }

  const t = setTimeout(drop, 300)
  timeouts.push(t)
})

onUnmounted(() => {
  intervals.forEach(clearInterval)
  timeouts.forEach(clearTimeout)
})
</script>

<template>
  <div
    class="loading-bricks" role="status" aria-label="Loading packages"
    :style="`--cell-size: ${CELL_SIZE}px; --grid-size: ${SIZE};`"
  >
    <div class="grid">
      <div
        v-for="(state, index) in cells"
        :key="index"
        class="cell"
        :class="{ filled: state === 1, falling: state === 2 }"
      ></div>
    </div>
    <p class="label">Loading packages&hellip;</p>
  </div>
</template>

<style scoped>
.loading-bricks {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1.5rem;
  padding: 4rem 3rem;
}

.grid {
  display: grid;
  grid-template-columns: repeat(var(--grid-size), var(--cell-size));
  grid-template-rows: repeat(var(--grid-size), var(--cell-size));
  gap: 1px;
  background: #ccc;
  border: 1px solid #fff;

}

.cell {
  width: var(--cell-size);
  height: var(--cell-size);
  background: #eee;
  transition: background 0.04s;
}

.cell.filled {
  background: #aaa;
}

.cell.falling {
  background: #bbb;
}

.label {
  margin: 0;
  font-style: italic;
  color: #444;
}
</style>
