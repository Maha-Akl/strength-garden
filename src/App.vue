<template>
  <div class="app">
    <button
      class="theme-toggle"
      :aria-label="isDark ? 'Switch to light theme' : 'Switch to dark theme'"
      @click="toggleTheme"
    >
      {{ isDark ? '☀️' : '🌙' }}
    </button>

    <h1>🏋️‍♀️ Strength Garden</h1>

    <div class="card">
      <h2>Add Exercise</h2>

      <div class="add-row">
        <input
          v-model="newExercise"
          placeholder="e.g. Bulgarian Split Squats"
          aria-label="New exercise name"
          :maxlength="MAX_NAME_LENGTH"
          @keyup.enter="addExercise"
          @input="addError = ''"
        />

        <button @click="addExercise">Add</button>
      </div>

      <p v-if="addError" class="error" role="alert">{{ addError }}</p>
    </div>

    <div class="card">
      <h2>Athlete Progress</h2>

      <p class="level">🏆 {{ athleteLevel }}</p>

      <div
        class="progress-bar"
        role="progressbar"
        aria-label="Today's workout progress"
        aria-valuemin="0"
        aria-valuemax="100"
        :aria-valuenow="progressPercentage"
      >
        <div
          class="progress-fill"
          :style="{ width: progressPercentage + '%' }"
        ></div>
      </div>

      <p>{{ progressPercentage }}% completed today</p>
      <p>⭐ {{ xp }} XP</p>
      <p>
        🔥 {{ streak }} day streak
        <span class="muted">(best: {{ bestStreak }})</span>
      </p>
    </div>

    <div class="card">
      <h2>Last 7 Days</h2>

      <div class="history">
        <div
          v-for="day in lastSevenDays"
          :key="day.key"
          class="history-day"
          :title="day.title"
        >
          <span class="dot" :class="day.status"></span>
          <small :class="{ today: day.isToday }">{{ day.label }}</small>
        </div>
      </div>

      <p class="legend">
        <span class="dot workout"></span>Workout
        <span class="dot rest"></span>Rest
        <span class="dot"></span>Nothing
      </p>
    </div>

    <div class="card">
      <h2>Today's Workout</h2>

      <p v-if="restDay" class="muted">
        Rest day: exercises are paused. Unmark the rest day to train.
      </p>

      <div
        v-for="exercise in exercises"
        :key="exercise.id"
        class="exercise"
      >
        <div v-if="editingId === exercise.id" class="edit-row">
          <input
            v-model="editName"
            v-focus
            aria-label="Rename exercise"
            :maxlength="MAX_NAME_LENGTH"
            @keyup.enter="saveEdit(exercise)"
            @keyup.esc="cancelEdit"
          />
          <button @click="saveEdit(exercise)">Save</button>
          <button class="secondary" @click="cancelEdit">Cancel</button>
        </div>

        <template v-else>
          <label>
            <input
              type="checkbox"
              v-model="exercise.completed"
              :disabled="restDay"
            />
            <span :class="{ done: exercise.completed }">
              {{ exercise.name }}
            </span>
          </label>

          <div class="exercise-actions">
            <button
              class="icon-button"
              :aria-label="`Rename ${exercise.name}`"
              @click="startEdit(exercise)"
            >
              ✏️
            </button>
            <button
              class="icon-button"
              :aria-label="`Delete ${exercise.name}`"
              @click="deleteExercise(exercise.id)"
            >
              🗑️
            </button>
          </div>
        </template>
      </div>

      <p v-if="editError" class="error" role="alert">{{ editError }}</p>

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

    <div class="card">
      <h2>Backup</h2>

      <div class="add-row">
        <button class="secondary wide" @click="exportData">⬇️ Export</button>
        <button class="secondary wide" @click="importInput.click()">
          ⬆️ Import
        </button>
        <input
          ref="importInput"
          type="file"
          accept="application/json"
          hidden
          @change="importData"
        />
      </div>

      <p v-if="backupMessage" class="muted">{{ backupMessage }}</p>
    </div>

    <button
      class="rest-button"
      :class="{ active: restDay }"
      :disabled="!restDay && completedCount > 0"
      @click="toggleRestDay"
    >
      {{ restDay ? '✅ Rest Day (+5 XP)' : '😴 Mark Rest Day' }}
    </button>

    <p v-if="!restDay && completedCount > 0" class="muted small">
      You've already trained today, so no rest day needed.
    </p>

    <button class="reset-button" @click="resetToday">
      Reset Today's Workout
    </button>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'

const MAX_NAME_LENGTH = 40

// Every key the app writes. Used by export/import.
const STORAGE_KEYS = [
  'exercises',
  'lastWorkoutDate',
  'awardedIds',
  'dayPlan',
  'xp',
  'streak',
  'bestStreak',
  'bestDay',
  'lastStreakDate',
  'restDate',
  'restXpDate',
  'restUndo',
  'history',
  'theme'
]

const now = new Date()
const today = now.toDateString()
const startOfToday = new Date(
  now.getFullYear(),
  now.getMonth(),
  now.getDate()
).getTime()

function daysAgo(n) {
  const date = new Date()
  date.setDate(date.getDate() - n)
  return date
}

function yesterday() {
  return daysAgo(1).toDateString()
}

function load(key, fallback) {
  try {
    const raw = localStorage.getItem(key)
    return raw === null ? fallback : JSON.parse(raw)
  } catch {
    return fallback
  }
}

function save(key, value) {
  localStorage.setItem(key, JSON.stringify(value))
}

// Lets the rename input focus itself when it appears.
const vFocus = { mounted: el => el.focus() }

// ---------- State ----------

const isNewDay = localStorage.getItem('lastWorkoutDate') !== today

const newExercise = ref('')
const addError = ref('')
const editingId = ref(null)
const editName = ref('')
const editError = ref('')
const backupMessage = ref('')
const importInput = ref(null)

const xp = ref(Number(localStorage.getItem('xp')) || 0)
const streak = ref(Number(localStorage.getItem('streak')) || 0)
const bestStreak = ref(
  Math.max(Number(localStorage.getItem('bestStreak')) || 0, streak.value)
)
const bestDay = ref(Number(localStorage.getItem('bestDay')) || 0)
const lastStreakDate = ref(localStorage.getItem('lastStreakDate') || '')
const restDay = ref(localStorage.getItem('restDate') === today)
const everRested = ref(localStorage.getItem('restXpDate') !== null)
const history = ref(load('history', {}))

// Saved choice wins; otherwise follow the system setting.
const savedTheme = localStorage.getItem('theme')
const isDark = ref(
  savedTheme
    ? savedTheme !== 'light'
    : !window.matchMedia('(prefers-color-scheme: light)').matches
)

// A missed day breaks the streak right away, not only when you next
// finish a full workout.
if (lastStreakDate.value !== today && lastStreakDate.value !== yesterday()) {
  streak.value = 0
  localStorage.setItem('streak', '0')
}

const exercises = ref(
  load('exercises', null) || [
    { id: 1, name: 'Squats', completed: false },
    { id: 2, name: 'Romanian Deadlifts', completed: false },
    { id: 3, name: 'Core Workout', completed: false }
  ]
)

// Exercise ids that have already paid out XP today. Unchecking and
// rechecking a box doesn't farm XP.
const awardedIds = ref(isNewDay ? [] : load('awardedIds', []))

// The exercises you started the day with (plus any added today).
// Deleting an unfinished one doesn't make the rest count as "all done".
const dayPlan = ref(isNewDay ? null : load('dayPlan', null))

if (isNewDay) {
  exercises.value = exercises.value.map(exercise => ({
    ...exercise,
    completed: false
  }))

  localStorage.setItem('lastWorkoutDate', today)
  localStorage.removeItem('restUndo')
  save('exercises', exercises.value)
}

if (!dayPlan.value) {
  dayPlan.value = exercises.value.map(exercise => exercise.id)
}

save('awardedIds', awardedIds.value)
save('dayPlan', dayPlan.value)

// ---------- Exercises ----------

function nameTaken(name, exceptId = null) {
  const lower = name.toLowerCase()

  return exercises.value.some(
    exercise => exercise.id !== exceptId && exercise.name.toLowerCase() === lower
  )
}

function addExercise() {
  const name = newExercise.value.trim().slice(0, MAX_NAME_LENGTH)
  if (!name) return

  if (nameTaken(name)) {
    addError.value = `"${name}" is already on your list.`
    return
  }

  const id = Date.now()

  exercises.value.push({ id, name, completed: false })
  dayPlan.value = [...dayPlan.value, id]

  newExercise.value = ''
  addError.value = ''
}

function deleteExercise(id) {
  exercises.value = exercises.value.filter(exercise => exercise.id !== id)

  // Something added today (e.g. a typo) was never part of the day's plan,
  // so deleting it shouldn't block the streak.
  if (id >= startOfToday) {
    dayPlan.value = dayPlan.value.filter(planId => planId !== id)
  }
}

function startEdit(exercise) {
  editingId.value = exercise.id
  editName.value = exercise.name
  editError.value = ''
}

function cancelEdit() {
  editingId.value = null
  editError.value = ''
}

function saveEdit(exercise) {
  const name = editName.value.trim().slice(0, MAX_NAME_LENGTH)

  if (!name) {
    editError.value = "The name can't be empty."
    return
  }

  if (nameTaken(name, exercise.id)) {
    editError.value = `"${name}" is already on your list.`
    return
  }

  exercise.name = name
  cancelEdit()
}

function resetToday() {
  if (!window.confirm("Uncheck all of today's exercises? Your XP stays.")) {
    return
  }

  exercises.value = exercises.value.map(exercise => ({
    ...exercise,
    completed: false
  }))
}

// ---------- Streak, rest days, history ----------

function registerStreakDay() {
  if (lastStreakDate.value === today) return

  streak.value = lastStreakDate.value === yesterday() ? streak.value + 1 : 1
  lastStreakDate.value = today
  localStorage.setItem('lastStreakDate', today)
}

function setHistory(status) {
  const next = { ...history.value }

  if (status) {
    next[today] = status
  } else {
    delete next[today]
  }

  // Keep only the last 60 days so storage doesn't grow forever.
  const keep = Object.keys(next)
    .sort((a, b) => Date.parse(a) - Date.parse(b))
    .slice(-60)

  history.value = Object.fromEntries(keep.map(key => [key, next[key]]))
}

function checkWorkoutComplete() {
  if (restDay.value) return

  const allChecked =
    exercises.value.length > 0 &&
    exercises.value.every(exercise => exercise.completed)

  const planDone = dayPlan.value.every(id => awardedIds.value.includes(id))

  if (allChecked && planDone) {
    registerStreakDay()
    setHistory('workout')
  }
}

function toggleRestDay() {
  // A rest day only makes sense if you haven't trained today.
  if (!restDay.value && completedCount.value > 0) return

  restDay.value = !restDay.value

  if (restDay.value) {
    localStorage.setItem('restDate', today)
    everRested.value = true

    // The +5 is paid once per day, not once per click.
    if (localStorage.getItem('restXpDate') !== today) {
      xp.value += 5
      localStorage.setItem('restXpDate', today)
    }

    // A rest day keeps the streak alive. Remember the old values so
    // unmarking the rest day can undo it.
    if (lastStreakDate.value !== today) {
      save('restUndo', {
        streak: streak.value,
        bestStreak: bestStreak.value,
        lastStreakDate: lastStreakDate.value
      })
      registerStreakDay()
    }

    if (history.value[today] !== 'workout') setHistory('rest')
  } else {
    localStorage.removeItem('restDate')

    const undo = load('restUndo', null)

    if (undo) {
      streak.value = undo.streak
      bestStreak.value = undo.bestStreak
      lastStreakDate.value = undo.lastStreakDate
      localStorage.setItem('lastStreakDate', undo.lastStreakDate)
      localStorage.removeItem('restUndo')
    }

    if (history.value[today] === 'rest') setHistory(null)
  }
}

// ---------- Theme ----------

function toggleTheme() {
  isDark.value = !isDark.value
  localStorage.setItem('theme', isDark.value ? 'dark' : 'light')
}

// ---------- Backup ----------

function exportData() {
  const data = {}

  for (const key of STORAGE_KEYS) {
    const value = localStorage.getItem(key)
    if (value !== null) data[key] = value
  }

  const backup = {
    app: 'strength-garden',
    version: 1,
    exportedAt: new Date().toISOString(),
    data
  }

  const blob = new Blob([JSON.stringify(backup, null, 2)], {
    type: 'application/json'
  })
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')

  link.href = url
  link.download = `strength-garden-${new Date().toISOString().slice(0, 10)}.json`
  link.click()

  setTimeout(() => URL.revokeObjectURL(url), 1000)
  backupMessage.value = 'Backup downloaded.'
}

async function importData(event) {
  const file = event.target.files[0]
  event.target.value = ''
  if (!file) return

  try {
    const backup = JSON.parse(await file.text())

    if (
      backup.app !== 'strength-garden' ||
      typeof backup.data !== 'object' ||
      backup.data === null
    ) {
      throw new Error('Not a backup')
    }

    if (!window.confirm('Replace all current data with this backup?')) return

    for (const key of STORAGE_KEYS) localStorage.removeItem(key)

    for (const [key, value] of Object.entries(backup.data)) {
      if (STORAGE_KEYS.includes(key) && typeof value === 'string') {
        localStorage.setItem(key, value)
      }
    }

    window.location.reload()
  } catch {
    backupMessage.value = "That file doesn't look like a Strength Garden backup."
  }
}

// ---------- Computed ----------

const completedCount = computed(() => {
  return exercises.value.filter(exercise => exercise.completed).length
})

const progressPercentage = computed(() => {
  if (exercises.value.length === 0) return 0

  return Math.round((completedCount.value / exercises.value.length) * 100)
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

const lastSevenDays = computed(() => {
  return Array.from({ length: 7 }, (_, index) => {
    const date = daysAgo(6 - index)
    const key = date.toDateString()
    const status = history.value[key] || 'none'

    return {
      key,
      status,
      isToday: key === today,
      label: date.toLocaleDateString(undefined, { weekday: 'short' }).slice(0, 2),
      title: `${date.toLocaleDateString()}: ${
        status === 'none' ? 'no activity' : status
      }`
    }
  })
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
    icon: '😴',
    name: 'Recharged',
    description: 'Take your first rest day',
    unlocked: everRested.value
  },
  {
    icon: '🔥',
    name: 'On Fire',
    description: 'Reach a 3 day streak',
    unlocked: bestStreak.value >= 3
  },
  {
    icon: '🗓️',
    name: 'Full Week',
    description: 'Reach a 7 day streak',
    unlocked: bestStreak.value >= 7
  },
  {
    icon: '⚡',
    name: 'Power Day',
    description: 'Complete 10 exercises in one day',
    unlocked: bestDay.value >= 10
  },
  {
    icon: '🌳',
    name: 'Garden Warrior',
    description: 'Earn 300 XP',
    unlocked: xp.value >= 300
  }
])

// ---------- Persistence ----------

watch(
  exercises,
  newValue => {
    save('exercises', newValue)

    const freshlyDone = newValue.filter(
      exercise => exercise.completed && !awardedIds.value.includes(exercise.id)
    )

    if (freshlyDone.length > 0) {
      xp.value += freshlyDone.length * 10
      awardedIds.value = [
        ...awardedIds.value,
        ...freshlyDone.map(exercise => exercise.id)
      ]
    }

    checkWorkoutComplete()
  },
  { deep: true }
)

watch(awardedIds, value => save('awardedIds', value))
watch(dayPlan, value => save('dayPlan', value))
watch(history, value => save('history', value))

watch(xp, value => {
  localStorage.setItem('xp', value)
})

watch(streak, value => {
  localStorage.setItem('streak', value)
  if (value > bestStreak.value) bestStreak.value = value
})

watch(bestStreak, value => {
  localStorage.setItem('bestStreak', value)
})

watch(completedCount, value => {
  if (value > bestDay.value) bestDay.value = value
})

watch(bestDay, value => {
  localStorage.setItem('bestDay', value)
})

// The theme class lives on <body> so the light background covers the
// whole page, not just the 700px column.
watch(
  isDark,
  value => {
    document.body.classList.toggle('light', !value)
  },
  { immediate: true }
)
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

button:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

button.secondary {
  background: #334155;
}

button.secondary:hover {
  background: #475569;
}

button.wide {
  flex: 1;
}

.error {
  color: #f87171;
  margin-bottom: 0;
}

.muted {
  color: #94a3b8;
}

.small {
  font-size: 0.9rem;
  margin: 8px 0 0;
}

/* Exercises */
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
  overflow-wrap: anywhere;
}

.exercise input[type='checkbox'] {
  width: 20px;
  height: 20px;
  flex-shrink: 0;
}

.exercise-actions {
  display: flex;
  gap: 4px;
  flex-shrink: 0;
}

.icon-button {
  background: transparent;
  padding: 6px;
  font-size: 1.1rem;
}

.icon-button:hover {
  background: #334155;
}

.edit-row {
  display: flex;
  gap: 8px;
  width: 100%;
}

.edit-row input {
  flex: 1;
  min-width: 0;
}

.done {
  text-decoration: line-through;
  color: #94a3b8;
}

/* History */
.history {
  display: flex;
  justify-content: space-between;
  gap: 8px;
  margin-top: 12px;
}

.history-day {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 6px;
  flex: 1;
}

.history-day small {
  color: #94a3b8;
}

.history-day small.today {
  color: inherit;
  font-weight: bold;
}

.dot {
  display: inline-block;
  width: 22px;
  height: 22px;
  border-radius: 50%;
  background: #334155;
}

.dot.workout {
  background: #22c55e;
}

.dot.rest {
  background: #8b5cf6;
}

.legend {
  font-size: 0.9rem;
  color: #94a3b8;
}

.legend .dot {
  width: 12px;
  height: 12px;
  vertical-align: middle;
  margin: 0 6px 0 14px;
}

/* Garden & progress */
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

/* Achievements */
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

/* Bottom buttons */
.rest-button {
  width: 100%;
  margin-top: 1.5rem;
  background: #8b5cf6;
}

.rest-button:hover:not(:disabled) {
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

body.light .exercise,
body.light .badge {
  background: #f1f5f9;
}

body.light .badge small,
body.light .muted,
body.light .legend,
body.light .history-day small {
  color: #475569;
}

body.light .history-day small.today {
  color: #0f172a;
}

body.light .progress-bar,
body.light .dot:not(.workout):not(.rest) {
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

body.light .icon-button:hover {
  background: #e2e8f0;
}

body.light button.secondary {
  background: #e2e8f0;
  color: #0f172a;
}

body.light button.secondary:hover {
  background: #cbd5e1;
}

body.light .error {
  color: #dc2626;
}

body.light .theme-toggle {
  background: transparent;
}

@media (prefers-reduced-motion: reduce) {
  .progress-fill {
    transition: none;
  }
}
</style>