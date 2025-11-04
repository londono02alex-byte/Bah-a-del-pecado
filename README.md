#!/usr/bin/env bash
set -e

ROOT="pongamozlo"
ZIPNAME="bahia-del-pecado.zip"

echo "Creando carpeta $ROOT ..."
rm -rf "$ROOT" "$ZIPNAME"
mkdir -p "$ROOT/assets"

echo "Escribiendo index.html ..."
cat > "$ROOT/index.html" <<'HTML'
<!doctype html>
<html lang="es">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1.0">
  <title>Pongamozlo</title>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <div id="menu" class="screen">
    <h1>Pongamozlo</h1>
    <div class="menu-row">
      <button id="btnPlay">Jugar</button>
      <button id="btnMode">Modo: Local</button>
      <button id="btnAudio">Sonidos: ON</button>
      <button id="btnHelp">Ayuda</button>
    </div>
    <div class="small">W/S izq • ↑/↓ der • P pausar • Espacio reiniciar bola</div>
  </div>

  <div id="game" class="screen hidden">
    <canvas id="screen" width="900" height="520"></canvas>
    <div id="hud">
      <span id="scoreLeft">0</span>
      <span id="scoreRight">0</span>
    </div>
    <div id="notifications"></div>
    <div id="controls">W/S para izquierda • Flechas arriba/abajo para derecha • P pausar • Espacio reinicia</div>
  </div>

  <div id="help" class="screen hidden">
    <h2>Ayuda</h2>
    <p>Controles: Jugador Izquierdo W/S, Jugador Derecho ↑/↓.</p>
    <p>Power-ups: aparecen cuadrados con iconos; recógelos con la paleta para activar un efecto temporal.</p>
    <button id="btnBack">Volver</button>
  </div>

  <script src="pong.js"></script>
</body>
</html>
HTML

echo "Escribiendo style.css ..."
cat > "$ROOT/style.css" <<'CSS'
* { box-sizing: border-box; margin: 0; padding: 0; font-family: Inter, Arial, Helvetica, sans-serif; }
body { display:flex; align-items:center; justify-content:center; min-height:100vh; background:linear-gradient(180deg,#071021 0%, #041025 100%); color:#e6eef6; }
.screen { text-align:center; width:100%; max-width:960px; padding:18px; }
.hidden { display:none; }
h1 { font-size:48px; margin:22px 0; color:#fff; letter-spacing:2px; text-shadow:0 2px 6px rgba(0,0,0,0.6); }
.menu-row { display:flex; gap:12px; justify-content:center; margin-bottom:18px; flex-wrap:wrap; }
button { background:#15202b; color:#e6eef6; border:1px solid #2b4454; padding:10px 16px; cursor:pointer; border-radius:8px; font-weight:600; box-shadow:0 2px 0 rgba(0,0,0,0.2); }
button:hover { transform:translateY(-2px); }
canvas { background:linear-gradient(180deg,#021124 0%, #041025 100%); display:block; margin:8px auto; border: 4px solid #173244; border-radius:8px; box-shadow:0 8px 30px rgba(0,0,0,0.6); }
#hud { margin-top:8px; font-size:26px; display:flex; justify-content:space-between; width:900px; margin-left:auto; margin-right:auto; color:#bfe3ff; }
#controls { margin-top:6px; font-size:14px; color:#9fb4c9; }
#notifications { position:relative; width:900px; height:28px; margin:8px auto 0; color:#fffbeb; font-weight:700; text-shadow:0 1px 0 rgba(0,0,0,0.6); }
.power-label { font-size:13px; color:#cfefff; }
.small { color:#9fb4c9; margin-top:10px; }
CSS

echo "Escribiendo pong.js ..."
cat > "$ROOT/pong.js" <<'JS'
// Pongamozlo — versión con balance de power-ups, iconos SVG locales y proyecto listo para empaquetar.

const menuEl = document.getElementById('menu');
const gameEl = document.getElementById('game');
const helpEl = document.getElementById('help');
const btnPlay = document.getElementById('btnPlay');
const btnMode = document.getElementById('btnMode');
const btnAudio = document.getElementById('btnAudio');
const btnHelp = document.getElementById('btnHelp');
const btnBack = document.getElementById('btnBack');
const notifEl = document.getElementById('notifications');

const canvas = document.getElementById('screen');
const ctx = canvas.getContext('2d');
const W = canvas.width, H = canvas.height;

let audioEnabled = true;
let mode = 'local'; // 'local' or 'ia'

// Power-up balance tunables
const POWER_SPAWN_MIN = 6500;   // min ms between spawns
const POWER_SPAWN_MAX = 12000;  // max ms between spawns
const POWER_DURATION = 7000;    // ms effect duration
const POWER_FALL_SPEED_MIN = 1.2;
const POWER_FALL_SPEED_MAX = 2.1;
const MAX_ACTIVE_EFFECTS_PER_PLAYER = 2;

btnMode.addEventListener('click', () => {
  mode = (mode === 'local') ? 'ia' : 'local';
  btnMode.textContent = `Modo: ${mode === 'local' ? 'Local' : 'IA'}`;
});
btnAudio.addEventListener('click', () => {
  audioEnabled = !audioEnabled;
  btnAudio.textContent = `Sonidos: ${audioEnabled ? 'ON' : 'OFF'}`;
});
btnHelp.addEventListener('click', () => { menuEl.classList.add('hidden'); helpEl.classList.remove('hidden'); });
btnBack.addEventListener('click', () => { helpEl.classList.add('hidden'); menuEl.classList.remove('hidden'); });

btnPlay.addEventListener('click', () => {
  menuEl.classList.add('hidden');
  gameEl.classList.remove('hidden');
  startGame();
});

// Mini WebAudio for simple SFX
const AudioCtx = window.AudioContext || window.webkitAudioContext;
let audioCtx = null;
function ensureAudio() { if (!audioCtx) audioCtx = new AudioCtx(); }
function playTone(freq=440, dur=0.06, type='sine', vol=0.08) {
  if (!audioEnabled) return;
  ensureAudio();
  const o = audioCtx.createOscillator();
  const g = audioCtx.createGain();
  o.type = type; o.frequency.value = freq;
  g.gain.value = vol;
  o.connect(g); g.connect(audioCtx.destination);
  o.start();
  g.gain.exponentialRampToValueAtTime(0.0001, audioCtx.currentTime + dur);
  setTimeout(()=> o.stop(), dur*1000 + 20);
}
const sfx = {
  hit: ()=> playTone(880,0.05,'square',0.06),
  wall: ()=> playTone(600,0.04,'sine',0.05),
  score: ()=> playTone(220,0.18,'sine',0.14),
  power: ()=> playTone(1200,0.08,'sawtooth',0.11)
};

// Game objects
const paddleDefault = { w: 14, h: 110 };
let left = { x: 12, y: H/2 - paddleDefault.h/2, w: paddleDefault.w, h: paddleDefault.h, dy: 0, score: 0, invert: false, activeEffects: [] };
let right = { x: W - 12 - paddleDefault.w, y: H/2 - paddleDefault.h/2, w: paddleDefault.w, h: paddleDefault.h, dy: 0, score: 0, invert: false, activeEffects: [] };
const ball = { x: W/2, y: H/2, vx: 6*(Math.random()>0.5?1:-1), vy: 3*(Math.random()*2-1), r: 10 };

let paused = false, running = false;
const keys = {};
window.addEventListener('keydown', e => { keys[e.key] = true; if (e.key === 'p' || e.key === 'P') paused = !paused; if (e.key === ' ') resetBall(Math.random()>0.5?1:-1); });
window.addEventListener('keyup', e => { keys[e.key] = false; });

// power-up system
const powerups = []; // {x,y,w,h,vy,type,icon}
const powerTypes = [
  { id:'bigPaddle', label:'Paleta Grande', color:'#2ee6a6', icon:'assets/big.svg' },
  { id:'smallPaddle', label:'Paleta Pequeña', color:'#ff7b7b', icon:'assets/small.svg' },
  { id:'slowBall', label:'Bola Lenta', color:'#6fb6ff', icon:'assets/slow.svg' },
  { id:'fastBall', label:'Bola Rápida', color:'#ffd24d', icon:'assets/fast.svg' },
  { id:'invertControls', label:'Controles Invertidos', color:'#b892ff', icon:'assets/invert.svg' }
];
let nextPowerSpawn = performance.now() + randRange(2000, 5000);

// AI
const ai = { enabled:false, speed: 5.2, reaction: 0.12 };

// start game
function startGame() {
  left.y = H/2 - paddleDefault.h/2; left.score = 0; left.h = paddleDefault.h; left.w = paddleDefault.w; left.invert=false; left.activeEffects=[];
  right.y = H/2 - paddleDefault.h/2; right.score = 0; right.h = paddleDefault.h; right.w = paddleDefault.w; right.invert=false; right.activeEffects=[];
  ball.x = W/2; ball.y = H/2;
  ball.vx = 6*(Math.random()>0.5?1:-1); ball.vy = 3*(Math.random()*2-1);
  paused = false; running = true;
  powerups.length = 0;
  nextPowerSpawn = performance.now() + randRange(2500, 6000);
  ai.enabled = (mode === 'ia');
  showNotification('¡A jugar!');
  requestAnimationFrame(loop);
}

function loop(ts) {
  if (!running) return;
  if (!paused) update(ts);
  draw();
  requestAnimationFrame(loop);
}

function update(ts) {
  // input
  left.dy = 0; right.dy = 0;
  if (!left.invert) {
    if (keys['w'] || keys['W']) left.dy = -8;
    if (keys['s'] || keys['S']) left.dy = 8;
  } else {
    if (keys['w'] || keys['W']) left.dy = 8;
    if (keys['s'] || keys['S']) left.dy = -8;
  }

  if (!ai.enabled) {
    if (!right.invert) {
      if (keys['ArrowUp']) right.dy = -8;
      if (keys['ArrowDown']) right.dy = 8;
    } else {
      if (keys['ArrowUp']) right.dy = 8;
      if (keys['ArrowDown']) right.dy = -8;
    }
  } else {
    // AI tracks ball with limited speed and randomness
    const desiredY = ball.y - right.h/2 + (Math.sin(performance.now()/700)*12);
    const diff = desiredY - right.y;
    right.y += clamp(diff * ai.reaction, -ai.speed, ai.speed);
  }

  left.y = clamp(left.y + left.dy, 0, H - left.h);
  right.y = clamp(right.y + right.dy, 0, H - right.h);

  // ball physics
  ball.x += ball.vx; ball.y += ball.vy;
  if (ball.y - ball.r < 0) { ball.y = ball.r; ball.vy *= -1; sfx.wall(); }
  if (ball.y + ball.r > H) { ball.y = H - ball.r; ball.vy *= -1; sfx.wall(); }

  // collisions with paddles (with speed cap)
  if (ball.vx < 0 && ball.x - ball.r < left.x + left.w &&
      ball.y > left.y && ball.y < left.y + left.h) {
    ball.vx = Math.min(14, -ball.vx * 1.06); // cap max speed
    const delta = (ball.y - (left.y + left.h/2)) / (left.h/2);
    ball.vy = 6 * delta;
    ball.x = left.x + left.w + ball.r;
    sfx.hit();
  }
  if (ball.vx > 0 && ball.x + ball.r > right.x &&
      ball.y > right.y && ball.y < right.y + right.h) {
    ball.vx = Math.max(-14, -ball.vx * 1.06);
    const delta = (ball.y - (right.y + right.h/2)) / (right.h/2);
    ball.vy = 6 * delta;
    ball.x = right.x - ball.r;
    sfx.hit();
  }

  // scoring
  if (ball.x < 0) { right.score += 1; sfx.score(); resetBall(1); }
  else if (ball.x > W) { left.score += 1; sfx.score(); resetBall(-1); }

  // spawn power-ups (random intervals)
  if (performance.now() > nextPowerSpawn) {
    spawnPowerup();
    nextPowerSpawn = performance.now() + randRange(POWER_SPAWN_MIN, POWER_SPAWN_MAX);
  }

  // move power-ups and detect pickup
  for (let i = powerups.length - 1; i >= 0; i--) {
    const p = powerups[i];
    p.y += p.vy;
    // check collision with paddles (rect vs rect)
    if (rectsOverlap(p, left)) {
      claimPower('left', p);
      powerups.splice(i,1);
      sfx.power();
    } else if (rectsOverlap(p, right)) {
      claimPower('right', p);
      powerups.splice(i,1);
      sfx.power();
    } else if (p.y > H + 40) {
      powerups.splice(i,1); // fell out
    }
  }

  // expire effects
  const now = Date.now();
  [left,right].forEach(player => {
    for (let j = player.activeEffects.length - 1; j >= 0; j--) {
      const ef = player.activeEffects[j];
      if (now > ef.until) {
        revertEffect(player, ef.type);
        player.activeEffects.splice(j,1);
        showNotification(`${ef.label} finalizó`, 1500);
      }
    }
  });
}

function draw() {
  ctx.clearRect(0,0,W,H);

  // center dashed line
  ctx.fillStyle = '#0c2a3b';
  for (let y=0; y<H; y+=22) ctx.fillRect(W/2 - 2, y, 4, 12);

  // paddles
  ctx.fillStyle = '#e6eef6';
  roundRect(ctx, left.x, left.y, left.w, left.h, 8, true, false);
  roundRect(ctx, right.x, right.y, right.w, right.h, 8, true, false);

  // ball
  ctx.beginPath();
  ctx.fillStyle = '#ffd966';
  ctx.arc(ball.x, ball.y, ball.r, 0, Math.PI*2);
  ctx.fill();

  // powerups (draw icon image if loaded, otherwise colored square + letter)
  for (const p of powerups) {
    drawPowerup(p);
  }

  // scores
  document.getElementById('scoreLeft').textContent = left.score;
  document.getElementById('scoreRight').textContent = right.score;
}

function resetBall(direction = (Math.random()>0.5?1:-1)) {
  ball.x = W/2; ball.y = H/2;
  ball.vx = 6 * direction;
  ball.vy = 3*(Math.random()*2-1);
  paused = true;
  setTimeout(()=> paused = false, 600);
}

/* ---------------- helpers ---------------- */
function roundRect(ctx,x,y,w,h,r,fill,stroke){
  if (typeof r === 'undefined') r=5;
  ctx.beginPath();
  ctx.moveTo(x+r,y);
  ctx.arcTo(x+w,y,x+w,y+h,r);
  ctx.arcTo(x+w,y+h,x,y+h,r);
  ctx.arcTo(x,y+h,x,y,r);
  ctx.arcTo(x,y,x+w,y,r);
  ctx.closePath();
  if(fill) ctx.fill();
  if(stroke) ctx.stroke();
}

function clamp(v,a,b){ return Math.max(a,Math.min(b,v)); }
function randRange(a,b){ return a + Math.random()*(b-a); }
function rectsOverlap(a,b){
  return a.x < b.x + b.w && a.x + a.w > b.x && a.y < b.y + b.h && a.y + a.h > b.y;
}

/* --------------- powerups ---------------- */
function spawnPowerup() {
  // pick type weighted to favor neutral effects less often
  const type = weightedPick([
    {id:'bigPaddle', w:1.2},
    {id:'smallPaddle', w:1.0},
    {id:'slowBall', w:1.1},
    {id:'fastBall', w:1.0},
    {id:'invertControls', w:0.9}
  ]);
  const meta = powerTypes.find(p=>p.id===type);
  const pu = {
    x: 60 + Math.random()*(W-120),
    y: -24,
    w: 28,
    h: 28,
    vy: randRange(POWER_FALL_SPEED_MIN, POWER_FALL_SPEED_MAX),
    type: meta.id,
    color: meta.color,
    label: meta.label,
    icon: meta.icon
  };
  powerups.push(pu);
}

function drawPowerup(p) {
  // try to draw icon image if available (preloaded)
  const img = loadedIcons[p.type];
  ctx.save();
  ctx.translate(p.x, p.y);
  // background rounded rect
  ctx.fillStyle = p.color;
  roundRect(ctx, -p.w/2, -p.h/2, p.w, p.h, 6, true, false);
  if (img) {
    ctx.drawImage(img, -p.w/2 + 3, -p.h/2 + 3, p.w - 6, p.h - 6);
  } else {
    ctx.fillStyle = '#021';
    ctx.font = '13px sans-serif';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    const label = p.label.split(' ').map(s=>s[0]).join('');
    ctx.fillText(label, 0, 0);
  }
  ctx.restore();
}

// pre-load SVG icons into Image objects (simple local assets)
const loadedIcons = {};
function preloadIcons() {
  powerTypes.forEach(pt => {
    const img = new Image();
    img.onload = ()=> { loadedIcons[pt.id] = img; };
    img.onerror = ()=> { /* ignore */ };
    img.src = pt.icon;
  });
}
preloadIcons();

function claimPower(who, p) {
  const target = (who === 'left') ? left : right;
  // limit simultaneous effects per player
  if (target.activeEffects.length >= MAX_ACTIVE_EFFECTS_PER_PLAYER) {
    showNotification('Máximo efectos activos', 1200);
    return;
  }
  // apply
  applyEffect(target, p.type, p.label);
}

function applyEffect(target, type, label) {
  const now = Date.now();
  const until = now + POWER_DURATION;
  target.activeEffects.push({ type, until, label });
  // persist application
  if (type === 'bigPaddle') {
    target.h = Math.min(H - 10, Math.round(target.h * 1.45));
  } else if (type === 'smallPaddle') {
    target.h = Math.max(32, Math.round(target.h * 0.55));
  } else if (type === 'slowBall') {
    ball.vx *= 0.75; ball.vy *= 0.75;
  } else if (type === 'fastBall') {
    ball.vx *= 1.33; ball.vy *= 1.33;
  } else if (type === 'invertControls') {
    target.invert = true;
  }
  showNotification(`${target === left ? 'Izquierda' : 'Derecha'} obtuvo ${label}`, 1800);
}

function revertEffect(target, type) {
  if (type === 'bigPaddle' || type === 'smallPaddle') {
    target.h = paddleDefault.h;
  } else if (type === 'slowBall' || type === 'fastBall') {
    // normalize ball speed to baseline preserving direction
    const sx = Math.sign(ball.vx) || 1;
    const sy = Math.sign(ball.vy) || 1;
    ball.vx = sx * (6 + Math.random() * 1.2);
    ball.vy = sy * (3 * (Math.random()*2-1));
  } else if (type === 'invertControls') {
    target.invert = false;
  }
}

/* notification */
let notifTimer = null;
function showNotification(text, ms = 1800) {
  notifEl.textContent = text;
  if (notifTimer) clearTimeout(notifTimer);
  notifTimer = setTimeout(()=> { if (notifEl) notifEl.textContent = ''; }, ms);
}

/* util */
function weightedPick(items) {
  const total = items.reduce((s,i)=>s+i.w,0);
  let r = Math.random() * total;
  for (const it of items) {
    r -= it.w;
    if (r <= 0) return it.id;
  }
  return items[items.length-1].id;
}

/* initial unlock audio on mobile */
window.addEventListener('touchstart', function unlock() {
  if (!audioCtx && audioEnabled) audioCtx = new AudioCtx();
  window.removeEventListener('touchstart', unlock);
});

// expose some helpers for debugging
window.pong = { resetBall, spawnPowerup };
JS

echo "Escribiendo README.md ..."
cat > "$ROOT/README.md" <<'MD'
# Pongamozlo — versión lista para probar

Contenido:
- index.html
- style.css
- pong.js
- assets/
  - big.svg
  - small.svg
  - slow.svg
  - fast.svg
  - invert.svg

Cómo probar en el navegador
1. Coloca todos los archivos en una carpeta (por ejemplo `pongamozlo/`). Asegúrate de mantener la carpeta `assets/` con los SVG.
2. Abre `index.html` en un navegador moderno (Chrome/Edge/Firefox). En móviles, abre desde servidor local (ver abajo) para evitar restricciones de archivos locales.
3. Menú → Jugar. Controles: W/S (izq), Flecha Arriba/Abajo (der), P para pausar, Espacio reinicia la bola.

Servidor local recomendado (opcional)
- Con Python 3:
  - cd pongamozlo
  - python -m http.server 8000
  - Abre http://localhost:8000

Crear APK (Capacitor) — pasos resumidos
1. Instala Node.js (v16+ recomendado) y Android Studio (con SDK).
2. Desde la carpeta del proyecto:
   - npm init -y
   - npm install @capacitor/cli @capacitor/core
   - npx cap init pongamozlo com.tuempresa.pongamozlo
3. Prepara la carpeta web:
   - Crea carpeta `www` y copia `index.html`, `style.css`, `pong.js` y `assets/` dentro.
4. Añade Android:
   - npx cap add android
   - npx cap open android
   - En Android Studio: Build > Build Bundle(s) / APK(s) > Build APK(s)
5. Si haces cambios en la web:
   - copia archivos a `www` y ejecuta:
     - npx cap copy android
     - npx cap sync android

Notas de balance y cambios hechos
- Intervalo de aparición de power-ups variable entre 6.5s y 12s (configurable).
- Duración de efectos: 7s (configurable).
- Límite de efectos simultáneos por jugador: 2.
- Iconos SVG en `assets/` utilizados en canvas; si el navegador no carga la imagen muestra abreviatura.
- Sonidos sencillos con WebAudio para evitar dependencias externas.
MD

echo "Escribiendo assets SVG ..."
cat > "$ROOT/assets/big.svg" <<'SVG'
<!-- Paleta grande (verde) -->
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 64 64">
  <rect width="64" height="64" rx="10" fill="#2ee6a6"/>
  <rect x="12" y="20" width="40" height="24" rx="4" fill="#022"/>
  <text x="32" y="36" font-size="18" font-family="Arial" text-anchor="middle" fill="#fff" font-weight="700">B</text>
</svg>
SVG

cat > "$ROOT/assets/small.svg" <<'SVG'
<!-- Paleta pequeña (rojo) -->
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 64 64">
  <rect width="64" height="64" rx="10" fill="#ff7b7b"/>
  <rect x="18" y="26" width="28" height="12" rx="3" fill="#021"/>
  <text x="32" y="36" font-size="14" font-family="Arial" text-anchor="middle" fill="#fff" font-weight="700">S</text>
</svg>
SVG

cat > "$ROOT/assets/slow.svg" <<'SVG'
<!-- Bola lenta (azul) -->
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 64 64">
  <rect width="64" height="64" rx="10" fill="#6fb6ff"/>
  <circle cx="32" cy="32" r="10" fill="#021"/>
  <text x="32" y="36" font-size="12" font-family="Arial" text-anchor="middle" fill="#fff" font-weight="700">L</text>
</svg>
SVG

cat > "$ROOT/assets/fast.svg" <<'SVG'
<!-- Bola rápida (amarillo) -->
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 64 64">
  <rect width="64" height="64" rx="10" fill="#ffd24d"/>
  <circle cx="32" cy="32" r="10" fill="#021"/>
  <text x="32" y="36" font-size="12" font-family="Arial" text-anchor="middle" fill="#021" font-weight="700">F</text>
</svg>
SVG

cat > "$ROOT/assets/invert.svg" <<'SVG'
<!-- Invertir controles (morado) -->
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 64 64">
  <rect width="64" height="64" rx="10" fill="#b892ff"/>
  <path d="M20 34h24v-4H20v-6l-8 8 8 8v-6z" fill="#021"/>
  <text x="32" y="50" font-size="9" font-family="Arial" text-anchor="middle" fill="#021">I</text>
</svg>
SVG

echo "Creando ZIP $ZIPNAME ..."
( cd "$ROOT" && zip -r "../$ZIPNAME" . ) >/dev/null

echo "Hecho. creado '$ZIPNAME' y carpeta '$ROOT'."
echo ""
echo "Siguientes pasos:"
echo " - Descomprime $ZIPNAME o abre $ROOT/index.html en tu navegador."
echo " - Recomiendo servir con: (cd $ROOT && python -m http.server 8000) y abrir http://localhost:8000"

