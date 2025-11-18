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
    <!-- Камера -->
    <div class="camera-wrapper">
      <video ref="video" playsinline muted class="camera-video"></video>
      <canvas ref="canvas" style="display: none;"></canvas>

      <div v-if="!stream" class="placeholder">
        Камера не доступна
      </div>
    </div>

    <!-- Ошибки -->
    <div v-if="errorMessage" class="error-message">
      {{ errorMessage }}
    </div>

    <!-- Кнопка сканирования -->
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
/* Главный контейнер — теперь просто центрирует всё на чистом фоне */
.scanner-container {
  min-height: 100vh;
  background: white;              /* ← чистый белый фон (можно поменять на #f5f5f5, #000 и т.д.) */
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 40px 20px 60px;
  gap: 30px;
}

/* Квадрат с камерой — остался прежний, но без серой обёртки */
.camera-wrapper {
  position: relative;
  width: 100%;
  max-width: 400px;
  aspect-ratio: 1 / 1;
  border-radius: 28px;
  overflow: hidden;
  box-shadow: 0 15px 40px rgba(0, 0, 0, 0.25);
  background: #000;               /* чёрный фон под видео */
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
  background: rgba(0, 0, 0, 0.8);
  color: white;
  font-size: 18px;
  text-align: center;
  padding: 20px;
}

/* Ошибки */
.error-message {
  background: #ff3b30;
  color: white;
  padding: 16px 24px;
  border-radius: 16px;
  font-weight: 600;
  max-width: 400px;
  text-align: center;
}

/* Кнопка — чуть больше воздуха снизу */
.scan-button {
  background: #000;
  color: white;
  border: none;
  padding: 20px 40px;
  border-radius: 50px;
  font-size: 19px;
  font-weight: bold;
  cursor: pointer;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.2);
  transition: all 0.2s;
  min-width: 280px;
}

.scan-button:active {
  transform: translateY(2px) scale(0.98);
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.2);
}
</style>