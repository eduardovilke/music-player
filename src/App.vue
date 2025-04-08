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
        @click="playOrPause"
        size="large"
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
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick, watch } from 'vue'
import WaveSurfer from 'wavesurfer.js'
import ZoomPlugin from 'wavesurfer.js/dist/plugins/zoom.esm.js'
import RegionsPlugin from 'wavesurfer.js/dist/plugins/regions.esm.js'

import Button from 'primevue/button'
import { Message } from 'primevue'

const wavesurfer = ref(null)
const playbackRate = ref(1.0)
const currentTime = ref(0)
const duration = ref(0)
const fileInput = ref(null)
const isPlaying = ref(false)
const regions = RegionsPlugin.create()
const minPxPerSec = ref(1)

const triggerFileInput = () => {
  fileInput.value.click()
}

const formatTime = (seconds) => {
  const min = Math.floor(seconds / 60)
  const sec = Math.floor(seconds % 60).toString().padStart(2, '0')
  return `${min}:${sec}`
}

onMounted(async () => {
  await nextTick();
  createWaveSurfer();
})

const playOrPause = () => {
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

  createWaveSurfer()
}

const handleFileUpload = (event) => {
  const file = event.target.files[0]
  if (file) {
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
      ZoomPlugin.create({
        scale: 0.5,
        maxZoom: 100,
      }),
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
    console.log('region-in', region)
    activeRegion = region
  })

  regions.on('region-out', (region) => {
    console.log('region-out', region)
    if (activeRegion === region) {
      region.play()
    }
  })

  regions.on('region-clicked', (region, e) => {
    console.log('regiao clicada')
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

watch(playbackRate, (newValue) => {
  wavesurfer.value?.setPlaybackRate(newValue)
})

watch(minPxPerSec, (newValue) => {
  wavesurfer.value?.zoom(newValue)
})

const forward = () => {
  const current = wavesurfer.value.getCurrentTime()
  const duration = wavesurfer.value.getDuration()
  wavesurfer.value.setTime(Math.min(current + 5, duration))
}

const backward = () => {
  const current = wavesurfer.value.getCurrentTime()
  wavesurfer.value.setTime(Math.max(current - 5, 0))
}
</script>

<style scoped>
.player {
  display: flex;
  flex-direction: column;
  justify-content: center;
  height: 100vh;
}

.controls, .speed-control {
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
</style>