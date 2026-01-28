<template>
  <div class="login-container">
    <div class="login-form">
      <h2 class="title">Login</h2>
      <form @submit.prevent="login">
        <div class="form-group">
          <input type="text" id="username" v-model="username" placeholder="Username" required>
        </div>
        <div class="form-group">
          <input type="password" id="password" v-model="password" placeholder="Password" required>
        </div>
        <button type="submit">Login</button>
      </form>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      username: '',
      password: ''
    };
  },
  methods: {
    login() {
      // For now, we'll just emit a 'loggedIn' event
      // In a real app, you'd have authentication logic here
      if (this.username && this.password) {
        this.$emit('loggedIn');
      }
    }
  }
};
</script>

<style scoped>
.login-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  background: linear-gradient(to right, #89f7fe, #66a6ff);
}

.login-form {
  width: 340px;
  padding: 40px;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 10px;
  box-shadow: 0 4px 30px rgba(0, 0, 0, 0.1);
  backdrop-filter: blur(5px);
  -webkit-backdrop-filter: blur(5px);
  border: 1px solid rgba(255, 255, 255, 0.3);
  color: white;
}

.title {
  text-align: center;
  margin-bottom: 30px;
  font-size: 24px;
}

.form-group {
  margin-bottom: 20px;
}

input {
  width: 100%;
  padding: 12px 15px;
  box-sizing: border-box;
  background: rgba(255, 255, 255, 0.2);
  border: 1px solid rgba(255, 255, 255, 0.3);
  border-radius: 5px;
  color: white;
  transition: background 0.3s, border 0.3s;
}

input::placeholder {
  color: rgba(255, 255, 255, 0.7);
}

input:focus {
  background: rgba(255, 255, 255, 0.3);
  border: 1px solid rgba(255, 255, 255, 0.5);
  outline: none;
}

button {
  width: 100%;
  padding: 12px;
  background: linear-gradient(to right, #89f7fe, #66a6ff);
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background 0.3s;
  font-size: 16px;
}

button:hover {
  background: linear-gradient(to right, #66a6ff, #89f7fe);
}
</style>
