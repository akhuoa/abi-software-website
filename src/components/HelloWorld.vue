<script setup lang="ts">
import { computed, ref } from 'vue'
import data from '../data.json'

defineProps<{ title: string }>()

const repos = ref(data)

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
