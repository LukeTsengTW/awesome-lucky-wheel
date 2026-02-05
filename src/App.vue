<script setup>
import { ref, computed, onMounted, watch, nextTick } from 'vue';
import confetti from 'canvas-confetti';

// --- data and state management ---
const rawInput = ref('1\n2\n3\n4\n5\n6\n7\n8\n9\n10'); // default dataset
const weightedMode = ref(false); // toggle weighted mode
const currentDrawer = ref(''); // current drawer/participant name
const customMessage = ref(''); // custom message to display in result
const showHistory = ref(false);
const historyRecords = ref([]);
const removeWinnerMode = ref(false); // toggle to remove winner from options after spin
const winnerToRemove = ref(null);
const locale = ref('zh');
const showHelp = ref(false);

const GITHUB_URL = 'https://github.com/LukeTsengTW/awesome-lucky-wheel';

const t = computed(() => {
  const dict = {
    zh: {
      title: 'LUCKY\nWHEEL',
      soundOn: '關閉音效',
      soundOff: '開啟音效',
      musicOn: '關閉音樂',
      musicOff: '開啟音樂',
      current: 'CURRENT',
      drawerLabel: '抽獎者',
      drawerPlaceholder: '輸入抽獎者名稱 (可留空)',
      messageLabel: '自訂訊息',
      messagePlaceholder: '結果顯示的自訂訊息 (可留空)',
      inputLabel: '輸入選項 (換行分隔)',
      autoFill: '⚡ 自動填入',
      weightedMode: '權重模式',
      removeWinner: '得獎後移除',
      prefix: '前綴',
      suffix: '後綴',
      prefixPlaceholder: '例: 司儀',
      suffixPlaceholder: '例: 號',
      startNumber: '起始數字',
      endNumber: '結束數字',
      padZero: '數字補 0（例: 01, 02, ...）',
      preview: '預覽',
      generate: '生成選項',
      placeholderWeighted: '格式：選項:權重\n例如：\n大獎:1\n小獎:5\n謝謝參與:10',
      placeholderNormal: '輸入選項...',
      totalOptions: '共',
      optionsSuffix: '個選項',
      weightHint: '（支援 名稱:權重 格式）',
      spinning: 'SPINNING...',
      go: 'GO!',
      history: '查看歷史紀錄',
      needTwo: '至少需要 2 個選項',
      swipe: 'Swipe to spin',
      winner: 'WINNER',
      winnerPicked: '抽中',
      resultHint: 'Press any key or click to continue',
      historyTitle: '歷史紀錄',
      clearHistory: '清空紀錄',
      noHistory: '尚無中獎紀錄',
      confirmClear: '確定要清空嗎？',
      cleared: '清空成功',
      lang: 'EN'
      ,help: '?'
      ,helpTitle: '使用說明'
      ,helpSteps: [
        '輸入選項（每行一個），可選擇權重模式使用「名稱:權重」格式。',
        '按下 GO! 或在轉盤上方快速上滑即可開始抽獎。',
        '抽獎結果會顯示於彈窗中，可點擊或按任意鍵關閉。',
        '可開啟「得獎後移除」以避免重複中獎。',
        '可查看歷史紀錄並清空。'
      ],
      helpGithub: 'GitHub：'
    },
    en: {
      title: 'LUCKY\nWHEEL',
      soundOn: 'Mute SFX',
      soundOff: 'Enable SFX',
      musicOn: 'Mute Music',
      musicOff: 'Enable Music',
      current: 'CURRENT',
      drawerLabel: 'Drawer',
      drawerPlaceholder: 'Enter drawer name (optional)',
      messageLabel: 'Custom message',
      messagePlaceholder: 'Message in result (optional)',
      inputLabel: 'Options (one per line)',
      autoFill: '⚡ Auto fill',
      weightedMode: 'Weighted mode',
      removeWinner: 'Remove winner',
      prefix: 'Prefix',
      suffix: 'Suffix',
      prefixPlaceholder: 'e.g., Host',
      suffixPlaceholder: 'e.g., #',
      startNumber: 'Start',
      endNumber: 'End',
      padZero: 'Pad with 0 (e.g., 01, 02...)',
      preview: 'Preview',
      generate: 'Generate',
      placeholderWeighted: 'Format: option:weight\nExample:\nGrand prize:1\nSmall prize:5\nThanks:10',
      placeholderNormal: 'Enter options...',
      totalOptions: 'Total',
      optionsSuffix: 'options',
      weightHint: '(Supports name:weight)',
      spinning: 'SPINNING...',
      go: 'GO!',
      history: 'History',
      needTwo: 'At least 2 options',
      swipe: 'Swipe to spin',
      winner: 'WINNER',
      winnerPicked: 'picked',
      resultHint: 'Press any key or click to continue',
      historyTitle: 'History',
      clearHistory: 'Clear',
      noHistory: 'No history yet',
      confirmClear: 'Clear all history?',
      cleared: 'Cleared successfully',
      lang: '繁',
      help: '?',
      helpTitle: 'How to use',
      helpSteps: [
        'Enter options (one per line). Use “name:weight” in weighted mode.',
        'Press GO! or swipe up on the wheel to spin.',
        'Result shows in a popup; click or press any key to close.',
        'Enable “Remove winner” to avoid repeats.',
        'View and clear history records.'
      ],
      helpGithub: 'GitHub: '
    }
  };
  return dict[locale.value] || dict.zh;
});

// --- auto-fill settings ---
const showAutoFill = ref(false);
const autoFillPrefix = ref('');
const autoFillSuffix = ref('');
const autoFillStart = ref(1);
const autoFillEnd = ref(10);
const autoFillPadZero = ref(true); // toggle leading zero padding

// Helper function to format number with optional zero padding
const formatAutoFillNumber = (num, maxNum, padEnabled) => {
  if (!padEnabled) return String(num);
  const padLength = maxNum >= 10 ? String(maxNum).length : 2;
  const paddedNum = String(Math.abs(num)).padStart(padLength, '0');
  return num < 0 ? `-${paddedNum}` : paddedNum;
};

// Preview for auto-fill
const autoFillPreviewStart = computed(() => {
  const start = parseInt(autoFillStart.value) || 1;
  const end = parseInt(autoFillEnd.value) || 10;
  const maxNum = Math.max(Math.abs(start), Math.abs(end));
  return `${autoFillPrefix.value}${formatAutoFillNumber(start, maxNum, autoFillPadZero.value)}${autoFillSuffix.value}`;
});

const autoFillPreviewEnd = computed(() => {
  const start = parseInt(autoFillStart.value) || 1;
  const end = parseInt(autoFillEnd.value) || 10;
  const maxNum = Math.max(Math.abs(start), Math.abs(end));
  return `${autoFillPrefix.value}${formatAutoFillNumber(end, maxNum, autoFillPadZero.value)}${autoFillSuffix.value}`;
});

const generateAutoFill = () => {
  const start = parseInt(autoFillStart.value) || 1;
  const end = parseInt(autoFillEnd.value) || 10;
  const prefix = autoFillPrefix.value;
  const suffix = autoFillSuffix.value;
  
  const lines = [];
  const step = start <= end ? 1 : -1;
  
  // Calculate max digits for padding
  const maxNum = Math.max(Math.abs(start), Math.abs(end));
  
  for (let i = start; step > 0 ? i <= end : i >= end; i += step) {
    const numStr = formatAutoFillNumber(i, maxNum, autoFillPadZero.value);
    lines.push(`${prefix}${numStr}${suffix}`);
  }
  
  rawInput.value = lines.join('\n');
  showAutoFill.value = false;
};

const addHistoryRecord = (winnerName) => {
  const timestamp = new Date().toLocaleString('zh-TW', {
    hour12: false
  });
  historyRecords.value.unshift({
    winner: winnerName,
    drawer: currentDrawer.value || '',
    time: timestamp
  });
};

const clearHistory = () => {
  if (confirm(t.value.confirmClear)) {
    historyRecords.value = [];
    alert(t.value.cleared);
  }
};

const removeWinnerFromInput = (winnerName) => {
  if (!winnerName) return;
  const lines = rawInput.value.split('\n');
  const winnerLine = lines.find(line => {
    const trimmed = line.trim();
    if (weightedMode.value && trimmed.includes(':')) {
      const parts = trimmed.split(':');
      const name = parts.slice(0, -1).join(':');
      return name === winnerName || trimmed === winnerName;
    }
    return trimmed === winnerName;
  });
  if (winnerLine !== undefined) {
    const idx = lines.indexOf(winnerLine);
    if (idx !== -1) {
      lines.splice(idx, 1);
      rawInput.value = lines.join('\n');
    }
  }
};

// Parse items - supports "name:weight" format when weighted mode is on
const parsedItems = computed(() => {
  return rawInput.value.split('\n')
    .filter(t => t.trim() !== '')
    .map(t => {
      const trimmed = t.trim();
      if (weightedMode.value && trimmed.includes(':')) {
        const parts = trimmed.split(':');
        const weight = parseFloat(parts[parts.length - 1]);
        const name = parts.slice(0, -1).join(':'); // Handle names with colons
        if (!isNaN(weight) && weight > 0) {
          return { name: name || trimmed, weight };
        }
      }
      return { name: trimmed, weight: 1 };
    });
});

// Simple items array for backward compatibility
const items = computed(() => parsedItems.value.map(p => p.name));

// Total weight for probability calculation
const totalWeight = computed(() => parsedItems.value.reduce((sum, p) => sum + p.weight, 0));

// wheel state
const isSpinning = ref(false);
const showResult = ref(false);
const currentRotation = ref(0); // accumulated rotation angle
const winner = ref(null);
const wheelEl = ref(null); // to bind SVG element
const currentItem = ref(''); // to display the value during spinning
const wheelContainer = ref(null); // for touch events

// --- touch gesture state ---
const touchStartY = ref(0);
const touchStartTime = ref(0);
const isTouching = ref(false);

// visual colors
const colors = [
  '#FF0055', '#00FF99', '#00CCFF', '#FF8800', '#9900FF', 
  '#FF3300', '#33FF00', '#0066FF', '#FF00CC', '#EEEE00'
];

// --- sound effects ---
const soundEnabled = ref(true);
const musicEnabled = ref(true);
const lastTickIndex = ref(-1);
const audioContext = ref(null);
const userInteracted = ref(false);

// --- background music ---
// Audio elements for background music
const bgmIdle = ref(null);
const sfxClick = ref(null);
const sfxWinner = ref(null);
const currentBgm = ref(null);

// Initialize audio elements
const initAudio = () => {
  const baseUrl = import.meta.env.BASE_URL;
  // Background music
  bgmIdle.value = new Audio(`${baseUrl}audio/bgm-idle.mp3`);
  bgmIdle.value.loop = true;
  bgmIdle.value.volume = 0.3;
  bgmIdle.value.preload = 'auto';
  bgmIdle.value.load();
  
  // Sound effects
  sfxClick.value = new Audio(`${baseUrl}audio/sfx-click.mp3`);
  sfxClick.value.volume = 0.5;
  sfxWinner.value = new Audio(`${baseUrl}audio/sfx-winner.mp3`);
  sfxWinner.value.volume = 0.6;
};

// Play specific BGM and stop others
const playBgm = (type) => {
  if (!musicEnabled.value || !userInteracted.value) return;
  
  // Stop current BGM
  stopAllBgm();
  
  let targetBgm = null;
  
  switch (type) {
    case 'idle':
      targetBgm = bgmIdle.value;
      break;
  }
  
  if (targetBgm) {
    targetBgm.currentTime = 0;
    targetBgm.play().catch(() => {
      // Browser may block autoplay, ignore error
    });
    currentBgm.value = targetBgm;
  }
};

const stopAllBgm = () => {
  [bgmIdle.value].forEach(bgm => {
    if (bgm) {
      bgm.pause();
      bgm.currentTime = 0;
    }
  });
  currentBgm.value = null;
};

const toggleMusic = () => {
  musicEnabled.value = !musicEnabled.value;
  if (!musicEnabled.value) {
    stopAllBgm();
  } else if (userInteracted.value && !isSpinning.value && !showResult.value) {
    playBgm('idle');
  }
};

// Play click sound effect
const playClickSound = () => {
  if (!soundEnabled.value || !sfxClick.value) return;
  sfxClick.value.currentTime = 0;
  sfxClick.value.play().catch(() => {});
};

const playWinnerSound = () => {
  if (!soundEnabled.value || !sfxWinner.value) return;
  sfxWinner.value.currentTime = 0;
  sfxWinner.value.play().catch(() => {});
};

const playTickSound = () => {
  if (!soundEnabled.value) return;
  
  // Create AudioContext on first use (browser requirement)
  if (!audioContext.value) {
    audioContext.value = new (window.AudioContext || window.webkitAudioContext)();
  }
  
  const ctx = audioContext.value;
  const oscillator = ctx.createOscillator();
  const gainNode = ctx.createGain();
  
  oscillator.connect(gainNode);
  gainNode.connect(ctx.destination);
  
  // Short "tick" sound - high frequency, quick decay
  oscillator.frequency.value = 800 + Math.random() * 400; // 800-1200 Hz
  oscillator.type = 'sine';
  
  gainNode.gain.setValueAtTime(0.3, ctx.currentTime);
  gainNode.gain.exponentialRampToValueAtTime(0.01, ctx.currentTime + 0.05);
  
  oscillator.start(ctx.currentTime);
  oscillator.stop(ctx.currentTime + 0.05);
};

// --- touch gesture handlers ---
const handleTouchStart = (e) => {
  if (isSpinning.value) return;
  
  const touch = e.touches[0];
  touchStartY.value = touch.clientY;
  touchStartTime.value = Date.now();
  isTouching.value = true;
};

const handleTouchEnd = (e) => {
  if (!isTouching.value || isSpinning.value) return;
  
  const touch = e.changedTouches[0];
  const deltaY = touchStartY.value - touch.clientY; // positive = swipe up
  const deltaTime = Date.now() - touchStartTime.value;
  
  isTouching.value = false;
  
  // Calculate velocity (pixels per millisecond)
  const velocity = Math.abs(deltaY) / deltaTime;
  
  // Trigger spin if swipe is fast enough (velocity > 0.5) and distance > 50px
  if (velocity > 0.5 && Math.abs(deltaY) > 50) {
    startSpin();
  }
};

onMounted(() => {
  const saved = localStorage.getItem('lucky-wheel-data');
  if (saved) rawInput.value = saved;

  const savedHistory = localStorage.getItem('lucky-wheel-history');
  if (savedHistory) {
    try {
      historyRecords.value = JSON.parse(savedHistory);
    } catch {
      historyRecords.value = [];
    }
  }

  const savedWeighted = localStorage.getItem('lucky-wheel-weighted');
  if (savedWeighted !== null) {
    weightedMode.value = savedWeighted === '1';
  }

  const savedRemoveWinner = localStorage.getItem('lucky-wheel-remove-winner');
  if (savedRemoveWinner !== null) {
    removeWinnerMode.value = savedRemoveWinner === '1';
  }

  const savedSound = localStorage.getItem('lucky-wheel-sound');
  if (savedSound !== null) {
    soundEnabled.value = savedSound === '1';
  }

  const savedMusic = localStorage.getItem('lucky-wheel-music');
  if (savedMusic !== null) {
    musicEnabled.value = savedMusic === '1';
  }

  const savedLocale = localStorage.getItem('lucky-wheel-locale');
  if (savedLocale === 'zh' || savedLocale === 'en') {
    locale.value = savedLocale;
  }
  
  // Initialize audio elements
  initAudio();
  
  // any key closes the result
  window.addEventListener('keydown', handleKeydown);
  
  // Start idle BGM on first user interaction (browser autoplay policy)
  const registerUserInteraction = () => {
    userInteracted.value = true;
    if (musicEnabled.value && !isSpinning.value && !showResult.value) {
      playBgm('idle');
    }
    document.removeEventListener('pointerdown', registerUserInteraction);
    document.removeEventListener('keydown', registerUserInteraction);
    document.removeEventListener('touchstart', registerUserInteraction);
  };
  document.addEventListener('pointerdown', registerUserInteraction, { once: true });
  document.addEventListener('keydown', registerUserInteraction, { once: true });
  document.addEventListener('touchstart', registerUserInteraction, { once: true });
});

watch(rawInput, (newVal) => {
  localStorage.setItem('lucky-wheel-data', newVal);
});

watch(historyRecords, (records) => {
  localStorage.setItem('lucky-wheel-history', JSON.stringify(records));
}, { deep: true });

watch(weightedMode, (val) => {
  localStorage.setItem('lucky-wheel-weighted', val ? '1' : '0');
});

watch(removeWinnerMode, (val) => {
  localStorage.setItem('lucky-wheel-remove-winner', val ? '1' : '0');
});

watch(soundEnabled, (val) => {
  localStorage.setItem('lucky-wheel-sound', val ? '1' : '0');
});

watch(musicEnabled, (val) => {
  localStorage.setItem('lucky-wheel-music', val ? '1' : '0');
});

watch(locale, (val) => {
  localStorage.setItem('lucky-wheel-locale', val);
});

const handleKeydown = (e) => {
  if (showResult.value) {
    resetWheel();
    return;
  }
  
  // Space key triggers spin (only when not in input fields)
  if (e.code === 'Space' && !isSpinning.value) {
    const activeEl = document.activeElement;
    const isInputFocused = activeEl?.tagName === 'TEXTAREA' || activeEl?.tagName === 'INPUT';
    
    if (!isInputFocused && items.value.length >= 2) {
      e.preventDefault();
      startSpin();
    }
  }
};

// --- wheel logic ---

// calculate SVG slice path
const getCoordinatesForPercent = (percent) => {
  const x = Math.cos(2 * Math.PI * percent);
  const y = Math.sin(2 * Math.PI * percent);
  return [x, y];
};

const slices = computed(() => {
  let cumulativePercent = 0;
  const count = parsedItems.value.length;
  if (count === 0) return [];

  const total = totalWeight.value;
  
  return parsedItems.value.map((item, index) => {
    const percent = item.weight / total; // weighted percentage
    const [startX, startY] = getCoordinatesForPercent(cumulativePercent);
    cumulativePercent += percent;
    const [endX, endY] = getCoordinatesForPercent(cumulativePercent);
    const largeArcFlag = percent > 0.5 ? 1 : 0;
    const pathData = `M 0 0 L ${startX} ${startY} A 1 1 0 ${largeArcFlag} 1 ${endX} ${endY} L 0 0`;
    
    // calculate text position (slice center angle)
    const midAngle = 2 * Math.PI * (cumulativePercent - percent / 2);
    const textX = Math.cos(midAngle) * 0.7; // text distance from center 70%
    const textY = Math.sin(midAngle) * 0.7;
    
    // text rotation angle
    const rotation = (cumulativePercent - percent / 2) * 360;
    
    // Dynamic font size based on text length and slice count
    const baseSize = 0.12;
    const textLen = item.name.length;
    let fontSize = baseSize;
    
    // Shrink font progressively for longer text (no truncation)
    if (textLen > 20) fontSize = 0.035;
    else if (textLen > 16) fontSize = 0.04;
    else if (textLen > 12) fontSize = 0.05;
    else if (textLen > 8) fontSize = 0.07;
    else if (textLen > 5) fontSize = 0.09;
    
    // Also shrink if too many slices
    if (count > 15) fontSize *= 0.7;
    else if (count > 10) fontSize *= 0.85;
    
    // Shrink more for very thin slices
    if (percent < 0.05) fontSize *= 0.7;
    else if (percent < 0.1) fontSize *= 0.85;
    
    // No character limit - show full text with smaller font
    const maxChars = 999;

    return {
      path: pathData,
      color: colors[index % colors.length],
      text: item.name,
      weight: item.weight,
      percent: percent,
      textPos: { x: textX, y: textY },
      rotation: rotation,
      fontSize: fontSize,
      maxChars: maxChars
    };
  });
});

// --- animation flow ---

const startSpin = async () => {
  if (isSpinning.value || items.value.length < 2) return;

  // Play click sound
  userInteracted.value = true;
  playClickSound();

  isSpinning.value = true;
  showResult.value = false;
  winner.value = null;
  currentItem.value = '...';
  lastTickIndex.value = -1; // Reset tick tracking

  await nextTick();

  // determine the result first (weighted random selection)
  let winnerIndex = 0;
  const total = totalWeight.value;
  let random = Math.random() * total;
  
  for (let i = 0; i < parsedItems.value.length; i++) {
    random -= parsedItems.value[i].weight;
    if (random <= 0) {
      winnerIndex = i;
      break;
    }
  }
  
  // Calculate target slice angle based on weights
  let targetSliceAngle = 0;
  for (let i = 0; i < winnerIndex; i++) {
    targetSliceAngle += (parsedItems.value[i].weight / total) * 360;
  }
  // Add half of winner's slice to point to center
  targetSliceAngle += (parsedItems.value[winnerIndex].weight / total) * 360 / 2;
  
  // calculate target angle
  // we want the pointer (usually on the right at 0° or on top at 270°) to point to the winner
  // SVG 0° is on the right (3 o'clock). We place the pointer on the right.
  // the goal is to rotate the slice center to 0°.
  
  const currentAngleMod = currentRotation.value % 360;
  
  // at least spin 5 rounds (1800 degrees) -> random offset to simulate inertia
  const extraSpins = 360 * 5; 
  
  // calculate the final total rotation needed (negative means counter-clockwise, which matches visual expectation)
  // formula: target = current - (current remainder) - (target slice angle) - (extra spins)
  // this ensures smooth continuation from the last angle
  const targetRotation = currentRotation.value - (currentAngleMod + targetSliceAngle + extraSpins);
  
  // if calculation results in positive rotation, force negative to keep direction consistent
  const finalRotation = targetRotation > currentRotation.value 
    ? targetRotation - 360 * 6 
    : targetRotation;

  currentRotation.value = finalRotation;
  
  trackRotation();

  // set delay to show result (match CSS transition 5s)
  setTimeout(() => {
    winner.value = items.value[winnerIndex];
    addHistoryRecord(winner.value);
    triggerConfetti();
    playWinnerSound();
    showResult.value = true;
    currentItem.value = winner.value;
    winnerToRemove.value = winner.value;
  }, 5000);
};

// --- track rotation angle ---
const trackRotation = () => {
  if (!isSpinning.value) return;

  if (wheelEl.value) {
    // Get the actual style of the current DOM
    const style = window.getComputedStyle(wheelEl.value);
    const matrix = style.transform;
    
    // Parse matrix to calculate angle (CSS transform is a matrix)
    // matrix(a, b, c, d, tx, ty)
    if (matrix !== 'none') {
      const values = matrix.split('(')[1].split(')')[0].split(',');
      const a = values[0];
      const b = values[1];

      let angle = Math.round(Math.atan2(b, a) * (180 / Math.PI));
      
      // Handle negative angles and coordinate system adjustment
      // need to convert CSS rotation angle (-180 ~ 180) to (0 ~ 360)
      // and correspond to our slice logic
      if (angle < 0) angle += 360;
      
      // Because the pointer is on the right side (0 degrees), and the wheel spins counterclockwise (CSS rotate decreases)
      // When the wheel spins, the slice angle under the pointer is actually "reversed"
      // Some math correction is needed here:
      // When the wheel spins 10 degrees, the pointer actually points to 350 degrees (if arranged counterclockwise)
      const effectiveAngle = (360 - angle) % 360;
      
      // Find index based on weighted slices
      const count = parsedItems.value.length;
      const total = totalWeight.value;
      let cumulativeAngle = 0;
      let index = 0;
      
      for (let i = 0; i < count; i++) {
        const sliceAngle = (parsedItems.value[i].weight / total) * 360;
        if (effectiveAngle < cumulativeAngle + sliceAngle) {
          index = i;
          break;
        }
        cumulativeAngle += sliceAngle;
      }
      
      // Prevent array out-of-bounds
      const safeIndex = index >= count ? count - 1 : index;
      
      if (items.value[safeIndex]) {
        currentItem.value = items.value[safeIndex];
        
        // Play tick sound when slice changes
        if (safeIndex !== lastTickIndex.value) {
          lastTickIndex.value = safeIndex;
          playTickSound();
        }
      }
    }
  }

  // Continue to next frame
  requestAnimationFrame(trackRotation);
};

const resetWheel = () => {
  if (removeWinnerMode.value && winnerToRemove.value) {
    removeWinnerFromInput(winnerToRemove.value);
    winnerToRemove.value = null;
  }
  showResult.value = false;
  isSpinning.value = false;
  // note: do not reset rotation to 0 to keep continuity for next spin
};

// celebration effect
const triggerConfetti = () => {
  const duration = 3000;
  const end = Date.now() + duration;

  (function frame() {
    confetti({
      particleCount: 5,
      angle: 60,
      spread: 55,
      origin: { x: 0 },
      colors: colors
    });
    confetti({
      particleCount: 5,
      angle: 120,
      spread: 55,
      origin: { x: 1 },
      colors: colors
    });

    if (Date.now() < end) {
      requestAnimationFrame(frame);
    }
  }());
};
</script>

<template>
  <div class="app-container" :class="{ 'mode-spin': isSpinning }">
    
    <div class="control-panel">
      <div class="header-row">
        <h1 class="title">{{ t.title.split('\n')[0] }}<br>{{ t.title.split('\n')[1] }}</h1>
        <div class="audio-toggles">
          <button 
            class="sound-toggle" 
            @click="soundEnabled = !soundEnabled"
            :title="soundEnabled ? t.soundOn : t.soundOff"
          >
            <span v-if="soundEnabled">🔊</span>
            <span v-else>🔇</span>
          </button>
          <button 
            class="music-toggle" 
            @click="toggleMusic"
            :title="musicEnabled ? t.musicOn : t.musicOff"
          >
            <span v-if="musicEnabled">🎵</span>
            <span v-else>🎶</span>
          </button>
          <button
            class="lang-toggle"
            @click="locale = locale === 'zh' ? 'en' : 'zh'"
            :title="t.lang"
          >
            {{ t.lang }}
          </button>
          <button
            class="help-toggle"
            @click="showHelp = true"
            :title="t.helpTitle"
          >
            {{ t.help }}
          </button>
        </div>
      </div>
      
      <div class="current-indicator" v-if="isSpinning">
        <div class="label">{{ t.current }}</div>
        <div class="value">{{ currentItem }}</div>
      </div>

      <div class="input-group" v-else>
        <!-- Drawer/Participant Input -->
        <div class="drawer-input">
          <label>{{ t.drawerLabel }}</label>
          <input 
            type="text" 
            v-model="currentDrawer" 
            :placeholder="t.drawerPlaceholder" 
          />
        </div>
        
        <!-- Custom Message Input -->
        <div class="drawer-input">
          <label>{{ t.messageLabel }}</label>
          <input 
            type="text" 
            v-model="customMessage" 
            :placeholder="t.messagePlaceholder" 
          />
        </div>
        
        <div class="input-header">
          <label>{{ t.inputLabel }}</label>
          <div class="header-actions">
            <button class="auto-fill-btn" @click="showAutoFill = !showAutoFill" :title="t.autoFill">
              {{ t.autoFill }}
            </button>
            <label class="weight-toggle">
              <input type="checkbox" v-model="weightedMode" />
              <span>{{ t.weightedMode }}</span>
            </label>
            <label class="weight-toggle">
              <input type="checkbox" v-model="removeWinnerMode" />
              <span>{{ t.removeWinner }}</span>
            </label>
          </div>
        </div>
        
        <!-- Auto-fill Panel -->
        <div class="auto-fill-panel" v-if="showAutoFill">
          <div class="auto-fill-row">
            <div class="auto-fill-field">
              <label>{{ t.prefix }}</label>
              <input type="text" v-model="autoFillPrefix" :placeholder="t.prefixPlaceholder" />
            </div>
            <div class="auto-fill-field">
              <label>{{ t.suffix }}</label>
              <input type="text" v-model="autoFillSuffix" :placeholder="t.suffixPlaceholder" />
            </div>
          </div>
          <div class="auto-fill-row">
            <div class="auto-fill-field">
              <label>{{ t.startNumber }}</label>
              <input type="number" v-model="autoFillStart" />
            </div>
            <div class="auto-fill-field">
              <label>{{ t.endNumber }}</label>
              <input type="number" v-model="autoFillEnd" />
            </div>
          </div>
          <label class="auto-fill-checkbox">
            <input type="checkbox" v-model="autoFillPadZero" />
            <span>{{ t.padZero }}</span>
          </label>
          <div class="auto-fill-preview">
            {{ t.preview }}: {{ autoFillPreviewStart }} ~ {{ autoFillPreviewEnd }}
          </div>
          <button class="auto-fill-generate" @click="generateAutoFill">{{ t.generate }}</button>
        </div>
        <textarea 
          v-model="rawInput" 
          :placeholder="weightedMode ? t.placeholderWeighted : t.placeholderNormal" 
          :disabled="isSpinning"
        ></textarea>
        <div class="stats">
          {{ t.totalOptions }} {{ items.length }} {{ t.optionsSuffix }}
          <span v-if="weightedMode" class="weight-hint">{{ t.weightHint }}</span>
        </div>
      </div>

      <button class="spin-btn" @click="startSpin" :disabled="isSpinning || items.length < 2">
        {{ isSpinning ? t.spinning : t.go }}
      </button>
      <button class="history-btn" @click="showHistory = true" :disabled="isSpinning">
        {{ t.history }}
      </button>
      <div class="hint" v-if="items.length < 2">{{ t.needTwo }}</div>
    </div>

    <div class="wheel-stage">
      <div 
        ref="wheelContainer"
        class="wheel-container"
        @touchstart.prevent="handleTouchStart"
        @touchend.prevent="handleTouchEnd"
      >
        <div class="pointer"></div>
        <div class="swipe-hint" v-if="!isSpinning && items.length >= 2">
          <span>↑</span>
          <span>{{ t.swipe }}</span>
        </div>
        
        <svg 
          ref="wheelEl"
          viewBox="-1 -1 2 2" 
          class="wheel-svg" 
          :style="{ transform: `rotate(${currentRotation}deg)` }"
        >
          <g v-for="(slice, i) in slices" :key="i">
             <path :d="slice.path" :fill="slice.color" stroke="#1a1a1a" stroke-width="0.01" />
             <text 
              :x="slice.textPos.x" 
              :y="slice.textPos.y" 
              fill="#fff" 
              :font-size="slice.fontSize" 
              font-weight="bold"
              text-anchor="middle" 
              dominant-baseline="middle"
              :transform="`rotate(${slice.rotation}, ${slice.textPos.x}, ${slice.textPos.y})`"
            >
              {{ slice.text.length > slice.maxChars ? slice.text.substring(0, slice.maxChars) + '..' : slice.text }}
            </text>
          </g>
        </svg>
      </div>
    </div>

    <Transition name="fade">
      <div v-if="showResult" class="result-overlay" @click="resetWheel">
        <div class="result-box">
          <div class="result-label">{{ t.winner }}</div>
          <div class="result-drawer" v-if="currentDrawer">🎯 {{ currentDrawer }} {{ t.winnerPicked }}</div>
          <div class="result-value">{{ winner }}</div>
          <div class="result-message" v-if="customMessage">{{ customMessage }}</div>
          <div class="result-hint">{{ t.resultHint }}</div>
        </div>
      </div>
    </Transition>

    <Transition name="fade">
      <div v-if="showHistory" class="history-overlay" @click="showHistory = false">
        <div class="history-box" @click.stop>
          <div class="history-header">
            <div class="history-title">{{ t.historyTitle }}</div>
            <div class="history-header-actions">
              <button class="history-clear" @click="clearHistory" v-if="historyRecords.length > 0">{{ t.clearHistory }}</button>
              <button class="history-close" @click="showHistory = false">✕</button>
            </div>
          </div>
          <div v-if="historyRecords.length === 0" class="history-empty">
            {{ t.noHistory }}
          </div>
          <ul v-else class="history-list">
            <li v-for="(record, idx) in historyRecords" :key="idx" class="history-item">
              <div class="history-winner">
                {{ record.winner }}
                <span v-if="record.drawer">（{{ record.drawer }}）</span>
              </div>
              <div class="history-time">{{ record.time }}</div>
            </li>
          </ul>
        </div>
      </div>
    </Transition>

    <Transition name="fade">
      <div v-if="showHelp" class="help-overlay" @click="showHelp = false">
        <div class="help-box" @click.stop>
          <div class="help-header">
            <div class="help-title">{{ t.helpTitle }}</div>
            <button class="help-close" @click="showHelp = false">✕</button>
          </div>
          <ol class="help-list">
            <li v-for="(step, idx) in t.helpSteps" :key="idx">{{ step }}</li>
          </ol>
          <div class="help-github">
            {{ t.helpGithub }}
            <a :href="GITHUB_URL" target="_blank" rel="noopener noreferrer">{{ GITHUB_URL }}</a>
          </div>
        </div>
      </div>
    </Transition>

    <div class="footer">© 2026 LukeTseng. All Rights Reserved.</div>
  </div>
</template>

<style lang="scss">

:root {
  --bg-color: #1a1a2e;
  --panel-bg: #16213e;
  --accent: #e94560;
  --text: #ffffff;
}

body {
  margin: 0;
  background-color: var(--bg-color);
  color: var(--text);
  font-family: 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
  overflow: hidden; /* prevent scrollbars during rotation */
}

.app-container {
  display: flex;
  height: 100vh;
  width: 100vw;
  transition: all 0.8s cubic-bezier(0.2, 0.8, 0.2, 1);
  
  &.mode-spin {
    .control-panel {
      // opacity: 0;
      // transform: translateX(-50px);
      pointer-events: none;
    }
    .wheel-stage {
      width: 100%;
      transform: scale(1.1);
    }
    // darken background for focus
    &::after {
      content: '';
      position: absolute;
      inset: 0;
      background: rgba(0,0,0,0.5);
      z-index: 5;
      pointer-events: none;
    }
  }
}

/* left control panel */
.control-panel {
  width: 340px;
  margin-left: -10px;
  background: var(--panel-bg);
  padding: 2rem;
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
  box-shadow: 5px 0 20px rgba(0,0,0,0.3);
  z-index: 10;
  transition: all 0.5s ease;

  .header-row {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 1rem;
  }

  .audio-toggles {
    display: flex;
    flex-direction: row;
    flex-wrap: wrap;
    gap: 0.4rem;
    justify-content: flex-end;
  }

  .sound-toggle,
  .music-toggle,
  .lang-toggle,
  .help-toggle {
    background: rgba(255, 255, 255, 0.1);
    border: 1px solid #444;
    border-radius: 8px;
    padding: 0.4rem 0.5rem;
    font-size: 1.2rem;
    cursor: pointer;
    transition: all 0.2s;
    line-height: 1;
    min-width: 40px;
    min-height: 36px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    
    &:hover {
      background: rgba(255, 255, 255, 0.2);
      border-color: var(--accent);
    }
  }

  .title {
    font-size: 3rem;
    line-height: 1;
    margin: 0;
    white-space: nowrap;
    background: linear-gradient(45deg, #ff00cc, #3333ff);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    font-weight: 900;
  }

  .current-indicator {
    flex: 1; /* Occupies the original input space */
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    background: rgba(255, 255, 255, 0.05);
    border-radius: 12px;
    border: 2px solid var(--accent);
    margin-bottom: 1rem;
    animation: pulse 0.5s infinite alternate;

    .label {
      font-size: 1.2rem;
      color: #888;
      text-transform: uppercase;
      letter-spacing: 2px;
      margin-bottom: 0.5rem;
    }

    .value {
      font-size: 3rem;
      font-weight: 900;
      color: #fff;
      text-align: center;
      word-break: break-all;
      line-height: 1.1;
      text-shadow: 0 0 20px var(--accent);
    }
  }

  @keyframes pulse {
    from { box-shadow: 0 0 10px rgba(233, 69, 96, 0.2); }
    to { box-shadow: 0 0 30px rgba(233, 69, 96, 0.6); }
  }

  .drawer-input {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    margin-bottom: 0.8rem;
    
    label {
      font-size: 0.9rem;
      color: #888;
      white-space: nowrap;
    }
    
    input {
      flex: 1;
      background: rgba(0,0,0,0.3);
      border: 1px solid #444;
      border-radius: 6px;
      color: #fff;
      padding: 0.5rem 0.8rem;
      font-size: 0.9rem;
      
      &:focus {
        outline: none;
        border-color: var(--accent);
      }
      
      &::placeholder {
        color: #555;
      }
    }
  }

  .input-group {
    flex: 1;
    display: flex;
    flex-direction: column;
    
    .input-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      flex-wrap: wrap;
      gap: 0.5rem;
      margin-bottom: 0.5rem;
    }
    
    label {
      font-size: 0.9rem;
      color: #888;
    }
    
    .header-actions {
      display: flex;
      align-items: center;
      gap: 0.8rem;
    }
    
    .auto-fill-btn {
      background: rgba(255, 255, 255, 0.1);
      border: 1px solid #444;
      color: #aaa;
      padding: 0.3rem 0.6rem;
      border-radius: 4px;
      font-size: 0.75rem;
      cursor: pointer;
      transition: all 0.2s;
      
      &:hover {
        background: rgba(233, 69, 96, 0.2);
        border-color: var(--accent);
        color: var(--accent);
      }
    }
    
    .weight-toggle {
      display: flex;
      align-items: center;
      gap: 0.4rem;
      cursor: pointer;
      font-size: 0.8rem;
      
      input[type="checkbox"] {
        width: 16px;
        height: 16px;
        accent-color: var(--accent);
        cursor: pointer;
      }
      
      span {
        color: #aaa;
      }
      
      &:hover span {
        color: var(--accent);
      }
    }
    
    .auto-fill-panel {
      background: rgba(0, 0, 0, 0.4);
      border: 1px solid #444;
      border-radius: 8px;
      padding: 0.8rem;
      margin-bottom: 0.5rem;
      
      .auto-fill-row {
        display: flex;
        gap: 0.5rem;
        margin-bottom: 0.5rem;
      }
      
      .auto-fill-field {
        flex: 1;
        
        label {
          display: block;
          font-size: 0.75rem;
          color: #666;
          margin-bottom: 0.2rem;
        }
        
        input {
          width: 100%;
          background: rgba(0, 0, 0, 0.3);
          border: 1px solid #555;
          border-radius: 4px;
          color: #fff;
          padding: 0.4rem;
          font-size: 0.85rem;
          box-sizing: border-box;
          
          &:focus {
            outline: none;
            border-color: var(--accent);
          }
        }
      }
      
      .auto-fill-preview {
        font-size: 0.75rem;
        color: #888;
        margin-bottom: 0.5rem;
        padding: 0.3rem;
        background: rgba(255, 255, 255, 0.05);
        border-radius: 4px;
        text-align: center;
      }
      
      .auto-fill-checkbox {
        display: flex;
        align-items: center;
        gap: 0.4rem;
        margin-bottom: 0.5rem;
        cursor: pointer;
        font-size: 0.8rem;
        
        input[type="checkbox"] {
          width: 16px;
          height: 16px;
          accent-color: var(--accent);
          cursor: pointer;
        }
        
        span {
          color: #aaa;
        }
        
        &:hover span {
          color: var(--accent);
        }
      }
      
      .auto-fill-generate {
        width: 100%;
        background: var(--accent);
        border: none;
        color: white;
        padding: 0.5rem;
        border-radius: 4px;
        cursor: pointer;
        font-weight: bold;
        transition: all 0.2s;
        
        &:hover {
          filter: brightness(1.1);
        }
      }
    }
    
    textarea {
      flex: 1;
      background: rgba(0,0,0,0.3);
      border: 1px solid #444;
      border-radius: 8px;
      color: #fff;
      padding: 1rem;
      resize: none;
      font-size: 1rem;
      &:focus {
        outline: none;
        border-color: var(--accent);
      }
    }
    .stats {
      text-align: right;
      font-size: 0.8rem;
      color: #666;
      margin-top: 0.5rem;
      
      .weight-hint {
        color: var(--accent);
        opacity: 0.8;
      }
    }
  }

  .spin-btn {
    padding: 1.2rem;
    font-size: 1.5rem;
    font-weight: bold;
    background: var(--accent);
    color: white;
    border: none;
    border-radius: 50px;
    cursor: pointer;
    box-shadow: 0 0 20px rgba(233, 69, 96, 0.5);
    transition: transform 0.2s, box-shadow 0.2s;
    
    &:hover:not(:disabled) {
      transform: scale(1.05);
      box-shadow: 0 0 30px rgba(233, 69, 96, 0.8);
    }
    
    &:disabled {
      background: #555;
      cursor: not-allowed;
      box-shadow: none;
    }
  }

  .history-btn {
    padding: 0.8rem;
    font-size: 1rem;
    font-weight: 600;
    background: rgba(255, 255, 255, 0.08);
    color: #fff;
    border: 1px solid #444;
    border-radius: 12px;
    cursor: pointer;
    transition: all 0.2s;

    &:hover:not(:disabled) {
      border-color: var(--accent);
      color: var(--accent);
      background: rgba(233, 69, 96, 0.1);
    }

    &:disabled {
      opacity: 0.6;
      cursor: not-allowed;
    }
  }
}

/* right wheel stage */
.wheel-stage {
  flex: 1;
  display: flex;
  justify-content: center;
  align-items: center;
  position: relative;
  transition: all 0.8s ease;
  z-index: 6; /* ensure above overlay */
}

.wheel-container {
  width: min(80vh, 80vw);
  height: min(80vh, 80vw);
  position: relative;
  /* wheel outer glow */
  border-radius: 50%;
  box-shadow: 0 0 50px rgba(255, 255, 255, 0.1);
  /* enable touch */
  touch-action: none;
  cursor: grab;
  
  &:active {
    cursor: grabbing;
  }
}

.swipe-hint {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  display: none;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
  color: rgba(255, 255, 255, 0.6);
  font-size: 1rem;
  pointer-events: none;
  z-index: 15;
  animation: bounceHint 1.5s ease-in-out infinite;
  
  span:first-child {
    font-size: 2rem;
  }
  
  @media (max-width: 768px) {
    display: flex;
  }
}

@keyframes bounceHint {
  0%, 100% { transform: translate(-50%, -50%); opacity: 0.6; }
  50% { transform: translate(-50%, -60%); opacity: 1; }
}

.wheel-svg {
  width: 100%;
  height: 100%;
  border-radius: 50%;
  /* control wheel rotation animation */
  transition: transform 5s cubic-bezier(0.1, 0.7, 0.1, 1);
  will-change: transform;
  
  /* idle state: slow rotation */
  .app-container:not(.mode-spin) & {
    animation: idleSpin 60s linear infinite;
  }
}

@keyframes idleSpin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

.pointer {
  position: absolute;
  top: 50%;
  right: -20px;
  transform: translateY(-50%);
  width: 0;
  height: 0;
  border-top: 20px solid transparent;
  border-bottom: 20px solid transparent;
  border-right: 40px solid #fff;
  z-index: 20;
  filter: drop-shadow(-2px 0 5px rgba(0,0,0,0.5));
}

/* result overlay */
.result-overlay {
  position: fixed;
  inset: 0;
  z-index: 100;
  display: flex;
  justify-content: center;
  align-items: center;
  background: rgba(0,0,0,0.8);
  backdrop-filter: blur(5px);
}

.result-box {
  background: white;
  color: #1a1a2e;
  padding: 3rem 5rem;
  border-radius: 20px;
  text-align: center;
  box-shadow: 0 0 50px rgba(255, 255, 255, 0.5);
  animation: popIn 0.5s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  
  .result-label {
    font-size: 1.5rem;
    color: #666;
    letter-spacing: 2px;
  }
  
  .result-drawer {
    font-size: 1.2rem;
    color: #888;
    margin-top: 0.5rem;
  }
  
  .result-value {
    font-size: 4rem;
    font-weight: 900;
    margin: 1rem 0;
    background: linear-gradient(45deg, #ff00cc, #3333ff);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
  }
  
  .result-message {
    font-size: 1.1rem;
    color: #666;
    margin-bottom: 0.5rem;
    padding: 0.5rem 1rem;
    background: rgba(233, 69, 96, 0.1);
    border-radius: 8px;
    border-left: 3px solid var(--accent);
  }

  .result-hint {
    margin-top: 2rem;
    font-size: 0.9rem;
    color: #999;
    animation: blink 1.5s infinite;
  }
}

.history-overlay {
  position: fixed;
  inset: 0;
  z-index: 110;
  display: flex;
  justify-content: center;
  align-items: center;
  background: rgba(0,0,0,0.7);
  backdrop-filter: blur(4px);
}

.history-box {
  width: min(520px, 90vw);
  max-height: 70vh;
  background: #fff;
  color: #1a1a2e;
  border-radius: 16px;
  padding: 1.5rem;
  box-shadow: 0 0 30px rgba(0,0,0,0.4);
  display: flex;
  flex-direction: column;
  gap: 1rem;
  overflow: hidden;
}

.history-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.history-title {
  font-size: 1.5rem;
  font-weight: 900;
  letter-spacing: 1px;
}

.history-header-actions {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.history-clear {
  background: #e94560;
  border: none;
  border-radius: 6px;
  padding: 0.4rem 0.8rem;
  font-size: 0.85rem;
  color: #fff;
  cursor: pointer;
  transition: all 0.2s;
  
  &:hover {
    background: #d13550;
  }
}

.history-close {
  background: transparent;
  border: none;
  font-size: 1.2rem;
  cursor: pointer;
  color: #666;
}

.history-empty {
  text-align: center;
  color: #777;
  padding: 2rem 0;
}

.history-list {
  list-style: none;
  padding: 0;
  margin: 0;
  overflow: auto;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.history-item {
  padding: 0.75rem 1rem;
  border-radius: 10px;
  background: #f5f5f7;
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 1rem;
}

.history-winner {
  font-weight: 700;
}

.history-time {
  font-size: 0.85rem;
  color: #666;
  white-space: nowrap;
}

.help-overlay {
  position: fixed;
  inset: 0;
  z-index: 120;
  display: flex;
  justify-content: center;
  align-items: center;
  background: rgba(0,0,0,0.7);
  backdrop-filter: blur(4px);
}

.help-box {
  width: min(560px, 92vw);
  max-height: 75vh;
  background: #fff;
  color: #1a1a2e;
  border-radius: 16px;
  padding: 1.5rem;
  box-shadow: 0 0 30px rgba(0,0,0,0.4);
  display: flex;
  flex-direction: column;
  gap: 1rem;
  overflow: auto;
}

.help-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.help-title {
  font-size: 1.5rem;
  font-weight: 900;
  letter-spacing: 1px;
}

.help-close {
  background: transparent;
  border: none;
  font-size: 1.2rem;
  cursor: pointer;
  color: #666;
}

.help-list {
  margin: 0;
  padding-left: 1.2rem;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  color: #333;
}

.help-github {
  font-size: 0.9rem;
  color: #444;
  word-break: break-all;
}

.help-github a {
  color: #2c6bed;
  text-decoration: none;
}

.help-github a:hover {
  text-decoration: underline;
}

.footer {
  position: fixed;
  left: 0;
  right: 0;
  bottom: 10px;
  text-align: center;
  font-size: 0.8rem;
  color: rgba(255, 255, 255, 0.6);
  z-index: 2;
  pointer-events: none;
}

@keyframes popIn {
  from { transform: scale(0.5); opacity: 0; }
  to { transform: scale(1); opacity: 1; }
}

@keyframes blink {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}

/* Vue transition */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

/* RWD: mobile setting */
@media (max-width: 768px) {
  .app-container {
    flex-direction: column-reverse;
  }
  
  .control-panel {
    width: 100%;
    height: 40%;
    padding: 1rem;
    box-sizing: border-box;
    z-index: 20;

    .header-row {
      align-items: flex-start;
    }
    
    .title { font-size: 2rem; display: inline-block; margin-right: 0.5rem;}
    .spin-btn { padding: 0.8rem; font-size: 1.2rem; }
  }

  .wheel-stage {
    flex: 1;
    width: 100%;
  }
  
  .wheel-container {
    width: min(90vw, 50vh);
    height: min(90vw, 50vh);
  }
}
</style>
