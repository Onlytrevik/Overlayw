<!DOCTYPE html>
<html lang="de">
<head>
<meta charset="UTF-8">
<title>TikTok Auction Overlay</title>
<style>
  @import url('https://fonts.googleapis.com/css2?family=Orbitron:wght@700;900&family=Inter:wght@400;600;700;800;900&display=swap');
  * { margin:0; padding:0; box-sizing:border-box; }

  body {
    background: transparent;
    font-family: 'Inter', sans-serif;
    width: 300px;
    overflow: hidden;
  }

  .overlay {
    background: #111111;
    border-radius: 18px;
    padding: 14px 14px 12px;
    border: 1.5px solid #1e1e1e;
    width: 300px;
  }

  /* ── MINIMUM ── */
  .top-bar {
    display: flex; align-items: center; justify-content: center; gap: 8px;
    background: #1a1a1a; border-radius: 10px; padding: 8px 14px;
    margin-bottom: 8px; border: 1px solid #252525;
  }
  .minimum-label { color: #FFD700; font-size: 13px; font-weight: 800; letter-spacing: 1px; text-transform: uppercase; }
  .minimum-value {
    display: flex; align-items: center; gap: 5px;
    background: #2a2200; border: 1.5px solid #FFD700;
    border-radius: 20px; padding: 3px 10px 3px 6px;
  }
  .coin-icon {
    width: 18px; height: 18px;
    background: radial-gradient(circle at 35% 35%, #FFE566, #FFD700, #B8860B);
    border-radius: 50%; flex-shrink: 0; box-shadow: 0 0 6px rgba(255,215,0,0.4);
  }
  .minimum-num { color: #FFD700; font-size: 15px; font-weight: 900; }

  /* ── PHASE BAR ── */
  .phase-bar {
    display: flex; align-items: center; justify-content: center; gap: 10px;
    border-radius: 10px; padding: 7px 14px; margin-bottom: 10px;
    border: 1px solid #1a3a5c; background: #0d1a2a;
    transition: background 0.5s, border-color 0.5s;
  }
  .phase-bar.snipe-mode {
    background: #1a0d2a; border-color: #7c2adc;
    animation: snipeBorder 1s ease-in-out infinite;
  }
  @keyframes snipeBorder { 0%,100%{border-color:#5c1a8c} 50%{border-color:#ac4afc} }

  .phase-dot {
    width: 8px; height: 8px; border-radius: 50%; flex-shrink: 0;
    background: #4fc3f7; box-shadow: 0 0 8px #4fc3f7;
    animation: dotBlink 1.5s ease-in-out infinite;
    transition: background 0.5s, box-shadow 0.5s;
  }
  .phase-dot.snipe { background: #c47fff; box-shadow: 0 0 8px #c47fff; }
  @keyframes dotBlink { 0%,100%{opacity:1} 50%{opacity:0.3} }

  .phase-label {
    font-size: 11px; font-weight: 700; letter-spacing: 1.5px;
    text-transform: uppercase; color: #b0d4f1; transition: color 0.5s;
  }
  .phase-label.snipe { color: #d4b0f1; }

  .phase-badge {
    font-size: 13px; font-weight: 900; padding: 3px 10px;
    border-radius: 8px; letter-spacing: 1px; color: #fff;
    background: #1565C0; transition: background 0.5s;
  }
  .phase-badge.snipe { background: #6a0dad; }

  /* ── COUNTDOWN ── */
  .countdown-wrap { text-align: center; margin-bottom: 4px; }
  .countdown {
    font-family: 'Orbitron', monospace;
    font-size: 56px; font-weight: 900; color: #00E676;
    line-height: 1; letter-spacing: 4px;
    text-shadow: 0 0 20px rgba(0,230,118,0.5);
    transition: color 0.4s, text-shadow 0.4s;
  }
  .countdown.snipe-mode { color: #c47fff; text-shadow: 0 0 20px rgba(196,127,255,0.5); }
  .countdown.urgent { color: #FF5252 !important; text-shadow: 0 0 20px rgba(255,82,82,0.7) !important; animation: shake 0.45s ease-in-out infinite; }
  .countdown.idle   { color: #2a2a2a; text-shadow: none; }
  @keyframes shake { 0%,100%{transform:translateX(0)} 25%{transform:translateX(-2px)} 75%{transform:translateX(2px)} }

  .phase-sub-label {
    text-align: center; font-size: 10px; font-weight: 700;
    letter-spacing: 2.5px; text-transform: uppercase;
    color: transparent; margin-bottom: 10px; min-height: 14px;
    transition: color 0.4s;
  }
  .phase-sub-label.visible { color: #9c3adc; }

  /* ── ENDED ── */
  .ended-banner {
    display: none; text-align: center;
    background: rgba(254,44,85,0.12); border: 1px solid rgba(254,44,85,0.35);
    border-radius: 10px; padding: 8px; margin-bottom: 10px;
    color: #FE2C55; font-size: 13px; font-weight: 800; letter-spacing: 1px;
  }

  /* ── BIDDERS ── */
  .bidders { display: flex; flex-direction: column; gap: 6px; margin-bottom: 10px; min-height: 36px; }
  .bidder-row {
    display: flex; align-items: center; gap: 8px;
    background: #1a1a1a; border-radius: 12px; padding: 8px 10px;
    border: 1px solid #252525;
    animation: slideIn 0.35s cubic-bezier(.22,.68,0,1.2);
  }
  @keyframes slideIn { from{opacity:0;transform:translateX(-10px)} to{opacity:1;transform:translateX(0)} }
  .bidder-row.rank-1 { border-color: #B8860B; background: #1a1400; }
  .bidder-row.rank-2 { border-color: #555;    background: #161616; }
  .bidder-row.rank-3 { border-color: #6B3A2A; background: #150d0a; }
  .rank-badge { font-size: 18px; flex-shrink: 0; }
  .avatar {
    width: 34px; height: 34px; border-radius: 50%;
    background: #2a2a2a; border: 2px solid #333;
    display: flex; align-items: center; justify-content: center;
    font-size: 15px; flex-shrink: 0; overflow: hidden;
  }
  .avatar img { width: 100%; height: 100%; object-fit: cover; border-radius: 50%; }
  .avatar.rank-1 { border-color: #FFD700; }
  .avatar.rank-2 { border-color: #aaa; }
  .avatar.rank-3 { border-color: #CD7F32; }
  .bidder-info { flex: 1; min-width: 0; }
  .bidder-name { font-size: 13px; font-weight: 700; color: #fff; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; max-width: 140px; }
  .bidder-sub  { font-size: 10px; color: rgba(255,255,255,0.3); margin-top: 1px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; max-width: 140px; }
  .bidder-amount { display: flex; align-items: center; gap: 4px; margin-top: 2px; }
  .bid-num { font-size: 14px; font-weight: 800; color: #FFD700; }
  .bid-coin { width: 12px; height: 12px; background: radial-gradient(circle at 35% 35%, #FFE566, #FFD700, #B8860B); border-radius: 50%; flex-shrink: 0; }
  .empty-state { text-align: center; color: rgba(255,255,255,0.15); font-size: 12px; padding: 14px 0; }

  .total-bar { text-align: center; color: rgba(255,255,255,0.3); font-size: 11px; font-weight: 600; }
  .total-bar span { color: rgba(255,255,255,0.6); }
</style>
</head>
<body>
<div class="overlay">

  <div class="top-bar">
    <span class="minimum-label">MINIMUM :</span>
    <div class="minimum-value">
      <div class="coin-icon"></div>
      <span class="minimum-num" id="minVal">1</span>
    </div>
  </div>

  <div class="phase-bar" id="phaseBar">
    <div class="phase-dot" id="phaseDot"></div>
    <span class="phase-label" id="phaseLabel">AUKTION</span>
    <span class="phase-badge" id="phaseBadge">60S</span>
  </div>

  <div class="countdown-wrap">
    <div class="countdown idle" id="cd">1:00</div>
  </div>
  <div class="phase-sub-label" id="phaseSub"></div>

  <div class="ended-banner" id="endedBanner">🏁 AUKTION BEENDET</div>

  <div class="bidders" id="bidderList">
    <div class="empty-state">Warte auf Geschenke… 🎁</div>
  </div>

  <div class="total-bar">🏆 Total participants: <span id="totalCount">0</span></div>
</div>

<script>
// ── URL Parameter ───────────────────────────────
// Benutze so:  overlay.html?duration=60&snipe=30&minimum=1
const P        = new URLSearchParams(location.search);
const DURATION = parseInt(P.get('duration') || '60');
const SNIPE    = parseInt(P.get('snipe')    || '30');
const MINIMUM  = parseInt(P.get('minimum')  || '1');

// ── State ───────────────────────────────────────
let bidders  = [];
let timeLeft = DURATION;
let phase    = 'idle';   // idle | auction | snipe | ended
let timer    = null;
let ws       = null;

const medals   = ['🥇','🥈','🥉'];
const fallback = ['🎮','👑','🔥','💎','🦁','⭐','🐯','🎯','🌟','🚀'];

// Init display
document.getElementById('minVal').textContent      = MINIMUM;
document.getElementById('phaseBadge').textContent  = DURATION + 'S';
setClock(DURATION);

// ── Timer ───────────────────────────────────────
function startAuction() {
  if (phase === 'auction' || phase === 'snipe') return;
  phase    = 'auction';
  timeLeft = DURATION;
  document.getElementById('endedBanner').style.display = 'none';
  applyPhaseUI();
  clearInterval(timer);
  timer = setInterval(tick, 1000);
  setClock(timeLeft);
}

function tick() {
  timeLeft--;
  setClock(timeLeft);
  if (phase === 'auction' && timeLeft <= 0) enterSnipe();
  else if (phase === 'snipe' && timeLeft <= 0) endAuction();
}

function enterSnipe() {
  phase    = 'snipe';
  timeLeft = SNIPE;
  applyPhaseUI();
  setClock(timeLeft);
}

function endAuction() {
  clearInterval(timer);
  phase = 'ended';
  applyPhaseUI();
  setClock(0);
  document.getElementById('endedBanner').style.display = 'block';
}

function resetAll() {
  clearInterval(timer);
  phase    = 'idle';
  bidders  = [];
  timeLeft = DURATION;
  document.getElementById('endedBanner').style.display = 'none';
  applyPhaseUI();
  setClock(DURATION);
  renderBidders();
}

// ── Phase UI ────────────────────────────────────
function applyPhaseUI() {
  const bar   = document.getElementById('phaseBar');
  const dot   = document.getElementById('phaseDot');
  const lbl   = document.getElementById('phaseLabel');
  const badge = document.getElementById('phaseBadge');
  const sub   = document.getElementById('phaseSub');
  const cd    = document.getElementById('cd');

  cd.classList.remove('snipe-mode','idle','urgent');

  if (phase === 'auction') {
    bar.className   = 'phase-bar';
    dot.className   = 'phase-dot';
    lbl.className   = 'phase-label';
    lbl.textContent = 'AUKTION';
    badge.className = 'phase-badge';
    badge.textContent = DURATION + 'S';
    sub.className   = 'phase-sub-label';
    sub.textContent = '';
  } else if (phase === 'snipe') {
    bar.className   = 'phase-bar snipe-mode';
    dot.className   = 'phase-dot snipe';
    lbl.className   = 'phase-label snipe';
    lbl.textContent = 'SNIPE DELAY';
    badge.className = 'phase-badge snipe';
    badge.textContent = SNIPE + 'S';
    sub.className   = 'phase-sub-label visible';
    sub.textContent = '⚡ BONUS ZEIT';
    cd.classList.add('snipe-mode');
  } else if (phase === 'ended') {
    bar.className   = 'phase-bar';
    dot.className   = 'phase-dot';
    lbl.className   = 'phase-label';
    lbl.textContent = 'BEENDET';
    badge.className = 'phase-badge';
    badge.textContent = '—';
    sub.className   = 'phase-sub-label';
    sub.textContent = '';
    cd.classList.add('idle');
  } else {
    bar.className   = 'phase-bar';
    dot.className   = 'phase-dot';
    lbl.className   = 'phase-label';
    lbl.textContent = 'AUKTION';
    badge.className = 'phase-badge';
    badge.textContent = DURATION + 'S';
    sub.className   = 'phase-sub-label';
    sub.textContent = '';
    cd.classList.add('idle');
  }
}

function setClock(s) {
  const m  = Math.floor(Math.abs(s) / 60);
  const ss = Math.abs(s) % 60;
  const cd = document.getElementById('cd');
  cd.textContent = m + ':' + String(ss).padStart(2, '0');
  const urgent = s <= 10 && s > 0 && (phase === 'auction' || phase === 'snipe');
  cd.classList.toggle('urgent', urgent);
}

// ── Bids ────────────────────────────────────────
function addBid(username, avatar, coins, gift) {
  if (coins < MINIMUM) return;
  if (phase === 'idle' || phase === 'ended') startAuction();
  // Snipe: reset timer
  if (phase === 'snipe') { timeLeft = SNIPE; setClock(timeLeft); }

  const ex = bidders.find(b => b.name === username);
  if (ex) {
    if (coins > ex.coins) { ex.coins = coins; ex.gift = gift; } else return;
  } else {
    bidders.push({ name: username, avatar, coins, gift, emoji: fallback[Math.floor(Math.random()*fallback.length)] });
  }
  bidders.sort((a,b) => b.coins - a.coins);
  renderBidders();
}

function renderBidders() {
  const list = document.getElementById('bidderList');
  document.getElementById('totalCount').textContent = bidders.length;
  const top3 = bidders.slice(0,3);
  if (!top3.length) { list.innerHTML = '<div class="empty-state">Warte auf Geschenke… 🎁</div>'; return; }
  list.innerHTML = top3.map((b,i) => {
    const av = b.avatar ? `<img src="${b.avatar}" onerror="this.replaceWith(document.createTextNode('${b.emoji}'))">` : b.emoji;
    return `<div class="bidder-row rank-${i+1}">
      <div class="rank-badge">${medals[i]}</div>
      <div class="avatar rank-${i+1}">${av}</div>
      <div class="bidder-info">
        <div class="bidder-name">${esc(b.name)}</div>
        <div class="bidder-sub">${esc(b.gift||'')}</div>
        <div class="bidder-amount"><span class="bid-num">${b.coins.toLocaleString('de-DE')}</span><div class="bid-coin"></div></div>
      </div></div>`;
  }).join('');
}

function esc(s) { return String(s).replace(/[<>&"']/g,c=>({'<':'&lt;','>':'&gt;','&':'&amp;','"':'&quot;',"'":'&#39;'}[c])); }

// ── BroadcastChannel (Controller ↔ Overlay) ────
try {
  const ch = new BroadcastChannel('auction_overlay');
  ch.onmessage = e => handleMsg(e.data);
  // Answer pings so controller knows overlay is open
  ch.onmessage = e => {
    if (e.data.type === 'ping') { ch.postMessage({type:'pong'}); return; }
    handleMsg(e.data);
  };
} catch(e) {}

// ── WebSocket (Python Server) ───────────────────
function connectWS() {
  try {
    ws = new WebSocket('ws://localhost:8765');
    ws.onopen    = () => {};
    ws.onclose   = () => setTimeout(connectWS, 3000);
    ws.onerror   = () => {};
    ws.onmessage = e => handleMsg(JSON.parse(e.data));
  } catch(e) {}
}
connectWS();

function handleMsg(d) {
  if (d.type === 'bid')    addBid(d.username, d.avatar, d.coins, d.gift);
  if (d.type === 'reset')  resetAll();
  if (d.type === 'start')  startAuction();
  if (d.type === 'ended')  endAuction();
  if (d.type === 'snipe')  { phase='snipe'; timeLeft=d.seconds; applyPhaseUI(); setClock(timeLeft); clearInterval(timer); timer=setInterval(tick,1000); }
  if (d.type === 'tick')   { timeLeft=d.timeLeft; phase=d.phase; setClock(timeLeft); applyPhaseUI(); }
  if (d.type === 'config') { /* overlay uses URL params, ignore */ }
  if (d.type === 'removeBid') { bidders=bidders.filter(b=>b.name!==d.username); renderBidders(); }
}
</script>
</body>
</html>
