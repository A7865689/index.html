# index.html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1.0,user-scalable=no">
<title>Coin Catcher</title>

<link rel="manifest" href="manifest.json">

<style>
*{box-sizing:border-box}

html,body{
  margin:0;
  width:100%;
  height:100%;
  overflow:hidden;
  touch-action:none;
  font-family:Arial,sans-serif;
  background:#10182b;
}

canvas{
  display:block;
  width:100%;
  height:100%;
}

#hud{
  position:fixed;
  top:12px;
  left:12px;
  right:12px;
  z-index:5;
  display:flex;
  justify-content:space-between;
  color:white;
  font-size:20px;
  font-weight:bold;
  text-shadow:2px 2px 4px #000;
}

#menu{
  position:fixed;
  inset:0;
  z-index:10;
  display:flex;
  flex-direction:column;
  align-items:center;
  justify-content:center;
  background:rgba(0,0,0,.75);
  color:white;
  text-align:center;
}

#menu h1{
  font-size:38px;
}

#play{
  border:0;
  border-radius:14px;
  padding:15px 45px;
  font-size:22px;
  font-weight:bold;
  background:#16c765;
  color:white;
}
</style>
</head>

<body>

<canvas id="game"></canvas>

<div id="hud">
  <span id="score">🪙 0</span>
  <span id="lives">❤️❤️❤️</span>
  <span id="best">🏆 0</span>
</div>

<div id="menu">
  <h1>🪙 COIN CATCHER</h1>
  <p>Finger se basket ko left aur right move karo.<br>
  Coins pakro, bombs se bacho!</p>
  <button id="play">PLAY</button>
</div>

<script>

const canvas=document.getElementById("game");
const ctx=canvas.getContext("2d");

let W,H;

function resize(){
  W=innerWidth;
  H=innerHeight;
  canvas.width=W;
  canvas.height=H;
}

resize();
addEventListener("resize",resize);

let playing=false;
let score=0;
let lives=3;

let best=Number(
  localStorage.getItem("coinCatcherBest")||0
);

let basket={
  x:0,
  y:0,
  w:120,
  h:45
};

let objects=[];
let spawnTimer=0;
let last=0;
let speed=220;


/* START */

function startGame(){

  playing=true;
  score=0;
  lives=3;
  objects=[];
  spawnTimer=0;
  speed=220;

  basket.x=W/2;
  basket.y=H-70;

  document.getElementById("menu").style.display="none";

  updateHUD();
}


/* HUD */

function updateHUD(){

  document.getElementById("score").textContent=
    "🪙 "+score;

  document.getElementById("lives").textContent=
    "❤️".repeat(lives);

  document.getElementById("best").textContent=
    "🏆 "+best;
}


/* OBJECT */

function spawn(){

  const isBomb=Math.random()<0.25;

  objects.push({
    x:30+Math.random()*(W-60),
    y:-30,
    r:18,
    bomb:isBomb
  });
}


/* BACKGROUND */

function background(){

  const g=ctx.createLinearGradient(
    0,0,0,H
  );

  g.addColorStop(0,"#152f59");
  g.addColorStop(1,"#07101d");

  ctx.fillStyle=g;
  ctx.fillRect(0,0,W,H);

  /* stars */

  ctx.fillStyle="white";

  for(let i=0;i<45;i++){

    const x=(i*83)%W;
    const y=(i*47)%(H*0.65);

    ctx.fillRect(x,y,2,2);
  }
}


/* COIN */

function drawCoin(o){

  ctx.beginPath();

  ctx.arc(
    o.x,o.y,o.r,
    0,Math.PI*2
  );

  ctx.fillStyle="#FFD21A";
  ctx.fill();

  ctx.strokeStyle="#FFF3A0";
  ctx.lineWidth=3;
  ctx.stroke();

  ctx.fillStyle="#9B7000";
  ctx.font="bold 18px Arial";
  ctx.textAlign="center";
  ctx.textBaseline="middle";

  ctx.fillText("$",o.x,o.y);
}


/* BOMB */

function drawBomb(o){

  ctx.beginPath();

  ctx.arc(
    o.x,o.y,o.r,
    0,Math.PI*2
  );

  ctx.fillStyle="#222";
  ctx.fill();

  ctx.strokeStyle="#777";
  ctx.lineWidth=3;
  ctx.stroke();

  ctx.strokeStyle="#ff4422";
  ctx.lineWidth=4;

  ctx.beginPath();

  ctx.moveTo(
    o.x+8,o.y-15
  );

  ctx.lineTo(
    o.x+18,o.y-28
  );

  ctx.stroke();

  ctx.fillStyle="#ff321d";

  ctx.beginPath();

  ctx.arc(
    o.x+18,
    o.y-30,
    5,
    0,
    Math.PI*2
  );

  ctx.fill();
}


/* BASKET */

function drawBasket(){

  const x=basket.x;
  const y=basket.y;

  ctx.fillStyle="#9b5a24";

  ctx.beginPath();

  ctx.roundRect(
    x-basket.w/2,
    y,
    basket.w,
    basket.h,
    12
  );

  ctx.fill();

  ctx.strokeStyle="#e3a15c";
  ctx.lineWidth=6;
  ctx.stroke();

  ctx.strokeStyle="#9b5a24";
  ctx.lineWidth=7;

  ctx.beginPath();

  ctx.arc(
    x,
    y+5,
    38,
    Math.PI,
    0
  );

  ctx.stroke();
}


/* COLLISION */

function hit(o){

  return(
    o.x>basket.x-basket.w/2 &&
    o.x<basket.x+basket.w/2 &&
    o.y+o.r>basket.y &&
    o.y-o.r<basket.y+basket.h
  );
}


/* GAME OVER */

function gameOver(){

  playing=false;

  if(score>best){

    best=score;

    localStorage.setItem(
      "coinCatcherBest",
      best
    );
  }

  document.getElementById("menu").style.display="flex";

  document.querySelector("#menu h1").textContent=
    "💥 GAME OVER";

  document.querySelector("#menu p").innerHTML=
    "Score: "+score+
    "<br>Best: "+best;

  document.getElementById("play").textContent=
    "PLAY AGAIN";
}


/* LOOP */

function loop(time){

  requestAnimationFrame(loop);

  const dt=Math.min(
    (time-last)/1000,
    0.05
  );

  last=time;

  background();

  if(!playing){
    return;
  }

  spawnTimer+=dt;

  if(spawnTimer>0.65){

    spawnTimer=0;
    spawn();
  }

  speed+=dt*5;

  for(let i=objects.length-1;i>=0;i--){

    const o=objects[i];

    o.y+=speed*dt;

    if(hit(o)){

      if(o.bomb){

        lives--;

      }else{

        score++;
      }

      objects.splice(i,1);

      updateHUD();

      if(lives<=0){
        gameOver();
      }

      continue;
    }

    if(o.y>H+50){
      objects.splice(i,1);
    }
  }

  for(const o of objects){

    if(o.bomb)
      drawBomb(o);
    else
      drawCoin(o);
  }

  drawBasket();

  updateHUD();
}


/* TOUCH */

canvas.addEventListener(
  "touchmove",
  function(e){

    if(!playing)return;

    e.preventDefault();

    basket.x=
      e.touches[0].clientX;

    basket.x=Math.max(
      basket.w/2,
      Math.min(
        W-basket.w/2,
        basket.x
      )
    );

  },
  {passive:false}
);


/* MOUSE */

canvas.addEventListener(
  "mousemove",
  function(e){

    if(!playing)return;

    basket.x=e.clientX;
  }
);


/* PLAY */

document.getElementById("play")
.addEventListener(
  "click",
  startGame
);


/* SERVICE WORKER */

if("serviceWorker" in navigator){

  window.addEventListener(
    "load",
    ()=>{
      navigator.serviceWorker.register(
        "service-worker.js"
      );
    }
  );
}


requestAnimationFrame(loop);

</script>

</body>
</html>
