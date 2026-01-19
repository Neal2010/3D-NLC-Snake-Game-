# 3D-NLC-Snake-Game-
Description This is a 3D Snake game made with Three.js. You control the snake in a 3D world, eat apples to grow longer, and collect stars for temporary immunity. When the snake hits a wall, it teleports to the opposite side of the map. The game runs полностью in the browser and works on both PC and mobile. 

<!DOCTYPE html>
<html lang="nl">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width,initial-scale=1.0">
  <title>3D Snake – Gemaakt door Neal Cordonnier</title>
  <style>
    /* ===================== STYLING ===================== */
    *{margin:0;padding:0;box-sizing:border-box;}
    body{background:#0a0a1a;overflow:hidden;color:#eee;font-family:Arial;}
    #container{width:100vw;height:100vh;position:relative;}
    #info{position:absolute;top:15px;left:15px;z-index:10;background:rgba(0,0,0,0.8);padding:15px;border-radius:12px;border:2px solid #4ade80;}
    .screen{position:absolute;top:50%;left:50%;transform:translate(-50%,-50%);background:rgba(0,0,0,0.95);padding:40px;border-radius:16px;text-align:center;z-index:20;width:90%;max-width:420px;}
    #start-screen{border:3px solid #4ade80;}
    #game-over{border:3px solid #ef4444;display:none;}
    #pause-screen{border:3px solid #60a5fa;display:none;}
    button{padding:15px 32px;margin-top:20px;background:#4ade80;color:#000;border:none;border-radius:12px;font-size:1.2em;cursor:pointer;}
    button:hover{background:#22c55e;}
    h1{color:#4ade80;text-shadow:0 0 15px #4ade80;}
    #game-over h1{color:#ef4444;text-shadow:0 0 15px #ef4444;}
    #pause-screen h1{color:#60a5fa;}
  </style>
</head>
<body>
  <!-- ===================== INFO ===================== -->
  <div id="container"></div>
  <div id="info">
    <h2>3D NLC Snake Game🐍</h2>
    Score: <span id="score">0</span> | Highscore: <span id="highscore">0</span><br>
    Immuniteit: <span id="immune-display">0</span>s
  </div>

  <!-- ===================== SCREENS ===================== -->
  <div id="start-screen" class="screen">
    <h1>3D SNAKE</h1>
    <p>Z ↑  Q ←  S ↓  D →  |  P = Pauze| Ster=Immuniteit </p>
    <button id="start-btn">START GAME</button>
  </div>
  <div id="game-over" class="screen">
    <h1>GAME OVER!</h1>
    <p>Score: <span id="final-score">0</span></p>
    <button id="restart-btn">Opnieuw</button>
  </div>
  <div id="pause-screen" class="screen">
    <h1>PAUZE</h1>
    <p>Druk op P om verder te gaan</p>
  </div>

  <!-- ===================== THREE.JS ===================== -->
  <script src="https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.min.js"></script>
  <script>
    // ===================== INITIAL SETUP =====================
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x0a0a1a);
    const camera = new THREE.PerspectiveCamera(75, innerWidth/innerHeight, 0.1, 1000);
    const renderer = new THREE.WebGLRenderer({antialias:true});
    renderer.setSize(innerWidth, innerHeight);
    document.getElementById('container').appendChild(renderer.domElement);

    scene.add(new THREE.AmbientLight(0xffffff, 0.8));
    scene.add(new THREE.DirectionalLight(0xffffff, 1).position.set(15,25,15));

    // ===================== GRID & WALL =====================
    const cellSize = 3;
    const gridSize = 51;
    const gridCells = Math.round(gridSize / cellSize);
    const maxIndex = (gridCells - 1) / 2;
    const wallLimit = maxIndex * cellSize;

    scene.add(new THREE.Mesh(new THREE.PlaneGeometry(51,51), new THREE.MeshBasicMaterial({color:0x16213e})).rotateX(-Math.PI/2));
    scene.add(new THREE.GridHelper(51, 51/3, 0x4ade80, 0x334155));

    // ===================== VARIABLES =====================
    let snake = [], direction = new THREE.Vector3(), nextDirection = new THREE.Vector3();
    let food, powerUp;
    let score = 0, highscore = localStorage.getItem('snakeHS')||0;
    let running = false, paused = false, moveTimer = 0, speed = 0.22;
    let immune = false, immuneTime = 0, targetColor = 0x4ade80;
    let canPassThroughSelf = false; // <-- Met ster mag je door jezelf
    const trails = [];

    const camPos = new THREE.Vector3();
    const camLook = new THREE.Vector3();
    document.getElementById('highscore').textContent = highscore;

    // ===================== MESH FUNCTIONS =====================
    function seg(pos){
      const m = new THREE.Mesh(new THREE.BoxGeometry(2.5,2.5,2.5), new THREE.MeshStandardMaterial({color:targetColor,emissive:targetColor,emissiveIntensity:0.8}));
      m.position.copy(pos); scene.add(m); return m;
    }
    function apple(pos){
      const m = new THREE.Mesh(new THREE.SphereGeometry(1.1,20,20), new THREE.MeshStandardMaterial({color:0xef4444,emissive:0xef4444,emissiveIntensity:0.8}));
      m.position.copy(pos); scene.add(m); return m;
    }
    function star(pos){
      const m = new THREE.Mesh(new THREE.OctahedronGeometry(1.4), new THREE.MeshStandardMaterial({color:0xfde047,emissive:0xfde047,emissiveIntensity:1.5}));
      m.position.copy(pos); scene.add(m); return m;
    }
    function trail(pos){
      const t = new THREE.Mesh(new THREE.PlaneGeometry(2.9,2.9), new THREE.MeshBasicMaterial({color:0x00ffcc,transparent:true,opacity:0.7}));
      t.rotation.x = -Math.PI/2; t.position.set(pos.x,0.02,pos.z); scene.add(t);
      trails.push({m:t,l:10});
    }
    function randPos(){
      return new THREE.Vector3(Math.floor(Math.random()*gridCells - maxIndex)*cellSize,1.5,Math.floor(Math.random()*gridCells - maxIndex)*cellSize);
    }

    // ===================== START GAME =====================
    function startGame(){
      snake.forEach(s=>scene.remove(s));
      trails.forEach(t=>scene.remove(t.m)); trails.length=0;
      if(food)scene.remove(food);
      if(powerUp)scene.remove(powerUp);
      snake=[]; score=0; immune=false; immuneTime=0; targetColor=0x4ade80;
      canPassThroughSelf = false;
      document.getElementById('score').textContent=score;
      document.getElementById('immune-display').textContent="0";
      document.getElementById('start-screen').style.display='none';
      document.getElementById('game-over').style.display='none';
      document.getElementById('pause-screen').style.display='none';

      const p = new THREE.Vector3(0,1.5,0);
      snake.push(seg(p), seg(new THREE.Vector3(-cellSize,1.5,0)), seg(new THREE.Vector3(-2*cellSize,1.5,0)));
      direction.set(cellSize,0,0);
      nextDirection.copy(direction);

      let pos; do{pos=randPos();}while(snake.some(s=>s.position.distanceTo(pos)<4));
      food = apple(pos);

      running = true; paused = false; moveTimer = 0;
      camPos.set(0,18,28);
      camLook.set(0,2,0);
      camera.position.copy(camPos);
      camera.lookAt(camLook);
    }

    // ===================== GAME OVER =====================
    function gameOver(){
      running = false;
      document.getElementById('final-score').textContent = score;
      if(score > highscore){
        highscore = score;
        localStorage.setItem('snakeHS', highscore);
        document.getElementById('highscore').textContent = highscore;
      }
      document.getElementById('game-over').style.display = 'block';
    }

    document.getElementById('start-btn').onclick = startGame;
    document.getElementById('restart-btn').onclick = startGame;

    // ===================== INPUT =====================
    document.addEventListener('keydown', e=>{
      if(e.key.toLowerCase()==='p'){ if(running){paused=!paused; document.getElementById('pause-screen').style.display=paused?'block':'none';} return; }
      if(!running || paused) return;
      if(e.key==='z') nextDirection.set(0,0,cellSize);
      if(e.key==='s') nextDirection.set(0,0,-cellSize);
      if(e.key==='q') nextDirection.set(cellSize,0,0);
      if(e.key==='d') nextDirection.set(-cellSize,0,0);
    });

    // ===================== MAIN LOOP =====================
    const clock = new THREE.Clock();
    function loop(){
      requestAnimationFrame(loop);
      const delta = clock.getDelta();

      if(running && !paused){
        if(food){ food.rotation.y += delta*3; food.scale.setScalar(1 + Math.sin(Date.now()*0.008)*0.1); }
        if(powerUp) powerUp.rotation.y += delta*4;

        moveTimer += delta;
        if(moveTimer >= speed){
          moveTimer -= speed;
          direction.copy(nextDirection);

          let headPos = snake[0].position.clone().add(direction);

          // ===================== WALL COLLISION =====================
          if(immune){ 
            if (headPos.x > wallLimit) headPos.x = -wallLimit;
            if (headPos.x < -wallLimit) headPos.x = wallLimit;
            if (headPos.z > wallLimit) headPos.z = -wallLimit;
            if (headPos.z < -wallLimit) headPos.z = wallLimit;
          } 
          else if(Math.abs(headPos.x) > wallLimit || Math.abs(headPos.z) > wallLimit){ 
            gameOver();
            return;
          }

          // ===================== SELF COLLISION =====================
          if(!canPassThroughSelf){ 
            if(snake.slice(1).some(s=>s.position.distanceTo(headPos)<2.5)){
              gameOver();
              return;
            }
          }

          // ===================== ADD NEW HEAD =====================
          const newHead = seg(headPos);
          snake.unshift(newHead);
          trail(headPos);

          // ===================== EAT FOOD =====================
          let ate = false;
          if(food && headPos.distanceTo(food.position)<2.5){
            score += 1;
            document.getElementById('score').textContent = score;
            scene.remove(food);
            let pos; do{pos=randPos();}while(snake.some(s=>s.position.distanceTo(pos)<4));
            food = apple(pos);
            targetColor = Math.random()*0xffffff;
            ate = true;
          }

          // ===================== EAT POWERUP =====================
          if(powerUp && headPos.distanceTo(powerUp.position)<2.5){
            immune = true;
            immuneTime = 1.5; 
            canPassThroughSelf = true; 
            scene.remove(powerUp);
            powerUp = null;
          }

          if(!ate) scene.remove(snake.pop());

          // ===================== COLORS =====================
          if(immune){
            immuneTime -= delta;
            document.getElementById('immune-display').textContent = Math.max(0,immuneTime).toFixed(1);
            if(immuneTime <= 0){
              immune = false;
              canPassThroughSelf = false; 
            }
            const t = performance.now()*0.001;
            snake.forEach((s,i)=>{
              const col = new THREE.Color().setHSL((t + i*0.05)%1, 0.9, 0.6);
              s.material.color.copy(col);
              s.material.emissive.copy(col);
            });
          }else{
            snake.forEach(s=>{
              s.material.color.lerp(new THREE.Color(targetColor), 0.15);
              s.material.emissive.lerp(new THREE.Color(targetColor), 0.15);
            });
          }

          trails.forEach((t,i)=>{ t.l--; t.m.material.opacity = t.l/50; if(t.l<=0){scene.remove(t.m); trails.splice(i,1);} });
        }

        // ===================== CAMERA =====================
        if(snake[0]){
          const desiredCamPos = new THREE.Vector3(snake[0].position.x, 18, snake[0].position.z - 20);
          const desiredLook = new THREE.Vector3(snake[0].position.x, 2, snake[0].position.z);

          camPos.lerp(desiredCamPos, 0.08);
          camLook.lerp(desiredLook, 0.08);

          camera.position.copy(camPos);
          camera.lookAt(camLook);
        }
      }
      renderer.render(scene,camera);
    }

    // ===================== POWERUP SPAWN =====================
    setInterval(()=>{
      if(running && !immune && !powerUp){
        let p; do{p=randPos();}while(snake.some(s=>s.position.distanceTo(p)<5));
        powerUp = star(p);
      }
    }, 50000);

    window.onresize = ()=>{ camera.aspect = innerWidth/innerHeight; camera.updateProjectionMatrix(); renderer.setSize(innerWidth,innerHeight); };
    camera.position.set(0,18,28);
    loop();
  </script>

  <!-- ===================== MOBIELE KNOPPEN ===================== -->
  <div id="mobile-controls">
    <div class="pad">
      <button class="dir" ontouchstart="setDir(0,0,3)">↑</button>
      <button class="dir" ontouchstart="setDir(3,0,0)">←</button>
      <button class="dir" ontouchstart="setDir(0,0,-3)">↓</button>
      <button class="dir" ontouchstart="setDir(-3,0,0)">→</button>
    </div>
    <button id="mobile-pause" ontouchstart="mobilePause()">Ⅱ</button>
  </div>

  <style>
    #mobile-controls {
      display: block;
      position: fixed;
      bottom: 20px;
      left: 50%;
      transform: translateX(-50%);
      width: 230px;
      height: 230px;
      z-index: 9999;
      pointer-events: auto;
    }
    .pad { position:absolute; bottom:0; left:0; width:100%; height:100%; pointer-events:auto; }
    .dir {
      position:absolute; width:64px; height:64px; background:rgba(74,222,128,0.96);
      border:4px solid #4ade80; border-radius:20px; color:#000; font-size:42px; font-weight:bold;
      display:flex; align-items:center; justify-content:center; box-shadow:0 6px 18px rgba(0,0,0,0.7);
    }
    .dir:nth-child(1) { top:8px; left:50%; transform:translateX(-50%); }
    .dir:nth-child(2) { top:50%; left:8px; transform:translateY(-50%); }
    .dir:nth-child(3) { top:50%; left:50%; transform:translate(-50%,-50%); }
    .dir:nth-child(4) { top:50%; right:8px; transform:translateY(-50%); }
    #mobile-pause {
      position:fixed; top:150px; right:150px; width:58px; height:58px; background:rgba(96,165,250,0.96);
      border:4px solid #60a5fa; border-radius:50%; color:white; font-size:32px; font-weight:bold;
      display:flex; align-items:center; justify-content:center; pointer-events:auto; box-shadow:0 6px 18px rgba(0,0,0,0.7);
    }
  </style>

  <script>
    // ===================== MOBIELE KNOPPEN LOGICA =====================
    if (!('ontouchstart' in window) || navigator.maxTouchPoints === 0 || screen.width > 1024) {
      document.getElementById('mobile-controls').style.display = 'none';
    }

    window.setDir = (x, y, z) => { if (running && !paused) nextDirection.set(x,y,z); };
    window.mobilePause = () => { if(running){ paused=!paused; document.getElementById('pause-screen').style.display = paused?'block':'none'; }};
  </script>

</body>
</html>





