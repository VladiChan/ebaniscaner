<!-- QrCameraScannerFixed.vue -->
<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import jsQR from 'jsqr'
import QrResultModal from './QrResultModal.vue' // <-- новый компонент

const video = ref(null)
const canvas = ref(null)
const stream = ref(null)
const errorMessage = ref('')        // только ошибки камеры
const showResultModal = ref(false)  // управляем модалкой
const lastScannedData = ref('')     // данные для передачи в модалку
const lastFrameImage = ref('')      // base64-кадр с зелёной рамкой

onMounted(async () => {
  try {
    stream.value = await navigator.mediaDevices.getUserMedia({
      video: { facingMode: 'environment' }
    })
    if (video.value) {
      video.value.srcObject = stream.value
      video.value.play()
    }
  } catch (err) {
    errorMessage.value = 'Нет доступа к камере или она не найдена'
    console.error(err)
  }
})

onUnmounted(() => {
  if (stream.value) {
    stream.value.getTracks().forEach(t => t.stop())
  }
})

const scanNow = () => {
  if (!video.value || video.value.readyState !== video.value.HAVE_ENOUGH_DATA) {
    errorMessage.value = 'Камера ещё не готова'
    return
  }

  const ctx = canvas.value.getContext('2d')
  canvas.value.width = video.value.videoWidth
  canvas.value.height = video.value.videoHeight
  ctx.drawImage(video.value, 0, 0, canvas.value.width, canvas.value.height)

  const imageData = ctx.getImageData(0, 0, canvas.value.width, canvas.value.height)
  const code = jsQR(imageData.data, imageData.width, imageData.height, {
    inversionAttempts: 'invertFirst'
  })

  if (code) {
    // Рисуем зелёную рамку
    drawGreenBorder(ctx, code.location)

    // Сохраняем кадр с рамкой как base64
    lastFrameImage.value = canvas.value.toDataURL('image/png')
    lastScannedData.value = code.data

    // Показываем модальное окно с результатом
    showResultModal.value = true
    errorMessage.value = '' // очищаем ошибки
  } else {
    errorMessage.value = 'QR-код не найден. Попробуйте ещё раз.'
  }
}

const drawGreenBorder = (ctx, location) => {
  ctx.strokeStyle = '#00ff00'
  ctx.lineWidth = 10
  ctx.beginPath()
  ctx.moveTo(location.topLeftCorner.x, location.topLeftCorner.y)
  ctx.lineTo(location.topRightCorner.x, location.topRightCorner.y)
  ctx.lineTo(location.bottomRightCorner.x, location.bottomRightCorner.y)
  ctx.lineTo(location.bottomLeftCorner.x, location.bottomLeftCorner.y)
  ctx.closePath()
  ctx.stroke()
}

// Закрытие модалки
const closeModal = () => {
  showResultModal.value = false
}
</script>

<template>
  <div class="scanner-container">
    <!-- Только камера и ошибки -->
    <div class="camera-wrapper">
      <video ref="video" playsinline muted class="camera-video"></video>
      
      <!-- Canvas нужен только для внутреннего захвата кадра (невидимый) -->
      <canvas ref="canvas" style="display: none;"></canvas>

      <div v-if="!stream" class="placeholder">
        Камера не доступна
      </div>
    </div>

    <!-- Сообщения об ошибках сканирования или статусе -->
    <div v-if="errorMessage" class="error-message">
      {{ errorMessage }}
    </div>

    <button @click="scanNow" class="scan-button">
      Сканировать QR-код
    </button>

    <!-- Модальное окно с результатом -->
    <QrResultModal
      v-if="showResultModal"
      :scanned-data="lastScannedData"
      :frame-image="lastFrameImage"
      @close="closeModal"
    />
  </div>
</template>

<style scoped>
/* ... оставляем почти всё как было ... */
.scanner-container {
  max-width: 420px;
  margin: 0 auto;
  padding: 20px;
  background: #e0e0e0;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.camera-wrapper {
  position: relative;
  background: #333;
  border-radius: 24px;
  overflow: hidden;
  aspect-ratio: 1 / 1;
  box-shadow: 0 10px 30px rgba(0,0,0,0.3);
}

.camera-video {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.placeholder {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #444;
  color: white;
  font-size: 18px;
  padding: 20px;
  text-align: center;
}

.error-message {
  background: #ff4444;
  color: white;
  padding: 14px;
  border-radius: 12px;
  text-align: center;
  font-weight: 500;
}

.scan-button {
  background: #333;
  color: white;
  border: none;
  padding: 18px;
  border-radius: 30px;
  font-size: 18px;
  font-weight: bold;
  cursor: pointer;
  box-shadow: 0 8px 20px rgba(0,0,0,0.2);
  transition: all 0.2s;
}

.scan-button:active {
  transform: scale(0.96);
}
</style>