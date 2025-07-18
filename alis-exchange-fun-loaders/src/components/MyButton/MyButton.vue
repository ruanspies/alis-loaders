<!-- src/components/MyButton/MyButton.vue -->
<template>
  <button class="my-button" :style="{ backgroundColor: color }">
    <slot>My Button</slot>
  </button>
</template>

<script setup lang="ts">
defineProps<{
  color?: string
}>()
</script>

<style scoped>
.my-button {
  padding: 10px 20px;
  border: none;
  border-radius: 8px;
  color: white;
  background-color: #42b983; /* Default color moved to CSS */
  font-weight: bold;
  cursor: pointer;
  transition: transform 0.2s;
}

.my-button:hover {
  transform: scale(1.05);
}
</style>