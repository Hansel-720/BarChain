<template>
  <div class="top-console">
    <div class="controls-row">
      <button v-if="!isPlaying" @click="playEngine" class="btn btn-play">▶ Play</button>
      <button v-else @click="stopEngine" class="btn btn-stop">⏹ Stop</button>
      
      <label>BPM: <input type="number" v-model.number="global.bpm" @change="saveData" style="width:60px;"></label>
      <label>基準: 
        <select v-model.number="global.noteValue" @change="saveData">
          <option :value="2">二分</option>
          <option :value="4">四分</option>
          <option :value="8">八分</option>
        </select>
      </label>
      <label><input type="checkbox" v-model="global.subAccent" @change="saveData"> 次重/反拍</label>
      <label>預備拍: 
        <select v-model.number="global.countIn" @change="saveData">
          <option :value="0">無</option>
          <option :value="1">1 小節</option>
          <option :value="2">2 小節</option>
        </select>
      </label>
      <label>漸進訓練: 
        <select v-model.number="global.trainer" @change="saveData">
          <option :value="-5">-5 BPM</option><option :value="-4">-4 BPM</option><option :value="-3">-3 BPM</option><option :value="-2">-2 BPM</option><option :value="-1">-1 BPM</option>
          <option :value="0">Off</option>
          <option :value="1">+1 BPM</option><option :value="2">+2 BPM</option><option :value="3">+3 BPM</option><option :value="4">+4 BPM</option><option :value="5">+5 BPM</option>
        </select>
      </label>
    </div>
    
    <div class="status-bar">
      <span v-if="displayState.isCountIn" class="status-countin">COUNT-IN: {{ Math.abs(displayState.bar) }}</span>
      <span v-else>Now: {{ displayState.secIdx + 1 }} | BARS: {{ displayState.bar + 1 }} / {{ sections[displayState.secIdx]?.bars || 0 }}</span>
    </div>
    <div class="led-container">
      <div v-for="(_, i) in currentLedCount" :key="i" :class="['led', getLedClass(i)]"></div>
    </div>
  </div>

  <div class="card-list">
    <div v-for="(sec, idx) in sections" :key="sec.id" :id="'card-'+idx" :class="['section-card', { 'active-card': !displayState.isCountIn && displayState.secIdx === idx }]">
      <div class="card-number">{{ idx + 1 }}</div>
      <div class="card-inputs">
        <label>小節 <input type="number" min="1" v-model.number="sec.bars" @change="updateSec" style="width:40px"></label>
        <button class="ts-btn" @click="openPicker(idx)">拍號: {{ sec.beatsPerBar }} / {{ sec.beatUnit }}</button>
        <label>重音 
          <select v-model="sec.mode" @change="updateSec">
            <option value="absolute">Absolute</option>
            <option value="force">Force</option>
          </select>
        </label>
        <select v-if="sec.mode === 'force'" v-model.number="sec.forceLoop" @change="updateSec" style="width:55px;">
          <option :value="2">2拍</option>
          <option :value="3">3拍</option>
          <option :value="4">4拍</option>
        </select>
      </div>
      <div class="card-actions">
        <button :class="['mute-btn', { 'is-muted': sec.isMuted }]" @click="toggleMute(idx)">{{ sec.isMuted ? '🔇' : '🔊' }}</button>
        <button @click="duplicateSection(idx)">複製</button>
        <button @click="deleteSection(idx)" style="color:#ff5252">刪除</button>
      </div>
    </div>
  </div>
  
  <div style="max-width: 600px; margin: 0 auto; padding: 0 20px;">
    <button @click="addSection" :disabled="32 - getTotalBars() <= 0" class="add-btn">
      + 新增段落 (剩餘 {{ 32 - getTotalBars() }} 小節)
    </button>
  </div>

  <div v-if="picker.show" class="modal-overlay" @click="closePicker">
    <div class="picker-container" @click.stopPropagation>
      <div class="helper-text">滑動選擇拍號，點擊外部儲存</div>
      <div class="picker-wheels">
        <div class="picker-col" id="col-num" @scroll="updatePickerStyle('col-num')">
          <div class="picker-item"></div>
          <div v-for="i in 16" :key="i" class="picker-item" :data-val="i">{{ i }}</div>
          <div class="picker-item"></div>
        </div>
        <div class="picker-col" id="col-den" @scroll="updatePickerStyle('col-den')">
          <div class="picker-item"></div>
          <div v-for="v in [2,4,8,16]" :key="v" class="picker-item" :data-val="v">{{ v }}</div>
          <div class="picker-item"></div>
        </div>
        <div class="picker-highlight"></div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { reactive, ref, computed, onMounted, nextTick } from 'vue';

interface Section {
  id: string;
  bars: number;
  beatsPerBar: number;
  beatUnit: number;
  isMuted: boolean;
  mode: 'absolute' | 'force';
  forceLoop: number;
}

const global = reactive({ bpm: 120, noteValue: 4, subAccent: true, volume: 0.8, countIn: 1, trainer: 0 });
const sections = reactive<Section[]>([{ id: 's1', bars: 4, beatsPerBar: 4, beatUnit: 4, isMuted: false, mode: 'absolute', forceLoop: 4 }]);

const isPlaying = ref(false);
const displayState = reactive({ secIdx: 0, bar: 0, beat: 0, beatIndex: 0, isCountIn: false, isFirst: false, isAccent: false, isDown: false, isUp: false });

const picker = reactive({ show: false, idx: -1 });

function getTotalBars() { return sections.reduce((sum, s) => sum + s.bars, 0); }

function enforceLimit() {
  let total = getTotalBars();
  if (total > 32) {
    let excess = total - 32;
    for (let i = sections.length - 1; i >= 0 && excess > 0; i--) {
      let reduce = Math.min(sections[i].bars, excess);
      sections[i].bars -= reduce;
      excess -= reduce;
    }
    const filtered = sections.filter(s => s.bars > 0);
    sections.length = 0;
    sections.push(...filtered);
  }
}

function loadData() {
  const saved = localStorage.getItem('barChainData');
  if (saved) {
    const parsed = JSON.parse(saved);
    if (parsed.global) Object.assign(global, parsed.global);
    if (parsed.sections) { sections.length = 0; sections.push(...parsed.sections); }
  }
}
function saveData() { localStorage.setItem('barChainData', JSON.stringify({ global, sections })); }

function addSection() {
  if (isPlaying.value) stopEngine();
  sections.push({ id: 's'+Date.now(), bars: 1, beatsPerBar: 4, beatUnit: 4, isMuted: false, mode: 'absolute', forceLoop: 4 });
  enforceLimit(); saveData();
}
function duplicateSection(idx: number) {
  let target = JSON.parse(JSON.stringify(sections[idx]));
  target.id = 's' + Date.now();
  sections.splice(idx + 1, 0, target);
  enforceLimit(); saveData();
}
function deleteSection(idx: number) {
  if (isPlaying.value) stopEngine();
  sections.splice(idx, 1);
  saveData();
}
function toggleMute(idx: number) { sections[idx].isMuted = !sections[idx].isMuted; saveData(); }
function updateSec() { if (isPlaying.value) stopEngine(); enforceLimit(); saveData(); }

// --- 3D Picker 邏輯 ---
function openPicker(idx: number) {
  if (isPlaying.value) stopEngine();
  picker.idx = idx;
  picker.show = true;
  nextTick(() => {
    let sec = sections[idx];
    let numEl = document.getElementById('col-num');
    let denEl = document.getElementById('col-den');
    if(numEl && denEl) {
      numEl.scrollTop = (sec.beatsPerBar - 1) * 50;
      denEl.scrollTop = [2,4,8,16].indexOf(sec.beatUnit) * 50;
      updatePickerStyle('col-num'); updatePickerStyle('col-den');
    }
  });
}
function closePicker() {
  if (picker.idx === -1) return;
  let numItem = document.querySelector('#col-num .picker-item.selected');
  let denItem = document.querySelector('#col-den .picker-item.selected');
  if (numItem && numItem.getAttribute('data-val')) sections[picker.idx].beatsPerBar = parseInt(numItem.getAttribute('data-val')!);
  if (denItem && denItem.getAttribute('data-val')) sections[picker.idx].beatUnit = parseInt(denItem.getAttribute('data-val')!);
  picker.show = false; picker.idx = -1;
  saveData();
}
function updatePickerStyle(colId: string) {
  let col = document.getElementById(colId);
  if (!col) return;
  let activeIndex = Math.round(col.scrollTop / 50) + 1;
  col.querySelectorAll('.picker-item').forEach((item, idx) => {
    if(idx === activeIndex) item.classList.add('selected');
    else item.classList.remove('selected');
  });
}

// --- Web Audio 引擎核心 ---
let audioCtx: AudioContext | null = null;
let worker: Worker | null = null;
let nextNoteTime = 0;
let cSecIdx = 0, cBar = 0, cTick = 0, engineCountIn = false;
let uiQueue: any[] = [];

const workerCode = `let t = null; self.onmessage = (e) => { if(e.data==='start') t=setInterval(()=>postMessage('k'),25); else clearInterval(t); };`;

function playEngine() {
  if (isPlaying.value || sections.length === 0) return;
  audioCtx = new (window.AudioContext || (window as any).webkitAudioContext)();
  worker = new Worker(URL.createObjectURL(new Blob([workerCode], { type: 'application/javascript' })));
  
  isPlaying.value = true;
  nextNoteTime = audioCtx.currentTime + 0.1;
  engineCountIn = global.countIn > 0;
  cSecIdx = 0; cBar = engineCountIn ? -global.countIn : 0; cTick = 0;
  uiQueue = [];
  
  worker.onmessage = () => {
    while (audioCtx && nextNoteTime < audioCtx.currentTime + 0.1) {
      let sec = engineCountIn ? sections[0] : sections[cSecIdx];
      let globalRef = global.noteValue;
      let tickUnit = Math.max(globalRef, sec.beatUnit);
      let ticksPerBeat = tickUnit / sec.beatUnit;
      let isDownbeat = (cTick % ticksPerBeat === 0);
      let beatIndex = Math.floor(cTick / ticksPerBeat);
      
      let isFirstOfSec = (cBar === 0 && cTick === 0) && !engineCountIn;
      let isAccent = false;
      
      if (isDownbeat) {
        if (engineCountIn) { if (beatIndex === 0) isAccent = true; }
        else {
          if (sec.mode === 'absolute' && beatIndex === 0) isAccent = true;
          if (sec.mode === 'force' && ((Math.abs(cBar) * sec.beatsPerBar) + beatIndex) % sec.forceLoop === 0) isAccent = true;
        }
      }
      
      let freq = 440;
      if (engineCountIn) freq = isAccent ? 880 : 440;
      else {
        if (isFirstOfSec) freq = 1760;
        else if (isAccent) freq = 880;
        else if (isDownbeat) freq = 440;
        else freq = 330;
      }
      
      if (!sec.isMuted || engineCountIn) {
        if (isFirstOfSec || isAccent || global.subAccent || engineCountIn) {
          let osc = audioCtx.createOscillator();
          let gain = audioCtx.createGain();
          osc.frequency.value = freq;
          gain.gain.setValueAtTime((freq===440||freq===330)?0.1:0.4, nextNoteTime);
          gain.gain.exponentialRampToValueAtTime(0.001, nextNoteTime + 0.05);
          osc.connect(gain); gain.connect(audioCtx.destination);
          osc.start(nextNoteTime); osc.stop(nextNoteTime + 0.05);
        }
      }
      
      uiQueue.push({ time: nextNoteTime, secIdx: cSecIdx, bar: cBar, beatIndex, isCountIn: engineCountIn, isFirst: isFirstOfSec, isAccent, isDown: isDownbeat, isUp: !isDownbeat });
      
      nextNoteTime += (60.0 / global.bpm) * (globalRef / tickUnit);
      cTick++;
      if (cTick >= sec.beatsPerBar * ticksPerBeat) {
        cTick = 0; cBar++;
        if (engineCountIn && cBar >= 0) { engineCountIn = false; }
        else if (!engineCountIn && cBar >= sec.bars) {
          cBar = 0; cSecIdx++;
          if (cSecIdx >= sections.length) {
            cSecIdx = 0;
            if (global.trainer !== 0) { global.bpm += global.trainer; saveData(); }
          }
        }
      }
    }
  };
  worker.postMessage('start');
  requestAnimationFrame(updateUI);
}

function stopEngine() {
  isPlaying.value = false;
  if (worker) { worker.postMessage('stop'); worker = null; }
  if (audioCtx) { audioCtx.close(); audioCtx = null; }
}

function updateUI() {
  if (!isPlaying.value) return;
  let currTime = audioCtx ? audioCtx.currentTime : 0;
  let ev: any = null;
  while (uiQueue.length > 0 && uiQueue[0].time <= currTime + 0.05) ev = uiQueue.shift();
  if (ev) {
    Object.assign(displayState, ev);
    if (!ev.isCountIn) {
      let card = document.getElementById('card-' + ev.secIdx);
      if (card) card.scrollIntoView({ behavior: 'smooth', block: 'center' });
    }
  }
  requestAnimationFrame(updateUI);
}

const currentLedCount = computed(() => {
  if (displayState.isCountIn) return sections[0]?.beatsPerBar || 0;
  return sections[displayState.secIdx]?.beatsPerBar || 0;
});

function getLedClass(i: number) {
  if (displayState.beatIndex !== i) return '';
  if (displayState.isCountIn || displayState.isFirst) return 'active-first';
  if (displayState.isAccent) return 'active-accent';
  if (displayState.isDown) return 'active-sub';
  return 'active-upbeat';
}

onMounted(() => { loadData(); });
</script>

<style scoped>
.top-console { background: #1e1e1e; padding: 15px 20px; position: sticky; top: 0; z-index: 100; box-shadow: 0 4px 10px rgba(0,0,0,0.5); border-radius: 8px; margin: 10px; }
.controls-row { display: flex; gap: 10px; align-items: center; flex-wrap: wrap; margin-bottom: 10px; font-size: 14px; }
.controls-row input, .controls-row select { background: #333; color: white; border: 1px solid #555; padding: 6px; border-radius: 4px; }
.btn { padding: 10px 20px; font-size: 16px; border: none; border-radius: 6px; cursor: pointer; font-weight: bold; }
.btn-play { background: #4CAF50; color: white; }
.btn-stop { background: #f44336; color: white; }
.status-bar { font-size: 18px; margin-bottom: 10px; color: #aaa; font-weight: bold; }
.status-countin { color: #00bcd4; }
.led-container { display: flex; gap: 8px; height: 30px; align-items: center; }
.led { width: 18px; height: 18px; border-radius: 50%; background: #333; }
.led.active-first { background: #00bcd4; box-shadow: 0 0 10px #00bcd4; } 
.led.active-accent { background: #ff5252; box-shadow: 0 0 10px #ff5252; } 
.led.active-sub { background: #4caf50; box-shadow: 0 0 10px #4caf50; }    
.led.active-upbeat { background: #ff9800; box-shadow: 0 0 10px #ff9800; } 
.card-list { padding: 20px; max-width: 600px; margin: 0 auto; }
.section-card { background: #222; border-left: 5px solid #555; padding: 15px; margin-bottom: 15px; border-radius: 8px; display: flex; align-items: center; justify-content: space-between; gap: 15px; }
.section-card.active-card { border-left-color: #4CAF50; background: #2a3b2a; }
.card-number { font-size: 32px; font-weight: bold; color: #555; width: 40px; text-align: center; }
.card-inputs { flex-grow: 1; display: flex; gap: 8px; align-items: center; flex-wrap: wrap; }
.card-inputs input, .card-inputs select { background: #111; color: white; border: 1px solid #444; padding: 6px; border-radius: 4px; }
.ts-btn { background: #333; color: #00bcd4; border: 1px solid #00bcd4; padding: 6px 12px; border-radius: 4px; cursor: pointer; font-weight: bold; }
.card-actions { display: flex; gap: 5px; flex-shrink: 0; flex-wrap: nowrap; }
.card-actions button { background: #444; color: white; border: none; padding: 6px 10px; border-radius: 4px; cursor: pointer; }
.card-actions .mute-btn.is-muted { background: #ff9800; color: #111; }
.add-btn { width: 100%; padding: 15px; font-size: 16px; background: #333; color: white; border: 2px dashed #555; border-radius: 8px; cursor: pointer; }
.modal-overlay { position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background: rgba(0,0,0,0.8); display: flex; justify-content: center; align-items: center; z-index: 999; }
.picker-container { background: #222; border-radius: 12px; padding: 20px; width: 200px; text-align: center; position: relative; }
.picker-wheels { display: flex; justify-content: space-between; height: 150px; position: relative; overflow: hidden; }
.picker-col { flex: 1; height: 100%; overflow-y: scroll; scroll-snap-type: y mandatory; }
.picker-item { height: 50px; display: flex; justify-content: center; align-items: center; font-size: 24px; scroll-snap-align: center; color: #aaa; }
.picker-item.selected { color: #fff; font-weight: bold; font-size: 28px; }
.picker-highlight { position: absolute; top: 50px; left: 0; width: 100%; height: 50px; border-top: 1px solid #4CAF50; border-bottom: 1px solid #4CAF50; pointer-events: none; }
.helper-text { font-size: 12px; color: #888; margin-bottom: 10px; }
</style>
