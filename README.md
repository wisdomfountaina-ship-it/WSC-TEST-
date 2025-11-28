<!doctype html>
<html lang="ar">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>WCST — Web Version</title>
<style>
  body{font-family: "Segoe UI", Tahoma, Arial; direction: rtl; text-align: center; margin:0; padding:20px; background:#f7f7f7}
  #container{max-width:980px;margin:0 auto;background:#fff;padding:18px;border-radius:10px;box-shadow:0 6px 18px rgba(0,0,0,0.08)}
  h1{margin-top:0}
  .row{display:flex;justify-content:center;gap:18px;flex-wrap:wrap;margin:10px 0}
  .card{width:150px;height:220px;border-radius:8px;background:#222;color:#fff;display:flex;align-items:center;justify-content:center;font-size:28px;cursor:pointer;user-select:none;position:relative}
  .card.small{width:100px;height:140px;font-size:20px}
  .top-info{display:flex; justify-content:space-between; align-items:center; margin-bottom:12px}
  button{padding:10px 14px;border-radius:6px;border:0;background:#2b79ff;color:white;cursor:pointer}
  #feedback{height:28px;margin-top:10px}
  .meta{font-size:13px;color:#444}
  #results{margin-top:12px; text-align:left}
  .hidden{display:none}
  #downloadBtn{background:#16a085}
</style>
</head>
<body>
<div id="container">
  <h1>اختبار تصنيف بطاقات Wisconsin — نسخة ويب</h1>
  <div class="top-info">
    <div>
      <label>اسم المشارك: <input id="username" placeholder="ادخل اسمك" /></label>
    </div>
    <div class="meta">
      <div>التجربة مسجلة: <span id="trialCounter">0</span> تجربة</div>
      <div>قاعدة نشطة: <span id="activeRuleText">---</span></div>
    </div>
  </div>

  <div style="margin-bottom:8px" class="meta">التعليمات: اضغط على البطاقة المرجعية التي تعتقد أنها تطابق البطاقة العلوية (وفق اللون أو الشكل أو الرقم). ستحصل/ين على ردّ (صح/خطأ). بعد 5 إجابات صحيحة تتغيّر القاعدة تلقائيًا.</div>

  <!-- عرض البطاقات المرجعية -->
  <div class="row" id="refRow"></div>

  <!-- البطاقة من السحب -->
  <div style="margin-top:18px">
    <div style="font-size:14px;color:#666;margin-bottom:6px">البطاقة الحالية (اضغط ثم اختر المطابقة)</div>
    <div id="mainCard" class="card" style="background:#333">--</div>
  </div>

  <div id="controls" style="margin-top:14px">
    <button id="startBtn">بدء التجربة</button>
    <button id="nextBtn" class="hidden">التالي</button>
    <button id="downloadBtn" class="hidden">تحميل بيانات CSV</button>
  </div>

  <div id="feedback" class="meta"></div>

  <div id="results" class="hidden">
    <h3>النتائج (مبدئية)</h3>
    <div>الدقة الإجمالية: <span id="accuracy">0%</span></div>
    <div>الفئات المكتملة: <span id="categoriesCompleted">0</span></div>
    <div>أخطاء الصمود (perseverative errors): <span id="perseverative">0</span></div>
  </div>

  <div style="margin-top:14px" class="meta">ملاحظة: هذه نسخة ويب مبنية لأغراض بحثية/تعليمية. البيانات محفوظة محليًا وتُحمّل كملف CSV عند الانتهاء.</div>
</div>

<script>
/*
  WCST Web — single-file implementation (basic)
  - Stimuli: cards defined by number(1..4), shape, color
  - Rules: "number", "shape", "color"
  - Active rule changes after winStreakThreshold correct in a row (default 5)
  - Tracks trial-level data, reaction time, correctness, matched categories, active rule, win streak
  - Exports CSV
*/

// SETTINGS
const winStreakThreshold = 5; // بعد هذه السلسلة تتغير القاعدة
const totalTrials = 60; // عدد التجارب (يمكن تعديل)
const refSpecs = [
  {id:1, number:1, shape:'مثلث', color:'أحمر', bg:'#d9534f'},
  {id:2, number:2, shape:'نجمة',  color:'أخضر', bg:'#5cb85c'},
  {id:3, number:3, shape:'مربع',  color:'أصفر', bg:'#f0ad4e'},
  {id:4, number:4, shape:'دائرة', color:'أزرق', bg:'#337ab7'}
];
const shapesToSymbol = {'مثلث':'▲','نجمة':'★','مربع':'■','دائرة':'●'};

// STATE
let deck = [];
let currentCard = null;
let activeRule = null;
let gameStarted = false;
let trialIndex = 0;
let winStreak = 0;
let gameData = []; // each trial: [trial#, card, chosenRef, success, matchedOn, activeRule, winStreak, rt_ms]
let lastStimulusTime = 0;
let username = '';// UTILS
function randInt(n){ return Math.floor(Math.random()*n); }
function shuffleArray(a){ for(let i=a.length-1;i>0;i--){const j=randInt(i+1);[a[i],a[j]]=[a[j],a[i]];} return a; }
function pickRandomRule(exclude=null){
  const rules=['number','shape','color'].filter(r=>r!==exclude);
  return rules[randInt(rules.length)];
}
function createDeck(size){
  // create random cards using numbers 1..4, shapes/colors mapped to refSpecs
  const cards = [];
  for(let i=0;i<size;i++){
    const r = refSpecs[randInt(refSpecs.length)];
    // create some variability by mixing attributes
    const num = 1 + randInt(4);
    const shape = refSpecs[ randInt(refSpecs.length) ].shape;
    const color = refSpecs[ randInt(refSpecs.length) ].color;
    const bg = refSpecs.find(x=>x.color===color).bg || '#666';
    cards.push({id:i+1, number:num, shape:shape, color:color, bg:bg});
  }
  return cards;
}

function renderRefRow(){
  const row = document.getElementById('refRow'); row.innerHTML='';
  refSpecs.forEach(r=>{
    const el = document.createElement('div');
    el.className='card small';
    el.style.background = r.bg;
    el.dataset.refId = r.id;
    el.dataset.number = r.number;
    el.dataset.shape = r.shape;
    el.dataset.color = r.color;
    el.innerHTML = <div style="font-size:16px">${shapesToSymbol[r.shape]}<div style="font-size:12px;margin-top:6px">${r.number}</div></div>;
    el.addEventListener('click', ()=>onRefClick(r.id));
    row.appendChild(el);
  });
}

function showMainCard(c){
  const mc = document.getElementById('mainCard');
  mc.style.background = c.bg;
  mc.innerHTML = <div style="font-size:36px">${shapesToSymbol[c.shape]}</div><div style="font-size:16px;margin-top:6px">${c.number}</div>;
}

function chooseStartingRule(){
  activeRule = pickRandomRule(null);
  document.getElementById('activeRuleText').textContent = ruleArabic(activeRule);
}

function ruleArabic(rule){
  if(rule==='number') return 'الرقم';
  if(rule==='shape') return 'الشكل';
  if(rule==='color') return 'اللون';
  return '---';
}

function startGame(){
  username = document.getElementById('username').value || 'anonymous';
  deck = createDeck(totalTrials);
  shuffleArray(deck);
  trialIndex = 0; winStreak = 0; gameData=[]; gameStarted=true;
  chooseStartingRule();
  document.getElementById('startBtn').classList.add('hidden');
  document.getElementById('nextBtn').classList.remove('hidden');
  document.getElementById('feedback').textContent = '';
  document.getElementById('results').classList.add('hidden');
  presentNext();
}

function presentNext(){
  if(trialIndex >= deck.length){
    endGame();
    return;
  }
  currentCard = deck[trialIndex];
  showMainCard(currentCard);
  lastStimulusTime = performance.now();
  document.getElementById('trialCounter').textContent = trialIndex+1;
  // hide next button until response
  document.getElementById('nextBtn').disabled = true;
}

function onRefClick(refId){
  if(!gameStarted || !currentCard) return;
  const rt = Math.round(performance.now() - lastStimulusTime);
  const ref = refSpecs.find(r=>r.id===refId);
  const chosen = {number:ref.number, shape:ref.shape, color:ref.color, id:ref.id};
  // determine if match under activeRule
  let success=false;
  let matchedOn=[];
  if(activeRule==='number' && currentCard.number===chosen.number) success=true;
  if(activeRule==='shape' && currentCard.shape===chosen.shape) success=true;
  if(activeRule==='color' && currentCard.color===chosen.color) success=true;
  // For "matched categories", check all possible matches (helpful for perseveration detection)
  if(currentCard.number===chosen.number) matchedOn.push('number');
  if(currentCard.shape===chosen.shape) matchedOn.push('shape');
  if(currentCard.color===chosen.color) matchedOn.push('color');

  // record trial data
  gameData.push({trial: trialIndex+1,
    card: ${currentCard.number}_${currentCard.shape}_${currentCard.color},
    chosen: ${chosen.number}_${chosen.shape}_${chosen.color},
    success: success,
    matchedOn: matchedOn.join('|') || 'none',
    activeRule: activeRule,
    winStreakBefore: winStreak,
    rt_ms: rt,
    timestamp: new Date().toISOString()
  });

  // feedback
  const fb = document.getElementById('feedback');
  fb.textContent = success ? صحيح — الوقت: ${rt} ms : خطاء — الوقت: ${rt} ms;
  fb.style.color = success ? 'green' : 'crimson';

  // update winStreak
  if(success){
    winStreak += 1;
  } else {
    winStreak = 0;
  }

  // maybe change rule
  let ruleChanged = false;
  if(winStreak >= winStreakThreshold){
    // choose different new rule
    const old = activeRule;
    activeRule = pickRandomRule(old);
    document.getElementById('activeRuleText').textContent = ruleArabic(activeRule);
    winStreak = 0; // reset after change
    ruleChanged = true;
  }

  // increment trial index
  trialIndex += 1;
  document.getElementById('nextBtn').disabled = false;
  updateResultsPreview();
}

function updateResultsPreview(){
  const total = gameData.length;
  if(total===0) return;
  const correct = gameData.filter(t=>t.success).length;
  const procent = Math.round((correct/total)*100);
  document.getElementById('accuracy').textContent = procent + '%';
  // completed categories: count number of times winStreakMarker reached threshold (we recorded activeRules changes)
  // approximate by counting number of times we changed rule (simple approach)
  // compute perseverative errors: trial was wrong but chosen matched previous rule (approximation)
  const categoriesCompleted = Math.max(0, gameData.filter(t=>t.winStreakBefore>=winStreakThreshold).length);
  document.getElementById('categoriesCompleted').textContent = categoriesCompleted;
  // simple perseverative calculation:
  let pres = 0;
  for(let i=1;i<gameData.length;i++){
    const prev = gameData[i-1];
    const cur = gameData[i];
    // if current is wrong but matches previous activeRule, count as perseverative
    if(!cur.success && prev.activeRule && cur.matchedOn.includes(prev.activeRule)){
      pres += 1;
    }
  }
  document.getElementById('perseverative').textContent = pres;
  document.getElementById('results').classList.remove('hidden');
}

function endGame(){
  gameStarted=false;
  document.getElementById('feedback').textContent = 'انتهت التجربة. يمكنك تنزيل الملف.';
  document.getElementById('nextBtn').classList.add('hidden');
  document.getElementById('downloadBtn').classList.remove('hidden');
  document.getElementById('startBtn').classList.remove('hidden');
  document.getElementById('startBtn').textContent = 'إعادة تشغيل';
}

function downloadCSV(){
  // convert gameData to CSV
  if(gameData.length===0) return;
  const header = ['participant','trial','card','chosen','success','matchedOn','activeRule','winStreakBefore','rt_ms','timestamp'];
  const rows = gameData.map(r=>[
    "${(username||'anonymous')}",
    r.trial, "${r.card}", "${r.chosen}", r.success?1:0, "${r.matchedOn}", "${r.activeRule}", r.winStreakBefore, r.rt_ms, "${r.timestamp}"
  ].join(','));
  const csv = [header.join(','), ...rows].join('\r\n');
  const blob = new Blob([csv], {type: 'text/csv;charset=utf-8;'});
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  const fname = WCST_${username||'participant'}_${new Date().toISOString().slice(0,19).replace(/[:T]/g,'-')}.csv;
  a.download = fname;
  document.body.appendChild(a); a.click();
  a.remove();
}

// UI wiring
document.getElementById('startBtn').addEventListener('click', startGame);
document.getElementById('nextBtn').addEventListener('click', ()=>{
  document.getElementById('nextBtn').disabled = true;
  presentNext();
});
document.getElementById('downloadBtn').addEventListener('click', downloadCSV);

// init UI
renderRefRow();
document.getElementById('activeRuleText').textContent = '---';
</script>
</body>
</html>
