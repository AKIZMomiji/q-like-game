const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
const commandForm = document.getElementById('commandForm');
const commandInput = document.getElementById('commandInput');
const statusText = document.getElementById('status');
const logEl = document.getElementById('log');

const world = {
  width: canvas.width,
  height: canvas.height,
  gravity: 0.5,
  gravityDirection: 1,
};

const player = {
  x: 70,
  y: 430,
  w: 28,
  h: 28,
  vx: 0,
  vy: 0,
  onGround: false,
};

const start = { x: 70, y: 430 };

const solids = [
  { x: 0, y: 510, w: 960, h: 30, type: 'floor' },
  { x: 0, y: 0, w: 20, h: 540, type: 'wall' },
  { x: 940, y: 0, w: 20, h: 540, type: 'wall' },
  { x: 0, y: 0, w: 960, h: 20, type: 'roof' },
  { x: 130, y: 450, w: 220, h: 18, type: 'platform' },
  { x: 420, y: 390, w: 220, h: 18, type: 'platform' },
  { x: 680, y: 330, w: 180, h: 18, type: 'platform' },
  { x: 650, y: 180, w: 120, h: 18, type: 'platform' },
  { x: 220, y: 245, w: 90, h: 18, type: 'platform' },
  { x: 280, y: 120, w: 120, h: 18, type: 'platform' },
  { x: 325, y: 430, w: 40, h: 14, type: 'bumper' },
];

const switchBox = { x: 830, y: 482, w: 24, h: 24 };
const goal = { x: 900, y: 80, w: 26, h: 36 };
const gate = { x: 770, y: 210, w: 24, h: 170, open: false };

let hasOpenedGate = false;
let message = 'Find the switch, open the gate, and reach the exit.';
let commandHistory = [];

function addLog(text, type = '') {
  const line = document.createElement('div');
  line.className = `log-line ${type}`.trim();
  line.textContent = text;
  logEl.prepend(line);
}

function setStatus(text) {
  statusText.textContent = text;
  message = text;
}

function resetGame() {
  player.x = start.x;
  player.y = start.y;
  player.vx = 0;
  player.vy = 0;
  player.onGround = false;
  gate.open = false;
  hasOpenedGate = false;
  world.gravityDirection = 1;
  world.gravity = 0.5;
  commandHistory = [];
  setStatus('Find the switch, open the gate, and reach the exit.');
  addLog('The chamber resets.', 'warning');
}

function intersects(a, b) {
  return (
    a.x < b.x + b.w &&
    a.x + a.w > b.x &&
    a.y < b.y + b.h &&
    a.y + a.h > b.y
  );
}

function resolvePhysics() {
  player.onGround = false;

  player.x += player.vx;
  for (const block of solids) {
    if (block.type === 'bumper') continue;
    if (intersects(player, block)) {
      if (player.vx > 0) {
        player.x = block.x - player.w;
      } else if (player.vx < 0) {
        player.x = block.x + block.w;
      }
      player.vx = 0;
    }
  }

  player.y += player.vy;
  for (const block of solids) {
    if (block.type === 'bumper') continue;
    if (gate.open && block === gate) continue;
    if (intersects(player, block)) {
      if (player.vy > 0) {
        player.y = block.y - player.h;
        player.vy = 0;
        player.onGround = true;
      } else if (player.vy < 0) {
        player.y = block.y + block.h;
        player.vy = 0;
      }
    }
  }

  if (!gate.open && intersects(player, gate)) {
    const prevX = player.x - player.vx;
    const prevY = player.y - player.vy;

    if (prevY + player.h <= gate.y || prevY >= gate.y + gate.h) {
      if (player.vy > 0) {
        player.y = gate.y - player.h;
        player.vy = 0;
        player.onGround = true;
      } else if (player.vy < 0) {
        player.y = gate.y + gate.h;
        player.vy = 0;
      }
    } else {
      if (player.vx > 0) {
        player.x = gate.x - player.w;
      } else if (player.vx < 0) {
        player.x = gate.x + gate.w;
      }
      player.vx = 0;
    }
  }

  if (player.y > world.height) {
    player.y = start.y;
    player.x = start.x;
    player.vx = 0;
    player.vy = 0;
    setStatus('You fell. Try again.');
    addLog('Fallen back into the room.', 'warning');
  }

  if (player.y + player.h < 0) {
    player.y = 0;
    player.vy = 0;
  }

  if (player.x < 0) player.x = 0;
  if (player.x + player.w > world.width) player.x = world.width - player.w;
}

function update() {
  player.vy += world.gravity * world.gravityDirection;
  if (world.gravityDirection === -1 && player.y + player.h >= world.height) {
    player.vy = 0;
  }
  resolvePhysics();

  const switchActive = intersects(player, switchBox);
  if (switchActive && !hasOpenedGate) {
    setStatus('Switch detected. Type "open" to raise the gate.');
  }

  if (intersects(player, goal) && gate.open) {
    setStatus('Exit reached. Puzzle solved.');
    addLog('Goal reached. You win.', 'success');
    player.vx = 0;
    player.vy = 0;
  }
}

function draw() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  ctx.fillStyle = '#0f1728';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  ctx.fillStyle = '#1d3658';
  for (const block of solids) {
    if (block === gate && !gate.open) {
      continue;
    }
    ctx.fillRect(block.x, block.y, block.w, block.h);
  }

  if (!gate.open) {
    ctx.fillStyle = '#b26d5e';
    ctx.fillRect(gate.x, gate.y, gate.w, gate.h);
  }

  ctx.fillStyle = '#5ad2ff';
  ctx.fillRect(switchBox.x, switchBox.y, switchBox.w, switchBox.h);

  ctx.fillStyle = '#8ef2a2';
  ctx.fillRect(goal.x, goal.y, goal.w, goal.h);

  ctx.fillStyle = '#d7edff';
  ctx.fillRect(player.x, player.y, player.w, player.h);

  ctx.fillStyle = '#96a9c7';
  ctx.fillRect(switchBox.x - 2, switchBox.y + 7, switchBox.w + 4, 10);

  ctx.fillStyle = '#c8efff';
  ctx.fillText('EXIT', goal.x + 5, goal.y + 22);
}

function runCommand(rawInput) {
  const command = rawInput.trim().toLowerCase();

  if (!command) {
    return;
  }

  commandHistory.push(command);

  const tokens = command.split(/\s+/);
  const main = tokens[0];

  switch (main) {
    case 'jump':
      if (player.onGround || Math.abs(player.vy) < 0.2) {
        player.vy = -11 * world.gravityDirection;
        player.onGround = false;
        addLog('You jump.', 'success');
      } else {
        addLog('You cannot jump while airborne.', 'warning');
      }
      break;

    case 'left':
      player.vx = -4;
      addLog('Move left.', 'success');
      break;

    case 'right':
      player.vx = 4;
      addLog('Move right.', 'success');
      break;

    case 'stop':
      player.vx = 0;
      addLog('Movement stopped.', 'success');
      break;

    case 'gravity':
      world.gravityDirection *= -1;
      player.vy = 0;
      addLog(`Gravity reversed to ${world.gravityDirection > 0 ? 'downward' : 'upward'}.`, 'success');
      break;

    case 'open':
      if (intersects(player, switchBox)) {
        gate.open = true;
        hasOpenedGate = true;
        addLog('Gate unlocked.', 'success');
        setStatus('Gate opened. Run for the exit.');
      } else {
        addLog('The switch is too far away.', 'warning');
      }
      break;

    case 'reset':
      resetGame();
      break;

    case 'help':
      addLog('Commands: jump, left, right, stop, gravity, open, reset', 'success');
      break;

    default:
      addLog(`Unknown command: "${command}"`, 'warning');
      return;
  }

  commandInput.value = '';
  commandInput.focus();
}

commandForm.addEventListener('submit', (event) => {
  event.preventDefault();
  runCommand(commandInput.value);
});

function loop() {
  update();
  draw();
  requestAnimationFrame(loop);
}

resetGame();
requestAnimationFrame(loop);
