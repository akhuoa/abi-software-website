<script setup lang="ts">
import HelloWorld from './components/HelloWorld.vue'
</script>

<template>
  <div class="logo-container">
    <a href="https://github.com/abi-software" class="logo">
      <img src="./assets/logo.svg" alt="logo" width="60" height="60" />
      <h1>ABI Software</h1>
    </a>
  </div>
  <HelloWorld orgName="abi-software" pkgPrefix="@abi-software/" />
</template>

<style scoped>
.logo-container {
  margin-bottom: 2rem;
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
</style>
