<template>
  <div id="app">
    <InputTag/>
  </div>
</template>

<script>
import InputTag from './components/InputTag.vue'

export default {
  name: 'app',
  components: {
    InputTag
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
