<script setup lang="ts">
import { computed, ref, onMounted } from 'vue'
import data from '../data.json'

defineProps<{ title: string }>()

const repos = ref([] as Array<any>)
const search = ref('')
const loading = ref(true)
const error = ref('')

// Helper: Map npm package to repo info, fallback to data.json for extra fields
function mapNpmPackage(pkg: any) {
  // Try to find extra info from data.json by name
  const extra = data.find((d) => d.name.toLowerCase() === pkg.name.replace('@abi-software/', '').toLowerCase())
  // Try to extract repo URL from package info
  let repoUrl = ''
  if (pkg.links && pkg.links.repository) {
    repoUrl = pkg.links.repository
  } else if (pkg.repository && pkg.repository.url) {
    repoUrl = pkg.repository.url.replace(/^git\+/, '').replace(/\.git$/, '')
  }
  return {
    name: pkg.name.replace('@abi-software/', ''),
    url: extra?.url || repoUrl || '',
    api: extra?.api || '',
    demo: extra?.demo || '',
    npm: `https://www.npmjs.com/package/${pkg.name}`,
    description: pkg.description || '',
    version: pkg.version || '',
  }
}

async function fetchNpmPackages() {
  loading.value = true
  error.value = ''
  try {
    // Fetch ABI-Software org packages from npmjs API
    const res = await fetch('https://registry.npmjs.org/-/v1/search?text=scope:abi-software&size=100')
    if (!res.ok) throw new Error('Failed to fetch from npmjs')
    const json = await res.json()
    // Map each package to our display format
    repos.value = json.objects.map((obj: any) => mapNpmPackage(obj.package))
  } catch (e: any) {
    error.value = e.message || 'Unknown error'
    // fallback to data.json if fetch fails
    repos.value = data
  } finally {
    loading.value = false
  }
}

onMounted(() => {
  fetchNpmPackages()
})

const filteredRepos = computed(() => {
  return repos.value.filter((repo) =>
    repo.name.toLowerCase().includes(search.value.toLowerCase())
  )
})
</script>

<template>
  <h1>{{ title }}</h1>

  <label class="search-box">
    Search:
    <input type="text" v-model="search" />
  </label>

  <div v-if="loading" class="loading">Loading packages...</div>
  <div v-else-if="error" class="error">{{ error }}</div>

  <div class="cards-container">
    <div class="card" v-for="repo in filteredRepos" :key="repo.name">
      <h3>{{ repo.name }}</h3>
      <p v-if="repo.description">{{ repo.description }}</p>
      <p v-if="repo.version"><strong>Version:</strong> {{ repo.version }}</p>
      <p>
        <a v-if="repo.url" :href="repo.url" target="_blank" rel="noopener">GitHub repo</a>
        <span v-else>No GitHub repo</span>
      </p>
      <p v-if="repo.api">
        <a :href="repo.api" target="_blank" rel="noopener">API documentation</a>
      </p>
      <p>
        <a :href="repo.npm" target="_blank" rel="noopener">NPM package</a>
      </p>
    </div>
  </div>

  <p class="read-the-docs">The collection of useful links.</p>
</template>

<style scoped>
.read-the-docs {
  color: #888;
}
.loading {
  color: #555;
  font-style: italic;
  margin: 1em 0;
}
.error {
  color: #c00;
  font-weight: bold;
  margin: 1em 0;
}
</style>
