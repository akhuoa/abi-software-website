<script setup lang="ts">
import { computed, ref } from 'vue'

defineProps<{ title: string }>()

const repos = ref([
  {
    name: 'mapintegratedvuer',
    url: 'https://github.com/ABI-Software/mapintegratedvuer',
    api: 'https://abi-software.github.io/mapintegratedvuer/',
    demo: ''
  },
  {
    name: 'map-side-bar',
    url: 'https://github.com/ABI-Software/map-sidebar',
    api: 'https://abi-software.github.io/map-sidebar/',
    demo: ''
  },
  {
    name: 'FlatmapVuer',
    url: 'https://github.com/ABI-Software/flatmapvuer',
    api: 'https://abi-software.github.io/flatmapvuer/',
    demo: ''
  },
  {
    name: 'Scaffoldvuer',
    url: 'https://github.com/ABI-Software/scaffoldvuer',
    api: 'https://abi-software.github.io/scaffoldvuer/',
    demo: ''
  },
  {
    name: 'SimulationVuer',
    url: 'https://github.com/ABI-Software/simulationvuer',
    api: 'https://abi-software.github.io/simulationvuer/',
    demo: ''
  },
  {
    name: 'ABI-Software Image Dialog',
    url: 'https://github.com/ABI-Software/image-dialog',
    api: '',
    demo: ''
  },
  {
    name: 'plotvuer',
    url: 'https://github.com/ABI-Software/plotvuer',
    api: 'https://abi-software.github.io/plotvuer/',
    demo: ''
  },
  {
    name: 'ABI-Software Gallery',
    url: 'https://github.com/ABI-Software/mapcore-gallery',
    api: 'https://abi-software.github.io/mapcore-gallery/',
    demo: ''
  },
  {
    name: 'svg-sprite',
    url: 'https://github.com/ABI-Software/svg-sprite',
    api: 'https://abi-software.github.io/svg-sprite/',
    demo: ''
  }
])

let search = ref('')

const filteredRepos = computed(() => {
  return repos.value.filter((repo) =>
    repo.name.toLocaleLowerCase().includes(search.value.toLocaleLowerCase())
  )
})
</script>

<template>
  <h1>{{ title }}</h1>

  <label class="search-box">
    Search:
    <input type="text" v-model="search" />
  </label>

  <div class="cards-container">
    <div class="card" v-for="repo in filteredRepos" :key="repo.name">
      <h3>{{ repo.name }}</h3>
      <p>
        <a :href="repo.url">GitHub repo</a>
      </p>
      <p>
        <a :href="repo.api">API documentation</a>
      </p>
    </div>
  </div>

  <p class="read-the-docs">The collection of useful links.</p>
</template>

<style scoped>
.read-the-docs {
  color: #888;
}
</style>
