<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
    <title>Josdie Run | By Avril Ola</title>
    <script src="https://cdn.jsdelivr.net/npm/canvas-confetti@1.5.1/dist/confetti.browser.min.js"></script>
    <style>
        /* FORCE LE PLEIN ÉCRAN ET SUPPRIME LES MARGES GITHUB */
        :root { --primary: #00ff88; }
        * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; margin: 0; padding: 0; }
        
        html, body { 
            width: 100% !important; 
            height: 100% !important; 
            overflow: hidden !important;
            background: #000 !important;
            position: fixed;
            touch-action: none;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        }

        /* L'ESPACE DE JEU DOIT PRENDRE TOUTE LA PLACE */
        #game-container { 
            position: absolute;
            top: 0; left: 0;
            width: 100vw; height: 100vh;
            display: none;
            z-index: 5;
        }

        #canvas { display: block; width: 100%; height: 100%; }

        /* INTERFACE CENTRÉE */
        #overlay, #death-screen { 
            position: fixed; inset: 0; z-index: 100; 
            display: flex; flex-direction: column; align-items: center; justify-content: center; 
            background: rgba(0,0,0,0.9); padding: 20px;
        }

        .box { 
            background: #111; padding: 30px; border-radius: 25px; 
            border: 1px solid #333; width: 90%; max-width: 400px; 
            box-shadow: 0 0 50px rgba(0,255,136,0.15);
            text-align: center;
        }

        h1 { font-size: 2.2rem; color: var(--primary); margin: 0; text-transform: uppercase; letter-spacing: -1px; }
        
        .grid-char { display: grid; grid-template-columns: repeat(4, 1fr); gap: 10px; margin: 20px 0; }
        .char-opt { 
            background: #222; font-size: 1.8rem; padding: 12px; border-radius: 15px; 
            cursor: pointer; border: 2px solid transparent; transition: 0.2s;
        }
        .char-opt.active { border-color: var(--primary); background: #2a2a2a; transform: scale(1.1); }
        
        input { 
            width: 100%; padding: 15px; background: #000; border: 1px solid #444; 
            color: #fff; border-radius: 50px; margin-bottom: 20px; 
            text-align: center; font-size: 1.1rem; outline: none;
        }

        .btn { 
            background: var(--primary); color: #000; padding: 16px; 
            border: none; border-radius: 50px; font-weight: 900; 
            cursor: pointer; width: 100%; text-transform: uppercase; font-size: 1rem;
        }

        #ui { position: absolute; top: 30px; width: 100%; text-align: center; pointer-events: none; z-index: 50; }
        #chrono { font-size: clamp(2.5rem, 10vw, 4rem); font-weight: 900; font-family: monospace; }
        #glide-ui { width: 150px; height: 8px; background: rgba(0,0,0,0.5); margin: 10px auto; border-radius: 10px; overflow: hidden; border: 1px solid rgba(255,255,255,0.2); }
        #glide-bar { width: 100%; height: 100%; background: var(--primary); }

        footer { position: fixed; bottom: 10px; width: 100%; text-align: center; font-size: 0.7rem; opacity: 0.4; z-index: 10; }
    </style>
</head>
<body>

    <div id="overlay">
        <div class="box">
            <h1>JOSDIE <span style="color:#fff">RUN</span></h1>
            <p style="opacity:0.5; font-size: 0.8rem; margin: 5px 0 20px;">BY AVRIL OLA</p>
            <div class="grid-char">
                <div class="char-opt active" onclick="setChar('🦁','🤠','#f4a460','#8b4513','savannah')">🦁</div>
                <div class="char-opt" onclick="setChar('🦊','🐺','#1a1a2e','#050505','forest')">🦊</div>
                <div class="char-opt" onclick="setChar('🐼','🐯','#e0e0e0','#a0a0a0','snow')">🐼</div>
                <div class="char-opt" onclick="setChar('🐸','🐍','#2e8b57','#1a3a2a','marsh')">🐸</div>
            </div>
            <input type="text" id="nick" placeholder="PSEUDO" autofocus onkeydown="if(event.key === 'Enter') startApp()">
            <button class="btn" onclick="startApp()">ENTRER DANS L'ARÈNE</button>
        </div>
    </div>

    <div id="death-screen" style="display:none;">
        <div class="box">
            <h1 style="color:#fff;">DÉVORÉ !</h1>
            <p id="death-msg" style="margin: 20px 0; font-size: 1.1rem;"></p>
            <button class="btn" onclick="resetGame()">RÉESSAYER</button>
            <button class="btn" style="background:#444; color:#fff; margin-top:10px;" onclick="location.reload()">MENU</button>
        </div>
    </div>

    <div id="game-container">
        <div id="ui">
            <div id="chrono">00:00:00</div>
            <div id="glide-ui"><div id="glide-bar"></div></div>
        </div>
        <canvas id="canvas"></canvas>
    </div>

    <footer>© 2026 - By Avril Ola - Tous droits réservés</footer>

    <script>
        const canvas = document.getElementById('canvas');
        const ctx = canvas.getContext('2d');
        const nickInput = document.getElementById('nick');

        let char = '🦁', pred = '🤠', sky = '#f4a460', ground = '#8b4513', biome = 'savannah';
        let isRunning = false, isPressing = false, playerY = 0, velocityY = 0, jumpCount = 0, glideEnergy = 100;
        let obstacles = [], backgroundElements = [];
        let speed = 0, startTime = 0, nextObs = 0, lastTime = 0;
        let scale = 1, groundY = 0;

        function resize() {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
            groundY = canvas.height * 0.8;
            scale = Math.min(canvas.width, canvas.height) / 450; 
            if (!isRunning) playerY = groundY - (55 * scale);
        }
        window.addEventListener('resize', resize);
        resize();

        function setChar(c, p, s, g, b) {
            char = c; pred = p; sky = s; ground = g; biome = b;
            document.querySelectorAll('.char-opt').forEach(el => el.classList.remove('active'));
            event.currentTarget.classList.add('active');
        }

        function startApp() {
            if(!nickInput.value.trim()) return alert("Pseudo !");
            document.getElementById('overlay').style.display = "none";
            document.getElementById('game-container').style.display = "block";
            resetGame();
        }

        function resetGame() {
            document.getElementById('death-screen').style.display = "none";
            obstacles = []; backgroundElements = [];
            isRunning = true; velocityY = 0; jumpCount = 0; glideEnergy = 100;
            speed = canvas.width / 130; 
            startTime = performance.now(); lastTime = performance.now();
            requestAnimationFrame(update);
        }

        function spawnBackground(x = canvas.width) {
            const biomes = {'savannah':['🌵','🏜️','☀️'],'forest':['🌲','🌲','🌑'],'snow':['🏔️','❄️','❄️'],'marsh':['🌿','🍄','🌱']};
            backgroundElements.push({
                x, y: groundY - (Math.random() * 180 * scale + 10),
                txt: biomes[biome][Math.floor(Math.random()*3)],
                size: (30 + Math.random() * 40) * scale,
                v: 0.15 + Math.random() * 0.3
            });
        }

        function update(now) {
            if (!isRunning) return;
            const dt = (now - lastTime) / 16.67;
            lastTime = now;

            ctx.fillStyle = sky; ctx.fillRect(0, 0, canvas.width, groundY);
            ctx.fillStyle = ground; ctx.fillRect(0, groundY, canvas.width, canvas.height - groundY);

            document.getElementById('chrono').innerText = fmt(now - startTime);
            speed += 0.0008 * dt * scale;

            if (Math.random() < 0.02) spawnBackground();
            backgroundElements.forEach((el, i) => {
                el.x -= speed * el.v * dt;
                ctx.globalAlpha = 0.3; ctx.font = `${el.size}px serif`;
                ctx.fillText(el.txt, el.x, el.y);
                if (el.x < -100) backgroundElements.splice(i, 1);
            });
            ctx.globalAlpha = 1.0;

            let gravity = 0.75 * scale;
            let jumpForce = -15 * scale;
            let curGrav = (isPressing && velocityY > 0 && glideEnergy > 0) ? (0.18 * scale) : gravity;

            if (isPressing && velocityY > 0 && glideEnergy > 0) {
                glideEnergy -= 1.4 * dt;
            } else if (playerY >= groundY - (60 * scale)) {
                if(glideEnergy < 100) glideEnergy += 0.5 * dt;
            }
            document.getElementById('glide-bar').style.width = glideEnergy + "%";

            velocityY += curGrav * dt;
            playerY += velocityY * dt;

            if (playerY > groundY - (55 * scale)) {
                playerY = groundY - (55 * scale); velocityY = 0; jumpCount = 0;
            }

            ctx.font = `${60 * scale}px serif`;
            ctx.fillText(char, canvas.width * 0.12, playerY + (50 * scale));

            if (now > nextObs) {
                obstacles.push({ x: canvas.width });
                nextObs = now + (1300 + Math.random() * 1100) / (speed/(canvas.width/130));
            }
            obstacles.forEach((obs, i) => {
                obs.x -= speed * dt;
                ctx.fillText(pred, obs.x, groundY - 5);

                let pX = canvas.width * 0.12, pSize = 45 * scale;
                if (Math.abs(obs.x - (pX + pSize/2)) < pSize * 0.7 && playerY > groundY - (90 * scale)) {
                    gameOver(now - startTime);
                }
                if (obs.x < -100) obstacles.splice(i, 1);
            });

            requestAnimationFrame(update);
        }

        function gameOver(score) {
            isRunning = false;
            document.getElementById('death-msg').innerText = `${nickInput.value}, tu as survécu ${fmt(score)}`;
            document.getElementById('death-screen').style.display = "flex";
            confetti({ particleCount: 100, spread: 70, origin: { y: 0.6 } });
        }

        function inputStart() { if (isRunning && jumpCount < 2) { velocityY = -15 * scale; jumpCount++; } isPressing = true; }
        function inputEnd() { isPressing = false; }

        window.addEventListener('keydown', e => { if(e.code === 'Space' || e.code === 'ArrowUp') { e.preventDefault(); inputStart(); } });
        window.addEventListener('keyup', e => { if(e.code === 'Space' || e.code === 'ArrowUp') inputEnd(); });
        canvas.addEventListener('touchstart', e => { e.preventDefault(); inputStart(); }, {passive: false});
        canvas.addEventListener('touchend', inputEnd);

        function fmt(t) {
            let m = Math.floor(t/60000), s = Math.floor((t%60000)/1000), ms = Math.floor((t%1000)/10);
            return `${m.toString().padStart(2,'0')}:${s.toString().padStart(2,'0')}:${ms.toString().padStart(2,'0')}`;
        }
    </script>
</body>
</html>
