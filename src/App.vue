<template>
  <div class="player">
    <div class="upload-remove">
      <div class="upload">
        <Button rounded icon="pi pi-upload" @click="triggerFileInput"/>
        <input type="file" accept="audio/*" ref="fileInput" @change="handleFileUpload" hidden />
      </div>
      <div class="remove">
        <Button 
          rounded 
          icon="pi pi-times" 
          severity="danger" 
          variant="outlined" 
          @click="resetPlayer" 
        />
      </div>
    </div>

    <div class="info">
      <Message size="small" icon="pi pi-exclamation-circle" severity="info">To mark the time, drag and drop.</Message>
    </div>

    <div class="remove-marks">
      <Button
        :disabled="regions.regions.length === 0"
        rounded
        size="small"
        icon="pi pi-times" 
        variant="text" 
        severity="warn"
        @click="clearRegions" 
      />
    </div>

    <div id="waveform"></div>

    <div class="timer-zoom">
      <div>
        {{ formatTime(currentTime) }} / {{ formatTime(duration) }}
      </div>
      <div class="zoom">
        <Button 
          rounded
          outlined
          icon="pi pi-search-minus"
          @click="minPxPerSec -= 5"
          variant="text"
        />
        <Button 
          rounded
          outlined
          icon="pi pi-search-plus"
          @click="minPxPerSec += 5"
          variant="text"
        />
      </div>
    </div>

    <div class="speed-control">
      <Button 
        rounded
        outlined
        icon="pi pi-minus"
        @click="playbackRate -= 0.1"
        variant="text"
      />
      <span>{{ playbackRate.toFixed(1) }}x</span>
      <Button 
        @click="playbackRate += 0.1"
        rounded 
        outlined
        variant="text"
        icon="pi pi-plus"
      />
    </div>

    <div class="bottom">
      <div class="music-title">
        {{ musicName }}
      </div>
      <div class="controls">
        <Button 
          :disabled="!fileInput" 
          icon="pi pi-backward"
          rounded
          variant="text"
          @click="backward" 
        />
        <Button
          :disabled="!fileInput" 
          :icon="isPlaying ? 'pi pi-pause' : 'pi pi-play'" 
          rounded
          outlined 
          @click="playPause"
          class="play-pause"
        />
        <Button 
          :disabled="!fileInput" 
          icon="pi pi-forward"
          variant="text"
          rounded 
          @click="forward" 
        />
      </div>
      <div class="volume">
        <Button 
          @click="muteDesmute"
          rounded 
          outlined
          variant="text"
          :icon="volume === 0 ? 'pi pi-volume-off' : 'pi pi-volume-up'"
        />
        <Slider v-model="volume" orientation="horizontal" />
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick, watch } from 'vue'
import { Message, Slider, Button } from 'primevue'
import WaveSurfer from 'wavesurfer.js'
import RegionsPlugin from 'wavesurfer.js/dist/plugins/regions.esm.js'

const wavesurfer = ref(null)
const playbackRate = ref(1.0)
const currentTime = ref(0)
const duration = ref(0)
const fileInput = ref(null)
const isPlaying = ref(false)
const regions = RegionsPlugin.create()
const minPxPerSec = ref(1)
const volume = ref(100)
const lastVolume = ref(null)
const musicName = ref(null)

watch(playbackRate, (newValue) => wavesurfer.value?.setPlaybackRate(newValue))
watch(minPxPerSec, (newValue) => wavesurfer.value?.zoom(newValue))
watch(volume, (newValue, oldValue) => {
  lastVolume.value = oldValue;
  wavesurfer.value.setVolume(newValue / 100)
})

onMounted(async () => {
  await nextTick();
  createWaveSurfer();
})

const triggerFileInput = () => fileInput.value.click()

const formatTime = (seconds) => {
  const min = Math.floor(seconds / 60)
  const sec = Math.floor(seconds % 60).toString().padStart(2, '0')
  return `${min}:${sec}`
}

const playPause = () => {
  isPlaying.value = !isPlaying.value;
  wavesurfer.value?.playPause()
}

const resetPlayer = () => {
  if (wavesurfer.value) {
    wavesurfer.value.destroy()
    wavesurfer.value = null
  }

  fileInput.value.value = ''
  currentTime.value = 0
  duration.value = 0
  isPlaying.value = false
  musicName.value = null

  createWaveSurfer()
}

const handleFileUpload = (event) => {
  const file = event.target.files[0]
  if (file) {
    musicName.value = file.name
    const objectUrl = URL.createObjectURL(file)
    wavesurfer.value.load(objectUrl)
  }
}

const createWaveSurfer = () => {
  wavesurfer.value = WaveSurfer.create({
    container: '#waveform',
    waveColor: '#34D399',
    progressColor: '#A78BFA',
    barWidth: 4,
    height: 200,
    minPxPerSec,
    barRadius: 10,
    plugins: [
      regions
    ]
  })

  wavesurfer.value.on('audioprocess', () => {
    currentTime.value = wavesurfer.value.getCurrentTime()
  })

  wavesurfer.value.on('ready', () => {
    duration.value = wavesurfer.value.getDuration()
  })

  regions?.enableDragSelection({
    color: 'rgba(253, 224, 71, 0.3)',
  })

  let activeRegion = null

  regions.on('region-in', (region) => {
    activeRegion = region
  })

  regions.on('region-out', (region) => {
    if (activeRegion === region) {
      region.play()
    }
  })

  regions.on('region-clicked', (region, e) => {
    e.stopPropagation()
    activeRegion = region
    region.play(true)
    region.setOptions({ color: 'rgba(244, 114, 182, 0.3)' })
  })

  wavesurfer.value.on('interaction', () => {
    activeRegion = null
  })
}

const clearRegions = () => regions.clearRegions()

const forward = () => {
  const current = wavesurfer.value.getCurrentTime()
  const duration = wavesurfer.value.getDuration()
  wavesurfer.value.setTime(Math.min(current + 5, duration))
}

const backward = () => wavesurfer.value.setTime(
  Math.max(
    wavesurfer.value.getCurrentTime() - 5, 
    0
  )
)
const muteDesmute = () => {
  if(volume.value === 0 ) {
    volume.value = lastVolume.value;
  } else {
    volume.value = 0;
  }
}
</script>

<style scoped>
.player {
  display: flex;
  flex-direction: column;
  justify-content: center;
  height: 100vh;
}

.speed-control, .controls {
  display: flex;
  justify-content: center;
  align-items: center;
}

.upload-remove {
  display: flex;
  justify-content: space-between;
}

.play-pause {
  margin: 0 1rem;
}

.info {
  margin: 2rem 0;
}

.remove-marks {
  display: flex;
  justify-content: flex-end;
}

.timer-zoom {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-top: 1rem;
}

.bottom {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  align-items: center;
}

.volume {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: flex-end;
}

.volume .p-slider {
  margin-left: 6px;
  width: 50%;
}

.music-title {
  max-width: 400px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
</style>