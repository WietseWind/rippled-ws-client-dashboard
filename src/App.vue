<template>
  <div id="app" class="container-fluid">
    <Dashboard/>
  </div>
</template>

<script>
import Dashboard from './components/Dashboard'

export default {
  name: 'App',
  components: {
    Dashboard
  }
}
</script>

<style lang="scss">
  #app {
    font-family: 'Avenir', Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    color: #2c3e50;
    margin-top: 10px;
  }
</style>
