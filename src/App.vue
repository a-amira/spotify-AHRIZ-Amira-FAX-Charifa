<script setup>
import { ref, computed, onMounted } from 'vue'

const query = ref('daft punk')
const lastQuery = ref('')
const tracks = ref([])
const current = ref(null)
const playing = ref(false)
const loading = ref(false)
const error = ref('')
const progress = ref(0)
const duration = ref(0)
const volume = ref(0.7)
const audio = ref(null)

const currentIndex = computed(() => tracks.value.findIndex((t) => t.id === current.value?.id))
const progressPct = computed(() => (duration.value ? (progress.value / duration.value) * 100 : 0))
const volumePct = computed(() => volume.value * 100)

function formatMs(ms) {
  return formatTime(Math.round(ms / 1000))
}

function formatTime(s) {
  if (!s || Number.isNaN(s)) return '0:00'
  return `${Math.floor(s / 60)}:${String(Math.floor(s % 60)).padStart(2, '0')}`
}

function coverOf(t, size = 0) {
  const imgs = t?.album?.images ?? []
  return imgs[size]?.url ?? imgs[0]?.url ?? ''
}

function artistsOf(t) {
  return t?.artists?.map((a) => a.name).join(', ') ?? ''
}

function normalize(r) {
  const art = (r.artworkUrl100 || '').replace('100x100bb', '600x600bb')
  return {
    id: String(r.trackId),
    name: r.trackName,
    artists: [{ name: r.artistName }],
    album: { name: r.collectionName, images: [{ url: art }, { url: art }, { url: art }] },
    duration_ms: r.trackTimeMillis,
    preview_url: r.previewUrl
  }
}

async function search() {
  const q = query.value.trim()
  if (!q) return
  loading.value = true
  error.value = ''
  try {
    const url = `https://itunes.apple.com/search?term=${encodeURIComponent(q)}&media=music&entity=song&limit=10`
    const res = await fetch(url)
    if (!res.ok) throw new Error(`Erreur de recherche (${res.status}).`)
    const data = await res.json()
    tracks.value = (data.results ?? []).filter((r) => r.previewUrl).map(normalize)
    lastQuery.value = q
    if (!tracks.value.length) error.value = `Aucun résultat pour « ${q} ».`
  } catch (e) {
    tracks.value = []
    error.value = e.message
  } finally {
    loading.value = false
  }
}

async function resolvePreview(t) {
  return t.preview_url
}

async function play(t) {
  if (current.value?.id === t.id) {
    togglePlay()
    return
  }
  current.value = t
  playing.value = false
  progress.value = 0
  duration.value = 0
  error.value = ''
  const url = await resolvePreview(t)
  if (current.value?.id !== t.id) return
  if (!url) {
    error.value = 'Aucun extrait audio disponible pour ce titre.'
    return
  }
  audio.value.src = url
  audio.value.volume = volume.value
  try {
    await audio.value.play()
  } catch {
    playing.value = false
  }
}

function togglePlay() {
  if (!current.value) {
    if (tracks.value[0]) play(tracks.value[0])
    return
  }
  if (!audio.value.src) return
  if (audio.value.paused) audio.value.play()
  else audio.value.pause()
}

function next() {
  if (!tracks.value.length) return
  play(tracks.value[(currentIndex.value + 1) % tracks.value.length])
}

function prev() {
  if (!tracks.value.length) return
  if (audio.value?.currentTime > 3) {
    audio.value.currentTime = 0
    return
  }
  const i = currentIndex.value <= 0 ? tracks.value.length - 1 : currentIndex.value - 1
  play(tracks.value[i])
}

function seek(e) {
  if (!audio.value?.src) return
  audio.value.currentTime = Number(e.target.value)
}

function setVolume(e) {
  volume.value = Number(e.target.value)
  if (audio.value) audio.value.volume = volume.value
}

function onTime() {
  progress.value = audio.value.currentTime
}

function onMeta() {
  duration.value = audio.value.duration
}

onMounted(search)
</script>

<template>
  <div class="shell">
    <aside class="sidebar">
      <div class="logo">
        <svg viewBox="0 0 24 24" width="32" height="32" aria-hidden="true">
          <circle cx="12" cy="12" r="12" fill="#1db954" />
          <path d="M5.5 9.2c4.2-1.2 9.2-.9 13 1.3M6.3 12.6c3.5-1 7.5-.7 10.700 1.200M7 15.8c2.800-.8 5.700-.5 8.200 1" stroke="#000" stroke-width="1.8" stroke-linecap="round" fill="none" />
        </svg>
        <span>Spotify</span>
      </div>
      <nav class="nav-box">
        <a class="nav-item" href="#" @click.prevent="query = 'daft punk'; search()">
          <svg viewBox="0 0 24 24" width="24" height="24" fill="currentColor"><path d="M12.5 3.2a1 1 0 0 0-1 0L3 8.400V21h6.500v-6.500h5V21H21V8.400l-8.500-5.200z" /></svg>
          Accueil
        </a>
        <a class="nav-item active" href="#">
          <svg viewBox="0 0 24 24" width="24" height="24" fill="none" stroke="currentColor" stroke-width="2.200" stroke-linecap="round"><circle cx="10.500" cy="10.500" r="7" /><path d="M16 16l5 5" /></svg>
          Rechercher
        </a>
      </nav>
      <div class="nav-box library">
        <div class="lib-title">
          <svg viewBox="0 0 24 24" width="24" height="24" fill="currentColor"><path d="M3 22V3h2v19H3zm5 0V3h2v19H8zm5.500 0L11 3.600l2-.4L17 21.600l-3.500.4z" /></svg>
          Bibliothèque
        </div>
        <div class="lib-card">
          <strong>Retrouvez vos titres</strong>
          <span>Recherchez un artiste pour lancer un extrait.</span>
        </div>
      </div>
    </aside>

    <main class="main">
      <header class="topbar">
        <form class="search" @submit.prevent="search">
          <svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="2.200" stroke-linecap="round"><circle cx="10.500" cy="10.500" r="7" /><path d="M16 16l5 5" /></svg>
          <input v-model="query" type="search" placeholder="Que souhaitez-vous écouter ?" aria-label="Rechercher" />
        </form>
      </header>

      <section class="content">
        <h2 v-if="lastQuery">Résultats pour « {{ lastQuery }} »</h2>
        <p v-if="loading" class="status">Chargement…</p>
        <p v-else-if="error" class="status error">{{ error }}</p>

        <div v-if="tracks.length" class="table">
          <div class="row head">
            <span class="c-num">#</span>
            <span>Titre</span>
            <span class="c-album">Album</span>
            <span class="c-dur">Durée</span>
          </div>
          <button
            v-for="(t, i) in tracks"
            :key="t.id"
            class="row item"
            :class="{ active: current?.id === t.id }"
            @click="play(t)"
          >
            <span class="c-num">
              <span class="num">{{ i + 1 }}</span>
              <svg class="ico-play" viewBox="0 0 24 24" width="16" height="16" fill="currentColor"><path d="M7 4v16l13-8z" /></svg>
            </span>
            <span class="c-title">
              <img :src="coverOf(t, 2)" :alt="`Pochette de ${t.album.name}`" />
              <span class="names">
                <span class="name">{{ t.name }}</span>
                <span class="artist">{{ artistsOf(t) }}</span>
              </span>
            </span>
            <span class="c-album">{{ t.album.name }}</span>
            <span class="c-dur">{{ formatMs(t.duration_ms) }}</span>
          </button>
        </div>
      </section>
    </main>

    <footer class="player">
      <div class="now">
        <template v-if="current">
          <img :src="coverOf(current, 2)" :alt="`Pochette de ${current.album.name}`" />
          <div class="names">
            <span class="name">{{ current.name }}</span>
            <span class="artist">{{ artistsOf(current) }}</span>
          </div>
        </template>
      </div>

      <div class="center">
        <div class="controls">
          <button class="ctl" aria-label="Précédent" @click="prev">
            <svg viewBox="0 0 24 24" width="22" height="22" fill="currentColor"><path d="M6 5h2v14H6zM20 5v14L9 12z" /></svg>
          </button>
          <button class="ctl main-btn" :aria-label="playing ? 'Pause' : 'Lecture'" @click="togglePlay">
            <svg v-if="!playing" viewBox="0 0 24 24" width="20" height="20" fill="currentColor"><path d="M7 4v16l13-8z" /></svg>
            <svg v-else viewBox="0 0 24 24" width="20" height="20" fill="currentColor"><path d="M6 4h4v16H6zM14 4h4v16h-4z" /></svg>
          </button>
          <button class="ctl" aria-label="Suivant" @click="next">
            <svg viewBox="0 0 24 24" width="22" height="22" fill="currentColor"><path d="M16 5h2v14h-2zM4 5l11 7L4 19z" /></svg>
          </button>
        </div>
        <div class="seek">
          <span class="time">{{ formatTime(progress) }}</span>
          <input
            type="range"
            min="0"
            :max="duration || 0"
            step="0.1"
            :value="progress"
            :style="{ '--p': progressPct + '%' }"
            aria-label="Progression"
            @input="seek"
          />
          <span class="time">{{ formatTime(duration) }}</span>
        </div>
      </div>

      <div class="volume">
        <svg viewBox="0 0 24 24" width="20" height="20" fill="currentColor"><path d="M3 9v6h4l5 4V5L7 9H3zm13.500 3a4.500 4.500 0 0 0-2.500-4v8a4.500 4.500 0 0 0 2.500-4z" /></svg>
        <input
          type="range"
          min="0"
          max="1"
          step="0.01"
          :value="volume"
          :style="{ '--p': volumePct + '%' }"
          aria-label="Volume"
          @input="setVolume"
        />
      </div>
    </footer>

    <audio
      ref="audio"
      @timeupdate="onTime"
      @loadedmetadata="onMeta"
      @play="playing = true"
      @pause="playing = false"
      @ended="next"
    ></audio>
  </div>
</template>

<style>
:root {
  --bg: #000;
  --panel: #121212;
  --card: #1f1f1f;
  --hover: #2a2a2a;
  --green: #1db954;
  --text: #fff;
  --muted: #b3b3b3;
}
* { box-sizing: border-box; }
html, body, #app { height: 100%; margin: 0; }
body {
  background: var(--bg);
  color: var(--text);
  font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif;
  overflow: hidden;
}
</style>

<style scoped>
.shell {
  height: 100%;
  display: grid;
  grid-template-columns: 300px 1fr;
  grid-template-rows: 1fr 88px;
  gap: 8px;
  padding: 8px;
}

.sidebar { display: flex; flex-direction: column; gap: 8px; min-height: 0; }
.nav-box { background: var(--panel); border-radius: 8px; padding: 8px 12px; }
.logo { display: flex; align-items: center; gap: 10px; padding: 12px 16px 4px; font-weight: 800; font-size: 1.4rem; }
.nav-item {
  display: flex; align-items: center; gap: 18px; padding: 10px 12px;
  color: var(--muted); text-decoration: none; font-weight: 700; border-radius: 6px;
}
.nav-item:hover, .nav-item.active { color: var(--text); }
.library { flex: 1; padding: 16px; overflow: auto; }
.lib-title { display: flex; align-items: center; gap: 14px; color: var(--muted); font-weight: 700; margin-bottom: 20px; }
.lib-card { background: var(--card); border-radius: 8px; padding: 18px; display: flex; flex-direction: column; gap: 8px; font-size: .9rem; }
.lib-card span { color: var(--muted); }

.main {
  background: linear-gradient(180deg, #2b2b2b 0, var(--panel) 280px);
  border-radius: 8px;
  overflow-y: auto;
  min-height: 0;
}
.topbar { position: sticky; top: 0; z-index: 2; padding: 16px 24px; background: rgba(18, 18, 18, 0.6); backdrop-filter: blur(8px); }
.search {
  display: flex; align-items: center; gap: 10px; max-width: 420px;
  background: var(--hover); border: 2px solid transparent; border-radius: 999px; padding: 0 16px; color: var(--muted);
}
.search:hover { border-color: #3a3a3a; }
.search:focus-within { border-color: var(--text); color: var(--text); }
.search input {
  flex: 1; height: 46px; background: transparent; border: 0; color: var(--text); font-size: .95rem; outline: none;
}

.content { padding: 8px 24px 32px; }
h2 { font-size: 1.5rem; margin: 8px 0 20px; }
.status { color: var(--muted); }
.error { color: #ff6b6b; }

.table { display: flex; flex-direction: column; }
.row {
  display: grid; grid-template-columns: 48px minmax(0, 1fr) minmax(0, 1fr) 64px;
  align-items: center; gap: 16px; padding: 8px 16px; border-radius: 4px;
  color: var(--muted); font-size: .9rem; text-align: left;
}
.head { border-bottom: 1px solid #2a2a2a; border-radius: 0; margin-bottom: 8px; font-size: .8rem; padding-bottom: 10px; }
.item { background: transparent; border: 0; font: inherit; cursor: pointer; width: 100%; }
.item:hover { background: rgba(255, 255, 255, 0.1); }
.item.active .name { color: var(--green); }
.c-num { display: flex; justify-content: center; align-items: center; }
.ico-play { display: none; color: var(--text); }
.item:hover .num { display: none; }
.item:hover .ico-play { display: block; }
.item.active .num { color: var(--green); }
.c-dur { text-align: right; }
.c-title { display: flex; align-items: center; gap: 14px; min-width: 0; }
.c-title img, .now img { width: 44px; height: 44px; border-radius: 4px; object-fit: cover; }
.c-album, .name, .artist { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.names { display: flex; flex-direction: column; min-width: 0; gap: 2px; }
.name { color: var(--text); font-size: 1rem; }
.item:hover .artist { color: var(--text); }

.player {
  grid-column: 1 / -1;
  display: grid; grid-template-columns: 1fr minmax(320px, 2fr) 1fr; align-items: center; gap: 16px;
  padding: 0 16px; background: var(--bg);
}
.now { display: flex; align-items: center; gap: 14px; min-width: 0; }
.now img { width: 56px; height: 56px; }
.now .name { font-size: .95rem; }
.now .artist { font-size: .8rem; color: var(--muted); }
.center { display: flex; flex-direction: column; align-items: center; gap: 8px; }
.controls { display: flex; align-items: center; gap: 20px; }
.ctl { background: none; border: 0; color: var(--muted); cursor: pointer; display: grid; place-items: center; padding: 4px; }
.ctl:hover { color: var(--text); }
.main-btn { width: 36px; height: 36px; border-radius: 50%; background: var(--text); color: #000; }
.main-btn:hover { color: #000; transform: scale(1.06); }
.seek { display: flex; align-items: center; gap: 10px; width: 100%; max-width: 620px; }
.time { font-size: .75rem; color: var(--muted); min-width: 34px; text-align: center; }
.volume { display: flex; align-items: center; justify-content: flex-end; gap: 10px; color: var(--muted); }
.volume input { width: 110px; }

input[type='range'] {
  -webkit-appearance: none; appearance: none; flex: 1; height: 4px; border-radius: 2px; cursor: pointer;
  background: linear-gradient(to right, var(--text) var(--p, 0%), #4d4d4d var(--p, 0%));
}
input[type='range']:hover { background: linear-gradient(to right, var(--green) var(--p, 0%), #4d4d4d var(--p, 0%)); }
input[type='range']::-webkit-slider-thumb { -webkit-appearance: none; width: 12px; height: 12px; border-radius: 50%; background: var(--text); opacity: 0; }
input[type='range']:hover::-webkit-slider-thumb { opacity: 1; }
input[type='range']::-moz-range-thumb { width: 12px; height: 12px; border: 0; border-radius: 50%; background: var(--text); }

button:focus-visible, a:focus-visible, input[type='range']:focus-visible { outline: 2px solid var(--green); outline-offset: 2px; }

@media (max-width: 800px) {
  .shell { grid-template-columns: 1fr; grid-template-rows: 1fr 120px; }
  .sidebar, .c-album, .volume { display: none; }
  .row { grid-template-columns: 36px minmax(0, 1fr) 56px; }
  .player { grid-template-columns: 1fr; padding: 8px 16px; gap: 4px; }
  .now { display: none; }
}
</style>