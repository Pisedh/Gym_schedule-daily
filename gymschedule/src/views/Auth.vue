<script setup>
import { ref } from 'vue';
import { useRouter } from 'vue-router';

const router = useRouter();
const authMode = ref('login');
const authForm = ref({ username: '', password: '' });
const authError = ref('');

const playSound = (type) => {
  const audio = new (window.AudioContext || window.webkitAudioContext)();
  const osc = audio.createOscillator();
  const gain = audio.createGain();
  const now = audio.currentTime;
  osc.connect(gain);
  gain.connect(audio.destination);
  if (type === 'error') {
    osc.type = 'sawtooth';
    osc.frequency.setValueAtTime(150, now);
    gain.gain.setValueAtTime(0.1, now);
    gain.gain.linearRampToValueAtTime(0, now + 0.3);
  } else {
    osc.frequency.setValueAtTime(200, now);
    osc.frequency.linearRampToValueAtTime(600, now + 0.3);
    gain.gain.setValueAtTime(0.1, now);
    gain.gain.linearRampToValueAtTime(0, now + 0.5);
  }
  osc.start(now); osc.stop(now + 0.5);
};

const defaultUser = { name: '', level: 1, xp: 0, nextLevelXp: 500, streak: 0, totalWorkouts: 0, history: [], queue: [] };

const handleAuth = () => {
  authError.value = '';
  if (!authForm.value.username || !authForm.value.password) {
    authError.value = 'Credentials Required'; playSound('error'); return;
  }
  const db = JSON.parse(localStorage.getItem('cybergym_users') || '{}');
  const { username, password } = authForm.value;

  if (authMode.value === 'register') {
    if (db[username]) { authError.value = 'User already exists'; playSound('error'); return; }
    const newUser = { ...defaultUser, name: username, password }; 
    db[username] = newUser;
    localStorage.setItem('cybergym_users', JSON.stringify(db));
    loginUser(username);
  } else {
    const user = db[username];
    if (!user || user.password !== password) { authError.value = 'Invalid Coordinates'; playSound('error'); return; }
    loginUser(username);
  }
};

const loginUser = (username) => {
  playSound('login');
  localStorage.setItem('cybergym_session', username);
  router.push('/');
};
</script>

<template>
  <div class="auth-container">
    <div class="auth-card">
      <div class="auth-logo">⚡ CYBER GYM</div>
      <div class="auth-tabs">
        <button :class="{ active: authMode === 'login' }" @click="authMode = 'login'; authError = ''">Login</button>
        <button :class="{ active: authMode === 'register' }" @click="authMode = 'register'; authError = ''">Register</button>
      </div>
      <form @submit.prevent="handleAuth" class="auth-form">
        <div class="input-group"><input v-model="authForm.username" type="text" placeholder=" " required><label>Username</label></div>
        <div class="input-group"><input v-model="authForm.password" type="password" placeholder=" " required><label>Password</label></div>
        <p v-if="authError" class="auth-error">⚠ {{ authError }}</p>
        <button type="submit" class="auth-btn">{{ authMode === 'login' ? 'ACCESS SYSTEM' : 'CREATE ID' }}</button>
      </form>
    </div>
  </div>
</template>

<style scoped>
.auth-container { position: absolute; inset: 0; z-index: 50; display: flex; align-items: center; justify-content: center; backdrop-filter: blur(5px); }
.auth-card { background: rgba(17, 24, 39, 0.9); border: 1px solid var(--border); border-radius: 24px; padding: 40px; width: 100%; max-width: 400px; box-shadow: 0 0 50px rgba(139, 92, 246, 0.2); position: relative; overflow: hidden; }
.auth-card::before { content: ''; position: absolute; top: 0; left: 0; width: 100%; height: 2px; background: linear-gradient(90deg, var(--primary), var(--accent)); }
.auth-logo { font-size: 24px; font-weight: 900; text-align: center; margin-bottom: 30px; letter-spacing: 2px; text-shadow: 0 0 10px var(--primary); color: white; }
.auth-tabs { display: flex; background: rgba(0,0,0,0.3); padding: 5px; border-radius: 12px; margin-bottom: 20px; }
.auth-tabs button { flex: 1; background: transparent; border: none; color: var(--text-dim); padding: 10px; cursor: pointer; border-radius: 8px; font-weight: 600; transition: 0.3s; }
.auth-tabs button.active { background: var(--surface-highlight); color: white; }
.input-group { position: relative; margin-bottom: 20px; }
.input-group input { width: 100%; background: transparent; border: 1px solid var(--border); padding: 12px; border-radius: 10px; color: white; font-size: 1rem; transition: 0.3s; }
.input-group input:focus { border-color: var(--primary); box-shadow: 0 0 10px var(--primary-glow); }
.input-group label { position: absolute; left: 12px; top: 12px; color: var(--text-dim); font-size: 0.9rem; pointer-events: none; transition: 0.3s; background: rgba(17, 24, 39, 0.9); padding: 0 4px; }
.input-group input:focus ~ label, .input-group input:not(:placeholder-shown) ~ label { top: -10px; font-size: 0.75rem; color: var(--primary); }
.auth-btn { width: 100%; padding: 14px; border: none; border-radius: 12px; background: linear-gradient(135deg, var(--primary), #6366f1); color: white; font-weight: 700; cursor: pointer; margin-top: 10px; font-size: 1rem; letter-spacing: 1px; transition: 0.3s; }
.auth-btn:hover { box-shadow: 0 0 20px var(--primary-glow); transform: scale(1.02); }
.auth-error { color: var(--danger); text-align: center; font-size: 0.9rem; margin-top: 15px; }
</style>