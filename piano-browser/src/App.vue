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

    <div class="controls" aria-label="Synth controls">
      <div class="panel-title">Synth Tone</div>
      <div class="control-card">
        <label for="attack">Attack</label>
        <input id="attack" v-model.number="synthParams.attack" type="range" min="0.001" max="0.5" step="0.001" />
        <span>{{ synthParams.attack.toFixed(3) }}s</span>
      </div>

      <div class="control-card">
        <label for="release">Release</label>
        <input id="release" v-model.number="synthParams.release" type="range" min="0.01" max="1" step="0.01" />
        <span>{{ synthParams.release.toFixed(2) }}s</span>
      </div>

      <div class="control-card">
        <label for="sustain">Sustain</label>
        <input id="sustain" v-model.number="synthParams.sustain" type="range" min="0.01" max="1" step="0.01" />
        <span>{{ synthParams.sustain.toFixed(2) }}</span>
      </div>

      <div class="control-card">
        <label for="minGain">Min Gain</label>
        <input id="minGain" v-model.number="synthParams.minGain" type="range" min="0.00001" max="0.2" step="0.00001" />
        <span>{{ synthParams.minGain.toFixed(5) }}</span>
      </div>

      <div class="control-card">
        <label for="waveform">Wave</label>
        <select id="waveform" v-model="synthParams.waveform">
          <option value="sawtooth">Saw</option>
          <option value="sine">Sine</option>
          <option value="square">Square</option>
          <option value="triangle">Triangle</option>
        </select>
      </div>
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

/* --- Estado para los modificadores de octava --- */
const commaDown = ref(false)
const shiftDown = ref(false)

let audioCtx = null
const activeVoices = new Map()
const pressedKeyIds = new Set()
const activeBaseStates = new Map()

function ensureAudioContext() {
  if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)()
}

const synthParams = reactive({
  attack: 0.005,
  release: 0.12,
  sustain: 0.95,
  minGain: 0.0001,
  waveform: 'sawtooth'
})

function createVoice(freq) {
  const now = audioCtx.currentTime
  const osc = audioCtx.createOscillator()
  const gain = audioCtx.createGain()
  osc.type = synthParams.waveform
  osc.frequency.value = freq
  gain.gain.setValueAtTime(synthParams.minGain, now)
  gain.gain.linearRampToValueAtTime(synthParams.sustain, now + synthParams.attack)
  osc.connect(gain)
  gain.connect(audioCtx.destination)
  osc.start(now)
  return {
    osc, gain,
    release: (releaseTime = synthParams.release) => {
      const t = audioCtx.currentTime
      try {
        gain.gain.cancelScheduledValues(t)
        const current = Math.max(gain.gain.value || synthParams.minGain, synthParams.minGain)
        gain.gain.setValueAtTime(current, t)
        gain.gain.exponentialRampToValueAtTime(synthParams.minGain, t + Math.max(0.01, releaseTime))
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
      const current = Math.max(v.gain.gain.value || synthParams.minGain, synthParams.minGain)
      v.gain.gain.setValueAtTime(current, now)
      v.gain.gain.exponentialRampToValueAtTime(synthParams.minGain, now + Math.max(0.01, synthParams.release))
      v.osc.stop(now + Math.max(0.02, synthParams.release + 0.01))
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
  for (const v of list) v.release(synthParams.release)
}

/* Wrappers: manejan pressedKeyIds y estado visual */
function startKey(keyChar, comma = false, shift = false) {
  ensureAudioContext()
  if (audioCtx.state === 'suspended') audioCtx.resume()
  const keyId = makeKeyId(keyChar, comma, shift)
  if (pressedKeyIds.has(keyId)) return
  pressedKeyIds.add(keyId)
  activeBaseStates.set(keyChar, { keyId, comma, shift })
  // marcar visualmente
  setPressedVisual(keyChar, true)
  startNoteForKey(keyChar, comma, shift)
}

function stopKey(keyChar, comma = false, shift = false) {
  const state = getActiveStateForKey(keyChar, comma, shift)
  if (pressedKeyIds.has(state.keyId)) pressedKeyIds.delete(state.keyId)
  activeBaseStates.delete(keyChar)
  // quitar visual
  setPressedVisual(keyChar, false)
  stopNoteForKey(keyChar, state.comma, state.shift)
}

/* --- Manejo de teclado físico: coma como modificador--- */
function handleKeyDown(e) {
  if (e.key === ',') {
    commaDown.value = true
    syncHeldNotesWithModifiers()
    e.preventDefault()
    return
  }

  if (e.key === 'Shift') {
    shiftDown.value = true
    syncHeldNotesWithModifiers()
    return
  }

  if (e.repeat) {
    e.preventDefault()
    return
  }

  const k = e.key.toLowerCase()
  if (!findBaseMapping(k)) return
  e.preventDefault()
  startKey(k, commaDown.value, shiftDown.value || e.shiftKey)
}

function handleKeyUp(e) {
  if (e.key === ',') {
    commaDown.value = false
    syncHeldNotesWithModifiers()
    setPressedVisual(',', false)
    e.preventDefault()
    return
  }

  if (e.key === 'Shift') {
    shiftDown.value = false
    syncHeldNotesWithModifiers()
    return
  }

  const k = e.key.toLowerCase()
  if (!findBaseMapping(k)) return
  e.preventDefault()
  stopKey(k, commaDown.value, shiftDown.value || e.shiftKey)
}

/* visibilidad y montaje */
function handleVisibilityChange() {
  if (document.hidden) {
    pressedKeyIds.clear()
    activeBaseStates.clear()
    commaDown.value = false
    shiftDown.value = false
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
          const current = Math.max(v.gain.gain.value || synthParams.minGain, synthParams.minGain)
          v.gain.gain.setValueAtTime(current, now)
          v.gain.gain.exponentialRampToValueAtTime(synthParams.minGain, now + Math.max(0.01, synthParams.release))
          v.osc.stop(now + Math.max(0.02, synthParams.release + 0.01))
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

function getActiveStateForKey(keyChar, comma = false, shift = false) {
  const existing = activeBaseStates.get(keyChar)
  if (existing) return existing
  const keyId = makeKeyId(keyChar, comma, shift)
  return { keyId, comma, shift }
}

function syncHeldNotesWithModifiers() {
  for (const [baseKey, currentState] of Array.from(activeBaseStates.entries())) {
    const targetKeyId = makeKeyId(baseKey, commaDown.value, shiftDown.value)
    if (currentState.keyId === targetKeyId) continue

    pressedKeyIds.delete(currentState.keyId)
    stopNoteForKey(baseKey, currentState.comma, currentState.shift)
    setPressedVisual(baseKey, true)

    const nextState = { keyId: targetKeyId, comma: commaDown.value, shift: shiftDown.value }
    activeBaseStates.set(baseKey, nextState)
    pressedKeyIds.add(targetKeyId)
    startNoteForKey(baseKey, commaDown.value, shiftDown.value)
  }
}

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
.controls {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(140px, 1fr));
  gap: 10px;
  margin: 14px auto 10px;
  max-width: 780px;
  padding: 14px;
  border-radius: 18px;
  background: linear-gradient(145deg, #181818, #0f0f0f);
  border: 1px solid rgba(255,255,255,0.08);
  box-shadow: inset 0 1px 0 rgba(255,255,255,0.08), 0 12px 30px rgba(0,0,0,0.35);
}
.panel-title {
  grid-column: 1 / -1;
  text-align: left;
  font-size: 11px;
  font-weight: 800;
  letter-spacing: 0.24em;
  text-transform: uppercase;
  color: #fbbf24;
  margin-bottom: 2px;
}
.control-card {
  display: flex;
  flex-direction: column;
  gap: 6px;
  text-align: left;
  font-size: 12px;
  color: #f5f5f5;
  padding: 10px;
  border-radius: 12px;
  background: linear-gradient(145deg, #272727, #1a1a1a);
  border: 1px solid rgba(255,255,255,0.06);
  box-shadow: inset 0 1px 0 rgba(255,255,255,0.06);
}
.control-card label {
  font-weight: 700;
  color: #fbbf24;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  font-size: 10px;
}
.control-card input[type="range"] {
  width: 100%;
  appearance: none;
  height: 6px;
  border-radius: 999px;
  background: linear-gradient(90deg, #4b5563, #111827);
  outline: none;
}
.control-card input[type="range"]::-webkit-slider-thumb {
  appearance: none;
  width: 16px;
  height: 16px;
  border-radius: 50%;
  background: radial-gradient(circle at 30% 30%, #fff, #d1d5db 30%, #4b5563 80%);
  border: 2px solid #111827;
  box-shadow: 0 2px 6px rgba(0,0,0,0.4);
}
.control-card select {
  border: 1px solid rgba(255,255,255,0.12);
  border-radius: 999px;
  padding: 6px 8px;
  font-size: 12px;
  background: #111827;
  color: #f5f5f5;
}
.control-card span {
  font-size: 11px;
  color: #cbd5e1;
}
.hint { margin-top: 10px; color: #727272; font-size: 16px; }
</style>
