<script setup lang="ts">
// This starter template is using Vue 3 <script setup> SFCs
// Check out https://vuejs.org/api/sfc-script-setup.html#script-setup
import 'boxicons/css/boxicons.min.css';
import Viz from './components/Viz.vue'
</script>

<template>
  <div :style="{
    margin: '0 2rem',
  }">
    <Viz
        audio-src="https://file.nya.one/misskey/627f040f-fc69-4daa-b094-b694b1494ca4.mp3"
        cover="https://file.nya.one/misskey/1dfe05b6-32d5-42ff-aa39-7e33aefb84ec.jpg"
    />
  </div>
</template>

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
