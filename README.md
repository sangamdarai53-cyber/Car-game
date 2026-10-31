
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>3D Car Racing Game</title>
  <style>
    * { margin: 0; box-sizing: border-box; }
    body {
      overflow: hidden;
      background: #080d20;
      font-family: Arial, sans-serif;
      color: white;
    }
    canvas { display: block; }
    #hud {
      position: fixed;
      top: 20px;
      left: 20px;
      z-index: 2;
      line-height: 2;
      text-shadow: 0 2px 8px black;
    }
    #menu {
      position: fixed;
      inset: 0;
      z-index: 3;
      display: grid;
      place-content: center;
      text-align: center;
      background: #071020cc;
    }
    h1 { color: #38eaff; margin-bottom: 15px; }
    p { margin-bottom: 20px; }
    button {
      padding: 13px 25px;
      background: #087fa8;
      border: 1px solid #38eaff;
      color: white;
      border-radius: 8px;
      font-weight: bold;
      cursor: pointer;
    }
    #controls {
      position: fixed;
      bottom: 20px;
      left: 0;
      right: 0;
      display: flex;
      justify-content: space-between;
      padding: 0 20px;
      z-index: 2;
      pointer-events: none;
    }
    .group { display: flex; gap: 10px; }
    .group button { pointer-events: auto; width: 65px; }
    .hidden { display: none !important; }
  </style>
</head>
<body>
  <div id="hud">
    <div>🏎️ Speed: <span id="speed">0</span> km/h</div>
    <div>🏆 Score: <span id="score">0</span></div>
    <div>💰 Coins: <span id="coins">0</span></div>
    <div>⚡ Nitro: <span id="nitro">100</span>%</div>
  </div>

  <div id="menu">
    <h1 id="title">NEON RACER 3D</h1>
    <p>Dodge traffic. Collect coins. Race!</p>
    <button id="start">START RACING</button>
  </div>

  <div id="controls">
    <div class="group">
      <button data-key="ArrowLeft">◀</button>
      <button data-key="ArrowRight">▶</button>
    </div>
    <div class="group">
      <button data-key="Space">NITRO</button>
      <button data-key="ArrowDown">BRAKE</button>
    </div>
  </div>

  <script type="module">
    import * as THREE from
      "https://cdn.jsdelivr.net/npm/three@0.185.0/build/three.module.js";

    // -----------------------------
    // 1. BASIC GAME SETTINGS
    // -----------------------------

    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x080d20);
    scene.fog = new THREE.Fog(0x080d20, 35, 150);

    const camera = new THREE.PerspectiveCamera(
      65, innerWidth / innerHeight, 0.1, 250
    );
    camera.position.set(0, 6, 13);

    const renderer = new THREE.WebGLRenderer({
      antialias: true
    });
    renderer.setSize(innerWidth, innerHeight);
    renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
    renderer.shadowMap.enabled = true;
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    document.body.appendChild(renderer.domElement);

    scene.add(new THREE.HemisphereLight(0xaacbff, 0x222233, 2));

    const sunlight = new THREE.DirectionalLight(0xffffff, 2.5);
    sunlight.position.set(-8, 15, 8);
    sunlight.castShadow = true;
    scene.add(sunlight);

    const clock = new THREE.Clock();

    // -----------------------------
    // 2. GAME STATE
    // -----------------------------

    const game = {
      running: false,
      speed: 0,
      maxSpeed: 80,
      score: 0,
      coins: 0,
      nitro: 100,
      health: 100,
      playerX: 0,
      targetX: 0,
      elapsed: 0,
      spawnTimer: 0,
      coinTimer: 0
    };

    const keys = new Set();
    const traffic = [];
    const coins = [];
    const roadMarkers = [];
    const scenery = [];

    const ROAD_WIDTH = 12;
    const PLAYER_Z = 5;

    // -----------------------------
    // 3. MATERIALS
    // -----------------------------

    function makeMaterial(color, options = {}) {
      return new THREE.MeshStandardMaterial({
        color,
        roughness: 0.45,
        metalness: 0.2,
        ...options
      });
    }

    const roadMaterial = makeMaterial(0x252b3b);
    const lineMaterial = makeMaterial(0xffffff, {
      emissive: 0x333333
    });
    const grassMaterial = makeMaterial(0x111d2b);
    const wheelMaterial = makeMaterial(0x080a10);
    const glassMaterial = makeMaterial(0x102d48, {
      metalness: 0.7,
      roughness: 0.15
    });

    // -----------------------------
    // 4. CREATE 3D OBJECTS
    // -----------------------------

    function createBox(
      parent, width, height, depth,
      x, y, z, material
    ) {
      const mesh = new THREE.Mesh(
        new THREE.BoxGeometry(width, height, depth),
        material
      );
      mesh.position.set(x, y, z);
      mesh.castShadow = true;
      mesh.receiveShadow = true;
      parent.add(mesh);
      return mesh;
    }

    function createCar(color = 0x19dfff) {
      const car = new THREE.Group();
      const bodyMaterial = makeMaterial(color, {
        metalness: 0.65,
        roughness: 0.25
      });

      createBox(
        car, 1.7, 0.5, 3.1,
        0, 0.65, 0, bodyMaterial
      );

      createBox(
        car, 1.2, 0.5, 1.5,
        0, 1.1, -0.2, glassMaterial
      );

      // Front headlights and rear lights
      for (const side of [-1, 1]) {
        createBox(
          car, 0.3, 0.15, 0.1,
          side * 0.53, 0.72, -1.57,
          makeMaterial(0xa8f5ff, {
            emissive: 0x38ddff,
            emissiveIntensity: 2
          })
        );

        createBox(
          car, 0.3, 0.15, 0.1,
          side * 0.53, 0.72, 1.57,
          makeMaterial(0xff204f, {
            emissive: 0xff0033,
            emissiveIntensity: 2
          })
        );

        for (const z of [-1.05, 1.05]) {
          const wheel = new THREE.Mesh(
            new THREE.CylinderGeometry(
              0.34, 0.34, 0.22, 16
            ),
            wheelMaterial
          );
          wheel.rotation.z = Math.PI / 2;
          wheel.position.set(side * 0.85, 0.36, z);
          car.add(wheel);
        }
      }

      car.userData.wheels = car.children.filter(
        obj => obj.geometry?.type === "CylinderGeometry"
      );

      return car;
    }

    // -----------------------------
    // 5. BUILD THE WORLD
    // -----------------------------

    const ground = new THREE.Mesh(
      new THREE.PlaneGeometry(400, 400),
      grassMaterial
    );
    ground.rotation.x = -Math.PI / 2;
    ground.position.set(0, -0.15, -80);
    scene.add(ground);

    const road = new THREE.Mesh(
      new THREE.PlaneGeometry(ROAD_WIDTH, 300),
      roadMaterial
    );
    road.rotation.x = -Math.PI / 2;
    road.position.set(0, 0, -100);
    scene.add(road);

    // Road edges
    for (const x of [-5.8, 5.8]) {
      const edge = new THREE.Mesh(
        new THREE.BoxGeometry(0.12, 0.04, 300),
        makeMaterial(0x24dfff, {
          emissive: 0x0088ff
        })
      );
      edge.position.set(x, 0.03, -100);
      scene.add(edge);
    }

    // Moving dashed lane markings
    for (let z = -240; z < 20; z += 9) {
      for (const x of [-2, 2]) {
        const marker = new THREE.Mesh(
          new THREE.BoxGeometry(0.1, 0.04, 4),
          lineMaterial
        );
        marker.position.set(x, 0.04, z);
        scene.add(marker);
        roadMarkers.push(marker);
      }
    }

    // Street lights and simple city buildings
    for (const side of [-1, 1]) {
      for (let z = -220; z < 20; z += 16) {
        const pole = createBox(
          scene, 0.12, 4, 0.12,
          side * 8, 2, z,
          makeMaterial(0x52627e)
        );

        const lamp = createBox(
          scene, 0.65, 0.12, 0.2,
          side * 7.7, 4, z,
          makeMaterial(0x35eaff, {
            emissive: 0x00aaff,
            emissiveIntensity: 2
          })
        );
        scenery.push(pole, lamp);
      }

      for (let i = 0; i < 22; i++) {
        const height = 5 + Math.random() * 18;
        const width = 3 + Math.random() * 4;
        const z = -220 + i * 10;

        const building = createBox(
          scene, width, height, 5,
          side * (12 + Math.random() * 12),
          height / 2, z,
          makeMaterial(
            [0x121f3b, 0x1c2140, 0x172d40][
              Math.floor(Math.random() * 3)
            ]
          )
        );
        scenery.push(building);
      }
    }

    // -----------------------------
    // 6. PLAYER CAR
    // -----------------------------

    const player = createCar(0x19dfff);
    player.position.set(0, 0, PLAYER_Z);
    scene.add(player);

    // -----------------------------
    // 7. TRAFFIC GENERATION
    // -----------------------------

    function spawnTraffic() {
      if (traffic.length >= 10) return;

      const lanes = [-4, 0, 4];
      const lane = lanes[
        Math.floor(Math.random() * lanes.length)
      ];

      const car = createCar(
        [0xff345e, 0xffb632, 0x8e64ff, 0x52ff9a][
          Math.floor(Math.random() * 4)
        ]
      );

      car.scale.setScalar(0.9);
      car.position.set(lane, 0, -90 - Math.random() * 35);
      car.userData.speed = 0.5 + Math.random() * 0.5;

      scene.add(car);
      traffic.push(car);
    }

    // -----------------------------
    // 8. COIN GENERATION
    // -----------------------------

    function spawnCoin() {
      if (coins.length >= 15) return;

      const coin = new THREE.Group();

      const gold = makeMaterial(0xffd447, {
        metalness: 0.7,
        emissive: 0xff9d00,
        emissiveIntensity: 1
      });

      const ring = new THREE.Mesh(
        new THREE.TorusGeometry(0.42, 0.12, 10, 20),
        gold
      );
      ring.rotation.y = Math.PI / 2;
      coin.add(ring);

      const lane = [-4, 0, 4][
        Math.floor(Math.random() * 3)
      ];

      coin.position.set(lane, 1.2, -80);
      scene.add(coin);
      coins.push(coin);
    }

    // -----------------------------
    // 9. INPUT CONTROLS
    // -----------------------------

    window.addEventListener("keydown", event => {
      keys.add(event.code);
      if (["ArrowLeft", "ArrowRight", "ArrowDown", "Space"]
          .includes(event.code)) {
        event.preventDefault();
      }
    });

    window.addEventListener("keyup", event => {
      keys.delete(event.code);
    });

    document.querySelectorAll("[data-key]").forEach(button => {
      const code = button.dataset.key;

      button.addEventListener("pointerdown", event => {
        event.preventDefault();
        keys.add(code);
        button.setPointerCapture(event.pointerId);
      });

      const release = () => keys.delete(code);
      button.addEventListener("pointerup", release);
      button.addEventListener("pointercancel", release);
      button.addEventListener("lostpointercapture", release);
    });

    // -----------------------------
    // 10. START AND RESET
    // -----------------------------

    function resetGame() {
      game.speed = 0;
      game.score = 0;
      game.coins = 0;
      game.nitro = 100;
      game.health = 100;
      game.playerX = 0;
      game.targetX = 0;
      game.elapsed = 0;
      game.spawnTimer = 0;
      game.coinTimer = 0;

      traffic.forEach(car => scene.remove(car));
      coins.forEach(coin => scene.remove(coin));
      traffic.length = 0;
      coins.length = 0;

      player.position.set(0, 0, PLAYER_Z);
    }

    document.getElementById("start").onclick = () => {
      resetGame();
      game.running = true;
      document.getElementById("menu").classList.add("hidden");
    };

    function gameOver() {
      game.running = false;
      document.getElementById("title").textContent = "GAME OVER";
      document.querySelector("#menu p").textContent =
        `Score: ${Math.floor(game.score)} | Coins: ${game.coins}`;
      document.getElementById("start").textContent = "RACE AGAIN";
      document.getElementById("menu").classList.remove("hidden");
    }

    // -----------------------------
    // 11. UPDATE PLAYER
    // -----------------------------

    function updatePlayer(dt) {
      const left =
        keys.has("ArrowLeft") || keys.has("KeyA");
      const right =
        keys.has("ArrowRight") || keys.has("KeyD");
      const brake =
        keys.has("ArrowDown") || keys.has("KeyS");
      const nitro =
        (keys.has("Space") || keys.has("ShiftLeft")) &&
        game.nitro > 0;

      const maxSpeed = game.maxSpeed * (nitro ? 1.45 : 1);

      if (brake) {
        game.speed = Math.max(0, game.speed - 45 * dt);
      } else {
        game.speed = Math.min(
          maxSpeed, game.speed + 25 * dt
        );
      }

      if (nitro) {
        game.nitro = Math.max(0, game.nitro - 25 * dt);
      } else {
        game.nitro = Math.min(100, game.nitro + 7 * dt);
      }

      if (left) game.targetX -= 7 * dt;
      if (right) game.targetX += 7 * dt;

      game.targetX = THREE.MathUtils.clamp(
        game.targetX, -4.5, 4.5
      );

      game.playerX = THREE.MathUtils.damp(
        game.playerX, game.targetX, 8, dt
      );

      player.position.x = game.playerX;
      player.rotation.z = right ? -0.09 : left ? 0.09 : 0;

      player.userData.wheels.forEach(wheel => {
        wheel.rotation.x -= game.speed * dt * 0.08;
      });
    }

    // -----------------------------
    // 12. MOVE ROAD AND SCENERY
    // -----------------------------

    function updateWorld(dt) {
      const movement = game.speed * dt * 0.65;

      roadMarkers.forEach(marker => {
        marker.position.z += movement;
        if (marker.position.z > 15) {
          marker.position.z -= 260;
        }
      });

      scenery.forEach(object => {
        object.position.z += movement;
        if (object.position.z > 25) {
          object.position.z -= 250;
        }
      });

      game.score += movement * 3;
    }

    // -----------------------------
    // 13. UPDATE TRAFFIC
    // -----------------------------

    function updateTraffic(dt) {
      for (let i = traffic.length - 1; i >= 0; i--) {
        const car = traffic[i];

        car.position.z +=
          game.speed * dt * (0.65 + car.userData.speed * 0.2);

        const dx = Math.abs(car.position.x - player.position.x);
        const dz = Math.abs(car.position.z - player.position.z);

        if (dx < 1.45 && dz < 2.2) {
          game.health -= 25;
          game.speed *= 0.35;
          scene.remove(car);
          traffic.splice(i, 1);

          if (game.health <= 0) {
            gameOver();
            return;
          }
          continue;
        }

        if (car.position.z > 20) {
          game.score += 100;
          scene.remove(car);
          traffic.splice(i, 1);
        }
      }
    }

    // -----------------------------
    // 14. UPDATE COINS
    // -----------------------------

    function updateCoins(dt) {
      for (let i = coins.length - 1; i >= 0; i--) {
        const coin = coins[i];

        coin.position.z += game.speed * dt * 0.65;
        coin.rotation.y += dt * 2.5;
        coin.position.y = 1.2 + Math.sin(
          performance.now() * 0.003 + i
        ) * 0.15;

        const dx = Math.abs(coin.position.x - player.position.x);
        const dz = Math.abs(coin.position.z - player.position.z);

        if (dx < 1.3 && dz < 1.8) {
          game.coins++;
          game.score += 250;
          game.nitro = Math.min(100, game.nitro + 10);
          scene.remove(coin);
          coins.splice(i, 1);
        } else if (coin.position.z > 20) {
          scene.remove(coin);
          coins.splice(i, 1);
        }
      }
    }

    // -----------------------------
    // 15. SPAWN TIMERS
    // -----------------------------

    function updateSpawning(dt) {
      game.spawnTimer += dt;
      game.coinTimer += dt;

      const trafficInterval = Math.max(
        0.55, 1.5 - game.elapsed * 0.005
      );

      if (game.spawnTimer >= trafficInterval) {
        game.spawnTimer = 0;
        spawnTraffic();
      }

      if (game.coinTimer >= 1.6) {
        game.coinTimer = 0;
        spawnCoin();
      }
    }

    // -----------------------------
    // 16. CAMERA
    // -----------------------------

    function updateCamera(dt) {
      camera.position.x = THREE.MathUtils.damp(
        camera.position.x, game.playerX * 0.3, 3, dt
      );

      camera.lookAt(game.playerX * 0.2, 0.7, -12);
    }

    // -----------------------------
    // 17. HUD
    // -----------------------------

    function updateHUD() {
      document.getElementById("speed").textContent =
        Math.round(game.speed * 2.15);

      document.getElementById("score").textContent =
        Math.floor(game.score);

      document.getElementById("coins").textContent =
        game.coins;

      document.getElementById("nitro").textContent =
        Math.floor(game.nitro);
    }

    // -----------------------------
    // 18. MAIN GAME LOOP
    // -----------------------------

    function animate() {
      requestAnimationFrame(animate);

      const dt = Math.min(clock.getDelta(), 0.04);

      if (game.running) {
        game.elapsed += dt;

        updatePlayer(dt);
        updateWorld(dt);
        updateSpawning(dt);
        updateTraffic(dt);
        if (game.running) updateCoins(dt);
        updateCamera(dt);
        updateHUD();
      }

      renderer.render(scene, camera);
    }

    // -----------------------------
    // 19. WINDOW RESIZE
    // -----------------------------

    window.addEventListener("resize", () => {
      camera.aspect = innerWidth / innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(innerWidth, innerHeight);
      renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
    });

    animate();
  </script>
</body>
</html>
