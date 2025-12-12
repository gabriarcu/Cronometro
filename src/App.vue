<template>
  <div id="app">
    <!-- Animated Background -->
    <div class="animated-bg">
      <div class="orb orb-1"></div>
      <div class="orb orb-2"></div>
      <div class="orb orb-3"></div>
    </div>

    <!-- Main Container -->
    <div class="container">
      <!-- Mode Toggle -->
      <div class="mode-toggle">
        <button :class="['mode-btn', { active: mode === 'stopwatch' }]" @click="switchMode('stopwatch')">
          <span class="icon">⏱️</span>
          Cronometro
        </button>
        <button :class="['mode-btn', { active: mode === 'countdown' }]" @click="switchMode('countdown')">
          <span class="icon">⏰</span>
          Countdown
        </button>
      </div>

      <!-- Timer Display -->
      <div class="timer-container">
        <img class="timer-image" src="./assets/cronometro.png" alt="Cronometro" />
        <div class="timer-display">
          {{ formattedTime }}
        </div>
      </div>

      <!-- End Time Input (Countdown Mode) -->
      <div v-if="mode === 'countdown'" class="end-time-section">
        <div class="input-group">
          <label>Ora di Fine:</label>
          <div class="time-inputs">
            <input type="number" v-model.number="endHour" min="0" max="23" placeholder="HH" class="time-input"
              :disabled="timer !== null" />
            <span class="separator">:</span>
            <input type="number" v-model.number="endMinute" min="0" max="59" placeholder="MM" class="time-input"
              :disabled="timer !== null" />
          </div>
        </div>
        <div v-if="countdownTarget" class="countdown-info">
          Tempo rimanente fino alle {{ formatEndTime }}
        </div>
      </div>

      <!-- Control Buttons -->
      <div class="controls">
        <button v-if="timer === null" class="btn btn-play" @click="play"
          :disabled="mode === 'countdown' && !isEndTimeValid">
          <span class="btn-icon">▶</span>
          VIA
        </button>
        <button v-else class="btn btn-pause" @click="stop">
          <span class="btn-icon">⏸</span>
          PAUSA
        </button>
        <button class="btn btn-reset" @click="clear">
          <span class="btn-icon">↻</span>
          RESET
        </button>
      </div>

      <!-- Progress Bar (Countdown Mode) -->
      <div v-if="mode === 'countdown' && countdownTarget" class="progress-container">
        <div class="progress-bar">
          <div class="progress-fill" :style="{ width: progressPercentage + '%' }"></div>
        </div>
      </div>

      <!-- Install Banner -->
      <div v-if="showInstallBanner" class="install-banner">
        <div class="install-content">
          <span class="install-icon">📲</span>
          <div class="install-text">
            <strong>Installa CronoTimer</strong>
            <p>Aggiungi alla schermata home per un accesso rapido</p>
          </div>
          <button class="btn-install" @click="installApp">Installa</button>
          <button class="btn-close-banner" @click="dismissBanner">✕</button>
        </div>
      </div>

      <!-- Interval History -->
      <div v-if="intervalList.length > 0" class="history">
        <h3 class="history-title">Storico Tempi</h3>
        <div class="history-list">
          <div v-for="(item, index) in intervalList" :key="index" class="history-item">
            <span class="history-number">#{{ intervalList.length - index }}</span>
            <span class="history-time">{{ item }}</span>
          </div>
        </div>
        <button class="btn btn-clear" @click="clearIntervalList">
          <span class="btn-icon">🗑️</span>
          SVUOTA STORICO
        </button>
      </div>
    </div>

    <!-- Footer -->
    <footer class="app-footer">
      <p>
        2025 - CronoTimer -
        <a href="https://www.gabriarcu.it" target="_blank" rel="noopener noreferrer">Gabriarcu</a>
      </p>
    </footer>
  </div>
</template>

<script>
import { ref, computed, onMounted, onUnmounted, watch } from 'vue'
import { useLocalStorage } from '@vueuse/core'
import { useToast } from 'vue-toastification'

export default {
  name: 'App',
  setup() {
    const toast = useToast()

    // State
    const mode = useLocalStorage('timer-mode', 'stopwatch')
    const sec = ref(0)
    const min = ref(0)
    const hour = ref(0)
    const timer = ref(null)
    const intervalList = useLocalStorage('timer-history', [])

    // Countdown specific
    const endHour = ref(null)
    const endMinute = ref(null)
    const countdownTarget = ref(null)
    const totalCountdownSeconds = ref(0)

    // PWA Install
    const showInstallBanner = ref(false)
    const deferredPrompt = ref(null)

    // Computed
    const formattedTime = computed(() => {
      return `${zfill(hour.value)}:${zfill(min.value)}:${zfill(sec.value)}`
    })

    const formatEndTime = computed(() => {
      if (endHour.value !== null && endMinute.value !== null) {
        return `${zfill(endHour.value)}:${zfill(endMinute.value)}`
      }
      return ''
    })

    const isEndTimeValid = computed(() => {
      return endHour.value !== null &&
        endMinute.value !== null &&
        endHour.value >= 0 &&
        endHour.value <= 23 &&
        endMinute.value >= 0 &&
        endMinute.value <= 59
    })

    const progressPercentage = computed(() => {
      if (totalCountdownSeconds.value === 0) return 100
      const currentSeconds = hour.value * 3600 + min.value * 60 + sec.value
      return Math.max(0, Math.min(100, (currentSeconds / totalCountdownSeconds.value) * 100))
    })

    // Methods
    const zfill = (number) => {
      return number.toString().padStart(2, '0')
    }

    const switchMode = (newMode) => {
      if (timer.value !== null) {
        toast.warning('Ferma il timer prima di cambiare modalità')
        return
      }
      mode.value = newMode
      clear()
    }

    const play = () => {
      if (mode.value === 'countdown') {
        if (!isEndTimeValid.value) {
          toast.error('Inserisci un orario di fine valido')
          return
        }
        startCountdown()
      } else {
        startStopwatch()
      }
    }

    const startStopwatch = () => {
      playing()
      timer.value = setInterval(playing, 1000)
      toast.success('Cronometro avviato')
    }

    const startCountdown = () => {
      // Calculate target time
      const now = new Date()
      const target = new Date()
      target.setHours(endHour.value, endMinute.value, 0, 0)

      // If target is in the past, set it for tomorrow
      if (target <= now) {
        target.setDate(target.getDate() + 1)
      }

      countdownTarget.value = target

      // Calculate total seconds
      const diffMs = target - now
      const diffSeconds = Math.floor(diffMs / 1000)
      totalCountdownSeconds.value = diffSeconds

      // Set initial time
      hour.value = Math.floor(diffSeconds / 3600)
      min.value = Math.floor((diffSeconds % 3600) / 60)
      sec.value = diffSeconds % 60

      timer.value = setInterval(countdown, 1000)
      toast.success('Countdown avviato')
    }

    const playing = () => {
      sec.value++
      if (sec.value >= 60) {
        sec.value = 0
        min.value++
      }
      if (min.value >= 60) {
        min.value = 0
        hour.value++
      }
    }

    const countdown = () => {
      if (sec.value > 0) {
        sec.value--
      } else if (min.value > 0) {
        sec.value = 59
        min.value--
      } else if (hour.value > 0) {
        sec.value = 59
        min.value = 59
        hour.value--
      } else {
        // Countdown finished
        stop()
        toast.success('⏰ Countdown terminato!', {
          timeout: 5000,
        })
        playNotificationSound()
      }
    }

    const stop = () => {
      if (timer.value !== null) {
        clearInterval(timer.value)
        timer.value = null
        pause()
        toast.info('Timer in pausa')
      }
    }

    const pause = () => {
      const formattedTimer = formattedTime.value
      intervalList.value.unshift(formattedTimer)

      // Keep only last 20 entries
      if (intervalList.value.length > 20) {
        intervalList.value = intervalList.value.slice(0, 20)
      }
    }

    const clear = () => {
      if (timer.value !== null) {
        clearInterval(timer.value)
        timer.value = null
      }
      sec.value = 0
      min.value = 0
      hour.value = 0
      countdownTarget.value = null
      totalCountdownSeconds.value = 0
    }

    const clearIntervalList = () => {
      intervalList.value = []
      toast.info('Storico cancellato')
    }

    const playNotificationSound = () => {
      // Create a simple beep sound using Web Audio API
      try {
        const audioContext = new (window.AudioContext || window.webkitAudioContext)()
        const oscillator = audioContext.createOscillator()
        const gainNode = audioContext.createGain()

        oscillator.connect(gainNode)
        gainNode.connect(audioContext.destination)

        oscillator.frequency.value = 800
        oscillator.type = 'sine'

        gainNode.gain.setValueAtTime(0.3, audioContext.currentTime)
        gainNode.gain.exponentialRampToValueAtTime(0.01, audioContext.currentTime + 0.5)

        oscillator.start(audioContext.currentTime)
        oscillator.stop(audioContext.currentTime + 0.5)
      } catch (e) {
        console.log('Audio notification not supported')
      }
    }

    // Keyboard shortcuts
    const handleKeyPress = (e) => {
      if (e.code === 'Space') {
        e.preventDefault()
        if (timer.value === null) {
          play()
        } else {
          stop()
        }
      } else if (e.code === 'KeyR') {
        e.preventDefault()
        clear()
      }
    }

    // PWA Install Methods
    const installApp = async () => {
      if (!deferredPrompt.value) return

      deferredPrompt.value.prompt()
      const { outcome } = await deferredPrompt.value.userChoice

      if (outcome === 'accepted') {
        toast.success('App installata con successo!')
      }

      deferredPrompt.value = null
      showInstallBanner.value = false
      localStorage.setItem('pwa-install-dismissed', 'true')
    }

    const dismissBanner = () => {
      showInstallBanner.value = false
      localStorage.setItem('pwa-install-dismissed', 'true')
    }

    // Lifecycle
    onMounted(() => {
      window.addEventListener('keydown', handleKeyPress)

      // PWA Install prompt
      const dismissed = localStorage.getItem('pwa-install-dismissed')
      if (!dismissed) {
        window.addEventListener('beforeinstallprompt', (e) => {
          e.preventDefault()
          deferredPrompt.value = e
          showInstallBanner.value = true
        })
      }
    })

    onUnmounted(() => {
      if (timer.value !== null) {
        clearInterval(timer.value)
      }
      window.removeEventListener('keydown', handleKeyPress)
    })

    // Watch for mode changes
    watch(mode, () => {
      if (mode.value === 'countdown') {
        const now = new Date()
        endHour.value = now.getHours()
        endMinute.value = now.getMinutes() + 5 // Default to 5 minutes from now
        if (endMinute.value >= 60) {
          endMinute.value -= 60
          endHour.value++
          if (endHour.value >= 24) endHour.value = 0
        }
      }
    })

    return {
      mode,
      sec,
      min,
      hour,
      timer,
      intervalList,
      endHour,
      endMinute,
      countdownTarget,
      formattedTime,
      formatEndTime,
      isEndTimeValid,
      progressPercentage,
      showInstallBanner,
      zfill,
      switchMode,
      play,
      stop,
      clear,
      clearIntervalList,
      installApp,
      dismissBanner
    }
  }
}
</script>

<style scoped>
/* Animated Background */
.animated-bg {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 0;
  overflow: hidden;
  pointer-events: none;
}

.orb {
  position: absolute;
  border-radius: 50%;
  filter: blur(80px);
  opacity: 0.5;
  animation: float 20s infinite ease-in-out;
}

.orb-1 {
  width: 400px;
  height: 400px;
  background: linear-gradient(135deg, #1C2E3A 0%, #274557 100%);
  top: -100px;
  left: -100px;
  animation-delay: 0s;
}

.orb-2 {
  width: 350px;
  height: 350px;
  background: linear-gradient(135deg, #274557 0%, #1C2E3A 100%);
  bottom: -100px;
  right: -100px;
  animation-delay: 7s;
}

.orb-3 {
  width: 300px;
  height: 300px;
  background: linear-gradient(135deg, #15232C 0%, #1C2E3A 100%);
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  animation-delay: 14s;
}

@keyframes float {

  0%,
  100% {
    transform: translate(0, 0) scale(1);
  }

  33% {
    transform: translate(50px, -50px) scale(1.1);
  }

  66% {
    transform: translate(-50px, 50px) scale(0.9);
  }
}

/* Main Container */
#app {
  display: flex;
  width: 100%;
  min-height: 100vh;
  align-items: center;
  justify-content: center;
  flex-direction: column;
  padding: 20px;
  position: relative;
}

.container {
  position: relative;
  z-index: 1;
  width: 100%;
  max-width: 600px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 30px;
}

/* Mode Toggle */
.mode-toggle {
  display: flex;
  gap: 15px;
  background: rgba(255, 255, 255, 0.05);
  backdrop-filter: blur(10px);
  border-radius: 20px;
  padding: 8px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3);
}

.mode-btn {
  padding: 12px 30px;
  border: none;
  border-radius: 15px;
  background: transparent;
  color: rgba(255, 255, 255, 0.6);
  font-size: 16px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  gap: 8px;
  font-family: 'Outfit', sans-serif;
}

.mode-btn .icon {
  font-size: 20px;
}

.mode-btn:hover {
  color: rgba(255, 255, 255, 0.9);
  background: rgba(255, 255, 255, 0.05);
}

.mode-btn.active {
  background: linear-gradient(135deg, #1C2E3A 0%, #274557 100%);
  color: white;
  box-shadow: 0 4px 20px rgba(28, 46, 58, 0.6);
}

/* Timer Container */
.timer-container {
  position: relative;
  width: 100%;
  max-width: 420px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.timer-image {
  width: 100%;
  max-width: 420px;
  height: auto;
  filter: drop-shadow(0 10px 40px rgba(28, 46, 58, 0.6));
  animation: pulse 3s infinite ease-in-out;
}

@keyframes pulse {

  0%,
  100% {
    transform: scale(1);
  }

  50% {
    transform: scale(1.02);
  }
}

.timer-display {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, calc(-50% + 8px));
  color: #fff;
  font-size: clamp(32px, 6vw, 52px);
  font-weight: 600;
  text-shadow:
    0 0 20px rgba(235, 160, 73, 0.6),
    0 0 40px rgba(235, 160, 73, 0.3),
    0 4px 8px rgba(0, 0, 0, 0.5);
  letter-spacing: 0.05em;
  font-variant-numeric: tabular-nums;
}

/* End Time Section */
.end-time-section {
  width: 100%;
  max-width: 420px;
  background: rgba(255, 255, 255, 0.05);
  backdrop-filter: blur(10px);
  border-radius: 20px;
  padding: 25px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3);
}

.input-group {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.input-group label {
  color: rgba(255, 255, 255, 0.9);
  font-size: 16px;
  font-weight: 500;
}

.time-inputs {
  display: flex;
  align-items: center;
  gap: 10px;
  justify-content: center;
}

.time-input {
  width: 80px;
  padding: 15px;
  font-size: 24px;
  font-weight: 600;
  text-align: center;
  background: rgba(255, 255, 255, 0.1);
  border: 2px solid rgba(255, 255, 255, 0.2);
  border-radius: 12px;
  color: white;
  font-family: 'Outfit', sans-serif;
  transition: all 0.3s ease;
}

.time-input:focus {
  outline: none;
  border-color: #EBA049;
  box-shadow: 0 0 20px rgba(235, 160, 73, 0.4);
  background: rgba(255, 255, 255, 0.15);
}

.time-input:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.time-input::placeholder {
  color: rgba(255, 255, 255, 0.3);
}

.separator {
  font-size: 32px;
  color: rgba(255, 255, 255, 0.6);
  font-weight: 600;
}

.countdown-info {
  margin-top: 15px;
  text-align: center;
  color: rgba(255, 255, 255, 0.7);
  font-size: 14px;
}

/* Progress Bar */
.progress-container {
  width: 100%;
  max-width: 420px;
}

.progress-bar {
  width: 100%;
  height: 8px;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 10px;
  overflow: hidden;
  box-shadow: inset 0 2px 4px rgba(0, 0, 0, 0.2);
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #EBA049 0%, #f4b563 100%);
  border-radius: 10px;
  transition: width 1s linear;
  box-shadow: 0 0 10px rgba(235, 160, 73, 0.6);
}

/* Controls */
.controls {
  display: flex;
  gap: 15px;
  flex-wrap: wrap;
  justify-content: center;
}

.btn {
  padding: 15px 35px;
  font-size: 18px;
  font-weight: 600;
  border: none;
  border-radius: 15px;
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  gap: 10px;
  font-family: 'Outfit', sans-serif;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
  position: relative;
  overflow: hidden;
}

.btn::before {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  width: 0;
  height: 0;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.2);
  transform: translate(-50%, -50%);
  transition: width 0.6s, height 0.6s;
}

.btn:hover::before {
  width: 300px;
  height: 300px;
}

.btn-icon {
  font-size: 20px;
  position: relative;
  z-index: 1;
}

.btn:active {
  transform: scale(0.95);
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-play {
  background: linear-gradient(135deg, #EBA049 0%, #f4b563 100%);
  color: white;
}

.btn-play:hover {
  box-shadow: 0 6px 25px rgba(235, 160, 73, 0.5);
  transform: translateY(-2px);
}

.btn-pause {
  background: linear-gradient(135deg, #8A949C 0%, #BCC7D1 100%);
  color: #15232C;
}

.btn-pause:hover {
  box-shadow: 0 6px 25px rgba(138, 148, 156, 0.4);
  transform: translateY(-2px);
}

.btn-reset {
  background: linear-gradient(135deg, #274557 0%, #1C2E3A 100%);
  color: white;
}

.btn-reset:hover {
  box-shadow: 0 6px 25px rgba(39, 69, 87, 0.5);
  transform: translateY(-2px);
}

.btn-clear {
  background: linear-gradient(135deg, #BCC7D1 0%, #F4F7FA 100%);
  color: #15232C;
  margin-top: 15px;
}

.btn-clear:hover {
  box-shadow: 0 6px 25px rgba(188, 199, 209, 0.4);
  transform: translateY(-2px);
}

/* History */
.history {
  width: 100%;
  max-width: 420px;
  background: rgba(255, 255, 255, 0.05);
  backdrop-filter: blur(10px);
  border-radius: 20px;
  padding: 25px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3);
  display: flex;
  flex-direction: column;
  align-items: center;
}

.history-title {
  color: rgba(255, 255, 255, 0.9);
  font-size: 20px;
  font-weight: 600;
  margin-bottom: 20px;
  text-align: center;
}

.history-list {
  width: 100%;
  max-height: 300px;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 10px;
  padding-right: 10px;
}

.history-list::-webkit-scrollbar {
  width: 6px;
}

.history-list::-webkit-scrollbar-track {
  background: rgba(255, 255, 255, 0.05);
  border-radius: 10px;
}

.history-list::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.2);
  border-radius: 10px;
}

.history-list::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.3);
}

.history-item {
  background: rgba(255, 255, 255, 0.08);
  padding: 15px 20px;
  border-radius: 12px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  transition: all 0.3s ease;
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.history-item:hover {
  background: rgba(255, 255, 255, 0.12);
  transform: translateX(5px);
}

.history-number {
  color: rgba(255, 255, 255, 0.5);
  font-size: 14px;
  font-weight: 500;
}

.history-time {
  color: rgba(255, 255, 255, 0.9);
  font-size: 18px;
  font-weight: 600;
  font-variant-numeric: tabular-nums;
}

/* Install Banner */
.install-banner {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: 1000;
  background: linear-gradient(135deg, #1C2E3A 0%, #274557 100%);
  backdrop-filter: blur(10px);
  border-bottom: 2px solid rgba(235, 160, 73, 0.3);
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
  animation: slideDown 0.5s ease-out;
}

@keyframes slideDown {
  from {
    transform: translateY(-100%);
  }

  to {
    transform: translateY(0);
  }
}

.install-content {
  max-width: 1020px;
  margin: 0 auto;
  padding: 15px 20px;
  display: flex;
  align-items: center;
  gap: 15px;
}

.install-icon {
  font-size: 32px;
  flex-shrink: 0;
}

.install-text {
  flex: 1;
  color: white;
}

.install-text strong {
  display: block;
  font-size: 16px;
  margin-bottom: 4px;
}

.install-text p {
  font-size: 13px;
  color: rgba(255, 255, 255, 0.8);
  margin: 0;
}

.btn-install {
  padding: 10px 24px;
  background: linear-gradient(135deg, #EBA049 0%, #f4b563 100%);
  color: white;
  border: none;
  border-radius: 8px;
  font-weight: 600;
  font-size: 14px;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 2px 10px rgba(235, 160, 73, 0.3);
  font-family: 'Outfit', sans-serif;
  flex-shrink: 0;
}

.btn-install:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 15px rgba(235, 160, 73, 0.5);
}

.btn-close-banner {
  width: 32px;
  height: 32px;
  background: rgba(255, 255, 255, 0.1);
  border: none;
  border-radius: 50%;
  color: white;
  font-size: 18px;
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.btn-close-banner:hover {
  background: rgba(255, 255, 255, 0.2);
  transform: rotate(90deg);
}

/* Footer */
.app-footer {
  width: 100%;
  text-align: center;
  padding: 30px 20px 20px;
  color: rgba(255, 255, 255, 0.6);
  font-size: 14px;
  margin-top: auto;
}

.app-footer p {
  margin: 0;
}

.app-footer a {
  color: #EBA049;
  text-decoration: none;
  font-weight: 500;
  transition: all 0.3s ease;
}

.app-footer a:hover {
  color: #f4b563;
  text-decoration: underline;
}

/* Responsive Design */
@media (max-width: 768px) {
  .container {
    gap: 20px;
  }

  .mode-btn {
    padding: 10px 20px;
    font-size: 14px;
  }

  .timer-image {
    max-width: 320px;
  }

  .controls {
    gap: 10px;
  }

  .btn {
    padding: 12px 25px;
    font-size: 16px;
  }

  .time-input {
    width: 70px;
    padding: 12px;
    font-size: 20px;
  }
}

@media (max-width: 480px) {
  .mode-toggle {
    gap: 8px;
    padding: 6px;
  }

  .mode-btn {
    padding: 8px 15px;
    font-size: 13px;
  }

  .mode-btn .icon {
    font-size: 16px;
  }

  .timer-image {
    max-width: 280px;
  }

  .btn {
    padding: 10px 20px;
    font-size: 14px;
  }

  .time-input {
    width: 60px;
    padding: 10px;
    font-size: 18px;
  }

  .install-content {
    flex-wrap: wrap;
    gap: 10px;
  }

  .install-text strong {
    font-size: 14px;
  }

  .install-text p {
    font-size: 12px;
  }

  .btn-install {
    padding: 8px 16px;
    font-size: 13px;
  }
}
</style>
