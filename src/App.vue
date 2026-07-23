<script setup>
import { ref, computed, onUnmounted } from 'vue';
import { questions } from './questions.js';

const TIMER_DURATION = 20;

/* --- État --- */
const screen = ref('home');
const mode = ref(null);
const sessionQ = ref([]);
const current = ref(0);
const selected = ref(null);
const score = ref(0);
const answers = ref([]);
const timerLeft = ref(TIMER_DURATION);
const reviewIdx = ref(null);
const best = ref(getBestScores());
let timerInterval = null;

/* --- Helpers --- */
const shuffle = (arr) => [...arr].sort(() => Math.random() - 0.5);

function getBestScores() {
  try { return JSON.parse(localStorage.getItem('qcm_js_best') || '{}'); }
  catch { return {}; }
}
function saveBestScore(m, s, total) {
  const b = getBestScores();
  const pct = Math.round((s / total) * 100);
  if (!b[m] || pct > b[m].pct) {
    b[m] = { score: s, total, pct };
    localStorage.setItem('qcm_js_best', JSON.stringify(b));
  }
}

function escHtml(s) {
  return String(s).replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;').replace(/"/g, '&quot;');
}
function highlightCode(code) {
  let h = escHtml(code);
  h = h.replace(/(\/\/.*)/g, '<span class="cmt">$1</span>');
  h = h.replace(/\b(let|const|var|function|return|if|else|for|while|do|switch|case|break|continue|typeof|new|of|in|default)\b/g, '<span class="kw">$1</span>');
  h = h.replace(/\b(true|false|null|undefined|NaN)\b/g, '<span class="bool">$1</span>');
  h = h.replace(/'([^']*)'/g, "'<span class=\"str\">$1</span>'");
  h = h.replace(/`([^`]*)`/g, "`<span class=\"str\">$1</span>`");
  h = h.replace(/\b(\d+)\b/g, '<span class="num">$1</span>');
  h = h.replace(/\b(console)\.(log|error|warn)\b/g, '<span class="fn">$1</span>.<span class="fn">$2</span>');
  h = h.replace(/\b([a-zA-Z_]\w*)\s*\(/g, '<span class="fn">$1</span>(');
  return h;
}

/* --- Timer --- */
function startTimer() {
  clearInterval(timerInterval);
  timerLeft.value = TIMER_DURATION;
  timerInterval = setInterval(() => {
    timerLeft.value--;
    if (timerLeft.value <= 0) {
      clearInterval(timerInterval);
      if (selected.value === null) {
        selected.value = -1;
        answers.value.push({ correct: false, timeout: true });
      }
    }
  }, 1000);
}
onUnmounted(() => clearInterval(timerInterval));

/* --- Actions --- */
function startQuiz(all) {
  sessionQ.value = all ? shuffle(questions) : shuffle(questions).slice(0, 15);
  mode.value = all ? 'full' : 'quick';
  screen.value = 'quiz';
  current.value = 0; selected.value = null; score.value = 0; answers.value = [];
  startTimer();
  window.scrollTo(0, 0);
}
function selectAnswer(idx) {
  if (selected.value !== null) return;
  clearInterval(timerInterval);
  selected.value = idx;
  if (idx === sessionQ.value[current.value].answer) score.value++;
  answers.value.push({ correct: idx === sessionQ.value[current.value].answer, selected: idx });
}
function nextQuestion() {
  if (current.value + 1 >= sessionQ.value.length) {
    clearInterval(timerInterval);
    saveBestScore(mode.value, score.value, sessionQ.value.length);
    best.value = getBestScores();
    screen.value = 'result';
  } else {
    current.value++; selected.value = null;
    startTimer();
  }
  window.scrollTo(0, 0);
}
function goHome() { clearInterval(timerInterval); screen.value = 'home'; reviewIdx.value = null; window.scrollTo(0, 0); }
function showReview(i) { reviewIdx.value = i; }
function closeReview() { reviewIdx.value = null; }

/* --- Dérivés --- */
const q = computed(() => sessionQ.value[current.value]);
const total = computed(() => sessionQ.value.length);
const timerClass = computed(() => (timerLeft.value <= 5 ? 'danger' : timerLeft.value <= 10 ? 'warning' : ''));
const isTimeout = computed(() => selected.value === -1);
const homeThemes = computed(() => [...new Set(questions.map((x) => x.icon + ' ' + x.theme))]);
const optClass = (idx) => {
  let cls = 'opt-btn';
  if (selected.value !== null) {
    if (idx === q.value.answer) cls += ' correct';
    else if (idx === selected.value) cls += ' wrong';
  }
  return cls;
};

/* résultat */
const pct = computed(() => Math.round((score.value / total.value) * 100));
const wrongCount = computed(() => answers.value.filter((a) => !a.correct).length);
const isNewBest = computed(() => best.value[mode.value] && best.value[mode.value].pct === pct.value);
const themeStats = computed(() => {
  const t = {};
  sessionQ.value.forEach((qq, i) => {
    if (!t[qq.theme]) t[qq.theme] = { icon: qq.icon, total: 0, correct: 0 };
    t[qq.theme].total++;
    if (answers.value[i]?.correct) t[qq.theme].correct++;
  });
  return Object.entries(t).map(([name, d]) => ({ name, ...d, p: d.total > 0 ? Math.round((d.correct / d.total) * 100) : 0 }));
});
const letter = (i) => String.fromCharCode(65 + i);

/* revue */
const rq = computed(() => (reviewIdx.value === null ? null : sessionQ.value[reviewIdx.value]));
const ra = computed(() => (reviewIdx.value === null ? null : answers.value[reviewIdx.value]));
const reviewOptClass = (idx) => {
  let cls = 'opt-btn';
  if (idx === rq.value.answer) cls += ' correct';
  else if (idx === ra.value?.selected) cls += ' wrong';
  return cls;
};
</script>

<template>
  <!-- ================= HOME ================= -->
  <div v-if="screen === 'home'" class="fade-in" style="max-width:480px;width:100%;margin:auto;text-align:center;padding-top:24px">
    <div style="font-size:48px;margin-bottom:10px">💻</div>
    <h1 style="font-size:22px;margin:0 0 8px;letter-spacing:1px">QCM JavaScript</h1>
    <p style="color:rgba(255,255,255,0.4);font-size:13px;margin-bottom:28px">{{ questions.length }} questions · 7 themes · Timer 20s</p>
    <div class="card">
      <div class="grid2">
        <div v-for="t in homeThemes" :key="t" class="theme-tag">{{ t }}</div>
      </div>
      <div class="btn-group">
        <button class="main-btn" @click="startQuiz(false)">⚡ Mode rapide — 15 questions</button>
        <p v-if="best.quick" style="color:rgba(255,255,255,0.3);font-size:11px;margin:-4px 0 4px">🏆 Meilleur : {{ best.quick.pct }}%</p>
        <button class="sec-btn" @click="startQuiz(true)">📋 Mode complet — {{ questions.length }} questions</button>
        <p v-if="best.full" style="color:rgba(255,255,255,0.3);font-size:11px;margin:-4px 0 0">🏆 Meilleur : {{ best.full.pct }}%</p>
      </div>
    </div>
    <p style="color:rgba(255,255,255,0.15);font-size:11px;margin-top:20px">Fait par JOJO DEV's</p>
  </div>

  <!-- ================= QUIZ ================= -->
  <div v-else-if="screen === 'quiz'" class="fade-in">
    <div class="header-row">
      <span>{{ current + 1 }} / {{ total }}</span>
      <div style="display:flex;gap:12px;align-items:center">
        <span class="timer-text" :class="timerClass">{{ selected === null ? timerLeft + 's' : '' }}</span>
        <span style="font-size:11px;color:rgba(255,255,255,0.3)">{{ mode === 'full' ? 'Complet' : 'Rapide' }}</span>
        <span class="score-display">✓ {{ score }}</span>
      </div>
    </div>
    <div v-if="selected === null" class="timer-bar">
      <div class="timer-fill" :class="timerClass" :style="{ width: (timerLeft / TIMER_DURATION) * 100 + '%' }"></div>
    </div>
    <div class="progress-bar"><div class="progress-fill" :style="{ width: (current / total) * 100 + '%' }"></div></div>
    <div class="card">
      <div class="badge">{{ q.icon }} {{ q.theme }}</div>
      <div class="notion">💡 {{ q.notion }}</div>
      <div v-if="q.code" class="code-block" v-html="highlightCode(q.code)"></div>
      <p class="question-text">{{ q.question }}</p>
      <div class="options">
        <button v-for="(opt, idx) in q.options" :key="idx" :class="optClass(idx)" :disabled="selected !== null" @click="selectAnswer(idx)">
          <span class="letter">{{ letter(idx) }}</span>
          <code style="font-family:'Courier New',monospace;font-size:13px">{{ opt }}</code>
        </button>
      </div>
      <template v-if="selected !== null">
        <div class="explication" :class="!isTimeout && selected === q.answer ? 'ok' : 'ko'">
          {{ isTimeout ? '⏱️ Temps ecoule ! ' : (selected === q.answer ? '✅ Correct ! ' : '❌ Pas tout a fait... ') }}{{ q.explanation }}
        </div>
        <button class="main-btn" @click="nextQuestion">
          {{ current + 1 >= total ? 'Voir le resultat →' : 'Question suivante →' }}
        </button>
      </template>
    </div>
  </div>

  <!-- ================= RESULT ================= -->
  <div v-else-if="screen === 'result'" class="fade-in" style="max-width:520px;width:100%;margin:auto">
    <div class="card" style="text-align:center">
      <div style="font-size:48px;margin-bottom:10px">{{ pct >= 80 ? '🏆' : pct >= 60 ? '💪' : '📚' }}</div>
      <h1 style="font-size:22px;margin:0 0 6px">Resultat</h1>
      <div v-if="isNewBest" class="best-score-badge">🏆 Nouveau record !</div>
      <p style="color:rgba(255,255,255,0.4);margin:0 0 20px;font-size:13px">{{ total }} questions · {{ wrongCount }} erreur{{ wrongCount > 1 ? 's' : '' }}</p>
      <div class="big-score">{{ score }}/{{ total }}</div>
      <p style="color:rgba(255,255,255,0.55);margin-bottom:20px;font-size:14px">
        {{ pct >= 80 ? "Excellent ! T'es pret·e 🎉" : pct >= 60 ? 'Bien joue ! Relis les ratees 🙂' : 'Continue a reviser 💡' }}
      </p>

      <div style="text-align:left;margin-bottom:20px">
        <p style="color:rgba(255,255,255,0.3);font-size:11px;text-transform:uppercase;letter-spacing:1px;margin-bottom:10px">Stats par theme</p>
        <div v-for="s in themeStats" :key="s.name" class="stat-row">
          <span class="stat-label">{{ s.icon }} {{ s.name }}</span>
          <div class="stat-bar-bg"><div class="stat-bar-fill" :class="{ low: s.p < 50 }" :style="{ width: s.p + '%' }"></div></div>
          <span class="stat-pct">{{ s.p }}%</span>
        </div>
      </div>

      <div style="text-align:left">
        <p style="color:rgba(255,255,255,0.3);font-size:11px;text-transform:uppercase;letter-spacing:1px;margin-bottom:8px">Detail (cliquer pour revoir)</p>
      </div>
      <div class="result-list">
        <div v-for="(qq, i) in sessionQ" :key="i" class="result-row" :class="answers[i]?.correct ? 'ok' : 'ko'" @click="showReview(i)">
          <span style="font-size:14px">{{ answers[i]?.correct ? '✅' : (answers[i]?.timeout ? '⏱️' : '❌') }}</span>
          <span style="flex:1">{{ (qq.icon + ' ' + qq.question).substring(0, 50) }}...</span>
          <span style="font-size:10px;color:rgba(255,255,255,0.25)">voir →</span>
        </div>
      </div>

      <div class="btn-group">
        <button class="main-btn" @click="startQuiz(false)">🔀 Nouvelle session rapide</button>
        <button class="sec-btn" @click="startQuiz(true)">📋 Tout revoir ({{ questions.length }} questions)</button>
        <button class="sec-btn" style="font-size:13px;padding:12px" @click="goHome">🏠 Accueil</button>
      </div>
    </div>
  </div>

  <!-- ================= REVUE ================= -->
  <div v-if="reviewIdx !== null" class="review-overlay" @click.self="closeReview">
    <div class="review-card">
      <button class="review-close" @click="closeReview">✕</button>
      <div class="badge">{{ rq.icon }} {{ rq.theme }}</div>
      <div style="margin-top:8px">
        <div class="notion">💡 {{ rq.notion }}</div>
        <div v-if="rq.code" class="code-block" v-html="highlightCode(rq.code)"></div>
        <p class="question-text">{{ rq.question }}</p>
        <div class="options">
          <button v-for="(opt, idx) in rq.options" :key="idx" :class="reviewOptClass(idx)" disabled>
            <span class="letter">{{ letter(idx) }}</span>
            <code style="font-family:'Courier New',monospace;font-size:13px">{{ opt }}</code>
          </button>
        </div>
        <div class="explication" :class="ra?.correct ? 'ok' : 'ko'">
          {{ ra?.timeout ? '⏱️ Temps ecoule. ' : (ra?.correct ? '✅ Correct ! ' : '❌ ') }}{{ rq.explanation }}
        </div>
      </div>
      <div style="display:flex;gap:8px;margin-top:12px">
        <button v-if="reviewIdx > 0" class="sec-btn" style="font-size:13px;padding:10px" @click="showReview(reviewIdx - 1)">← Precedent</button>
        <button v-if="reviewIdx < sessionQ.length - 1" class="sec-btn" style="font-size:13px;padding:10px" @click="showReview(reviewIdx + 1)">Suivant →</button>
      </div>
    </div>
  </div>
</template>
