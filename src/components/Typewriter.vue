<script setup>
import { ref, computed, onMounted, watch } from 'vue';

const rawText = ref('');
const inputRef = ref(null);
const dictionary = ref(new Set());
const isLoadingDict = ref(true);
const isLocked = ref(false);

// Speech Queue
const speechQueue = ref([]);
const isProcessingQueue = ref(false);
const currentUtterance = ref(null); // Keep ref to prevent GC

// Dictionary Logic
onMounted(async () => {
  try {
    // Full French Dictionary (Taknok) - ~300k words
    // Handles accents and proper French vocabulary better than frequency list
    const response = await fetch('https://raw.githubusercontent.com/Taknok/French-Wordlist/master/francais.txt');
    if (!response.ok) throw new Error('Failed to load dict');
    const text = await response.text();
    // File is newline separated
    const words = text.split(/\r?\n/);
    dictionary.value = new Set(words.map(w => w.trim().toLowerCase()));
    
    // Extensions for very common kid words if missing
    const extensions = [
        'papa', 'maman', 'super', 'école', 'jouet', 'doudou',
        'lundi', 'mardi', 'mercredi', 'jeudi', 'vendredi', 'samedi', 'dimanche'
    ];
    extensions.forEach(w => dictionary.value.add(w.toLowerCase()));
    
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
    
    // 1. Direct match
    if (dictionary.value.has(lower)) return true;
    
    // 2. Single letters: 'a', 'y' valid. Others valid if just typing? 
    // Let's be strict: single letters like 'b' are errors if isolated.
    if (lower.length === 1) return ['a', 'y'].includes(lower);
    
    // 3. Elisions (l'école, d'accord)
    // Common prefixes: l, d, j, n, s, m, t, c, qu
    const elisionMatch = lower.match(/^([ldjnsmtc]|qu)['’](.+)/);
    if (elisionMatch) {
        const suffix = elisionMatch[2];
        return dictionary.value.has(suffix);
    }
    
    // Simple plural 's' check if not found (basic heuristic)
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
    
    u.onend = () => {
        isProcessingQueue.value = false;
        processQueue();
    };
    u.onerror = (e) => {
        console.error("TTS Error", e);
        isProcessingQueue.value = false;
        processQueue();
    };
    
    currentUtterance.value = u; // Prevent GC
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

// Watcher
let lastText = '';
watch(rawText, (newVal) => {
  if (newVal.length > lastText.length && newVal.startsWith(lastText)) {
      const added = newVal.slice(lastText.length);
      for (const char of added) {
          speakEventually(char);
      }
  }
  lastText = newVal;
});

const handleEnter = () => { /* Handle enter if needed */ };

const playAll = () => {
    speakNow(rawText.value);
    ensureFocus();
};

const restart = () => {
    rawText.value = '';
    lastText = '';
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

const ensureFocus = () => {
    inputRef.value?.focus();
};
</script>

<template>
  <div class="typewriter-container" @click="ensureFocus">
    <!-- Paper Sheet -->
    <div class="paper-sheet" :class="{ 'fullscreen-sheet': isLocked }">
        <div class="content">
            <span 
                v-for="(t, i) in tokens" 
                :key="i"
                :class="{ 'error-word': t.error, 'word': t.type === 'word' }"
            >{{ t.text }}</span>
            <span class="cursor">|</span>
        </div>
    </div>

    <!-- Controls -->
    <div class="controls">
        <button @click.stop="playAll" title="Lire tout">
             🔊 Lire
        </button>
        <button @click.stop="restart" class="btn-restart" title="Recommencer">
             🗑️ Effacer
        </button>
        <button @click.stop="toggleLock" :class="{ locked: isLocked }" title="Verrouiller">
             {{ isLocked ? '🔒' : '🔓' }}
        </button>
        <div v-if="isLoadingDict" class="loader">Dict...</div>
    </div>

    <textarea 
        ref="inputRef"
        v-model="rawText"
        @input="onInput"
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
    
    /* Seyes Background */
    background-image: 
        linear-gradient(90deg, transparent 59px, var(--color-margin) 59px, var(--color-margin) 61px, transparent 61px),
        linear-gradient(90deg, rgba(0,0,0,0.05) 1px, transparent 1px),
        linear-gradient(0deg, var(--color-base) 2px, transparent 2px),
        linear-gradient(0deg, transparent 24px, var(--color-inter) 24px, var(--color-inter) 25px, transparent 25px),
        linear-gradient(0deg, transparent 49px, var(--color-inter) 49px, var(--color-inter) 50px, transparent 50px),
        linear-gradient(0deg, transparent 74px, var(--color-inter) 74px, var(--color-inter) 75px, transparent 75px);
        
    background-size: 
        100% 100%,
        25px 100%,
        100% var(--line-height), 
        100% var(--line-height), 
        100% var(--line-height), 
        100% var(--line-height);
        
    font-size: 80px; 
    line-height: var(--line-height); /* 100px */
    padding-left: 80px; 
    padding-top: 0;
}

.content {
    white-space: pre-wrap; 
    word-break: break-word;
    min-height: 100%;
}

.error-word {
    color: #e74c3c; 
    text-decoration: underline wavy #e74c3c;
    text-decoration-thickness: 3px;
}

.cursor {
    animation: blink 1s infinite;
    color: #2c3e50;
    font-weight: 100;
}

.controls {
    margin-top: 15px;
    display: flex;
    gap: 15px;
    z-index: 10;
    align-items: center;
}

.hidden-input {
    position: absolute;
    opacity: 0; 
    top: -1000px;
}

.fullscreen-sheet {
    max-width: 95%;
    height: 80vh;
}
.loader { font-size: 0.8rem; color: #666; }
@keyframes blink { 50% { opacity: 0; } }
</style>
