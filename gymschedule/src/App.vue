<script setup>
import { ref, computed, watch, onMounted } from 'vue';

// --- State ---
const viewMode = ref('tracker'); // 'tracker' or 'dashboard'
const exercises = ref([]);
const filter = ref('All'); 

const form = ref({
  name: '',
  type: 'Strength',
  sets: 3,
  reps: 10,
  weight: 0,
  duration: 30, // mins
  intensity: 'Medium' // Low, Medium, High
});

// --- Constants ---
const TYPES = ['Strength', 'Cardio', 'Flexibility'];
const INTENSITIES = ['Low', 'Medium', 'High'];

// --- Actions ---
const addExercise = () => {
  if (!form.value.name) return;

  const newExercise = {
    id: crypto.randomUUID(),
    ...form.value,
    completed: false,
    date: new Date().toISOString(),
    calories: calculateCalories(form.value)
  };

  exercises.value.unshift(newExercise);
  resetForm();
};

const removeExercise = (id) => {
  exercises.value = exercises.value.filter(ex => ex.id !== id);
};

const toggleComplete = (exercise) => {
  exercise.completed = !exercise.completed;
};

const resetDay = () => {
  if(confirm('Clear all exercises for today?')) {
    exercises.value = [];
  }
};

const resetForm = () => {
  form.value.name = '';
  // Keep previous settings for ease of entry
};

// --- Helpers ---
// specific calorie estimation logic
const calculateCalories = (data) => {
  let multiplier = 1;
  if (data.intensity === 'Medium') multiplier = 1.5;
  if (data.intensity === 'High') multiplier = 2;

  if (data.type === 'Cardio') {
    return Math.round(data.duration * 8 * multiplier); // Approx formula
  } else if (data.type === 'Strength') {
    return Math.round((data.sets * data.reps * 0.5) * multiplier + (data.weight * 0.1));
  } else {
    return Math.round(data.duration * 3 * multiplier);
  }
};

// --- Computed ---
const filteredExercises = computed(() => {
  if (filter.value === 'All') return exercises.value;
  return exercises.value.filter(ex => ex.type === filter.value);
});

const progress = computed(() => {
  if (!exercises.value.length) return 0;
  const done = exercises.value.filter(ex => ex.completed).length;
  return Math.round((done / exercises.value.length) * 100);
});

const totalCalories = computed(() => {
  return exercises.value
    .filter(ex => ex.completed)
    .reduce((acc, curr) => acc + curr.calories, 0);
});

const statsByType = computed(() => {
  const counts = { Strength: 0, Cardio: 0, Flexibility: 0 };
  exercises.value.forEach(ex => {
    if (ex.completed) counts[ex.type]++;
  });
  return counts;
});

// --- Persistence ---
onMounted(() => {
  const saved = localStorage.getItem('gym-daily-modern');
  if (saved) exercises.value = JSON.parse(saved);
});

watch(exercises, (newVal) => {
  localStorage.setItem('gym-daily-modern', JSON.stringify(newVal));
}, { deep: true });
</script>

<template>
  <div class="app-wrapper">
    <div class="glow-orb orb-1"></div>
    <div class="glow-orb orb-2"></div>

    <div class="glass-container">
      
      <header>
        <div class="brand">
          <div class="logo">⚡</div>
          <div>
            <h1>Gym Daily</h1>
            <p class="date">{{ new Date().toLocaleDateString('en-US', { weekday: 'long', day: 'numeric', month: 'short'}) }}</p>
          </div>
        </div>

        <div class="view-switch">
          <button :class="{ active: viewMode === 'tracker' }" @click="viewMode = 'tracker'">Tracker</button>
          <button :class="{ active: viewMode === 'dashboard' }" @click="viewMode = 'dashboard'">Stats</button>
        </div>
      </header>

      <Transition name="fade" mode="out-in">
        <div v-if="viewMode === 'dashboard'" class="dashboard-view" key="dashboard">
          <div class="stat-card big">
            <h3>Progress</h3>
            <div class="circle-chart" :style="{ '--p': progress }">
              <span>{{ progress }}%</span>
            </div>
          </div>
          
          <div class="stat-grid">
            <div class="stat-card">
              <span class="icon">🔥</span>
              <div class="info">
                <h3>{{ totalCalories }}</h3>
                <p>Kcal Burned</p>
              </div>
            </div>
            <div class="stat-card">
              <span class="icon">🏋️</span>
              <div class="info">
                <h3>{{ exercises.filter(e => e.completed).length }}</h3>
                <p>Completed</p>
              </div>
            </div>
          </div>

          <div class="type-breakdown">
            <h3>Completed by Type</h3>
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

        <div v-else class="tracker-view" key="tracker">
          
          <section class="add-card">
            <div class="input-row main">
              <input v-model="form.name" placeholder="What are we doing today?" autofocus />
              <select v-model="form.type">
                <option v-for="t in TYPES" :key="t">{{ t }}</option>
              </select>
            </div>

            <div class="input-row details">
              <template v-if="form.type === 'Strength'">
                <div class="field">
                  <label>Sets</label>
                  <input type="number" v-model="form.sets">
                </div>
                <div class="field">
                  <label>Reps</label>
                  <input type="number" v-model="form.reps">
                </div>
                <div class="field">
                  <label>Kg/Lbs</label>
                  <input type="number" v-model="form.weight">
                </div>
              </template>

              <template v-else>
                <div class="field full">
                  <label>Duration (Min)</label>
                  <input type="number" v-model="form.duration">
                </div>
              </template>
              
              <div class="field">
                <label>Intensity</label>
                <select v-model="form.intensity">
                  <option v-for="i in INTENSITIES" :key="i">{{ i }}</option>
                </select>
              </div>
            </div>

            <button class="add-btn" @click="addExercise">Add Exercise</button>
          </section>

          <div class="filter-bar">
            <div class="tabs">
              <span 
                v-for="f in ['All', ...TYPES]" 
                :key="f"
                :class="{ active: filter === f }"
                @click="filter = f"
              >
                {{ f }}
              </span>
            </div>
            <button class="clear-btn" @click="resetDay">Reset Day</button>
          </div>

          <div class="list-container">
            <TransitionGroup name="list">
              <div 
                v-for="ex in filteredExercises" 
                :key="ex.id" 
                class="task-item"
                :class="{ completed: ex.completed }"
              >
                <div class="check-area" @click="toggleComplete(ex)">
                  <div class="checkbox">
                    <svg v-if="ex.completed" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="3">
                      <polyline points="20 6 9 17 4 12"></polyline>
                    </svg>
                  </div>
                </div>

                <div class="content">
                  <div class="header">
                    <h4>{{ ex.name }}</h4>
                    <span class="calories">~{{ ex.calories }} kcal</span>
                  </div>
                  <div class="sub-details">
                    <span class="badge" :class="ex.type.toLowerCase()">{{ ex.type }}</span>
                    <span class="info-text" v-if="ex.type === 'Strength'">
                      {{ ex.sets }} x {{ ex.reps }} <span v-if="ex.weight > 0">@ {{ ex.weight }}kg</span>
                    </span>
                    <span class="info-text" v-else>
                      {{ ex.duration }} mins • {{ ex.intensity }}
                    </span>
                  </div>
                </div>

                <button class="del-btn" @click="removeExercise(ex.id)">
                  <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                    <path d="M18 6L6 18M6 6l12 12"></path>
                  </svg>
                </button>
              </div>
            </TransitionGroup>
            
            <div v-if="filteredExercises.length === 0" class="empty-state">
              No exercises yet. Start adding!
            </div>
          </div>
        </div>
      </Transition>

    </div>
  </div>
</template>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;500;700;800&display=swap');

/* --- Global Reset & Variables --- */
:root {
  --primary: #6366f1; /* Indigo */
  --accent: #10b981;  /* Emerald */
  --cardio: #3b82f6;
  --flex: #f59e0b;
  --glass: rgba(255, 255, 255, 0.05);
  --glass-border: rgba(255, 255, 255, 0.1);
  --text: #f8fafc;
  --text-mute: #94a3b8;
}

* { box-sizing: border-box; outline: none; }

.app-wrapper {
  font-family: 'Plus Jakarta Sans', sans-serif;
  min-height: 100vh;
  background: #0f172a;
  color: var(--text);
  display: flex;
  justify-content: center;
  padding: 20px;
  position: relative;
  overflow: hidden;
}

/* --- Glow Backgrounds --- */
.glow-orb {
  position: absolute;
  width: 300px;
  height: 300px;
  border-radius: 50%;
  filter: blur(80px);
  z-index: 0;
  opacity: 0.4;
}
.orb-1 { top: -50px; left: -50px; background: var(--primary); }
.orb-2 { bottom: -50px; right: -50px; background: var(--accent); }

/* --- Glass Container --- */
.glass-container {
  width: 100%;
  max-width: 500px;
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
}

/* --- Header --- */
header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 24px;
}

.brand { display: flex; align-items: center; gap: 12px; }
.logo {
  font-size: 24px;
  background: linear-gradient(135deg, var(--primary), var(--accent));
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 12px;
  box-shadow: 0 4px 12px rgba(99, 102, 241, 0.3);
}
h1 { font-size: 1.25rem; margin: 0; font-weight: 800; letter-spacing: -0.5px; }
.date { font-size: 0.8rem; color: var(--text-mute); margin: 0; }

.view-switch {
  background: rgba(0,0,0,0.3);
  padding: 4px;
  border-radius: 12px;
  display: flex;
  gap: 4px;
}
.view-switch button {
  background: transparent;
  border: none;
  color: var(--text-mute);
  padding: 6px 12px;
  border-radius: 8px;
  font-size: 0.8rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}
.view-switch button.active {
  background: var(--glass-border);
  color: #fff;
  box-shadow: 0 2px 4px rgba(0,0,0,0.2);
}

/* --- Dashboard Stats --- */
.stat-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
  margin-bottom: 12px;
}

.stat-card {
  background: var(--glass);
  border: 1px solid var(--glass-border);
  padding: 16px;
  border-radius: 16px;
  display: flex;
  align-items: center;
  gap: 12px;
}

.stat-card.big {
  flex-direction: column;
  justify-content: center;
  margin-bottom: 12px;
  text-align: center;
}

.stat-card h3 { margin: 0; font-size: 1.5rem; }
.stat-card p { margin: 0; color: var(--text-mute); font-size: 0.8rem; }
.stat-card .icon { font-size: 1.5rem; }

/* Circle Chart CSS Trick */
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

.type-breakdown .bar-row {
  display: flex;
  align-items: center;
  margin-top: 10px;
  gap: 10px;
  font-size: 0.85rem;
}
.bar-track { flex: 1; height: 6px; background: rgba(255,255,255,0.1); border-radius: 10px; overflow: hidden; }
.bar-fill { height: 100%; transition: width 0.5s; }
.bar-fill.strength { background: var(--accent); }
.bar-fill.cardio { background: var(--cardio); }
.bar-fill.flexibility { background: var(--flex); }

/* --- Add Exercise Card --- */
.add-card {
  background: var(--glass);
  border: 1px solid var(--glass-border);
  padding: 16px;
  border-radius: 20px;
  margin-bottom: 24px;
}

.input-row { display: flex; gap: 10px; margin-bottom: 12px; }
.input-row.main input { flex: 2; }
.input-row.main select { flex: 1; }

input, select {
  background: rgba(0,0,0,0.3);
  border: 1px solid transparent;
  padding: 12px;
  border-radius: 10px;
  color: #fff;
  font-size: 0.9rem;
  width: 100%;
}
input:focus, select:focus { border-color: var(--primary); background: rgba(0,0,0,0.5); }

.details { display: flex; gap: 8px; }
.field { display: flex; flex-direction: column; gap: 4px; flex: 1; }
.field.full { flex: 2; }
.field label { font-size: 0.7rem; color: var(--text-mute); font-weight: 600; text-transform: uppercase; }

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
.add-btn:hover { background: #4f46e5; }

/* --- Filters --- */
.filter-bar { display: flex; justify-content: space-between; align-items: center; margin-bottom: 16px; }
.tabs { display: flex; gap: 12px; }
.tabs span {
  font-size: 0.85rem;
  color: var(--text-mute);
  cursor: pointer;
  position: relative;
  padding-bottom: 4px;
}
.tabs span.active { color: #fff; font-weight: 600; }
.tabs span.active::after {
  content: ''; position: absolute; bottom: 0; left: 0; width: 100%; height: 2px; background: var(--accent); border-radius: 2px;
}
.clear-btn { background: none; border: none; color: #ef4444; font-size: 0.75rem; cursor: pointer; }

/* --- List Items --- */
.list-container { display: flex; flex-direction: column; gap: 12px; padding-bottom: 20px; }

.task-item {
  background: linear-gradient(to right, rgba(255,255,255,0.03), transparent);
  border-left: 3px solid var(--accent);
  padding: 12px;
  border-radius: 12px;
  display: flex;
  align-items: center;
  gap: 12px;
  transition: all 0.3s;
}
.task-item:hover { background: rgba(255,255,255,0.06); transform: translateX(2px); }

.checkbox {
  width: 22px; height: 22px;
  border: 2px solid var(--text-mute);
  border-radius: 6px;
  display: flex; align-items: center; justify-content: center;
  cursor: pointer;
  transition: 0.2s;
}
.checkbox svg { width: 14px; color: #0f172a; }

.task-item.completed .checkbox { background: var(--accent); border-color: var(--accent); }
.task-item.completed { opacity: 0.5; border-left-color: var(--text-mute); }
.task-item.completed h4 { text-decoration: line-through; }

.content { flex: 1; }
.header { display: flex; justify-content: space-between; align-items: center; }
h4 { margin: 0; font-size: 1rem; font-weight: 600; }
.calories { font-size: 0.7rem; color: #fbbf24; font-weight: 600; }

.sub-details { display: flex; align-items: center; gap: 8px; margin-top: 4px; }
.badge { font-size: 0.65rem; padding: 2px 6px; border-radius: 4px; text-transform: uppercase; background: rgba(255,255,255,0.1); }
.badge.strength { color: var(--accent); background: rgba(16, 185, 129, 0.15); }
.badge.cardio { color: var(--cardio); background: rgba(59, 130, 246, 0.15); }
.badge.flexibility { color: var(--flex); background: rgba(245, 158, 11, 0.15); }
.info-text { font-size: 0.8rem; color: var(--text-mute); }

.del-btn { background: none; border: none; color: var(--text-mute); cursor: pointer; padding: 4px; }
.del-btn:hover { color: #ef4444; }

.empty-state { text-align: center; color: var(--text-mute); margin-top: 20px; font-size: 0.9rem; font-style: italic; }

/* Transitions */
.list-enter-active, .list-leave-active, .fade-enter-active, .fade-leave-active { transition: all 0.3s ease; }
.list-enter-from, .list-leave-to, .fade-enter-from, .fade-leave-to { opacity: 0; transform: translateY(10px); }
</style>