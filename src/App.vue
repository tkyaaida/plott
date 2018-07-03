<template>
  <div id="app">
    <!--<img src="./assets/logo.png">-->
    <!--<HelloWorld msg="Welcome to Your Vue.js App"/>-->
    <PlotCanvas
      :size="[500, 500]" :xRange="[-1, 1]" :yRange="[-1, 1]"
      :strokeWidth="2"
    ></PlotCanvas>
  </div>
</template>

<script>
import HelloWorld from './components/HelloWorld.vue'
import PlotCanvas from './components/PlotCanvas'

// source of truth
// var store = {
//   points: [],
// };

export default {
  name: 'app',
  components: {
    HelloWorld, PlotCanvas
  },
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
