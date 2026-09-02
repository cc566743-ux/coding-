<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>3D Doom Zombie Shooter</title>
    <style>
        body {
            margin: 0;
            overflow: hidden;
            background-color: #000;
            font-family: 'Courier New', Courier, monospace;
            color: #fff;
            user-select: none;
        }
        canvas {
            display: block;
            width: 100vw;
            height: 100vh;
        }
        #ui-layer {
            position: absolute;
            top: 0; left: 0; width: 100%; height: 100%;
            pointer-events: none;
            display: none;
        }
        #crosshair {
            position: absolute;
            top: 50%; left: 50%;
            width: 20px; height: 20px;
            margin-left: -10px; margin-top: -10px;
            text-align: center; line-height: 20px;
            font-size: 24px; color: rgba(0, 255, 0, 0.8);
            font-weight: bold;
        }
        .hud-box {
            position: absolute;
            bottom: 20px;
            background: rgba(255, 0, 0, 0.2);
            border: 2px solid #a00;
            padding: 10px 20px;
            font-size: 24px;
            text-shadow: 2px 2px 0 #000;
            border-radius: 6px;
        }
        #health-hud { left: 20px; }
        #ammo-hud { right: 20px; text-align: right; }
        #weapon-hud { right: 20px; bottom: 80px; font-size: 18px; color: #aaa; }
        #pickup-prompt {
            position: absolute;
            top: 60%; left: 50%;
            transform: translate(-50%, -50%);
            font-size: 22px; color: yellow;
            background: rgba(0,0,0,0.7);
            padding: 10px 20px; display: none;
            border: 1px solid yellow;
            border-radius: 8px;
        }
        
        #menu-layer {
            position: absolute;
            top: 0; left: 0; width: 100%; height: 100%;
            background-color: rgba(0, 0, 0, 0.88);
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            z-index: 10;
        }
        h1 { color: #f00; font-size: 56px; margin-bottom: 10px; text-transform: uppercase; letter-spacing: 4px; text-align: center; }
        .instructions { font-size: 18px; line-height: 1.6; text-align: center; color: #ccc; margin-bottom: 30px; }
        .key { background: #333; padding: 2px 8px; border-radius: 4px; border: 1px solid #666; color: white;}
        button {
            padding: 15px 40px; font-size: 22px; font-family: inherit; font-weight: bold;
            background: #a00; color: #fff; border: 2px solid #f00; cursor: pointer;
            border-radius: 6px; transition: 0.2s;
        }
        button:hover { background: #f00; color: #000; box-shadow: 0 0 15px #f00; }
        
        #damage-flash {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            background-color: rgba(255, 0, 0, 0.4);
            opacity: 0; pointer-events: none; transition: opacity 0.1s;
        }
        #score-hud { position: absolute; top: 20px; left: 20px; font-size: 22px; text-shadow: 2px 2px 0 #000; }
        #boss-hud { position: absolute; top: 50px; left: 20px; font-size: 18px; color: #ffcc00; text-shadow: 2px 2px 0 #000; font-weight: bold; }
    </style>
    <!-- Include Three.js library -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
</head>
<body>

    <div id="ui-layer">
        <div id="crosshair">+</div>
        <div id="score-hud">Kills: <span id="score-val">0</span></div>
        <div id="boss-hud">Next Boss: <span id="boss-val">5 Kills</span></div>
        <div id="health-hud" class="hud-box">HP: <span id="hp-val">100</span></div>
        <div id="weapon-hud"><span id="wep-name">Pistol</span></div>
        <div id="ammo-hud" class="hud-box">AMMO: <span id="ammo-val">15/15</span></div>
        <div id="pickup-prompt">Press <span class="key">Q</span> to Pickup</div>
        <div id="damage-flash"></div>
    </div>

    <div id="menu-layer">
        <h1 id="menu-title">DOOM SURVIVAL</h1>
        <div class="instructions">
            Survive the Endless Zombie Horde.<br><br>
            <span class="key">W</span> <span class="key">A</span> <span class="key">S</span> <span class="key">D</span> to Move &nbsp;|&nbsp; <b>Mouse</b> to Aim<br>
            <b>Left Click</b> to Shoot &nbsp;|&nbsp; <span class="key">R</span> to Reload<br>
            <span class="key">Q</span> to Pickup Weapons (Blue Crates)
        </div>
        <button id="start-btn">CLICK TO PLAY</button>
    </div>

    <script>
        let camera, scene, renderer;
        let pitchObject, yawObject;
        
        // Game State
        let isGameRunning = false;
        let score = 0;
        let killsSinceLastBoss = 0;
        let zombies = [];
        let walls = [];
        let pickups = [];
        let lastTime = performance.now();
        let isMouseDown = false;
        let gameTime = 0; // Tracks active gameplay time in seconds
        let clearedThreeMinCrates = false;
        
        // Player State & Fine-tuned Collision Cylinder
        const player = {
            height: 2.0,
            radius: 0.8,
            baseSpeed: 15,
            hp: 100,
            maxHp: 100,
            velocity: new THREE.Vector3(),
            direction: new THREE.Vector3(),
            isReloading: false
        };

        // Weapons Data
        // Speed Multipliers: Pistol & Shotgun = +15% speed (1.15x), MG = -15% speed (0.85x) & -45% when firing (0.55x stack = 0.4675x)
        const weapons = {
            pistol:  { name: "Pistol", damage: 25, clip: 15, maxClip: 15, fireRate: 200, color: 0x999999, auto: false, reloadTime: 900, moveSpeedMult: 1.15, firingSpeedMult: 1.0 },
            shotgun: { name: "Shotgun", damage: 100, clip: 8, maxClip: 8, fireRate: 750, color: 0xaa5500, auto: false, reloadTime: 900, moveSpeedMult: 1.15, firingSpeedMult: 1.0 },
            mg:      { name: "Machine Gun", damage: 50, clip: 40, maxClip: 40, fireRate: 110, color: 0x333333, auto: true, reloadTime: 5000, moveSpeedMult: 0.85, firingSpeedMult: 0.55 }
        };
        let currentWeaponKey = 'pistol';
        let currentWeapon = weapons.pistol;
        let lastFireTime = 0;
        let nearestPickup = null;
        let gunMesh, recoil = 0;

        // Input Controls State
        const keys = { w: false, a: false, s: false, d: false, q: false, r: false };
        const PI_2 = Math.PI / 2;

        // Map layout (1 = wall, 0 = empty walkway)
        const mapGrid = [
            [1,1,1,1,1,1,1,1,1,1,1,1,1,1,1],
            [1,0,0,0,1,0,0,0,0,0,1,0,0,0,1],
            [1,0,1,0,1,0,1,1,1,0,1,0,1,0,1],
            [1,0,1,0,0,0,0,0,1,0,0,0,1,0,1],
            [1,0,1,1,1,1,1,0,1,1,1,1,1,0,1],
            [1,0,0,0,0,0,0,0,0,0,0,0,0,0,1],
            [1,0,1,1,1,0,1,1,1,0,1,1,1,0,1],
            [1,0,1,0,0,0,1,0,0,0,1,0,0,0,1],
            [1,0,1,0,1,0,1,0,1,0,1,0,1,0,1],
            [1,0,0,0,1,0,0,0,1,0,0,0,1,0,1],
            [1,1,1,1,1,1,1,1,1,1,1,1,1,1,1]
        ];
        const tileSize = 10;

        // UI Element References
        const uiLayer = document.getElementById('ui-layer');
        const menuLayer = document.getElementById('menu-layer');
        const startBtn = document.getElementById('start-btn');
        const hpVal = document.getElementById('hp-val');
        const ammoVal = document.getElementById('ammo-val');
        const wepName = document.getElementById('wep-name');
        const scoreVal = document.getElementById('score-val');
        const bossVal = document.getElementById('boss-val');
        const pickupPrompt = document.getElementById('pickup-prompt');
        const damageFlash = document.getElementById('damage-flash');
        const menuTitle = document.getElementById('menu-title');

        window.onload = init;

        function createGunMesh(type) {
            const group = new THREE.Group();
            if (type === 'mg') {
                const body = new THREE.Mesh(
                    new THREE.BoxGeometry(0.14, 0.18, 0.7),
                    new THREE.MeshLambertMaterial({ color: 0x222222 })
                );
                body.position.set(0, 0, -0.2);
                
                const longBarrel = new THREE.Mesh(
                    new THREE.CylinderGeometry(0.04, 0.04, 0.8, 8),
                    new THREE.MeshLambertMaterial({ color: 0x111111 })
                );
                longBarrel.rotation.x = Math.PI / 2;
                longBarrel.position.set(0, 0.02, -0.75);

                const mag = new THREE.Mesh(
                    new THREE.BoxGeometry(0.08, 0.35, 0.15),
                    new THREE.MeshLambertMaterial({ color: 0x555522 })
                );
                mag.position.set(0, -0.2, -0.25);
                mag.rotation.x = 0.2;

                const stock = new THREE.Mesh(
                    new THREE.BoxGeometry(0.12, 0.2, 0.35),
                    new THREE.MeshLambertMaterial({ color: 0x4a2a18 })
                );
                stock.position.set(0, -0.05, 0.2);

                const sight = new THREE.Mesh(
                    new THREE.BoxGeometry(0.06, 0.08, 0.2),
                    new THREE.MeshLambertMaterial({ color: 0x666666 })
                );
                sight.position.set(0, 0.12, -0.2);

                group.add(body, longBarrel, mag, stock, sight);
            } else if (type === 'shotgun') {
                const barrel = new THREE.Mesh(
                    new THREE.BoxGeometry(0.18, 0.12, 0.8),
                    new THREE.MeshLambertMaterial({ color: 0x444444 })
                );
                barrel.position.set(0, 0, -0.4);

                const pump = new THREE.Mesh(
                    new THREE.BoxGeometry(0.16, 0.14, 0.25),
                    new THREE.MeshLambertMaterial({ color: 0x7c4923 })
                );
                pump.position.set(0, -0.05, -0.45);

                const grip = new THREE.Mesh(
                    new THREE.BoxGeometry(0.1, 0.22, 0.35),
                    new THREE.MeshLambertMaterial({ color: 0x7c4923 })
                );
                grip.position.set(0, -0.1, 0.1);
                grip.rotation.x = -0.3;

                group.add(barrel, pump, grip);
            } else {
                const barrel = new THREE.Mesh(
                    new THREE.BoxGeometry(0.11, 0.12, 0.45),
                    new THREE.MeshLambertMaterial({ color: 0x888888 })
                );
                barrel.position.set(0, 0, -0.2);

                const grip = new THREE.Mesh(
                    new THREE.BoxGeometry(0.09, 0.25, 0.14),
                    new THREE.MeshLambertMaterial({ color: 0x111111 })
                );
                grip.position.set(0, -0.15, -0.05);
                grip.rotation.x = -Math.PI / 8;

                group.add(barrel, grip);
            }
            return group;
        }

        function updateGunMesh() {
            if (gunMesh) pitchObject.remove(gunMesh);
            gunMesh = createGunMesh(currentWeaponKey);
            gunMesh.position.set(0.35, -0.3, -0.7);
            pitchObject.add(gunMesh);
        }

        function init() {
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x0a0303);
            scene.fog = new THREE.FogExp2(0x0a0303, 0.035);

            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            
            pitchObject = new THREE.Object3D();
            pitchObject.add(camera);
            yawObject = new THREE.Object3D();
            yawObject.position.y = player.height;
            yawObject.add(pitchObject);
            scene.add(yawObject);

            updateGunMesh();

            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.setPixelRatio(window.devicePixelRatio);
            document.body.appendChild(renderer.domElement);

            const ambientLight = new THREE.AmbientLight(0x555555);
            scene.add(ambientLight);
            const dirLight = new THREE.DirectionalLight(0xffaaaa, 0.6);
            dirLight.position.set(10, 20, 10);
            scene.add(dirLight);

            buildMap();

            window.addEventListener('resize', onWindowResize, false);
            document.addEventListener('keydown', onKeyDown);
            document.addEventListener('keyup', onKeyUp);
            document.addEventListener('mousedown', onMouseDown);
            document.addEventListener('mouseup', onMouseUp);
            document.addEventListener('mousemove', onMouseMove);
            
            startBtn.addEventListener('click', () => {
                document.body.requestPointerLock();
            });

            document.addEventListener('pointerlockchange', () => {
                if (document.pointerLockElement === document.body) {
                    isGameRunning = true;
                    menuLayer.style.display = 'none';
                    uiLayer.style.display = 'block';
                    if (player.hp <= 0) resetGame();
                } else {
                    isGameRunning = false;
                    menuLayer.style.display = 'flex';
                    uiLayer.style.display = 'none';
                    if (player.hp > 0) startBtn.innerText = "RESUME GAME";
                }
            });

            setInterval(spawnZombie, 1200);
            setInterval(spawnPickup, 30000);

            spawnPickup();

            updateHUD();
            animate();
        }

        function buildMap() {
            const floorGeo = new THREE.PlaneGeometry(200, 200);
            const floorMat = new THREE.MeshLambertMaterial({ color: 0x222222 });
            const floor = new THREE.Mesh(floorGeo, floorMat);
            floor.rotation.x = -Math.PI / 2;
            scene.add(floor);
            
            const ceilMat = new THREE.MeshLambertMaterial({ color: 0x111111 });
            const ceiling = new THREE.Mesh(floorGeo, ceilMat);
            ceiling.rotation.x = Math.PI / 2;
            ceiling.position.y = tileSize;
            scene.add(ceiling);

            const wallGeo = new THREE.BoxGeometry(tileSize, tileSize, tileSize);
            const wallMat = new THREE.MeshLambertMaterial({ color: 0x444444 });
            
            const offsetX = (mapGrid[0].length * tileSize) / 2;
            const offsetZ = (mapGrid.length * tileSize) / 2;

            for (let z = 0; z < mapGrid.length; z++) {
                for (let x = 0; x < mapGrid[z].length; x++) {
                    const posX = x * tileSize - offsetX + tileSize/2;
                    const posZ = z * tileSize - offsetZ + tileSize/2;

                    if (mapGrid[z][x] === 1) {
                        const wall = new THREE.Mesh(wallGeo, wallMat);
                        wall.position.set(posX, tileSize/2, posZ);
                        scene.add(wall);
                        
                        wall.geometry.computeBoundingBox();
                        wall.updateMatrixWorld();
                        wall.boundingBox = new THREE.Box3().setFromObject(wall);
                        walls.push(wall);
                    } else if (mapGrid[z][x] === 0 && yawObject.position.x === 0 && yawObject.position.z === 0) {
                        yawObject.position.set(posX, player.height, posZ);
                    }
                }
            }
        }

        function onKeyDown(e) {
            const key = e.key.toLowerCase();
            if (keys.hasOwnProperty(key)) keys[key] = true;
            if (key === 'r') reload();
            if (key === 'q') tryPickupWeapon();
        }
        function onKeyUp(e) {
            const key = e.key.toLowerCase();
            if (keys.hasOwnProperty(key)) keys[key] = false;
        }
        function onMouseMove(e) {
            if (!isGameRunning) return;
            const movementX = e.movementX || 0;
            const movementY = e.movementY || 0;
            
            yawObject.rotation.y -= movementX * 0.0022;
            pitchObject.rotation.x -= movementY * 0.0022;
            pitchObject.rotation.x = Math.max(-PI_2, Math.min(PI_2, pitchObject.rotation.x));
        }
        function onMouseDown(e) {
            if (!isGameRunning || e.button !== 0) return;
            isMouseDown = true;
            shoot();
        }
        function onMouseUp(e) {
            if (e.button === 0) isMouseDown = false;
        }

        function shoot() {
            if (player.isReloading || currentWeapon.clip <= 0) return;
            const now = performance.now();
            if (now - lastFireTime < currentWeapon.fireRate) return;

            lastFireTime = now;
            currentWeapon.clip--;
            updateHUD();

            recoil = 0.15;

            uiLayer.style.backgroundColor = "rgba(255,255,150,0.15)";
            setTimeout(() => uiLayer.style.backgroundColor = "transparent", 40);

            const raycaster = new THREE.Raycaster();
            const direction = new THREE.Vector3(0, 0, -1).applyQuaternion(camera.getWorldQuaternion(new THREE.Quaternion()));
            raycaster.set(camera.getWorldPosition(new THREE.Vector3()), direction);

            const intersects = raycaster.intersectObjects(zombies, true);
            if (intersects.length > 0) {
                const hitPart = intersects[0].object;
                if (hitPart.userData.isZombiePart) {
                    damageZombie(hitPart.userData.parentZombie, currentWeapon.damage);
                    createBlood(intersects[0].point);
                }
            }

            if (currentWeapon.clip === 0) reload();
        }

        function reload() {
            if (player.isReloading || currentWeapon.clip === currentWeapon.maxClip) return;
            player.isReloading = true;
            ammoVal.innerText = "RELOADING...";
            ammoVal.style.color = "yellow";
            
            setTimeout(() => {
                currentWeapon.clip = currentWeapon.maxClip;
                player.isReloading = false;
                updateHUD();
                ammoVal.style.color = "white";
            }, currentWeapon.reloadTime);
        }

        function createBlood(position) {
            const geo = new THREE.BoxGeometry(0.3, 0.3, 0.3);
            const mat = new THREE.MeshBasicMaterial({ color: 0xaa0000 });
            const blood = new THREE.Mesh(geo, mat);
            blood.position.copy(position);
            scene.add(blood);
            setTimeout(() => scene.remove(blood), 150);
        }

        function spawnZombie(forcedBoss = false) {
            if (!isGameRunning || zombies.length >= 30) return;

            const isBoss = forcedBoss || (killsSinceLastBoss >= 5) || (Math.random() < 0.10);
            if (isBoss) killsSinceLastBoss = 0;

            const zombie = new THREE.Group();
            
            const skinColor = isBoss ? 0x990000 : 0x2e5328;
            const clothesColor = isBoss ? 0x440000 : 0x3d3a37;
            const emissiveColor = isBoss ? 0x660000 : 0x000000;

            const skinMat = new THREE.MeshLambertMaterial({ color: skinColor, emissive: emissiveColor });
            const clothesMat = new THREE.MeshLambertMaterial({ color: clothesColor });

            const head = new THREE.Mesh(new THREE.BoxGeometry(0.8, 0.8, 0.8), skinMat);
            head.position.y = 3.3;
            
            const torso = new THREE.Mesh(new THREE.BoxGeometry(1.2, 1.4, 0.6), clothesMat);
            torso.position.y = 2.2;
            
            const armGeo = new THREE.BoxGeometry(0.3, 0.3, 1.1);
            const armL = new THREE.Mesh(armGeo, skinMat);
            armL.position.set(-0.75, 2.5, -0.4); 
            const armR = new THREE.Mesh(armGeo, skinMat);
            armR.position.set(0.75, 2.5, -0.4);
            
            const legGeo = new THREE.BoxGeometry(0.4, 1.4, 0.4);
            const legL = new THREE.Mesh(legGeo, clothesMat);
            legL.position.set(-0.3, 0.7, 0);
            const legR = new THREE.Mesh(legGeo, clothesMat);
            legR.position.set(0.3, 0.7, 0);

            zombie.add(head, torso, armL, armR, legL, legR);

            if (isBoss) {
                zombie.scale.set(1.8, 1.8, 1.8);
            }
            
            zombie.children.forEach(child => {
                child.userData.isZombiePart = true;
                child.userData.parentZombie = zombie;
            });

            const angle = Math.random() * Math.PI * 2;
            const dist = 25 + Math.random() * 20;
            zombie.position.set(
                yawObject.position.x + Math.cos(angle) * dist,
                0,
                yawObject.position.z + Math.sin(angle) * dist
            );
            
            zombie.userData = { 
                isBoss: isBoss,
                hp: isBoss ? 500 : 100,
                maxHp: isBoss ? 500 : 100,
                speed: isBoss ? 5.2 : (4.0 + Math.random() * 2.0), 
                radius: isBoss ? 1.4 : 0.8, 
                height: isBoss ? 6.5 : 3.8,
                attackDamage: isBoss ? 40 : 25,
                windUpDuration: isBoss ? 1.0 : 1.5,
                attackState: 'idle',
                windUpTimer: 0,
                cooldownTimer: 0,
                armL: armL,
                armR: armR
            };
            
            scene.add(zombie);
            zombies.push(zombie);
            updateHUD();
        }

        function damageZombie(zombie, amt) {
            zombie.userData.hp -= amt;
            const isBoss = zombie.userData.isBoss;

            zombie.children.forEach(child => {
                if (child.material) child.material.emissive.setHex(0xff0000); 
            });
            setTimeout(() => {
                if (zombie && zombie.parent) {
                    zombie.children.forEach(child => { 
                        if (child.material) child.material.emissive.setHex(isBoss ? 0x660000 : 0x000000); 
                    });
                }
            }, 100);

            if (zombie.userData.hp <= 0) {
                scene.remove(zombie);
                zombies = zombies.filter(z => z !== zombie);
                
                score++;
                if (!isBoss) {
                    killsSinceLastBoss++;
                }
                
                if (isBoss) score += 3;

                scoreVal.innerText = score;
                updateHUD();
            }
        }

        function spawnPickup() {
            if (!isGameRunning || pickups.length >= 10) return;

            const weaponKeys = Object.keys(weapons);
            const type = weaponKeys[Math.floor(Math.random() * (weaponKeys.length - 1)) + 1]; 
            const wepData = weapons[type];

            const geo = new THREE.BoxGeometry(1.5, 1.5, 1.5);
            const mat = new THREE.MeshLambertMaterial({ color: 0x0088ff, emissive: 0x0033aa });
            const pickup = new THREE.Mesh(geo, mat);

            const x = (Math.random() - 0.5) * 70;
            const z = (Math.random() - 0.5) * 70;
            
            pickup.position.set(yawObject.position.x + x, 1.2, yawObject.position.z + z);
            pickup.userData = { type: type, wepData: wepData };

            scene.add(pickup);
            pickups.push(pickup);
        }

        function clearAllPickups() {
            pickups.forEach(p => scene.remove(p));
            pickups = [];
            if (nearestPickup) {
                nearestPickup = null;
                pickupPrompt.style.display = 'none';
            }
        }

        function tryPickupWeapon() {
            if (nearestPickup) {
                currentWeaponKey = nearestPickup.userData.type;
                currentWeapon = nearestPickup.userData.wepData;
                currentWeapon.clip = currentWeapon.maxClip;
                
                updateGunMesh();
                
                scene.remove(nearestPickup);
                pickups = pickups.filter(p => p !== nearestPickup);
                nearestPickup = null;
                
                pickupPrompt.style.display = 'none';
                updateHUD();
                
                uiLayer.style.backgroundColor = "rgba(0,255,0,0.15)";
                setTimeout(() => uiLayer.style.backgroundColor = "transparent", 120);
            }
        }

        function checkWallCollision(position, radius, height) {
            const entityBox = new THREE.Box3().setFromCenterAndSize(
                position,
                new THREE.Vector3(radius * 2, height, radius * 2)
            );
            
            for (let i = 0; i < walls.length; i++) {
                if (walls[i].boundingBox.intersectsBox(entityBox)) {
                    return true;
                }
            }
            return false;
        }

        function takeDamage(amt) {
            player.hp -= amt;
            updateHUD();
            
            damageFlash.style.opacity = 1;
            setTimeout(() => damageFlash.style.opacity = 0, 150);

            if (player.hp <= 0) {
                document.exitPointerLock();
                startBtn.innerText = "RESTART GAME";
                menuTitle.innerText = "YOU DIED";
                menuTitle.style.color = "#f00";
            }
        }

        function animate() {
            requestAnimationFrame(animate);
            
            const time = performance.now();
            const delta = Math.min((time - lastTime) / 1000, 0.1);
            lastTime = time;

            if (isGameRunning) {
                gameTime += delta;

                if (gameTime >= 180 && !clearedThreeMinCrates) {
                    clearedThreeMinCrates = true;
                    clearAllPickups();
                    for (let i = 0; i < 3; i++) {
                        spawnPickup();
                    }
                }

                if (isMouseDown && currentWeapon.auto) {
                    shoot();
                }

                // --- PLAYER MOVEMENT & SPEED CALCULATIONS ---
                // Calculate speed modifier based on current equipped weapon & active firing state
                let currentSpeedMult = currentWeapon.moveSpeedMult;
                if (isMouseDown && currentWeapon.clip > 0 && !player.isReloading) {
                    currentSpeedMult *= currentWeapon.firingSpeedMult; // Stacking debuff while shooting
                }
                const effectiveSpeed = player.baseSpeed * currentSpeedMult;

                player.velocity.x -= player.velocity.x * 10.0 * delta;
                player.velocity.z -= player.velocity.z * 10.0 * delta;

                player.direction.z = Number(keys.s) - Number(keys.w);
                player.direction.x = Number(keys.d) - Number(keys.a);
                player.direction.normalize();

                if (keys.w || keys.s) player.velocity.z += player.direction.z * (effectiveSpeed * 6.0) * delta;
                if (keys.a || keys.d) player.velocity.x += player.direction.x * (effectiveSpeed * 6.0) * delta;

                const moveDistX = player.velocity.x * delta;
                const moveDistZ = player.velocity.z * delta;
                
                const startX = yawObject.position.x;
                const startZ = yawObject.position.z;

                yawObject.translateX(moveDistX);
                if (checkWallCollision(yawObject.position, player.radius, player.height)) {
                    yawObject.position.x = startX;
                    yawObject.position.z = startZ;
                }

                const postX_X = yawObject.position.x;
                const postX_Z = yawObject.position.z;

                yawObject.translateZ(moveDistZ);
                if (checkWallCollision(yawObject.position, player.radius, player.height)) {
                    yawObject.position.x = postX_X;
                    yawObject.position.z = postX_Z;
                }

                // View Bobbing
                if (keys.w || keys.a || keys.s || keys.d) {
                    camera.position.y = Math.sin(time * 0.012) * 0.08;
                    gunMesh.position.y = -0.3 + Math.sin(time * 0.015) * 0.04;
                } else {
                    camera.position.y = 0;
                    gunMesh.position.y = -0.3;
                }

                // Recoil decay
                if (recoil > 0) recoil = Math.max(0, recoil - delta * 4.0);
                gunMesh.position.z = -0.7 + recoil;

                // --- ZOMBIE AI & ATTACK LOGIC ---
                for (let i = 0; i < zombies.length; i++) {
                    const z = zombies[i];
                    const zData = z.userData;
                    z.lookAt(yawObject.position.x, z.position.y, yawObject.position.z);

                    const dist = z.position.distanceTo(yawObject.position);
                    const attackTriggerDist = player.radius + zData.radius + 0.8;

                    if (zData.cooldownTimer > 0) {
                        zData.cooldownTimer -= delta;
                    }

                    if (zData.attackState === 'idle') {
                        if (dist <= attackTriggerDist && zData.cooldownTimer <= 0) {
                            zData.attackState = 'windup';
                            zData.windUpTimer = zData.windUpDuration;
                        } else if (dist > player.radius + zData.radius + 0.2) {
                            const dir = new THREE.Vector3().subVectors(yawObject.position, z.position);
                            dir.y = 0;
                            dir.normalize();

                            const stepX = dir.x * zData.speed * delta;
                            const stepZ = dir.z * zData.speed * delta;

                            z.position.x += stepX;
                            if (checkWallCollision(z.position, zData.radius, zData.height)) {
                                z.position.x -= stepX;
                            }

                            z.position.z += stepZ;
                            if (checkWallCollision(z.position, zData.radius, zData.height)) {
                                z.position.z -= stepZ;
                            }

                            zData.armL.rotation.x = Math.sin(time * 0.008 + i) * 0.35;
                            zData.armR.rotation.x = Math.sin(time * 0.008 + i + Math.PI) * 0.35;
                        }
                    } else if (zData.attackState === 'windup') {
                        zData.windUpTimer -= delta;

                        const progress = 1.0 - (zData.windUpTimer / zData.windUpDuration);
                        zData.armL.rotation.x = -Math.PI / 2 - (progress * 0.8);
                        zData.armR.rotation.x = -Math.PI / 2 - (progress * 0.8);

                        z.children.forEach(child => {
                            if (child.material) child.material.emissive.setHex(0xff0000);
                        });

                        if (zData.windUpTimer <= 0) {
                            if (dist <= attackTriggerDist + 1.5) {
                                takeDamage(zData.attackDamage);
                            }

                            zData.attackState = 'idle';
                            zData.cooldownTimer = 2.5;

                            zData.armL.rotation.x = 0;
                            zData.armR.rotation.x = 0;
                            z.children.forEach(child => {
                                if (child.material) child.material.emissive.setHex(zData.isBoss ? 0x660000 : 0x000000);
                            });
                        }
                    }

                    if (zData.attackState === 'idle' && !zData.isBoss && z.children[0].material.emissive.getHex() !== 0x0) {
                        z.children.forEach(child => {
                            if (child.material) child.material.emissive.setHex(0x000000);
                        });
                    }
                }

                // --- PICKUP LOGIC ---
                nearestPickup = null;
                pickupPrompt.style.display = 'none';
                for (let i = 0; i < pickups.length; i++) {
                    const p = pickups[i];
                    p.rotation.x += 1.2 * delta;
                    p.rotation.y += 2.0 * delta;
                    
                    if (p.position.distanceTo(yawObject.position) < 3.5) {
                        nearestPickup = p;
                        pickupPrompt.style.display = 'block';
                        pickupPrompt.innerHTML = `Press <span class="key">Q</span> for ${p.userData.wepData.name}`;
                    }
                }
            }

            renderer.render(scene, camera);
        }

        function updateHUD() {
            hpVal.innerText = player.hp;
            hpVal.style.color = player.hp <= 30 ? '#ff3333' : '#ffffff';
            ammoVal.innerText = `${currentWeapon.clip}/${currentWeapon.maxClip}`;
            wepName.innerText = currentWeapon.name;

            const activeBosses = zombies.filter(z => z.userData.isBoss).length;
            if (activeBosses > 0) {
                bossVal.innerText = `WARNING: ${activeBosses} ACTIVE!`;
                bossVal.style.color = '#ff3333';
            } else {
                const killsNeeded = Math.max(0, 5 - killsSinceLastBoss);
                bossVal.innerText = `${killsNeeded} Kills`;
                bossVal.style.color = '#ffcc00';
            }
        }

        function resetGame() {
            player.hp = player.maxHp;
            score = 0;
            killsSinceLastBoss = 0;
            gameTime = 0;
            clearedThreeMinCrates = false;
            scoreVal.innerText = score;
            currentWeaponKey = 'pistol';
            currentWeapon = weapons.pistol;
            currentWeapon.clip = currentWeapon.maxClip;
            updateGunMesh();
            
            zombies.forEach(z => scene.remove(z));
            zombies = [];
            clearAllPickups();

            yawObject.position.set(0, player.height, 0);
            menuTitle.innerText = "DOOM SURVIVAL";
            menuTitle.style.color = "#f00";
            
            spawnPickup();
            updateHUD();
        }

        function onWindowResize() {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        }
    </script>
</body>
</html>
