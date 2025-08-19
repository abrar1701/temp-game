# temp-game

<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Snake — HTML Canvas + Touch Buttons</title>
  <style>
    :root {
      --bg: #0f172a;       /* slate-900 */
      --panel: #111827cc;  /* gray-900/80 */
      --text: #e5e7eb;     /* gray-200 */
      --accent: #22c55e;   /* green-500 */
      --accent-2: #16a34a; /* green-600 */
      --danger: #f43f5e;   /* rose-500 */
      --grid: #1f2937;     /* gray-800 */
    }
    * { box-sizing: border-box; }
    html, body { height: 100%; }
    body {
      margin: 0; display: grid; place-items: center; height: 100%;
      background: radial-gradient(1200px 800px at 90% -10%, #1e293b 5%, var(--bg) 60%);
      color: var(--text); font-family: system-ui, -apple-system, Segoe UI, Roboto, Ubuntu, Cantarell, Noto Sans, "Helvetica Neue", Arial, "Apple Color Emoji", "Segoe UI Emoji";
    }
    .wrap {
      width: min(92vw, 820px);
      max-width: 820px;
      display: grid;
      gap: 14px;
      grid-template-columns: 1fr;
      padding: 16px;
    }
    header { display:flex; align-items:center; justify-content:space-between; }
    .title { font-weight: 800; letter-spacing:.3px; font-size: clamp(18px, 2.4vw, 24px); }
    .scorebox { display:flex; gap: 10px; align-items:center; }
    .pill { background: var(--panel); padding: 8px 12px; border-radius: 999px; font-variant-numeric: tabular-nums; box-shadow: 0 6px 20px #00000044 inset, 0 1px 0 #ffffff0a; }
    .canvas-panel {
      background: linear-gradient(180deg, #0b1222, #0a1020);
      border-radius: 18px; padding: 14px; box-shadow: 0 12px 40px #0006;
      border: 1px solid #ffffff12;
    }
    canvas { width: 100%; height: auto; display:block; border-radius: 12px; background: #0a0f1d; }
    .controls { display:grid; grid-template-columns: 1fr 1fr; gap: 12px; }
    .btns {
      display:grid; grid-template-columns: 70px 70px 70px; grid-template-rows: 70px 70px 70px; gap:10px;
      justify-content:center; align-content:center; place-self:center; 
      touch-action: none; user-select: none;
    }
    .dpad-btn, .action-btn {
      background: var(--panel);
      color: var(--text);
      border: 1px solid #ffffff1a;
      box-shadow: 0 2px 0 #0009, 0 10px 24px #0006;
      border-radius: 16px;
      display:grid; place-items:center;
      font-weight: 700; font-size: 16px;
      cursor: pointer;
      -webkit-tap-highlight-color: transparent;
      transition: transform .04s ease, background .15s ease;
    }
    .dpad-btn:active, .action-btn:active { transform: translateY(1px) scale(.98); }
    .dpad-btn svg, .action-btn svg { width: 22px; height: 22px; }
    .dpad-btn[aria-pressed="true"] { outline: 2px solid var(--accent); }
    .spacer { visibility: hidden; }

    .actions {
      display:grid; grid-template-columns: repeat(3, minmax(0,1fr)); gap:10px; align-content:center; 
    }
    .action-btn.start { background: linear-gradient(180deg, var(--accent), var(--accent-2)); color:#031b0b; }
    .action-btn.pause { background: #2a3348; }
    .action-btn.reset { background: linear-gradient(180deg, #f87171, var(--danger)); }

    footer { opacity:.8; font-size: 12px; text-align:center; }
    .help { line-height: 1.3; }

    /* Responsive tweaks */
    @media (max-width: 720px) {
      .controls { grid-template-columns: 1fr; }
      .btns { grid-template-columns: 64px 64px 64px; grid-template-rows: 64px 64px 64px; }
      .actions { grid-template-columns: repeat(3, 1fr); }
    }
  </style>
</head>
<body>
  <div class="wrap">
    <header>
      <div class="title">🐍 Snake</div>
      <div class="scorebox">
        <div class="pill">Score: <span id="score">0</span></div>
        <div class="pill">Best: <span id="best">0</span></div>
        <div class="pill" title="Game speed (lower = faster)">Lvl: <span id="level">1</span></div>
      </div>
    </header>

    <div class="canvas-panel">
      <canvas id="board" width="640" height="640" aria-label="Snake game board" role="img"></canvas>
    </div>

    <section class="controls" aria-label="Controls">
      <div class="btns" aria-label="D‑pad">
        <button class="dpad-btn spacer" disabled></button>
        <button class="dpad-btn" id="btn-up" aria-label="Up" aria-pressed="false">▲</button>
        <button class="dpad-btn spacer" disabled></button>

        <button class="dpad-btn" id="btn-left" aria-label="Left" aria-pressed="false">◀</button>
        <button class="dpad-btn" id="btn-center" disabled>●</button>
        <button class="dpad-btn" id="btn-right" aria-label="Right" aria-pressed="false">▶</button>

        <button class="dpad-btn spacer" disabled></button>
        <button class="dpad-btn" id="btn-down" aria-label="Down" aria-pressed="false">▼</button>
        <button class="dpad-btn spacer" disabled></button>
      </div>

      <div class="actions" aria-label="Game actions">
        <button class="action-btn start" id="btn-start" aria-label="Start / Resume">Start</button>
        <button class="action-btn pause" id="btn-pause" aria-label="Pause">Pause</button>
        <button class="action-btn reset" id="btn-reset" aria-label="Reset">Reset</button>
        <div class="help" style="grid-column: 1/-1; opacity:.8;">
          Tip: Use Arrow Keys / WASD on keyboard. On touch, tap the D‑pad. Avoid reversing direction instantly.
        </div>
      </div>
    </section>

    <footer>Made with canvas. No libs. Works offline.</footer>
  </div>

  <script>
    // --- Config ---
    const COLS = 20, ROWS = 20;            // grid size
    const CELL = 32;                        // pixel size at 1x scale (canvas uses DPR scaling below)
    const BASE_TICK_MS = 140;               // starting speed (lower = faster)
    const SPEEDUP_EVERY = 5;                // levels up every N foods
    const SPEEDUP_FACTOR = 0.92;            // how much faster each level

    // --- State ---
    const board = document.getElementById('board');
    const ctx = board.getContext('2d');
    const scoreEl = document.getElementById('score');
    const bestEl = document.getElementById('best');
    const levelEl = document.getElementById('level');

    let snake, dir, nextDir, food, playing, lastTime, acc, tickMs, score, level, foodsEaten;

    const bestKey = 'snake_best_score_v1';
    bestEl.textContent = localStorage.getItem(bestKey) || 0;

    function reset() {
      snake = [ {x: Math.floor(COLS/2), y: Math.floor(ROWS/2)}, {x: Math.floor(COLS/2)-1, y: Math.floor(ROWS/2)} ];
      dir = {x: 1, y: 0};
      nextDir = {x: 1, y: 0};
      placeFood();
      playing = false;
      lastTime = 0;
      acc = 0;
      tickMs = BASE_TICK_MS;
      score = 0;
      level = 1;
      foodsEaten = 0;
      updateHud();
      draw();
    }

    function updateHud() {
      scoreEl.textContent = score;
      levelEl.textContent = level;
      const best = Math.max(parseInt(localStorage.getItem(bestKey) || '0', 10), score);
      bestEl.textContent = best;
      if (score > (localStorage.getItem(bestKey)|0)) {
        localStorage.setItem(bestKey, String(score));
      }
    }

    function placeFood() {
      do {
        food = { x: Math.floor(Math.random()*COLS), y: Math.floor(Math.random()*ROWS) };
      } while (snake.some(s => s.x === food.x && s.y === food.y));
    }

    // High‑DPI scaling for crisp pixels
    function resizeCanvas() {
      const dpr = Math.max(1, Math.min(window.devicePixelRatio || 1, 2));
      const displayW = COLS * CELL;
      const displayH = ROWS * CELL;
      board.style.width = '100%';
      const rect = board.getBoundingClientRect();
      const scale = Math.min(rect.width / displayW, 1);
      board.width = Math.floor(displayW * dpr * scale);
      board.height = Math.floor(displayH * dpr * scale);
      ctx.setTransform(board.width / (COLS*CELL), 0, 0, board.height / (ROWS*CELL), 0, 0);
      draw();
    }
    window.addEventListener('resize', resizeCanvas);

    function drawGrid() {
      ctx.fillStyle = '#0a0f1d';
      ctx.fillRect(0,0, COLS*CELL, ROWS*CELL);
      ctx.strokeStyle = '#0f1a2e';
      ctx.lineWidth = 1;
      for (let x=0; x<=COLS; x++) {
        ctx.beginPath(); ctx.moveTo(x*CELL+0.5, 0); ctx.lineTo(x*CELL+0.5, ROWS*CELL); ctx.stroke();
      }
      for (let y=0; y<=ROWS; y++) {
        ctx.beginPath(); ctx.moveTo(0, y*CELL+0.5); ctx.lineTo(COLS*CELL, y*CELL+0.5); ctx.stroke();
      }
    }

    function drawSnake() {
      ctx.fillStyle = '#22c55e';
      const head = snake[0];
      for (let i=snake.length-1; i>=0; i--) {
        const s = snake[i];
        const r = i === 0 ? 8 : 6;
        roundedRect(s.x*CELL+3, s.y*CELL+3, CELL-6, CELL-6, r);
        ctx.fill();
      }
      // eyes on head
      ctx.fillStyle = '#052e16';
      ctx.beginPath(); ctx.arc(head.x*CELL + CELL*0.35, head.y*CELL + CELL*0.38, 3, 0, Math.PI*2); ctx.fill();
      ctx.beginPath(); ctx.arc(head.x*CELL + CELL*0.65, head.y*CELL + CELL*0.38, 3, 0, Math.PI*2); ctx.fill();
    }

    function drawFood() {
      ctx.fillStyle = '#f59e0b';
      roundedRect(food.x*CELL+4, food.y*CELL+4, CELL-8, CELL-8, 10); ctx.fill();
      ctx.fillStyle = '#b45309';
      ctx.beginPath(); ctx.arc(food.x*CELL + CELL*0.5, food.y*CELL + CELL*0.5, 4, 0, Math.PI*2); ctx.fill();
    }

    function roundedRect(x,y,w,h,r) {
      ctx.beginPath();
      ctx.moveTo(x+r, y);
      ctx.arcTo(x+w, y, x+w, y+h, r);
      ctx.arcTo(x+w, y+h, x, y+h, r);
      ctx.arcTo(x, y+h, x, y, r);
      ctx.arcTo(x, y, x+w, y, r);
      ctx.closePath();
    }

    function draw() {
      drawGrid();
      drawFood();
      drawSnake();
    }

    function gameLoop(ts) {
      if (!playing) return;
      if (!lastTime) lastTime = ts;
      const dt = ts - lastTime; lastTime = ts; acc += dt;
      // update direction once per tick to avoid instant reverse
      while (acc >= tickMs) {
        acc -= tickMs;
        step();
      }
      draw();
      requestAnimationFrame(gameLoop);
    }

    function step() {
      // apply nextDir if not reversing
      if (!(nextDir.x === -dir.x && nextDir.y === -dir.y)) dir = nextDir;
      const head = { x: snake[0].x + dir.x, y: snake[0].y + dir.y };
      // wall collision
      if (head.x < 0 || head.y < 0 || head.x >= COLS || head.y >= ROWS) {
        return gameOver();
      }
      // self collision
      if (snake.some(s => s.x === head.x && s.y === head.y)) {
        return gameOver();
      }
      snake.unshift(head);
      let ate = (head.x === food.x && head.y === food.y);
      if (ate) {
        score += 1; foodsEaten += 1; updateHud(); placeFood();
        if (foodsEaten % SPEEDUP_EVERY === 0) {
          tickMs = Math.max(60, tickMs * SPEEDUP_FACTOR);
          level += 1; levelEl.textContent = level;
        }
      } else {
        snake.pop();
      }
    }

    function gameOver() {
      playing = false;
      // flash effect
      const flash = 3;
      let n = 0;
      const id = setInterval(() => {
        n++;
        ctx.save();
        ctx.globalAlpha = 0.25 + (n % 2 ? 0.35 : 0);
        ctx.fillStyle = '#f43f5e';
        ctx.fillRect(0,0, COLS*CELL, ROWS*CELL);
        ctx.restore();
        if (n > flash*2) { clearInterval(id); draw(); drawGameOverText(); }
      }, 80);
    }

    function drawGameOverText() {
      const msg = 'Game Over';
      ctx.save();
      ctx.fillStyle = '#000c';
      ctx.fillRect(0,0, COLS*CELL, ROWS*CELL);
      ctx.fillStyle = '#fff';
      ctx.font = 'bold 32px system-ui, sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText(msg, COLS*CELL/2, ROWS*CELL/2 - 10);
      ctx.font = '16px system-ui, sans-serif';
      ctx.fillText('Press Start to play again', COLS*CELL/2, ROWS*CELL/2 + 20);
      ctx.restore();
    }

    function start() {
      if (playing) return;
      playing = true; lastTime = 0; acc = 0; requestAnimationFrame(gameLoop);
    }
    function pause() { playing = false; }

    // --- Inputs: keyboard ---
    window.addEventListener('keydown', (e) => {
      const k = e.key.toLowerCase();
      if (['arrowup','w'].includes(k)) setDir(0,-1);
      else if (['arrowdown','s'].includes(k)) setDir(0,1);
      else if (['arrowleft','a'].includes(k)) setDir(-1,0);
      else if (['arrowright','d'].includes(k)) setDir(1,0);
      else if (k === ' ') { playing ? pause() : start(); }
    });

    // --- Inputs: on‑screen buttons ---
    const btnUp = document.getElementById('btn-up');
    const btnDown = document.getElementById('btn-down');
    const btnLeft = document.getElementById('btn-left');
    const btnRight = document.getElementById('btn-right');
    const btnStart = document.getElementById('btn-start');
    const btnPause = document.getElementById('btn-pause');
    const btnReset = document.getElementById('btn-reset');

    function bindPress(el, fn) {
      const press = (ev) => { ev.preventDefault(); el.setAttribute('aria-pressed','true'); fn(); };
      const release = (ev) => { ev.preventDefault(); el.setAttribute('aria-pressed','false'); };
      el.addEventListener('mousedown', press);
      el.addEventListener('mouseup', release);
      el.addEventListener('mouseleave', release);
      el.addEventListener('touchstart', press, {passive:false});
      el.addEventListener('touchend', release, {passive:false});
      el.addEventListener('touchcancel', release, {passive:false});
    }

    bindPress(btnUp,   () => setDir(0,-1));
    bindPress(btnDown, () => setDir(0,1));
    bindPress(btnLeft, () => setDir(-1,0));
    bindPress(btnRight,() => setDir(1,0));

    btnStart.addEventListener('click', start);
    btnPause.addEventListener('click', pause);
    btnReset.addEventListener('click', () => { reset(); });

    function setDir(x,y) {
      nextDir = {x,y};
      if (!playing) start();
    }

    // init
    reset();
    resizeCanvas();
  </script>
</body>
</html>
