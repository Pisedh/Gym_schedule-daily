<script setup>
import { ref, computed, onMounted } from 'vue';
import { useRouter } from 'vue-router';

const router = useRouter();
const view = ref('dashboard');
const showLibrary = ref(false);
const showLevelUp = ref(false);
const currentUser = ref(null);
const activeWorkout = ref(null);
const timer = ref(0);
const timerInterval = ref(null);
const isResting = ref(false);
const restTime = ref(0);

const library = {
  'Chest': ['Bench Press', 'Incline Dumbbell Press', 'Cable Flys', 'Dips'],
  'Back': ['Pull Ups', 'Barbell Row', 'Lat Pulldown', 'Deadlift'],
  'Legs': ['Squat', 'Leg Press', 'Romanian Deadlift', 'Calf Raises'],
  'Shoulders': ['Overhead Press', 'Lateral Raises', 'Face Pulls'],
  'Arms': ['Barbell Curl', 'Tricep Pushdown', 'Hammer Curls', 'Skullcrushers']
};

const playSound = (type) => {
  const audio = new (window.AudioContext || window.webkitAudioContext)();
  const osc = audio.createOscillator();
  const gain = audio.createGain();
  const now = audio.currentTime;
  osc.connect(gain);
  gain.connect(audio.destination);
  if (type === 'click') {
    osc.frequency.setValueAtTime(800, now);
    gain.gain.setValueAtTime(0.05, now);
    gain.gain.exponentialRampToValueAtTime(0.001, now + 0.1);
    osc.start(now); osc.stop(now + 0.1);
  } else if (type === 'levelUp') {
    osc.type = 'triangle';
    osc.frequency.setValueAtTime(400, now);
    osc.frequency.linearRampToValueAtTime(800, now + 0.2);
    gain.gain.setValueAtTime(0.2, now);
    gain.gain.linearRampToValueAtTime(0, now + 1);
    osc.start(now); osc.stop(now + 1);
  }
};

onMounted(() => {
  const sessionUser = localStorage.getItem('cybergym_session');
  if (!sessionUser) return router.push('/login');
  const db = JSON.parse(localStorage.getItem('cybergym_users') || '{}');
  currentUser.value = db[sessionUser];
});

const saveUserData = () => {
  if (!currentUser.value) return;
  const db = JSON.parse(localStorage.getItem('cybergym_users') || '{}');
  db[currentUser.value.name] = { ...currentUser.value, password: db[currentUser.value.name].password };
  localStorage.setItem('cybergym_users', JSON.stringify(db));
};

const logout = () => {
  saveUserData();
  localStorage.removeItem('cybergym_session');
  router.push('/login');
};

const addXp = (amount) => {
  currentUser.value.xp += amount;
  if (currentUser.value.xp >= currentUser.value.nextLevelXp) {
    currentUser.value.level++;
    currentUser.value.xp -= currentUser.value.nextLevelXp;
    currentUser.value.nextLevelXp = Math.floor(currentUser.value.nextLevelXp * 1.2);
    showLevelUp.value = true;
    playSound('levelUp');
    setTimeout(() => showLevelUp.value = false, 4000);
  }
  saveUserData();
};

const addToQueue = (name) => {
  currentUser.value.queue.push({
    id: crypto.randomUUID(), name,
    sets: [{ weight: 0, reps: 0, type: 'Normal', completed: false }, { weight: 0, reps: 0, type: 'Normal', completed: false }, { weight: 0, reps: 0, type: 'Normal', completed: false }]
  });
  showLibrary.value = false;
  playSound('click');
  saveUserData();
};

const addSet = (exId) => {
  currentUser.value.queue.find(e => e.id === exId).sets.push({ weight: 0, reps: 0, type: 'Normal', completed: false });
};

const startWorkout = () => {
  if (!currentUser.value.queue.length) return;
  activeWorkout.value = { start: Date.now() };
  view.value = 'active';
  timerInterval.value = setInterval(() => {
    timer.value++;
    if (isResting.value && restTime.value > 0) restTime.value--;
    if (isResting.value && restTime.value === 0) isResting.value = false;
  }, 1000);
};

const toggleSet = (exIndex, setIndex) => {
  const set = currentUser.value.queue[exIndex].sets[setIndex];
  set.completed = !set.completed;
  if (set.completed) { addXp(25); isResting.value = true; restTime.value = 60; }
  playSound('click');
};

const finishWorkout = () => {
  clearInterval(timerInterval.value);
  const volume = currentUser.value.queue.reduce((acc, ex) => acc + ex.sets.filter(s => s.completed).reduce((sAcc, s) => sAcc + (s.weight * s.reps), 0), 0);
  currentUser.value.history.unshift({ id: crypto.randomUUID(), date: new Date().toISOString(), duration: timer.value, volume, exercises: [...currentUser.value.queue] });
  currentUser.value.totalWorkouts++;
  currentUser.value.streak++;
  addXp(200);
  currentUser.value.queue = []; activeWorkout.value = null; timer.value = 0; view.value = 'dashboard';
  saveUserData();
};

const formatTime = (s) => `${Math.floor(s/60).toString().padStart(2,'0')}:${(s%60).toString().padStart(2,'0')}`;
const xpProgress = computed(() => currentUser.value ? (currentUser.value.xp / currentUser.value.nextLevelXp) * 100 : 0);
const heatmapDays = computed(() => {
  if(!currentUser.value) return [];
  const days = []; const today = new Date();
  for (let i = 104; i >= 0; i--) {
    const d = new Date(); d.setDate(today.getDate() - i);
    const dateStr = d.toISOString().split('T')[0];
    const workout = currentUser.value.history.find(h => h.date.startsWith(dateStr));
    days.push({ date: dateStr, intensity: workout ? (workout.volume > 5000 ? 3 : 2) : 0 });
  }
  return days;
});
</script>

<template>
  <div v-if="currentUser" class="dashboard-wrapper">
    <Transition name="pop"><div v-if="showLevelUp" class="level-up-overlay"><h1>LEVEL UP!</h1><div class="level-badge">{{ currentUser.level }}</div></div></Transition>

    <nav class="cyber-dock">
      <div class="dock-logo">⚡</div>
      <button :class="{ active: view === 'dashboard' }" @click="view = 'dashboard'"><span class="icon">▣</span> <span class="lbl">Dash</span></button>
      <button :class="{ active: view === 'planner' }" @click="view = 'planner'"><span class="icon">✚</span> <span class="lbl">Build</span></button>
      <button :class="{ active: view === 'active', pulse: activeWorkout }" @click="view = 'active'"><span class="icon">▶</span> <span class="lbl">Train</span></button>
      <button :class="{ active: view === 'journey' }" @click="view = 'journey'"><span class="icon">📅</span> <span class="lbl">Logs</span></button>
    </nav>

    <main class="cyber-stage">
      <Transition name="fade" mode="out-in">
        <div v-if="view === 'dashboard'" class="bento-grid">
          <div class="tile profile-tile">
            <div class="tile-head"><div class="avatar">{{ currentUser.name.charAt(0).toUpperCase() }}</div><div><h2>{{ currentUser.name }}</h2><span class="rank">Cyber Athlete</span></div><button class="logout-btn" @click="logout">⏻</button></div>
            <div class="xp-bar-container"><div class="xp-info"><span>XP Progress</span><span>{{ currentUser.xp }} / {{ currentUser.nextLevelXp }}</span></div><div class="xp-track"><div class="xp-fill" :style="{ width: xpProgress + '%' }"></div></div></div>
          </div>
          <div class="tile actions-tile"><h3>Quick Actions</h3><div class="action-grid"><button @click="view = 'planner'; showLibrary = true">New Workout</button><button class="outline" @click="view = 'journey'">History</button></div></div>
          <div class="tile stat-tile"><span class="label">Total Sessions</span><div class="big-val">{{ currentUser.totalWorkouts }}</div></div>
          <div class="tile stat-tile alt"><span class="label">Consistency</span><div class="big-val">{{ currentUser.streak }}🔥</div></div>
          <div class="tile heat-tile"><h3>Consistency Map</h3><div class="heatmap-mini"><div v-for="(day, i) in heatmapDays" :key="i" class="dot" :class="`i-${day.intensity}`" :title="day.date"></div></div></div>
        </div>

        <div v-else-if="view === 'planner'" class="planner-view">
          <div class="header-row"><h2>Workout Builder</h2><button class="neon-btn" @click="showLibrary = true">+ Add Exercise</button></div>
          <div v-if="currentUser.queue.length === 0" class="empty-state"><p>Queue Empty. Initialize Routine.</p></div>
          <div class="queue-list"><div v-for="(ex, i) in currentUser.queue" :key="ex.id" class="ex-card"><div class="ex-header"><h4>{{ i+1 }}. {{ ex.name }}</h4><button class="del" @click="currentUser.queue.splice(i, 1); saveUserData()">×</button></div><div class="set-preview"><span class="tag">{{ ex.sets.length }} Sets Planned</span></div></div></div>
          <button v-if="currentUser.queue.length" class="launch-btn" @click="startWorkout">INITIATE TRAINING</button>
        </div>

        <div v-else-if="view === 'active'" class="active-view">
          <div class="timer-hud" :class="{ resting: isResting }"><span class="hud-label">{{ isResting ? 'RECOVER' : 'ACTIVE' }}</span><span class="hud-time">{{ formatTime(isResting ? restTime : timer) }}</span></div>
          <div class="training-scroll"><div v-for="(ex, i) in currentUser.queue" :key="ex.id" class="training-card"><h3>{{ ex.name }}</h3><div class="set-table"><div class="th"><span>Set</span><span>Kg</span><span>Reps</span><span>Type</span><span>✓</span></div><div v-for="(set, si) in ex.sets" :key="si" class="tr" :class="{ done: set.completed }"><span class="idx">{{ si + 1 }}</span><input type="number" v-model="set.weight" placeholder="0"><input type="number" v-model="set.reps" placeholder="0"><select v-model="set.type" class="type-select"><option>Normal</option><option>Warmup</option><option>Drop</option><option>Fail</option></select><button class="check-btn" @click="toggleSet(i, si)"></button></div><button class="add-set-link" @click="addSet(ex.id)">+ Add Set</button></div></div></div>
          <button class="finish-btn" @click="finishWorkout">COMPLETE SESSION</button>
        </div>

        <div v-else-if="view === 'journey'" class="journey-view">
          <h2>Training Logs</h2>
          <div v-if="currentUser.history.length === 0" class="empty-state">No Data Recorded.</div>
          <div class="log-feed"><div v-for="h in currentUser.history" :key="h.id" class="log-card"><div class="log-head"><span class="date">{{ new Date(h.date).toLocaleDateString() }}</span><span class="vol">Vol: {{ h.volume }}kg</span></div><div class="log-body"><span class="ex-chip" v-for="ex in h.exercises" :key="ex.id">{{ ex.name }}</span></div></div></div>
        </div>
      </Transition>
    </main>

    <Transition name="scale"><div v-if="showLibrary" class="modal-backdrop" @click.self="showLibrary = false"><div class="library-modal"><div class="modal-head"><h3>Exercise Database</h3><button @click="showLibrary = false">✕</button></div><div class="category-grid"><div v-for="(items, cat) in library" :key="cat" class="cat-col"><h4>{{ cat }}</h4><div class="btn-group"><button v-for="ex in items" :key="ex" @click="addToQueue(ex)">{{ ex }}</button></div></div></div></div></div></Transition>
  </div>
</template>

<style scoped>
.dashboard-wrapper { display: flex; height: 100vh; width: 100vw; }
/* Reused styles from previous steps */
.cyber-dock { width: 80px; height: 100vh; background: rgba(17, 24, 39, 0.6); backdrop-filter: blur(20px); border-right: 1px solid var(--border); display: flex; flex-direction: column; align-items: center; padding-top: 20px; z-index: 10; }
.dock-logo { font-size: 24px; margin-bottom: 40px; text-shadow: 0 0 10px var(--accent); }
.cyber-dock button { background: transparent; border: none; color: var(--text-dim); width: 60px; height: 60px; border-radius: 16px; margin-bottom: 10px; display: flex; flex-direction: column; align-items: center; justify-content: center; gap: 4px; cursor: pointer; transition: 0.3s; }
.cyber-dock button.active { background: var(--surface-highlight); color: var(--accent); box-shadow: 0 0 15px rgba(6, 182, 212, 0.2); }
.cyber-dock button.pulse { animation: pulse-nav 2s infinite; color: var(--success); }
.cyber-stage { flex: 1; padding: 30px; overflow-y: auto; position: relative; z-index: 1; }
.bento-grid { display: grid; grid-template-columns: repeat(4, 1fr); grid-template-rows: auto auto; gap: 20px; max-width: 1200px; margin: 0 auto; }
.tile { background: var(--surface); border: 1px solid var(--border); border-radius: 24px; padding: 20px; position: relative; overflow: hidden; transition: transform 0.2s; }
.tile:hover { border-color: rgba(255,255,255,0.2); }
.profile-tile { grid-column: span 2; display: flex; flex-direction: column; justify-content: space-between; }
.tile-head { display: flex; align-items: center; gap: 15px; }
.avatar { width: 50px; height: 50px; background: linear-gradient(135deg, var(--primary), var(--accent)); border-radius: 50%; display: grid; place-items: center; font-weight: 800; font-size: 1.2rem; }
.logout-btn { margin-left: auto; background: transparent; border: 1px solid var(--border); color: var(--danger); width: 32px; height: 32px; border-radius: 8px; cursor: pointer; display: flex; align-items: center; justify-content: center; transition: 0.2s; }
.logout-btn:hover { background: var(--danger); color: white; }
.rank { font-size: 0.8rem; color: var(--text-dim); letter-spacing: 1px; text-transform: uppercase; }
.xp-bar-container { margin-top: 20px; }
.xp-info { display: flex; justify-content: space-between; font-size: 0.8rem; margin-bottom: 5px; color: var(--text-dim); }
.xp-track { height: 8px; background: rgba(255,255,255,0.05); border-radius: 4px; overflow: hidden; }
.xp-fill { height: 100%; background: linear-gradient(90deg, var(--primary), var(--accent)); border-radius: 4px; transition: width 0.5s ease; }
.actions-tile { grid-column: span 2; display: flex; flex-direction: column; justify-content: center; }
.action-grid { display: flex; gap: 10px; margin-top: 10px; }
.action-grid button { flex: 1; padding: 12px; border-radius: 12px; border: none; font-weight: 700; cursor: pointer; background: var(--text); color: var(--bg); transition: 0.2s; }
.action-grid button.outline { background: transparent; border: 1px solid var(--border); color: var(--text); }
.stat-tile { display: flex; flex-direction: column; align-items: center; justify-content: center; aspect-ratio: 1; }
.stat-tile.alt .big-val { color: var(--accent); }
.label { font-size: 0.8rem; color: var(--text-dim); }
.big-val { font-size: 2.5rem; font-weight: 800; line-height: 1; margin-top: 5px; }
.heat-tile { grid-column: span 2; }
.heatmap-mini { display: grid; grid-template-columns: repeat(21, 1fr); gap: 4px; margin-top: 10px; }
.dot { aspect-ratio: 1; background: #1f2937; border-radius: 2px; }
.dot.i-1 { background: #064e3b; }
.dot.i-2 { background: #059669; }
.dot.i-3 { background: #10b981; box-shadow: 0 0 5px var(--success); }
.planner-view { max-width: 800px; margin: 0 auto; }
.header-row { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }
.neon-btn { background: var(--primary); color: white; border: none; padding: 10px 20px; border-radius: 50px; font-weight: 600; cursor: pointer; box-shadow: 0 0 15px var(--primary-glow); }
.empty-state { text-align: center; margin-top: 100px; opacity: 0.5; }
.queue-list, .log-feed { display: flex; flex-direction: column; gap: 10px; }
.ex-card, .log-card { background: var(--surface); border: 1px solid var(--border); padding: 15px; border-radius: 16px; }
.ex-header, .log-head { display: flex; justify-content: space-between; align-items: center; }
.del { background: transparent; border: 1px solid var(--border); color: #ef4444; width: 30px; height: 30px; border-radius: 8px; cursor: pointer; }
.launch-btn { position: fixed; bottom: 30px; right: 30px; background: var(--success); color: #000; padding: 15px 40px; border-radius: 50px; font-weight: 800; border: none; font-size: 1.1rem; cursor: pointer; box-shadow: 0 0 20px rgba(16,185,129,0.4); animation: float 3s infinite ease-in-out; }
.active-view { max-width: 600px; margin: 0 auto; padding-bottom: 100px; }
.timer-hud { position: sticky; top: 0; z-index: 5; background: rgba(3,7,18,0.8); backdrop-filter: blur(10px); border: 1px solid var(--border); border-radius: 20px; padding: 15px; text-align: center; margin-bottom: 20px; }
.timer-hud.resting { border-color: var(--accent); box-shadow: 0 0 20px rgba(6,182,212,0.2); }
.hud-time { display: block; font-size: 3rem; font-weight: 900; line-height: 1; font-variant-numeric: tabular-nums; }
.training-card { background: var(--surface); border-radius: 20px; padding: 20px; margin-bottom: 20px; border: 1px solid var(--border); }
.set-table { display: flex; flex-direction: column; gap: 8px; }
.th, .tr { display: grid; grid-template-columns: 30px 1fr 1fr 1fr 40px; gap: 10px; align-items: center; }
.tr { background: rgba(0,0,0,0.2); padding: 8px; border-radius: 10px; }
.tr.done { background: rgba(16,185,129,0.1); border: 1px solid var(--success); opacity: 0.6; }
.tr input, .tr select { background: transparent; border: none; border-bottom: 1px solid var(--border); color: white; text-align: center; padding: 5px; width: 100%; }
.check-btn { width: 30px; height: 30px; border: 2px solid var(--border); border-radius: 8px; background: transparent; cursor: pointer; }
.tr.done .check-btn { background: var(--success); border-color: var(--success); }
.finish-btn { width: 100%; background: var(--primary); color: white; padding: 15px; border-radius: 16px; font-weight: 700; border: none; margin-top: 20px; cursor: pointer; }
.modal-backdrop { position: fixed; inset: 0; background: rgba(0,0,0,0.8); backdrop-filter: blur(5px); z-index: 100; display: flex; align-items: center; justify-content: center; }
.library-modal { background: var(--surface); width: 90%; max-width: 600px; max-height: 80vh; overflow-y: auto; border-radius: 24px; border: 1px solid var(--border); padding: 30px; }
.modal-head { display: flex; justify-content: space-between; margin-bottom: 20px; }
.modal-head button { background: none; border: none; color: white; font-size: 1.5rem; cursor: pointer; }
.cat-col { margin-bottom: 20px; }
.btn-group { display: flex; flex-wrap: wrap; gap: 10px; }
.btn-group button { background: rgba(255,255,255,0.05); border: 1px solid var(--border); color: var(--text); padding: 8px 16px; border-radius: 8px; cursor: pointer; transition: 0.2s; }
.log-head { color: var(--primary); font-weight: 700; margin-bottom: 8px; }
.ex-chip { display: inline-block; font-size: 0.75rem; background: rgba(255,255,255,0.05); padding: 2px 8px; border-radius: 4px; margin-right: 5px; margin-bottom: 5px; }
.level-up-overlay { position: fixed; inset: 0; z-index: 200; background: rgba(0,0,0,0.9); display: flex; flex-direction: column; align-items: center; justify-content: center; gap: 20px; }
.level-up-overlay h1 { font-size: 4rem; background: linear-gradient(to right, var(--primary), var(--accent)); -webkit-background-clip: text; -webkit-text-fill-color: transparent; animation: zoom 0.5s cubic-bezier(0.175, 0.885, 0.32, 1.275); }
.level-badge { width: 150px; height: 150px; border-radius: 50%; border: 10px solid var(--text); display: grid; place-items: center; font-size: 4rem; font-weight: 900; background: var(--primary); box-shadow: 0 0 100px var(--primary); }
@media (max-width: 768px) { .cyber-shell { flex-direction: column-reverse; } .cyber-dock { width: 100%; height: 70px; flex-direction: row; justify-content: space-evenly; border-right: none; border-top: 1px solid var(--border); padding: 0; } .dock-logo { display: none; } .cyber-dock button { width: auto; height: 100%; flex: 1; border-radius: 0; margin: 0; } .bento-grid { grid-template-columns: 1fr 1fr; } }
</style>
