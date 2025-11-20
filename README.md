<!DOCTYPE html>
<html lang="tr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Karanlık Kuşatma - Zombi Nişancı</title>
    <!-- Tailwind CSS for modern layout/styling -->
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap" rel="stylesheet">
    <style>
        :root {
            --game-width: 800px;
            --game-height: 600px;
            --background-color: #0f0f1c;
            --player-color: #00bcd4;
            --zombie-color: #800080;
            --bullet-color: #ff9800;
        }

        body {
            font-family: 'Inter', sans-serif;
            background-color: #0d0d1e;
            color: #f0f0f0;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            padding: 20px;
        }

        #game-container {
            display: flex;
            gap: 24px;
            max-width: 1400px;
            width: 95%;
            flex-wrap: wrap;
            justify-content: center;
        }

        #game-area {
            display: flex;
            flex-direction: column;
            align-items: center;
            padding: 20px;
            background: #1a1a2e;
            border-radius: 16px;
            box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
            position: relative; /* Canvas ve HUD'ı tutar */
        }

        #shooter-canvas {
            border: 4px solid var(--player-color);
            background-color: var(--background-color);
            box-shadow: 0 0 20px rgba(0, 188, 212, 0.5);
            width: var(--game-width);
            height: var(--game-height);
            border-radius: 8px;
            cursor: crosshair;
        }

        .info-panel {
            min-width: 300px;
            background: #232340;
            padding: 20px;
            border-radius: 16px;
            box-shadow: inset 0 0 10px rgba(0, 0, 0, 0.3);
            display: flex;
            flex-direction: column;
            gap: 16px;
        }

        .info-box {
            background: #1a1a2e;
            padding: 12px;
            border-radius: 8px;
            border: 1px solid #3f3f6f;
        }

        .info-box h3 {
            color: var(--bullet-color);
            font-size: 1.2rem;
            margin-bottom: 8px;
            border-bottom: 2px solid var(--bullet-color);
            padding-bottom: 4px;
        }
        
        /* Satın alma butonları */
        .upgrade-button {
            transition: all 0.1s;
            background-color: #4f46e5;
        }
        .upgrade-button:hover {
            background-color: #6366f1;
            transform: translateY(-1px);
        }
        .upgrade-button:disabled {
            background-color: #4b5563;
            cursor: not-allowed;
            opacity: 0.7;
            transform: none;
        }
        
        .user-id-display {
            font-size: 0.75rem;
            word-break: break-all;
            margin-top: 10px;
            padding: 8px;
            background: #232340;
            border-radius: 4px;
            border: 1px solid var(--player-color);
        }
        
        /* Mobile adjustments */
        @media (max-width: 1200px) {
            #game-container {
                flex-direction: column;
                align-items: center;
            }
            #shooter-canvas {
                max-width: 100%;
                max-height: 75vw;
                width: calc(100vw - 40px);
                height: calc((100vw - 40px) * 0.75); 
            }
        }
    </style>
</head>
<body>

    <div id="game-container">
        <!-- Oyun Alanı -->
        <div id="game-area" class="order-1">
            <h1 class="text-3xl font-extrabold mb-4 text-player-color">KARANLIK KUŞATMA</h1>
            <canvas id="shooter-canvas" width="800" height="600"></canvas>
            
            <!-- Basitleştirilmiş Kontrol Alanı (Canvas HUD'a taşındı) -->
            <div class="mt-4 p-4 bg-gray-700/50 rounded-lg text-sm w-full max-w-2xl flex flex-wrap justify-center items-center gap-4">
                <button id="start-button" class="p-3 bg-green-600 hover:bg-green-700 text-white font-bold rounded-lg transition text-lg w-full max-w-xs">
                    Oyunu Başlat
                </button>
            </div>
            
            <div class="mt-2 text-xs text-gray-300 w-full max-w-2xl text-center">
                Kontroller: WASD/Shift (Hareket) | Sol Tık (Ateş) | Q/R/F (Skiller) | E (Yükseltme)
            </div>
        </div>

        <!-- Bilgi Paneli (Sadece Silah Detayları ve Skorlar Kaldı) -->
        <div class="info-panel order-2">
            
            <div class="info-box weapon-status">
                <h3>Silah Durumu (E) - Detay</h3>
                <p>Mevcut Seviye: <span id="weapon-level" class="font-bold">1</span></p>
                <p>Ad: <span id="current-weapon-name">Tabanca</span></p>
                <p>Hasar: <span id="weapon-damage">10</span></p>
                <p>Ateş Hızı: <span id="weapon-rate">Yavaş</span></p>
                <!-- Yükseltme butonu HTML'den kaldırıldı -->
            </div>

            <div class="info-box high-scores">
                <h3>En İyi Skorlar</h3>
                <ol id="high-scores-list" class="list-decimal list-inside">
                    <li>Yükleniyor...</li>
                </ol>
            </div>

            <div class="text-xs text-gray-400 mt-4 w-full">
                Kullanıcı ID'niz (Skor kaydı için):
                <div id="user-id-display" class="user-id-display">Yükleniyor...</div>
            </div>
        </div>

    </div>

    <!-- Firebase SDK'ları -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, setDoc, onSnapshot, collection, query, limit } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        
        // --- FIREBASE AYARLARI VE SKOR YÖNETİMİ ---
        
        let app, db, auth;
        let userId = null;
        let isAuthReady = false;

        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : null;
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
        const COLLECTION_PATH = `/artifacts/${appId}/public/data/zombie_shooter_scores`;

        if (firebaseConfig) {
            setLogLevel('debug');
            app = initializeApp(firebaseConfig);
            db = getFirestore(app);
            auth = getAuth(app);

            onAuthStateChanged(auth, (user) => {
                if (user) {
                    userId = user.uid;
                } else if (!auth.currentUser) {
                    signInAnonymously(auth)
                        .then(userCredential => {
                            userId = userCredential.user.uid;
                        })
                        .catch(error => console.error("Anonim oturum açma hatası:", error));
                }
                isAuthReady = true;
                document.getElementById('user-id-display').textContent = userId ? userId : 'Anonim';
                if (userId) loadHighScores();
            });

            if (initialAuthToken) {
                signInWithCustomToken(auth, initialAuthToken)
                    .catch(error => console.log("Özel token ile oturum açma hatası:", error));
            }
        } else {
            console.error("Firebase yapılandırması eksik! Skorlar kaydedilemeyecek.");
            isAuthReady = true;
        }

        async function saveScore(score) {
            if (!isAuthReady || !userId || !db) return;

            const scoreData = {
                score: score,
                userId: userId,
                timestamp: Date.now(),
            };

            try {
                const userDocRef = doc(db, COLLECTION_PATH, userId);
                await setDoc(userDocRef, scoreData, { merge: true });
            } catch (e) {
                console.error("Skor kaydetme hatası: ", e);
            }
        }

        function loadHighScores() {
            if (!db) return;

            const q = query(
                collection(db, COLLECTION_PATH),
                limit(100)
            );

            onSnapshot(q, (querySnapshot) => {
                const highScoresList = document.getElementById('high-scores-list');
                highScoresList.innerHTML = '';
                
                if (querySnapshot.empty) {
                    highScoresList.innerHTML = '<li>Henüz skor yok. İlk siz olun!</li>';
                    return;
                }

                const scores = [];
                querySnapshot.forEach((doc) => {
                    scores.push(doc.data());
                });

                scores.sort((a, b) => b.score - a.score);

                const topScores = scores.slice(0, 10);

                let rank = 1;
                topScores.forEach((data) => {
                    const listItem = document.createElement('li');
                    
                    const isCurrentUser = data.userId === userId;
                    const name = isCurrentUser ? "Siz" : "Oyuncu " + data.userId.substring(0, 4);
                    
                    listItem.className = isCurrentUser ? 'font-extrabold text-green-300' : 'text-gray-300';

                    listItem.innerHTML = `
                        <span class="score-rank">${rank}. ${name}</span>
                        <span class="score-value float-right">${data.score}</span>
                    `;
                    highScoresList.appendChild(listItem);
                    rank++;
                });
            }, (error) => {
                console.error("Yüksek skorları yükleme hatası: ", error);
            });
        }


        // --- OYUN MANTIĞI BAŞLANGIÇ ---

        const canvas = document.getElementById('shooter-canvas');
        const ctx = canvas.getContext('2d');
        const W = canvas.width;
        const H = canvas.height;

        // HTML elementlerine artık sadece bilgi paneli için ihtiyacımız var, HUD canvas'a taşındı.
        const startButton = document.getElementById('start-button');
        const currentWeaponNameDisplay = document.getElementById('current-weapon-name');
        const weaponLevelDisplay = document.getElementById('weapon-level');
        const weaponDamageDisplay = document.getElementById('weapon-damage');
        const weaponRateDisplay = document.getElementById('weapon-rate');

        // Sabitler
        const MAX_BOMBS = 5;
        const MAX_HEALS = 2;
        const MAX_FEARS = 3; 
        const FEAR_DURATION = 5000; 
        const DIFFICULTY_INTERVAL = 30000;
        const ZOMBIE_SCORE_VALUE = 20; 
        const ZOMBIE_XP_VALUE = 20; 
        const PLAYER_COLOR = '#00bcd4';
        const BULLET_COLOR = '#ff9800';
        const BASE_ZOMBIE_HEALTH = 20; 
        
        // Boyutlar
        const TILE_SIZE = 12; 
        const PLAYER_RADIUS = 12; 
        const SPAWN_DISTANCE = 600; 

        // FPS Kontrol Sabitleri
        // Hareket düzeltmesi: FPS oranına göre daha tutarlı hız ve ivme
        const PLAYER_MAX_SPEED = 8; 
        const PLAYER_ACCELERATION = 0.8; 
        const PLAYER_FRICTION = 0.9; 
        const TARGET_FPS = 60;
        const FRAME_TIME = 1000 / TARGET_FPS;
        
        // Boost Sabitleri
        const BOOST_MULTIPLIER = 1.8; 
        const BOOST_COOLDOWN = 10000; 
        
        // XP Sabitleri: Level-up için gereken temel XP
        const BASE_XP_TO_LEVEL = 100;
        const XP_MULTIPLIER = 1.5; 

        // Oyun Durumu
        let isGameOver = true;
        let animationId;
        let score = 0;
        let bombs = MAX_BOMBS;
        let heals = MAX_HEALS;
        let fears = MAX_FEARS; 
        let lastTime = 0;
        let gameTime = 0;
        let difficultyLevel = 1;
        let keys = {}; 
        let currentXP = 0;
        let playerLevel = 1;

        // Kamera/Dünya Koordinatları
        let cameraX = 0;
        let cameraY = 0;
        
        // Boost Durumu
        let boostCooldownTimer = 0; 

        // Nesne Dizileri
        let enemies = [];
        let bullets = [];
        let particles = [];
        let lootItems = []; 
        let shockwaveEffect = null; 

        // Player (Ninja) - x, y artık DÜNYA koordinatlarıdır
        const player = {
            x: W / 2, 
            y: H / 2,
            radius: PLAYER_RADIUS, 
            health: 100,
            maxHealth: 100,
            color: PLAYER_COLOR,
            isShooting: false,
            lastShotTime: 0,
            targetX: W / 2, 
            targetY: H / 2, 
            damageTimer: 0,
            damageCooldown: 500,
            velocityX: 0, 
            velocityY: 0
        };

        // Silah Sistemi Tanımları
        const WEAPONS = [
            { name: "Tabanca", cost: 0, damage: 10, rate: 500, projectileSpeed: 5, color: '#ff9800' }, 
            { name: "SMG", cost: 500, damage: 6, rate: 100, projectileSpeed: 7, color: '#ff5722' },
            { name: "Pompalı", cost: 1500, damage: 25, rate: 800, projectileSpeed: 4, color: '#fdd835', spread: 5 },
            { name: "Otomatik Tüfek", cost: 4000, damage: 15, rate: 150, projectileSpeed: 8, color: '#4caf50' } 
        ];
        let weaponLevel = 1;
        let currentWeapon = WEAPONS[0];

        // --- Deneyim Hesaplama ---
        function getRequiredXP(level) {
            if (level === 1) return BASE_XP_TO_LEVEL;
            return Math.floor(BASE_XP_TO_LEVEL * (XP_MULTIPLIER ** (level - 1)));
        }

        function gainXP(amount) {
            currentXP += amount;
            
            let requiredXP = getRequiredXP(playerLevel);

            while (currentXP >= requiredXP) {
                currentXP -= requiredXP;
                playerLevel++;
                
                player.maxHealth += 10;
                player.health = player.maxHealth;
                
                requiredXP = getRequiredXP(playerLevel); 
            }
        }

        // Düşman (Zombie) Sınıfı
        class Enemy {
            constructor(x, y) {
                this.x = x; // Dünya Koordinatları
                this.y = y;
                this.radius = TILE_SIZE;
                this.health = BASE_ZOMBIE_HEALTH * difficultyLevel; 
                this.maxHealth = this.health;
                this.speed = (0.5 + difficultyLevel * 0.1); // Zombi hızı ayarlandı
                this.color = '#800080'; 
                this.fearTimer = 0; 
                this.stunTimer = 0; 
            }

            update(deltaTime) { 
                const dt = deltaTime / FRAME_TIME;

                if (this.stunTimer > 0) {
                    this.stunTimer -= deltaTime;
                    this.color = '#ffff00'; // Dondurma rengi sarı
                    return; // Dondurulmuşsa hareket etme
                }

                if (this.fearTimer > 0) {
                    this.fearTimer -= deltaTime;
                    
                    // Oyuncudan uzaklaş
                    const angle = Math.atan2(player.y - this.y, player.x - this.x) + Math.PI;
                    
                    this.x += Math.cos(angle) * this.speed * 1.5 * dt; 
                    this.y += Math.sin(angle) * this.speed * 1.5 * dt;
                    this.color = '#ff00ff'; 
                } else {
                    // Normal hareket
                    const angle = Math.atan2(player.y - this.y, player.x - this.x);
                    this.x += Math.cos(angle) * this.speed * dt;
                    this.y += Math.sin(angle) * this.speed * dt;
                    this.color = '#800080'; 
                }
            }

            draw() {
                // Dünya koordinatları zaten ctx.translate ile ekrana çevriliyor
                ctx.save();
                ctx.translate(this.x, this.y);

                // Zombi Silüeti
                ctx.fillStyle = this.color;
                ctx.beginPath();
                ctx.arc(0, 0, this.radius, 0, Math.PI * 2);
                ctx.fill();
                
                // Korku veya Dondurma efekti göstergeleri
                if (this.fearTimer > 0) {
                    ctx.font = '14px sans-serif';
                    ctx.fillText('😱', -5, -this.radius - 5);
                } else if (this.stunTimer > 0) {
                     ctx.font = '14px sans-serif';
                     ctx.fillText('✨', -5, -this.radius - 5);
                }


                // Gözler (Kırmızı)
                ctx.fillStyle = 'red';
                ctx.fillRect(-this.radius / 2, -this.radius / 3, 3, 3);
                ctx.fillRect(this.radius / 2 - 3, -this.radius / 3, 3, 3);
                
                // Ağız
                ctx.strokeStyle = 'darkred';
                ctx.lineWidth = 2;
                ctx.beginPath();
                ctx.moveTo(-this.radius / 2, this.radius / 3);
                ctx.lineTo(this.radius / 2, this.radius / 3);
                ctx.stroke();

                ctx.restore();
                
                // Sağlık Barı (Canvas translation aktif)
                const healthRatio = this.health / this.maxHealth;
                ctx.fillStyle = 'red';
                ctx.fillRect(this.x - this.radius, this.y - this.radius - 5, this.radius * 2, 3);
                ctx.fillStyle = 'lime';
                ctx.fillRect(this.x - this.radius, this.y - this.radius - 5, this.radius * 2 * healthRatio, 3);
            }
        }
        
        // Loot Sınıfı
        class Loot {
            constructor(x, y, type) {
                this.x = x;
                this.y = y;
                this.radius = 8; 
                this.type = type; 
                
                if (type === 'health') this.color = '#FF4500'; 
                else if (type === 'bomb') this.color = '#FFD700'; 
                else if (type === 'heal') this.color = '#00FF7F'; 
            }
            
            draw() {
                ctx.fillStyle = this.color;
                ctx.beginPath();
                ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
                ctx.fill();
                
                ctx.strokeStyle = 'black';
                ctx.lineWidth = 2;
                
                ctx.font = '10px Inter';
                ctx.textAlign = 'center';
                ctx.fillStyle = 'black';
                
                if (this.type === 'health') {
                    ctx.beginPath();
                    ctx.moveTo(this.x - 4, this.y);
                    ctx.lineTo(this.x + 4, this.y);
                    ctx.moveTo(this.x, this.y - 4);
                    ctx.lineTo(this.x, this.y + 4);
                    ctx.stroke();
                } else if (this.type === 'bomb') {
                    ctx.fillText('B', this.x, this.y + 3);
                } else if (this.type === 'heal') {
                    ctx.fillText('H', this.x, this.y + 3);
                }
            }
        }

        // Mermi (Bullet) Sınıfı
        class Bullet {
            constructor(x, y, angle) {
                this.x = x;
                this.y = y;
                this.radius = 3;
                this.damage = currentWeapon.damage;
                this.speed = currentWeapon.projectileSpeed;
                this.color = currentWeapon.color;
                this.velocityX = Math.cos(angle) * this.speed;
                this.velocityY = Math.sin(angle) * this.speed;
            }

            update(deltaTime) {
                const dt = deltaTime / FRAME_TIME;
                this.x += this.velocityX * dt;
                this.y += this.velocityY * dt;
            }

            draw() {
                ctx.fillStyle = this.color;
                ctx.beginPath();
                ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
                ctx.fill();
            }
        }

        // Parçacık (Particle) Sınıfı (Patlama Efekti)
        class Particle {
            constructor(x, y, color) {
                this.x = x;
                this.y = y;
                this.radius = Math.random() * 2 + 1;
                this.color = color;
                this.velocity = {
                    x: (Math.random() - 0.5) * (Math.random() * 6),
                    y: (Math.random() - 0.5) * (Math.random() * 6)
                };
                this.alpha = 1;
            }

            update(deltaTime) {
                const dt = deltaTime / FRAME_TIME;
                this.velocity.x *= 0.99;
                this.velocity.y *= 0.99;
                this.x += this.velocity.x * dt;
                this.y += this.velocityY * dt;
                this.alpha -= 0.01;
            }

            draw() {
                ctx.save();
                ctx.globalAlpha = this.alpha;
                ctx.fillStyle = this.color;
                ctx.beginPath();
                ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
                ctx.fill();
                ctx.restore();
            }
        }
        
        /**
         * Oyuncunun hareketini FPS tarzı hızlanma ve sürtünme ile günceller.
         */
        function movePlayer(deltaTime) {
            const dt = deltaTime / FRAME_TIME; // Delta time'ı 60FPS'e göre normalize et
            
            let targetVx = 0;
            let targetVy = 0;
            
            const boostActive = keys['shift'] && boostCooldownTimer === 0;
            const currentMaxSpeed = boostActive ? PLAYER_MAX_SPEED * BOOST_MULTIPLIER : PLAYER_MAX_SPEED;
            const currentAcceleration = boostActive ? PLAYER_ACCELERATION * BOOST_MULTIPLIER : PLAYER_ACCELERATION;
            
            if (keys['w']) targetVy -= currentAcceleration;
            if (keys['s']) targetVy += currentAcceleration;
            if (keys['a']) targetVx -= currentAcceleration;
            if (keys['d']) targetVx += currentAcceleration;

            if (targetVx !== 0 && targetVy !== 0) {
                // Çapraz hareketi normalleştirme (hızlanmayı korur)
                const magnitude = Math.sqrt(targetVx * targetVx + targetVy * targetVy);
                targetVx = (targetVx / magnitude) * currentAcceleration;
                targetVy = (targetVy / magnitude) * currentAcceleration;
            }

            // Hızı Sürtünme ve Yeni İvme ile güncelle
            player.velocityX = player.velocityX * PLAYER_FRICTION + targetVx * dt;
            player.velocityY = player.velocityY * PLAYER_FRICTION + targetVy * dt;

            const currentSpeed = Math.hypot(player.velocityX, player.velocityY);
            if (currentSpeed > currentMaxSpeed) {
                const ratio = currentMaxSpeed / currentSpeed;
                player.velocityX *= ratio;
                player.velocityY *= ratio;
            }
            
            // Konumu güncelle (Dünya Koordinatları)
            player.x += player.velocityX * dt;
            player.y += player.velocityY * dt;

            // Hızı sıfırla (Çok küçük değerleri sıfır kabul et)
            if (Math.abs(player.velocityX) < 0.1) player.velocityX = 0;
            if (Math.abs(player.velocityY) < 0.1) player.velocityY = 0;

            // Boost Soğuma Süresi Yönetimi
            if (boostActive && boostCooldownTimer === 0) {
                player.isBoosting = true;
                boostCooldownTimer = BOOST_COOLDOWN; 
            }
            if (!keys['shift'] && player.isBoosting) {
                player.isBoosting = false;
            }
            if (!boostActive && boostCooldownTimer > 0) {
                boostCooldownTimer = Math.max(0, boostCooldownTimer - deltaTime);
            }
        }


        /**
         * Oyuncuyu farenin hedefine doğru döndürerek ekranın ortasına çizer.
         */
        function drawPlayer() {
            // Oyuncunun ekran koordinatları
            const screenX = W / 2;
            const screenY = H / 2;
            
            // Hedef koordinatları mouse hareketinden geliyor (Ekran Koordinatları)
            const angle = Math.atan2(player.targetY - screenY, player.targetX - screenX);
            
            ctx.save();
            ctx.translate(player.x, player.y); // Dünya pozisyonuna çevir
            ctx.rotate(angle);

            // 1. Oyuncu gövdesi (Daire)
            ctx.fillStyle = player.color;
            ctx.beginPath();
            ctx.arc(0, 0, player.radius, 0, Math.PI * 2);
            ctx.fill();
            
            // 2. Ninja Yüzü/Kafa Bandı
            ctx.fillStyle = '#f0f0f0'; 
            ctx.beginPath();
            ctx.arc(0, 0, player.radius - 3, 0, Math.PI * 2);
            ctx.fill();

            ctx.fillStyle = '#ff5722'; 
            ctx.fillRect(-player.radius, -player.radius/3, player.radius * 2, player.radius/3);
            
            // 3. Gözler
            ctx.fillStyle = 'black';
            ctx.fillRect(-4, -1, 2, 2);
            ctx.fillRect(2, -1, 2, 2);


            // 4. Silahı/Nişan Hattını çiz
            const gunLength = player.radius + 5;
            const gunWidth = 3;
            
            ctx.fillStyle = currentWeapon.color; 
            ctx.fillRect(player.radius, -gunWidth / 2, gunLength, gunWidth); 
            
            ctx.restore();
            
            // Hasar soğuma efekti (Dünya koordinatlarında)
            if (player.damageTimer > 0) {
                const pulse = Math.abs(Math.sin(performance.now() / 50)) * 0.5;
                ctx.fillStyle = `rgba(255, 0, 0, ${pulse})`;
                ctx.beginPath();
                ctx.arc(player.x, player.y, player.radius + 5, 0, Math.PI * 2);
                ctx.fill();
            }
        }

        /**
         * Oyuncunun bulunduğu konumdan hedef koordinatlara doğru mermi fırlatır.
         */
        function shoot() {
            const now = Date.now();
            if (now - player.lastShotTime < currentWeapon.rate) {
                return;
            }
            player.lastShotTime = now;

            // Mermi açısı ekran koordinatlarından hesaplanır, dünya koordinatlarına göre fırlatılır
            const screenX = W / 2;
            const screenY = H / 2;
            const angle = Math.atan2(player.targetY - screenY, player.targetX - screenX);
            
            const startDistance = player.radius + 5; 
            const startX = player.x + Math.cos(angle) * startDistance; // Dünya koordinatları
            const startY = player.y + Math.sin(angle) * startDistance; // Dünya koordinatları


            if (currentWeapon.spread) { 
                for (let i = 0; i < currentWeapon.spread; i++) {
                    const spreadAngle = angle + (Math.random() - 0.5) * 0.3;
                    bullets.push(new Bullet(startX, startY, spreadAngle));
                }
            } else { 
                bullets.push(new Bullet(startX, startY, angle));
            }
        }

        /**
         * Oyuncunun mevcut dünya pozisyonuna göre rastgele bir yerde zombi oluşturur.
         */
        function spawnEnemy() {
            let x, y;
            
            // Oyuncudan en az SPAWN_DISTANCE uzakta spawn et
            const angle = Math.random() * Math.PI * 2;
            x = player.x + Math.cos(angle) * SPAWN_DISTANCE * (1 + Math.random());
            y = player.y + Math.sin(angle) * SPAWN_DISTANCE * (1 + Math.random());

            enemies.push(new Enemy(x, y));
        }

        let spawnIntervalId;

        /**
         * Oyunu başlatır veya yeniden başlatır.
         */
        function startGame() {
            if (animationId) cancelAnimationFrame(animationId);
            
            isGameOver = false;
            score = 0;
            bombs = MAX_BOMBS;
            heals = MAX_HEALS;
            fears = MAX_FEARS; 
            difficultyLevel = 1;
            gameTime = 0;
            currentXP = 0;
            playerLevel = 1;
            player.maxHealth = 100; 
            player.health = player.maxHealth;
            player.x = W / 2; 
            player.y = H / 2;
            player.velocityX = 0; 
            player.velocityY = 0; 
            player.isBoosting = false;
            boostCooldownTimer = 0;
            enemies = [];
            bullets = [];
            particles = [];
            lootItems = []; 
            shockwaveEffect = null;
            weaponLevel = 1;
            currentWeapon = WEAPONS[0];
            keys = {};

            updateUI();
            
            if (spawnIntervalId) clearInterval(spawnIntervalId);
            spawnIntervalId = setInterval(spawnEnemy, 1000 / (1 + difficultyLevel * 0.5)); // Zombi spawn etme aralığı
            
            startButton.textContent = "Oynuyor (WASD ile hareket et)";
            startButton.disabled = true;

            lastTime = performance.now();
            animationId = requestAnimationFrame(update);
        }
        
        /**
         * Her 30 saniyede bir zorluğu artırır.
         */
        function increaseDifficulty() {
            difficultyLevel++;
            console.log("Zorluk seviyesi arttı: ", difficultyLevel);
            clearInterval(spawnIntervalId);
            spawnIntervalId = setInterval(spawnEnemy, 1000 / (1 + difficultyLevel * 0.5));
        }

        /**
         * Bomba yeteneğini kullanır.
         */
        function useBomb() {
            if (isGameOver || bombs <= 0) return;

            bombs--;
            
            score += enemies.length * ZOMBIE_SCORE_VALUE; 
            gainXP(enemies.length * ZOMBIE_XP_VALUE); 
            enemies = [];
            
            canvas.style.opacity = 0.5;
            setTimeout(() => {
                 canvas.style.opacity = 1;
            }, 300);
        }
        
        /**
         * Can doldurma yeteneğini kullanır.
         */
        function useHeal() {
            if (isGameOver || heals <= 0) return;

            heals--;
            player.health = player.maxHealth;
        }

        /**
         * Korku yeteneğini kullanır.
         */
        function useFear() {
            if (isGameOver || fears <= 0) return;

            fears--;

            enemies.forEach(enemy => {
                enemy.fearTimer = FEAR_DURATION;
            });
        }
        
        /**
         * Silah yükseltmesi yapar.
         */
        function upgradeWeapon() {
            const nextLevel = weaponLevel + 1;
            if (nextLevel > WEAPONS.length) {
                return;
            }

            const nextWeapon = WEAPONS[weaponLevel]; 
            if (score >= nextWeapon.cost) {
                score -= nextWeapon.cost;
                weaponLevel = nextLevel;
                currentWeapon = nextWeapon;
            }
        }
        
        /**
         * Loot paketlerini toplama işlevini kontrol eder.
         */
        function collectLoot(loot, lIndex) {
            if (loot.type === 'health') {
                player.health = Math.min(player.maxHealth, player.health + 25);
            } else if (loot.type === 'bomb') {
                bombs = Math.min(MAX_BOMBS, bombs + 1);
            } else if (loot.type === 'heal') {
                heals = Math.min(MAX_HEALS, heals + 1);
            }
            
            // Not: Loot dizisinden silme işlemi update döngüsünde ters döngü ile yapılıyor.
        }

        function dropLoot(x, y) {
            const rand = Math.random();
            if (rand < 0.3) { 
                const itemRand = Math.random();
                let type;
                if (itemRand < 0.6) { 
                    type = 'health';
                } else if (itemRand < 0.8) { 
                    type = 'bomb';
                } else { 
                    type = 'heal';
                }
                lootItems.push(new Loot(x, y, type));
            }
        }
        
        /**
         * Kullanıcı arayüzünü (HTML/CSS) günceller.
         */
        function updateUI() {
            // HTML güncellemeleri (Sadece Bilgi Paneli için)
            
            weaponLevelDisplay.textContent = weaponLevel;
            currentWeaponNameDisplay.textContent = currentWeapon.name;
            weaponDamageDisplay.textContent = currentWeapon.damage;
            weaponRateDisplay.textContent = currentWeapon.rate < 200 ? 'Hızlı' : (currentWeapon.rate < 600 ? 'Orta' : 'Yavaş');
            
            // Yükseltme butonu HTML'den kaldırıldığı için burası basitleştirildi.
        }
        
        /**
         * Skill ikonlarını çizer.
         */
        function drawSkillIcon(x, y, count, key, symbol, disabled) {
            const size = 45;
            
            ctx.fillStyle = disabled ? 'rgba(35, 35, 64, 0.5)' : '#232340';
            ctx.strokeStyle = disabled ? '#5a5a7d55' : '#5a5a7d';
            ctx.lineWidth = 2;
            
            ctx.fillRect(x, y, size, size);
            ctx.strokeRect(x, y, size, size);

            // İkon
            ctx.font = '24px sans-serif';
            ctx.textAlign = 'center';
            ctx.fillStyle = disabled ? '#9ca3af' : 'white';
            ctx.fillText(symbol, x + size / 2, y + size / 2 + 5);

            // Sayaç
            ctx.fillStyle = '#ff5722';
            ctx.beginPath();
            ctx.arc(x + size, y, 10, 0, Math.PI * 2);
            ctx.fill();
            
            ctx.font = '12px Inter';
            ctx.fillStyle = 'white';
            ctx.fillText(count, x + size, y + 4);

            // Tuş
            ctx.fillStyle = 'rgba(0, 0, 0, 0.7)';
            ctx.fillRect(x + size - 20, y + size - 15, 20, 15);
            ctx.font = '10px Inter';
            ctx.fillStyle = 'white';
            ctx.fillText(key, x + size - 10, y + size - 4);
            
            ctx.textAlign = 'left';
        }
        
        /**
         * HUD (Heads-Up Display) öğelerini doğrudan Canvas üzerine çizer.
         */
        function drawHUD() {
            
            // 1. Can Barı (Sol Üst)
            const healthBarWidth = 200;
            const healthBarHeight = 15;
            const healthRatio = player.health / player.maxHealth;
            
            ctx.fillStyle = 'rgba(0, 0, 0, 0.5)';
            ctx.fillRect(10, 10, healthBarWidth + 40, healthBarHeight + 10);
            
            ctx.fillStyle = 'gray';
            ctx.fillRect(30, 15, healthBarWidth, healthBarHeight);
            ctx.fillStyle = 'red';
            ctx.fillRect(30, 15, healthBarWidth * healthRatio, healthBarHeight);

            ctx.fillStyle = 'white';
            ctx.font = '14px Inter';
            ctx.fillText(`CAN: ${Math.max(0, player.health)}/${player.maxHealth}`, 35, 28);
            
            // Kalp Simgesi (Ekstra)
            ctx.font = '20px sans-serif';
            ctx.fillText('❤️', 5, 30);
            
            
            // 2. XP Barı (Sağ Üst)
            const xpBarWidth = 250;
            const xpBarHeight = 15;
            const requiredXP = getRequiredXP(playerLevel);
            const xpRatio = currentXP / requiredXP;
            const xpX = W - xpBarWidth - 10;
            
            ctx.fillStyle = 'rgba(0, 0, 0, 0.5)';
            ctx.fillRect(xpX - 5, 10, xpBarWidth + 10, xpBarHeight + 10);
            
            ctx.fillStyle = '#232340';
            ctx.fillRect(xpX, 15, xpBarWidth, xpBarHeight);

            ctx.fillStyle = '#ff00ff'; // Mor XP dolgusu
            ctx.fillRect(xpX, 15, xpBarWidth * xpRatio, xpBarHeight);

            ctx.fillStyle = 'white';
            ctx.font = '12px Inter';
            ctx.textAlign = 'center';
            ctx.fillText(`LVL ${playerLevel} - ${currentXP}/${requiredXP} XP`, xpX + xpBarWidth / 2, 28);
            
            
            // 3. Skor (Orta Üst)
            ctx.fillStyle = 'white';
            ctx.font = '24px Inter';
            ctx.textAlign = 'center';
            ctx.fillText(`SKOR: ${score}`, W / 2, 30);
            
            // 4. Boost Barı (Sol Alt)
            const boostBarWidth = 120;
            const boostBarHeight = 15;
            const boostX = 10;
            const boostY = H - 25;
            
            ctx.fillStyle = 'rgba(0, 0, 0, 0.5)';
            ctx.fillRect(boostX, boostY - 5, boostBarWidth, boostBarHeight + 10);
            
            const boostRatio = boostCooldownTimer / BOOST_COOLDOWN;
            const boostFill = boostCooldownTimer === 0 ? boostBarWidth : Math.max(0, boostBarWidth * (1 - boostRatio));
            
            ctx.fillStyle = '#0e7490';
            ctx.fillRect(boostX, boostY, boostBarWidth, boostBarHeight);
            
            ctx.fillStyle = boostCooldownTimer === 0 ? '#38bdf8' : '#38bdf844';
            ctx.fillRect(boostX, boostY, boostFill, boostBarHeight);
            
            ctx.fillStyle = 'white';
            ctx.font = '10px Inter';
            ctx.textAlign = 'center';
            
            const boostLabel = boostCooldownTimer === 0 ? `BOOST (Shift)` : `DOLUYOR... (${Math.ceil(boostCooldownTimer / 1000)}s)`;
            ctx.fillText(boostLabel, boostX + boostBarWidth / 2, boostY + 11);
            
            
            // 5. Silah Durumu ve Yükseltme (XP Barının Altı - Canvas)
            const weaponBoxWidth = 250;
            const weaponBoxHeight = 55;
            const weaponX = W - weaponBoxWidth - 10;
            const weaponY = 55; // XP barının 10px altına

            ctx.fillStyle = 'rgba(0, 0, 0, 0.5)';
            ctx.fillRect(weaponX, weaponY, weaponBoxWidth + 10, weaponBoxHeight + 10);
            
            // Silah Detayları
            ctx.fillStyle = 'white';
            ctx.font = '14px Inter';
            ctx.textAlign = 'left';
            ctx.fillText(`LVL ${weaponLevel}: ${currentWeapon.name}`, weaponX + 5, weaponY + 18);
            ctx.fillText(`HASAR: ${currentWeapon.damage}`, weaponX + 5, weaponY + 38);
            ctx.fillText(`AT. HIZI: ${currentWeapon.rate < 200 ? 'Hızlı' : (currentWeapon.rate < 600 ? 'Orta' : 'Yavaş')}`, weaponX + 130, weaponY + 38);
            
            // Yükseltme Durumu
            const nextWeaponIndex = weaponLevel;
            const upgradeY = weaponY + 40;
            
            if (nextWeaponIndex < WEAPONS.length) {
                const nextWeapon = WEAPONS[nextWeaponIndex];
                const canUpgrade = score >= nextWeapon.cost;
                
                ctx.fillStyle = canUpgrade ? 'rgba(52, 211, 153, 0.3)' : 'rgba(185, 28, 28, 0.3)'; // Kırmızı/Yeşil arka plan
                ctx.fillRect(weaponX, upgradeY, weaponBoxWidth + 10, 25); 

                ctx.fillStyle = canUpgrade ? '#34d399' : '#f87171'; // Yeşil/Kırmızı yazı
                ctx.font = '12px Inter';
                ctx.textAlign = 'center';
                
                const upgradeText = `YÜKSELT (E): ${nextWeapon.name} (${nextWeapon.cost} Puan)`;
                ctx.fillText(upgradeText, weaponX + weaponBoxWidth / 2 + 5, upgradeY + 15);
            } else {
                ctx.fillStyle = 'rgba(5, 150, 105, 0.5)';
                ctx.fillRect(weaponX, upgradeY, weaponBoxWidth + 10, 25);
                ctx.fillStyle = '#059669';
                ctx.font = '12px Inter';
                ctx.textAlign = 'center';
                ctx.fillText('MAKS SİLAH SEVİYESİ', weaponX + weaponBoxWidth / 2 + 5, upgradeY + 15);
            }
            
            
            // 6. Skill Barı (Sağ Alt) - Canvas Üzerine Çizim
            const skillIconSize = 45;
            const skillSpacing = 55; // 45 + 10 boşluk
            const totalSkillWidth = (3 * skillIconSize) + (2 * 10); // 3 skill, 2 boşluk
            const skillStartX = W - totalSkillWidth - 25; // Sağ kenardan 25px boşluk
            const skillStartY = H - 60;
            
            // Q - Bomba
            drawSkillIcon(skillStartX, skillStartY, bombs, 'Q', '💣', bombs <= 0);
            // R - Can Doldurma
            drawSkillIcon(skillStartX + skillSpacing, skillStartY, heals, 'R', '❤️', heals <= 0);
            // F - Korku
            drawSkillIcon(skillStartX + skillSpacing * 2, skillStartY, fears, 'F', '😱', fears <= 0);

            // Text alignment'ı resetle
            ctx.textAlign = 'left';
        }


        /**
         * Ana oyun döngüsü.
         */
        function update(currentTime) {
            if (isGameOver) {
                clearInterval(spawnIntervalId);
                return;
            }

            const deltaTime = currentTime - lastTime;
            lastTime = currentTime;
            gameTime += deltaTime;
            player.damageTimer = Math.max(0, player.damageTimer - deltaTime);

            if (gameTime >= difficultyLevel * DIFFICULTY_INTERVAL) {
                increaseDifficulty();
            }

            movePlayer(deltaTime);

            // Kamera takibi (Oyuncuyu ekranın merkezine sabitle)
            cameraX = player.x - W / 2;
            cameraY = player.y - H / 2;


            // Ekranı temizle
            ctx.fillStyle = 'rgba(15, 15, 28, 0.3)';
            ctx.fillRect(0, 0, W, H);
            
            // --- Oyun Dünya Çizimine Başla ---
            ctx.save();
            ctx.translate(-cameraX, -cameraY); // Kamerayı uygula
            
            // 1. Otomatik ateş etme (eğer mouse basılıysa)
            if (player.isShooting) {
                shoot();
            }

            // 2. Loot Paketleri Güncelleme ve Çizme (GÜVENLİ TERS DÖNGÜ)
            for (let i = lootItems.length - 1; i >= 0; i--) {
                const loot = lootItems[i];
                loot.draw();
                // Oyuncu-Loot Çarpışması
                const dist = Math.hypot(player.x - loot.x, player.y - loot.y);
                if (dist - player.radius - loot.radius < 1) {
                    collectLoot(loot, i); // Logiği çalıştır
                    lootItems.splice(i, 1); // Güvenli silme
                }
            }


            // 3. Mermileri güncelle (GÜVENLİ TERS DÖNGÜ)
            for (let i = bullets.length - 1; i >= 0; i--) {
                const bullet = bullets[i];
                bullet.update(deltaTime); 

                let hit = false;
                
                // Mermi-Zombi Çarpışması (GÜVENLİ TERS DÖNGÜ)
                for (let j = enemies.length - 1; j >= 0; j--) {
                    const enemy = enemies[j];
                    const dist = Math.hypot(bullet.x - enemy.x, bullet.y - enemy.y);
                    
                    if (dist - bullet.radius - enemy.radius < 1) {
                        enemy.health -= bullet.damage;
                        hit = true;
                        
                        if (enemy.health <= 0) {
                            // Enemy death logic
                            score += ZOMBIE_SCORE_VALUE; 
                            gainXP(ZOMBIE_XP_VALUE); 
                            for (let k = 0; k < 15; k++) {
                                particles.push(new Particle(enemy.x, enemy.y, 'lime'));
                            }
                            
                            dropLoot(enemy.x, enemy.y);
                            enemies.splice(j, 1); // Güvenli silme
                        }
                        break; // Mermi tek bir zombiye vurabilir
                    }
                }

                // Mermi yok etme kontrolü
                if (hit || Math.abs(bullet.x - player.x) > W || Math.abs(bullet.y - player.y) > H) {
                    bullets.splice(i, 1); // Güvenli silme
                    continue;
                }
                bullet.draw();
            }

            // 4. Düşmanları güncelle (Sadece pozisyon/durum güncellemesi)
            for (let i = enemies.length - 1; i >= 0; i--) {
                const enemy = enemies[i];
                // Eğer şok dalgasıyla öldürülmediyse stun/fear timer'larını güncelle
                enemy.update(deltaTime); 

                // Düşman-Oyuncu Çarpışması
                const dist = Math.hypot(player.x - enemy.x, player.y - enemy.y);
                if (dist - player.radius - enemy.radius < 1) {
                    if (player.damageTimer <= 0) {
                        player.health -= 10; 
                        player.damageTimer = player.damageCooldown; 
                        
                        if (player.health <= 0) {
                            isGameOver = true;
                            saveScore(score);
                            gameOverScreen();
                            return;
                        }
                        // Zombiyi geri it
                        const angle = Math.atan2(enemy.y - player.y, enemy.x - player.x);
                        enemy.x += Math.cos(angle) * 30;
                        enemy.y += Math.sin(angle) * 30;
                    }
                }

                enemy.draw();
            }
            
            // 5. Parçacıkları güncelle (GÜVENLİ TERS DÖNGÜ)
            for (let i = particles.length - 1; i >= 0; i--) {
                const particle = particles[i];
                if (particle.alpha <= 0) {
                    particles.splice(i, 1); // Güvenli silme
                } else {
                    particle.update(deltaTime);
                    particle.draw();
                }
            }

            drawPlayer(); // Oyuncuyu diğer her şeyin üstüne çiz

            ctx.restore(); // Kamera çevirisini geri al

            // --- HUD Çizimine Başla ---
            drawHUD();

            updateUI(); // HTML elementlerini güncelle (Silah/Upgrade)
            animationId = requestAnimationFrame(update);
        }

        /**
         * Oyun bitti ekranını gösterir.
         */
        function gameOverScreen() {
            ctx.fillStyle = 'rgba(0, 0, 0, 0.9)';
            ctx.fillRect(0, 0, W, H);
            ctx.fillStyle = 'red';
            ctx.font = '60px Inter';
            ctx.textAlign = 'center';
            ctx.fillText('OYUN BİTTİ!', W / 2, H / 2 - 60);
            ctx.fillStyle = 'white';
            ctx.font = '30px Inter';
            ctx.fillText(`Skorunuz: ${score}`, W / 2, H / 2);
            ctx.font = '24px Inter';
            ctx.fillText(`Ulaştığınız Seviye: ${playerLevel}`, W / 2, H / 2 + 40);
            startButton.textContent = "Tekrar Oyna";
            startButton.disabled = false;
        }


        // --- ETKİNLİK DİNLEYİCİLERİ ---

        // Mouse hareketini izle (Nişan alma)
        canvas.addEventListener('mousemove', (event) => {
            const rect = canvas.getBoundingClientRect();
            const scaleX = canvas.width / rect.width;
            const scaleY = canvas.height / rect.height;

            // Target X ve Y ekran koordinatlarıdır (HUD çizimi için)
            player.targetX = (event.clientX - rect.left) * scaleX;
            player.targetY = (event.clientY - rect.top) * scaleY;
        });

        // Mouse basma/bırakma (Ateş etme)
        canvas.addEventListener('mousedown', () => {
            if (!isGameOver) player.isShooting = true;
        });
        canvas.addEventListener('mouseup', () => {
            player.isShooting = false;
        });
        
        // Klavye olayları (Hareket ve Skiller)
        document.addEventListener('keydown', (event) => {
            const key = event.key.toLowerCase();

            // WASD/Shift'in tarayıcıda scroll yapmasını engeller
            if (['w', 'a', 's', 'd', 'q', 'r', 'f', 'e', 'shift'].includes(key)) {
                event.preventDefault(); 
            }
            
            keys[key] = true;
            
            if (isGameOver) return;
            
            if (key === 'q') {
                useBomb();
            } else if (key === 'r') {
                useHeal();
            } else if (key === 'f') { 
                useFear();
            } else if (key === 'e') {
                upgradeWeapon();
            }
        });

        document.addEventListener('keyup', (event) => {
            keys[event.key.toLowerCase()] = false;
        });
        
        // Buton Dinleyicileri (Sadece HTML Butonları Kaldı)
        startButton.addEventListener('click', startGame);

        // Pencere yüklendiğinde başlatma
        window.onload = function () {
            currentWeapon = WEAPONS[0];
            updateUI(); 
            
            ctx.fillStyle = 'white';
            ctx.font = '30px Inter';
            ctx.textAlign = 'center';
            ctx.fillText('BAŞLAMAK İÇİN TIKLA', W / 2, H / 2);
            ctx.textAlign = 'left';
        };

        // --- OYUN MANTIĞI SONU ---
    </script>
</body>
</html>
