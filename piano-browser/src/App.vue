<template>
  <div class="piano">
    <div class="keys" ref="keysContainer">
      <!-- teclas blancas -->
      <button
        v-for="(k, i) in whiteKeys"
        :key="k.key"
        class="key white"
        :data-index="i"
        :class="{ pressed: isPressedVisual(k.key) }"
        @mousedown.prevent="startKey(k.key, commaDown, $event.shiftKey)"
        @mouseup.prevent="stopKey(k.key, commaDown, $event.shiftKey)"
        @mouseleave.prevent="stopKey(k.key, commaDown, $event.shiftKey)"
        @touchstart.prevent="startKey(k.key, false, false)"
        @touchend.prevent="stopKey(k.key, false, false)"
      >
        <div class="label">{{ k.key }}</div>
        <div class="note">{{ k.name }}</div>
      </button>

      <!-- teclas negras (absolutas) -->
      <button
        v-for="(k, i) in blackKeys"
        :key="k.key"
        class="key black"
        :style="{ left: blackLefts[i] + 'px' }"
        :class="{ pressed: isPressedVisual(k.key) }"
        @mousedown.prevent="startKey(k.key, commaDown, $event.shiftKey)"
        @mouseup.prevent="stopKey(k.key, commaDown, $event.shiftKey)"
        @mouseleave.prevent="stopKey(k.key, commaDown, $event.shiftKey)"
        @touchstart.prevent="startKey(k.key, false, false)"
        @touchend.prevent="stopKey(k.key, false, false)"
      >
        <div class="label">{{ k.key }}</div>
        <div class="note">{{ k.name }}</div>
      </button>
    </div>

    <p class="hint">Mantén la tecla <strong>, </strong>(coma) para una octava abajo, <strong>Shift</strong> para una octava arriba.</p>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, onUnmounted, nextTick } from 'vue'

/* --- Mapeo de teclas con notas --- */
const whiteKeys = [
  { key: 'a', name: 'C4',  freq: 261.63 },
  { key: 's', name: 'D4',  freq: 293.66 },
  { key: 'd', name: 'E4',  freq: 329.63 },
  { key: 'f', name: 'F4',  freq: 349.23 },
  { key: 'g', name: 'G4',  freq: 392.00 },
  { key: 'h', name: 'A4',  freq: 440.00 },
  { key: 'j', name: 'B4',  freq: 493.88 }
]

const blackKeys = [
  { key: 'w', name: 'C#4', freq: 277.18, leftAnchor: 0 },
  { key: 'e', name: 'D#4', freq: 311.13, leftAnchor: 1 },
  { key: 't', name: 'F#4', freq: 369.99, leftAnchor: 3 },
  { key: 'y', name: 'G#4', freq: 415.30, leftAnchor: 4 },
  { key: 'u', name: 'A#4', freq: 466.16, leftAnchor: 5 }
]

/* --- Posicionamiento de teclas negras--- */
const keysContainer = ref(null)
const blackLefts = ref(Array(blackKeys.length).fill(0))

function updateBlackPositions() {
  nextTick(() => {
    const container = keysContainer.value
    if (!container) return
    const whites = Array.from(container.querySelectorAll('.key.white'))
    if (whites.length === 0) return
    const containerRect = container.getBoundingClientRect()
    const newLefts = blackKeys.map((bk) => {
      const anchor = bk.leftAnchor
      const leftWhite = whites[anchor]
      const rightWhite = whites[anchor + 1]
      if (!leftWhite || !rightWhite) return 0
      const rectA = leftWhite.getBoundingClientRect()
      const rectB = rightWhite.getBoundingClientRect()
      const center = (rectA.left + rectA.width / 2 + rectB.left + rectB.width / 2) / 2
      return Math.round(center - containerRect.left)
    })
    blackLefts.value = newLefts
  })
}

/* --- Estado visual: teclas presionadas--- */
const pressedVisual = reactive({}) 
function setPressedVisual(key, val) {
  if (val) pressedVisual[key] = true
  else delete pressedVisual[key]
}
function isPressedVisual(key) {
  return !!pressedVisual[key]
}

/* --- Estado para la coma como modificador --- */
const commaDown = ref(false)

/* --- Web Audio API: polyfonía y retrigger seguro con soporte de octava --- */
let audioCtx = null
const activeVoices = new Map()
const pressedKeyIds = new Set()

function ensureAudioContext() {
  if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)()
}

const ATTACK = 0.005
const RELEASE = 0.12
const SUSTAIN = 0.95
const MIN_GAIN = 0.0001

function createVoice(freq) {
  const now = audioCtx.currentTime
  const osc = audioCtx.createOscillator()
  const gain = audioCtx.createGain()
  osc.type = 'sawtooth'
  osc.frequency.value = freq
  gain.gain.setValueAtTime(MIN_GAIN, now)
  gain.gain.linearRampToValueAtTime(SUSTAIN, now + ATTACK)
  osc.connect(gain)
  gain.connect(audioCtx.destination)
  osc.start(now)
  return {
    osc, gain,
    release: (releaseTime = RELEASE) => {
      const t = audioCtx.currentTime
      try {
        gain.gain.cancelScheduledValues(t)
        const current = Math.max(gain.gain.value || MIN_GAIN, MIN_GAIN)
        gain.gain.setValueAtTime(current, t)
        gain.gain.exponentialRampToValueAtTime(MIN_GAIN, t + Math.max(0.01, releaseTime))
        osc.stop(t + Math.max(0.01, releaseTime) + 0.02)
      } catch (e) {}
    }
  }
}

function findBaseMapping(keyChar) {
  const k = typeof keyChar === 'string' && keyChar.length === 1 && /[a-zA-Z]/.test(keyChar)
    ? keyChar.toLowerCase()
    : keyChar
  return whiteKeys.find(x => x.key === k) || blackKeys.find(x => x.key === k)
}

function makeKeyId(keyChar, comma, shift) {
  return `${keyChar}|c${comma ? 1 : 0}s${shift ? 1 : 0}`
}

function getFreqWithOctave(keyChar, comma, shift) {
  const base = findBaseMapping(keyChar)
  if (!base) return null
  let freq = base.freq
  if (comma) freq = freq / 2
  if (shift) freq = freq * 2
  return freq
}

function forceReleaseAndRemove(keyId) {
  const arr = activeVoices.get(keyId)
  if (!arr) return
  const now = audioCtx.currentTime
  for (const v of arr) {
    try {
      v.gain.gain.cancelScheduledValues(now)
      const current = Math.max(v.gain.gain.value || MIN_GAIN, MIN_GAIN)
      v.gain.gain.setValueAtTime(current, now)
      v.gain.gain.exponentialRampToValueAtTime(MIN_GAIN, now + 0.02)
      v.osc.stop(now + 0.03)
    } catch (e) {}
  }
  activeVoices.delete(keyId)
}

/* Iniciar nota y marcar visual */
function startNoteForKey(keyChar, comma = false, shift = false) {
  const freq = getFreqWithOctave(keyChar, comma, shift)
  if (!freq) return
  ensureAudioContext()
  forceReleaseAndRemoveByBase(keyChar)
  clearPressedIdsForBase(keyChar)

  const keyId = makeKeyId(keyChar, comma, shift)
  const voice = createVoice(freq)
  voice.osc.onended = () => {
    try { voice.gain.disconnect() } catch (e) {}
    try { voice.osc.disconnect() } catch (e) {}
    const list = activeVoices.get(keyId)
    if (list) {
      const idx = list.indexOf(voice)
      if (idx !== -1) list.splice(idx, 1)
      if (list.length === 0) activeVoices.delete(keyId)
    }
  }
  const list = activeVoices.get(keyId) || []
  list.push(voice)
  activeVoices.set(keyId, list)
}

/* terminar con la nota */
function stopNoteForKey(keyChar, comma = false, shift = false) {
  const keyId = makeKeyId(keyChar, comma, shift)
  const list = activeVoices.get(keyId)
  if (!list) return
  for (const v of list) v.release(RELEASE)
}

/* Wrappers: manejan pressedKeyIds y estado visual */
function startKey(keyChar, comma = false, shift = false) {
  ensureAudioContext()
  if (audioCtx.state === 'suspended') audioCtx.resume()
  const keyId = makeKeyId(keyChar, comma, shift)
  if (pressedKeyIds.has(keyId)) return
  pressedKeyIds.add(keyId)
  // marcar visualmente
  setPressedVisual(keyChar, true)
  startNoteForKey(keyChar, comma, shift)
}

function stopKey(keyChar, comma = false, shift = false) {
  const keyId = makeKeyId(keyChar, comma, shift)
  if (pressedKeyIds.has(keyId)) pressedKeyIds.delete(keyId)
  // quitar visual
  setPressedVisual(keyChar, false)
  stopNoteForKey(keyChar, comma, shift)
}

/* --- Manejo de teclado físico: coma como modificador--- */
function handleKeyDown(e) {
  // coma como modificador
  if (e.key === ',') {
    commaDown.value = true
    e.preventDefault()
    return
  }

  const k = e.key.toLowerCase()
  if (!findBaseMapping(k)) return
  e.preventDefault()
  startKey(k, commaDown.value, e.shiftKey)
}

function handleKeyUp(e) {
  if (e.key === ',') {
    commaDown.value = false
    setPressedVisual(',', false)
    e.preventDefault()
    return
  }

  const k = e.key.toLowerCase()
  if (!findBaseMapping(k)) return
  e.preventDefault()
  stopKey(k, commaDown.value, e.shiftKey)
}

/* visibilidad y montaje */
function handleVisibilityChange() {
  if (document.hidden) {
    pressedKeyIds.clear()
    commaDown.value = false
    // limpiar visuales
    for (const k in pressedVisual) delete pressedVisual[k]
    for (const k of Array.from(activeVoices.keys())) forceReleaseAndRemove(k)
  }
}

/* --- helper: forzar release de todas las voces de una tecla base (ignora modificadores) --- */
function forceReleaseAndRemoveByBase(keyChar) {
  // recorremos todas las keyIds en activeVoices y liberamos las que empiecen por `${keyChar}|`
  const prefix = `${keyChar}|`
  const now = audioCtx ? audioCtx.currentTime : 0
  for (const keyId of Array.from(activeVoices.keys())) {
    if (keyId.startsWith(prefix)) {
      const arr = activeVoices.get(keyId)
      if (!arr) continue
      for (const v of arr) {
        try {
          v.gain.gain.cancelScheduledValues(now)
          const current = Math.max(v.gain.gain.value || MIN_GAIN, MIN_GAIN)
          v.gain.gain.setValueAtTime(current, now)
          v.gain.gain.exponentialRampToValueAtTime(MIN_GAIN, now + 0.02)
          v.osc.stop(now + 0.03)
        } catch (e) {}
      }
      activeVoices.delete(keyId)
    }
  }
}

/* --- helper: limpiar pressedKeyIds relacionados con la tecla base --- */
function clearPressedIdsForBase(keyChar) {
  const prefix = `${keyChar}|`
  for (const id of Array.from(pressedKeyIds)) {
    if (id.startsWith(prefix)) pressedKeyIds.delete(id)
  }
}

/* --- Modificación mínima en startNoteForKey: forzar release por base antes de crear nueva voz --- */


onMounted(() => {
  updateBlackPositions()
  window.addEventListener('resize', updateBlackPositions)
  window.addEventListener('keydown', handleKeyDown)
  window.addEventListener('keyup', handleKeyUp)
  document.addEventListener('visibilitychange', handleVisibilityChange)
})

onUnmounted(() => {
  window.removeEventListener('resize', updateBlackPositions)
  window.removeEventListener('keydown', handleKeyDown)
  window.removeEventListener('keyup', handleKeyUp)
  document.removeEventListener('visibilitychange', handleVisibilityChange)
  if (audioCtx) {
    for (const k of Array.from(activeVoices.keys())) forceReleaseAndRemove(k)
  }
})
</script>

<style scoped>
.piano { max-width: 900px; margin: 1rem auto; text-align: center; user-select: none; }
.keys {
  position: relative;
  height: 200px;
  display: flex;
  align-items: flex-end;
  gap: 2px;
  justify-content: center;
  padding: 0px;
}
.key.white {
  position: relative;
  width: 60px;
  height: 200px;
  background: linear-gradient(#fff,#f3f3f3);
  border: 1px solid #333;
  z-index: 1;
  display:flex;
  flex-direction:column;
  justify-content:flex-end;
  align-items:center;
  padding-bottom:8px;
  border-radius: 3px;
  transition: background 0.08s ease;
}
.key.black {
  position: absolute;
  top: 0;
  width: 44px;
  height: 130px;
  background: linear-gradient(#111,#000);
  color: #fff;
  z-index: 10;
  transform: translateX(-50%);
  display:flex;
  flex-direction:column;
  justify-content:flex-end;
  align-items:center;
  padding-bottom:8px;
  box-shadow: 0 6px 10px rgba(0,0,0,0.35);
  border: 1px solid rgba(255,255,255,0.03);
  pointer-events: auto;
  transition: background 0.08s ease, transform 0.06s ease;
}

/* estilos cuando están presionadas */
.key.white.pressed {
  background: #dcdcdc; 
  transform: translateY(2px);
}
.key.black.pressed {
  background: #666666;
  transform: translateY(2px);
   transform: translateX(-50%);
}

/* etiquetas */
.label { font-weight: 700; font-size: 12px; pointer-events: none; }
.note { font-size: 11px; opacity: 0.85; pointer-events: none; }
.hint { margin-top: 10px; color: #727272; font-size: 16px; }
</style>
