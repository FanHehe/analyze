<template>
  <div id="app">
    <router-view></router-view>
  </div>
</template>

<script>
export default {
  name: 'app'
};
</script>

<style>
@import "./assets/color-dark.css";

#app {
  color: #2c3e50;
  height: 100%;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  /*font-family: 'Avenir', Helvetica, Arial, sans-serif;*/
  background-color: #4f7f9b;
  background-repeat: repeat-x;
  background-image: linear-gradient(to bottom, #4f7f9b, #293c55);
}
</style>
