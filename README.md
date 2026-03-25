<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>CHAMPION .XO • DOOM</title>
<style>
  body { margin:0; overflow:hidden; background:#000; font-family: 'Courier New', monospace; }
  canvas { display:block; image-rendering: pixelated; }
  #hud { position:absolute; top:0; left:0; width:100%; height:100%; pointer-events:none; color:#ff00ff; text-shadow:0 0 15px #ff00ff; }
  #crosshair { position:absolute; top:50%; left:50%; width:30px; height:30px; margin:-15px 0 0 -15px; font-size:36px; line-height:26px; text-align:center; filter: drop-shadow(0 0 8px #ff00ff); }
  #info { position:absolute; bottom:15px; left:15px; font-size:13px; opacity:0.75; }
  #minimap { position:absolute; top:15px; right:15px; border:2px solid #ff00ff; box-shadow:0 0 20px #ff00ff; display:none; }
  #mobile-controls { position:absolute; bottom:0; left:0; width:100%; height:220px; display:none; pointer-events:none; }
  .joystick { position:absolute; bottom:40px; width:140px; height:140px; background:rgba(255,0,255,0.12); border:4px solid #ff00ff; border-radius:50%; pointer-events:auto; box-shadow:0 0 30px #ff00ff; }
  #left-joy { left:30px; }
</style>
</head>
<body>
<canvas id="c"></canvas>

<div id="hud">
  <div id="crosshair">✕</div>
  <div style="position:absolute;top:20px;left:20px;font-size:26px;">HEALTH: <span id="health">100</span></div>
  <div style="position:absolute;top:20px;right:20px;font-size:26px;">SCORE: <span id="score">00000</span></div>
  <div id="info">WASD + MOUSE (click to lock) • Mobile: Left = move • Right = look • Tap right = shoot • M = minimap</div>
</div>

<canvas id="minimap" width="180" height="180"></canvas>

<div id="mobile-controls">
  <div id="left-joy" class="joystick"></div>
</div>

<script>
// =============== CHAMPION .XO DOOM v2 (AI Improved) ===============
const canvas = document.getElementById('c');
const ctx = canvas.getContext('2d', { alpha: true });
let w = canvas.width = window.innerWidth;
let h = canvas.height = window.innerHeight;

window.addEventListener('resize', () => {
  w = canvas.width = window.innerWidth;
  h = canvas.height = window.innerHeight;
});

const map = [
  [1,1,1,1,1,1,1,1,1,1,1,1,1,1,1],
  [1,0,0,0,0,0,0,0,0,0,0,0,0,0,1],
  [1,0,2,0,0,0,0,0,0,0,0,3,0,0,1],
  [1,0,0,0,1,1,1,0,1,1,1,0,0,0,1],
  [1,0,0,0,1,0,0,0,0,0,1,0,0,0,1],
  [1,0,0,0,1,0,0,0,0,0,1,0,0,0,1],
  [1,0,0,0,0,0,0,0,0,0,0,0,0,0,1],
  [1,0,0,0,0,0,0,0,0,0,0,0,0,0,1],
  [1,0,0,0,1,0,0,0,0,0,1,0,0,0,1],
  [1,0,0,0,1,1,1,0,1,1,1,0,0,0,1],
  [1,0,3,0,0,0,0,0,0,0,0,2,0,0,1],
  [1,0,0,0,0,0,0,0,0,0,0,0,0,0,1],
  [1,1,1,1,1,1,1,1,1,1,1,1,1,1,1]
];

const wallColors = {1: '#ff0088', 2: '#00ffff', 3: '#ffff00'};

let player = {
  x: 2.5, y: 2.5,
  dirX: -1, dirY: 0,
  planeX: 0, planeY: 0.66,   // camera plane (controls FOV)
  health: 100,
  score: 0
};

let enemies = [
  {x: 7.5, y: 3.5, health: 3, alive: true, bob: 0},
  {x: 11.5, y: 9.5, health: 3, alive: true, bob: 0}
];

let keys = {};
let isPointerLocked = false;
let minimapVisible = false;

document.addEventListener('keydown', e => {
  keys[e.key.toLowerCase()] = true;
  if (e.key.toLowerCase() === 'm') minimapVisible = !minimapVisible;
});
document.addEventListener('keyup', e => keys[e.key.toLowerCase()] = false);

canvas.addEventListener('click', () => {
  if (!isPointerLocked) canvas.requestPointerLock();
  shoot();
});

document.addEventListener('pointerlockchange', () => {
  isPointerLocked = document.pointerLockElement === canvas;
});

document.addEventListener('mousemove', e => {
  if (!isPointerLocked) return;
  const sensitivity = 0.0025;
  const oldDirX = player.dirX;
  player.dirX = player.dirX * Math.cos(-e.movementX * sensitivity) - player.dirY * Math.sin(-e.movementX * sensitivity);
  player.dirY = oldDirX * Math.sin(-e.movementX * sensitivity) + player.dirY * Math.cos(-e.movementX * sensitivity);
  const oldPlaneX = player.planeX;
  player.planeX = player.planeX * Math.cos(-e.movementX * sensitivity) - player.planeY * Math.sin(-e.movementX * sensitivity);
  player.planeY = oldPlaneX * Math.sin(-e.movementX * sensitivity) + player.planeY * Math.cos(-e.movementX * sensitivity);
});

// Mobile touch controls (improved)
let touchMove = {active: false, baseX:0, baseY:0};
let touchLook = {active: false, prevX:0};

canvas.addEventListener('touchstart', e => {
  e.preventDefault();
  for (let t of e.changedTouches) {
    if (t.clientX < w * 0.5) {
      touchMove.active = true;
      touchMove.baseX = t.clientX;
      touchMove.baseY = t.clientY;
    } else {
      touchLook.active = true;
      touchLook.prevX = t.clientX;
      shoot(); // tap to shoot on right side
    }
  }
});

canvas.addEventListener('touchmove', e => {
  e.preventDefault();
  for (let t of e.touches) {
    if (touchMove.active && t.clientX < w * 0.5) {
      const dx = (t.clientX - touchMove.baseX) * 0.006;
      const dy = (t.clientY - touchMove.baseY) * 0.006;
      const moveSpeed = 0.09;
      let newX = player.x + player.dirX * dy * moveSpeed + player.dirY * dx * 0.6;
      let newY = player.y + player.dirY * dy * moveSpeed - player.dirX * dx * 0.6;
      if (map[Math.floor(newY)][Math.floor(newX)] === 0) {
        player.x = newX; player.y = newY;
      }
      touchMove.baseX = t.clientX;
      touchMove.baseY = t.clientY;
    }
    if (touchLook.active && t.clientX > w * 0.5) {
      const deltaX = (t.clientX - touchLook.prevX) * 0.006;
      const oldDirX = player.dirX;
      player.dirX = player.dirX * Math.cos(-deltaX) - player.dirY * Math.sin(-deltaX);
      player.dirY = oldDirX * Math.sin(-deltaX) + player.dirY * Math.cos(-deltaX);
      const oldPlaneX = player.planeX;
      player.planeX = player.planeX * Math.cos(-deltaX) - player.planeY * Math.sin(-deltaX);
      player.planeY = oldPlaneX * Math.sin(-deltaX) + player.planeY * Math.cos(-deltaX);
      touchLook.prevX = t.clientX;
    }
  }
});

canvas.addEventListener('touchend', () => {
  touchMove.active = false;
  touchLook.active = false;
});

function shoot() {
  ctx.fillStyle = 'rgba(255, 240, 180, 0.55)';
  ctx.fillRect(0, 0, w, h);

  enemies.forEach(en => {
    if (!en.alive) return;
    const dx = en.x - player.x;
    const dy = en.y - player.y;
    const dist = Math.hypot(dx, dy);
    const angleToEnemy = Math.atan2(dy, dx);
    const playerAngle = Math.atan2(player.dirY, player.dirX);
    if (Math.abs(angleToEnemy - playerAngle) < 0.25 && dist < 8) {
      en.health--;
      if (en.health <= 0) {
        en.alive = false;
        player.score += 250;
      } else {
        player.score += 50;
      }
      document.getElementById('score').textContent = player.score.toString().padStart(5, '0');
    }
  });
}

function drawMinimap() {
  const mm = document.getElementById('minimap');
  if (!minimapVisible) { mm.style.display = 'none'; return; }
  mm.style.display = 'block';
  const mctx = mm.getContext('2d');
  const scale = 12;
  mctx.clearRect(0,0,180,180);
  for (let y=0; y<map.length; y++) {
    for (let x=0; x<map[y].length; x++) {
      if (map[y][x] > 0) {
        mctx.fillStyle = wallColors[map[y][x]] || '#ff00ff';
        mctx.fillRect(x*scale, y*scale, scale, scale);
      }
    }
  }
  // player
  mctx.fillStyle = '#00ffcc';
  mctx.fillRect(player.x*scale - 3, player.y*scale - 3, 6, 6);
}

function draw() {
  // Neon sky
  const sky = ctx.createLinearGradient(0,0,0,h*0.55);
  sky.addColorStop(0, '#1a0033');
  sky.addColorStop(1, '#440066');
  ctx.fillStyle = sky;
  ctx.fillRect(0,0,w,h*0.55);

  // Floor with subtle grid
  ctx.fillStyle = '#0f0022';
  ctx.fillRect(0, h*0.5, w, h*0.5);

  const numRays = Math.floor(w);
  const zBuffer = new Array(numRays);

  for (let x = 0; x < numRays; x++) {
    const cameraX = 2 * x / w - 1;
    const rayDirX = player.dirX + player.planeX * cameraX;
    const rayDirY = player.dirY + player.planeY * cameraX;

    let mapX = Math.floor(player.x);
    let mapY = Math.floor(player.y);

    const deltaDistX = rayDirX === 0 ? Infinity : Math.abs(1 / rayDirX);
    const deltaDistY = rayDirY === 0 ? Infinity : Math.abs(1 / rayDirY);

    let stepX = rayDirX < 0 ? -1 : 1;
    let stepY = rayDirY < 0 ? -1 : 1;
    let sideDistX = rayDirX < 0 ? (player.x - mapX) * deltaDistX : (mapX + 1 - player.x) * deltaDistX;
    let sideDistY = rayDirY < 0 ? (player.y - mapY) * deltaDistY : (mapY + 1 - player.y) * deltaDistY;

    let hit = false, side = 0;
    while (!hit) {
      if (sideDistX < sideDistY) {
        sideDistX += deltaDistX;
        mapX += stepX;
        side = 0;
      } else {
        sideDistY += deltaDistY;
        mapY += stepY;
        side = 1;
      }
      if (map[mapY][mapX] > 0) hit = true;
    }

    const perpWallDist = side === 0 ? sideDistX - deltaDistX : sideDistY - deltaDistY;
    const lineHeight = Math.floor(h / perpWallDist);

    let drawStart = Math.max(0, -lineHeight / 2 + h / 2);
    let drawEnd = Math.min(h, lineHeight / 2 + h / 2);

    let color = wallColors[map[mapY][mapX]] || '#ff00ff';
    if (side === 1) {
      const c = parseInt(color.slice(1), 16);
      color = `rgb(${((c>>16)&255)*0.55|0},${((c>>8)&255)*0.55|0},${(c&255)*0.55|0})`;
    }

    ctx.shadowBlur = 22;
    ctx.shadowColor = color;
    ctx.fillStyle = color;
    ctx.fillRect(x, drawStart, 1, drawEnd - drawStart);
    ctx.shadowBlur = 0;

    zBuffer[x] = perpWallDist;
  }

  // Draw enemies (improved with shading)
  enemies.forEach(en => {
    if (!en.alive) return;
    en.bob = Math.sin(Date.now() / 180) * 0.12;

    const spriteX = en.x - player.x;
    const spriteY = en.y - player.y;
    const invDet = 1 / (player.planeX * player.dirY - player.dirX * player.planeY);
    const transformX = invDet * (player.dirY * spriteX - player.dirX * spriteY);
    const transformY = invDet * (-player.planeY * spriteX + player.planeX * spriteY);

    if (transformY < 0.15) return;

    const spriteScreenX = Math.floor((w / 2) * (1 + transformX / transformY));
    let spriteHeight = Math.abs(Math.floor(h / transformY * 1.1));
    const spriteWidth = Math.abs(Math.floor(h / transformY * 0.85));

    const drawStartY = Math.max(0, -spriteHeight/2 + h/2 + en.bob * 40);
    const drawEndY = Math.min(h-1, spriteHeight/2 + h/2 + en.bob * 40);

    const drawStartX = Math.max(0, spriteScreenX - spriteWidth/2);
    const drawEndX = Math.min(w-1, spriteScreenX + spriteWidth/2);

    const brightness = Math.max(0.4, 1 - transformY * 0.12);
    ctx.fillStyle = `rgba(255, 30, 80, ${brightness})`;
    ctx.shadowBlur = 35;
    ctx.shadowColor = '#ff0066';

    for (let stripe = drawStartX; stripe < drawEndX; stripe++) {
      if (transformY < zBuffer[stripe]) {
        ctx.fillRect(stripe, drawStartY, 2, drawEndY - drawStartY);
      }
    }
    ctx.shadowBlur = 0;
  });

  // Subtle scanlines for retro feel
  ctx.fillStyle = 'rgba(255,255,255,0.03)';
  for (let i = 0; i < h; i += 3) ctx.fillRect(0, i, w, 1);

  drawMinimap();
}

function update() {
  const moveSpeed = 0.085;
  const rotSpeed = 0.045;

  if (keys['w']) {
    let nx = player.x + player.dirX * moveSpeed;
    let ny = player.y + player.dirY * moveSpeed;
    if (map[Math.floor(ny)][Math.floor(nx)] === 0) { player.x = nx; player.y = ny; }
  }
  if (keys['s']) {
    let nx = player.x - player.dirX * moveSpeed;
    let ny = player.y - player.dirY * moveSpeed;
    if (map[Math.floor(ny)][Math.floor(nx)] === 0) { player.x = nx; player.y = ny; }
  }
  if (keys['a']) {
    let nx = player.x - player.dirY * moveSpeed * 0.75;
    let ny = player.y + player.dirX * moveSpeed * 0.75;
    if (map[Math.floor(ny)][Math.floor(nx)] === 0) { player.x = nx; player.y = ny; }
  }
  if (keys['d']) {
    let nx = player.x + player.dirY * moveSpeed * 0.75;
    let ny = player.y - player.dirX * moveSpeed * 0.75;
    if (map[Math.floor(ny)][Math.floor(nx)] === 0) { player.x = nx; player.y = ny; }
  }

  // Enemy proximity damage
  enemies.forEach(en => {
    if (en.alive) {
      const dist = Math.hypot(en.x - player.x, en.y - player.y);
      if (dist < 1.2) player.health -= 0.08;
    }
  });

  if (player.health <= 0) {
    player.health = 0;
    alert("GAME OVER — You fought well, Champion.");
    location.reload();
  }

  // Win condition
  if (enemies.every(en => !en.alive)) {
    alert(`VICTORY! Final Score: ${player.score} — Eternal Champion!`);
  }
}

function gameLoop() {
  update();
  draw();
  document.getElementById('health').textContent = Math.floor(player.health);
  requestAnimationFrame(gameLoop);
}

gameLoop();

// Show mobile UI on touch devices
if ('ontouchstart' in window) {
  document.getElementById('mobile-controls').style.display = 'block';
  document.getElementById('info').innerHTML += '<br>Tap screen to lock view on desktop';
}
</script>
</body>
</html>