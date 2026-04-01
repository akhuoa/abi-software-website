<script setup lang="ts">
import { computed, ref, onMounted } from 'vue'
import data from '../data.json'

type Repo = {
  name: string
  url: string
  api: string
  demo: string
  npm: string
  description: string
  version: string
  license: string
  updatedAt: string
  keywords: string[]
  maintainers: Maintainer[]
  readme: string
}

type Maintainer = {
  name?: string
  email?: string
}

type NpmPackageVersion = {
  name?: string
  description?: string
  version?: string
  license?: string
  keywords?: string[]
  maintainers?: Maintainer[]
  readme?: string
  links?: {
    repository?: string
  }
  repository?: {
    url?: string
  }
}

type NpmPackageMetadata = {
  name?: string
  license?: string
  keywords?: string[]
  maintainers?: Maintainer[]
  readme?: string
  time?: {
    modified?: string
  }
  versions?: Record<string, NpmPackageVersion>
  'dist-tags'?: {
    latest?: string
  }
}

const props = defineProps<{
  orgName?: string
  pkgPrefix?: string
}>()

const repos = ref<Repo[]>([])
const search = ref('')
const sortBy = ref<'updated' | 'name'>('updated')
const activeReadmeRepo = ref<Repo | null>(null)
const loading = ref(true)
const error = ref('')

// Helper: Map npm package to repo info, fallback to data.json for extra fields
function mapNpmPackage(pkg: NpmPackageMetadata): Repo {
  const latestVersion = pkg['dist-tags']?.latest
  const latestPackage = latestVersion ? pkg.versions?.[latestVersion] : undefined
  const packageInfo = latestPackage || {}
  const packageName = packageInfo.name || pkg.name || ''

  // Try to find extra info from data.json by name
  const extra = data.find((d) => d.name.toLowerCase() === packageName.replace(props.pkgPrefix || '', '').toLowerCase())
  // Try to extract repo URL from package info
  let repoUrl = ''
  if (packageInfo.links?.repository) {
    repoUrl = packageInfo.links.repository
  } else if (packageInfo.repository?.url) {
    repoUrl = packageInfo.repository.url.replace(/^git\+/, '').replace(/\.git$/, '')
  }

  return {
    name: packageName,
    url: extra?.url || repoUrl || '',
    api: extra?.api || '',
    demo: extra?.demo || '',
    npm: `https://www.npmjs.com/package/${packageName}`,
    description: packageInfo.description || '',
    version: latestVersion || packageInfo.version || '',
    license: packageInfo.license || pkg.license || '',
    updatedAt: pkg.time?.modified || '',
    keywords: packageInfo.keywords || pkg.keywords || [],
    maintainers: packageInfo.maintainers || pkg.maintainers || [],
    readme: packageInfo.readme || pkg.readme || '',
  }
}

function buildFallbackRepo(packageName: string): Repo {
  return {
    name: packageName,
    url: '',
    api: '',
    demo: '',
    npm: `https://www.npmjs.com/package/${packageName}`,
    description: '',
    version: '',
    license: '',
    updatedAt: '',
    keywords: [],
    maintainers: [],
    readme: '',
  }
}

function openReadme(repo: Repo) {
  activeReadmeRepo.value = repo
}

function closeReadme() {
  activeReadmeRepo.value = null
}

function formatUpdatedAt(updatedAt: string) {
  if (!updatedAt) {
    return 'Unknown'
  }

  return new Intl.DateTimeFormat('en-NZ', {
    year: 'numeric',
    month: 'short',
    day: 'numeric',
  }).format(new Date(updatedAt))
}

async function fetchNpmPackages() {
  loading.value = true
  error.value = ''
  try {
    const npmApiUrl = `https://registry.npmjs.org/-/org/${props.orgName}/package`
    const proxyBase = 'https://pmrapp-api-proxy.akya984.workers.dev/cors-proxy?' // to resolve CORS issues
    const proxyUrl = proxyBase + 'target=' + encodeURIComponent(npmApiUrl)
    const res = await fetch(proxyUrl)
    if (!res.ok) throw new Error('Failed to fetch from npmjs')
    const json = await res.json()
    const packageNames = Object.keys(json)
    // Fetch metadata for each package
    const metaPromises = packageNames.map(async (packageName) => {
      const metadataAPIUrl = `https://registry.npmjs.org/${packageName}`
      const metaProxyUrl = proxyBase + 'target=' + encodeURIComponent(metadataAPIUrl)
      try {
        const metaRes = await fetch(metaProxyUrl)
        if (!metaRes.ok) throw new Error('meta fetch failed')
        const meta = await metaRes.json() as NpmPackageMetadata
        return mapNpmPackage(meta)
      } catch {
        // fallback to minimal info if metadata fetch fails
        return buildFallbackRepo(packageName)
      }
    })
    repos.value = await Promise.all(metaPromises)
  } catch (e) {
    const err = e as Error
    error.value = err.message || 'Unknown error'
    // fallback to data.json if fetch fails
    repos.value = data.map((repo) => ({
      ...repo,
      description: '',
      version: '',
      license: '',
      updatedAt: '',
      keywords: [],
      maintainers: [],
      readme: '',
      npm: `https://www.npmjs.com/package/${repo.name}`,
    }))
  } finally {
    loading.value = false
  }
}

onMounted(() => {
  fetchNpmPackages()
})

const filteredRepos = computed(() => {
  const filtered = repos.value.filter((repo) =>
    repo.name.toLowerCase().includes(search.value.toLowerCase())
  )

  return [...filtered].sort((left: Repo, right: Repo) => {
    if (sortBy.value === 'name') {
      return left.name.localeCompare(right.name)
    }

    const leftTime = left.updatedAt ? new Date(left.updatedAt).getTime() : 0
    const rightTime = right.updatedAt ? new Date(right.updatedAt).getTime() : 0

    if (rightTime !== leftTime) {
      return rightTime - leftTime
    }

    return left.name.localeCompare(right.name)
  })
})
</script>

<template>
  <div class="page-header">
    <h2>NPM Packages</h2>
    <div class="toolbar-controls">
      <label class="search-box">
        Search:
        <input type="text" v-model="search" name="search" class="search-input" />
      </label>
      <label class="sort-box">
        Sort:
        <select v-model="sortBy" name="sort" class="sort-select">
          <option value="updated">Latest updated</option>
          <option value="name">Name</option>
        </select>
      </label>
    </div>
  </div>

  <div v-if="loading" class="loading">Loading packages...</div>
  <div v-else-if="error" class="error">{{ error }}</div>

  <div class="cards-container">
    <div class="card" v-for="repo in filteredRepos" :key="repo.name">
      <h3>{{ repo.name }}</h3>
      <p class="card-meta-line">
        <span><strong>v</strong> {{ repo.version || 'Unknown' }}</span>
        <span class="meta-separator">|</span>
        <span><strong>Updated:</strong> {{ formatUpdatedAt(repo.updatedAt) }}</span>
      </p>
      <p v-if="repo.description">{{ repo.description }}</p>
      <div v-if="repo.keywords.length" class="meta-block">
        <strong>Keywords:</strong>
        <div class="keyword-list">
          <span class="keyword-chip" v-for="keyword in repo.keywords" :key="keyword">{{ keyword }}</span>
        </div>
      </div>
      <div v-if="repo.maintainers.length" class="meta-block">
        <strong>Maintainers:</strong>
        <span class="maintainer-links">
          <template v-for="(maintainer, index) in repo.maintainers" :key="`${maintainer.name || 'unknown'}-${maintainer.email || ''}`">
            <a :href="`mailto:${maintainer.email || ''}`">{{ maintainer.name || maintainer.email || 'Unknown' }}</a>
            <span v-if="index < repo.maintainers.length - 1">, </span>
          </template>
        </span>
      </div>
      <button v-if="repo.readme" class="readme-button" type="button" @click="openReadme(repo)">View README</button>
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
      <p>
        <span><strong>License:</strong> {{ repo.license || 'Unknown' }}</span>
      </p>
    </div>
  </div>

  <div v-if="activeReadmeRepo" class="lightbox" @click.self="closeReadme">
    <div class="lightbox-content" role="dialog" aria-modal="true" aria-label="Package README">
      <div class="lightbox-header">
        <h3>{{ activeReadmeRepo.name }} README</h3>
        <button type="button" class="lightbox-close" @click="closeReadme">Close</button>
      </div>
      <pre class="readme-markdown">{{ activeReadmeRepo.readme }}</pre>
    </div>
  </div>
</template>

<style scoped>
.page-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  flex-wrap: wrap;
  padding-left: 2rem;
  padding-right: 2rem;

  h2 {
    margin: 0;
    color: #646cff;
  }
}
.toolbar-controls {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  flex-wrap: wrap;
}
.cards-container {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(min(100%, 18rem), 1fr));
  gap: 1rem;
  padding: 2rem;
}
.card {
  width: auto;
  height: 100%;
  border: 1px solid #ccc;
  border-radius: 8px;
  padding: 1rem;
  text-align: left;
  overflow: hidden;

  h3 {
    margin-top: 0;
    color: #333;
  }
}
.meta-block {
  margin-bottom: 0.75rem;
}
.card-meta-line {
  display: flex;
  align-items: center;
  gap: 0.45rem;
  margin: -0.2rem 0 0.75rem;
  color: #666;
  font-size: 0.9rem;
}
.meta-separator {
  color: #aaa;
}
.keyword-list {
  display: flex;
  flex-wrap: wrap;
  gap: 0.4rem;
  margin-top: 0.4rem;
}
.keyword-chip {
  border: 1px solid #ddd;
  border-radius: 999px;
  padding: 0.1rem 0.6rem;
  font-size: 0.85rem;
  line-height: 1.4;
}
.maintainer-links {
  margin-left: 0.35rem;
}
.readme-button {
  border: 1px solid #646cff;
  color: #646cff;
  background: #fff;
  border-radius: 6px;
  padding: 0.4rem 0.75rem;
  cursor: pointer;
  margin-bottom: 0.75rem;
}
.lightbox {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 1rem;
  z-index: 1000;
}
.lightbox-content {
  width: min(900px, 100%);
  max-height: min(80vh, 900px);
  overflow: auto;
  background: #fff;
  border-radius: 10px;
  border: 1px solid #ddd;
  padding: 1rem;
}
.lightbox-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  margin-bottom: 0.75rem;
}
.lightbox-close {
  border: 1px solid #ddd;
  border-radius: 6px;
  background: #fff;
  padding: 0.35rem 0.65rem;
  cursor: pointer;
}
.readme-markdown {
  margin: 0;
  text-align: left;
  white-space: pre-wrap;
  word-break: break-word;
  line-height: 1.4;
}
.search-input,
.sort-select {
  padding: 0.5rem;
  border-radius: 4px;
  border-color: #ccc;
  font-family: inherit;
  color: inherit;
  font-size: inherit;
}
.search-box,
.sort-box {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
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
