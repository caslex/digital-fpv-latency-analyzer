<script setup lang="ts">
import { RouterLink, RouterView } from "vue-router";
</script>

<template>
  <header>
    <!--
    <nav>
      <RouterLink to="/">Home</RouterLink>
    </nav>
    -->
  </header>

  <RouterView />
</template>

<style lang="scss">
body {
  padding: 2rem;
}
</style>
