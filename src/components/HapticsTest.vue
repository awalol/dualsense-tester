<script setup lang="ts">
import { computed, onUnmounted, ref, shallowRef } from 'vue'
import { useDevice } from '@/composables/useInjectValues'
import { DeviceConnectionType } from '@/device-based-router/shared'
import { fillOutputReportChecksum } from '@/utils/dualsense/crc32.util'
import { hidLogger } from '@/utils/logger.util'
import DouButton from './base/DouButton.vue'
import GeneralContainer from './common/GeneralContainer.vue'

// --- Constants (from SAxense) ---
const REPORT_ID = 0x32
const REPORT_DATA_SIZE = 141 // bytes sent via sendReport (excluding report_id handled by WebHID)
const SAMPLE_SIZE = 64 // audio bytes per report
const SAMPLE_RATE = 3000
const SEND_INTERVAL_MS = (1000 * SAMPLE_SIZE) / (SAMPLE_RATE * 2) // ~10.67ms

const device = useDevice()
const isBluetooth = computed(() => device.value.connectionType === DeviceConnectionType.Bluetooth)

// --- State ---
const pcmFile = shallowRef<File | null>(null)
const pcmData = shallowRef<Uint8Array | null>(null)
const isPlaying = ref(false)
const playbackProgress = ref(0) // 0-100
const playbackOffset = ref(0)
const statusText = ref('')
const errorText = ref('')

// --- Report building ---
let reportSeqCounter = 0
let packetCounter = 0

/**
 * Build a 0x32 haptics report (SAxense format).
 *
 * Layout (141 bytes, report_id 0x32 is separate in WebHID):
 *   [0]       tag(4) | seq(4)
 *   [1]       packet_0x11 header: pid=0x11, sized=1 → 0x91
 *   [2]       packet_0x11 length: 7
 *   [3..9]    packet_0x11 data: {0xFE, 0, 0, 0, 0, 0xFF, counter}
 *   [10]      packet_0x12 header: pid=0x12, sized=1 → 0x92
 *   [11]      packet_0x12 length: 64
 *   [12..75]  PCM audio samples (64 bytes)
 *   [76..136] zeroes (padding)
 *   [137..140] CRC32 (little-endian)
 *
 * Reference: https://github.com/egormanga/SAxense
 */
function buildReport(samples: Uint8Array): Uint8Array {
  const data = new Uint8Array(REPORT_DATA_SIZE)

  // Byte 0: tag=0 (lower nibble), seq (upper nibble)
  data[0] = (reportSeqCounter & 0x0F) << 4

  // Packet 0x11 (control)
  data[1] = 0x91 // pid=0x11(6bits) | unk=0(1bit) | sized=1(1bit)
  data[2] = 0x07 // length=7
  data[3] = 0xFE // 0b11111110
  data[4] = 0x00
  data[5] = 0x00
  data[6] = 0x00
  data[7] = 0x00
  data[8] = 0xFF
  data[9] = packetCounter & 0xFF // rolling counter

  // Packet 0x12 (audio data)
  data[10] = 0x92 // pid=0x12(6bits) | unk=0(1bit) | sized=1(1bit)
  data[11] = 0x40 // length=64
  data.set(samples.subarray(0, SAMPLE_SIZE), 12)

  // CRC32: fillOutputReportChecksum computes crc32([0xA2, reportId], payload)
  // which matches SAxense's CRC32 with 0xA2 seed + report_id prefix
  fillOutputReportChecksum(REPORT_ID, data)

  packetCounter++
  return data
}

async function sendHapticsReport(samples: Uint8Array) {
  const reportData = buildReport(samples)
  try {
    await device.value.device.sendReport(REPORT_ID, reportData as BufferSource)
  }
  catch (err) {
    hidLogger.error('Failed to send 0x32 report', err)
    throw err
  }
}

// --- File handling ---
function onFileChange(event: Event) {
  const input = event.target as HTMLInputElement
  const file = input.files?.[0]
  if (!file)
    return

  errorText.value = ''
  pcmFile.value = file
  statusText.value = `已加载: ${file.name} (${(file.size / 1024).toFixed(1)} KB)`

  const reader = new FileReader()
  reader.onload = () => {
    pcmData.value = new Uint8Array(reader.result as ArrayBuffer)
    const durationSec = pcmData.value.length / (SAMPLE_RATE * 2) // 2 channels, 1 byte each
    statusText.value = `已加载: ${file.name} (${(file.size / 1024).toFixed(1)} KB, ~${durationSec.toFixed(1)}s @ ${SAMPLE_RATE}Hz u8 stereo)`
  }
  reader.onerror = () => {
    errorText.value = '文件读取失败'
  }
  reader.readAsArrayBuffer(file)
}

// --- Playback (serialized send loop, no concurrent HID writes) ---
const stopRequested = ref(false)

function startPlayback() {
  if (!pcmData.value || isPlaying.value)
    return
  if (!isBluetooth.value) {
    errorText.value = '0x32 报文仅支持蓝牙连接'
    return
  }

  isPlaying.value = true
  errorText.value = ''
  playbackOffset.value = 0
  playbackProgress.value = 0
  reportSeqCounter = 0
  packetCounter = 0
  stopRequested.value = false

  const rawData = pcmData.value
  let packetIndex = 0
  const totalPackets = Math.ceil(rawData.length / SAMPLE_SIZE)

  // 预构建所有报文，避免播放时的 GC 和分配开销
  const reports: Uint8Array[] = new Array(totalPackets)
  for (let i = 0; i < totalPackets; i++) {
    const chunk = new Uint8Array(SAMPLE_SIZE)
    const offset = i * SAMPLE_SIZE
    const toCopy = Math.min(SAMPLE_SIZE, rawData.length - offset)
    chunk.set(rawData.subarray(offset, offset + toCopy))
    reports[i] = chunk
  }

  // 串行发送循环：等上一包发完再发下一包，通过 sleep 校正节奏
  async function sendLoop() {
    const startTime = performance.now()
    let batchStartTime = startTime

    while (packetIndex < totalPackets && !stopRequested.value) {
      try {
        await sendHapticsReport(reports[packetIndex]!)
      }
      catch (err) {
        errorText.value = `发送失败: ${err}`
        break
      }

      reportSeqCounter = (reportSeqCounter + 1) & 0x0F
      packetIndex++

      // 更新 UI（节流：每 8 包更新一次，避免频繁触发响应式）
      if ((packetIndex & 7) === 0 || packetIndex === totalPackets) {
        playbackOffset.value = packetIndex * SAMPLE_SIZE
        playbackProgress.value = Math.min(100, Math.round((packetIndex / totalPackets) * 100))
      }

      // 日志：每 10 包统计一次发送耗时
      if (packetIndex % 10 === 0) {
        const elapsedMs = performance.now() - batchStartTime
        hidLogger.debug(`HapticsTest`, `Sent 10 packets in ${elapsedMs.toFixed(1)} ms`)
        batchStartTime = performance.now()
      }

      // 自校正等待：根据理论时间线计算距下一包的剩余时间
      const nextPacketTime = startTime + packetIndex * SEND_INTERVAL_MS
      const delay = nextPacketTime - performance.now()
      if (delay > 1) {
        await new Promise(r => setTimeout(r, delay))
      }
      // delay <= 1 时直接发下一包（追赶），但因为串行所以不会并发堆积
    }

    playbackProgress.value = 100
    isPlaying.value = false
  }

  sendLoop()
}

function stopPlayback() {
  stopRequested.value = true
  isPlaying.value = false
}

onUnmounted(() => {
  stopPlayback()
})

// --- File size info ---
const fileDuration = computed(() => {
  if (!pcmData.value)
    return ''
  const sec = pcmData.value.length / (SAMPLE_RATE * 2)
  const min = Math.floor(sec / 60)
  const s = (sec % 60).toFixed(1)
  return min > 0 ? `${min}m ${s}s` : `${s}s`
})
</script>

<template>
  <GeneralContainer title="Haptics Audio (0x32)" tag="SAxense">
    <div class="flex flex-col gap-3 p-3">
      <!-- Connection type warning -->
      <div v-if="!isBluetooth" class="text-sm text-amber-500">
        ⚠ 0x32 haptics 报文仅适用于蓝牙连接
      </div>

      <!-- File input -->
      <div class="flex flex-col gap-2">
        <label class="text-sm font-medium">
          选择 PCM 文件
          <span class="ms-1 text-xs opacity-60">(u8, 2ch, {{ SAMPLE_RATE }}Hz, 无头)</span>
        </label>
        <input
          type="file"
          accept=".pcm,.raw,.bin,*"
          class="file-input"
          :disabled="isPlaying"
          @change="onFileChange"
        >
      </div>

      <!-- Status -->
      <div v-if="statusText" class="text-sm opacity-80">
        {{ statusText }}
      </div>
      <div v-if="errorText" class="text-sm text-red-500">
        {{ errorText }}
      </div>

      <!-- Playback controls -->
      <div class="flex items-center gap-3">
        <DouButton
          :disabled="!pcmData || isPlaying || !isBluetooth"
          @click="startPlayback"
        >
          ▶ 播放
        </DouButton>
        <DouButton
          :disabled="!isPlaying"
          @click="stopPlayback"
        >
          ⏹ 停止
        </DouButton>
        <span v-if="pcmData" class="text-sm opacity-60">
          {{ fileDuration }}
        </span>
      </div>

      <!-- Progress bar -->
      <div v-if="isPlaying || playbackProgress > 0" class="flex flex-col gap-1">
        <div class="h-2 w-full overflow-hidden rounded-full bg-gray-200 dark:bg-gray-700">
          <div
            class="h-full rounded-full bg-primary transition-all duration-100"
            :style="{ width: `${playbackProgress}%` }"
          />
        </div>
        <div class="text-end text-xs opacity-60">
          {{ playbackProgress }}%
        </div>
      </div>

      <!-- Info -->
      <details class="text-xs opacity-50">
        <summary class="cursor-pointer select-none">
          报文格式说明
        </summary>
        <div class="mt-1 flex flex-col gap-0.5 border-s border-current/20 ps-2">
          <div>Report ID: 0x32 ({{ REPORT_DATA_SIZE }} bytes payload)</div>
          <div>音频格式: unsigned 8-bit, 2 channels, {{ SAMPLE_RATE }} Hz</div>
          <div>每包 {{ SAMPLE_SIZE }} 字节音频数据, 发送间隔 ~{{ SEND_INTERVAL_MS.toFixed(1) }}ms</div>
          <div>包含 CRC32 校验 (0xA2 seed)</div>
          <div>
            参考:
            <a
              href="https://github.com/egormanga/SAxense"
              target="_blank"
              class="underline"
            >egormanga/SAxense</a>
          </div>
        </div>
      </details>
    </div>
  </GeneralContainer>
</template>

<style scoped>
.file-input {
  @apply text-sm file:mr-3 file:rounded-full file:border-0 file:px-4 file:py-1.5
  file:text-sm file:font-medium file:bg-primary/10 file:text-primary
  file:cursor-pointer hover:file:bg-primary/20 file:transition
  cursor-pointer;
}
</style>
