<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no">
<title>3D Street Racer</title>

<style>
*{box-sizing:border-box}
html,body{
  margin:0;
  width:100%;
  height:100%;
  overflow:hidden;
  touch-action:none;
  font-family:Arial,sans-serif;
  background:#87ceeb;
}
canvas{display:block}

#hud{
  position:fixed;
  top:10px;
  left:10px;
  right:10px;
  z-index:5;
  display:flex;
  justify-content:space-between;
  gap:8px;
  color:#fff;
  font-size:15px;
  font-weight:bold;
  text-shadow:2px 2px 4px #000;
  pointer-events:none;
}

.box{
  background:rgba(0,0,0,.45);
  padding:7px 10px;
  border-radius:10px;
}

#menu{
  position:fixed;
  inset:0;
  z-index:20;
  display:flex;
  flex-direction:column;
  justify-content:center;
  align-items:center;
  background:rgba(0,0,0,.75);
  color:#fff;
  text-align:center;
}

#menu h1{
  margin:5px;
  font-size:36px;
}

#menu p{
  line-height:1.6;
}

button{
  border:0;
  border-radius:13px;
  padding:14px 28px;
  margin:6px;
  font-size:18px;
  font-weight:bold;
}

#raceBtn{background:#16c765;color:white}
#freeBtn{background:#1976d2;color:white}

#touch{
  position:fixed;
  bottom:18px;
  left:0;
  right:0;
  z-index:10;
  display:none;
  justify-content:space-between;
  padding:0 18px;
  pointer-events:none;
}

.touchBtn{
  width:75px;
  height:75px;
  border-radius:50%;
  border:2px solid rgba(255,255,255,.7);
  background:rgba(0,0,0,.45);
  color:white;
  font-size:30px;
  pointer-events:auto;
}

#nitro{
  width:85px;
  height:55px;
  font-size:16px;
  background:rgba(220,40,30,.75);
}
</style>
</head>

<body>

<div id="hud">
  <div class="box">🏁 <span id="position">1/6</span></div>
  <div class="box">🏆 <span id="score">0</span></div>
  <div class="box">⚡ <span id="speed">0</span></div>
  <div class="box">🏆 <span id="best">0</span></div>
</div>

<div id="menu">
  <h1>🏎️ 3D STREET RACER</h1>
  <p>
    Swipe left/right to steer<br>
    Collect coins and avoid cars
  </p>

  <div>
    <button id="raceBtn">🏁 RACE</button>
    <button id="freeBtn">🚗 FREE DRIVE</button>
  </div>
</div>

<div id="touch">
  <button class="touchBtn" id="left">◀</button>
  <button class="touchBtn" id="nitro">NITRO</button>
  <button class="touchBtn" id="right">▶</button>
</div>

<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/build/three.min.js"></script>

<script>

/* =========================
   BASIC SETUP
========================= */

let scene;
let camera;
let renderer;

let player;

let traffic=[];
let opponents=[];
let coins=[];
let scenery=[];

let running=false;
let mode="race";

let score=0;

let best=
Number(localStorage.getItem("streetRacerBest")||0);

let distance=0;

let speed=0;
let baseSpeed=.65;

let nitro=false;

let lane=0;

let targetX=0;

let curve=0;
let curveTarget=0;

let roadSegments=[];

let spawnTimer=0;
let coinTimer=0;

let lastTime=performance.now();

let touchStartX=0;


/* =========================
   SCENE
========================= */

scene=new THREE.Scene();

scene.background=
new THREE.Color(0x76b9e8);

scene.fog=
new THREE.Fog(
  0x76b9e8,
  35,
  190
);


/* CAMERA */

camera=
new THREE.PerspectiveCamera(
  65,
  innerWidth/innerHeight,
  .1,
  600
);

camera.position.set(
  0,
  4.5,
  9
);


/* RENDERER */

renderer=
new THREE.WebGLRenderer({
  antialias:true
});

renderer.setSize(
  innerWidth,
  innerHeight
);

renderer.setPixelRatio(
  Math.min(devicePixelRatio,2)
);

document.body.appendChild(
  renderer.domElement
);


/* LIGHT */

scene.add(
  new THREE.HemisphereLight(
    0xffffff,
    0x555555,
    1.8
  )
);

const sun=
new THREE.DirectionalLight(
  0xffffff,
  1.3
);

sun.position.set(
  20,
  30,
  10
);

scene.add(sun);


/* =========================
   ROAD
========================= */

const grass=
new THREE.Mesh(
  new THREE.PlaneGeometry(
    150,
    600
  ),
  new THREE.MeshStandardMaterial({
    color:0x398a38
  })
);

grass.rotation.x=
-Math.PI/2;

grass.position.set(
  0,
  -.12,
  -260
);

scene.add(grass);


/* road pieces */

for(
  let i=0;
  i<80;
  i++
){

  const road=
  new THREE.Mesh(
    new THREE.PlaneGeometry(
      14,
      8
    ),
    new THREE.MeshStandardMaterial({
      color:0x303030
    })
  );

  road.rotation.x=
  -Math.PI/2;

  road.position.z=
  10-i*8;

  scene.add(road);

  roadSegments.push(road);


  /* lane markings */

  for(
    const x of [-2.3,2.3]
  ){

    const line=
    new THREE.Mesh(
      new THREE.BoxGeometry(
        .12,
        .04,
        3.5
      ),
      new THREE.MeshStandardMaterial({
        color:0xffffff
      })
    );

    line.position.set(
      x,
      .025,
      road.position.z
    );

    scene.add(line);

    roadSegments.push(line);
  }
}


/* =========================
   CAR CREATOR
========================= */

function createCar(color){

  const car=
  new THREE.Group();


  const body=
  new THREE.Mesh(
    new THREE.BoxGeometry(
      1.8,
      .65,
      3.4
    ),
    new THREE.MeshStandardMaterial({
      color:color,
      metalness:.25,
      roughness:.65
    })
  );

  body.position.y=.65;

  car.add(body);


  const hood=
  new THREE.Mesh(
    new THREE.BoxGeometry(
      1.55,
      .25,
      1.15
    ),
    new THREE.MeshStandardMaterial({
      color:color
    })
  );

  hood.position.set(
    0,
    .92,
    -1.05
  );

  car.add(hood);


  const cabin=
  new THREE.Mesh(
    new THREE.BoxGeometry(
      1.25,
      .65,
      1.5
    ),
    new THREE.MeshStandardMaterial({
      color:0x1b2635,
      metalness:.3,
      roughness:.3
    })
  );

  cabin.position.set(
    0,
    1.15,
    .05
  );

  car.add(cabin);


  /* wheels */

  for(
    const x of [-.95,.95]
  ){

    for(
      const z of [-1.05,1.05]
    ){

      const wheel=
      new THREE.Mesh(
        new THREE.CylinderGeometry(
          .38,
          .38,
          .24,
          16
        ),
        new THREE.MeshStandardMaterial({
          color:0x111111
        })
      );

      wheel.rotation.z=
      Math.PI/2;

      wheel.position.set(
        x,
        .38,
        z
      );

      car.add(wheel);
    }
  }


  /* headlights */

  for(
    const x of [-.58,.58]
  ){

    const light=
    new THREE.Mesh(
      new THREE.BoxGeometry(
        .28,
        .16,
        .08
      ),
      new THREE.MeshStandardMaterial({
        color:0xffffcc,
        emissive:0xffffaa,
        emissiveIntensity:1
      })
    );

    light.position.set(
      x,
      .72,
      -1.72
    );

    car.add(light);
  }

  return car;
}


/* PLAYER */

player=
createCar(0x1976d2);

player.position.set(
  0,
  0,
  5
);

scene.add(player);


/* =========================
   BUILDINGS
========================= */

function createBuilding(x,z){

  const h=
  5+Math.random()*12;

  const colors=[
    0x607d8b,
    0x795548,
    0x78909c,
    0x8d6e63,
    0x546e7a,
    0x9e9e9e
  ];

  const building=
  new THREE.Mesh(
    new THREE.BoxGeometry(
      4,
      h,
      6
    ),
    new THREE.MeshStandardMaterial({
      color:
        colors[
          Math.floor(
            Math.random()*colors.length
          )
        ]
    })
  );

  building.position.set(
    x,
    h/2,
    z
  );

  scene.add(building);

  scenery.push(building);


  /* windows */

  for(
    let y=2;
    y<h-1;
    y+=2.2
  ){

    for(
      let side=-1;
      side<=1;
      side+=2
    ){

      const window=
      new THREE.Mesh(
        new THREE.BoxGeometry(
          .55,
          .7,
          .06
        ),
        new THREE.MeshStandardMaterial({
          color:0x9edcff,
          emissive:0x224466,
          emissiveIntensity:.5
        })
      );

      window.position.set(
        x+side*2.02,
        y,
        z-1
      );

      scene.add(window);

      scenery.push(window);
    }
  }
}


/* =========================
   TREES
========================= */

function createTree(x,z){

  const tree=
  new THREE.Group();


  const trunk=
  new THREE.Mesh(
    new THREE.CylinderGeometry(
      .18,.25,2,8
    ),
    new THREE.MeshStandardMaterial({
      color:0x6d421f
    })
  );

  trunk.position.y=1;

  tree.add(trunk);


  const leaves=
  new THREE.Mesh(
    new THREE.SphereGeometry(
      1.15,
      12,
      12
    ),
    new THREE.MeshStandardMaterial({
      color:0x198c3a
    })
  );

  leaves.position.y=2.3;

  tree.add(leaves);


  tree.position.set(
    x,
    0,
    z
  );

  scene.add(tree);

  scenery.push(tree);
}


/* scenery */

for(
  let z=0;
  z>-500;
  z-=14
){

  if(
    Math.random()<.65
  ){

    createBuilding(
      -10-Math.random()*3,
      z
    );
  }

  if(
    Math.random()<.65
  ){

    createBuilding(
      10+Math.random()*3,
      z-5
    );
  }


  createTree(
    -7-Math.random()*2,
    z-3
  );

  createTree(
    7+Math.random()*2,
    z-8
  );
}


/* =========================
   STREET LIGHTS
========================= */

function createLamp(x,z){

  const group=
  new THREE.Group();


  const pole=
  new THREE.Mesh(
    new THREE.CylinderGeometry(
      .06,.08,4,8
    ),
    new THREE.MeshStandardMaterial({
      color:0x333333
    })
  );

  pole.position.y=2;

  group.add(pole);


  const lamp=
  new THREE.Mesh(
    new THREE.SphereGeometry(
      .22,
      10,
      10
    ),
    new THREE.MeshStandardMaterial({
      color:0xffffaa,
      emissive:0xffff66,
      emissiveIntensity:1
    })
  );

  lamp.position.set(
    0,
    4,
    0
  );

  group.add(lamp);


  group.position.set(
    x,
    0,
    z
  );

  scene.add(group);

  scenery.push(group);
}


for(
  let z=0;
  z>-450;
  z-=20
){

  createLamp(-8,z);

  createLamp(8,z-10);
}


/* =========================
   TRAFFIC CAR
========================= */

function spawnTraffic(){

  const colors=[
    0xdd2222,
    0xffaa00,
    0xeeeeee,
    0x7e57c2,
    0x222222
  ];

  const car=
  createCar(
    colors[
      Math.floor(
        Math.random()*colors.length
      )
    ]
  );

  const lanes=[
    -3,
    0,
    3
  ];

  car.position.set(
    lanes[
      Math.floor(
        Math.random()*3
      )
    ],
    0,
    -120
  );

  scene.add(car);

  traffic.push(car);
}


/* =========================
   RACE OPPONENT
========================= */

function spawnOpponent(i){

  const colors=[
    0xff2222,
    0xffcc00,
    0x00aa66,
    0xffffff,
    0x8e44ad
  ];

  const car=
  createCar(
    colors[
      i%colors.length
    ]
  );

  const lanes=[
    -3,
    0,
    3
  ];

  car.position.set(
    lanes[i%3],
    0,
    -20-i*18
  );

  car.userData.raceSpeed=
    .35+
    Math.random()*.22;

  scene.add(car);

  opponents.push(car);
}


/* =========================
   COIN
========================= */

function spawnCoin(){

  const coin=
  new THREE.Mesh(
    new THREE.TorusGeometry(
      .4,
      .12,
      12,
      24
    ),
    new THREE.MeshStandardMaterial({
      color:0xffd700,
      metalness:.8,
      roughness:.2,
      emissive:0x4a3500
    })
  );

  const lanes=[
    -3,
    0,
    3
  ];

  coin.position.set(
    lanes[
      Math.floor(
        Math.random()*3
      )
    ],
    1.1,
    -100
  );

  scene.add(coin);

  coins.push(coin);
}


/* =========================
   COLLISION
========================= */

function collision(a,b){

  const boxA=
  new THREE.Box3()
  .setFromObject(a);

  const boxB=
  new THREE.Box3()
  .setFromObject(b);

  return boxA.intersectsBox(boxB);
}


function checkCollisions(){

  for(
    const car of traffic
  ){

    if(
      collision(player,car)
    ){

      gameOver();
      return;
    }
  }


  for(
    const car of opponents
  ){

    if(
      collision(player,car)
    ){

      gameOver();
      return;
    }
  }


  for(
    let i=coins.length-1;
    i>=0;
    i--
  ){

    if(
      collision(player,coins[i])
    ){

      score+=25;

      scene.remove(
        coins[i]
      );

      coins.splice(i,1);
    }
  }
}


/* =========================
   GAME OVER
========================= */

function gameOver(){

  running=false;

  const finalScore=
  Math.floor(score);


  if(
    finalScore>best
  ){

    best=finalScore;

    localStorage.setItem(
      "streetRacerBest",
      best
    );
  }


  document.getElementById(
    "touch"
  ).style.display="none";


  document.getElementById(
    "menu"
  ).style.display="flex";


  document.querySelector(
    "#menu h1"
  ).innerText=
    "💥 GAME OVER";


  document.querySelector(
    "#menu p"
  ).innerHTML=
    "Score: "+
    finalScore+
    "<br>Best: "+
    best;


  document.getElementById(
    "raceBtn"
  ).innerText=
    "🏁 RACE AGAIN";


  document.getElementById(
    "freeBtn"
  ).innerText=
    "🚗 FREE DRIVE";
}


/* =========================
   START GAME
========================= */

function startGame(gameMode){

  mode=gameMode;

  running=true;

  score=0;

  distance=0;

  speed=baseSpeed;

  lane=0;

  targetX=0;

  curve=0;

  curveTarget=0;


  traffic.forEach(
    x=>scene.remove(x)
  );

  opponents.forEach(
    x=>scene.remove(x)
  );

  coins.forEach(
    x=>scene.remove(x)
  );


  traffic=[];
  opponents=[];
  coins=[];


  player.position.set(
    0,
    0,
    5
  );


  if(
    mode==="race"
  ){

    for(
      let i=0;
      i<5;
      i++
    ){

      spawnOpponent(i);
    }

  }


  document.getElementById(
    "menu"
  ).style.display="none";


  document.getElementById(
    "touch"
  ).style.display="flex";
}


/* =========================
   STEERING
========================= */

function steer(direction){

  lane+=direction;

  lane=
  Math.max(
    -1,
    Math.min(
      1,
      lane
    )
  );

  targetX=
    lane*3;
}


document.getElementById(
  "left"
).onclick=
()=>steer(-1);

document.getElementById(
  "right"
).onclick=
()=>steer(1);


/* keyboard */

document.addEventListener(
  "keydown",
  e=>{

    if(
      e.key==="ArrowLeft"
    )
      steer(-1);

    if(
      e.key==="ArrowRight"
    )
      steer(1);

    if(
      e.code==="Space"
    )
      nitro=true;
  }
);


document.addEventListener(
  "keyup",
  e=>{

    if(
      e.code==="Space"
    )
      nitro=false;
  }
);


/* =========================
   TOUCH SWIPE
========================= */

renderer.domElement.addEventListener(
  "touchstart",
  e=>{

    touchStartX=
      e.touches[0].clientX;
  },
  {passive:true}
);


renderer.domElement.addEventListener(
  "touchend",
  e=>{

    const endX=
      e.changedTouches[0].clientX;

    const dx=
      endX-touchStartX;


    if(
      Math.abs(dx)>35
    ){

      if(dx>0)
        steer(1);
      else
        steer(-1);
    }
  },
  {passive:true}
);


/* NITRO */

const nitroButton=
document.getElementById("nitro");

nitroButton.addEventListener(
  "touchstart",
  e=>{
    e.preventDefault();
    nitro=true;
  },
  {passive:false}
);

nitroButton.addEventListener(
  "touchend",
  e=>{
    e.preventDefault();
    nitro=false;
  },
  {passive:false}
);


/* =========================
   ROAD CURVES
========================= */

function updateRoad(dt){

  curveTarget=
    Math.sin(
      distance*.002
    )*.55+
    Math.sin(
      distance*.0008
    )*.35;

  curve +=
    (
      curveTarget-
      curve
    )*.02;


  /* move road/scenery sideways
     to create curved-road feeling */

  for(
    const obj of roadSegments
  ){

    const z=
      obj.position.z;

    obj.position.x=
      Math.sin(
        (z+distance)*.012
      )*
      curve*5;
  }
}


/* =========================
   UPDATE TRAFFIC
========================= */

function updateTraffic(dt){

  for(
    let i=traffic.length-1;
    i>=0;
    i--
  ){

    const car=
    traffic[i];

    car.position.z+=
      speed;

    if(
      car.position.z>25
    ){

      scene.remove(car);

      traffic.splice(i,1);
    }
  }


  for(
    let i=opponents.length-1;
    i>=0;
    i--
  ){

    const car=
    opponents[i];

    car.position.z+=
      speed-
      car.userData.raceSpeed;


    if(
      car.position.z>25
    ){

      car.position.z=
        -150-
        Math.random()*50;
    }
  }
}


/* =========================
   UPDATE COINS
========================= */

function updateCoins(){

  for(
    let i=coins.length-1;
    i>=0;
    i--
  ){

    const coin=
    coins[i];

    coin.position.z+=
      speed;

    coin.rotation.y+=.09;

    if(
      coin.position.z>25
    ){

      scene.remove(coin);

      coins.splice(i,1);
    }
  }
}


/* =========================
   GAME LOOP
========================= */

function animate(time){

  requestAnimationFrame(
    animate
  );


  const dt=
    Math.min(
      (time-lastTime)/1000,
      .05
    );

  lastTime=time;


  if(running){

    distance+=
      speed*100;


    /* speed gradually increases */

    speed+=
      dt*.012;


    if(nitro)
      speed+=
        dt*.25;


    /* spawn */

    spawnTimer+=dt;

    coinTimer+=dt;


    if(
      spawnTimer>
      Math.max(
        .65,
        1.4-
        speed*.25
      )
    ){

      spawnTimer=0;

      spawnTraffic();
    }


    if(
      coinTimer>.75
    ){

      coinTimer=0;

      spawnCoin();
    }


    /* steering */

    player.position.x +=
      (
        targetX-
        player.position.x
      )*.13;


    /* keep car on road */

    player.position.x=
      Math.max(
        -5.1,
        Math.min(
          5.1,
          player.position.x
        )
      );


    /* slight car tilt */

    player.rotation.z=
      (
        targetX-
        player.position.x
      )*.06;


    updateRoad(dt);

    updateTraffic(dt);

    updateCoins();

    checkCollisions();


    /* camera */

    camera.position.x +=
      (
        player.position.x-
        camera.position.x
      )*.08;


    camera.position.y=
      4.5;


    camera.position.z=
      9;


    camera.lookAt(
      player.position.x,
      .8,
      -18
    );


    /* score */

    score+=
      dt*
      (10+
      speed*5);


    document.getElementById(
      "score"
    ).innerText=
      Math.floor(score);


    document.getElementById(
      "speed"
    ).innerText=
      Math.floor(speed*100);


    document.getElementById(
      "best"
    ).innerText=
      best;


    if(
      mode==="race"
    ){

      document.getElementById(
        "position"
      ).innerText=
        "1/6";
    }
    else{

      document.getElementById(
        "position"
      ).innerText=
        "FREE";
    }
  }


  renderer.render(
    scene,
    camera
  );
}


/* =========================
   BUTTONS
========================= */

document.getElementById(
  "raceBtn"
).onclick=
()=>startGame("race");


document.getElementById(
  "freeBtn"
).onclick=
()=>startGame("free");


/* =========================
   RESIZE
========================= */

addEventListener(
  "resize",
  ()=>{

    camera.aspect=
      innerWidth/
      innerHeight;

    camera.updateProjectionMatrix();

    renderer.setSize(
      innerWidth,
      innerHeight
    );
  }
);


/* START */

requestAnimationFrame(
  animate
);

</script>

</body>
</html>
