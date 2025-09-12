<template>
  <!-- 音乐播放器容器 -->
  <div class="music-player">
    <div v-if="loading" class="loading">加载中...</div>
    <div v-else-if="error" class="error">{{ error }}</div>

    <div v-else-if="songs.length" class="stack-container" @touchstart="onTouchStart" @touchmove="onTouchMove"
      @touchend="onTouchEnd">
      <!-- 上一首 -->
      <div v-if="prevSong" class="song-cover prev" @click="changeSong(-1)">
        <img :src="prevSong.cover" :alt="`${prevSong.name}封面`" />
      </div>

      <!-- 当前播放 -->
      <div class="song-cover current">
        <img :src="currentSong.cover" :alt="`${currentSong.name}封面`" />
        <div class="play-btn" @click.stop="togglePlay">
          <div :class="['icon', isPlaying ? 'pause' : 'play']"></div>
        </div>
      </div>

      <!-- 下一首 -->
      <div v-if="nextSong" class="song-cover next"  @click="changeSong(1)">
        <img :src="nextSong.cover" :alt="`${nextSong.name}封面`" />
      </div>
    </div>

    <!-- 歌曲信息 -->
    <div v-if="currentSong.name" class="song-info">
      <h2>{{ currentSong.name }} - {{ currentSong.artist }}</h2>
    </div>

    <!-- 歌词 -->
    <div v-if="lyrics.length" class="lyrics" ref="lyricsRef">
      <transition-group name="lyric-fade" tag="div">
        <div v-for="line in lyrics" :key="line.time" class="lyric-line" :class="{ active: line === currentLyric }"
          :ref="el => lineRefs[line.time] = el">
          {{ line.text }}
        </div>
      </transition-group>
    </div>

    <audio ref="audioRef" :src="currentSong.url" @ended="changeSong(1)" @play="isPlaying = true"
      @pause="isPlaying = false" @timeupdate="onTimeUpdate" preload="metadata"
      style="position:absolute; width:0; height:0; opacity:0;"></audio>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted, nextTick } from 'vue'

const songs = ref([])
const loading = ref(true)
const error = ref(null)
const currentIndex = ref(0)
const audioRef = ref(null)
const isPlaying = ref(false)

const lyrics = ref([])
const currentLyricIndex = ref(0)
const lyricsRef = ref(null)
const lineRefs = ref({})

let scrollAnimationFrame = null

const currentSong = computed(() => songs.value[currentIndex.value] || {})
const prevSong = computed(() => songs.value.length > 1 ? songs.value[(currentIndex.value - 1 + songs.value.length) % songs.value.length] : null)
const nextSong = computed(() => songs.value.length > 1 ? songs.value[(currentIndex.value + 1) % songs.value.length] : null)
const currentLyric = computed(() => lyrics.value[currentLyricIndex.value] || {})

// 手势相关
let touchStartX = 0
let touchStartY = 0
let touchDeltaX = 0
let isSwiping = false
const SWIPE_THRESHOLD = 60 // 触发切歌的最小距离（px）

const onTouchStart = (e) => {
  const touch = e.touches[0]
  touchStartX = touch.clientX
  touchStartY = touch.clientY
  touchDeltaX = 0
  isSwiping = true
}

const onTouchMove = (e) => {
  if (!isSwiping) return
  const touch = e.touches[0]
  const deltaX = touch.clientX - touchStartX
  const deltaY = touch.clientY - touchStartY
  touchDeltaX = deltaX

  if (Math.abs(deltaX) > Math.abs(deltaY)) {
    e.preventDefault()
  }
}

const onTouchEnd = () => {
  if (!isSwiping) return
  if (touchDeltaX > SWIPE_THRESHOLD) {
    changeSong(-1)
  } else if (touchDeltaX < -SWIPE_THRESHOLD) {
    changeSong(1)
  }
  isSwiping = false
  touchDeltaX = 0
}

/* 播放/暂停 */
const togglePlay = async () => {
  if (!audioRef.value) return
  try {
    if (isPlaying.value) {
      audioRef.value.pause()
      isPlaying.value = false
    } else {
      await audioRef.value.play()
      isPlaying.value = true
    }
  } catch (err) {
    console.error('播放失败', err)
    isPlaying.value = false
  }
}

/* 切歌（优化版） */
const changeSong = async (step) => {
  if (!songs.value.length) return
  const nextIndex = (currentIndex.value + step + songs.value.length) % songs.value.length
  currentIndex.value = nextIndex

  await nextTick()
  if (audioRef.value && currentSong.value?.url) {
    audioRef.value.src = currentSong.value.url
    try {
      await audioRef.value.play()
      isPlaying.value = true
    } catch {
      isPlaying.value = false
    }
  }
}

/* 歌词解析 */
const parseLRC = text => text.split('\n').map(line => {
  const match = /\[(\d{2}):(\d{2})(?:\.(\d{2,3}))?\]/.exec(line)
  if (!match) return null
  const min = parseInt(match[1]), sec = parseInt(match[2])
  const ms = match[3] ? parseInt(match[3].padEnd(3, '0')) : 0
  const time = min * 60 + sec + ms / 1000
  const textContent = line.replace(/\[\d{2}:\d{2}(?:\.\d{2,3})?\]/, '').trim()
  return textContent ? { time, text: textContent } : null
}).filter(Boolean).sort((a, b) => a.time - b.time)

watch(currentSong, async newSong => {
  if (!newSong.url || !audioRef.value) return
  lyrics.value = []
  currentLyricIndex.value = 0
  if (newSong.lrc) {
    try {
      const res = await fetch(newSong.lrc)
      const text = await res.text()
      lyrics.value = parseLRC(text)
    } catch (e) { console.error('歌词加载失败', e) }
  }
})

/* 歌词滚动优化 + 修复最后一句跳回 */
const onTimeUpdate = () => {
  if (!lyrics.value.length) return
  const currentTime = audioRef.value.currentTime
  let i = lyrics.value.findIndex(line => line.time > currentTime)

  if (i === -1) {
    currentLyricIndex.value = lyrics.value.length - 1
  } else {
    currentLyricIndex.value = i > 0 ? i - 1 : 0
  }

  scrollToCurrentLyric()
  savePlaybackState()
}

const scrollToCurrentLyric = () => {
  if (scrollAnimationFrame) cancelAnimationFrame(scrollAnimationFrame)
  const activeEl = lineRefs.value[currentLyric.value.time]
  const container = lyricsRef.value
  if (!activeEl || !container) return
  const offset = activeEl.offsetTop - container.clientHeight / 2 + activeEl.clientHeight / 2
  scrollAnimationFrame = requestAnimationFrame(() => {
    container.scrollTo({ top: offset, behavior: 'smooth' })
  })
}

/* 保存播放状态（增加歌词进度） */
const savePlaybackState = () => {
  if (!currentSong.value || !audioRef.value) return
  localStorage.setItem('music-player-state', JSON.stringify({
    index: currentIndex.value,
    time: audioRef.value.currentTime,
    lyricIndex: currentLyricIndex.value
  }))
}

onMounted(async () => {
  try {
    const res = await fetch('https://raw.githubusercontent.com/OFXIV/resources/refs/heads/main/json/music.json')
    if (!res.ok) throw new Error('获取歌曲列表失败')
    const data = await res.json()
    songs.value = Array.isArray(data) ? data : []

    // 尝试恢复上次播放状态
    const state = JSON.parse(localStorage.getItem('music-player-state') || '{}')
    if (state.index >= 0 && state.index < songs.value.length) {
      currentIndex.value = state.index
      await nextTick()
      if (audioRef.value && state.time) {
        audioRef.value.currentTime = state.time
      }
      if (state.lyricIndex >= 0) {
        currentLyricIndex.value = state.lyricIndex
      }
    }
  } catch (err) {
    console.error(err)
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
  text-align: center;
  height: auto;
  padding-top: 10px;
  position: relative;
  color: #333;
  font-family: 'Helvetica Neue', Arial, sans-serif;
}

.stack-container {
  position: relative;
  width: 100%;
  min-height: 300px;
  display: flex;
  justify-content: center;
  align-items: center;
  perspective: 1200px;
}

.song-cover {
  position: absolute;
  top: 50%;
  width: 280px;
  height: 280px;
  border-radius: 14px;
  overflow: hidden;
  cursor: pointer;
  transform-style: preserve-3d;
  transition: transform 0.5s cubic-bezier(0.68, -0.55, 0.27, 1.55), 
              opacity 0.4s ease, box-shadow 0.4s;
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.295);
}

.song-cover img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  border-radius: inherit;
  user-select: none;
}

.song-cover.current {
  transform: translateX(0) translateY(-50%) rotateY(0deg) scale(1);
  z-index: 3;
}

.song-cover.prev {
  transform: translateX(-160px) translateY(-50%) rotateY(15deg) scale(0.85);
  opacity: 0.5;
  z-index: 2;
}

.song-cover.next {
  transform: translateX(160px) translateY(-50%) rotateY(-15deg) scale(0.85);
  opacity: 0.5;
  z-index: 2;
}

.song-info {
  font-weight: 500;
  color: #222;
  font-size: 14px;
}

.lyrics {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  height: 40px;
  backdrop-filter: blur(12px);
  overflow: hidden;
  padding: 14px;
  text-align: center;
  font-size: 14px;
  z-index: 9999;
}

.lyric-line {
  will-change: transform, opacity;
  opacity: 0.5;
  transform: scale(0.95);
  transition: all 0.15s ease;
  white-space: nowrap;
}

.lyric-line.active {
  color: #000;
  font-weight: 540;
  opacity: 1;
  transform: scale(1.1);
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
}

.lyric-fade-enter-active,
.lyric-fade-leave-active {
  transition: all 0.15s ease;
}

.lyric-fade-enter-from,
.lyric-fade-leave-to {
  opacity: 0;
  transform: translateY(12px);
}

.lyric-fade-enter-to,
.lyric-fade-leave-from {
  opacity: 1;
  transform: translateY(0);
}

/* 播放按钮样式 */
.play-btn {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 60px;
  height: 60px;
  border-radius: 50%;
  background: rgba(0, 0, 0, 0.35);
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: background 0.3s, transform 0.2s, box-shadow 0.3s;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
}

.play-btn:hover {
  background: rgba(0, 0, 0, 0.55);
  box-shadow: 0 6px 16px rgba(0, 0, 0, 0.25);
}

.play-btn:active {
  transform: translate(-50%, -50%) scale(0.92);
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.15);
}

.icon.play {
  width: 0;
  height: 0;
  border-left: 18px solid #fff;
  border-top: 12px solid transparent;
  border-bottom: 12px solid transparent;
}

.icon.pause {
  width: 18px;
  height: 18px;
  position: relative;
}

.icon.pause::before,
.icon.pause::after {
  content: '';
  position: absolute;
  top: 0;
  width: 5px;
  height: 100%;
  background: #fff;
  border-radius: 2px;
}

.icon.pause::before {
  left: 0;
}

.icon.pause::after {
  right: 0;
}

/* 响应式适配 */
@media (max-width: 480px) {
  .stack-container {
    min-height: 220px;
  }

  .song-cover {
    width: 180px;
    height: 180px;
  }

  .song-cover.prev {
    transform: translateX(-100px) translateY(-50%) rotateY(15deg) scale(0.75);
    opacity: 0.4;
  }

  .song-cover.next {
    transform: translateX(100px) translateY(-50%) rotateY(-15deg) scale(0.75);
    opacity: 0.4;
  }

  .play-btn {
    width: 40px;
    height: 40px;
  }

  .icon.play {
    border-left: 14px solid #fff;
    border-top: 9px solid transparent;
    border-bottom: 9px solid transparent;
  }

  .icon.pause {
    width: 14px;
    height: 14px;
  }

  .icon.pause::before,
  .icon.pause::after {
    width: 4px;
  }

  .song-info h2 {
    font-size: 12px;
    margin-top: 2px;
  }

  /* 歌词区域优化 */
  .lyrics {
    height: 30px;
    padding: 10px;
    font-size: 12px;
    line-height: 1.4;
    backdrop-filter: blur(8px);
  }

  .lyric-line {
    opacity: 0.6;
    transform: scale(0.95);
  }

  .lyric-line.active {
    font-size: 14px;
    opacity: 1;
    transform: scale(1);
  }

  /* 过渡动画保持流畅 */
  .lyric-fade-enter-from,
  .lyric-fade-leave-to {
    transform: translateY(8px);
  }
}
</style>