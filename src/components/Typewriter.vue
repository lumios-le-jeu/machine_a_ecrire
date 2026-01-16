<script setup>
import { ref, computed, onMounted, watch } from 'vue';

const rawText = ref('');
const inputRef = ref(null);
const dictionary = ref(new Set());
let cachedDictionaryArray = []; // optimization for linear search
const isLoadingDict = ref(true);
const isLocked = ref(false);

// Speech Queue
const speechQueue = ref([]);
const isProcessingQueue = ref(false);
const currentUtterance = ref(null); // Keep ref to prevent GC

// Levenshtein Distance
const levenshtein = (a, b) => {
  const matrix = [];
  for (let i = 0; i <= b.length; i++) matrix[i] = [i];
  for (let j = 0; j <= a.length; j++) matrix[0][j] = j;

  for (let i = 1; i <= b.length; i++) {
    for (let j = 1; j <= a.length; j++) {
      if (b.charAt(i - 1) === a.charAt(j - 1)) {
        matrix[i][j] = matrix[i - 1][j - 1];
      } else {
        matrix[i][j] = Math.min(
          matrix[i - 1][j - 1] + 1, // substitution
          Math.min(matrix[i][j - 1] + 1, matrix[i - 1][j] + 1) // insertion/deletion
        );
      }
    }
  }
  return matrix[b.length][a.length];
};

// Autocorrect Logic
const findBestMatch = (word) => {
    if (!word || word.length < 2) return null;
    const lower = word.toLowerCase();
    let bestWord = null;
    let minDist = Infinity;

    // Optimization: Filter candidates first
    // 1. Same first letter (very high probability correct)
    // 2. Length within +/- 2 chars
    const candidates = cachedDictionaryArray.filter(w => 
        Math.abs(w.length - lower.length) <= 2 && 
        w[0] === lower[0]
    );

    // If no candidates (e.g. wrong first letter), try looser filter
    const searchSet = candidates.length > 0 ? candidates : cachedDictionaryArray;

    const maxSearch = 10000; // Limit search to avoid freeze
    let count = 0;

    for (const w of searchSet) {
        if (count++ > maxSearch) break; 
        const dist = levenshtein(lower, w);
        if (dist === 0) return w; // Exact match
        if (dist < minDist) {
            minDist = dist;
            bestWord = w;
        }
    }

    return (minDist <= 3) ? bestWord : null; // Only correct if reasonably close
};

// Dictionary Logic
onMounted(async () => {
  try {
    // Full French Dictionary (Taknok) - ~300k words
    const response = await fetch('https://raw.githubusercontent.com/Taknok/French-Wordlist/master/francais.txt');
    if (!response.ok) throw new Error('Failed to load dict');
    const text = await response.text();
    const words = text.split(/\r?\n/);
    dictionary.value = new Set(words.map(w => w.trim().toLowerCase()));
    
    // Extensions
    const extensions = [
        'papa', 'maman', 'super', 'école', 'jouet', 'doudou',
        'lundi', 'mardi', 'mercredi', 'jeudi', 'vendredi', 'samedi', 'dimanche'
    ];
    extensions.forEach(w => dictionary.value.add(w.toLowerCase()));
    
    // Cache array for Auto-correct
    cachedDictionaryArray = Array.from(dictionary.value);

    isLoadingDict.value = false;
  } catch (e) {
    console.error("Dict error", e);
    isLoadingDict.value = false;
  }
  
  ensureFocus();
  document.addEventListener('fullscreenchange', () => { isLocked.value = !!document.fullscreenElement; });
});

// Input Filtering
const onInput = (e) => {
    const val = e.target.value;
    const clean = val.replace(/[^a-zA-ZàâäéèêëîïôöùûüçÀÂÄÉÈÊËÎÏÔÖÙÛÜÇ \.\!\?\n'-]/g, '');
    if (clean !== val) rawText.value = clean;
};

// Word Validation Logic
const validateWord = (word) => {
    if (!word || isLoadingDict.value) return true;
    const lower = word.toLowerCase();
    if (dictionary.value.has(lower)) return true;
    if (lower.length === 1) return ['a', 'y'].includes(lower);
    
    const elisionMatch = lower.match(/^([ldjnsmtc]|qu)['’](.+)/);
    if (elisionMatch) {
         return dictionary.value.has(elisionMatch[2]);
    }
    
    if (lower.endsWith('s') && dictionary.value.has(lower.slice(0, -1))) return true;
    return false;
};

// Computed Tokens
const tokens = computed(() => {
  const regex = /([a-zA-ZàâäéèêëîïôöùûüçÀÂÄÉÈÊËÎÏÔÖÙÛÜÇ'-]+)/g;
  const parts = rawText.value.split(regex);
  return parts.map(part => {
    if (!part) return null;
    const isWord = part.match(regex);
    if (isWord) {
        const isValid = validateWord(part);
        return { text: part, type: 'word', error: !isValid && !isLoadingDict.value };
    }
    return { text: part, type: 'other', error: false };
  }).filter(p => p !== null);
});

// Sound Logic
const processQueue = () => {
    if (isProcessingQueue.value || speechQueue.value.length === 0) return;
    isProcessingQueue.value = true;
    const nextText = speechQueue.value.shift();
    
    const u = new SpeechSynthesisUtterance(nextText);
    u.lang = 'fr-FR';
    u.rate = 1.0;
    
    u.onend = () => { isProcessingQueue.value = false; processQueue(); };
    u.onerror = (e) => { console.error("TTS Error", e); isProcessingQueue.value = false; processQueue(); };
    
    currentUtterance.value = u; 
    window.speechSynthesis.speak(u);
};

const speakEventually = (text) => {
    if (!text || text.trim().length === 0) return;
    speechQueue.value.push(text);
    processQueue();
};

const speakNow = (text) => {
    window.speechSynthesis.cancel();
    speechQueue.value = [];
    isProcessingQueue.value = false;
    if (!text) return;
    const u = new SpeechSynthesisUtterance(text);
    u.lang = 'fr-FR';
    currentUtterance.value = u;
    window.speechSynthesis.speak(u);
};

watch(rawText, (newVal, oldVal) => {
  if (newVal.length > oldVal.length && newVal.startsWith(oldVal)) {
      const added = newVal.slice(oldVal.length);
      for (const char of added) speakEventually(char);
  }
});

// Key Handlers
const handleEnter = () => {
    playAll();
};

const handleTab = (e) => {
    const cursor = e.target.selectionStart;
    const textBefore = rawText.value.slice(0, cursor);
    
    // Find last word token before cursor
    const match = textBefore.match(/([a-zA-ZàâäéèêëîïôöùûüçÀÂÄÉÈÊËÎÏÔÖÙÛÜÇ'-]+)$/);
    if (match) {
        const word = match[0];
        const isValid = validateWord(word);
        
        if (!isValid) {
            const correction = findBestMatch(word);
            if (correction) {
                // Determine capitalization
                let replacement = correction;
                if (word[0] === word[0].toUpperCase()) {
                    replacement = correction.charAt(0).toUpperCase() + correction.slice(1);
                }
                
                // Replace in text
                const newText = rawText.value.slice(0, match.index) + replacement + rawText.value.slice(cursor);
                rawText.value = newText;
                
                // Correction Sound? Optional.
            }
        }
    }
};

const playAll = () => {
    speakNow(rawText.value);
    ensureFocus();
};

const restart = () => {
    rawText.value = '';
    window.speechSynthesis.cancel();
    ensureFocus();
};

const toggleLock = () => {
    if (!document.fullscreenElement) {
        document.documentElement.requestFullscreen().catch(() => {});
    } else {
        document.exitFullscreen().catch(() => {});
    }
};

const ensureFocus = () => { inputRef.value?.focus(); };
</script>

<template>
  <div class="typewriter-container" @click="ensureFocus">
    <div class="paper-sheet" :class="{ 'fullscreen-sheet': isLocked }">
        <div class="content">
            <span v-for="(t, i) in tokens" :key="i" :class="{ 'error-word': t.error, 'word': t.type === 'word' }">{{ t.text }}</span>
            <span class="cursor">|</span>
        </div>
    </div>

    <div class="controls">
        <button @click.stop="playAll" title="Lire tout">🔊 Lire</button>
        <button @click.stop="restart" class="btn-restart" title="Recommencer">🗑️ Effacer</button>
        <button @click.stop="toggleLock" :class="{ locked: isLocked }" title="Verrouiller">{{ isLocked ? '🔒' : '🔓' }}</button>
        <div v-if="isLoadingDict" class="loader">Dict...</div>
    </div>

    <textarea 
        ref="inputRef"
        v-model="rawText"
        @input="onInput"
        @keydown.enter="handleEnter"
        @keydown.tab.prevent="handleTab"
        class="hidden-input"
        spellcheck="false"
    ></textarea>
  </div>
</template>

<style scoped>
.typewriter-container {
    display: flex;
    flex-direction: column;
    align-items: center;
    width: 100%;
    height: 100%;
    position: relative;
    background: #e0e5ecc4; 
    padding: 20px;
}

.paper-sheet {
    background-color: var(--paper-color);
    width: 100%;
    max-width: 90%;
    height: 70vh;
    box-shadow: 0 10px 30px rgba(0,0,0,0.1);
    border-radius: 4px;
    overflow-y: auto;
    cursor: text;
    position: relative;
    background-image: 
        linear-gradient(90deg, transparent 59px, var(--color-margin) 59px, var(--color-margin) 61px, transparent 61px),
        linear-gradient(90deg, rgba(0,0,0,0.05) 1px, transparent 1px),
        linear-gradient(0deg, var(--color-base) 2px, transparent 2px),
        linear-gradient(0deg, transparent 24px, var(--color-inter) 24px, var(--color-inter) 25px, transparent 25px),
        linear-gradient(0deg, transparent 49px, var(--color-inter) 49px, var(--color-inter) 50px, transparent 50px),
        linear-gradient(0deg, transparent 74px, var(--color-inter) 74px, var(--color-inter) 75px, transparent 75px);
    background-size: 100% 100%, 25px 100%, 100% var(--line-height), 100% var(--line-height), 100% var(--line-height), 100% var(--line-height);
    font-size: 80px; 
    line-height: var(--line-height); 
    padding-left: 80px; 
    padding-top: 0;
}

.content { white-space: pre-wrap; word-break: break-word; min-height: 100%; }
.error-word { color: #e74c3c; text-decoration: underline wavy #e74c3c; text-decoration-thickness: 3px; }
.cursor { animation: blink 1s infinite; color: #2c3e50; font-weight: 100; }

.controls {
    position: absolute;
    top: 20px;
    right: 20px;
    display: flex;
    flex-direction: column;
    gap: 15px;
    z-index: 100;
    align-items: flex-end; 
}

@media (max-width: 600px) {
    .controls { top: 10px; right: 10px; gap: 10px; }
    .controls button { padding: 8px 16px; font-size: 0.9rem; }
}

.hidden-input { position: absolute; opacity: 0; top: -1000px; }
.fullscreen-sheet { max-width: 95%; height: 80vh; }
.loader { font-size: 0.8rem; color: #666; }
@keyframes blink { 50% { opacity: 0; } }
</style>
