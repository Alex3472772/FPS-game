# FPS-game
<!DOCTYPE html>
<html lang="fr">
<head>
<meta charset="UTF-8">
<title>VOID RUNNER — FPS 3D</title>
<style>
  html,body{margin:0;padding:0;overflow:hidden;background:#000;font-family:'Courier New',monospace;}
  #c{display:block;width:100vw;height:100vh;}

  #hud{position:fixed;inset:0;pointer-events:none;color:#9fffb0;}
  #crosshair{
    position:absolute;top:50%;left:50%;transform:translate(-50%,-50%);
    width:22px;height:22px;
  }
  #crosshair::before,#crosshair::after{
    content:"";position:absolute;background:#bfffcf;
  }
  #crosshair::before{left:50%;top:0;width:2px;height:100%;transform:translateX(-50%);}
  #crosshair::after{top:50%;left:0;height:2px;width:100%;transform:translateY(-50%);}

  #stats{
    position:absolute;left:18px;bottom:18px;font-size:16px;letter-spacing:1px;
    text-shadow:0 0 6px rgba(159,255,176,.7);
  }
  #stats div{margin:2px 0;}
  #health-bar{width:200px;height:10px;border:1px solid #9fffb0;margin-top:4px;}
  #health-fill{height:100%;background:#9fffb0;width:100%;transition:width .15s;}

  #score{
    position:absolute;right:18px;top:18px;font-size:18px;text-align:right;
    text-shadow:0 0 6px rgba(159,255,176,.7);
  }

  #weaponBar{
    position:absolute;bottom:18px;right:18px;display:flex;gap:8px;
  }
  .wslot{
    border:1px solid #2a5d3a;color:#5a9f70;padding:8px 12px;font-size:13px;
    letter-spacing:1px;background:rgba(10,20,14,.5);
  }
  .wslot.active{
    border-color:#9fffb0;color:#bfffcf;background:rgba(159,255,176,.12);
    text-shadow:0 0 6px rgba(159,255,176,.8);
  }

  #msg{
    position:absolute;top:40%;left:50%;transform:translate(-50%,-50%);
    text-align:center;color:#bfffcf;text-shadow:0 0 10px rgba(159,255,176,.9);
    width:90%;max-width:520px;
  }
  #title{font-size:42px;letter-spacing:6px;margin-bottom:10px;}
  #sub{font-size:14px;opacity:.85;margin-bottom:18px;line-height:1.7;}
  #startBtn{
    pointer-events:all;cursor:pointer;background:transparent;color:#9fffb0;
    border:1px solid #9fffb0;padding:12px 28px;font-family:inherit;font-size:16px;
    letter-spacing:3px;
  }
  #startBtn:hover{background:#9fffb0;color:#001a06;}

  #flash{
    position:fixed;inset:0;background:#ff2b2b;opacity:0;pointer-events:none;
    transition:opacity .1s;
  }
  #muzzleFlash{
    position:fixed;inset:0;pointer-events:none;
    background:radial-gradient(circle at 50% 78%, rgba(255,224,140,.85), rgba(255,180,60,.25) 35%, transparent 60%);
    opacity:0;
  }

  .hidden{display:none !important;}
</style>
</head>
<body>

<canvas id="c"></canvas>

<div id="hud">
  <div id="crosshair"></div>
  <div id="stats">
    <div>VIE</div>
    <div id="health-bar"><div id="health-fill"></div></div>
    <div id="ammo" style="margin-top:8px;">MUNITIONS: 12 / 12</div>
  </div>
  <div id="score">
    SCORE: 0<br>VAGUE: 1
    <div id="map-name" style="font-size:12px; opacity:0.7; margin-top:4px;">MAP: COMPLEXE VOID</div>
  </div>
  <div id="weaponBar">
    <div class="wslot active" id="slot1">1 · PISTOLET</div>
    <div class="wslot" id="slot2">2 · POMPE</div>
    <div class="wslot" id="slot3">3 · MITRAILLETTE</div>
    <div class="wslot" id="slot4">4 · SNIPER</div>
  </div>
</div>

<div id="flash"></div>
<div id="muzzleFlash"></div>

<div id="msg">
  <div id="title">VOID RUNNER</div>
  <div id="sub">
    ZQSD / WASD — se déplacer&nbsp;&nbsp;|&nbsp;&nbsp;SOURIS — viser<br>
    CLIC GAUCHE — tirer&nbsp;&nbsp;|&nbsp;&nbsp;CLIC DROIT — zoom (Sniper)<br>
    ESPACE — sauter&nbsp;&nbsp;|&nbsp;&nbsp;R — recharger&nbsp;&nbsp;|&nbsp;&nbsp;1/2/3/4 ou MOLETTE — armes<br><br>
    Éliminez les drones avant qu'ils ne vous atteignent.
  </div>
  <button id="startBtn">CLIQUER POUR JOUER</button>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script>
(function(){
  "use strict";

  // ================= AUDIO =================
  var AudioCtx = window.AudioContext || window.webkitAudioContext;
  var actx = null;
  function ensureAudio(){
    if(!actx) actx = new AudioCtx();
    if(actx.state === 'suspended') actx.resume();
  }

  function noiseBuffer(duration){
    var sr = actx.sampleRate;
    var len = Math.floor(sr*duration);
    var buf = actx.createBuffer(1, len, sr);
    var data = buf.getChannelData(0);
    for(var i=0;i<len;i++){ data[i] = (Math.random()*2-1) * Math.pow(1-i/len, 1.4); }
    return buf;
  }

  function playGunshot(opts){
    ensureAudio();
    opts = opts || {};
    var dur = opts.dur || 0.18;
    var freq = opts.freq || 140;
    var vol = opts.vol === undefined ? 0.5 : opts.vol;
    var now = actx.currentTime;

    var noise = actx.createBufferSource();
    noise.buffer = noiseBuffer(dur);
    var bp = actx.createBiquadFilter();
    bp.type = 'bandpass';
    bp.frequency.value = opts.filterFreq || 1800;
    bp.Q.value = 0.7;
    var noiseGain = actx.createGain();
    noiseGain.gain.setValueAtTime(vol, now);
    noiseGain.gain.exponentialRampToValueAtTime(0.001, now+dur);
    noise.connect(bp); bp.connect(noiseGain); noiseGain.connect(actx.destination);
    noise.start(now); noise.stop(now+dur);

    var osc = actx.createOscillator();
    osc.type = 'square';
    osc.frequency.setValueAtTime(freq, now);
    osc.frequency.exponentialRampToValueAtTime(freq*0.4, now+dur*0.6);
    var oscGain = actx.createGain();
    oscGain.gain.setValueAtTime(vol*0.7, now);
    oscGain.gain.exponentialRampToValueAtTime(0.001, now+dur*0.6);
    osc.connect(oscGain); oscGain.connect(actx.destination);
    osc.start(now); osc.stop(now+dur*0.6);
  }

  function playReload(){
    ensureAudio();
    var now = actx.currentTime;
    [0, 0.18].forEach(function(t){
      var osc = actx.createOscillator();
      osc.type = 'square';
      osc.frequency.setValueAtTime(420, now+t);
      osc.frequency.exponentialRampToValueAtTime(220, now+t+0.06);
      var g = actx.createGain();
      g.gain.setValueAtTime(0.18, now+t);
      g.gain.exponentialRampToValueAtTime(0.001, now+t+0.08);
      osc.connect(g); g.connect(actx.destination);
      osc.start(now+t); osc.stop(now+t+0.09);
    });
  }

  function playEmptyClick(){
    ensureAudio();
    var now = actx.currentTime;
    var osc = actx.createOscillator();
    osc.type = 'square';
    osc.frequency.setValueAtTime(900, now);
    var g = actx.createGain();
    g.gain.setValueAtTime(0.12, now);
    g.gain.exponentialRampToValueAtTime(0.001, now+0.05);
    osc.connect(g); g.connect(actx.destination);
    osc.start(now); osc.stop(now+0.05);
  }

  function playHitMarker(){
    ensureAudio();
    var now = actx.currentTime;
    var osc = actx.createOscillator();
    osc.type = 'triangle';
    osc.frequency.setValueAtTime(1200, now);
    var g = actx.createGain();
    g.gain.setValueAtTime(0.15, now);
    g.gain.exponentialRampToValueAtTime(0.001, now+0.08);
    osc.connect(g); g.connect(actx.destination);
    osc.start(now); osc.stop(now+0.08);
  }

  function playKill(){
    ensureAudio();
    var now = actx.currentTime;
    [0,0.07].forEach(function(t, idx){
      var osc = actx.createOscillator();
      osc.type = 'sawtooth';
      osc.frequency.setValueAtTime(idx===0?500:760, now+t);
      var g = actx.createGain();
      g.gain.setValueAtTime(0.12, now+t);
      g.gain.exponentialRampToValueAtTime(0.001, now+t+0.12);
      osc.connect(g); g.connect(actx.destination);
      osc.start(now+t); osc.stop(now+t+0.12);
    });
  }

  // ================= BASIC SETUP =================
  var canvas = document.getElementById('c');
  var renderer = new THREE.WebGLRenderer({canvas: canvas, antialias: true});
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.setSize(window.innerWidth, window.innerHeight);

  var scene = new THREE.Scene();
  scene.background = new THREE.Color(0x04060a);
  var fogColor = 0x04060a;
  scene.fog = new THREE.Fog(fogColor, 10, 90);

  var camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);
  var yawObject = new THREE.Object3D();
  var pitchObject = new THREE.Object3D();
  pitchObject.add(camera);
  yawObject.add(pitchObject);
  yawObject.position.set(0, 1.7, 10);
  scene.add(yawObject);

  // ---- gun model attached to camera ----
  var gunGroup = new THREE.Group();
  var gunMat = new THREE.MeshStandardMaterial({color:0x222831, roughness:0.5, metalness:0.4});
  var gunAccentMat = new THREE.MeshStandardMaterial({color:0x9fffb0, emissive:0x174a26, roughness:0.4});
  var gunBody = new THREE.Mesh(new THREE.BoxGeometry(0.12,0.12,0.55), gunMat);
  var gunBarrel = new THREE.Mesh(new THREE.CylinderGeometry(0.025,0.025,0.35,10), gunMat);
  gunBarrel.rotation.x = Math.PI/2;
  gunBarrel.position.set(0,0.02,-0.4);
  var gunGrip = new THREE.Mesh(new THREE.BoxGeometry(0.08,0.2,0.1), gunMat);
  gunGrip.position.set(0,-0.13,0.15);
  var gunStripe = new THREE.Mesh(new THREE.BoxGeometry(0.13,0.02,0.1), gunAccentMat);
  gunStripe.position.set(0,0.04,0.05);
  gunGroup.add(gunBody, gunBarrel, gunGrip, gunStripe);
  gunGroup.position.set(0.28, -0.28, -0.6);
  gunGroup.rotation.y = 0.05;
  camera.add(gunGroup);

  var muzzleLight = new THREE.PointLight(0xffd27a, 0, 5);
  muzzleLight.position.set(0, 0.02, -0.6);
  gunGroup.add(muzzleLight);

  var gunBasePos = gunGroup.position.clone();
  var gunRecoilOffset = 0;
  var bobTime = 0;

  // ================= LIGHTING =================
  var hemi = new THREE.HemisphereLight(0x8899ff, 0x080808, 0.7);
  scene.add(hemi);
  var dirLight = new THREE.DirectionalLight(0x9fffb0, 0.6);
  dirLight.position.set(20, 30, 10);
  scene.add(dirLight);

  // ================= ARENA & MAP DATA =================
  var ARENA_SIZE = 70;
  var half = ARENA_SIZE/2;
  var floorGeo = new THREE.PlaneGeometry(ARENA_SIZE, ARENA_SIZE, 1, 1);
  var floorMat = new THREE.MeshStandardMaterial({color: 0x10151c, roughness: 0.95});
  var floor = new THREE.Mesh(floorGeo, floorMat);
  floor.rotation.x = -Math.PI/2;
  scene.add(floor);

  var grid = null; 
  var wallMat = new THREE.MeshStandardMaterial({color: 0x12161d, roughness: 1});
  var wallHeight = 8;
  
  // Générer les murs extérieurs d'enceinte
  function makeWall(w,h,d,x,y,z){
    var g = new THREE.BoxGeometry(w,h,d);
    var m = new THREE.Mesh(g, wallMat);
    m.position.set(x,y,z);
    scene.add(m);
  }
  makeWall(ARENA_SIZE, wallHeight, 1, 0, wallHeight/2, -half);
  makeWall(ARENA_SIZE, wallHeight, 1, 0, wallHeight/2, half);
  makeWall(1, wallHeight, ARENA_SIZE, -half, wallHeight/2, 0);
  makeWall(1, wallHeight, ARENA_SIZE, half, wallHeight/2, 0);

  // Configuration des 3 maps différentes
  var MAPS = [
    {
      name: "COMPLEXE VOID",
      bgColor: 0x04060a,
      gridColor: 0x2a5d3a,
      wallColor: 0x12161d,
      obstacles: [
        [-15,-10,4,3,4],[15,-12,4,3,4],[-18,8,3,2.5,6],[18,10,3,2.5,6],
        [0,-20,6,2,3],[-8,15,3,3,3],[8,18,3,3,3],[0,5,2.5,2,8]
      ]
    },
    {
      name: "LABORATOIRE NÉON",
      bgColor: 0x0b0512,
      gridColor: 0x8f1d74,
      wallColor: 0x1f112b,
      obstacles: [
        [0,0,10,6,10], [-22,-18,6,4,6], [22,18,6,4,6], [-22,18,6,4,6], [22,-18,6,4,6],
        [-12,0,4,2,4], [12,0,4,2,4], [0,-22,5,3,5]
      ]
    },
    {
      name: "NID DE CRYPTE",
      bgColor: 0x020805,
      gridColor: 0x00ff55,
      wallColor: 0x09170e,
      obstacles: [
        [-20,0,3,5,24], [20,0,3,5,24], [0,-20,24,5,3], [0,20,24,5,3],
        [-10,-10,4,4,4], [10,10,4,4,4], [-10,10,4,4,4], [10,-10,4,4,4]
      ]
    }
  ];

  var obstacles = [];
  var obstacleMeshes = [];
  var obstacleMat = new THREE.MeshStandardMaterial({color: 0x1c2430, roughness: 0.8});

  function loadMap(idx) {
    // Nettoyer l'ancienne map
    obstacleMeshes.forEach(function(m) { scene.remove(m); });
    obstacleMeshes = [];
    obstacles = [];
    if(grid) scene.remove(grid);

    var mData = MAPS[idx];

    // Appliquer le style visuel de la map
    scene.background.setHex(mData.bgColor);
    scene.fog.color.setHex(mData.bgColor);
    wallMat.color.setHex(mData.wallColor);

    grid = new THREE.GridHelper(ARENA_SIZE, 28, mData.gridColor, 0x16241c);
    grid.position.y = 0.01;
    scene.add(grid);

    // Instancier les nouveaux obstacles
    mData.obstacles.forEach(function(p){
      var g = new THREE.BoxGeometry(p[2], p[3], p[4]);
      var m = new THREE.Mesh(g, obstacleMat);
      m.position.set(p[0], p[3]/2, p[1]);
      scene.add(m);
      obstacleMeshes.push(m);
      obstacles.push({mesh:m, x:p[0], z:p[1], hx:p[2]/2, hz:p[4]/2});
    });

    var mapNameEl = document.getElementById('map-name');
    if(mapNameEl) mapNameEl.textContent = "MAP: " + mData.name;
  }

  // Particules ambiantes
  var particleGeo = new THREE.BufferGeometry();
  var particleCount = 200;
  var positions = new Float32Array(particleCount*3);
  for(var i=0;i<particleCount;i++){
    positions[i*3] = (Math.random()-0.5)*ARENA_SIZE;
    positions[i*3+1] = Math.random()*10;
    positions[i*3+2] = (Math.random()-0.5)*ARENA_SIZE;
  }
  particleGeo.setAttribute('position', new THREE.BufferAttribute(positions,3));
  var particleMat = new THREE.PointsMaterial({color:0x9fffb0, size:0.05, transparent:true, opacity:0.5});
  var particles = new THREE.Points(particleGeo, particleMat);
  scene.add(particles);

  // ================= WEAPONS =================
  var WEAPONS = {
    pistol: {
      name: 'PISTOLET', clipMax: 12, fireDelay: 0.32, reloadTime: 0.85,
      damage: 34, pellets: 1, spread: 0.0, recoil: 0.012, kick: 0.05,
      gunshot: {dur:0.16, freq:160, filterFreq:2200, vol:0.45}
    },
    shotgun: {
      name: 'POMPE', clipMax: 6, fireDelay: 0.75, reloadTime: 1.3,
      damage: 22, pellets: 7, spread: 0.07, recoil: 0.05, kick: 0.16,
      gunshot: {dur:0.28, freq:90, filterFreq:1200, vol:0.65}
    },
    smg: {
      name: 'MITRAILLETTE', clipMax: 30, fireDelay: 0.09, reloadTime: 1.1,
      damage: 14, pellets: 1, spread: 0.025, recoil: 0.006, kick: 0.025,
      gunshot: {dur:0.09, freq:220, filterFreq:2600, vol:0.32}
    },
    sniper: {
      name: 'SNIPER', clipMax: 5, fireDelay: 1.2, reloadTime: 1.8,
      damage: 120, pellets: 1, spread: 0.0, recoil: 0.09, kick: 0.28,
      gunshot: {dur:0.45, freq:65, filterFreq:950, vol:0.8}
    }
  };
  var weaponOrder = ['pistol','shotgun','smg', 'sniper'];
  var currentWeaponKey = 'pistol';

  var player = {
    health: 100,
    speed: 9,
    velocityY: 0,
    onGround: true,
    radius: 0.5,
    reloading: false,
    lastShot: -999,
  };

  var ammoState = {
    pistol: WEAPONS.pistol.clipMax,
    shotgun: WEAPONS.shotgun.clipMax,
    smg: WEAPONS.smg.clipMax,
    sniper: WEAPONS.sniper.clipMax
  };

  var keys = {};
  var pointerLocked = false;
  var gameRunning = false;
  var gamePaused = false;
  var isZoomed = false; // Gestion de la visée sniper

  // ================= POINTER LOCK =================
  var startBtn = document.getElementById('startBtn');
  var msg = document.getElementById('msg');

  function requestLock(){
    ensureAudio();
    canvas.requestPointerLock = canvas.requestPointerLock || canvas.mozRequestPointerLock;
    canvas.requestPointerLock();
  }

  startBtn.addEventListener('click', function(){
    requestLock();
  });

  document.addEventListener('pointerlockchange', onLockChange);
  document.addEventListener('mozpointerlockchange', onLockChange);

  function onLockChange(){
    if(document.pointerLockElement === canvas || document.mozPointerLockElement === canvas){
      pointerLocked = true;
      msg.classList.add('hidden');
      if(!gameRunning){ startGame(); }
      gamePaused = false;
    } else {
      pointerLocked = false;
      isZoomed = false;
      if(gameRunning){
        gamePaused = true;
        showPause();
      }
    }
  }

  function showPause(){
    document.getElementById('title').textContent = 'PAUSE';
    document.getElementById('sub').innerHTML = 'Cliquez pour reprendre la partie.';
    startBtn.textContent = 'REPRENDRE';
    msg.classList.remove('hidden');
  }

  document.addEventListener('mousemove', function(e){
    if(!pointerLocked || gamePaused) return;
    var movementX = e.movementX || e.mozMovementX || 0;
    var movementY = e.movementY || e.mozMovementY || 0;

    // Réduire la sensibilité si le joueur zoome
    var sensitivity = 0.0022;
    if (isZoomed && currentWeaponKey === 'sniper') sensitivity = 0.0006;

    yawObject.rotation.y -= movementX * sensitivity;
    pitchObject.rotation.x -= movementY * sensitivity;
    pitchObject.rotation.x = Math.max(-Math.PI/2 + 0.05, Math.min(Math.PI/2 - 0.05, pitchObject.rotation.x));
  });

  // ================= INPUT =================
  var mouseDown = false;

  window.addEventListener('contextmenu', function(e){ e.preventDefault(); }); // Bloque le menu clic droit

  window.addEventListener('keydown', function(e){
    keys[e.code] = true;
    if(e.code === 'KeyR') reload();
    if(e.code === 'Digit1') switchWeapon('pistol');
    if(e.code === 'Digit2') switchWeapon('shotgun');
    if(e.code === 'Digit3') switchWeapon('smg');
    if(e.code === 'Digit4') switchWeapon('sniper');
  });
  window.addEventListener('keyup', function(e){
    keys[e.code] = false;
  });

  window.addEventListener('resize', function(){
    camera.aspect = window.innerWidth/window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  });

  canvas.addEventListener('wheel', function(e){
    if(!pointerLocked || !gameRunning) return;
    var idx = weaponOrder.indexOf(currentWeaponKey);
    if(e.deltaY > 0) idx = (idx+1) % weaponOrder.length;
    else idx = (idx-1+weaponOrder.length) % weaponOrder.length;
    switchWeapon(weaponOrder[idx]);
  });

  canvas.addEventListener('mousedown', function(e){
    if(!pointerLocked || gamePaused || !gameRunning) return;
    if(e.button === 0){ mouseDown = true; tryShoot(); }
    if(e.button === 2){ isZoomed = true; } // Zoom visée
  });
  canvas.addEventListener('mouseup', function(e){
    if(e.button === 0) mouseDown = false;
    if(e.button === 2) isZoomed = false; // Dézoome
  });

  // ================= WEAPON SWITCH UI & VISUAL MODEL =================
  var slotEls = {
    pistol: document.getElementById('slot1'), 
    shotgun: document.getElementById('slot2'), 
    smg: document.getElementById('slot3'),
    sniper: document.getElementById('slot4')
  };
  
  function switchWeapon(key){
    if(player.reloading) return;
    if(currentWeaponKey === key) return;
    currentWeaponKey = key;
    player.lastShot = -999;
    isZoomed = false;
    updateAmmoHud();
    Object.keys(slotEls).forEach(function(k){
      slotEls[k].classList.toggle('active', k === key);
    });
    
    // Altérer dynamiquement le modèle 3D géométrique de l'arme selon son type
    if(key === 'sniper') {
      gunBody.scale.set(1.0, 0.9, 1.7);
      gunBarrel.scale.set(0.6, 2.2, 0.6);
      gunStripe.material.color.setHex(0xffaa00); // Ligne Orange
    } else if (key === 'shotgun') {
      gunBody.scale.set(1.4, 1.1, 1.1);
      gunBarrel.scale.set(1.5, 1.0, 1.5);
      gunStripe.material.color.setHex(0xff3333); // Ligne Rouge
    } else if (key === 'smg') {
      gunBody.scale.set(0.8, 0.9, 0.9);
      gunBarrel.scale.set(0.8, 0.7, 0.8);
      gunStripe.material.color.setHex(0x33ccff); // Ligne Bleue
    } else {
      gunBody.scale.set(1, 1, 1);
      gunBarrel.scale.set(1, 1, 1);
      gunStripe.material.color.setHex(0x9fffb0); // Ligne Verte d'origine
    }

    gunGroup.position.y = gunBasePos.y - 0.15;
  }

  // ================= SHOOTING =================
  var raycaster = new THREE.Raycaster();
  var flashEl = document.getElementById('flash');
  var muzzleFlashEl = document.getElementById('muzzleFlash');
  var ammoEl = document.getElementById('ammo');
  var healthFill = document.getElementById('health-fill');
  var scoreEl = document.getElementById('score');

  var score = 0;
  var wave = 1;

  function tryShoot(){
    var w = WEAPONS[currentWeaponKey];
    var now = performance.now()/1000;
    if(now - player.lastShot < w.fireDelay) return;
    if(player.reloading){ return; }
    if(ammoState[currentWeaponKey] <= 0){
      playEmptyClick();
      player.lastShot = now;
      return;
    }
    player.lastShot = now;
    shoot(w);
  }

  function shoot(w){
    ammoState[currentWeaponKey]--;
    updateAmmoHud();
    playGunshot(w.gunshot);
    muzzleFlashEffect();
    gunKick(w.kick);
    cameraRecoil(w.recoil);

    var hitSomething = false;
    for(var p=0; p<w.pellets; p++){
      var spreadX = (Math.random()-0.5) * w.spread;
      var spreadY = (Math.random()-0.5) * w.spread;
      var ndc = new THREE.Vector2(spreadX, spreadY);
      raycaster.setFromCamera(ndc, camera);

      var meshes = enemies.map(function(en){ return en.mesh; });
      var intersects = raycaster.intersectObjects(meshes, true);
      drawTracer(raycaster.ray, intersects.length ? intersects[0].distance : 40);

      if(intersects.length > 0){
        var hitMesh = intersects[0].object;
        var enemy = enemies.find(function(en){ return en.mesh === hitMesh || en.mesh.children.indexOf(hitMesh) !== -1; });
        if(enemy){
          damageEnemy(enemy, w.damage);
          hitSomething = true;
        }
      }
    }
    if(hitSomething) playHitMarker();
  }

  function muzzleFlashEffect(){
    muzzleFlashEl.style.transition = 'none';
    muzzleFlashEl.style.opacity = 0.9;
    muzzleLight.intensity = 4;
    requestAnimationFrame(function(){
      muzzleFlashEl.style.transition = 'opacity .09s';
      muzzleFlashEl.style.opacity = 0;
    });
    setTimeout(function(){ muzzleLight.intensity = 0; }, 70);
  }

  function gunKick(amount){
    gunRecoilOffset = amount;
  }

  function cameraRecoil(amount){
    pitchObject.rotation.x += amount;
  }

  var tracers = [];
  function drawTracer(ray, dist){
    var muzzleWorld = new THREE.Vector3();
    gunBarrel.getWorldPosition(muzzleWorld);
    var end = ray.origin.clone().add(ray.direction.clone().multiplyScalar(Math.min(dist, 40)));
    var geom = new THREE.BufferGeometry().setFromPoints([muzzleWorld, end]);
    var mat = new THREE.LineBasicMaterial({color:0xfff2b0, transparent:true, opacity:0.9});
    var line = new THREE.Line(geom, mat);
    scene.add(line);
    tracers.push({line:line, life: 0.06});
  }

  function updateTracers(dt){
    for(var i=tracers.length-1;i>=0;i--){
      tracers[i].life -= dt;
      tracers[i].line.material.opacity = Math.max(0, tracers[i].life/0.06) * 0.9;
      if(tracers[i].life <= 0){
        scene.remove(tracers[i].line);
        tracers.splice(i,1);
      }
    }
  }

  function reload(){
    var w = WEAPONS[currentWeaponKey];
    if(player.reloading || ammoState[currentWeaponKey] === w.clipMax) return;
    player.reloading = true;
    playReload();
    ammoEl.textContent = 'RECHARGEMENT...';
    setTimeout(function(){
      ammoState[currentWeaponKey] = w.clipMax;
      player.reloading = false;
      updateAmmoHud();
    }, w.reloadTime*1000);
  }

  function updateAmmoHud(){
    var w = WEAPONS[currentWeaponKey];
    ammoEl.textContent = 'MUNITIONS: ' + ammoState[currentWeaponKey] + ' / ' + w.clipMax;
  }

  // ================= ENEMIES =================
  var enemies = [];
  var enemyGeo = new THREE.OctahedronGeometry(0.7, 0);
  function spawnEnemy(){
    var mat = new THREE.MeshStandardMaterial({color:0xff4040, emissive:0x441111, roughness:0.4});
    var mesh = new THREE.Mesh(enemyGeo, mat);

    var angle = Math.random()*Math.PI*2;
    var dist = 25 + Math.random()*10;
    var x = Math.cos(angle)*dist;
    var z = Math.sin(angle)*dist;
    mesh.position.set(x, 1.2, z);
    scene.add(mesh);

    var glow = new THREE.PointLight(0xff4040, 0.6, 4);
    mesh.add(glow);

    enemies.push({
      mesh: mesh,
      health: 100,
      speed: 2.6 + Math.random()*1.2,
      alive: true,
      attackCooldown: 0
    });
  }

  function damageEnemy(enemy, amount){
    if(!enemy.alive) return;
    enemy.health -= amount;
    enemy.mesh.material.emissive.setHex(0xffffff);
    setTimeout(function(){
      if(enemy.mesh && enemy.mesh.material) enemy.mesh.material.emissive.setHex(0x441111);
    }, 80);

    if(enemy.health <= 0){
      killEnemy(enemy);
    }
  }

  function killEnemy(enemy){
    enemy.alive = false;
    scene.remove(enemy.mesh);
    enemies = enemies.filter(function(en){ return en !== enemy; });
    score += 100;
    scoreEl.innerHTML = 'SCORE: ' + score + '<br>VAGUE: ' + wave + '<br><div id="map-name" style="font-size:12px; opacity:0.7; margin-top:4px;">MAP: ' + MAPS[(wave-1)%MAPS.length].name + '</div>';
    playKill();

    if(enemies.length === 0){
      wave++;
      setTimeout(startWave, 1500);
    }
  }

  function startWave(){
    if(!gameRunning) return;
    
    // Charger dynamiquement la map correspondante à la vague actuelle
    var mapIdx = (wave - 1) % MAPS.length;
    loadMap(mapIdx);

    var count = 3 + wave;
    for(var i=0;i<count;i++) spawnEnemy();
  }

  // ================= GAME FLOW =================
  function startGame(){
    gameRunning = true;
    gamePaused = false;
    player.health = 100;
    ammoState = {
      pistol: WEAPONS.pistol.clipMax, 
      shotgun: WEAPONS.shotgun.clipMax, 
      smg: WEAPONS.smg.clipMax,
      sniper: WEAPONS.sniper.clipMax
    };
    currentWeaponKey = 'pistol';
    isZoomed = false;
    switchWeapon('pistol');
    score = 0;
    wave = 1;
    updateAmmoHud();
    healthFill.style.width = '100%';
    enemies.forEach(function(en){ scene.remove(en.mesh); });
    enemies = [];
    yawObject.position.set(0, 1.7, 10);
    startWave();
  }

  function gameOver(){
    gameRunning = false;
    isZoomed = false;
    camera.fov = 75;
    camera.updateProjectionMatrix();
    document.exitPointerLock && document.exitPointerLock();
    document.getElementById('title').textContent = 'ÉLIMINÉ';
    document.getElementById('title').style.color = '#ff5050';
    document.getElementById('title').style.textShadow = '0 0 10px rgba(255,80,80,.9)';
    document.getElementById('sub').innerHTML = 'Score final : ' + score + '<br>Vague atteinte : ' + wave;
    startBtn.textContent = 'REJOUER';
    msg.classList.remove('hidden');

    enemies.forEach(function(en){ scene.remove(en.mesh); });
    enemies = [];
  }

  function takeDamage(amount){
    player.health -= amount;
    flashEl.style.opacity = 0.5;
    setTimeout(function(){ flashEl.style.opacity = 0; }, 120);
    if(player.health <= 0){
      player.health = 0;
      healthFill.style.width = '0%';
      gameOver();
      return;
    }
    healthFill.style.width = player.health + '%';
  }

  // ================= COLLISION =================
  function collidesWithObstacles(x, z, radius){
    for(var i=0;i<obstacles.length;i++){
      var o = obstacles[i];
      var dx = Math.abs(x - o.x);
      var dz = Math.abs(z - o.z);
      if(dx < o.hx + radius && dz < o.hz + radius){
        return true;
      }
    }
    return false;
  }

  // ================= MAIN LOOP =================
  var clock = new THREE.Clock();

  function animate(){
    requestAnimationFrame(animate);
    var dt = Math.min(clock.getDelta(), 0.05);

    if(gameRunning && !gamePaused){
      updatePlayer(dt);
      updateEnemies(dt);
      updateTracers(dt);
      updateGun(dt);
      particles.rotation.y += dt*0.02;

      // Gestion de la visée fluide (Interpolation FOV)
      var targetFov = (isZoomed && currentWeaponKey === 'sniper') ? 20 : 75;
      if (Math.abs(camera.fov - targetFov) > 0.1) {
        camera.fov += (targetFov - camera.fov) * 15 * dt;
        camera.updateProjectionMatrix();
      }

      if(mouseDown) tryShoot();
    }

    renderer.render(scene, camera);
  }

  function updateGun(dt){
    gunRecoilOffset = Math.max(0, gunRecoilOffset - dt*0.4);
    gunGroup.position.z = gunBasePos.z + gunRecoilOffset;

    var moving = (keys['KeyW']||keys['KeyA']||keys['KeyS']||keys['KeyD']||keys['ArrowUp']||keys['ArrowDown']||keys['ArrowLeft']||keys['ArrowRight']) && player.onGround;
    bobTime += dt * (moving ? 8 : 2);
    var bobAmt = moving ? 0.012 : 0.004;
    
    // Réduire l'effet de balancement si on vise au sniper
    if (isZoomed && currentWeaponKey === 'sniper') bobAmt *= 0.1;

    gunGroup.position.y += (gunBasePos.y + Math.sin(bobTime)*bobAmt - gunGroup.position.y) * Math.min(1, dt*10);
    gunGroup.position.x += (gunBasePos.x + Math.cos(bobTime*0.5)*bobAmt*0.5 - gunGroup.position.x) * Math.min(1, dt*10);
  }

  function updatePlayer(dt){
    var moveX = 0, moveZ = 0;
    if(keys['KeyW'] || keys['ArrowUp']) moveZ -= 1;
    if(keys['KeyS'] || keys['ArrowDown']) moveZ += 1;
    if(keys['KeyA'] || keys['ArrowLeft']) moveX -= 1;
    if(keys['KeyD'] || keys['ArrowRight']) moveX += 1;

    var len = Math.sqrt(moveX*moveX + moveZ*moveZ);
    if(len > 0){ moveX /= len; moveZ /= len; }

    var forward = new THREE.Vector3(0,0,-1).applyQuaternion(yawObject.quaternion);
    forward.y = 0; forward.normalize();
    var right = new THREE.Vector3(1,0,0).applyQuaternion(yawObject.quaternion);
    right.y = 0; right.normalize();

    // Ralentir la vitesse du joueur s'il utilise la lunette de visée
    var currentSpeed = player.speed;
    if (isZoomed && currentWeaponKey === 'sniper') currentSpeed *= 0.4;

    var dx = (forward.x*-moveZ + right.x*moveX) * currentSpeed * dt;
    var dz = (forward.z*-moveZ + right.z*moveX) * currentSpeed * dt;

    var newX = yawObject.position.x + dx;
    var newZ = yawObject.position.z + dz;

    var bound = half - 1.2;
    newX = Math.max(-bound, Math.min(bound, newX));
    newZ = Math.max(-bound, Math.min(bound, newZ));

    if(!collidesWithObstacles(newX, yawObject.position.z, player.radius)) yawObject.position.x = newX;
    if(!collidesWithObstacles(yawObject.position.x, newZ, player.radius)) yawObject.position.z = newZ;

    if((keys['Space']) && player.onGround){
      player.velocityY = 5.2;
      player.onGround = false;
    }
    player.velocityY -= 14 * dt;
    yawObject.position.y += player.velocityY * dt;
    if(yawObject.position.y <= 1.7){
      yawObject.position.y = 1.7;
      player.velocityY = 0;
      player.onGround = true;
    }
  }

  function updateEnemies(dt){
    enemies.forEach(function(enemy){
      if(!enemy.alive) return;
      var pos = enemy.mesh.position;
      var dirX = yawObject.position.x - pos.x;
      var dirZ = yawObject.position.z - pos.z;
      var dist = Math.sqrt(dirX*dirX + dirZ*dirZ);

      enemy.mesh.rotation.y += dt*2;
      enemy.mesh.position.y = 1.2 + Math.sin(Date.now()*0.004 + pos.x)*0.15;

      if(dist > 1.8){
        dirX /= dist; dirZ /= dist;
        var nx = pos.x + dirX*enemy.speed*dt;
        var nz = pos.z + dirZ*enemy.speed*dt;
        if(!collidesWithObstacles(nx, nz, 0.7)){
          pos.x = nx; pos.z = nz;
        }
      } else {
        enemy.attackCooldown -= dt;
        if(enemy.attackCooldown <= 0){
          takeDamage(10);
          enemy.attackCooldown = 1.0;
        }
      }
    });
  }

  animate();

})();
</script>
</body>
</html>
