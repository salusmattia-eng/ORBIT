
<!DOCTYPE html>
<html lang="it">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>ORBIT</title>
<style>
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    overflow: hidden;
    background: #080b18;
    color: white;
    font-family: Arial, sans-serif;
}

canvas {
    display: block;
    width: 100vw;
    height: 100vh;
    touch-action: manipulation;
}

#hud {
    position: absolute;
    top: 25px;
    left: 25px;
    right: 25px;
    display: flex;
    justify-content: space-between;
    pointer-events: none;
}

#logo {
    color: #9eeaff;
    letter-spacing: 7px;
    font-weight: bold;
    font-size: 20px;
}

#score {
    font-size: 28px;
    font-weight: bold;
}

#screen {
    position: absolute;
    inset: 0;
    display: flex;
    justify-content: center;
    align-items: center;
    background: #080b18bb;
}

.panel {
    text-align: center;
    background: #111b34;
    padding: 40px;
    border: 1px solid #31517b;
    border-radius: 22px;
    width: min(90%, 380px);
}

h1 {
    font-size: 48px;
    letter-spacing: 10px;
    margin: 15px;
    color: #8deaff;
}

p {
    color: #aabbd8;
    line-height: 1.6;
    margin-bottom: 25px;
}

button {
    background: linear-gradient(90deg, #72dcff, #9af5d2);
    border: none;
    border-radius: 30px;
    padding: 15px 45px;
    font-weight: bold;
    font-size: 15px;
    cursor: pointer;
}

.hidden {
    display: none !important;
}

#hint {
    position: absolute;
    bottom: 25px;
    width: 100%;
    text-align: center;
    color: #788baa;
    font-size: 11px;
    letter-spacing: 2px;
    pointer-events: none;
}
</style>
</head>

<body>

<canvas id="game"></canvas>

<div id="hud">
    <div id="logo">ORBIT</div>
    <div id="score">0</div>
</div>

<div id="hint">TOCCA PER INVERTIRE LA DIREZIONE</div>

<div id="screen">
    <div class="panel">
        <h1>ORBIT</h1>
        <p>
            Una sola orbita.<br>
            Un movimento infinito.<br>
            Evita gli ostacoli e resisti il più possibile.
        </p>
        <button id="play">GIOCA</button>
    </div>
</div>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

const scoreText = document.getElementById("score");
const screen = document.getElementById("screen");
const playButton = document.getElementById("play");

let W, H, cx, cy, radius;

function resize() {
    const dpr = Math.min(devicePixelRatio || 1, 2);
    W = innerWidth;
    H = innerHeight;

    canvas.width = W * dpr;
    canvas.height = H * dpr;
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0);

    cx = W / 2;
    cy = H / 2;
    radius = Math.min(W, H) * 0.28;
}

addEventListener("resize", resize);
resize();

let angle;
let direction;
let obstacles;
let score;
let elapsed;
let spawnTimer;
let playing = false;
let lastTime = 0;

const TAU = Math.PI * 2;

function random(min, max) {
    return Math.random() * (max - min) + min;
}

function difference(a, b) {
    return Math.atan2(Math.sin(a - b), Math.cos(a - b));
}

function startGame() {
    angle = -Math.PI / 2;
    direction = 1;
    obstacles = [];
    score = 0;
    elapsed = 0;
    spawnTimer = 1;

    scoreText.textContent = "0";
    screen.classList.add("hidden");

    playing = true;
    lastTime = performance.now();

    requestAnimationFrame(loop);
}

function reverse() {
    if (playing) {
        direction *= -1;
    }
}

canvas.addEventListener("pointerdown", reverse);

playButton.addEventListener("click", startGame);

addEventListener("keydown", e => {
    if (e.code === "Space") {
        e.preventDefault();
        reverse();
    }
});

function gameOver() {
    playing = false;

    screen.innerHTML = `
        <div class="panel">
            <h1>ORBIT</h1>
            <p>PARTITA TERMINATA<br>Il tuo punteggio: ${score}</p>
            <button onclick="startGame()">RIPROVA</button>
        </div>
    `;

    screen.classList.remove("hidden");
}

function createObstacle() {
    const a = angle + direction * random(1.2, 2.4);

    obstacles.push({
        angle: (a + TAU) % TAU,
        width: random(0.19, 0.28)
    });
}

function update(dt) {
    elapsed += dt;

    const speed = 1.3 + Math.min(elapsed * 0.018, 1.3);

    angle += direction * speed * dt;
    angle = (angle + TAU) % TAU;

    spawnTimer -= dt;

    if (spawnTimer <= 0) {
        createObstacle();
        spawnTimer = Math.max(0.65, 1.5 - elapsed * 0.005) + random(0, 0.4);
    }

    score = Math.floor(elapsed * 10);
    scoreText.textContent = score;

    for (const obstacle of obstacles) {
        const d = Math.abs(difference(angle, obstacle.angle));

        if (d < obstacle.width / 2 + 0.035) {
            gameOver();
            return;
        }
    }
}

function draw() {
    ctx.clearRect(0, 0, W, H);

    // Sfondo stellato
    for (let i = 0; i < 65; i++) {
        const x = (i * 197.3) % W;
        const y = (i * 113.7) % H;

        ctx.fillStyle = "rgba(150,190,255,0.4)";
        ctx.beginPath();
        ctx.arc(x, y, 1, 0, TAU);
        ctx.fill();
    }

    // Alone centrale
    const glow = ctx.createRadialGradient(
        cx, cy, 5,
        cx, cy, radius
    );

    glow.addColorStop(0, "rgba(50,120,220,0.25)");
    glow.addColorStop(1, "rgba(10,20,50,0)");

    ctx.fillStyle = glow;
    ctx.beginPath();
    ctx.arc(cx, cy, radius, 0, TAU);
    ctx.fill();

    // Orbita fissa
    ctx.beginPath();
    ctx.arc(cx, cy, radius, 0, TAU);
    ctx.strokeStyle = "rgba(120,180,255,0.4)";
    ctx.lineWidth = 2;
    ctx.stroke();

    // Nucleo centrale
    ctx.shadowBlur = 30;
    ctx.shadowColor = "#4bbcff";

    const core = ctx.createRadialGradient(
        cx - 10, cy - 10, 2,
        cx, cy, radius * 0.2
    );

    core.addColorStop(0, "#e4ffff");
    core.addColorStop(0.3, "#65caff");
    core.addColorStop(1, "#17356b");

    ctx.fillStyle = core;
    ctx.beginPath();
    ctx.arc(cx, cy, radius * 0.19, 0, TAU);
    ctx.fill();

    ctx.shadowBlur = 0;

    // Ostacoli
    for (const obstacle of obstacles) {
        ctx.beginPath();

        ctx.arc(
            cx, cy, radius,
            obstacle.angle - obstacle.width / 2,
            obstacle.angle + obstacle.width / 2
        );

        ctx.strokeStyle = "#ff527e";
        ctx.lineWidth = 10;
        ctx.lineCap = "round";

        ctx.shadowBlur = 18;
        ctx.shadowColor = "#ff527e";
        ctx.stroke();
        ctx.shadowBlur = 0;
    }

    // Pallina del giocatore
    const px = cx + Math.cos(angle) * radius;
    const py = cy + Math.sin(angle) * radius;

    ctx.shadowBlur = 25;
    ctx.shadowColor = "#6deaff";

    ctx.fillStyle = "#ffffff";
    ctx.beginPath();
    ctx.arc(px, py, 8, 0, TAU);
    ctx.fill();

    ctx.shadowBlur = 0;

    ctx.beginPath();
    ctx.arc(px, py, 13, 0, TAU);
    ctx.strokeStyle = "rgba(100,230,255,0.5)";
    ctx.lineWidth = 2;
    ctx.stroke();
}

function loop(time) {
    if (!playing) return;

    const dt = Math.min((time - lastTime) / 1000, 0.035);
    lastTime = time;

    update(dt);
    draw();

    if (playing) {
        requestAnimationFrame(loop);
    }
}

draw();
</script>

</body>
</html>
