<script setup>
import { ref, computed, watch, onMounted, onUnmounted } from 'vue'
import AuthView from './views/AuthView.vue'

const currentUser = ref(null)
const viewMode = ref('tracker')
const exercises = ref([])
const filter = ref('All')
const editingId = ref(null)
const editForm = ref({})
const timerTime = ref(0)
const timerActive = ref(false)
const timerInterval = ref(null)

const form = ref({
  name: '',
  type: 'Strength',
  sets: 3,
  reps: 10,
  weight: 0,
  duration: 30,
  intensity: 'Medium'
})

const TYPES = ['Strength', 'Cardio', 'Flexibility']
const INTENSITIES = ['Low', 'Medium', 'High']

const lastHistoryMatch = computed(() => {
  if (!form.value.name) return null
  return exercises.value.find(
    e => e.name.toLowerCase() === form.value.name.toLowerCase() && e.completed
  ) || null
})

const applyHistory = () => {
  if (lastHistoryMatch.value) {
    form.value = { ...form.value, ...lastHistoryMatch.value, id: undefined, completed: false, date: undefined }
  }
}

const toggleTimer = () => {
  if (timerActive.value) {
    clearInterval(timerInterval.value)
    timerActive.value = false
  } else {
    timerActive.value = true
    timerInterval.value = setInterval(() => timerTime.value++, 1000)
  }
}

const resetTimer = () => {
  timerTime.value = 0
  if (timerActive.value) toggleTimer()
}

const formatTime = (seconds) => {
  const m = Math.floor(seconds / 60).toString().padStart(2, '0')
  const s = (seconds % 60).toString().padStart(2, '0')
  return `${m}:${s}`
}

const heatmapData = computed(() => {
  const days = []
  for (let i = 27; i >= 0; i--) {
    const d = new Date()
    d.setDate(d.getDate() - i)
    const isToday = i === 0
    const hasWorkout = exercises.value.length > 0 && isToday
    const intensity = isToday && hasWorkout ? 'high' : (Math.random() > 0.7 ? 'med' : 'none')
    days.push({ date: d.getDate(), intensity })
  }
  return days
})

const addExercise = () => {
  if (!form.value.name) return
  const newExercise = {
    id: crypto.randomUUID(),
    ...form.value,
    completed: false,
    date: new Date().toISOString(),
    calories: calculateCalories(form.value)
  }
  exercises.value.unshift(newExercise)
  resetForm()
}

const addTemplate = (templateName) => {
  const templates = {
    'Leg Day': [
      { name: 'Squats', type: 'Strength', sets: 4, reps: 8, weight: 60, intensity: 'High' },
      { name: 'Lunges', type: 'Strength', sets: 3, reps: 12, weight: 20, intensity: 'Medium' }
    ],
    'Cardio Blast': [
      { name: 'Treadmill Run', type: 'Cardio', duration: 20, intensity: 'High' },
      { name: 'Rowing', type: 'Cardio', duration: 15, intensity: 'Medium' }
    ],
    'Morning Yoga': [
      { name: 'Sun Salutation', type: 'Flexibility', duration: 10, intensity: 'Low' }
    ]
  }
  templates[templateName]?.forEach(item => {
    exercises.value.unshift({
      id: crypto.randomUUID(),
      ...item,
      completed: false,
      date: new Date().toISOString(),
      calories: calculateCalories(item)
    })
  })
}

const removeExercise = (id) => {
  exercises.value = exercises.value.filter(ex => ex.id !== id)
}

const toggleComplete = (ex) => {
  ex.completed = !ex.completed
}

const resetForm = () => {
  form.value.name = ''
}

const startEditing = (ex) => {
  if (ex.completed) return
  editingId.value = ex.id
  editForm.value = { ...ex }
}

const saveEdit = () => {
  const index = exercises.value.findIndex(ex => ex.id === editingId.value)
  if (index !== -1) {
    editForm.value.calories = calculateCalories(editForm.value)
    exercises.value[index] = { ...editForm.value }
  }
  editingId.value = null
}

const calculateCalories = (data) => {
  let multiplier = data.intensity === 'High' ? 2 : (data.intensity === 'Medium' ? 1.5 : 1)
  if (data.type === 'Cardio') return Math.round((data.duration || 0) * 8 * multiplier)
  if (data.type === 'Strength') return Math.round(((data.sets || 0) * (data.reps || 0) * 0.5) * multiplier + ((data.weight || 0) * 0.1))
  return Math.round((data.duration || 0) * 3 * multiplier)
}

const filteredExercises = computed(() =>
  filter.value === 'All' ? exercises.value : exercises.value.filter(ex => ex.type === filter.value)
)

const progress = computed(() =>
  !exercises.value.length ? 0 : Math.round((exercises.value.filter(ex => ex.completed).length / exercises.value.length) * 100)
)

const totalCalories = computed(() =>
  exercises.value.filter(ex => ex.completed).reduce((acc, curr) => acc + curr.calories, 0)
)

const statsByType = computed(() => {
  const counts = { Strength: 0, Cardio: 0, Flexibility: 0 }
  exercises.value.forEach(ex => { if (ex.completed) counts[ex.type]++ })
  return counts
})

const loadUserExercises = () => {
  if (!currentUser.value) return
  const key = `gym-daily-exercises-${currentUser.value.username}`
  const saved = localStorage.getItem(key)
  if (saved) exercises.value = JSON.parse(saved)
}

const logout = () => {
  if (currentUser.value) {
    const key = `gym-daily-exercises-${currentUser.value.username}`
    localStorage.setItem(key, JSON.stringify(exercises.value))
  }
  currentUser.value = null
  localStorage.removeItem('gym-daily-current-user')
  exercises.value = []
  viewMode.value = 'tracker'
}

onMounted(() => {
  const savedUser = localStorage.getItem('gym-daily-current-user')
  if (savedUser) {
    currentUser.value = JSON.parse(savedUser)
    loadUserExercises()
  }
})

watch(exercises, (newVal) => {
  if (currentUser.value) {
    const key = `gym-daily-exercises-${currentUser.value.username}`
    localStorage.setItem(key, JSON.stringify(newVal))
  }
}, { deep: true })

onUnmounted(() => clearInterval(timerInterval.value))
</script>

<template>
  <div class="app-wrapper">
    <div class="glow-orb orb-1"></div>
    <div class="glow-orb orb-2"></div>

    <AuthView v-if="!currentUser" @login-success="loadUserExercises" />

    <div v-else class="glass-container">
      <header>
        <div class="brand">
          <div class="logo">⚡</div>
          <div class="brand-text">
            <h1>Gym Daily</h1>
            <p class="date">{{ new Date().toLocaleDateString('en-US', { weekday: 'short', day: 'numeric' }) }}</p>
          </div>
        </div>
        <div style="display:flex; align-items:center; gap:12px;">
          <span style="font-size:0.9rem; color:var(--text-mute);">{{ currentUser.username }}</span>
          <button class="logout-btn" @click="logout">Logout</button>
        </div>
      </header>

      <nav class="nav-pills">
        <button :class="{ active: viewMode === 'tracker' }" @click="viewMode = 'tracker'" title="Tracker">📝</button>
        <button :class="{ active: viewMode === 'zone' }" @click="viewMode = 'zone'" title="The Zone">⏱️</button>
        <button :class="{ active: viewMode === 'dashboard' }" @click="viewMode = 'dashboard'" title="Stats">📊</button>
        <button :class="{ active: viewMode === 'journal' }" @click="viewMode = 'journal'" title="Journal">📅</button>
      </nav>

      <Transition name="fade" mode="out-in">
        <div v-if="viewMode === 'tracker'" class="view-section" key="tracker">
          <section class="add-card">
            <div class="input-row main">
              <div class="input-wrapper">
                <input v-model="form.name" placeholder="Exercise name..." />
                <div v-if="lastHistoryMatch" class="history-badge" @click="applyHistory">
                  ↺ Last: {{ lastHistoryMatch.weight }}kg
                </div>
              </div>
              <select v-model="form.type">
                <option v-for="t in TYPES" :key="t">{{ t }}</option>
              </select>
            </div>
            <div class="input-row details">
              <template v-if="form.type === 'Strength'">
                <div class="field"><label>Sets</label><input type="number" v-model="form.sets"></div>
                <div class="field"><label>Reps</label><input type="number" v-model="form.reps"></div>
                <div class="field"><label>Kg</label><input type="number" v-model="form.weight"></div>
              </template>
              <template v-else>
                <div class="field full"><label>Mins</label><input type="number" v-model="form.duration"></div>
              </template>
              <div class="field">
                <label>Int</label>
                <select v-model="form.intensity">
                  <option v-for="i in INTENSITIES" :key="i">{{ i }}</option>
                </select>
              </div>
            </div>
            <button class="add-btn" @click="addExercise">Add to Workout</button>
          </section>

          <div class="list-container">
            <TransitionGroup name="list">
              <div
                v-for="ex in filteredExercises"
                :key="ex.id"
                class="task-item"
                :class="{ completed: ex.completed, editing: editingId === ex.id }"
              >
                <div class="check-area" @click="toggleComplete(ex)">
                  <div class="checkbox">
                    <svg v-if="ex.completed" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="3">
                      <polyline points="20 6 9 17 4 12"></polyline>
                    </svg>
                  </div>
                </div>
                <div class="content">
                  <div v-if="editingId !== ex.id" class="view-content" @click="startEditing(ex)">
                    <div class="header">
                      <h4>{{ ex.name }}</h4>
                      <span class="calories">~{{ ex.calories }}</span>
                    </div>
                    <div class="sub-details">
                      <span class="badge" :class="ex.type.toLowerCase()">{{ ex.type }}</span>
                      <span class="info-text" v-if="ex.type === 'Strength'">
                        {{ ex.sets }}×{{ ex.reps }}
                        <span v-if="ex.weight > 0">@ {{ ex.weight }}kg</span>
                      </span>
                      <span class="info-text" v-else>{{ ex.duration }}m • {{ ex.intensity }}</span>
                    </div>
                  </div>
                  <div v-else class="edit-content">
                    <input v-model="editForm.name" class="edit-name">
                    <div class="edit-actions">
                      <button class="save-mini" @click="saveEdit">Save</button>
                      <button class="cancel-mini" @click="editingId = null">✕</button>
                    </div>
                  </div>
                </div>
                <button class="del-btn" @click="removeExercise(ex.id)" v-if="editingId !== ex.id">✕</button>
              </div>
            </TransitionGroup>

            <div v-if="filteredExercises.length === 0" class="empty-state">
              <p>Quick Start Templates</p>
              <div class="template-grid">
                <button class="tpl-card" @click="addTemplate('Leg Day')">🦵 Legs</button>
                <button class="tpl-card" @click="addTemplate('Cardio Blast')">🏃 Cardio</button>
              </div>
            </div>
          </div>
        </div>

        <div v-else-if="viewMode === 'zone'" class="view-section zone-view" key="zone">
          <div class="zone-header">
            <h3>Focus Mode</h3>
            <div class="live-indicator">● LIVE</div>
          </div>
          <div class="timer-orb-container">
            <div class="timer-orb" :class="{ breathing: timerActive }">
              <div class="timer-content">
                <span class="timer-label">{{ timerActive ? 'ACTIVE' : 'READY' }}</span>
                <span class="timer-digits">{{ formatTime(timerTime) }}</span>
              </div>
            </div>
          </div>
          <div class="timer-controls">
            <button class="control-btn main" @click="toggleTimer">
              {{ timerActive ? 'PAUSE' : 'START' }}
            </button>
            <button class="control-btn sub" @click="resetTimer">RESET</button>
          </div>
          <div class="up-next-card">
            <span class="label">UP NEXT</span>
            <div v-if="exercises.find(e => !e.completed)">
              <h4>{{ exercises.find(e => !e.completed).name }}</h4>
              <p>{{ exercises.find(e => !e.completed).sets }} Sets • {{ exercises.find(e => !e.completed).weight }}kg</p>
            </div>
            <div v-else class="all-done">All exercises completed! 🎉</div>
          </div>
        </div>

        <div v-else-if="viewMode === 'dashboard'" class="view-section" key="dashboard">
          <div class="stat-card big">
            <h3>Daily Progress</h3>
            <div class="circle-chart" :style="{ '--p': progress }"><span>{{ progress }}%</span></div>
          </div>
          <div class="stat-grid">
            <div class="stat-card">
              <span class="icon">🔥</span>
              <div class="info"><h3>{{ totalCalories }}</h3><p>Kcal</p></div>
            </div>
            <div class="stat-card">
              <span class="icon">✅</span>
              <div class="info"><h3>{{ exercises.filter(e => e.completed).length }}</h3><p>Done</p></div>
            </div>
          </div>
          <div class="type-breakdown">
            <h3>Focus Area</h3>
            <div class="bars">
              <div class="bar-row" v-for="(count, type) in statsByType" :key="type">
                <span class="label">{{ type }}</span>
                <div class="bar-track">
                  <div class="bar-fill" :class="type.toLowerCase()" :style="{ width: (count * 10) + '%' }"></div>
                </div>
                <span class="count">{{ count }}</span>
              </div>
            </div>
          </div>
        </div>

        <div v-else-if="viewMode === 'journal'" class="view-section journal-view" key="journal">
          <div class="journal-header">
            <h3>Consistency</h3>
            <p>Last 28 Days</p>
          </div>
          <div class="heatmap-grid">
            <div
              v-for="(day, i) in heatmapData"
              :key="i"
              class="heat-box"
              :class="day.intensity"
            >
              <div class="tooltip">{{ day.date }}th</div>
            </div>
          </div>
          <div class="trophy-case">
            <h3>Recent Records</h3>
            <div class="trophy-row">
              <div class="trophy-card">
                <span class="emoji">🏆</span>
                <div><h4>Bench Press</h4><p>85kg × 5</p></div>
              </div>
              <div class="trophy-card">
                <span class="emoji">⚡</span>
                <div><h4>5k Run</h4><p>24:30 min</p></div>
              </div>
            </div>
          </div>
          <div class="monthly-summary">
            <div class="summary-item"><span>Total Workouts</span><strong>12</strong></div>
            <div class="summary-item"><span>Best Streak</span><strong>4 Days</strong></div>
          </div>
        </div>
      </Transition>
    </div>
  </div>
</template>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;500;700;800&display=swap');

:root {
  --primary: #6366f1;
  --accent: #10b981;
  --cardio: #3b82f6;
  --flex: #f59e0b;
  --glass: rgba(255, 255, 255, 0.05);
  --glass-border: rgba(255, 255, 255, 0.1);
  --text: #f8fafc;
  --text-mute: #94a3b8;
  --bg-dark: #0f172a;
}

* { box-sizing: border-box; outline: none; }

.app-wrapper {
  font-family: 'Plus Jakarta Sans', sans-serif;
  min-height: 100vh;
  background: var(--bg-dark);
  color: var(--text);
  display: flex;
  justify-content: center;
  padding: 20px;
  position: relative;
  overflow-x: hidden;
}

.glow-orb {
  position: absolute;
  width: 300px;
  height: 300px;
  border-radius: 50%;
  filter: blur(80px);
  z-index: 0;
  opacity: 0.4;
  animation: float 10s infinite ease-in-out;
}

.orb-1 { top: -50px; left: -50px; background: var(--primary); }
.orb-2 { bottom: -50px; right: -50px; background: var(--accent); animation-delay: 5s; }

@keyframes float {
  0%, 100% { transform: translate(0, 0); }
  50% { transform: translate(20px, 40px); }
}

.glass-container {
  width: 100%;
  max-width: 420px;
  background: rgba(30, 41, 59, 0.7);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border: 1px solid var(--glass-border);
  border-radius: 24px;
  padding: 24px;
  z-index: 1;
  box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.5);
  display: flex;
  flex-direction: column;
  min-height: 80vh;
}

.auth-container {
  justify-content: center;
  min-height: 70vh;
  padding: 40px 24px;
}

header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 24px;
}

.brand { display: flex; align-items: center; gap: 10px; }

.logo {
  font-size: 20px;
  background: linear-gradient(135deg, var(--primary), var(--accent));
  width: 36px; height: 36px;
  display: flex; align-items: center; justify-content: center;
  border-radius: 10px;
  box-shadow: 0 4px 12px rgba(99,102,241,0.3);
}

h1 { font-size: 1rem; margin: 0; font-weight: 800; }

.date { font-size: 0.7rem; color: var(--text-mute); margin: 0; }

.nav-pills {
  background: rgba(0,0,0,0.3);
  padding: 4px;
  border-radius: 12px;
  display: flex;
  gap: 2px;
}

.nav-pills button {
  background: transparent;
  border: none;
  font-size: 1.2rem;
  padding: 8px 12px;
  border-radius: 8px;
  cursor: pointer;
  opacity: 0.6;
  transition: 0.2s;
}

.nav-pills button.active {
  background: var(--glass-border);
  opacity: 1;
  box-shadow: 0 2px 4px rgba(0,0,0,0.2);
}

.logout-btn {
  background: rgba(239, 68, 68, 0.2);
  color: #f87171;
  border: none;
  padding: 6px 14px;
  border-radius: 8px;
  font-size: 0.9rem;
  cursor: pointer;
}

.logout-btn:hover { background: rgba(239, 68, 68, 0.4); }

.error-msg {
  color: #f87171;
  background: rgba(248,113,113,0.15);
  padding: 10px;
  border-radius: 10px;
  margin-bottom: 1.4rem;
  text-align: center;
}

.add-card { background: var(--glass); border: 1px solid var(--glass-border); padding: 16px; border-radius: 20px; margin-bottom: 24px; }

.input-row { display: flex; gap: 10px; margin-bottom: 12px; }

.input-row.main .input-wrapper { flex: 2; position: relative; }

.input-row.main select { flex: 1; }

input, select {
  background: rgba(0,0,0,0.3);
  border: 1px solid transparent;
  padding: 12px;
  border-radius: 10px;
  color: #fff;
  width: 100%;
  font-family: inherit;
}

input:focus, select:focus { border-color: var(--primary); background: rgba(0,0,0,0.5); }

.add-btn {
  width: 100%;
  padding: 12px;
  background: var(--primary);
  color: white;
  border: none;
  border-radius: 10px;
  font-weight: 700;
  cursor: pointer;
  transition: 0.2s;
}

.history-badge {
  position: absolute;
  right: 0;
  top: -20px;
  font-size: 0.65rem;
  color: var(--accent);
  cursor: pointer;
  background: rgba(16, 185, 129, 0.1);
  padding: 2px 6px;
  border-radius: 4px;
}

.details { display: flex; gap: 8px; }

.field { display: flex; flex-direction: column; flex: 1; }

.field label { font-size: 0.65rem; color: var(--text-mute); font-weight: 700; }

.field.full { flex: 2; }

.list-container { display: flex; flex-direction: column; gap: 12px; }

.task-item {
  background: linear-gradient(to right, rgba(255,255,255,0.03), transparent);
  border-left: 3px solid var(--accent);
  padding: 12px;
  border-radius: 12px;
  display: flex;
  align-items: center;
  gap: 12px;
}

.check-area { cursor: pointer; }

.checkbox {
  width: 22px;
  height: 22px;
  border: 2px solid var(--text-mute);
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.checkbox svg { width: 14px; color: #0f172a; }

.task-item.completed .checkbox { background: var(--accent); border-color: var(--accent); }

.task-item.completed { opacity: 0.5; border-left-color: var(--text-mute); }

.content { flex: 1; }

.header { display: flex; justify-content: space-between; }

h4 { margin: 0; font-size: 0.95rem; }

.sub-details { display: flex; gap: 8px; margin-top: 4px; font-size: 0.75rem; color: var(--text-mute); }

.badge { padding: 2px 6px; border-radius: 4px; background: rgba(255,255,255,0.1); text-transform: uppercase; font-size: 0.65rem; }

.del-btn { background: none; border: none; color: var(--text-mute); cursor: pointer; }

.empty-state { text-align: center; color: var(--text-mute); font-size: 0.85rem; margin-top: 20px; }

.template-grid { display: flex; gap: 8px; justify-content: center; margin-top: 10px; }

.tpl-card {
  background: rgba(255,255,255,0.05);
  border: 1px solid var(--glass-border);
  color: var(--text);
  padding: 8px 12px;
  border-radius: 8px;
  cursor: pointer;
  font-size: 0.8rem;
}

.edit-content { display: flex; gap: 5px; }

.edit-name { padding: 4px 8px; font-size: 0.85rem; }

.save-mini, .cancel-mini { border: none; border-radius: 4px; cursor: pointer; padding: 0 8px; }

.save-mini { background: var(--accent); color: #000; }

.cancel-mini { background: rgba(255,255,255,0.1); color: #fff; }

.zone-view { display: flex; flex-direction: column; align-items: center; text-align: center; height: 100%; padding-top: 10px; }

.zone-header { display: flex; justify-content: space-between; width: 100%; align-items: center; margin-bottom: 30px; }

.live-indicator {
  background: #ef4444;
  color: white;
  padding: 2px 8px;
  border-radius: 12px;
  font-size: 0.7rem;
  font-weight: 800;
  animation: pulse 2s infinite;
}

.timer-orb-container { position: relative; width: 220px; height: 220px; display: flex; justify-content: center; align-items: center; margin-bottom: 30px; }

.timer-orb {
  width: 100%;
  height: 100%;
  border-radius: 50%;
  border: 4px solid var(--glass-border);
  display: flex;
  align-items: center;
  justify-content: center;
  background: radial-gradient(circle at 30% 30%, rgba(99,102,241,0.2), transparent);
  transition: all 0.5s ease;
  box-shadow: 0 0 20px rgba(99,102,241,0.1);
}

.timer-orb.breathing {
  border-color: var(--accent);
  box-shadow: 0 0 50px rgba(16,185,129,0.4);
  animation: breathe 4s infinite ease-in-out;
}

.timer-content { display: flex; flex-direction: column; }

.timer-label { font-size: 0.8rem; letter-spacing: 2px; color: var(--text-mute); margin-bottom: 4px; }

.timer-digits { font-size: 3.5rem; font-weight: 800; font-variant-numeric: tabular-nums; text-shadow: 0 4px 12px rgba(0,0,0,0.5); }

.timer-controls { display: flex; gap: 12px; margin-bottom: 30px; width: 100%; max-width: 250px; }

.control-btn {
  flex: 1;
  padding: 14px;
  border: none;
  border-radius: 12px;
  font-weight: 700;
  cursor: pointer;
  transition: 0.2s;
}

.control-btn.main { background: var(--text); color: var(--bg-dark); }

.control-btn.main:hover { background: var(--primary); color: white; }

.control-btn.sub { background: rgba(255,255,255,0.1); color: var(--text); }

.up-next-card {
  background: var(--glass);
  width: 100%;
  padding: 16px;
  border-radius: 16px;
  border-left: 3px solid var(--primary);
  text-align: left;
}

.up-next-card .label { font-size: 0.65rem; color: var(--text-mute); letter-spacing: 1px; }

.stat-card { background: var(--glass); border: 1px solid var(--glass-border); padding: 16px; border-radius: 16px; }

.stat-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; margin-bottom: 12px; }

.stat-card.big { text-align: center; margin-bottom: 12px; display: flex; flex-direction: column; align-items: center; }

.circle-chart {
  width: 100px;
  height: 100px;
  border-radius: 50%;
  background: conic-gradient(var(--accent) calc(var(--p) * 1%), rgba(255,255,255,0.1) 0);
  display: flex;
  align-items: center;
  justify-content: center;
  margin-top: 10px;
  position: relative;
}

.circle-chart::before {
  content: '';
  position: absolute;
  width: 80px;
  height: 80px;
  background: #1e293b;
  border-radius: 50%;
}

.circle-chart span { position: relative; font-weight: 800; font-size: 1.2rem; }

.journal-view { display: flex; flex-direction: column; gap: 20px; }

.journal-header { border-bottom: 1px solid var(--glass-border); padding-bottom: 10px; }

.heatmap-grid { display: grid; grid-template-columns: repeat(7, 1fr); gap: 6px; }

.heat-box {
  aspect-ratio: 1;
  border-radius: 4px;
  background: rgba(255,255,255,0.05);
  position: relative;
}

.heat-box.med { background: rgba(16,185,129, 0.4); box-shadow: 0 0 8px rgba(16,185,129,0.3); }

.heat-box.high { background: #10b981; box-shadow: 0 0 12px rgba(16,185,129,0.6); }

.trophy-case { background: var(--glass); padding: 16px; border-radius: 16px; border: 1px solid var(--glass-border); }

.trophy-row { display: flex; gap: 12px; margin-top: 12px; }

.trophy-card {
  flex: 1;
  background: rgba(0,0,0,0.2);
  padding: 10px;
  border-radius: 8px;
  display: flex;
  align-items: center;
  gap: 10px;
}

.monthly-summary {
  display: flex;
  justify-content: space-between;
  background: rgba(99,102,241,0.1);
  padding: 16px;
  border-radius: 16px;
}

.summary-item { display: flex; flex-direction: column; align-items: center; }

.summary-item span { font-size: 0.7rem; color: var(--text-mute); }

.summary-item strong { font-size: 1.2rem; }

@keyframes breathe { 0%, 100% { transform: scale(1); } 50% { transform: scale(1.05); border-color: var(--primary); } }

@keyframes pulse { 0%, 100% { opacity: 1; } 50% { opacity: 0.5; } }

.fade-enter-active, .fade-leave-active { transition: opacity 0.2s ease; }
.fade-enter-from, .fade-leave-to { opacity: 0; }

.list-enter-active, .list-leave-active { transition: all 0.4s ease; }
.list-enter-from, .list-leave-to { opacity: 0; transform: translateX(-30px); }
</style>