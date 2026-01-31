<script setup>
import { ref } from 'vue'

const emit = defineEmits(['login-success'])

const authView = ref('login')
const authError = ref('')

const loginForm = ref({ username: '', password: '' })
const registerForm = ref({ username: '', password: '', confirmPassword: '' })

const register = () => {
  authError.value = ''
  const username = registerForm.value.username.trim()

  if (!username) return authError.value = 'Username is required'
  if (registerForm.value.password.length < 4) return authError.value = 'Password must be at least 4 characters'
  if (registerForm.value.password !== registerForm.value.confirmPassword) return authError.value = 'Passwords do not match'

  let users = JSON.parse(localStorage.getItem('gym-daily-users') || '[]')
  if (users.some(u => u.username === username)) return authError.value = 'Username already taken'

  users.push({ username, password: registerForm.value.password })
  localStorage.setItem('gym-daily-users', JSON.stringify(users))

  localStorage.setItem('gym-daily-current-user', JSON.stringify({ username }))
  emit('login-success')

  authView.value = 'login'
  registerForm.value = { username: '', password: '', confirmPassword: '' }
}

const login = () => {
  authError.value = ''
  const users = JSON.parse(localStorage.getItem('gym-daily-users') || '[]')
  const user = users.find(u =>
    u.username === loginForm.value.username &&
    u.password === loginForm.value.password
  )

  if (!user) return authError.value = 'Incorrect username or password'

  localStorage.setItem('gym-daily-current-user', JSON.stringify({ username: user.username }))
  emit('login-success')
}
</script>

<template>
  <div class="glass-container auth-container">
    <div class="brand" style="margin: 0 auto 2.5rem; flex-direction: column; text-align: center;">
      <div class="logo" style="font-size: 3.5rem; width: 90px; height: 90px; margin-bottom: 1rem;">⚡</div>
      <h1 style="font-size: 2.4rem;">Gym Daily</h1>
    </div>

    <Transition name="fade" mode="out-in">
      <div v-if="authView === 'login'" key="login">
        <h2 style="text-align:center; margin-bottom:2rem;">Sign In</h2>
        <div v-if="authError" class="error-msg">{{ authError }}</div>
        <div class="input-wrapper" style="margin-bottom:1.2rem;">
          <input v-model="loginForm.username" placeholder="Username" autocomplete="username" />
        </div>
        <div class="input-wrapper" style="margin-bottom:1.8rem;">
          <input v-model="loginForm.password" type="password" placeholder="Password" autocomplete="current-password" />
        </div>
        <button class="add-btn" @click="login">Login</button>
        <p style="text-align:center; margin-top:1.6rem; color:var(--text-mute);">
          No account yet? <a href="#" @click.prevent="authView = 'register'">Register</a>
        </p>
      </div>

      <div v-else key="register">
        <h2 style="text-align:center; margin-bottom:2rem;">Create Account</h2>
        <div v-if="authError" class="error-msg">{{ authError }}</div>
        <div class="input-wrapper" style="margin-bottom:1.2rem;">
          <input v-model="registerForm.username" placeholder="Username" autocomplete="username" />
        </div>
        <div class="input-wrapper" style="margin-bottom:1.2rem;">
          <input v-model="registerForm.password" type="password" placeholder="Password" autocomplete="new-password" />
        </div>
        <div class="input-wrapper" style="margin-bottom:1.8rem;">
          <input v-model="registerForm.confirmPassword" type="password" placeholder="Confirm password" autocomplete="new-password" />
        </div>
        <button class="add-btn" @click="register">Create Account</button>
        <p style="text-align:center; margin-top:1.6rem; color:var(--text-mute);">
          Already have an account? <a href="#" @click.prevent="authView = 'login'">Sign in</a>
        </p>
      </div>
    </Transition>
  </div>
</template>

<style scoped>
.auth-container {
  justify-content: center;
  min-height: 70vh;
  padding: 40px 24px;
}
</style>