<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, shrink-to-fit=no">
    <title>Pesawat Tembak · Tetris</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
            user-select: none;
            -webkit-tap-highlight-color: transparent;
        }
        body {
            background: #0a0f1e;
            min-height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            font-family: 'Segoe UI', system-ui, -apple-system, sans-serif;
            touch-action: pan-y;
        }
        .game-wrapper {
            background: #141a2b;
            padding: 16px 12px 20px;
            border-radius: 48px 48px 28px 28px;
            box-shadow: 0 16px 32px rgba(0,0,0,0.7), inset 0 1px 2px rgba(255,255,255,0.06);
            max-width: 420px;
            width: 100%;
            margin: 12px;
        }
        canvas {
            display: block;
            width: 100%;
            height: auto;
            aspect-ratio: 360 / 520;
            background: #0b111f;
            border-radius: 28px;
            box-shadow: inset 0 0 0 1px #2d3a5e, 0 12px 24px rgba(0,0,0,0.6);
            touch-action: none;
            cursor: crosshair;
        }
        .info-panel {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin: 16px 8px 4px;
            color: #b7c9e6;
            font-weight: 600;
            text-shadow: 0 2px 3px rgba(0,0,0,0.6);
            letter-spacing: 0.5px;
        }
        .score-box, .lives-box {
            background: #1e2742;
            padding: 6px 18px;
            border-radius: 60px;
            font-size: 1.1rem;
            box-shadow: inset 0 2px 6px rgba(0,0,0,0.6), 0 1px 0 #3c4b74;
            backdrop-filter: blur(2px);
            display: flex;
            align-items: center;
            gap: 6px;
        }
        .score-box span, .lives-box span {
            color: #f2e9c2;
            font-size: 1.3rem;
            min-width: 2.4rem;
            text-align: center;
        }
        .lives-box span {
            color: #ff7b89;
        }
        .restart-btn {
            background: #2c385a;
            border: none;
            color: #d4e0ff;
            font-size: 1.4rem;
            padding: 0 16px;
            height: 40px;
            border-radius: 60px;
            font-weight: bold;
            box-shadow: 0 4px 0 #131b2b, 0 6px 12px rgba(0,0,0,0.4);
            transition: 0.08s linear;
            touch-action: manipulation;
            cursor: pointer;
            display: flex;
            align-items: center;
            gap: 4px;
            border: 1px solid #4d5f8a;
        }
        .restart-btn:active {
            transform: translateY(4px);
            box-shadow: 0 1px 0 #131b2b;
        }
        .restart-btn i {
            font-style: normal;
            font-size: 1.2rem;
        }
        @media (max-width: 420px) {
            .game-wrapper { padding: 10px 8px 14px; }
            .score-box, .lives-box { font-size: 0.9rem; padding: 4px 14px; }
            .restart-btn { font-size: 1.2rem; height: 36px; padding: 0 14px; }
        }
    </style>
</head>
<body>
<div class="game-wrapper">
    <canvas id="gameCanvas" width="360" height="520"></canvas>
    <div class="info-panel">
        <div class="score-box">🎯 <span id="scoreDisplay">0</span></div>
        <div class="lives-box">❤️ <span id="livesDisplay">3</span></div>
        <button class="restart-btn" id="restartBtn">↻ <i>ulang</i></button>
    </div>
</div>
<script>
    (function(){
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const scoreSpan = document.getElementById('scoreDisplay');
        const livesSpan = document.getElementById('livesDisplay');

        // Ukuran kanvas 360x520
        const CW = 360, CH = 520;

        // ---- Parameter permainan ----
        const PLAYER_WIDTH = 28;
        const PLAYER_HEIGHT = 20;
        const PLAYER_SPEED = 10;          // per geser (touch)
        const BULLET_WIDTH = 4;
        const BULLET_HEIGHT = 12;
        const BULLET_SPEED = 5;
        const ENEMY_SIZE = 26;            // kotak tetris
        const ENEMY_SPEED = 0.8;
        const MAX_ENEMIES = 12;
        const SPAWN_RATE = 22;            // frame antar spawn

        // ---- State game ----
        let player = { x: CW/2, y: CH-60, w: PLAYER_WIDTH, h: PLAYER_HEIGHT };
        let bullets = [];
        let enemies = [];
        let score = 0;
        let lives = 3;
        let gameOver = false;
        let frameCount = 0;
        let spawnCounter = 0;

        // ---- Touch / mouse state ----
        let touchActive = false;
        let touchX = 0;

        // ---- Inisialisasi ulang ----
        function resetGame() {
            player.x = CW/2;
            player.y = CH-60;
            bullets = [];
            enemies = [];
            score = 0;
            lives = 3;
            gameOver = false;
            frameCount = 0;
            spawnCounter = 0;
            updateDisplay();
        }

        // ---- UI update ----
        function updateDisplay() {
            scoreSpan.textContent = score;
            livesSpan.textContent = lives;
        }

        // ---- Spawn musuh (kotak Tetris warna-warni) ----
        function spawnEnemy() {
            if (enemies.length >= MAX_ENEMIES) return;
            // bentuk tetris: kotak dengan warna random dari palette
            const colors = ['#f94144', '#f3722c', '#f8961e', '#f9c74f', '#90be6d', '#43aa8b', '#4d908e', '#577590', '#277da1', '#b5838d'];
            const color = colors[Math.floor(Math.random() * colors.length)];
            const x = 20 + Math.random() * (CW - ENEMY_SIZE - 40);
            const y = -ENEMY_SIZE - Math.random() * 20;
            enemies.push({
                x, y,
                w: ENEMY_SIZE, h: ENEMY_SIZE,
                color: color,
                speedY: ENEMY_SPEED + Math.random() * 0.3
            });
        }

        // ---- Tembak peluru ----
        function shootBullet() {
            if (gameOver) return;
            bullets.push({
                x: player.x + player.w/2 - BULLET_WIDTH/2,
                y: player.y - 4,
                w: BULLET_WIDTH,
                h: BULLET_HEIGHT,
                speed: BULLET_SPEED
            });
        }

        // ---- Update game logic ----
        function update() {
            if (gameOver) return;

            frameCount++;

            // Gerakkan player (touch / mouse)
            if (touchActive) {
                let targetX = touchX - player.w/2;
                // clamp ke canvas
                targetX = Math.max(0, Math.min(CW - player.w, targetX));
                player.x = targetX;
            }

            // Spawn musuh (tetris)
            spawnCounter++;
            if (spawnCounter >= SPAWN_RATE) {
                spawnCounter = 0;
                spawnEnemy();
                // kadang spawn 2 sekaligus (level naik)
                if (score > 20 && Math.random() < 0.25) spawnEnemy();
                if (score > 50 && Math.random() < 0.35) spawnEnemy();
            }

            // Update peluru
            for (let i = bullets.length-1; i >= 0; i--) {
                const b = bullets[i];
                b.y -= b.speed;
                if (b.y + b.h < 0) {
                    bullets.splice(i, 1);
                    continue;
                }
            }

            // Update musuh + collision peluru
            for (let i = enemies.length-1; i >= 0; i--) {
                const e = enemies[i];
                e.y += e.speedY;

                // collision peluru vs musuh
                let hit = false;
                for (let j = bullets.length-1; j >= 0; j--) {
                    const b = bullets[j];
                    // AABB collision
                    if (b.x < e.x + e.w &&
                        b.x + b.w > e.x &&
                        b.y < e.y + e.h &&
                        b.y + b.h > e.y) {
                        // tembak kena!
                        bullets.splice(j, 1);
                        hit = true;
                        break;
                    }
                }

                if (hit) {
                    enemies.splice(i, 1);
                    score++;
                    updateDisplay();
                    continue;
                }

                // jika musuh melewati bawah (nyawa berkurang)
                if (e.y + e.h > CH + 10) {
                    enemies.splice(i, 1);
                    lives--;
                    updateDisplay();
                    if (lives <= 0) {
                        gameOver = true;
                        lives = 0;
                        updateDisplay();
                    }
                    continue;
                }

                // collision player vs musuh (jika belum game over)
                if (!gameOver) {
                    if (player.x < e.x + e.w &&
                        player.x + player.w > e.x &&
                        player.y < e.y + e.h &&
                        player.y + player.h > e.y) {
                        // tabrakan: hilangkan musuh, kurangi nyawa
                        enemies.splice(i, 1);
                        lives--;
                        updateDisplay();
                        if (lives <= 0) {
                            gameOver = true;
                            lives = 0;
                            updateDisplay();
                        }
                        continue;
                    }
                }
            }

            // Jika game over, stop update lebih lanjut
            if (gameOver) {
                // bersihkan beberapa enemy biar ga numpuk (opsional)
                // tapi biarkan agar visual terlihat
            }
        }

        // ---- Render ----
        function draw() {
            ctx.clearRect(0, 0, CW, CH);

            // ---- Latar grid (efek) ----
            ctx.strokeStyle = '#1f2a44';
            ctx.lineWidth = 0.5;
            for (let i = 0; i < CW; i += 30) {
                ctx.beginPath();
                ctx.moveTo(i, 0);
                ctx.lineTo(i, CH);
                ctx.stroke();
            }
            for (let i = 0; i < CH; i += 30) {
                ctx.beginPath();
                ctx.moveTo(0, i);
                ctx.lineTo(CW, i);
                ctx.stroke();
            }

            // ---- Player (pesawat) ----
            const p = player;
            // badan pesawat
            ctx.shadowColor = '#8ab3ff';
            ctx.shadowBlur = 16;
            ctx.fillStyle = '#6c8cff';
            ctx.beginPath();
            ctx.roundRect(p.x, p.y, p.w, p.h, 6);
            ctx.fill();
            // kokpit
            ctx.shadowBlur = 8;
            ctx.fillStyle = '#b3d0ff';
            ctx.beginPath();
            ctx.roundRect(p.x+6, p.y-4, 6, 6, 4);
            ctx.fill();
            ctx.fillStyle = '#d9ecff';
            ctx.beginPath();
            ctx.roundRect(p.x+16, p.y-4, 6, 6, 4);
            ctx.fill();
            // sayap kecil
            ctx.fillStyle = '#5a7ae0';
            ctx.fillRect(p.x-4, p.y+4, 4, 8);
            ctx.fillRect(p.x+p.w, p.y+4, 4, 8);
            ctx.shadowBlur = 0;

            // ---- Peluru ----
            ctx.shadowColor = '#ffd966';
            ctx.shadowBlur = 14;
            for (let b of bullets) {
                ctx.fillStyle = '#ffea8a';
                ctx.fillRect(b.x, b.y, b.w, b.h);
                // efek glow
                ctx.fillStyle = '#ffb347';
                ctx.fillRect(b.x+1, b.y-2, b.w-2, 4);
            }

            // ---- Musuh (kotak Tetris) ----
            ctx.shadowColor = '#ffffff30';
            ctx.shadowBlur = 12;
            for (let e of enemies) {
                // kotak dengan efek 3d
                ctx.fillStyle = e.color;
                ctx.beginPath();
                ctx.roundRect(e.x, e.y, e.w, e.h, 4);
                ctx.fill();
                // highlight
                ctx.fillStyle = 'rgba(255,255,255,0.2)';
                ctx.beginPath();
                ctx.roundRect(e.x+2, e.y+2, e.w*0.4, 4, 2);
                ctx.fill();
                // outline
                ctx.strokeStyle = 'rgba(0,0,0,0.3)';
                ctx.lineWidth = 1.5;
                ctx.beginPath();
                ctx.roundRect(e.x, e.y, e.w, e.h, 4);
                ctx.stroke();
            }

            // ---- Game Over overlay ----
            if (gameOver) {
                ctx.shadowBlur = 0;
                ctx.fillStyle = 'rgba(0,0,0,0.6)';
                ctx.fillRect(0, 0, CW, CH);
                ctx.fillStyle = '#f2c94c';
                ctx.font = 'bold 34px "Segoe UI", system-ui, sans-serif';
                ctx.textAlign = 'center';
                ctx.textBaseline = 'middle';
                ctx.shadowColor = '#000000';
                ctx.shadowBlur = 20;
                ctx.fillText('💥 GAME OVER', CW/2, CH/2 - 20);
                ctx.font = '18px sans-serif';
                ctx.fillStyle = '#d4dfff';
                ctx.fillText('sentuh layar / tekan ulang', CW/2, CH/2 + 44);
                ctx.shadowBlur = 0;
            }

            // ---- info di canvas (skor & nyawa) ----
            ctx.font = 'bold 14px monospace';
            ctx.fillStyle = '#d0dfff';
            ctx.shadowBlur = 8;
            ctx.shadowColor = '#000000';
            ctx.fillText('❤️ '+lives, 12, 32);
            ctx.fillText('🎯 '+score, CW-70, 32);
            ctx.shadowBlur = 0;
        }

        // ---- roundRect polyfill untuk canvas ----
        if (!CanvasRenderingContext2D.prototype.roundRect) {
            CanvasRenderingContext2D.prototype.roundRect = function (x, y, w, h, r) {
                if (r > w/2) r = w/2;
                if (r > h/2) r = h/2;
                this.moveTo(x + r, y);
                this.lineTo(x + w - r, y);
                this.quadraticCurveTo(x + w, y, x + w, y + r);
                this.lineTo(x + w, y + h - r);
                this.quadraticCurveTo(x + w, y + h, x + w - r, y + h);
                this.lineTo(x + r, y + h);
                this.quadraticCurveTo(x, y + h, x, y + h - r);
                this.lineTo(x, y + r);
                this.quadraticCurveTo(x, y, x + r, y);
                this.closePath();
                return this;
            };
        }

        // ---- Loop animasi ----
        function gameLoop() {
            update();
            draw();
            requestAnimationFrame(gameLoop);
        }

        // ---- Event handlers (touch + mouse) ----
        function handleStart(e) {
            e.preventDefault();
            if (gameOver) {
                resetGame();
                return;
            }
            const rect = canvas.getBoundingClientRect();
            const scaleX = canvas.width / rect.width;
            const clientX = e.touches ? e.touches[0].clientX : e.clientX;
            const canvasX = (clientX - rect.left) * scaleX;
            touchX = Math.min(CW, Math.max(0, canvasX));
            touchActive = true;
            // tembak otomatis saat sentuh
            shootBullet();
        }

        function handleMove(e) {
            e.preventDefault();
            if (gameOver) return;
            const rect = canvas.getBoundingClientRect();
            const scaleX = canvas.width / rect.width;
            const clientX = e.touches ? e.touches[0].clientX : e.clientX;
            const canvasX = (clientX - rect.left) * scaleX;
            touchX = Math.min(CW, Math.max(0, canvasX));
            touchActive = true;
            // tembak saat digeser (agar lebih responsif)
            shootBullet();
        }

        function handleEnd(e) {
            e.preventDefault();
            touchActive = false;
        }

        // ---- Daftarkan event ----
        function attachEvents() {
            canvas.addEventListener('touchstart', handleStart, { passive: false });
            canvas.addEventListener('touchmove', handleMove, { passive: false });
            canvas.addEventListener('touchend', handleEnd, { passive: false });
            canvas.addEventListener('touchcancel', handleEnd, { passive: false });

            // Mouse support (untuk desktop / debugging)
            canvas.addEventListener('mousedown', handleStart);
            canvas.addEventListener('mousemove', (e) => {
                if (e.buttons === 1) { // tombol kiri ditekan
                    handleMove(e);
                }
            });
            canvas.addEventListener('mouseup', handleEnd);
            canvas.addEventListener('mouseleave', handleEnd);
        }

        // ---- Tombol restart ----
        document.getElementById('restartBtn').addEventListener('click', function(e) {
            e.preventDefault();
            resetGame();
        });

        // ---- Inisialisasi ----
        resetGame();
        attachEvents();

        // ---- Mulai loop ----
        gameLoop();

        // ---- Handle resize (tetap proporsional) ----
        function resizeCanvas() {
            // css sudah menangani, tidak perlu resize internal
        }
        window.addEventListener('resize', resizeCanvas);

        // ---- Tambahan: tembak dengan spasi (untuk akses desktop) ----
        document.addEventListener('keydown', (e) => {
            if (e.key === ' ' || e.key === 'Space') {
                e.preventDefault();
                if (gameOver) {
                    resetGame();
                } else {
                    shootBullet();
                }
            }
        });

        // ---- pesan awal ----
        console.log('Pesawat Tembak Tetris — sentuh layar untuk menembak & menggerakkan');
    })();
</script>
</body>
</html>
