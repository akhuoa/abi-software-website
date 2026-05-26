<script setup lang="ts">
import { computed, ref, onMounted } from 'vue'
import DOMPurify from 'dompurify'
import MarkdownIt from 'markdown-it'
import data from '../data.json'
import PackageCard from './PackageCard.vue'

const markdownRenderer = new MarkdownIt({
  html: false,
  linkify: true,
  breaks: true,
})

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
    description: cleanDescription(packageInfo.description || ''),
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

function cleanDescription(rawDescription: string) {
  if (!rawDescription) {
    return ''
  }

  let cleaned = rawDescription.trimStart()

  // Remove leading badge blocks repeatedly (handles multiple badges on one line,
  // separate lines, and partially malformed badge markdown from registry data).
  for (let i = 0; i < 10; i += 1) {
    const before = cleaned

    cleaned = cleaned.replace(/^\s*\[!\[[\s\S]*?\]\([\s\S]*?\)\]\([\s\S]*?(?:\)|$)\s*/i, '')
    cleaned = cleaned.replace(/^\s*!\[[\s\S]*?\]\([\s\S]*?(?:\)|$)\s*/i, '')
    cleaned = cleaned.replace(/^\s*<a[^>]*>\s*<img[\s\S]*?<\/a>\s*/i, '')
    cleaned = cleaned.replace(/^\s*<img[^>]*>\s*/i, '')
    cleaned = cleaned.replace(/^\s*\[[^\]]+\]:\s*\S+\s*/i, '')

    if (cleaned === before) {
      break
    }
  }

  return cleaned.trim()
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

const activeReadmeHtml = computed(() => {
  if (!activeReadmeRepo.value?.readme) {
    return ''
  }

  return DOMPurify.sanitize(markdownRenderer.render(activeReadmeRepo.value.readme))
})
</script>

<template>
  <div class="page-header">
    <div>
      <h2>NPM Packages</h2>
      <p class="description">
        The npm packages are fetched from
        <a href="https://www.npmjs.com/org/abi-software" target="_blank" rel="noopener noreferrer">the npm registry</a>.
      </p>
      <p class="package-count">There are currently <strong>{{ repos.length }}</strong> packages in total.</p>
    </div>
    <div class="toolbar-controls">
      <label class="search-box">
        Search:
        <input type="search" v-model="search" name="search" class="search-input" />
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
    <PackageCard
      v-for="repo in filteredRepos"
      :key="repo.name"
      :repo="repo"
      @open-readme="openReadme"
    />

    <div v-if="!loading && !error && !filteredRepos.length" class="empty-state">
      <p>No packages found matching your search criteria.</p>
    </div>
  </div>

  <div v-if="activeReadmeRepo" class="lightbox" @click.self="closeReadme">
    <div class="lightbox-content" role="dialog" aria-modal="true" aria-label="Package README">
      <div class="lightbox-header">
        <h3>{{ activeReadmeRepo.name }} README</h3>
        <button type="button" class="lightbox-close" @click="closeReadme">Close</button>
      </div>
      <div class="lightbox-body">
        <div class="readme-markdown" v-html="activeReadmeHtml"></div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.page-header {
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  gap: 1rem;
  flex-wrap: wrap;
  padding-left: 2rem;
  padding-right: 2rem;

  h2 {
    margin: 0;
    color: #646cff;
  }

  .description {
    color: #888;
  }
}
.package-count {
  margin-top: 0.5rem;
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
  overflow: hidden;
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
.lightbox-body {
  width: 100%;
  max-height: min(80vh, 900px);
  overflow: auto;
  background: #fff;
  border-radius: 8px;
  border: 1px solid #ddd;
  padding: 1rem;
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
  line-height: 1.4;
}
.readme-markdown :deep(h1),
.readme-markdown :deep(h2),
.readme-markdown :deep(h3),
.readme-markdown :deep(h4),
.readme-markdown :deep(h5),
.readme-markdown :deep(h6) {
  margin: 1.25rem 0 0.75rem;
  line-height: 1.2;
}
.readme-markdown :deep(h1) {
  font-size: 1.75rem;
}
.readme-markdown :deep(h2) {
  font-size: 1.4rem;
}
.readme-markdown :deep(h3) {
  font-size: 1.15rem;
}
.readme-markdown :deep(h1:first-child),
.readme-markdown :deep(h2:first-child),
.readme-markdown :deep(h3:first-child) {
  margin-top: 0;
}
.readme-markdown :deep(p),
.readme-markdown :deep(ul),
.readme-markdown :deep(ol),
.readme-markdown :deep(blockquote) {
  margin: 0 0 1rem;
}
.readme-markdown :deep(ul),
.readme-markdown :deep(ol) {
  padding-left: 1.5rem;
}
.readme-markdown :deep(li + li) {
  margin-top: 0.25rem;
}
.readme-markdown :deep(a) {
  color: #0056d6;
  text-decoration: underline;
  word-break: break-word;
}
.readme-markdown :deep(code) {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, Liberation Mono, monospace;
  font-size: 0.9em;
  background: #f5f6f8;
  border-radius: 4px;
  padding: 0.1rem 0.3rem;
}
.readme-markdown :deep(pre) {
  overflow-x: auto;
  background: #2c3344;
  color: #f9fafb;
  border-radius: 8px;
  padding: 1rem;
  margin: 0 0 1rem;
}
.readme-markdown :deep(pre code) {
  background: transparent;
  color: inherit;
  padding: 0;
}
.readme-markdown :deep(blockquote) {
  border-left: 3px solid #d5d9e0;
  padding-left: 1rem;
  color: #4b5563;
}
.readme-markdown :deep(hr) {
  border: 0;
  border-top: 1px solid #e5e7eb;
  margin: 1.25rem 0;
}
.readme-markdown :deep(table) {
  width: 100%;
  border-collapse: collapse;
  margin: 0 0 1rem;
}
.readme-markdown :deep(th),
.readme-markdown :deep(td) {
  border: 1px solid #e5e7eb;
  padding: 0.5rem 0.75rem;
  text-align: left;
}
.readme-markdown :deep(img) {
  max-width: 100%;
  height: auto;
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
  margin: 2rem;
}
.error {
  color: #c00;
  font-weight: bold;
  margin: 2rem;
}
.empty-state {
  color: #555;
  font-style: italic;
  border: 1px dashed #ccc;
  border-radius: 6px;
  padding: 1rem;

  p {
    margin: 0;
  }
}
</style>
