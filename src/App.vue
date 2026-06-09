<template>
  <div class="app">
    <h1>🏋️‍♀️ Strength Garden</h1>

    <div class="card">
      <h2>Add Exercise</h2>

      <div class="add-row">
        <input
          v-model="newExercise"
          placeholder="e.g. Bulgarian Split Squats"
          @keyup.enter="addExercise"
        />

        <button @click="addExercise">Add</button>
      </div>
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
          <span>{{ exercise.name }}</span>
        </label>

        <button
          class="delete-button"
          @click="deleteExercise(exercise.id)"
        >
          🗑️
        </button>
      </div>
    </div>

    <div class="card">
      <h2>Your Garden</h2>

      <div class="garden">
        {{ garden }}
      </div>

      <p>{{ completedCount }} exercises completed today</p>
      <p>🔥 {{ streak }} day streak</p>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'

const newExercise = ref('')
const streak = ref(3)

const exercises = ref(
  JSON.parse(localStorage.getItem('exercises')) || [
    { id: 1, name: 'Squats', completed: false },
    { id: 2, name: 'Romanian Deadlifts', completed: false },
    { id: 3, name: 'Core Workout', completed: false }
  ]
)

function addExercise() {
  if (!newExercise.value.trim()) return

  exercises.value.push({
    id: Date.now(),
    name: newExercise.value,
    completed: false
  })

  newExercise.value = ''
}

function deleteExercise(id) {
  exercises.value = exercises.value.filter(
    exercise => exercise.id !== id
  )
}

watch(
  exercises,
  (value) => {
    localStorage.setItem('exercises', JSON.stringify(value))
  },
  { deep: true }
)

const completedCount = computed(() => {
  return exercises.value.filter(exercise => exercise.completed).length
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
  background: #0f172a;
  color: #f8fafc;
  margin: 0;
}

.app {
  max-width: 700px;
  margin: auto;
  padding: 2rem;
}

h1 {
  text-align: center;
  color: #f8fafc;
}

h2 {
  text-align: center;
  color: #f8fafc;
}

.card {
  background: #1e293b;
  padding: 1.5rem;
  margin-top: 1.5rem;
  border-radius: 16px;
}

.add-row {
  display: flex;
  gap: 12px;
}

input {
  padding: 12px;
  border-radius: 8px;
  border: 1px solid #475569;
  background: #334155;
  color: white;
  font-size: 1rem;
}

.add-row input {
  flex: 1;
}

input::placeholder {
  color: #cbd5e1;
}

button {
  padding: 12px 18px;
  cursor: pointer;
  border: none;
  border-radius: 8px;
  background: #22c55e;
  color: white;
  font-size: 1rem;
}

button:hover {
  background: #16a34a;
}

.exercise {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
  margin-top: 14px;
  padding: 12px;
  background: #0f172a;
  border-radius: 12px;
}

.exercise label {
  display: flex;
  align-items: center;
  gap: 12px;
  color: #f8fafc;
  font-size: 1.1rem;
}

.exercise input[type="checkbox"] {
  width: 20px;
  height: 20px;
}

.delete-button {
  background: transparent;
  padding: 6px;
  font-size: 1.1rem;
}

.delete-button:hover {
  background: #334155;
}

.garden {
  font-size: 3rem;
  text-align: center;
  margin: 20px 0;
}

p {
  text-align: center;
  font-size: 1.1rem;
}
</style>