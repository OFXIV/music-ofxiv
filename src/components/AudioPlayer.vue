<template>
  <div class="music-player">
    <!-- 加载状态 -->
    <div v-if="loading" class="loading">加载中...</div>
    <div v-else-if="error" class="error">{{ error }}</div>

    <!-- 主体 -->
    <div
      v-else-if="songs.length"
      class="stack-container"
      @touchstart="onTouchStart"
      @touchmove="onTouchMove"
      @touchend="onTouchEnd"
    >
      <!-- 上一首 -->
      <div
        v-if="prevSong"
        class="song-cover prev"
        :class="animClass('prev')"
        @click="changeSong(-1)"
      >
        <img :src="prevSong.cover" :alt="`${prevSong.name}封面`" />
      </div>

      <!-- 当前播放 -->
      <div
        class="song-cover current"
        :class="animClass('current')"
        :style="{ transform: animationState.transform }"
      >
        <img :src="currentSong.cover" :alt="`${currentSong.name}封面`" />
        <div
          class="play-btn"
          @click.stop="ready ? togglePlay() : null"
          :class="{ disabled: !ready }"
        >
          <div :class="['icon', isPlaying ? 'pause' : 'play']"></div>
        </div>
      </div>

      <!-- 下一首 -->
      <div
        v-if="nextSong"
        class="song-cover next"
        :class="animClass('next')"
        @click="changeSong(1)"
      >
        <img :src="nextSong.cover" :alt="`${nextSong.name}封面`" />
      </div>
    </div>

    <!-- 歌曲信息 -->
    <div v-if="songs.length" class="song-info">
      <h2>{{ currentSong.name }} - {{ currentSong.artist }}</h2>
    </div>

    <!-- 进度条 -->
    <div v-if="songs.length" class="progress-bar">
      <span class="time">{{ formatTime(currentTime) }}</span>
      <input
        type="range"
        min="0"
        :max="duration"
        step="0.1"
        v-model="sliderValue"
        @input="onSeek"
        @change="onSeekEnd"
      />
      <span class="time">{{ formatTime(duration) }}</span>
    </div>

    <!-- 音量控制 -->
    <div class="volume-bar">
      <button @click="toggleMute" class="volume-btn">
        {{ muted ? '🔇' : '🔊' }}
      </button>
      <input
        type="range"
        min="0"
        max="1"
        step="0.01"
        v-model="volume"
        @input="onVolumeChange"
      />
    </div>

    <!-- 歌词 -->
    <div v-if="lyrics.length" class="lyrics" ref="lyricsRef">
      <transition-group name="lyric-fade" tag="div">
        <div
          class="lyric-line"
          v-for="line in lyrics"
          :key="line.time"
          :class="{ active: line === lyrics[currentLyricIndex] }"
          :ref="(el) => lineRefs[line.time] = el"
        >
          {{ line.text }}
        </div>
      </transition-group>
    </div>

    <!-- 隐藏播放器 -->
    <audio
      ref="audioRef"
      :src="currentSong.url"
      @ended="changeSong(1)"
      @play="isPlaying = true"
      @pause="isPlaying = false"
      @canplay="onCanPlay"
      @timeupdate="onTimeUpdate"
      preload="auto"
      style="display:none"
    ></audio>
  </div>
</template>

<script setup>
import { ref, reactive, computed, watch, onMounted, nextTick } from 'vue'

/* ===== 状态 ===== */
const songs = ref([])
const loading = ref(true)
const error = ref(null)
const currentIndex = ref(0)
const audioRef = ref(null)
const isPlaying = ref(false)
const ready = ref(false)

const currentTime = ref(0)
const duration = ref(0)
const sliderValue = ref(0)
const seeking = ref(false)

const volume = ref(1)
const muted = ref(false)

const lyrics = ref([])
const currentLyricIndex = ref(0)
const lyricsRef = ref(null)
const lineRefs = ref({})

/* ===== 动画状态合并 ===== */
const animationState = reactive({
  direction: '',
  animating: false,
  transform: ''
})

/* ===== 计算属性 ===== */
const currentSong = computed(() => songs.value[currentIndex.value] || {})
const prevSong = computed(() =>
  songs.value.length > 1
    ? songs.value[(currentIndex.value - 1 + songs.value.length) % songs.value.length]
    : null
)
const nextSong = computed(() =>
  songs.value.length > 1
    ? songs.value[(currentIndex.value + 1) % songs.value.length]
    : null
)
const animClass = (type) => animationState.direction ? `${type}-${animationState.direction}` : ''

/* ===== 播放控制 ===== */
const togglePlay = async () => {
  if (!audioRef.value) return
  try {
    if (!audioRef.value.paused) audioRef.value.pause()
    else await audioRef.value.play()
  } catch (err) {
    console.error('播放失败:', err)
    alert('播放失败，请检查网络或格式')
    isPlaying.value = false
  }
}

const changeSong = async (step) => {
  if (!songs.value.length || animationState.animating) return
  animationState.animating = true
  animationState.direction = step > 0 ? 'next' : 'prev'
  
  currentIndex.value = (currentIndex.value + step + songs.value.length) % songs.value.length

  await nextTick()
  animationState.animating = false
  animationState.direction = ''
  animationState.transform = ''
}

/* ===== 歌词解析 & 缓存 ===== */
const lyricCache = new Map()
const parseLRC = (text) => {
  const lines = text.split('\n')
  const result = []
  const timeReg = /\[(\d{2}):(\d{2})(?:\.(\d{2,3}))?\]/
  for (let line of lines) {
    const match = timeReg.exec(line)
    if (!match) continue
    const min = parseInt(match[1])
    const sec = parseInt(match[2])
    const ms = match[3] ? parseInt(match[3].padEnd(3, '0')) : 0
    const time = min * 60 + sec + ms / 1000
    const text = line.replace(timeReg, '').trim()
    if (text) result.push({ time, text })
  }
  return result.sort((a, b) => a.time - b.time)
}

/* ===== 自动播放 & 歌词加载 ===== */
watch(currentSong, async (newSong) => {
  if (!newSong?.url || !audioRef.value) return
  audioRef.value.load()
  if (isPlaying.value) await audioRef.value.play()

  lyrics.value = []
  currentLyricIndex.value = 0

  if (newSong.lrc) {
    if (lyricCache.has(newSong.lrc)) lyrics.value = lyricCache.get(newSong.lrc)
    else {
      try {
        const res = await fetch(newSong.lrc)
        const text = await res.text()
        const parsed = parseLRC(text)
        lyrics.value = parsed
        lyricCache.set(newSong.lrc, parsed)
      } catch (e) {
        console.error('歌词加载失败:', e)
      }
    }
  }
})

/* ===== 播放器事件 ===== */
const onCanPlay = () => {
  ready.value = true
  duration.value = audioRef.value?.duration || 0
}

const onTimeUpdate = () => {
  if (!seeking.value) {
    currentTime.value = audioRef.value?.currentTime || 0
    sliderValue.value = currentTime.value
    updateCurrentLyric()
  }
}

/* ===== 进度条 ===== */
const onSeek = (e) => { seeking.value = true; sliderValue.value = parseFloat(e.target.value) }
const onSeekEnd = () => { audioRef.value.currentTime = sliderValue.value; currentTime.value = sliderValue.value; seeking.value = false }
watch(currentTime, val => { if (!seeking.value) sliderValue.value = val })

/* ===== 音量 ===== */
watch(volume, val => {
  if(audioRef.value){
    audioRef.value.volume = val
    muted.value = val === 0
  }
})
const onVolumeChange = () => { if(audioRef.value) audioRef.value.volume = volume.value }
const toggleMute = () => { if(audioRef.value){ muted.value = !muted.value; audioRef.value.muted = muted.value } }

/* ===== 歌词滚动 ===== */
let lastIndex = -1
const updateCurrentLyric = () => {
  if(!lyrics.value.length) return
  for(let i=0;i<lyrics.value.length;i++){
    if(currentTime.value < lyrics.value[i].time){
      if(lastIndex !== i-1){
        currentLyricIndex.value = Math.max(0,i-1)
        scrollToCurrentLyric()
        lastIndex = i-1
      }
      return
    }
  }
  if(lastIndex !== lyrics.value.length-1){
    currentLyricIndex.value = lyrics.value.length-1
    scrollToCurrentLyric()
    lastIndex = lyrics.value.length-1
  }
}

const scrollToCurrentLyric = () => {
  const activeEl = lineRefs.value[lyrics.value[currentLyricIndex.value]?.time]
  const container = lyricsRef.value
  if(!activeEl || !container) return
  const offset = activeEl.offsetTop - container.clientHeight/2 + activeEl.clientHeight/2
  container.scrollTo({ top: offset, behavior:'smooth' })
}

/* ===== 格式化时间 ===== */
const formatTime = (sec) => {
  if (!sec || isNaN(sec)) return '00:00'
  const m = Math.floor(sec / 60)
  const s = Math.floor(sec % 60)
  return `${String(m).padStart(2,'0')}:${String(s).padStart(2,'0')}`
}

/* ===== 手势滑动 ===== */
let startX = 0, moveX = 0, raf
const threshold = 60

const onTouchStart = (e) => {
  if(animationState.animating) return
  startX = e.touches[0].clientX
  moveX = 0
}

const onTouchMove = (e) => {
  if(animationState.animating) return
  moveX = e.touches[0].clientX - startX
  const percent = Math.min(Math.abs(moveX)/100,1)
  animationState.direction = moveX>0?'prev':'next'
  
  cancelAnimationFrame(raf)
  raf = requestAnimationFrame(() => {
    animationState.transform = `translate3d(-50%,-50%,0) rotateY(${percent*60*(moveX>0?-1:1)}deg) scale(${1-0.1*percent})`
  })
}

const onTouchEnd = () => {
  if(animationState.animating) return
  if(Math.abs(moveX) > threshold) changeSong(moveX>0?-1:1)
  animationState.direction = ''
  animationState.transform = ''
  startX = moveX = 0
}

/* ===== 初始化 ===== */
onMounted(async () => {
  try{
    const res = await fetch('https://raw.githubusercontent.com/OFXIV/resources/refs/heads/main/json/music.json')
    if(!res.ok) throw new Error('获取歌曲列表失败')
    const data = await res.json()
    songs.value = Array.isArray(data)?data:[]
  }catch(err){
    console.error('加载歌曲失败:',err)
    error.value = '加载歌曲失败，请稍后重试'
  }finally{ loading.value = false }
})
</script>

<style scoped>
.music-player { display:flex; flex-direction:column; align-items:center; text-align:center; height:100vh; padding-top:20px; position:relative; }
.stack-container { position:relative; width:100%; height:400px; perspective:1000px; }
.song-cover { position:absolute; left:50%; transform:translate3d(-50%,-50%,0); border-radius:12px; overflow:hidden; transition:transform 0.5s cubic-bezier(0.4,0,0.2,1), opacity 0.5s ease; cursor:pointer; transform-style: preserve-3d; backface-visibility:hidden; }
.song-cover img { width:100%; height:100%; object-fit:cover; border-radius:inherit; }
.song-cover.current { top:50%; width:280px; height:280px; z-index:3; box-shadow:0 12px 24px rgba(0,0,0,0.3); }
.song-cover.prev { top:25%; width:200px; height:150px; z-index:2; opacity:0.6; transform: translate3d(-50%,-50%,0) scale(0.8) rotateX(10deg); }
.song-cover.next { top:75%; width:200px; height:150px; z-index:2; opacity:0.6; transform: translate3d(-50%,-50%,0) scale(0.8) rotateX(-10deg); }

.current-next { transform:translate3d(-50%, -120%,0) rotateX(60deg) scale(0.7); opacity:0; }
.next-next { transform:translate3d(-50%,-50%,0) scale(1) rotateX(0); opacity:1; z-index:3; }
.prev-next { transform:translate3d(-50%,120%,0) scale(0.7) rotateX(-30deg); opacity:0; }
.current-prev { transform:translate3d(-50%,120%,0) rotateX(-60deg) scale(0.7); opacity:0; }
.prev-prev { transform:translate3d(-50%,-50%,0) scale(1) rotateX(0); opacity:1; z-index:3; }
.next-prev { transform:translate3d(-50%,-120%,0) scale(0.7) rotateX(30deg); opacity:0; }

.play-btn { position:absolute; top:50%; left:50%; transform:translate(-50%,-50%); width:60px; height:60px; border-radius:50%; background:rgba(0,0,0,0.3); display:flex; align-items:center; justify-content:center; cursor:pointer; transition: background 0.3s, transform 0.2s; }
.play-btn:hover { background: rgba(0,0,0,0.5); }
.play-btn:active { transform: translate(-50%,-50%) scale(0.9); }
.play-btn.disabled { cursor: not-allowed; opacity:0.5; }
.icon.play { width:0;height:0;border-left:16px solid white;border-top:10px solid transparent;border-bottom:10px solid transparent; }
.icon.pause { width:16px;height:16px; position:relative; }
.icon.pause::before,.icon.pause::after{ content:''; position:absolute; top:0; width:4px; height:16px; background:white; }
.icon.pause::before{ left:0; } .icon.pause::after{ right:0; }

.song-info { font-weight:bold; color:#333; margin-top:20px; }
.progress-bar { display:flex; align-items:center; gap:10px; width:80%; max-width:400px; margin-top:15px; }
.progress-bar input[type='range']{ flex:1; -webkit-appearance:none; height:4px; background:#ddd; border-radius:2px; outline:none; cursor:pointer; }
.progress-bar input[type='range']::-webkit-slider-thumb{ -webkit-appearance:none; width:12px; height:12px; border-radius:50%; background:#333; cursor:pointer; }
.time { font-size:12px; color:#555; width:40px; text-align:center; }

.volume-bar { display:flex; align-items:center; gap:10px; margin-top:10px; width:80%; max-width:400px; }
.volume-bar input[type='range']{ flex:1; -webkit-appearance:none; height:4px; background:#ddd; border-radius:2px; }
.volume-bar input[type='range']::-webkit-slider-thumb{ -webkit-appearance:none; width:12px; height:12px; border-radius:50%; background:#333; cursor:pointer; }
.volume-btn { background:none; border:none; font-size:16px; cursor:pointer; }

.lyrics { position:fixed; bottom:0; left:0; right:0; height:120px; background:rgba(255,255,255,0.85); backdrop-filter:blur(8px); overflow:hidden; padding:10px; text-align:center; font-size:14px; line-height:1.8; z-index:9999; }
.lyric-line { opacity:0.5; transform:scale(0.95); transition: all 0.4s ease; }
.lyric-line.active { color:#000; font-weight:bold; font-size:16px; opacity:1; transform:scale(1); }

.lyric-fade-enter-active, .lyric-fade-leave-active { transition: all 0.4s ease; }
.lyric-fade-enter-from, .lyric-fade-leave-to { opacity:0; transform:translateY(10px); }
.lyric-fade-enter-to, .lyric-fade-leave-from { opacity:1; transform:translateY(0); }
</style>
