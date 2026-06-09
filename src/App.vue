<template>
  <div class="app">
    <h1>🌱 Strength Garden</h1>

    <div class="card">
      <h2>Add Exercise</h2>

      <input
        v-model="newExercise"
        placeholder="e.g. Squats"
        @keyup.enter="addExercise"
      />

      <button @click="addExercise">
        Add
      </button>
    </div>

    <div class="card">
      <h2>Today's Workout</h2>

      <div
        v-for="exercise in exercises"
        :key="exercise.id"
        class="exercise"
      >
        <label>
          <input
            type="checkbox"
            v-model="exercise.completed"
          />
          {{ exercise.name }}
        </label>
      </div>
    </div>

    <div class="card">
      <h2>Your Garden</h2>

      <div class="garden">
        {{ garden }}
      </div>

      <p>{{ completedCount }} exercises completed today</p>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

const newExercise = ref('')

const exercises = ref([
  { id: 1, name: 'Squats', completed: false },
  { id: 2, name: 'Romanian Deadlifts', completed: false },
  { id: 3, name: 'Core Workout', completed: false }
])

function addExercise() {
  if (!newExercise.value.trim()) return

  exercises.value.push({
    id: Date.now(),
    name: newExercise.value,
    completed: false
  })

  newExercise.value = ''
}

const completedCount = computed(() => {
  return exercises.value.filter(e => e.completed).length
})

const garden = computed(() => {
  if (completedCount.value === 0) return '🌱'
  if (completedCount.value === 1) return '🌱 🌱'
  if (completedCount.value === 2) return '🌱 🪴 🌱'
  if (completedCount.value === 3) return '🌱 🪴 🌷 🌱'
  return '🌳 🌷 🪴 🌳'
})
</script>

<style>
body {
  font-family: Arial, sans-serif;
  background: #f4f8f2;
  margin: 0;
}

.app {
  max-width: 700px;
  margin: auto;
  padding: 2rem;
}

h1 {
  text-align: center;
}

.card {
  background: white;
  padding: 1rem;
  margin-top: 1rem;
  border-radius: 12px;
}

input {
  padding: 10px;
  width: 70%;
}

button {
  padding: 10px 16px;
  margin-left: 10px;
}

.exercise {
  margin-top: 10px;
}

.garden {
  font-size: 3rem;
  text-align: center;
  margin: 20px 0;
}
</style>