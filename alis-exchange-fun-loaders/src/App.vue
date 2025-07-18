<!-- src/App.vue -->
<template>
  <div>
    <h1>My Vue Component Library</h1>
    <p>Testing the components locally.</p>
    
    <div class="component-showcase">
      <h3>MyButton Component</h3>
      <MyButton>Click Me!</MyButton>
      <MyButton color="#e74c3c" style="margin-left: 10px;">Red Button</MyButton>
      <MyButton color="#3498db" style="margin-left: 10px;">Blue Button</MyButton>
    </div>
  </div>
</template>

<script setup lang="ts">
import { MyButton } from './index'; // Import from your library's entry point
</script>

<style>
/* Add some basic styling for the test page */
body {
  font-family: sans-serif;
  padding: 2rem;
}
.component-showcase {
  margin-top: 2rem;
  padding: 1rem;
  border: 1px solid #ccc;
  border-radius: 8px;
}
</style>