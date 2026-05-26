<script setup lang="ts">
import PackageDirectory from './components/PackageDirectory.vue'
</script>

<template>
  <div class="logo-container">
    <h1>
      <div class="logo">
        <img src="./assets/logo.svg" alt="logo" width="60" height="60" />
        ABI Software
      </div>
    </h1>
    <p>The collection of npm packages, GitHub repositories, and tools
      developed by <a href="https://github.com/abi-software" target="_blank" rel="noopener noreferrer">ABI-Software</a>.</p>
  </div>
  <PackageDirectory orgName="abi-software" pkgPrefix="@abi-software/" />
  <div class="footer">Disclaimer: This is an unofficial, for-fun project.</div>
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
  font-weight: bold;
}

.logo img {
  will-change: filter;
  transition: filter 0.3s ease;
}

.logo:hover img {
  filter: drop-shadow(0 0 2em #646cffaa);
}

h1 {
  margin: 0;
  color: #646cff;
}

p {
  color: #888;
  margin-top: 0.5rem;
  margin-bottom: 0;
}

.footer {
  color: #888;
  padding: 2rem;
  text-align: center;
  font-size: 0.9rem;
  border-top: 1px solid #eee;
  background-color: #f9f9f9;
}
</style>
