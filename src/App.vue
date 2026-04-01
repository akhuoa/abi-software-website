<script setup lang="ts">
import PackageDirectory from './components/PackageDirectory.vue'
</script>

<template>
  <div class="logo-container">
    <a href="https://github.com/abi-software" class="logo">
      <img src="./assets/logo.svg" alt="logo" width="60" height="60" />
      <h1>ABI Software</h1>
    </a>
  </div>
  <PackageDirectory orgName="abi-software" pkgPrefix="@abi-software/" />
  <div class="read-the-docs">The collection of useful links.</div>
</template>

<style scoped>
.logo-container {
  padding: 2rem;
  margin-bottom: 2rem;
  text-align: center;
  background-color: #646cff26;
}

.logo {
  display: inline-flex;
  align-items: center;
  text-decoration: none;
  gap: 1rem;
}

.logo img {
  will-change: filter;
  transition: filter 0.3s ease;
}

.logo:hover img {
  filter: drop-shadow(0 0 2em #646cffaa);
}

.logo h1 {
  margin: 0;
  color: #646cff;
}

.read-the-docs {
  color: #888;
  padding: 2rem;
  text-align: center;
}
</style>
