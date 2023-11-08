<template>
  <div id="app">
    <euromap-converter></euromap-converter>
  </div>
</template>

<script>
import EuromapConverter from "./components/converter.vue"; // Use the correct relative path

export default {
  components: {
    "euromap-converter": EuromapConverter,
  },
};
</script>
