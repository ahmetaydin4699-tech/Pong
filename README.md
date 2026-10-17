```html
<!DOCTYPE html>
<html lang="tr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Resimli Ping Pong Oyunu</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        body {
            background: radial-gradient(circle at center, #1e1b4b 0%, #0f172a 100%);
            font-family: 'Inter', sans-serif;
            touch-action: none;
            overflow: hidden;
        }
        canvas {
            box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.7), 0 0 30px rgba(99, 102, 241, 0.2);
            border: 2px solid rgba(255, 255, 255, 0.1);
        }
    </style>
</head>
<body class="h-screen w-screen flex flex-col items-center justify-center p-2 text-white select-none">

    <!-- Skor Paneli -->
    <div class="flex items-center justify-between w-full max-w-md mb-3 px-5 py-3 bg-slate-900/90 backdrop-blur-md rounded-2xl border border-indigo-500/30 shadow-2xl">
        <div class="text-center">
            <span class="text-[10px] tracking-widest text-indigo-300 font-bold uppercase block mb-1">RAKİP (AI)</span>
            <span id="aiScore" class="text-3xl font-black text-rose-400 drop-shadow-[0_0_10px_rgba(244,63,94,0.5)]">0</span>
        </div>
        <div class="flex flex-col items-center">
            <span class="text-xs font-black px-3 py-1 bg-gradient-to-r from-indigo-500 to-purple-500 text-white rounded-full shadow-md">
                PING PONG
            </span>
        </div>
        <div class="text-center">
            <span class="text-[10px] tracking-widest text-indigo-300 font-bold uppercase block mb-1">SİZ</span>
            <span id="playerScore" class="text-3xl font-black text-emerald-400 drop-shadow-[0_0_10px_rgba(16,185,129,0.5)]">0</span>
        </div>
    </div>

    <!-- Oyun Tuvali (Canvas) -->
    <div class="relative w-full max-w-md flex-1 max-h-[68vh] flex items-center justify-center">
        <canvas id="gameCanvas" class="bg-slate-950 rounded-2xl w-full h-full"></canvas>
        
        <!-- Oyun Başlangıç / Bitiş Ekranı -->
        <div id="gameOverlay" class="absolute inset-0 bg-slate-950/90 backdrop-blur-md rounded-2xl flex flex-col items-center justify-center p-6 text-center z-10 transition-opacity duration-300">
            <div class="w-20 h-20 mb-4 rounded-full border-4 border-indigo-500 overflow-hidden shadow-[0_0_20px_rgba(99,102,241,0.6)] animate-pulse">
                <img id="avatarPreview" src="1000163699.jpg" alt="Top Resmi" class="w-full h-full object-cover" onerror="this.src='https://placehold.co/200x200/6366f1/ffffff?text=TOP'">
            </div>
            <h1 class="text-3xl font-black text-transparent bg-clip-text bg-gradient-to-r from-indigo-400 via-purple-400 to-pink-400 mb-2">
                EĞLENCELİ PONG
            </h1>
            <p class="text-slate-300 text-xs leading-relaxed mb-6 max-w-xs">
                Yön tuşları veya <b>WASD</b> ile raketinizi <span class="text-emerald-400 font-bold">Sağ, Sol, Yukarı ve Aşağı</span> hareket ettirin!
            </p>
            <button id="startBtn" class="px-8 py-3.5 bg-gradient-to-r from-indigo-500 via-purple-500 to-pink-500 hover:from-indigo-600 hover:to-pink-600 text-white font-black rounded-xl shadow-[0_0_25px_rgba(99,102,241,0.5)] transform transition active:scale-95 cursor-pointer tracking-wider">
                BAŞLAT
            </button>
        </div>
    </div>

    <!-- Mobil Yön Kontrolleri -->
    <div class="w-full max-w-md mt-3 flex justify-center items-center gap-2 md:hidden">
        <div class="grid grid-cols-3 gap-2 w-52">
            <div></div>
            <button id="btnUp" class="bg-slate-800/90 active:bg-indigo-600 p-3 rounded-xl text-center font-bold text-xl border border-slate-700 active:scale-95 transition">▲</button>
            <div></div>
            <button id="btnLeft" class="bg-slate-800/90 active:bg-indigo-600 p-3 rounded-xl text-center font-bold text-xl border border-slate-700 active:scale-95 transition">◄</button>
            <button id="btnDown" class="bg-slate-800/90 active:bg-indigo-600 p-3 rounded-xl text-center font-bold text-xl border border-slate-700 active:scale-95 transition">▼</button>
            <button id="btnRight" class="bg-slate-800/90 active:bg-indigo-600 p-3 rounded-xl text-center font-bold text-xl border border-slate-700 active:scale-95 transition">►</button>
        </div>
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const playerScoreElem = document.getElementById('playerScore');
        const aiScoreElem = document.getElementById('aiScore');
        const overlay = document.getElementById('gameOverlay');
        const startBtn = document.getElementById('startBtn');

        // Yüklenen Resmin Doğrudan Tanımlanması
        const ballImage = new Image();
        let imageLoaded = false;

        ballImage.onload = () => {
            imageLoaded = true;
        };

        // Eğer dosya yerelde bulunamazsa otomatik varsayılan görsele düşer
        ballImage.onerror = () => {
            ballImage.src = 'https://placehold.co/200x200/6366f1/ffffff?text=TOP';
        };

        ballImage.src = '1000163699.jpg';

        // Oyun Durum Değişkenleri
        let gameRunning = false;
        let playerScore = 0;
        let aiScore = 0;

        function resizeCanvas() {
            const rect = canvas.parentElement.getBoundingClientRect();
            canvas.width = rect.width;
            canvas.height = rect.height;
        }
        window.addEventListener('resize', resizeCanvas);
        resizeCanvas();

        // Oyuncu (Alt Raket)
        const player = {
            width: 95,
            height: 14,
            x: 0,
            y: 0,
            speed: 7,
            minY: 0,
            maxY: 0
        };

        // Rakip (Üst Raket)
        const ai = {
            width: 95,
            height: 14,
            x: 0,
            y: 15,
            speed: 4.8
        };

        // Top
        const ball = {
            x: 0,
            y: 0,
            radius: 26, // Fotoğraf net görünsün diye top boyutu ideal tutuldu
            speedX: 0,
            speedY: 0,
            baseSpeed: 5.5,
            rotation: 0
        };

        // Klavye Tuş Kontrolleri
        const keys = {
            ArrowLeft: false,
            ArrowRight: false,
            ArrowUp: false,
            ArrowDown: false,
            KeyA: false,
            KeyD: false,
            KeyW: false,
            KeyS: false
        };

        window.addEventListener('keydown', (e) => {
            if (keys.hasOwnProperty(e.code)) keys[e.code] = true;
        });

        window.addEventListener('keyup', (e) => {
            if (keys.hasOwnProperty(e.code)) keys[e.code] = false;
        });

        // Mobil Dokunmatik Buton Kontrolleri
        const bindTouch = (id, key) => {
            const btn = document.getElementById(id);
            if(!btn) return;
            const start = (e) => { e.preventDefault(); keys[key] = true; };
            const end = (e) => { e.preventDefault(); keys[key] = false; };
            btn.addEventListener('touchstart', start);
            btn.addEventListener('touchend', end);
            btn.addEventListener('mousedown', start);
            btn.addEventListener('mouseup', end);
        };
        bindTouch('btnUp', 'ArrowUp');
        bindTouch('btnDown', 'ArrowDown');
        bindTouch('btnLeft', 'ArrowLeft');
        bindTouch('btnRight', 'ArrowRight');

        function resetPositions() {
            player.x = canvas.width / 2 - player.width / 2;
            player.y = canvas.height - 45;
            player.minY = canvas.height / 2 + 10; // Kendi sahasında hareket sınırı
            player.maxY = canvas.height - player.height - 10;

            ai.x = canvas.width / 2 - ai.width / 2;
            ai.y = 15;

            ball.x = canvas.width / 2;
            ball.y = canvas.height / 2;
            
            const angle = (Math.random() * Math.PI / 4) - Math.PI / 8;
            const direction = Math.random() > 0.5 ? 1 : -1;
            ball.speedX = ball.baseSpeed * Math.sin(angle);
            ball.speedY = ball.baseSpeed * Math.cos(angle) * direction;
        }

        function update() {
            if (!gameRunning) return;

            // Oyuncu Hareketi (4 Yönlü)
            if (keys.ArrowLeft || keys.KeyA) player.x -= player.speed;
            if (keys.ArrowRight || keys.KeyD) player.x += player.speed;
            if (keys.ArrowUp || keys.KeyW) player.y -= player.speed;
            if (keys.ArrowDown || keys.KeyS) player.y += player.speed;

            // Oyuncu Saha Sınırları
            if (player.x < 0) player.x = 0;
            if (player.x + player.width > canvas.width) player.x = canvas.width - player.width;
            if (player.y < player.minY) player.y = player.minY;
            if (player.y > player.maxY) player.y = player.maxY;

            // Yapay Zeka Hareketi
            const targetX = ball.y < canvas.height / 2 ? ball.x - ai.width / 2 : canvas.width / 2 - ai.width / 2;
            if (ai.x + ai.width / 2 < targetX - 6) {
                ai.x += ai.speed;
            } else if (ai.x + ai.width / 2 > targetX + 6) {
                ai.x -= ai.speed;
            }
            if (ai.x < 0) ai.x = 0;
            if (ai.x + ai.width > canvas.width) ai.x = canvas.width - ai.width;

            // Top Hareketi & Dönüşü
            ball.x += ball.speedX;
            ball.y += ball.speedY;
            ball.rotation += (ball.speedX * 0.03);

            // Sol / Sağ Yan Duvar Sekmesi
            if (ball.x - ball.radius < 0) {
                ball.x = ball.radius;
                ball.speedX *= -1;
            } else if (ball.x + ball.radius > canvas.width) {
                ball.x = canvas.width - ball.radius;
                ball.speedX *= -1;
            }

            // Oyuncu Raketine Vuruş
            if (
                ball.y + ball.radius >= player.y &&
                ball.y - ball.radius <= player.y + player.height &&
                ball.x + ball.radius >= player.x &&
                ball.x - ball.radius <= player.x + player.width &&
                ball.speedY > 0
            ) {
                let hitPoint = (ball.x - (player.x + player.width / 2)) / (player.width / 2);
                let angle = hitPoint * (Math.PI / 3);
                let speed = Math.hypot(ball.speedX, ball.speedY) * 1.03;
                ball.speedX = speed * Math.sin(angle);
                ball.speedY = -speed * Math.cos(angle);
                ball.y = player.y - ball.radius;
            }

            // AI Raketine Vuruş
            if (
                ball.y - ball.radius <= ai.y + ai.height &&
                ball.y + ball.radius >= ai.y &&
                ball.x + ball.radius >= ai.x &&
                ball.x - ball.radius <= ai.x + ai.width &&
                ball.speedY < 0
            ) {
                let hitPoint = (ball.x - (ai.x + ai.width / 2)) / (ai.width / 2);
                let angle = hitPoint * (Math.PI / 3);
                let speed = Math.hypot(ball.speedX, ball.speedY) * 1.03;
                ball.speedX = speed * Math.sin(angle);
                ball.speedY = speed * Math.cos(angle);
                ball.y = ai.y + ai.height + ball.radius;
            }

            // Sayı Kontrolü
            if (ball.y - ball.radius > canvas.height) {
                aiScore++;
                aiScoreElem.textContent = aiScore;
                resetPositions();
            }

            if (ball.y + ball.radius < 0) {
                playerScore++;
                playerScoreElem.textContent = playerScore;
                resetPositions();
            }
        }

        function draw() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // Saha Çizgileri
            ctx.strokeStyle = 'rgba(255, 255, 255, 0.08)';
            ctx.lineWidth = 2;
            ctx.setLineDash([8, 8]);
            ctx.beginPath();
            ctx.moveTo(0, canvas.height / 2);
            ctx.lineTo(canvas.width, canvas.height / 2);
            ctx.stroke();
            ctx.setLineDash([]);

            ctx.beginPath();
            ctx.arc(canvas.width / 2, canvas.height / 2, 45, 0, Math.PI * 2);
            ctx.stroke();

            // Oyuncu Raketi
            const playerGrad = ctx.createLinearGradient(player.x, player.y, player.x, player.y + player.height);
            playerGrad.addColorStop(0, '#10b981');
            playerGrad.addColorStop(1, '#059669');
            ctx.fillStyle = playerGrad;
            ctx.shadowColor = 'rgba(16, 185, 129, 0.5)';
            ctx.shadowBlur = 12;
            ctx.beginPath();
            ctx.roundRect(player.x, player.y, player.width, player.height, 8);
            ctx.fill();

            // Yapay Zeka Raketi
            const aiGrad = ctx.createLinearGradient(ai.x, ai.y, ai.x, ai.y + ai.height);
            aiGrad.addColorStop(0, '#f43f5e');
            aiGrad.addColorStop(1, '#e11d48');
            ctx.fillStyle = aiGrad;
            ctx.shadowColor = 'rgba(244, 63, 94, 0.5)';
            ctx.shadowBlur = 12;
            ctx.beginPath();
            ctx.roundRect(ai.x, ai.y, ai.width, ai.height, 8);
            ctx.fill();
            ctx.shadowBlur = 0; // Gölge efektini sıfırla

            // --- FOTOĞRAFLI TOP ÇİZİMİ ---
            ctx.save();
            ctx.translate(ball.x, ball.y);
            ctx.rotate(ball.rotation);

            // Dış Parlak Çerçeve Halka
            ctx.beginPath();
            ctx.arc(0, 0, ball.radius + 3, 0, Math.PI * 2);
            ctx.fillStyle = '#818cf8';
            ctx.shadowColor = '#6366f1';
            ctx.shadowBlur = 15;
            ctx.fill();
            ctx.shadowBlur = 0;

            // Dairesel Kırpma Alanı
            ctx.beginPath();
            ctx.arc(0, 0, ball.radius, 0, Math.PI * 2);
            ctx.clip();

            if (imageLoaded) {
                // Yüklenen Resim (1000163699.jpg) Topun Üzerine Çizilir
                ctx.drawImage(
                    ballImage,
                    -ball.radius,
                    -ball.radius,
                    ball.radius * 2,
                    
