<template>
  <div id="app">
    <Login v-if="!loggedIn" @loggedIn="handleLogin" />
    <GymSchedule v-else />
  </div>
</template>

<script>
import GymSchedule from './components/GymSchedule.vue'
import Login from './components/Login.vue'

export default {
  name: 'App',
  components: {
    GymSchedule,
    Login
  },
  data() {
    return {
      loggedIn: false
    }
  },
  methods: {
    handleLogin() {
      this.loggedIn = true;
    }
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>