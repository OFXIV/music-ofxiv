<template>
  <div class="music-player">
    <div v-if="loading" class="loading">加载中...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else-if="songs.length > 0" class="stack-container">
      <!-- 上一首 -->
      <div
        v-if="prevSongData"
        class="song-cover prev"
        @click="goPrev"
      >
        <img :src="prevSongData.cover" :alt="`${prevSongData.name}封面`" />
      </div>

      <!-- 当前播放 -->
      <div class="song-cover current">
        <img :src="currentSong.cover" :alt="`${currentSong.name}封面`" />
        <div class="play-btn" @click.stop="togglePlay">
          <div :class="['icon', isPlaying ? 'pause' : 'play']"></div>
        </div>
      </div>

      <!-- 下一首 -->
      <div
        v-if="nextSongData"
        class="song-cover next"
        @click="goNext"
      >
        <img :src="nextSongData.cover" :alt="`${nextSongData.name}封面`" />
      </div>
    </div>

    <!-- 歌曲信息 -->
    <div v-if="songs.length > 0" class="song-info">
      <h2>{{ currentSong.name }} - {{ currentSong.artist }}</h2>
    </div>

    <div v-else>
      <p>暂无歌曲可播放</p>
    </div>

    <audio ref="audioRef" :src="currentSong.url" @ended="playNext" preload="metadata" style="display:none"></audio>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, nextTick } from 'vue'

const songs = ref([])
const loading = ref(true)
const error = ref(null)
const currentIndex = ref(0)
const audioRef = ref(null)
const isPlaying = ref(false)

const currentSong = computed(() => songs.value[currentIndex.value] || {})
const prevSongData = computed(() => songs.value[(currentIndex.value - 1 + songs.value.length) % songs.value.length] || null)
const nextSongData = computed(() => songs.value[(currentIndex.value + 1) % songs.value.length] || null)

const togglePlay = () => {
  if (!audioRef.value) return
  if (isPlaying.value) {
    audioRef.value.pause()
    isPlaying.value = false
  } else {
    audioRef.value.play()
    isPlaying.value = true
  }
}

const playNext = () => {
  currentIndex.value = (currentIndex.value + 1) % songs.value.length
  nextTick(() => {
    if (audioRef.value) {
      audioRef.value.play()
      isPlaying.value = true
    }
  })
}

const goPrev = () => {
  currentIndex.value = (currentIndex.value - 1 + songs.value.length) % songs.value.length
  nextTick(() => {
    if (audioRef.value) {
      audioRef.value.play()
      isPlaying.value = true
    }
  })
}

const goNext = () => {
  currentIndex.value = (currentIndex.value + 1) % songs.value.length
  nextTick(() => {
    if (audioRef.value) {
      audioRef.value.play()
      isPlaying.value = true
    }
  })
}

onMounted(async () => {
  try {
    const res = await fetch('https://cdn.jsdelivr.net/gh/ofxiv/resources/json/music.json')
    if (!res.ok) throw new Error('获取歌曲列表失败')
    const data = await res.json()
    songs.value = Array.isArray(data) ? data : []
  } catch (err) {
    console.error('加载歌曲失败:', err)
    error.value = '加载歌曲失败，请稍后重试'
  } finally {
    loading.value = false
  }
})
</script>

<style scoped>
.music-player {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: flex-start;
  height: 90vh;
  margin: 0 auto;
  text-align: center;
  position: relative;
  padding-top: 20px;
}

.stack-container {
  position: relative;
  width: 100%;
  height: 400px;
}

/* 当前播放封面 */
.song-cover.current {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 300px;
  height: 300px;
  transform: translate(-50%, -50%);
  border-radius: 12px;
  box-shadow: 0 12px 24px rgba(0, 0, 0, 0.3);
  z-index: 3;
}

/* 播放按钮 */
.play-btn {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 60px;
  height: 60px;
  border-radius: 50%;
  background: rgba(0, 0, 0, 0.25);
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
}

.icon.play {
  width: 0;
  height: 0;
  border-left: 16px solid white;
  border-top: 10px solid transparent;
  border-bottom: 10px solid transparent;
}

.icon.pause {
  width: 16px;
  height: 16px;
  position: relative;
}

.icon.pause::before,
.icon.pause::after {
  content: '';
  position: absolute;
  top: 0;
  width: 4px;
  height: 16px;
  background: white;
}

.icon.pause::before { left: 0; }
.icon.pause::after { right: 0; }

/* 上一首和下一首封面 */
.song-cover.prev,
.song-cover.next {
  width: 200px;
  height: 150px;
  border-radius: 12px;
  opacity: 0.5;
  filter: blur(1px);
  position: absolute;
  left: 50%;
  transform: translate(-50%, -50%) scale(0.8);
  z-index: 2;
  cursor: pointer;
}

.song-cover.prev { top: 10%; }
.song-cover.next { top: 90%; }

.song-cover img {
  width: 100%;
  height: 100%;
  border-radius: inherit;
  object-fit: cover;
}

.song-info {
  display: flex;
  font-weight: bold;
  color: #333;
}

.loading, .error {
  font-size: 1.2rem;
  color: #666;
}

.error { color: #ff4444; }
</style>
