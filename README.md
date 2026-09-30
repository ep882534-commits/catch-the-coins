from pathlib import Path

html = r'''<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Atrapa las Monedas — Aventura</title>
<style>
*{box-sizing:border-box}
body{
  margin:0;padding:20px;
  font-family:Arial,sans-serif;
  background:#0b1120;color:#fff;
  display:flex;justify-content:center;
}
.game{width:min(760px,100%)}
h1{text-align:center;margin:5px 0 16px}
.hud{display:flex;flex-wrap:wrap;gap:8px;margin-bottom:10px;align-items:center;justify-content:center}
.box,button{
  padding:9px 12px;border:1px solid #475569;border-radius:10px;
  background:#1e293b;color:#fff;font-weight:bold
}
button{cursor:pointer}
button:hover{background:#334155}
.progress{height:9px;background:#334155;border-radius:20px;overflow:hidden;margin-bottom:10px}
#progress{height:100%;width:100%;background:#facc15;transition:width .2s}
.arena{
  position:relative;min-height:430px;overflow:hidden;
  border:2px solid #475569;border-radius:20px;
  background:linear-gradient(#172554,#0f172a);
}
.obj{
  position:absolute;border:0;cursor:pointer;user-select:none;
  touch-action:manipulation;padding:0;background:transparent;
}
.coin{
  width:50px;height:50px;border-radius:50%;
  background:#facc15;font-size:27px;
  box-shadow:0 3px 12px #0008;
}
.gem{
  width:48px;height:48px;border-radius:12px;
  background:#60a5fa;font-size:25px;transform:rotate(45deg);
}
.gem span{display:block;transform:rotate(-45deg)}
.enemy{
  width:54px;height:54px;border-radius:50%;
  background:#ef4444;font-size:27px;
  box-shadow:0 3px 12px #0008;
}
.overlay{
  position:absolute;inset:0;display:flex;align-items:center;
  justify-content:center;text-align:center;padding:24px;
  background:#020617dd;z-index:10;
}
.panel{
  width:min(460px,100%);padding:28px;
  border:1px solid #475569;border-radius:18px;background:#1e293b;
}
.big{font-size:32px;font-weight:800;margin-bottom:10px}
.hidden{display:none}
.small{font-size:14px;color:#cbd5e1}
</style>
</head>

<body>
<main class="game">
<h1>🪙 Atrapa las Monedas — Aventura</h1>

<div class="hud" aria-live="polite">
  <div class="box">⭐ <b id="score">0</b></div>
  <div class="box">🏆 Récord: <b id="best">0</b></div>
  <div class="box">❤️ <b id="lives">3</b></div>
  <div class="box">🎯 Nivel: <b id="level">1</b></div>
  <div class="box">⏱️ <b id="time">22</b>s</div>
  <button id="pause" type="button">⏸️ Pausa</button>
</div>

<div class="progress">
  <div id="progress"></div>
</div>

<div class="arena" id="arena" aria-label="Zona de juego">
  <div class="overlay" id="overlay">
    <div class="panel">
      <div class="big">🪙 Atrapa las Monedas</div>
      <p>
        Consigue monedas, recoge gemas y evita a los enemigos.
        ¡Completa los 5 niveles!
      </p>
      <p class="small">Moneda = +10 puntos · Gema = +30 puntos</p>
      <button id="play" type="button">▶ Comenzar aventura</button>
    </div>
  </div>
</div>
</main>

<script>
(() => {
  const arena = document.getElementById("arena");
  const overlay = document.getElementById("overlay");
  const play = document.getElementById("play");
  const pause = document.getElementById("pause");

  const scoreEl = document.getElementById("score");
  const bestEl = document.getElementById("best");
  const livesEl = document.getElementById("lives");
  const levelEl = document.getElementById("level");
  const timeEl = document.getElementById("time");
  const progress = document.getElementById("progress");

  let score = 0;
  let best = Number(localStorage.getItem("coinBest") || 0);
  let lives = 3;
  let level = 1;
  let time = 22;
  let running = false;
  let paused = false;
  let timer = null;
  let spawnTimer = null;
  let objects = [];

  bestEl.textContent = best;

  function update() {
    scoreEl.textContent = score;
    bestEl.textContent = best;
    livesEl.textContent = lives;
    levelEl.textContent = level;
    timeEl.textContent = time;

    const maxTime = 20 + level * 2;
    progress.style.width =
      Math.max(0, Math.min(100, (time / maxTime) * 100)) + "%";
  }

  function saveBest() {
    if (score > best) {
      best = score;
      localStorage.setItem("coinBest", String(best));
    }
  }

  function clearObjects() {
    objects.forEach(obj => obj.remove());
    objects = [];
    clearTimeout(spawnTimer);
  }

  function finish(completed) {
    running = false;
    paused = false;
    clearInterval(timer);
    clearTimeout(spawnTimer);
    clearObjects();
    saveBest();
    update();

    overlay.innerHTML = `
      <div class="panel">
        <div class="big">${completed ? "🏆 ¡Victoria!" : "💥 ¡Fin de la aventura!"}</div>
        <p>
          ${completed
            ? "¡Completaste los 5 niveles!"
            : "Te quedaste sin vidas."}
        </p>
        <p>
          ⭐ Puntuación: <b>${score}</b><br>
          🎯 Nivel alcanzado: <b>${level}</b><br>
          🏆 Récord: <b>${best}</b>
        </p>
        <button id="again" type="button">🔄 Jugar de nuevo</button>
      </div>
    `;

    overlay.classList.remove("hidden");

    document.getElementById("again").addEventListener("click", startGame);
  }

  function hitEnemy(element) {
    element.remove();
    objects = objects.filter(obj => obj !== element);

    lives--;
    update();

    if (lives <= 0) {
      finish(false);
    }
  }

  function spawn() {
    if (!running || paused) return;

    const roll = Math.random();
    let type;

    if (roll < 0.62) type = "coin";
    else if (roll < 0.78) type = "gem";
    else type = "enemy";

    const element = document.createElement("button");
    element.type = "button";
    element.className = "obj " + type;

    if (type === "coin") {
      element.textContent = "🪙";
      element.setAttribute("aria-label", "Atrapar moneda");

      element.addEventListener("click", () => {
        score += 10;
        saveBest();
        update();
        element.remove();
        objects = objects.filter(obj => obj !== element);
      });

    } else if (type === "gem") {
      element.innerHTML = "<span>💎</span>";
      element.setAttribute("aria-label", "Atrapar gema");

      element.addEventListener("click", () => {
        score += 30;
        saveBest();
        update();
        element.remove();
        objects = objects.filter(obj => obj !== element);
      });

    } else {
      element.textContent = "👾";
      element.setAttribute("aria-label", "Enemigo");

      element.addEventListener("click", () => {
        hitEnemy(element);
      });
    }

    const maxX = Math.max(5, arena.clientWidth - 70);
    const maxY = Math.max(5, arena.clientHeight - 70);

    element.style.left = Math.random() * maxX + "px";
    element.style.top = Math.random() * maxY + "px";

    arena.appendChild(element);
    objects.push(element);

    const lifeTime = Math.max(650, 1500 - level * 80);

    setTimeout(() => {
      if (objects.includes(element)) {
        element.remove();
        objects = objects.filter(obj => obj !== element);
      }
    }, lifeTime);

    spawnTimer = setTimeout(
      spawn,
      Math.max(300, 850 - level * 55)
    );
  }

  function nextLevel() {
    level++;

    if (level > 5) {
      finish(true);
      return;
    }

    time = 20 + level * 2;
    lives = Math.min(3, lives + 1);

    clearObjects();
    update();
    spawn();
  }

  function startGame() {
    clearInterval(timer);
    clearTimeout(spawnTimer);
    clearObjects();

    score = 0;
    lives = 3;
    level = 1;
    time = 22;
    running = true;
    paused = false;

    pause.textContent = "⏸️ Pausa";
    overlay.classList.add("hidden");

    update();
    spawn();

    timer = setInterval(() => {
      if (!running || paused) return;

      time--;

      if (time <= 0) {
        nextLevel();
        return;
      }

      update();
    }, 1000);
  }

  play.addEventListener("click", startGame);

  pause.addEventListener("click", () => {
    if (!running) return;

    paused = !paused;

    if (paused) {
      pause.textContent = "▶ Continuar";

      overlay.innerHTML = `
        <div class="panel">
          <div class="big">⏸️ Juego en pausa</div>
          <p>Tu partida está pausada.</p>
          <button id="resume" type="button">▶ Continuar</button>
        </div>
      `;

      overlay.classList.remove("hidden");

      document.getElementById("resume").addEventListener("click", () => {
        paused = false;
        pause.textContent = "⏸️ Pausa";
        overlay.classList.add("hidden");
        spawn();
      });

    } else {
      pause.textContent = "⏸️ Pausa";
      overlay.classList.add("hidden");
      spawn();
    }
  });

  update();
})();
</script>
</body>
</html>
'''

path = Path("/mnt/data/index.html")
path.write_text(html, encoding="utf-8")
print(f"Archivo creado: {path}")
