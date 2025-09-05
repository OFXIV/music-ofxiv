<template>
  <div>
    <div v-for="s in songs" :key="s.filename" style="margin-bottom: 20px;">
      <h2>{{ s.name }} - {{ s.artist }}</h2>
      <audio :src="s.url" controls></audio>
      <img :src="s.cover" alt="cover" style="width:200px; margin-top:5px;">
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'

const songs = ref([])

onMounted(async () => {
  try {
    const res = await fetch('https://cdn.jsdelivr.net/gh/ofxiv/resources/json/music.json')
    songs.value = await res.json()
  } catch (err) {
    console.error('加载歌曲失败:', err)
  }
})
</script>