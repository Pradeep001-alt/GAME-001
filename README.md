<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Fruit Frenzy</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  html, body {
    height: 100%;
    overflow: hidden;
    font-family: 'Trebuchet MS', 'Segoe UI', sans-serif;
    background: radial-gradient(circle at 50% 0%, #ffe08a 0%, #ff9ecb 45%, #7f7fff 100%);
  }
  #wrap {
    width: 100%;
    height: 100vh;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 10px;
  }
  #hud {
    display: flex;
    gap: 22px;
    color: #fff;
    font-weight: 900;
    font-size: 22px;
    text-shadow: 0 3px 0 rgba(0,0,0,0.2);
    user-select: none;
  }
  #hud span.label { font-size: 13px; opacity: 0.85; display:block; font-weight: 700; }
  canvas {
    border-radius: 18px;
    box-shadow: 0 14px 40px rgba(0,0,0,0.3);
    background: #fff8ea;
    touch-action: none;
  }
  #controls {
    display: grid;
    grid-template-columns: 56px 56px 56px;
    grid-template-rows: 56px 56px;
    gap: 8px;
    margin-top: 4px;
    user-select: none;
  }
  #controls button {
    border: none;
    border-radius: 12px;
    background: rgba(255,255,255,0.85);
    font-size: 22px;
    font-weight: 900;
    color: #6a4bd6;
    box-shadow: 0 4px 0 rgba(0,0,0,0.15);
    cursor: pointer;
  }
  #controls button:active { transform: translateY(2px); box-shadow: none; }
  #up { grid-column: 2; grid-row: 1; }
  #left { grid-column: 1; grid-row: 2; }
  #down { grid-column: 2; grid-row: 2; }
  #right { grid-column: 3; grid-row: 2; }
  #overlay {
    position: fixed;
    inset: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    text-align: center;
    color: #fff;
    background: rgba(40, 20, 70, 0.4);
    pointer-events: none;
  }
  #overlay h1 { font-size: 42px; text-shadow: 0 4px 0 rgba(0,0,0,0.25); margin-bottom: 8px; }
  #overlay p { font-size: 17px; font-weight: 600; text-shadow: 0 2px 0 rgba(0,0,0,0.2); }
  #overlay .hint { margin-top: 14px; font-size: 14px; opacity: 0.85; }
  @media (hover: hover) {
    #controls { display: none; }
  }
</style>
</head>
<body>
<div id="wrap">
  <div id="hud">
    <div><span class="label">SCORE</span><span id="score">0</span></div>
    <div><span class="label">BEST</span><span id="best">0</span></div>
  </div>
  <canvas id="c" width="420" height="420"></canvas>
  <div id="controls">
    <button id="up">▲</button>
    <button id="left">◀</button>
    <button id="down">▼</button>
    <button id="right">▶</button>
  </div>
</div>
<div id="overlay">
  <h1 id="title">Fruit Frenzy 🍓</h1>
  <p id="sub">Arrow keys or WASD to move</p>
  <p class="hint">Press any direction key to start</p>
</div>

<script>
const canvas = document.getElementById('c');
const ctx = canvas.getContext('2d');
const GRID = 21;
const CELL = canvas.width / GRID;

const scoreEl = document.getElementById('score');
const bestEl = document.getElementById('best');
const overlay = document.getElementById('overlay');
const title = document.getElementById('title');
const sub = document.getElementById('sub');

let best = 0;
try {
  const m = window.name.match(/fruitfrenzy_best=(\d+)/);
  if (m) best = parseInt(m[1], 10);
} catch(e) {}
bestEl.textContent = best;

const FRUITS = [
  {color:'#ff5d5d', name:'apple'},
  {color:'#ffbb3d', name:'orange'},
  {color:'#8bd45a', name:'grape'},
  {color:'#ff7fc4', name:'berry'},
];

let snake, dir, nextDir, food, score, state, tickTimer, speed, particles;

function reset() {
  snake = [{x:10,y:10},{x:9,y:10},{x:8,y:10}];
  dir = {x:1,y:0};
  nextDir = {x:1,y:0};
  score = 0;
  speed = 120;
  particles = [];
  scoreEl.textContent = 0;
  placeFood();
  state = 'ready';
}

function placeFood() {
  let pos;
  do {
    pos = {x: Math.floor(Math.random()*GRID), y: Math.floor(Math.random()*GRID)};
  } while (snake.some(s => s.x===pos.x && s.y===pos.y));
  const f = FRUITS[Math.floor(Math.random()*FRUITS.length)];
  food = {...pos, color: f.color};
}

reset();

function setDir(x,y) {
  if (state === 'ready') { state = 'playing'; overlay.style.display = 'none'; startLoop(); }
  if (state === 'dead') { reset(); state = 'playing'; overlay.style.display = 'none'; startLoop(); return; }
  if (dir.x === -x && dir.y === -y) return; // no reverse
  nextDir = {x,y};
}

document.addEventListener('keydown', e => {
  const k = e.key.toLowerCase();
  if (['arrowup','w'].includes(k)) setDir(0,-1);
  else if (['arrowdown','s'].includes(k)) setDir(0,1);
  else if (['arrowleft','a'].includes(k)) setDir(-1,0);
  else if (['arrowright','d'].includes(k)) setDir(1,0);
  else return;
  e.preventDefault();
});

document.getElementById('up').onclick = () => setDir(0,-1);
document.getElementById('down').onclick = () => setDir(0,1);
document.getElementById('left').onclick = () => setDir(-1,0);
document.getElementById('right').onclick = () => setDir(1,0);

function startLoop() {
  clearTimeout(tickTimer);
  tick();
}

function tick() {
  if (state !== 'playing') return;
  dir = nextDir;
  const head = {x: snake[0].x + dir.x, y: snake[0].y + dir.y};

  const hitWall = head.x < 0 || head.y < 0 || head.x >= GRID || head.y >= GRID;
  const hitSelf = snake.some(s => s.x===head.x && s.y===head.y);

  if (hitWall || hitSelf) {
    die();
    return;
  }

  snake.unshift(head);

  if (head.x === food.x && head.y === food.y) {
    score++;
    scoreEl.textContent = score;
    for (let i=0;i<10;i++){
      particles.push({x: food.x*CELL+CELL/2, y: food.y*CELL+CELL/2, vx:(Math.random()-0.5)*4, vy:(Math.random()-0.5)*4, life:18, color: food.color});
    }
    placeFood();
    speed = Math.max(55, 120 - score*3);
  } else {
    snake.pop();
  }

  render();
  tickTimer = setTimeout(tick, speed);
}

function die() {
  state = 'dead';
  if (score > best) {
    best = score;
    bestEl.textContent = best;
    try { window.name = 'fruitfrenzy_best=' + best; } catch(e) {}
  }
  overlay.style.display = 'flex';
  title.textContent = 'Score: ' + score;
  sub.textContent = (score === best && score > 0) ? '🎉 New best!' : 'Press a direction to retry';
  render();
}

function drawRoundedCell(x,y,color,inset=1.5,radius=6) {
  const px = x*CELL+inset, py = y*CELL+inset, s = CELL-inset*2;
  ctx.fillStyle = color;
  ctx.beginPath();
  ctx.roundRect(px, py, s, s, radius);
  ctx.fill();
}

function render() {
  ctx.clearRect(0,0,canvas.width,canvas.height);
  // checker background
  for (let y=0;y<GRID;y++){
    for (let x=0;x<GRID;x++){
      ctx.fillStyle = (x+y)%2===0 ? '#fff8ea' : '#fdeecb';
      ctx.fillRect(x*CELL,y*CELL,CELL,CELL);
    }
  }
  // food
  ctx.save();
  ctx.shadowColor = food.color;
  ctx.shadowBlur = 10;
  drawRoundedCell(food.x, food.y, food.color, 3, 10);
  ctx.restore();

  // snake
  snake.forEach((s,i) => {
    const t = i / Math.max(snake.length-1,1);
    const hue = i===0 ? '#5cd65c' : mixColor('#5cd65c', '#3aa53a', t);
    drawRoundedCell(s.x, s.y, i===0 ? '#4bc94b' : hue, 1.5, 7);
  });
  // eyes on head
  if (snake.length) {
    const h = snake[0];
    ctx.fillStyle = '#2b2b2b';
    const cx = h.x*CELL+CELL/2, cy = h.y*CELL+CELL/2;
    const ex = dir.x*4, ey = dir.y*4;
    ctx.beginPath();
    ctx.arc(cx+ex-4+dir.y*4, cy+ey-4-dir.x*4, 2, 0, Math.PI*2);
    ctx.arc(cx+ex+4+dir.y*4, cy+ey+4-dir.x*4, 2, 0, Math.PI*2);
    ctx.fill();
  }

  // particles
  particles.forEach(p => {
    ctx.save();
    ctx.globalAlpha = Math.max(p.life/18,0);
    ctx.fillStyle = p.color;
    ctx.beginPath();
    ctx.arc(p.x,p.y,3,0,Math.PI*2);
    ctx.fill();
    ctx.restore();
    p.x += p.vx; p.y += p.vy; p.life--;
  });
  particles = particles.filter(p => p.life > 0);
  if (particles.length) requestAnimationFrame(render);
}

function mixColor(a,b,t) {
  const pa = parseInt(a.slice(1),16), pb = parseInt(b.slice(1),16);
  const ar=(pa>>16)&255, ag=(pa>>8)&255, ab=pa&255;
  const br=(pb>>16)&255, bg=(pb>>8)&255, bb=pb&255;
  const r = Math.round(ar+(br-ar)*t), g = Math.round(ag+(bg-ag)*t), bch = Math.round(ab+(bb-ab)*t);
  return `rgb(${r},${g},${bch})`;
}

render();
</script>
</body>
</html>
