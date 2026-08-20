<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>SWERVE — Hand-Controlled Traffic Dodger</title>
<style>
  :root{
    --bg: #05070d;
    --neon-cyan: #00c4e0;
    --neon-magenta: #ff2e7e;
    --neon-amber: #ffb800;
    --text-dim: #4a5568;
    --text: #101418;
  }
  *{ box-sizing: border-box; margin:0; padding:0; }
  html,body{
    width:100%; height:100%; overflow:hidden;
    background: #9ed8ff;
    font-family: 'Consolas', 'SF Mono', 'Courier New', monospace;
    color: var(--text);
  }
  #canvas-holder{ position:fixed; inset:0; }
  canvas{ display:block; }

  /* ---------- HUD ---------- */
  #hud{
    position:fixed; top:0; left:0; right:0;
    display:flex; justify-content:space-between; align-items:flex-start;
    padding:22px 28px;
    pointer-events:none;
    z-index:10;
  }
  .hud-block{
    display:flex; flex-direction:column; gap:2px;
    background: rgba(8,10,16,0.55);
    backdrop-filter: blur(3px);
    padding:8px 16px;
    border-radius:6px;
    border:1px solid rgba(255,255,255,0.08);
  }
  .hud-label{
    font-size:11px; letter-spacing:3px; color:#9aa4b5;
    text-transform:uppercase;
  }
  .hud-value{
    font-size:34px; font-weight:700; letter-spacing:1px;
    color:var(--neon-cyan);
    text-shadow: 0 0 12px rgba(0,196,224,0.55), 0 0 2px rgba(0,196,224,0.9);
  }
  .hud-value.amber{
    color:var(--neon-amber);
    text-shadow: 0 0 12px rgba(255,184,0,0.55), 0 0 2px rgba(255,184,0,0.9);
  }
  #hud-right{ text-align:right; align-items:flex-end; }
  #hud-center{ text-align:center; align-items:center; }
  #hud-center .hud-value{ font-size:18px; letter-spacing:3px; color:var(--neon-amber); text-shadow:none; }

  /* speed lines signature accent along top edge */
  #speed-strip{
    position:fixed; top:0; left:0; right:0; height:3px;
    background: linear-gradient(90deg, transparent, var(--neon-cyan), transparent);
    opacity:0.5; z-index:11; pointer-events:none;
    animation: strip-pulse 2.4s ease-in-out infinite;
  }
  @keyframes strip-pulse{ 0%,100%{opacity:.25;} 50%{opacity:.65;} }

  /* ---------- webcam preview ---------- */
  #cam-wrap{
    position:fixed; bottom:22px; left:22px;
    width:150px; height:112px;
    border:1px solid rgba(0,196,224,0.45);
    border-radius:6px;
    overflow:hidden;
    background:#000;
    z-index:10;
    box-shadow: 0 0 18px rgba(0,0,0,0.25);
  }
  #cam-wrap video, #cam-wrap canvas{
    position:absolute; inset:0;
    width:100%; height:100%;
    object-fit:cover;
    transform: scaleX(-1);
  }
  #cam-label{
    position:absolute; bottom:0; left:0; right:0;
    font-size:9px; letter-spacing:2px; text-align:center;
    padding:3px 0; color:#9aa4b5;
    background:rgba(0,0,0,0.55);
  }
  #hand-indicator{
    position:absolute; top:6px; right:6px;
    width:8px; height:8px; border-radius:50%;
    background:#3a4356;
    box-shadow:0 0 6px rgba(0,0,0,0.5);
    transition: background .15s, box-shadow .15s;
  }
  #hand-indicator.active{
    background: var(--neon-magenta);
    box-shadow: 0 0 8px var(--neon-magenta);
  }

  /* ---------- overlays ---------- */
  .overlay{
    position:fixed; inset:0;
    display:flex; align-items:center; justify-content:center;
    flex-direction:column;
    z-index:30;
    background: radial-gradient(ellipse at center, rgba(5,7,13,0.75) 0%, rgba(5,7,13,0.97) 100%);
    text-align:center;
    padding:24px;
    color: var(--text-dim);
  }
  .hidden{ display:none !important; }

  .title{
    font-size:52px; font-weight:800; letter-spacing:6px;
    color:#e8edf5;
    text-shadow: 0 0 24px rgba(0,196,224,0.4);
    margin-bottom:6px;
  }
  .title span{ color:var(--neon-magenta); text-shadow: 0 0 24px rgba(255,46,126,0.5); }
  .subtitle{
    font-size:13px; letter-spacing:2px; color:#8b93a5;
    margin-bottom:30px; max-width:460px; line-height:1.7;
  }
  .subtitle b{ color:var(--neon-amber); }

  .diff-row{ display:flex; gap:16px; margin-bottom:8px; }

  .btn{
    font-family:inherit;
    background:transparent;
    border:1px solid var(--neon-cyan);
    color:var(--neon-cyan);
    font-size:15px; letter-spacing:3px;
    padding:14px 30px;
    cursor:pointer;
    text-transform:uppercase;
    transition: all .15s ease;
  }
  .btn:hover{
    background: rgba(0,196,224,0.1);
    box-shadow: 0 0 22px rgba(0,196,224,0.35);
  }
  .btn.retry{ border-color: var(--neon-magenta); color:var(--neon-magenta); }
  .btn.retry:hover{ background: rgba(255,46,126,0.08); box-shadow: 0 0 22px rgba(255,46,126,0.35); }
  .btn.normal{ border-color: var(--neon-amber); color:var(--neon-amber); }
  .btn.normal:hover{ background: rgba(255,184,0,0.08); box-shadow: 0 0 22px rgba(255,184,0,0.35); }
  .btn.hard{ border-color: var(--neon-magenta); color:var(--neon-magenta); }
  .btn.hard:hover{ background: rgba(255,46,126,0.08); box-shadow: 0 0 22px rgba(255,46,126,0.35); }
  .btn.home{ border-color: #8b93a5; color:#c7ccd6; }
  .btn.home:hover{ background: rgba(255,255,255,0.06); box-shadow: 0 0 22px rgba(255,255,255,0.15); }

  #status-line{
    margin-top:22px; font-size:11px; letter-spacing:2px; color:#8b93a5;
    min-height:16px;
  }

  #gameover-score{
    font-size:64px; font-weight:800; color:var(--neon-magenta);
    text-shadow:0 0 30px rgba(255,46,126,0.55);
    margin: 8px 0 2px;
  }
  #gameover-diff{ font-size:11px; letter-spacing:2px; color:#8b93a5; margin-bottom:10px; }
  #gameover-high{
    font-size:14px; letter-spacing:2px; color:var(--neon-amber);
    margin-bottom:30px;
  }
  #gameover-high.new-high{ animation: flash 0.6s ease-in-out 3; }
  @keyframes flash{ 0%,100%{opacity:1;} 50%{opacity:0.35;} }

  /* red damage flash on collision */
  #flash{
    position:fixed; inset:0; background:var(--neon-magenta);
    opacity:0; pointer-events:none; z-index:25;
    mix-blend-mode: screen;
  }

  /* control hint chip */
  #control-hint{
    position:fixed; bottom:22px; right:22px;
    font-size:10px; letter-spacing:1.5px; color:#dfe4ee;
    text-align:right; line-height:1.6; z-index:10;
    background: rgba(8,10,16,0.55);
    padding:8px 14px; border-radius:6px;
    border:1px solid rgba(255,255,255,0.08);
  }
  #control-hint b{ color:#fff; }
</style>
</head>
<body>

<div id="canvas-holder"></div>
<div id="speed-strip"></div>
<div id="flash"></div>

<!-- HUD -->
<div id="hud">
  <div class="hud-block">
    <div class="hud-label">Score</div>
    <div class="hud-value" id="score-val">0</div>
  </div>
  <div class="hud-block" id="hud-center">
    <div class="hud-label">Difficulty</div>
    <div class="hud-value" id="diff-val">—</div>
  </div>
  <div class="hud-block" id="hud-right">
    <div class="hud-label">All-Time High</div>
    <div class="hud-value amber" id="high-val">0</div>
  </div>
</div>

<div id="cam-wrap">
  <video id="webcam" autoplay playsinline muted></video>
  <canvas id="cam-overlay"></canvas>
  <div id="hand-indicator"></div>
  <div id="cam-label">HAND TRACKING</div>
</div>

<div id="control-hint">
  Move hand <b>left / right</b> to snap lanes<br>
  (Arrow keys / A·D also work — tap, don't hold)
</div>

<!-- Start screen -->
<div class="overlay" id="start-screen">
  <div class="title">SW<span>ERVE</span></div>
  <div class="subtitle">
    You are the car. Oncoming traffic fills the highway across 4 lanes.
    Show the camera your hand and move <b>left</b> or <b>right</b> to snap lanes and dodge — every car you clear scores a point.
  </div>
  <div class="diff-row">
    <button class="btn" id="btn-easy" data-diff="easy">Easy</button>
    <button class="btn normal" id="btn-normal" data-diff="normal">Normal</button>
    <button class="btn hard" id="btn-hard" data-diff="hard">Hard</button>
  </div>
  <div id="status-line">pick a difficulty to begin — camera access will be requested</div>
</div>

<!-- Game over screen -->
<div class="overlay hidden" id="gameover-screen">
  <div class="hud-label">Wrecked</div>
  <div id="gameover-score">0</div>
  <div id="gameover-diff">Difficulty: —</div>
  <div id="gameover-high">ALL-TIME HIGH — 0</div>
  <div class="diff-row">
    <button class="btn retry" id="retry-btn">Retry</button>
    <button class="btn home" id="home-btn">Home</button>
  </div>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands@0.4.1675469240/hands.js" crossorigin="anonymous"></script>

<script>
/* =========================================================================
   SWERVE — hand-controlled 3D traffic dodger
   Sections: [1] Three.js scene + scenery  [2] Traffic/game logic
   [3] Hand tracking  [4] Collision / explosion / shake
   [5] Game state, difficulty & UI wiring
   ========================================================================= */

/* ---------------------------------------------------------------------
   [1] THREE.JS SCENE SETUP — daytime, road + fence + sidewalk + buildings
--------------------------------------------------------------------- */
const holder = document.getElementById('canvas-holder');
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x9ed8ff);
scene.fog = new THREE.Fog(0xbfe6ff, 55, 190);

const camera = new THREE.PerspectiveCamera(62, window.innerWidth / window.innerHeight, 0.1, 300);
const baseCamOffset = new THREE.Vector3(0, 5.4, 9.5);
camera.position.copy(baseCamOffset);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
holder.appendChild(renderer.domElement);

window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});

// daytime lighting
scene.add(new THREE.HemisphereLight(0xbfe3ff, 0x5a9451, 0.95));
const sun = new THREE.DirectionalLight(0xfff4d6, 1.1);
sun.position.set(30, 45, 15);
scene.add(sun);
scene.add(new THREE.AmbientLight(0xffffff, 0.25));

// ----- road -----
const ROAD_WIDTH = 16;      // 4 lanes, 4 units each
const ROAD_HALF = ROAD_WIDTH / 2;
const ROAD_LENGTH = 400;
const ROAD_Z = -ROAD_LENGTH / 2 + 20;

// procedural asphalt texture (speckle + expansion-joint seams) so the road
// surface itself visibly scrolls, instead of being one flat static slab
function makeAsphaltTexture(){
  const canvas = document.createElement('canvas');
  canvas.width = 64; canvas.height = 256;
  const ctx = canvas.getContext('2d');
  ctx.fillStyle = '#3a3f47';
  ctx.fillRect(0, 0, 64, 256);
  for (let i = 0; i < 900; i++) {
    const x = Math.random() * 64, y = Math.random() * 256;
    const b = Math.floor(Math.random() * 30) - 15;
    ctx.fillStyle = `rgba(${58 + b},${63 + b},${71 + b},0.5)`;
    ctx.fillRect(x, y, 1.4, 1.4);
  }
  ctx.fillStyle = 'rgba(18,20,24,0.65)';
  for (let y = 0; y < 256; y += 64) ctx.fillRect(0, y, 64, 2);
  const tex = new THREE.CanvasTexture(canvas);
  tex.wrapS = THREE.RepeatWrapping;
  tex.wrapT = THREE.RepeatWrapping;
  tex.repeat.set(3, 55);
  return tex;
}
const roadTexture = makeAsphaltTexture();
const roadMat = new THREE.MeshStandardMaterial({ map: roadTexture, color: 0xffffff, roughness: 0.95, metalness: 0.05 });
const road = new THREE.Mesh(new THREE.PlaneGeometry(ROAD_WIDTH, ROAD_LENGTH), roadMat);
road.rotation.x = -Math.PI / 2;
road.position.z = ROAD_Z;
scene.add(road);

// road edge lines
function makeEdge(x){
  const geo = new THREE.BoxGeometry(0.15, 0.05, ROAD_LENGTH);
  const mat = new THREE.MeshStandardMaterial({ color: 0xf5f0e0, roughness: 0.6 });
  const edge = new THREE.Mesh(geo, mat);
  edge.position.set(x, 0.04, ROAD_Z);
  scene.add(edge);
}
makeEdge(-ROAD_HALF + 0.15);
makeEdge(ROAD_HALF - 0.15);

// lane dividers (3 dashed lines splitting 4 lanes)
const dashes = [];
const DASH_COUNT = 40;
const dashMat = new THREE.MeshStandardMaterial({ color: 0xfff2b0, roughness: 0.6 });
[-4, 0, 4].forEach(lineX => {
  for (let i = 0; i < DASH_COUNT; i++) {
    const dash = new THREE.Mesh(new THREE.BoxGeometry(0.15, 0.05, 1.4), dashMat);
    dash.position.set(lineX, 0.03, -i * 6);
    scene.add(dash);
    dashes.push(dash);
  }
});

// ----- grass beyond the road -----
const groundMat = new THREE.MeshStandardMaterial({ color: 0x5a9451, roughness: 1 });
const ground = new THREE.Mesh(new THREE.PlaneGeometry(500, 500), groundMat);
ground.rotation.x = -Math.PI / 2;
ground.position.y = -0.03;
scene.add(ground);

// ----- fence (between road and sidewalk) -----
// posts are tracked so the main loop can scroll + recycle them, same trick
// as the lane dashes — otherwise only the traffic ever appears to move
const fencePosts = [];
const FENCE_POST_SPACING = 4;
function buildFence(sideSign){
  const xBase = sideSign * (ROAD_HALF + 0.8);
  const railMat = new THREE.MeshStandardMaterial({ color: 0xe4e7ec, metalness: 0.3, roughness: 0.5 });
  [0.45, 0.85].forEach(y => {
    const rail = new THREE.Mesh(new THREE.BoxGeometry(0.08, 0.08, ROAD_LENGTH), railMat);
    rail.position.set(xBase, y, ROAD_Z);
    scene.add(rail);
  });
  const postMat = new THREE.MeshStandardMaterial({ color: 0xb7bcc4, metalness: 0.3, roughness: 0.6 });
  const postCount = Math.floor(ROAD_LENGTH / FENCE_POST_SPACING);
  for (let i = 0; i < postCount; i++) {
    const post = new THREE.Mesh(new THREE.BoxGeometry(0.1, 1.0, 0.1), postMat);
    post.position.set(xBase, 0.5, 20 - i * FENCE_POST_SPACING);
    scene.add(post);
    fencePosts.push(post);
  }
}
buildFence(-1);
buildFence(1);
const FENCE_SPAN = Math.floor(ROAD_LENGTH / FENCE_POST_SPACING) * FENCE_POST_SPACING;

// ----- sidewalk (after the fence) -----
const SIDEWALK_WIDTH = 4.6;
function buildSidewalk(sideSign){
  const xCenter = sideSign * (ROAD_HALF + 0.8 + SIDEWALK_WIDTH / 2 + 0.3);
  const sidewalk = new THREE.Mesh(
    new THREE.PlaneGeometry(SIDEWALK_WIDTH, ROAD_LENGTH),
    new THREE.MeshStandardMaterial({ color: 0xc7cbd1, roughness: 0.9 })
  );
  sidewalk.rotation.x = -Math.PI / 2;
  sidewalk.position.set(xCenter, 0.05, ROAD_Z);
  scene.add(sidewalk);
  // return the OUTER edge as a plain distance-from-center magnitude,
  // not a signed coordinate, so both sides can share the same math below
  return Math.abs(xCenter) + SIDEWALK_WIDTH / 2;
}
const sidewalkOuterEdgeL = buildSidewalk(-1);
const sidewalkOuterEdgeR = buildSidewalk(1);

// ----- buildings (one straight row right next to each sidewalk) -----
// each building is grouped with its roof cap and tracked in an array so
// the main loop can scroll the whole skyline toward the camera and wrap
// it back, the same recycling trick used for the lane dashes and fence
const buildingGroups = []; // { group, span } — span = total row length to wrap by
const buildingColors = [0xd8cfc0, 0xc7b7a3, 0xb7c0c9, 0x9aa6b0, 0xe0c9a6, 0xcdd3d8, 0xa9b4a0];
function buildBuildings(sideSign, edgeDist){
  // every building's near face sits on this exact line, regardless of its
  // own width — that's what keeps the whole row flush and straight
  const lineDist = edgeDist + 1.0;
  const count = 20;
  const zStep = ROAD_LENGTH / count;
  const rowSpan = count * zStep;

  for (let i = 0; i < count; i++) {
    const w = 6 + Math.random() * 7;   // depth away from the road (x)
    const d = zStep * 0.82;            // length along the street (z), sized to avoid gaps/overlaps
    const h = 7 + Math.random() * 24;
    const mat = new THREE.MeshStandardMaterial({
      color: buildingColors[Math.floor(Math.random() * buildingColors.length)],
      roughness: 0.85
    });
    const b = new THREE.Mesh(new THREE.BoxGeometry(w, h, d), mat);
    b.position.set(0, h / 2, 0);

    const cap = new THREE.Mesh(
      new THREE.BoxGeometry(w * 0.6, 0.6, d * 0.6),
      new THREE.MeshStandardMaterial({ color: 0x707a86, roughness: 0.7 })
    );
    cap.position.set(0, h + 0.3, 0);

    // center offset by half its own width so the INNER face always
    // lands exactly at sideSign * lineDist -> flush straight-line row
    const xOff = sideSign * (lineDist + w / 2);
    const z = 20 - i * zStep - d / 2;

    const group = new THREE.Group();
    group.add(b, cap);
    group.position.set(xOff, 0, z);
    scene.add(group);
    buildingGroups.push({ group, span: rowSpan });
  }
}
buildBuildings(-1, sidewalkOuterEdgeL);
buildBuildings(1, sidewalkOuterEdgeR);

/* ------- car builder (stylised low-poly car made of boxes) -------
   Local "front" of the geometry faces -z (headlights, white) and the
   "back" faces +z (brake/tail lights, red). Whichever world direction
   a car actually travels in is handled by rotating the whole group. */
function buildCar(bodyColor, cabinColor){
  const group = new THREE.Group();

  const body = new THREE.Mesh(
    new THREE.BoxGeometry(1.7, 0.5, 3.4),
    new THREE.MeshStandardMaterial({ color: bodyColor, metalness: 0.4, roughness: 0.35 })
  );
  body.position.y = 0.42;
  group.add(body);

  const cabin = new THREE.Mesh(
    new THREE.BoxGeometry(1.3, 0.45, 1.7),
    new THREE.MeshStandardMaterial({ color: cabinColor, metalness: 0.2, roughness: 0.2 })
  );
  cabin.position.set(0, 0.86, -0.15);
  group.add(cabin);

  const wheelGeo = new THREE.CylinderGeometry(0.32, 0.32, 0.32, 12);
  const wheelMat = new THREE.MeshStandardMaterial({ color: 0x0a0a0a, roughness: 0.8 });
  const wheelPositions = [
    [-0.85, 0.32, 1.1], [0.85, 0.32, 1.1],
    [-0.85, 0.32, -1.1], [0.85, 0.32, -1.1]
  ];
  wheelPositions.forEach(p => {
    const wheel = new THREE.Mesh(wheelGeo, wheelMat);
    wheel.rotation.z = Math.PI / 2;
    wheel.position.set(p[0], p[1], p[2]);
    group.add(wheel);
  });

  // front = headlights (white), back = brake/tail lights (red)
  const lightGeo = new THREE.BoxGeometry(0.25, 0.12, 0.08);
  const headMat = new THREE.MeshStandardMaterial({ color: 0xffffff, emissive: 0xffffff, emissiveIntensity: 1.2 });
  const tailMat = new THREE.MeshStandardMaterial({ color: 0xff2e2e, emissive: 0xff2e2e, emissiveIntensity: 1.6 });
  [-0.6, 0.6].forEach(x => {
    const hl = new THREE.Mesh(lightGeo, headMat);
    hl.position.set(x, 0.45, -1.72);
    group.add(hl);
    const tl = new THREE.Mesh(lightGeo, tailMat);
    tl.position.set(x, 0.45, 1.72);
    group.add(tl);
  });

  return group;
}

// player car — travels in -z (down the road, away from camera), so its
// default "back" (+z, brake lights) faces the camera. No flip needed:
// this is what makes the lights we see behind it brake lights, not headlights.
const player = buildCar(0x00c2ff, 0x0a1420);
player.position.set(0, 0, 2.5);
scene.add(player);

/* ---------------------------------------------------------------------
   [2] TRAFFIC / GAME LOGIC
--------------------------------------------------------------------- */
const trafficColors = [
  [0xff3b3b, 0x2a0a0a], [0xffb020, 0x2a1c05], [0x9d4bff, 0x1c0a2a],
  [0x3bff8e, 0x0a2a17], [0xff6fb0, 0x2a0a1a]
];

const LANES = [-6, -2, 2, 6]; // four lane centers
let currentLaneIndex = 1;     // player's snapped lane

const obstacles = []; // {mesh, x, scored}

let spawnTimer = 0;
let spawnInterval = 1.15;
let baseSpeed = 16;
let elapsed = 0;

function spawnObstacle(){
  const [bodyC, cabinC] = trafficColors[Math.floor(Math.random() * trafficColors.length)];
  const mesh = buildCar(bodyC, cabinC);
  const lane = LANES[Math.floor(Math.random() * LANES.length)] + (Math.random() - 0.5) * 1.4;
  mesh.position.set(lane, 0, -90 - Math.random() * 10);
  // same orientation as the player car — brake/tail lights (red) face the
  // player, keeping every car on the road visually consistent.
  scene.add(mesh);
  obstacles.push({ mesh, x: lane, scored: false });
}

/* ---------------------------------------------------------------------
   [3] HAND TRACKING (MediaPipe Hands) — drives lane snapping
--------------------------------------------------------------------- */
const videoEl = document.getElementById('webcam');
const camOverlay = document.getElementById('cam-overlay');
const camCtx = camOverlay.getContext('2d');
const handIndicator = document.getElementById('hand-indicator');
const statusLine = document.getElementById('status-line');

let handTargetX = 0;      // normalized -1..1 hand position
let handActive = false;
let handAvailable = false;

let hands = null;

async function initHandTracking(){
  try {
    const stream = await navigator.mediaDevices.getUserMedia({ video: { width: 320, height: 240 }, audio: false });
    videoEl.srcObject = stream;
    await videoEl.play();
    camOverlay.width = 320;
    camOverlay.height = 240;

    hands = new Hands({
      locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands@0.4.1675469240/${file}`
    });
    hands.setOptions({
      maxNumHands: 1,
      modelComplexity: 0,
      minDetectionConfidence: 0.6,
      minTrackingConfidence: 0.5
    });
    hands.onResults(onHandResults);

    handAvailable = true;
    statusLine.textContent = 'camera ready — move your hand to snap lanes';

    const pump = async () => {
      if (videoEl.readyState >= 2) {
        await hands.send({ image: videoEl });
      }
      requestAnimationFrame(pump);
    };
    pump();
  } catch (err) {
    handAvailable = false;
    statusLine.textContent = 'camera unavailable — use arrow keys / A·D to steer';
    document.getElementById('cam-wrap').style.opacity = '0.35';
  }
}

function onHandResults(results){
  camCtx.save();
  camCtx.clearRect(0, 0, camOverlay.width, camOverlay.height);

  if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
    handActive = true;
    handIndicator.classList.add('active');
    const lm = results.multiHandLandmarks[0];

    camCtx.fillStyle = '#00c4e0';
    lm.forEach(pt => {
      camCtx.beginPath();
      camCtx.arc(pt.x * camOverlay.width, pt.y * camOverlay.height, 2.5, 0, Math.PI * 2);
      camCtx.fill();
    });

    const cx = (lm[0].x + lm[9].x) / 2;
    // video is mirrored via CSS, so this maps naturally: hand right -> car right
    handTargetX = (cx - 0.5) * 2;
  } else {
    handActive = false;
    handIndicator.classList.remove('active');
  }
  camCtx.restore();
}

function laneIndexFromHandX(hx){
  const t = Math.max(0, Math.min(1, (hx + 1) / 2)); // 0..1
  return Math.max(0, Math.min(LANES.length - 1, Math.floor(t * LANES.length)));
}

// keyboard fallback — a single tap snaps one lane over, no need to hold
window.addEventListener('keydown', (e) => {
  if (e.repeat || !gameRunning) return;
  const key = e.key.toLowerCase();
  if (e.key === 'ArrowLeft' || key === 'a') currentLaneIndex = Math.max(0, currentLaneIndex - 1);
  if (e.key === 'ArrowRight' || key === 'd') currentLaneIndex = Math.min(LANES.length - 1, currentLaneIndex + 1);
});

/* ---------------------------------------------------------------------
   [4] COLLISION / EXPLOSION / CAMERA SHAKE
--------------------------------------------------------------------- */
let shakeTime = 0;
const SHAKE_DURATION = 0.55;
let explosionParticles = [];

function triggerExplosion(position){
  const count = 26;
  for (let i = 0; i < count; i++) {
    const size = 0.12 + Math.random() * 0.22;
    const colorChoices = [0xff6a00, 0xffce00, 0xff2e2e, 0x666666];
    const mat = new THREE.MeshStandardMaterial({
      color: colorChoices[Math.floor(Math.random() * colorChoices.length)],
      emissive: 0xff5500,
      emissiveIntensity: 0.9
    });
    const geo = Math.random() > 0.5
      ? new THREE.BoxGeometry(size, size, size)
      : new THREE.SphereGeometry(size * 0.6, 6, 6);
    const p = new THREE.Mesh(geo, mat);
    p.position.copy(position);
    p.position.y += 0.4;
    scene.add(p);

    const angle = Math.random() * Math.PI * 2;
    const upSpeed = 3 + Math.random() * 5;
    const outSpeed = 2 + Math.random() * 5;
    explosionParticles.push({
      mesh: p,
      vel: new THREE.Vector3(Math.cos(angle) * outSpeed, upSpeed, Math.sin(angle) * outSpeed),
      life: 1.0
    });
  }

  const flash = document.getElementById('flash');
  flash.style.transition = 'none';
  flash.style.opacity = '0.55';
  requestAnimationFrame(() => {
    flash.style.transition = 'opacity 0.6s ease-out';
    flash.style.opacity = '0';
  });

  shakeTime = SHAKE_DURATION;
}

function updateExplosion(dt){
  for (let i = explosionParticles.length - 1; i >= 0; i--) {
    const part = explosionParticles[i];
    part.vel.y -= 9.8 * dt;
    part.mesh.position.addScaledVector(part.vel, dt);
    part.life -= dt * 1.1;
    part.mesh.scale.setScalar(Math.max(part.life, 0));
    part.mesh.material.opacity = Math.max(part.life, 0);
    part.mesh.material.transparent = true;
    if (part.life <= 0) {
      scene.remove(part.mesh);
      explosionParticles.splice(i, 1);
    }
  }
}

function updateCameraShake(dt){
  if (shakeTime > 0) {
    shakeTime = Math.max(0, shakeTime - dt);
    const strength = (shakeTime / SHAKE_DURATION) * 0.5;
    camera.position.set(
      baseCamOffset.x + player.position.x * 0.35 + (Math.random() - 0.5) * strength,
      baseCamOffset.y + (Math.random() - 0.5) * strength,
      baseCamOffset.z + (Math.random() - 0.5) * strength
    );
  } else {
    const desiredX = player.position.x * 0.35;
    camera.position.x += (baseCamOffset.x + desiredX - camera.position.x) * 0.08;
    camera.position.y = baseCamOffset.y;
    camera.position.z = baseCamOffset.z;
  }
  camera.lookAt(player.position.x * 0.5, 0.6, -8);
}

/* ---------------------------------------------------------------------
   [5] GAME STATE, DIFFICULTY & MAIN LOOP
--------------------------------------------------------------------- */
const DIFFICULTIES = {
  easy:   { label: 'Easy',   startSpeed: 12, ramp: 0.18, spawnStart: 1.4,  spawnMin: 0.70 },
  normal: { label: 'Normal', startSpeed: 16, ramp: 0.35, spawnStart: 1.15, spawnMin: 0.45 },
  hard:   { label: 'Hard',   startSpeed: 21, ramp: 0.55, spawnStart: 0.9,  spawnMin: 0.30 }
};
let selectedDifficulty = DIFFICULTIES.normal;

let score = 0;
let highScore = parseInt(localStorage.getItem('swerveHighScore') || '0', 10);
document.getElementById('high-val').textContent = highScore;

let gameRunning = false;
let steerX = 0;

const scoreEl = document.getElementById('score-val');
const diffValEl = document.getElementById('diff-val');
const startScreen = document.getElementById('start-screen');
const gameoverScreen = document.getElementById('gameover-screen');
const gameoverScoreEl = document.getElementById('gameover-score');
const gameoverDiffEl = document.getElementById('gameover-diff');
const gameoverHighEl = document.getElementById('gameover-high');

function resetGame(){
  obstacles.forEach(o => scene.remove(o.mesh));
  obstacles.length = 0;
  explosionParticles.forEach(p => scene.remove(p.mesh));
  explosionParticles.length = 0;

  score = 0;
  scoreEl.textContent = '0';
  diffValEl.textContent = selectedDifficulty.label.toUpperCase();
  spawnTimer = 0;
  spawnInterval = selectedDifficulty.spawnStart;
  baseSpeed = selectedDifficulty.startSpeed;
  elapsed = 0;
  currentLaneIndex = 1;
  steerX = LANES[currentLaneIndex];
  handTargetX = 0;
  player.position.set(steerX, 0, 2.5);
  player.rotation.z = 0;
  camera.position.copy(baseCamOffset);
  shakeTime = 0;

  gameoverScreen.classList.add('hidden');
  gameRunning = true;
}

function endGame(){
  gameRunning = false;
  triggerExplosion(player.position);
  player.visible = false;

  if (score > highScore) {
    highScore = score;
    localStorage.setItem('swerveHighScore', String(highScore));
    document.getElementById('high-val').textContent = highScore;
    gameoverHighEl.classList.add('new-high');
    gameoverHighEl.textContent = `NEW ALL-TIME HIGH — ${highScore}`;
  } else {
    gameoverHighEl.classList.remove('new-high');
    gameoverHighEl.textContent = `ALL-TIME HIGH — ${highScore}`;
  }
  gameoverDiffEl.textContent = `Difficulty: ${selectedDifficulty.label}`;

  setTimeout(() => {
    gameoverScoreEl.textContent = score;
    gameoverScreen.classList.remove('hidden');
    player.visible = true;
    player.position.set(steerX, -2, 2.5);
  }, 550);
}

const clock = new THREE.Clock();

function animate(){
  requestAnimationFrame(animate);
  const dt = Math.min(clock.getDelta(), 0.05);

  const dashSpeed = gameRunning ? baseSpeed : 6;
  dashes.forEach(d => {
    d.position.z += dashSpeed * dt;
    if (d.position.z > 8) d.position.z -= DASH_COUNT * 6;
  });

  // scroll the whole backdrop toward the camera so the world reads as
  // moving, not just the traffic — the road surface, the fence, and the
  // skyline all shift at (roughly) the same speed and wrap back like a loop
  roadTexture.offset.y -= dashSpeed * dt / 8;

  fencePosts.forEach(post => {
    post.position.z += dashSpeed * dt;
    if (post.position.z > 20) post.position.z -= FENCE_SPAN;
  });

  const buildingSpeed = dashSpeed * 0.88; // slightly slower than the road = subtle depth parallax
  buildingGroups.forEach(({ group, span }) => {
    group.position.z += buildingSpeed * dt;
    if (group.position.z > 40) group.position.z -= span;
  });

  if (gameRunning) {
    elapsed += dt;

    // --- lane snapping steering ---
    if (handAvailable && handActive) {
      currentLaneIndex = laneIndexFromHandX(handTargetX);
    }
    const desiredX = LANES[currentLaneIndex];
    steerX += (desiredX - steerX) * Math.min(1, dt * 8);
    player.position.x = steerX;
    player.rotation.z = (desiredX - steerX) * -0.15;

    // --- difficulty ramp ---
    baseSpeed = selectedDifficulty.startSpeed + elapsed * selectedDifficulty.ramp;
    spawnInterval = Math.max(selectedDifficulty.spawnMin, selectedDifficulty.spawnStart - elapsed * 0.012);

    // --- spawn obstacles ---
    spawnTimer -= dt;
    if (spawnTimer <= 0) {
      spawnObstacle();
      spawnTimer = spawnInterval;
    }

    // --- move obstacles + collision + scoring ---
    for (let i = obstacles.length - 1; i >= 0; i--) {
      const o = obstacles[i];
      o.mesh.position.z += baseSpeed * dt;

      if (!o.scored && Math.abs(o.mesh.position.z - player.position.z) < 1.5) {
        const dx = Math.abs(o.mesh.position.x - player.position.x);
        if (dx < 1.55) {
          o.scored = true;
          endGame();
        } else if (o.mesh.position.z > player.position.z) {
          o.scored = true;
          score++;
          scoreEl.textContent = score;
        }
      }

      if (o.mesh.position.z > 15) {
        scene.remove(o.mesh);
        obstacles.splice(i, 1);
      }
    }
  }

  updateExplosion(dt);
  updateCameraShake(dt);

  renderer.render(scene, camera);
}
animate();

/* ---------------------------------------------------------------------
   UI WIRING
--------------------------------------------------------------------- */
document.querySelectorAll('.diff-row .btn').forEach(btn => {
  btn.addEventListener('click', async () => {
    selectedDifficulty = DIFFICULTIES[btn.dataset.diff];
    startScreen.classList.add('hidden');
    statusLine.textContent = 'requesting camera…';
    await initHandTracking();
    resetGame();
  });
});

document.getElementById('retry-btn').addEventListener('click', () => {
  resetGame();
});

document.getElementById('home-btn').addEventListener('click', () => {
  gameRunning = false;
  gameoverScreen.classList.add('hidden');
  startScreen.classList.remove('hidden');
  statusLine.textContent = handAvailable
    ? 'camera ready — pick a difficulty to begin'
    : 'camera unavailable — use arrow keys / A·D to steer';

  obstacles.forEach(o => scene.remove(o.mesh));
  obstacles.length = 0;
  explosionParticles.forEach(p => scene.remove(p.mesh));
  explosionParticles.length = 0;

  currentLaneIndex = 1;
  steerX = LANES[currentLaneIndex];
  score = 0;
  scoreEl.textContent = '0';
  player.visible = true;
  player.rotation.z = 0;
  player.position.set(steerX, 0, 2.5);
  camera.position.copy(baseCamOffset);
  shakeTime = 0;
});
</script>
</body>
</html>
