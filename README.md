<script type="module">
        // Firebase ile ilgili içe aktarmalar buraya kopyalanmıştır.
        // Ancak bu kod, GitHub Pages'de çalışmadığı için Firebase bağlantısını atlayacaktır.
        
        let app, db, auth;
        let userId = 'GitHubAnonimKullanici'; // Sabit bir ID atandı
        let isAuthReady = true;

        // Firebase yapılandırması kontrolü kaldırıldı. Varsayılan olarak skor sistemi devredışı.
        console.warn("Firebase (Skor Sistemi) devre dışı. Oyunun ana mantığı çalıştırılıyor.");

        // Skor Kaydetme Fonksiyonu (Etkisizleştirildi)
        async function saveScore(score) {
            console.log("Skor kaydetme devre dışı. Kaydedilecek Skor:", score);
            // Gerçek kaydetme işlemi yapılmayacak.
        }

        // Yüksek Skorları Yükleme Fonksiyonu (Mock Data Kullanacak)
        function loadHighScores() {
            const highScoresList = document.getElementById('high-scores-list');
            highScoresList.innerHTML = '';
            
            // Mock (Sahte) skor listesi
            const mockScores = [
                { score: 5000, userId: 'Oyuncu A' },
                { score: 3500, userId: 'Oyuncu B' },
                { score: 2100, userId: 'Oyuncu C' },
            ];

            let rank = 1;
            mockScores.forEach((data) => {
                const listItem = document.createElement('li');
                const name = data.userId;
                
                listItem.className = 'text-gray-300';
                listItem.innerHTML = `
                    <span class="score-rank">${rank}. ${name}</span>
                    <span class="score-value float-right">${data.score}</span>
                `;
                highScoresList.appendChild(listItem);
                rank++;
            });
            document.getElementById('user-id-display').textContent = userId;
        }
        
        // --- OYUN MANTIĞI BAŞLANGIÇ ---
        
        // **********************************************
        // Mevcut oyun mantığınızın geri kalan kodunu buraya kopyalayın.
        // **********************************************
        
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
