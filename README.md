<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
    <title>Josdie Run | Clemiz Studio</title>
    <script src="https://cdn.jsdelivr.net/npm/canvas-confetti@1.5.1/dist/confetti.browser.min.js"></script>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
        html, body { width: 100%; height: 100%; overflow: hidden; background: #000; position: fixed; font-family: 'Arial Black', sans-serif; color: #fff; }
        
        #overlay, #death-screen { 
            position: absolute; inset: 0; z-index: 100; display: flex; flex-direction: column; 
            align-items: center; justify-content: center; background: rgba(0,0,0,0.8); backdrop-filter: blur(12px);
        }

        .box { 
            background: rgba(26, 26, 26, 0.9); padding: 35px; border-radius: 40px; border: 4px solid #00ff88; 
            width: 90%; max-width: 400px; text-align: center; box-shadow: 0 0 60px rgba(0,255,136,0.3);
        }

        h1 { font-size: 2.5rem; color: #00ff88; text-shadow: 3px 3px #000; }
        .studio { font-size: 0.8rem; color: #aaa; margin-bottom: 25px; letter-spacing: 3px; }
        
        .grid { display: grid; grid-template-columns: repeat(4, 1fr); gap: 15px; margin-bottom: 25px; }
        .opt { 
            background: #333; font-size: 2.2rem; padding: 12px; border-radius: 20px; 
            cursor: pointer; border: 3px solid transparent; transition: 0.2s;
        }
        .opt.active { border-color: #00ff88; background: #444; transform: scale(1.1); box-shadow: 0 0 20px #00ff88; }
        
        input { width: 100%; padding: 18px; background: #fff; border: none; color: #000; border-radius: 50px; margin-bottom: 20px; text-align: center; font-size: 1.3rem; font-weight: 900; }

        .btn { 
            background: #444; color: #fff; padding: 18px; border: none; border-radius: 50px; 
            font-weight: 900; cursor: pointer; width: 100%; text-transform: uppercase; 
            font-size: 1.1rem; margin-bottom: 12px; border: 3px solid transparent; transition: 0.2s;
        }
        .btn.selected, .btn:hover { background: #00ff88; color: #000; border-color: #fff; box-shadow: 0 0 20px rgba(0,255,136,0.5); transform: scale(1.05); }

        #ui { position: absolute; top: 40px; width: 100%; text-align: center; pointer-events: none; z-index: 50; }
        #chrono { font-size: 4rem; font-weight: 900; text-shadow: 4px 4px 0 #000; }
        
        canvas { display: block; width: 100%; height: 100%; }
        footer { position: absolute; bottom: 20px; width: 100%; text-align: center; font-size: 0.8rem; opacity: 0.6; }
    </style>
</head>
<body>
    <div id="game-wrap">
        <div id="overlay">
            <div class="box">
                <h1>JOSDIE RUN</h1>
                <p class="studio">BY CLEMIZ STUDIO</p>
                <div class="grid" id="charGrid">
                    <div class="opt active" onclick="selectChar(0)">🦁</div>
                    <div class="opt" onclick="selectChar(1)">🦊</div>
                    <div class="opt" onclick="selectChar(2)">🐼</div>
                    <div class="opt" onclick="selectChar(3)">🐸</div>
                </div>
                <input type="text" id="nick" placeholder="TON PSEUDO" maxlength="10">
                <button class="btn selected" onclick="startApp()">JOUER</button>
            </div>
        </div>

        <div id="death-screen" style="display:none;">
            <div class="box">
                <h1 style="color:#ff3e3e">DOMMAGE !</h1>
                <p id="msg" style="margin: 20px 0; font-size: 1.4rem; font-weight: bold;"></p>
                <button class="btn selected" id="btn-retry" onclick="resetGame()">RÉESSAYER</button>
                <button class="btn" id="btn-menu" onclick="goBackToMenu()">MENU</button>
            </div>
        </div>

        <div id="ui"><div id="chrono">00:00:00</div></div>
        <canvas id="canvas"></canvas>
        <footer>© 2026 - By Clemiz Studio</footer>
    </div>

    <script>
        const canvas = document.getElementById('canvas'), ctx = canvas.getContext('2d');
        const nickInput = document.getElementById('nick');
        const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
        
        let musicInterval;
        const characters = [
            {c:'🦁', p:'🤠', sky:'#FF4E50', ground:'#8B4513', decor:'#FC913A', note: 261}, // Savane
            {c:'🦊', p:'🐺', sky:'#1A1A2E', ground:'#0F0C29', decor:'#302B63', note: 293}, // Forêt
            {c:'🐼', p:'🐯', sky:'#E0E0E0', ground:'#757F9A', decor:'#D7DDE8', note: 329}, // Neige
            {c:'🐸', p:'🐍', sky:'#11998E', ground:'#1A3A2A', decor:'#38EF7D', note: 349}  // Jungle
        ];

        let charIndex = 0, isRunning = false, playerY = 0, velocityY = 0, jumpCount = 0, menuSel = 0;
        let obs = [], decorItems = [], speed = 0, startTime = 0, nextObs = 0, lastTime = 0, scale = 1, groundY = 0;

        function playSound(freq, vol=0.1, type='square') {
            const osc = audioCtx.createOscillator(), gain = audioCtx.createGain();
            osc.type = type; osc.frequency.setValueAtTime(freq, audioCtx.currentTime);
            gain.gain.setValueAtTime(vol, audioCtx.currentTime);
            gain.gain.exponentialRampToValueAtTime(0.0001, audioCtx.currentTime + 0.2);
            osc.connect(gain); gain.connect(audioCtx.destination);
            osc.start(); osc.stop(audioCtx.currentTime + 0.2);
        }

        function startMusic() {
            clearInterval(musicInterval);
            let step = 0;
            musicInterval = setInterval(() => {
                if (!isRunning) return;
                const n = characters[charIndex].note;
                if (step % 4 === 0) playSound(n/2, 0.15, 'sine'); // Basse
                if (step % 8 === 2) playSound(n, 0.05, 'square'); // Mélodie
                step++;
            }, 150);
        }

        function resize() {
            canvas.width = window.innerWidth; canvas.height = window.innerHeight;
            groundY = canvas.height * 0.85; scale = Math.min(canvas.width, canvas.height) / 450;
        }
        window.addEventListener('resize', resize); resize();

        function selectChar(idx) {
            charIndex = (idx + 4) % 4;
            document.querySelectorAll('.opt').forEach((el, i) => el.classList.toggle('active', i === charIndex));
            if(audioCtx.state === 'suspended') audioCtx.resume();
            playSound(characters[charIndex].note);
        }

        function resetGame() {
            isRunning = true;
            document.getElementById('death-screen').style.display = 'none';
            obs = []; decorItems = []; velocityY = 0; jumpCount = 0;
            speed = canvas.width / 130; startTime = performance.now(); lastTime = performance.now();
            startMusic(); requestAnimationFrame(update);
        }

        function update(now) {
            if(!isRunning) return;
            const dt = (now - lastTime) / 16.67; lastTime = now;
            const cur = characters[charIndex];

            // DÉCOR FILIGRANE ULTRA
            ctx.fillStyle = cur.sky; ctx.fillRect(0, 0, canvas.width, groundY);
            ctx.fillStyle = cur.decor; ctx.globalAlpha = 0.3;
            // Dessin de collines/montagnes douces en arrière-plan
            ctx.beginPath();
            ctx.moveTo(0, groundY);
            for(let i=0; i<=canvas.width; i+=100) { ctx.lineTo(i, groundY - 100 - Math.sin(i*0.01 + now*0.001)*50); }
            ctx.lineTo(canvas.width, groundY); ctx.fill();
            ctx.globalAlpha = 1.0;

            ctx.fillStyle = cur.ground; ctx.fillRect(0, groundY, canvas.width, canvas.height - groundY);

            document.getElementById('chrono').innerText = fmt(now - startTime);
            speed += 0.001 * dt * scale;
            velocityY += 0.8 * scale * dt; playerY += velocityY * dt;
            if (playerY > groundY - (65 * scale)) { playerY = groundY - (65 * scale); velocityY = 0; jumpCount = 0; }

            ctx.font = `${70 * scale}px serif`;
            ctx.fillText(cur.c, canvas.width * 0.15, playerY + (55 * scale));

            if (now > nextObs) {
                obs.push({ x: canvas.width });
                nextObs = now + (1100 + Math.random() * 800) / (speed / (canvas.width / 130));
            }

            obs.forEach((o, i) => {
                o.x -= speed * dt;
                ctx.fillText(cur.p, o.x, groundY - 10);
                let hitSize = 45 * scale;
                if (Math.abs(o.x - (canvas.width * 0.15 + hitSize/2)) < hitSize * 0.7 && playerY > groundY - (100 * scale)) gameOver(now - startTime);
                if (o.x < -100) obs.splice(i, 1);
            });
            requestAnimationFrame(update);
        }

        function gameOver(s) {
            isRunning = false; clearInterval(musicInterval);
            playSound(100, 0.2, 'sawtooth');
            document.getElementById('msg').innerText = `${nickInput.value || 'Joueur'} : ${fmt(s)}`;
            document.getElementById('death-screen').style.display = 'flex';
            menuSel = 0; updateMenuUI();
            confetti({ particleCount: 100, spread: 70, origin: { y: 0.6 } });
        }

        function updateMenuUI() {
            document.getElementById('btn-retry').classList.toggle('selected', menuSel === 0);
            document.getElementById('btn-menu').classList.toggle('selected', menuSel === 1);
        }

        function startApp() {
            if(!nickInput.value.trim()) { alert("Pseudo requis !"); return; }
            document.getElementById('overlay').style.display = 'none';
            resetGame();
        }

        function goBackToMenu() {
            isRunning = false; clearInterval(musicInterval);
            document.getElementById('death-screen').style.display = 'none';
            document.getElementById('overlay').style.display = 'flex';
        }

        // CONTROLES SOURIS & CLAVIER UNIFIÉS
        window.addEventListener('keydown', e => {
            if (document.getElementById('overlay').style.display !== 'none') {
                if (e.key === 'ArrowRight') selectChar(charIndex + 1);
                if (e.key === 'ArrowLeft') selectChar(charIndex - 1);
                if (e.key === 'Enter') startApp();
                return;
            }
            if (document.getElementById('death-screen').style.display !== 'none') {
                if (e.key === 'ArrowUp' || e.key === 'ArrowDown') { menuSel = 1 - menuSel; updateMenuUI(); }
                if (e.key === 'Enter') menuSel === 0 ? resetGame() : goBackToMenu();
                return;
            }
            if ((e.code === 'Space' || e.key === 'ArrowUp') && isRunning && jumpCount < 2) {
                e.preventDefault(); velocityY = -16 * scale; jumpCount++;
                playSound(440, 0.05, 'triangle');
            }
        });

        canvas.addEventListener('touchstart', e => { 
            if(isRunning && jumpCount < 2) { e.preventDefault(); velocityY = -16 * scale; jumpCount++; playSound(440, 0.05, 'triangle'); }
        });

        // Gestion de la souris sur le panneau d'échec
        document.getElementById('btn-retry').onmouseenter = () => { menuSel = 0; updateMenuUI(); };
        document.getElementById('btn-menu').onmouseenter = () => { menuSel = 1; updateMenuUI(); };

        function fmt(t) {
            let m = Math.floor(t/60000), s = Math.floor((t%60000)/1000), ms = Math.floor((t%1000)/10);
            return `${m.toString().padStart(2,'0')}:${s.toString().padStart(2,'0')}:${ms.toString().padStart(2,'0')}`;
        }
    </script>
</body>
</html>
