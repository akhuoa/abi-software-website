<script setup lang="ts">
import { computed, ref } from 'vue'
import DOMPurify from 'dompurify'
import MarkdownIt from 'markdown-it'

type Maintainer = {
  name?: string
  email?: string
}

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

type InstallTool = 'npm' | 'pnpm' | 'yarn' | 'bun'

const props = defineProps<{
  repo: Repo
  installTool: InstallTool
}>()

const emit = defineEmits<{
  openReadme: [repo: Repo]
}>()

const markdownRenderer = new MarkdownIt({
  html: false,
  linkify: true,
  breaks: true,
})

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

const descriptionMarkdownHTML = computed(() => {
  if (!props.repo.description) {
    return ''
  }

  return DOMPurify.sanitize(markdownRenderer.render(props.repo.description))
})

const displayName = computed(() => props.repo.name.replace(/^@[^/]+\//, ''))
const copied = ref(false)

const installCommand = computed(() => {
  switch (props.installTool) {
    case 'pnpm':
      return `pnpm add ${props.repo.name}`
    case 'yarn':
      return `yarn add ${props.repo.name}`
    case 'bun':
      return `bun add ${props.repo.name}`
    case 'npm':
    default:
      return `npm i ${props.repo.name}`
  }
})

async function copyInstallCommand() {
  const command = installCommand.value

  try {
    if (navigator.clipboard?.writeText) {
      await navigator.clipboard.writeText(command)
    } else {
      const textarea = document.createElement('textarea')
      textarea.value = command
      textarea.setAttribute('readonly', '')
      textarea.style.position = 'fixed'
      textarea.style.opacity = '0'
      document.body.appendChild(textarea)
      textarea.select()
      document.execCommand('copy')
      document.body.removeChild(textarea)
    }

    copied.value = true
    setTimeout(() => {
      copied.value = false
    }, 1200)
  } catch {
    copied.value = false
  }
}

function openReadme() {
  emit('openReadme', props.repo)
}
</script>

<template>
  <div class="card">
    <div class="card-header">
      <h3>{{ displayName }}</h3>
      <p v-if="displayName !== repo.name" class="original-package-name">{{ repo.name }}</p>
      <p class="card-meta-line">
        <span><strong>v</strong> {{ repo.version || 'Unknown' }}</span>
        <span class="meta-separator">|</span>
        <span><strong>Updated:</strong> {{ formatUpdatedAt(repo.updatedAt) }}</span>
      </p>
    </div>

    <div class="card-body">
      <div v-if="repo.description" class="card-description">
        <div class="readme-markdown" v-html="descriptionMarkdownHTML"></div>
      </div>

      <button v-if="repo.readme" class="readme-button" type="button" @click="openReadme">
        README
      </button>

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
            <a :href="`https://www.npmjs.com/~${maintainer.name || ''}`" target="_blank">{{ maintainer.name || maintainer.email || 'Unknown' }}</a>
            <span v-if="index < repo.maintainers.length - 1">, </span>
          </template>
        </span>
      </div>
    </div>

    <div class="card-footer">
      <div class="install-row">
        <code class="install-command">{{ installCommand }}</code>
        <button
          type="button"
          class="copy-button"
          :aria-label="copied ? 'Install command copied' : 'Copy install command'"
          :title="copied ? 'Copied' : 'Copy command'"
          @click="copyInstallCommand"
        >
          <svg v-if="!copied" class="copy-button-icon" viewBox="0 0 24 24" aria-hidden="true">
            <path
              fill="currentColor"
              d="M8 7V3h12v14h-4v4H4V7zm2-2v2h8V5zm-4 4v10h8V9z"
            />
          </svg>
          <svg v-else class="copy-button-icon" viewBox="0 0 24 24" aria-hidden="true">
            <path
              fill="currentColor"
              d="M9.55 18.2 4.9 13.55l1.41-1.41 3.24 3.23 8.14-8.14 1.41 1.42z"
            />
          </svg>
        </button>
      </div>

      <div class="action-row">
        <a
          v-if="repo.url"
          class="action-button github"
          :href="repo.url"
          target="_blank"
          rel="noopener"
        >
          <svg class="action-icon" viewBox="0 0 24 24" aria-hidden="true">
            <path
              fill="currentColor"
              d="M12 .5a12 12 0 0 0-3.79 23.39c.6.11.82-.26.82-.58v-2.03c-3.34.73-4.04-1.42-4.04-1.42-.55-1.38-1.33-1.75-1.33-1.75-1.08-.75.08-.74.08-.74 1.2.08 1.83 1.22 1.83 1.22 1.06 1.8 2.79 1.28 3.47.98.11-.76.41-1.28.74-1.57-2.67-.3-5.47-1.32-5.47-5.87 0-1.29.46-2.34 1.22-3.17-.12-.3-.53-1.52.12-3.16 0 0 1-.32 3.3 1.21a11.62 11.62 0 0 1 6.01 0c2.3-1.53 3.3-1.21 3.3-1.21.65 1.64.24 2.86.12 3.16.76.83 1.22 1.88 1.22 3.17 0 4.56-2.8 5.57-5.48 5.87.42.36.8 1.08.8 2.18v3.23c0 .32.22.69.82.58A12 12 0 0 0 12 .5Z"
            />
          </svg>
          GitHub
        </a>

        <a
          v-if="repo.api"
          class="action-button api"
          :href="repo.api"
          target="_blank"
          rel="noopener"
        >
          <svg class="action-icon" viewBox="0 0 24 24" aria-hidden="true">
            <path
              fill="currentColor"
              d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8zm0 2.5L18.5 9H14zM8 13h8v1.5H8zm0 3h8v1.5H8zm0-6h5v1.5H8z"
            />
          </svg>
          API Docs
        </a>

        <a
          class="action-button npm"
          :href="repo.npm"
          target="_blank"
          rel="noopener"
        >
          <svg class="action-icon" viewBox="0 0 24 24" aria-hidden="true">
            <path
              fill="currentColor"
              d="M2 7.5v9h20v-9zm1.5 1.5h17v6h-2.5v-4.5h-4v4.5H9.5v-4.5h-4v4.5H3.5Z"
            />
          </svg>
          NPM
        </a>
      </div>
      <p class="card-meta-line license">
        <span><strong>License:</strong> {{ repo.license || 'Unknown' }}</span>
      </p>
    </div>
  </div>
</template>

<style scoped>
.card {
  width: auto;
  height: 100%;
  display: flex;
  flex-direction: column;
  align-items: start;
  background: linear-gradient(135deg, #c7c9f426 0%, #646cff3e 100%);
  border-radius: 8px;
  padding: 1rem;
  text-align: left;
  overflow: hidden;
  gap: 1rem;

  h3 {
    margin-top: 0;
    margin-bottom: 0.25rem;
  }

  p {
    margin: 0;
  }
}
.card-header,
.card-body,
.card-footer {
  width: 100%;
}
.card-footer {
  margin-top: auto;
  border-top: 1px solid #fafafa;
  padding-top: 1.5rem;
}
.card-description {
  margin: 0.75rem 0;
}
.install-row {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.5rem;
  margin: 0 0 0.75rem;
  padding: 0.45rem 0.55rem;
  background: #f6f7f9;
  border: 1px solid #e5e7eb;
  border-radius: 6px;
}
.install-command {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, Liberation Mono, monospace;
  font-size: 0.82rem;
  color: #1f2937;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.copy-button {
  border: 1px solid #c9ccd2;
  background: #fff;
  color: #374151;
  border-radius: 5px;
  width: 1.9rem;
  height: 1.9rem;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  cursor: pointer;
  flex-shrink: 0;
}
.copy-button:hover {
  border-color: #9aa2af;
}
.copy-button-icon {
  width: 0.95rem;
  height: 0.95rem;
}
.original-package-name {
  margin: -0.25rem 0 0.6rem;
  color: #6b7280;
  font-size: 0.8rem;
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, Liberation Mono, monospace;
  letter-spacing: 0.01em;
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
  padding: 0.35rem 0.6rem;
  cursor: pointer;
  font-size: 0.85rem;
  text-decoration: none;
  margin: 0.75rem 0;
}
.action-row {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-top: 1.5rem;
  margin-bottom: 0.75rem;
}
.action-button {
  display: inline-flex;
  align-items: center;
  gap: 0.35rem;
  border: 1px solid #d5d5d5;
  border-radius: 6px;
  padding: 0.35rem 0.6rem;
  font-size: 0.85rem;
  text-decoration: none;
  transition: background-color 0.2s ease, border-color 0.2s ease;
}
.action-button.github {
  color: #24292f;
  background: #f6f8fa;
}
.action-button.api {
  color: #0056d6;
  background: #eef4ff;
}
.action-button.npm {
  color: #b51f2d;
  background: #fff1f2;
}
.action-button:hover {
  border-color: #b9b9b9;
}
.action-icon {
  width: 0.95rem;
  height: 0.95rem;
}

.readme-markdown {
  margin: 0;
  text-align: left;
  line-height: 1.4;
}
.readme-markdown :deep(p),
.readme-markdown :deep(ul),
.readme-markdown :deep(ol),
.readme-markdown :deep(blockquote) {
  margin: 0;
}
.readme-markdown :deep(a) {
  color: #0056d6;
  text-decoration: underline;
  word-break: break-word;
}
</style>
