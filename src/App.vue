<template>
  <div class="app">
    <button class="theme-toggle" @click="isDark = !isDark">
      {{ isDark ? '☀️' : '🌙' }}
    </button>

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
      <h2>Athlete Progress</h2>

      <p class="level">🏆 {{ athleteLevel }}</p>

      <div class="progress-bar">
        <div
          class="progress-fill"
          :style="{ width: progressPercentage + '%' }"
        ></div>
      </div>

      <p>{{ progressPercentage }}% completed today</p>
      <p>⭐ {{ xp }} XP</p>
      <p>🔥 {{ streak }} day streak</p>
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
          <span :class="{ done: exercise.completed }">
            {{ exercise.name }}
          </span>
        </label>

        <button
          class="delete-button"
          @click="deleteExercise(exercise.id)"
        >
          🗑️
        </button>
      </div>

      <p v-if="exercises.length === 0">
        No exercises yet. Add your first one above 🌱
      </p>
    </div>

    <div class="card">
      <h2>Your Garden</h2>

      <div class="garden">
        {{ garden }}
      </div>

      <p>{{ completedCount }} exercises completed today</p>
    </div>

    <div class="card">
      <h2>Achievements</h2>

      <div class="badges">
        <div
          v-for="achievement in achievements"
          :key="achievement.name"
          class="badge"
          :class="{ locked: !achievement.unlocked }"
        >
          <span>{{ achievement.icon }}</span>
          <strong>{{ achievement.name }}</strong>
          <small>{{ achievement.description }}</small>
        </div>
      </div>
    </div>

    <button
      class="rest-button"
      :class="{ active: restDay }"
      @click="toggleRestDay"
    >
      {{ restDay ? '✅ Rest Day (+5 XP)' : '😴 Mark Rest Day' }}
    </button>

    <button class="reset-button" @click="resetToday">
      Reset Today's Workout
    </button>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'

const today = new Date().toDateString()

function yesterday() {
  const date = new Date()
  date.setDate(date.getDate() - 1)
  return date.toDateString()
}

function load(key, fallback) {
  try {
    const raw = localStorage.getItem(key)
    return raw === null ? fallback : JSON.parse(raw)
  } catch {
    return fallback
  }
}

const savedExercises = load('exercises', null)
const savedDate = localStorage.getItem('lastWorkoutDate')
const isNewDay = savedDate !== today

const newExercise = ref('')
const streak = ref(Number(localStorage.getItem('streak')) || 0)
const xp = ref(Number(localStorage.getItem('xp')) || 0)
const isDark = ref(localStorage.getItem('theme') !== 'light')
const restDay = ref(localStorage.getItem('restDate') === today)

// The day the streak was last counted, so it can only go up once per day.
const lastStreakDate = ref(localStorage.getItem('lastStreakDate') || '')

// Exercise ids that have already paid out XP today. Unchecking and
// rechecking a box no longer farms 10 XP a click.
const awardedIds = ref(isNewDay ? [] : load('awardedIds', []))

const exercises = ref(
  savedExercises || [
    { id: 1, name: 'Squats', completed: false },
    { id: 2, name: 'Romanian Deadlifts', completed: false },
    { id: 3, name: 'Core Workout', completed: false }
  ]
)

if (isNewDay) {
  exercises.value = exercises.value.map(exercise => ({
    ...exercise,
    completed: false
  }))

  localStorage.setItem('lastWorkoutDate', today)
  localStorage.setItem('awardedIds', '[]')
}

function addExercise() {
  if (!newExercise.value.trim()) return

  exercises.value.push({
    id: Date.now(),
    name: newExercise.value.trim(),
    completed: false
  })

  newExercise.value = ''
}

function deleteExercise(id) {
  exercises.value = exercises.value.filter(exercise => exercise.id !== id)
}

function resetToday() {
  exercises.value = exercises.value.map(exercise => ({
    ...exercise,
    completed: false
  }))
}

function registerStreakDay() {
  if (lastStreakDate.value === today) return

  streak.value = lastStreakDate.value === yesterday() ? streak.value + 1 : 1
  lastStreakDate.value = today
  localStorage.setItem('lastStreakDate', today)
}

function toggleRestDay() {
  restDay.value = !restDay.value

  if (restDay.value) {
    localStorage.setItem('restDate', today)

    // The +5 is paid once per day, not once per click.
    if (localStorage.getItem('restXpDate') !== today) {
      xp.value += 5
      localStorage.setItem('restXpDate', today)
    }
  } else {
    localStorage.removeItem('restDate')
  }
}

watch(
  exercises,
  newValue => {
    localStorage.setItem('exercises', JSON.stringify(newValue))

    const freshlyDone = newValue.filter(
      exercise =>
        exercise.completed && !awardedIds.value.includes(exercise.id)
    )

    if (freshlyDone.length > 0) {
      xp.value += freshlyDone.length * 10
      awardedIds.value = [
        ...awardedIds.value,
        ...freshlyDone.map(exercise => exercise.id)
      ]
      localStorage.setItem('awardedIds', JSON.stringify(awardedIds.value))
    }

    if (newValue.length > 0 && newValue.every(exercise => exercise.completed)) {
      registerStreakDay()
    }
  },
  { deep: true }
)

watch(xp, value => {
  localStorage.setItem('xp', value)
})

watch(streak, value => {
  localStorage.setItem('streak', value)
})

// The theme class lives on <body> so the light background covers the
// whole page, not just the 700px column.
watch(
  isDark,
  value => {
    localStorage.setItem('theme', value ? 'dark' : 'light')
    document.body.classList.toggle('light', !value)
  },
  { immediate: true }
)

const completedCount = computed(() => {
  return exercises.value.filter(exercise => exercise.completed).length
})

const progressPercentage = computed(() => {
  if (exercises.value.length === 0) return 0

  return Math.round(
    (completedCount.value / exercises.value.length) * 100
  )
})

const athleteLevel = computed(() => {
  if (xp.value < 50) return 'Seed Athlete 🌱'
  if (xp.value < 150) return 'Building Strength 💪'
  if (xp.value < 300) return 'Strong Momentum 🔥'
  if (xp.value < 500) return 'Garden Warrior 🌳'

  return 'Strength Legend 🏆'
})

const garden = computed(() => {
  if (restDay.value) return '😴 🌙 🛌'
  if (completedCount.value === 0) return '🌱'

  const plants = ['🌱', '🪴', '🌷', '🌻', '🌳', '🍄', '🌸']

  return Array.from({ length: completedCount.value }, (_, index) => {
    return plants[index % plants.length]
  }).join(' ')
})

const achievements = computed(() => [
  {
    icon: '🌱',
    name: 'First Step',
    description: 'Complete your first exercise',
    unlocked: completedCount.value >= 1 || xp.value >= 10
  },
  {
    icon: '💪',
    name: 'Getting Stronger',
    description: 'Earn 50 XP',
    unlocked: xp.value >= 50
  },
  {
    icon: '🔥',
    name: 'On Fire',
    description: 'Reach a 3 day streak',
    unlocked: streak.value >= 3
  },
  {
    icon: '🌳',
    name: 'Garden Warrior',
    description: 'Earn 300 XP',
    unlocked: xp.value >= 300
  }
])
</script>

<style>
body {
  font-family: Arial, sans-serif;
  background: #0f172a;
  color: #f8fafc;
  margin: 0;
  min-height: 100vh;
}

.app {
  max-width: 700px;
  margin: auto;
  padding: 2rem;
  position: relative;
}

h1,
h2 {
  text-align: center;
  color: inherit;
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

button:focus-visible {
  outline: 2px solid #38bdf8;
  outline-offset: 2px;
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
  color: inherit;
  font-size: 1.1rem;
}

.exercise input[type='checkbox'] {
  width: 20px;
  height: 20px;
}

.done {
  text-decoration: line-through;
  color: #94a3b8;
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
  line-height: 1.4;
}

.level {
  text-align: center;
  font-size: 1.3rem;
  font-weight: bold;
}

.progress-bar {
  width: 100%;
  height: 20px;
  background: #334155;
  border-radius: 999px;
  overflow: hidden;
  margin: 15px 0;
}

.progress-fill {
  height: 100%;
  background: #22c55e;
  border-radius: 999px;
  transition: width 0.3s ease;
}

.badges {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
  gap: 12px;
  margin-top: 16px;
}

.badge {
  background: #0f172a;
  padding: 14px;
  border-radius: 12px;
  text-align: center;
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.badge span {
  font-size: 2rem;
}

.badge small {
  color: #cbd5e1;
}

.badge.locked {
  opacity: 0.35;
  filter: grayscale(1);
}

.rest-button {
  width: 100%;
  margin-top: 1.5rem;
  background: #8b5cf6;
}

.rest-button:hover {
  background: #7c3aed;
}

.rest-button.active {
  background: #6d28d9;
}

.reset-button {
  width: 100%;
  margin-top: 1.5rem;
  background: #ef4444;
}

.reset-button:hover {
  background: #dc2626;
}

p {
  text-align: center;
  font-size: 1.1rem;
}

/* Theme toggle */
.theme-toggle {
  position: fixed;
  top: 16px;
  right: 16px;
  background: transparent;
  font-size: 1.4rem;
  padding: 8px;
}

/* Light theme */
body.light {
  background: #f8fafc;
  color: #0f172a;
}

body.light .card {
  background: #ffffff;
  border: 1px solid #e2e8f0;
}

body.light .exercise {
  background: #f1f5f9;
}

body.light .badge {
  background: #f1f5f9;
}

body.light .badge small {
  color: #475569;
}

body.light .progress-bar {
  background: #e2e8f0;
}

body.light input {
  background: #f1f5f9;
  color: #0f172a;
  border: 1px solid #cbd5e1;
}

body.light input::placeholder {
  color: #64748b;
}

body.light .delete-button:hover {
  background: #e2e8f0;
}

@media (prefers-reduced-motion: reduce) {
  .progress-fill {
    transition: none;
  }
}
</style>