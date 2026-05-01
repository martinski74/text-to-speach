<script setup>
import { ref, computed, watch, onMounted, onBeforeUnmount } from 'vue';

// ── State ──────────────────────────────────────────────
const isSupported       = ref('speechSynthesis' in window);
const inputText         = ref('');
const allVoices         = ref([]);   // master list — populated on voiceschanged
const selectedVoiceURI  = ref('');   // stable key — voiceURI never changes
const langFilter        = ref('');
const rate              = ref(1.0);
const pitch             = ref(1.0);
const volume            = ref(1.0);
const isSpeaking        = ref(false);
const isPaused          = ref(false);
const errorMessage      = ref('');

// ── Computed ──────────────────────────────────────────

/** Unique sorted language list derived from allVoices. */
const availableLangs = computed(() =>
  [...new Set(allVoices.value.map(v => v.lang))].sort()
);

/** Voices filtered by selected language (or all when no filter). */
const voices = computed(() =>
  langFilter.value
    ? allVoices.value.filter(v => v.lang === langFilter.value)
    : allVoices.value
);

/** Currently selected SpeechSynthesisVoice object (null-safe). */
const selectedVoice = computed(() =>
  allVoices.value.find(v => v.voiceURI === selectedVoiceURI.value) ?? null
);

const statusType = computed(() => {
  if (errorMessage.value) return 'error';
  if (isPaused.value)     return 'paused';
  if (isSpeaking.value)   return 'speaking';
  return 'idle';
});

const statusMessage = computed(() => {
  if (errorMessage.value) return `Грешка: ${errorMessage.value}`;
  if (isPaused.value)     return 'Паузирано — натисни Продължи';
  if (isSpeaking.value)   return 'Говори...';
  return 'Готово за говор';
});

// ── Voice loading ─────────────────────────────────────

/**
 * Populates allVoices from the browser.
 * Called eagerly and again on voiceschanged (Chrome defers loading).
 */
function loadVoices() {
  const loaded = window.speechSynthesis.getVoices();
  if (loaded.length === 0) return;

  allVoices.value = loaded;

  // Keep current selection if still valid after reload
  const stillValid = loaded.some(v => v.voiceURI === selectedVoiceURI.value);
  if (!stillValid) {
    // Prefer Bulgarian; fall back to first available voice
    const bg = loaded.find(v => v.lang.startsWith('bg'));
    selectedVoiceURI.value = (bg ?? loaded[0]).voiceURI;
  }
}

// ── Handlers ──────────────────────────────────────────

/** Starts (or resumes) speech synthesis. */
function handleSpeak() {
  if (isPaused.value) {
    window.speechSynthesis.resume();
    isSpeaking.value = true;
    isPaused.value   = false;
    return;
  }

  errorMessage.value = '';
  window.speechSynthesis.cancel();

  const utterance    = new SpeechSynthesisUtterance(inputText.value);
  utterance.rate     = rate.value;
  utterance.pitch    = pitch.value;
  utterance.volume   = volume.value;

  // Assign the voice object chosen by the user
  if (selectedVoice.value) {
    utterance.voice = selectedVoice.value;
  }

  utterance.onstart = () => { isSpeaking.value = true;  isPaused.value = false; };
  utterance.onend   = () => { isSpeaking.value = false; isPaused.value = false; };
  utterance.onerror = (e) => {
    // 'interrupted' fires on manual cancel — not a real error
    if (e.error !== 'interrupted') errorMessage.value = e.error;
    isSpeaking.value = false;
    isPaused.value   = false;
  };

  window.speechSynthesis.speak(utterance);
}

/** Pauses current speech. */
function handlePause() {
  if (!isSpeaking.value || isPaused.value) return;
  window.speechSynthesis.pause();
  isPaused.value = true;
}

/** Cancels and resets speech. */
function handleStop() {
  window.speechSynthesis.cancel();
  isSpeaking.value = false;
  isPaused.value   = false;
}

// ── Watchers ─────────────────────────────────────────

// When language filter changes, pick first voice in filtered list
watch(langFilter, () => {
  const first = voices.value[0];
  if (first) selectedVoiceURI.value = first.voiceURI;
});

// ── Lifecycle ─────────────────────────────────────────
onMounted(() => {
  if (!isSupported.value) return;
  loadVoices();
  window.speechSynthesis.addEventListener('voiceschanged', loadVoices);
});

onBeforeUnmount(() => {
  window.speechSynthesis.cancel();
  window.speechSynthesis.removeEventListener('voiceschanged', loadVoices);
});
</script>

<template>
  <div v-if="!isSupported" class="no-support">
    ⚠ Браузърът не поддържа Web Speech API.
  </div>

  <template v-else>
    <div class="header">
      <div class="header-tag">web speech api</div>
      <h1>Text<span>.</span>to<span>.</span>Speech</h1>
    </div>

    <!-- Text input card -->
    <div class="card">
      <div class="card-label">Въведи текст</div>
      <textarea
        v-model="inputText"
        placeholder="Напиши нещо тук..."
        :disabled="isSpeaking"
      ></textarea>
      <div class="char-count" :class="{ warn: inputText.length > 900 }">
        {{ inputText.length }} / 1000
      </div>
    </div>

    <!-- Settings card -->
    <div class="card">
      <div class="card-label">Настройки</div>

      <div class="controls-grid">
        <!-- Voice selector -->
        <div class="control-group">
          <label>Глас</label>
          <select v-model="selectedVoiceURI">
            <option
              v-for="voice in voices"
              :key="voice.voiceURI"
              :value="voice.voiceURI"
            >{{ voice.name }} ({{ voice.lang }})</option>
          </select>
        </div>

        <!-- Language filter -->
        <div class="control-group">
          <label>Филтър по език</label>
          <select v-model="langFilter">
            <option value="">Всички езици</option>
            <option v-for="lang in availableLangs" :key="lang" :value="lang">
              {{ lang }}
            </option>
          </select>
        </div>
      </div>

      <!-- Rate slider -->
      <div class="control-group" style="margin-bottom: 16px;">
        <label>Скорост</label>
        <div class="slider-row">
          <input type="range" v-model.number="rate" min="0.5" max="2" step="0.1" />
          <span class="slider-value">{{ rate.toFixed(1) }}x</span>
        </div>
      </div>

      <!-- Pitch slider -->
      <div class="control-group" style="margin-bottom: 16px;">
        <label>Тон</label>
        <div class="slider-row">
          <input type="range" v-model.number="pitch" min="0" max="2" step="0.1" />
          <span class="slider-value">{{ pitch.toFixed(1) }}</span>
        </div>
      </div>

      <!-- Volume slider -->
      <div class="control-group">
        <label>Сила на звука</label>
        <div class="slider-row">
          <input type="range" v-model.number="volume" min="0" max="1" step="0.05" />
          <span class="slider-value">{{ Math.round(volume * 100) }}%</span>
        </div>
      </div>
    </div>

    <!-- Waveform -->
    <div class="waveform" :class="{ active: isSpeaking && !isPaused }">
      <div class="bar" v-for="n in 13" :key="n"></div>
    </div>

    <!-- Action buttons -->
    <div class="btn-row">
      <!-- Speak / Resume -->
      <button
        class="btn btn-primary"
        @click="handleSpeak"
        :disabled="!inputText.trim() || inputText.length > 1000 || (isSpeaking && !isPaused)"
      >
        <svg width="16" height="16" viewBox="0 0 24 24" fill="currentColor">
          <path d="M8 5v14l11-7z"/>
        </svg>
        {{ isPaused ? 'Продължи' : 'Говори' }}
      </button>

      <!-- Pause -->
      <button
        class="btn btn-ghost"
        @click="handlePause"
        :disabled="!isSpeaking || isPaused"
      >
        <svg width="16" height="16" viewBox="0 0 24 24" fill="currentColor">
          <path d="M6 19h4V5H6v14zm8-14v14h4V5h-4z"/>
        </svg>
        Пауза
      </button>

      <!-- Stop -->
      <button
        class="btn btn-danger"
        @click="handleStop"
        :disabled="!isSpeaking && !isPaused"
      >
        <svg width="16" height="16" viewBox="0 0 24 24" fill="currentColor">
          <path d="M6 6h12v12H6z"/>
        </svg>
        Спри
      </button>
    </div>

    <!-- Status bar -->
    <div class="status-bar" :class="'status-' + statusType">
      <div class="status-dot"></div>
      <span>{{ statusMessage }}</span>
    </div>
  </template>
</template>
