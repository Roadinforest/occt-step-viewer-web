<template>
  <div id="app">
    <OcctImportDemo />
  </div>
</template>

<script>
import OcctImportDemo from './components/OcctImportDemo.vue';

export default {
  name: 'App',
  components: {
    OcctImportDemo,
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
  margin-top: 30px; /* Slightly adjust margin-top from default */
}
</style>