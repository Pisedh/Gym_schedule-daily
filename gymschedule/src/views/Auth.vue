<script setup>
import { ref, computed } from 'vue';
import { useRouter } from 'vue-router';

const router = useRouter();

// --- State ---
const authMode = ref('login');
const isLoading = ref(false);
const showPassword = ref(false);
const rememberMe = ref(false);
const authError = ref('');

const authForm = ref({ 
  username: '', 
  password: '',
  confirmPassword: ''
});

// --- Computed ---
const passwordStrength = computed(() => {
  const pass = authForm.value.password;
  if (!pass) return 0;
  let score = 0;
  if (pass.length > 5) score += 33;
  if (pass.length > 8 && /[A-Z]/.test(pass)) score += 33;
  if (/[0-9]/.test(pass) && /[^A-Za-z0-9]/.test(pass)) score += 34;
  return score;
});

const isFormValid = computed(() => {
  if (!authForm.value.username || !authForm.value.password) return false;
  if (authMode.value === 'register' && authForm.value.password !== authForm.value.confirmPassword) return false;
  return true;
});

// --- Audio Effects ---
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

// --- Handlers ---
const toggleMode = (mode) => {
  authMode.value = mode;
  authError.value = '';
  authForm.value.confirmPassword = '';
};

const handleAuth = async () => {
  authError.value = '';

  if (authMode.value === 'register' && authForm.value.password !== authForm.value.confirmPassword) {
    authError.value = 'Passwords do not match';
    playSound('error');
    return;
  }

  // Simulate Network Delay for UX
  isLoading.value = true;
  await new Promise(resolve => setTimeout(resolve, 800));

  const db = JSON.parse(localStorage.getItem('cybergym_users') || '{}');
  const { username, password } = authForm.value;

  if (authMode.value === 'register') {
    if (db[username]) { 
      authError.value = 'User ID already exists'; 
      playSound('error'); 
    } else {
      const defaultUser = { name: username, level: 1, xp: 0, streak: 0, history: [] };
      db[username] = { ...defaultUser, password }; // In production, never store raw passwords!
      localStorage.setItem('cybergym_users', JSON.stringify(db));
      loginUser(username);
    }
  } else {
    const user = db[username];
    if (!user || user.password !== password) { 
      authError.value = 'Invalid Coordinates (Incorrect Username or Password)'; 
      playSound('error'); 
    } else {
      loginUser(username);
    }
  }
  isLoading.value = false;
};

const loginUser = (username) => {
  playSound('login');
  localStorage.setItem('cybergym_session', username);
  if (rememberMe.value) {
    localStorage.setItem('cybergym_remember', username);
  }
  router.push('/');
};
</script>

<template>
  <div class="auth-container">
    <div class="auth-card">
      <div class="auth-header">
        <div class="auth-logo">⚡ CYBER GYM</div>
        <p class="auth-subtitle">Neural Interface Login</p>
      </div>

      <div class="auth-tabs">
        <button :class="{ active: authMode === 'login' }" @click="toggleMode('login')">Initialize</button>
        <button :class="{ active: authMode === 'register' }" @click="toggleMode('register')">Register</button>
      </div>

      <form @submit.prevent="handleAuth" class="auth-form">
        <div class="input-group">
          <input v-model="authForm.username" type="text" placeholder=" " required id="username">
          <label for="username">Username / ID</label>
        </div>

        <div class="input-group">
          <input 
            v-model="authForm.password" 
            :type="showPassword ? 'text' : 'password'" 
            placeholder=" " 
            required 
            id="password"
          >
          <label for="password">Password</label>
          <button type="button" class="icon-btn" @click="showPassword = !showPassword">
            {{ showPassword ? '👁' : '🔒' }}
          </button>
        </div>

        <div v-if="authMode === 'register' && authForm.password" class="strength-meter">
          <div class="strength-bar" :style="{ width: passwordStrength + '%', background: passwordStrength > 66 ? '#10b981' : passwordStrength > 33 ? '#f59e0b' : '#ef4444' }"></div>
        </div>

        <Transition name="fade">
          <div v-if="authMode === 'register'" class="input-group">
            <input v-model="authForm.confirmPassword" type="password" placeholder=" " required id="confirm">
            <label for="confirm">Confirm Password</label>
          </div>
        </Transition>

        <div v-if="authMode === 'login'" class="form-options">
          <label class="checkbox-container">
            <input type="checkbox" v-model="rememberMe">
            <span class="checkmark"></span>
            Remember Me
          </label>
          <a href="#" class="forgot-link">Forgot Code?</a>
        </div>

        <Transition name="shake">
          <p v-if="authError" class="auth-error">⚠ {{ authError }}</p>
        </Transition>

        <button type="submit" class="auth-btn" :disabled="isLoading || !isFormValid">
          <span v-if="isLoading" class="spinner"></span>
          <span v-else>{{ authMode === 'login' ? 'ACCESS SYSTEM' : 'CREATE PROFILE' }}</span>
        </button>
      </form>
    </div>
  </div>
</template>

<style scoped>
/* Base Theme Variables */
* {
  --primary: #8b5cf6;
  --accent: #06b6d4;
  --bg-dark: #0f172a;
  --surface: rgba(30, 41, 59, 0.7);
  --border: rgba(139, 92, 246, 0.3);
  --text-primary: #f8fafc;
  --text-dim: #94a3b8;
  --danger: #ef4444;
}

.auth-container {
  position: absolute;
  inset: 0;
  z-index: 50;
  display: flex;
  align-items: center;
  justify-content: center;
  background: radial-gradient(circle at center, var(--bg-dark), #000);
  backdrop-filter: blur(10px);
}

.auth-card {
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: 20px;
  padding: 40px;
  width: 100%;
  max-width: 420px;
  box-shadow: 0 0 40px rgba(139, 92, 246, 0.15), inset 0 0 20px rgba(255,255,255,0.02);
  backdrop-filter: blur(12px);
  position: relative;
  overflow: hidden;
}

.auth-card::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 3px;
  background: linear-gradient(90deg, var(--primary), var(--accent));
}

.auth-header {
  text-align: center;
  margin-bottom: 30px;
}

.auth-logo {
  font-size: 28px;
  font-weight: 900;
  color: var(--text-primary);
  letter-spacing: 2px;
  text-shadow: 0 0 15px var(--primary);
}

.auth-subtitle {
  color: var(--text-dim);
  font-size: 0.85rem;
  letter-spacing: 1px;
  margin-top: 5px;
}

.auth-tabs {
  display: flex;
  background: rgba(0,0,0,0.5);
  padding: 6px;
  border-radius: 12px;
  margin-bottom: 25px;
}

.auth-tabs button {
  flex: 1;
  background: transparent;
  border: none;
  color: var(--text-dim);
  padding: 10px;
  cursor: pointer;
  border-radius: 8px;
  font-weight: 600;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.auth-tabs button.active {
  background: var(--primary);
  color: white;
  box-shadow: 0 0 15px rgba(139, 92, 246, 0.5);
}

/* Form Inputs */
.input-group {
  position: relative;
  margin-bottom: 20px;
}

.input-group input {
  width: 100%;
  background: rgba(15, 23, 42, 0.6);
  border: 1px solid rgba(255,255,255,0.1);
  padding: 14px 12px;
  border-radius: 10px;
  color: var(--text-primary);
  font-size: 1rem;
  transition: 0.3s;
  outline: none;
}

.input-group input:focus {
  border-color: var(--primary);
  box-shadow: 0 0 15px rgba(139, 92, 246, 0.3);
}

.input-group label {
  position: absolute;
  left: 12px;
  top: 14px;
  color: var(--text-dim);
  font-size: 0.9rem;
  pointer-events: none;
  transition: 0.3s;
  background: transparent;
}

.input-group input:focus ~ label, 
.input-group input:not(:placeholder-shown) ~ label {
  top: -10px;
  left: 10px;
  font-size: 0.75rem;
  color: var(--primary);
  background: var(--bg-dark);
  padding: 0 6px;
  border-radius: 4px;
}

.icon-btn {
  position: absolute;
  right: 12px;
  top: 14px;
  background: transparent;
  border: none;
  cursor: pointer;
  font-size: 1.1rem;
  opacity: 0.7;
  transition: 0.2s;
}

.icon-btn:hover {
  opacity: 1;
}

/* Extras (Strength, Remember Me) */
.strength-meter {
  height: 4px;
  background: rgba(255,255,255,0.1);
  border-radius: 2px;
  margin: -10px 0 20px 0;
  overflow: hidden;
}

.strength-bar {
  height: 100%;
  transition: width 0.3s ease, background 0.3s ease;
}

.form-options {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 0.85rem;
  margin-bottom: 20px;
  color: var(--text-dim);
}

.forgot-link {
  color: var(--primary);
  text-decoration: none;
  transition: 0.2s;
}

.forgot-link:hover {
  text-shadow: 0 0 5px var(--primary);
}

/* Submit Button & Loaders */
.auth-btn {
  width: 100%;
  padding: 14px;
  border: none;
  border-radius: 12px;
  background: linear-gradient(135deg, var(--primary), var(--accent));
  color: white;
  font-weight: 700;
  cursor: pointer;
  font-size: 1rem;
  letter-spacing: 1.5px;
  transition: all 0.3s;
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 50px;
}

.auth-btn:hover:not(:disabled) {
  box-shadow: 0 0 25px rgba(139, 92, 246, 0.6);
  transform: translateY(-2px);
}

.auth-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
  background: #475569;
}

.spinner {
  width: 20px;
  height: 20px;
  border: 3px solid rgba(255,255,255,0.3);
  border-top-color: white;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

@keyframes spin { 100% { transform: rotate(360deg); } }

.auth-error {
  color: var(--danger);
  text-align: center;
  font-size: 0.9rem;
  margin-top: 10px;
  margin-bottom: 15px;
  background: rgba(239, 68, 68, 0.1);
  padding: 8px;
  border-radius: 6px;
}

/* Transitions */
.fade-enter-active, .fade-leave-active { transition: opacity 0.3s ease, transform 0.3s ease; }
.fade-enter-from, .fade-leave-to { opacity: 0; transform: translateY(-10px); }

.shake-enter-active { animation: shake 0.5s; }
@keyframes shake {
  0%, 100% { transform: translateX(0); }
  25% { transform: translateX(-5px); }
  75% { transform: translateX(5px); }
}
</style>