
<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
<title>Football Heads Champions</title>
<style>
*{box-sizing:border-box;touch-action:none}
body{margin:0;background:#101827;color:white;font-family:Arial;overflow:hidden}
.hidden{display:none!important}
.screen{width:100%;height:100dvh;display:flex;flex-direction:column;align-items:center;justify-content:center;text-align:center;padding:15px}
h1{font-size:clamp(30px,7vw,52px);color:#ffda35;text-shadow:3px 4px #111;margin:8px}
h2{color:white}
button{border:0;border-radius:12px;padding:13px 20px;margin:6px;font-size:16px;font-weight:bold;background:#ffda35;color:#172033}
button:active{transform:scale(.94)}
.blue{background:#2788ff;color:white}
.red{background:#e63946;color:white}
#selectionArea{display:flex;flex-wrap:wrap;justify-content:center;max-width:500px}
.character{background:#263a55;border:3px solid transparent;border-radius:13px;padding:12px;margin:5px;min-width:90px}
.character.selected{border-color:#ffda35}
#game{display:none;width:100vw;height:100dvh;position:relative;overflow:hidden;background:#172b40}
canvas{width:100%;height:100%;display:block}
#hud{position:absolute;top:8px;left:0;width:100%;display:flex;justify-content:center;gap:25px;font-size:25px;font-weight:bold;text-shadow:2px 2px 5px black;pointer-events:none}
#gameButtons{position:absolute;right:5px;top:48px}
#gameButtons button{font-size:12px;padding:9px}
#controls{position:absolute;bottom:12px;left:0;width:100%;display:flex;justify-content:space-between;align-items:end;padding:0 8px;pointer-events:none}
.controlGroup{display:flex;gap:3px;pointer-events:auto}
.control{width:clamp(42px,10vw,60px);height:clamp(42px,10vw,60px);border-radius:50%;padding:0;margin:2px;background:#ffffffd9;color:#111;font-size:17px}
.action{background:#ffda35;width:clamp(50px,12vw,68px);height:clamp(50px,12vw,68px);font-size:12px}
#message{position:absolute;top:35%;width:100%;text-align:center;font-size:clamp(30px,8vw,55px);font-weight:bold;color:#ffda35;text-shadow:3px 3px 5px black;pointer-events:none}
#result{font-size:27px;color:#ffda35}
</style>
</head>
<body>

<div id="mainScreen" class="screen">
<h1>FOOTBALL HEADS</h1>
<h2>CHAMPIONS</h2>
<button onclick="chooseMode('bot')">1 VS BOT</button>
<button class="blue" onclick="chooseMode('local')">1 VS 1</button>
</div>

<div id="deviceScreen" class="screen hidden">
<h1>ELIGE TU DISPOSITIVO</h1>
<button onclick="chooseDevice('mobile')">CELULAR</button>
<button class="blue" onclick="chooseDevice('pc')">PC</button>
</div>

<div id="selectScreen" class="screen hidden">
<h1>ELIGE TU JUGADOR</h1>
<div id="selectionArea"></div>
<p id="selectInfo">Selecciona tu personaje</p>
<button onclick="startGame()">JUGAR</button>
<button class="red" onclick="backToMenu()">VOLVER</button>
</div>

<div id="game">
<canvas id="field"></canvas>
<div id="hud">
<span id="score">0 - 0</span>
<span id="clock">90</span>
</div>
<div id="gameButtons">
<button onclick="backToMenu()">MENÚ</button>
</div>
<div id="message"></div>
<div id="controls">
<div class="controlGroup">
<button class="control" data-key="left">◀</button>
<button class="control" data-key="right">▶</button>
<button class="control action" id="jumpBtn">JUMP</button>
</div>
<div class="controlGroup">
<button class="control action" id="passBtn">PASE</button>
<button class="control action" id="shootBtn">DISPARO</button>
</div>
</div>
</div>

<div id="endScreen" class="screen hidden">
<h1>PARTIDO TERMINADO</h1>
<p id="result"></p>
<button onclick="restartGame()">REVANCHA</button>
<button class="blue" onclick="backToMenu()">MENÚ</button>
</div>

<script>
(()=>{
"use strict";

const $=id=>document.getElementById(id);
const canvas=$("field");
const ctx=canvas.getContext("2d");

const W=1000,H=560,FLOOR=470;
const names=["MESSI","CRISTIANO","MBAPPÉ","NEYMAR","HAALAND"];
const colors=["#168bff","#ed3c42"];

let mode="bot",device="mobile";
let chosen=[0,1],selection=0;
let players=[],ball={};
let score=[0,0],time=90;
let running=false,lastTime=0,animation=0,timerInterval=null;
let messageTimeout=null,keys={};
let viewScale=1,offsetX=0,offsetY=0;
let botJumpTimer=0;
let botDirection=-1;

function resize(){
const rect=canvas.getBoundingClientRect();
const dpr=Math.min(window.devicePixelRatio||1,2);
canvas.width=Math.max(1,Math.floor(rect.width*dpr));
canvas.height=Math.max(1,Math.floor(rect.height*dpr));
viewScale=Math.min(rect.width/W,rect.height/H);
offsetX=(rect.width-W*viewScale)/2;
offsetY=(rect.height-H*viewScale)/2;
ctx.setTransform(dpr*viewScale,0,0,dpr*viewScale,dpr*offsetX,dpr*offsetY);
}
window.addEventListener("resize",resize);

function showScreen(id){
const screens=["mainScreen","deviceScreen","selectScreen","game","endScreen"];
screens.forEach(x=>{
$(x).classList.add("hidden");
$(x).style.display="none";
});
const target=$(id);
target.classList.remove("hidden");
target.style.display=id==="game"?"block":"flex";
if(id==="game")resize();
}

window.chooseMode=function(m){
mode=m;
showScreen("deviceScreen");
};

window.chooseDevice=function(d){
device=d;
selection=0;
chosen=[0,1];
buildSelection();
showScreen("selectScreen");
};

function buildSelection(){
const area=$("selectionArea");
area.innerHTML="";
names.forEach((name,i)=>{
const div=document.createElement("div");
div.className="character";
if(chosen.includes(i))div.classList.add("selected");
div.innerHTML="<strong>"+name+"</strong><br><br>⚽";
div.onclick=()=>{
if(mode==="bot"){
chosen[0]=i;
chosen[1]=(i+1)%names.length;
}else{
chosen[selection]=i;
selection=selection===0?1:0;
}
buildSelection();
};
area.appendChild(div);
});
$("selectInfo").textContent=mode==="bot"?"Elige tu futbolista":"Jugador "+(selection+1)+": elige un futbolista";
}

function makePlayer(x,team,index){
return{
x,y:FLOOR,vx:0,vy:0,team,index,
name:names[chosen[index]],color:colors[team],
facing:team===0?1:-1,
jump:0,jumpV:0,speed:300,
cooldown:0,hasBall:false
};
}

function resetPositions(){
players=[makePlayer(230,0,0),makePlayer(770,1,1)];
ball={x:W/2,y:FLOOR-20,vx:0,vy:0,r:19,owner:-1,rotation:0};
botJumpTimer=0;
botDirection=-1;
}

window.startGame=function(){
score=[0,0];time=90;running=true;keys={};
showScreen("game");
resetPositions();
$("score").textContent="0 - 0";
$("clock").textContent="90";
$("message").textContent="";
clearInterval(timerInterval);
timerInterval=setInterval(()=>{
if(!running)return;
time--;
$("clock").textContent=time;
if(time<=0)finishGame();
},1000);
cancelAnimationFrame(animation);
lastTime=performance.now();
animation=requestAnimationFrame(loop);
};

/* CANCHA */
function drawField(){
const sky=ctx.createLinearGradient(0,0,0,FLOOR);
sky.addColorStop(0,"#60b8ed");
sky.addColorStop(1,"#d4edff");
ctx.fillStyle=sky;ctx.fillRect(0,0,W,H);

ctx.fillStyle="#ffffffaa";
for(let i=0;i<6;i++){
let x=(i*210+60)%W;
ctx.beginPath();
ctx.arc(x,75,22,0,Math.PI*2);
ctx.arc(x+25,65,28,0,Math.PI*2);
ctx.arc(x+53,77,20,0,Math.PI*2);
ctx.fill();
}

ctx.fillStyle="#344d69";ctx.fillRect(0,150,W,150);
for(let row=0;row<4;row++){
ctx.fillStyle=row%2?"#526b83":"#405a73";
ctx.fillRect(0,155+row*34,W,27);
}

const crowd=["#ffdb3b","#f34b4b","#ffffff","#31a5ff","#41c777"];
for(let i=0;i<95;i++){
ctx.fillStyle=crowd[i%crowd.length];
ctx.beginPath();ctx.arc((i*47)%W,170+(i%4)*28,5,0,Math.PI*2);ctx.fill();
}

ctx.fillStyle="#299d45";ctx.fillRect(0,300,W,H-300);
for(let i=0;i<10;i++){
ctx.fillStyle=i%2?"#26913e":"#32a94d";
ctx.fillRect(i*100,300,100,H-300);
}

ctx.strokeStyle="white";ctx.lineWidth=4;
ctx.strokeRect(8,302,W-16,FLOOR-302);
ctx.beginPath();ctx.moveTo(W/2,302);ctx.lineTo(W/2,FLOOR);ctx.stroke();
ctx.beginPath();ctx.arc(W/2,386,65,0,Math.PI*2);ctx.stroke();
ctx.strokeRect(8,335,115,115);ctx.strokeRect(W-123,335,115,115);
ctx.fillStyle="#eeeeee";
ctx.fillRect(0,355,22,115);ctx.fillRect(W-22,355,22,115);
ctx.fillStyle="#ffffff55";
ctx.fillRect(22,355,35,115);ctx.fillRect(W-57,355,35,115);
ctx.strokeStyle="white";ctx.lineWidth=5;
ctx.strokeRect(0,355,55,115);ctx.strokeRect(W-55,355,55,115);
ctx.fillStyle="#176d32";ctx.fillRect(0,FLOOR,W,H-FLOOR);
ctx.fillStyle="#ffffff33";ctx.fillRect(0,FLOOR,W,5);
}

/* PERSONAJES */
function drawPlayer(p){
ctx.save();
const lift=p.jump;
const headY=FLOOR-75-lift;
const bodyY=FLOOR-lift;
const walk=Math.sin(performance.now()/75)*5;

ctx.fillStyle="#0004";
ctx.beginPath();
ctx.ellipse(p.x,FLOOR-2,35*(1-lift/1000),9,0,0,Math.PI*2);ctx.fill();

ctx.strokeStyle="#222";ctx.lineWidth=10;ctx.lineCap="round";
ctx.beginPath();
ctx.moveTo(p.x-10,bodyY-28);ctx.lineTo(p.x-12+walk,bodyY-5);
ctx.moveTo(p.x+10,bodyY-28);ctx.lineTo(p.x+12-walk,bodyY-5);ctx.stroke();

ctx.fillStyle="#111";
ctx.beginPath();
ctx.ellipse(p.x-17+walk,bodyY-5,12,6,0,0,Math.PI*2);
ctx.ellipse(p.x+17-walk,bodyY-5,12,6,0,0,Math.PI*2);ctx.fill();

ctx.fillStyle=p.color;ctx.fillRect(p.x-19,bodyY-58,38,32);
ctx.fillStyle="#ffffff55";ctx.fillRect(p.x-8,bodyY-55,16,26);

ctx.strokeStyle=p.color;ctx.lineWidth=10;
ctx.beginPath();
ctx.moveTo(p.x-17,bodyY-51);ctx.lineTo(p.x-28,bodyY-32);
ctx.moveTo(p.x+17,bodyY-51);ctx.lineTo(p.x+28,bodyY-32);ctx.stroke();

ctx.fillStyle="#e8b18c";ctx.fillRect(p.x-9,bodyY-68,18,14);
const skin=["#e9b58e","#d9a078","#f0c09b","#dca47e","#e5b08a"][p.index];
ctx.fillStyle=skin;ctx.beginPath();ctx.arc(p.x,headY,39,0,Math.PI*2);ctx.fill();
ctx.strokeStyle="#593b2d";ctx.lineWidth=2;ctx.stroke();

ctx.fillStyle=["#35251d","#302018","#1c1714","#52301e","#34251e"][p.index];
ctx.beginPath();ctx.arc(p.x,headY-9,39,Math.PI,Math.PI*2);ctx.fill();
ctx.fillRect(p.x-38,headY-10,8,20);ctx.fillRect(p.x+30,headY-10,8,18);

ctx.fillStyle="white";ctx.beginPath();
ctx.ellipse(p.x-14,headY-3,9,11,0,0,Math.PI*2);
ctx.ellipse(p.x+14,headY-3,9,11,0,0,Math.PI*2);ctx.fill();

ctx.fillStyle="#171717";ctx.beginPath();
ctx.arc(p.x-12,headY-2,4,0,Math.PI*2);
ctx.arc(p.x+16,headY-2,4,0,Math.PI*2);ctx.fill();

ctx.strokeStyle="#39271e";ctx.lineWidth=3;ctx.beginPath();
ctx.moveTo(p.x-23,headY-17);ctx.lineTo(p.x-7,headY-18);
ctx.moveTo(p.x+7,headY-18);ctx.lineTo(p.x+23,headY-17);ctx.stroke();

ctx.strokeStyle="#8c4137";ctx.lineWidth=2;ctx.beginPath();
ctx.arc(p.x,headY+12,10,.15,Math.PI-.15);ctx.stroke();

ctx.fillStyle="white";ctx.font="bold 13px Arial";ctx.textAlign="center";
ctx.fillText(p.name,p.x,headY-49);
ctx.restore();
}

/* PELOTA */
function drawBall(){
ctx.save();ctx.translate(ball.x,ball.y);ctx.rotate(ball.rotation);
ctx.fillStyle="white";ctx.beginPath();ctx.arc(0,0,ball.r,0,Math.PI*2);ctx.fill();
ctx.strokeStyle="#111";ctx.lineWidth=2;ctx.stroke();
ctx.fillStyle="#202020";ctx.beginPath();ctx.arc(0,0,7,0,Math.PI*2);ctx.fill();
ctx.beginPath();ctx.moveTo(-15,-7);ctx.lineTo(-7,-15);ctx.lineTo(0,-7);ctx.lineTo(-7,0);ctx.closePath();ctx.fill();
ctx.restore();
}
function draw(){drawField();players.forEach(drawPlayer);drawBall();}

/* MOVIMIENTO */
function movePlayer(p,dir,dt){
if(dir!==0)p.facing=Math.sign(dir);
p.vx=dir*p.speed;
p.x+=p.vx*dt;
p.x=Math.max(65,Math.min(W-65,p.x));
if(p.cooldown>0)p.cooldown-=dt;
}
function jumpPlayer(p){if(p.jump===0)p.jumpV=650;}
function updateJump(p,dt){
p.jump+=p.jumpV*dt;p.jumpV-=1550*dt;
if(p.jump<0){p.jump=0;p.jumpV=0;}
}
function controlPlayer(dt){
const p=players[0];let dir=0;
if(keys.left)dir--;
if(keys.right)dir++;
movePlayer(p,dir,dt);
}

/* IA DEL BOT */
function botAI(dt){
const p=players[1],rival=players[0];

if(mode==="local"){
let dir=0;
if(keys.left2)dir--;
if(keys.right2)dir++;
movePlayer(p,dir,dt);
return;
}

botJumpTimer-=dt;

const ballNearFeet=
Math.abs(ball.x-p.x)<68 &&
ball.y>FLOOR-65-p.jump &&
ball.y<FLOOR+15-p.jump;

if(ball.owner===-1 && ballNearFeet){
ball.owner=1;
p.hasBall=true;
players[0].hasBall=false;
p.facing=-1;
}

if(ball.owner===1){
p.facing=-1;
botDirection=-1;
movePlayer(p,-1,dt);

const distanceToGoal=ball.x-18;
if(distanceToGoal<430 && p.cooldown<=0){
botShoot();
return;
}

const rivalBetween=
rival.x<ball.x-35 &&
rival.x>30 &&
Math.abs(rival.x-p.x)<150;

if(rivalBetween && p.jump===0 && botJumpTimer<=0){
jumpPlayer(p);
botJumpTimer=1;
}
return;
}

if(ball.owner===0){
const target=rival.x+65;
const difference=target-p.x;

if(Math.abs(difference)>45){
botDirection=difference>0?1:-1;
movePlayer(p,botDirection,dt);
}else{
movePlayer(p,0,dt);
}

if(Math.abs(rival.x-p.x)<115 &&
rival.jump>25 && p.jump===0 && botJumpTimer<=0){
jumpPlayer(p);
botJumpTimer=1;
}
return;
}

let targetX=ball.x;
if(Math.abs(ball.x-p.x)>100){
targetX=ball.x+ball.vx*.08;
}

targetX=Math.max(65,Math.min(W-65,targetX));
const difference=targetX-p.x;

if(Math.abs(difference)>55){
botDirection=difference>0?1:-1;
movePlayer(p,botDirection,dt);
}else{
movePlayer(p,0,dt);
}

if(Math.abs(ball.x-p.x)<115 &&
ball.y<FLOOR-65 &&
p.jump===0 && botJumpTimer<=0){
jumpPlayer(p);
botJumpTimer=.8;
}
}

/* DISPARO INTELIGENTE */
function botShoot(){
const p=players[1];
const rival=players[0];
if(ball.owner!==1 || p.cooldown>0)return;

const goalX=18;
const distance=ball.x-goalX;
if(distance>430)return;

let targetY=412;

if(rival.x<ball.x-40 && rival.x>goalX){
targetY=rival.jump>30?375:450;
}else{
targetY=Math.random()<.5?375:450;
}

if(distance<170)targetY=412;

const dx=goalX-ball.x;
const dy=targetY-ball.y;
const length=Math.hypot(dx,dy)||1;
const power=distance<190?780:900;

ball.owner=-1;
p.hasBall=false;
ball.vx=(dx/length)*power;
ball.vy=(dy/length)*power-70;
p.cooldown=.85;
}

/* PELOTA CON ARRASTRE LIBRE */
function attachBall(){
/* Ya no se pega la pelota al jugador.
   La pelota conserva su propia velocidad y física. */
return;
}

function checkPossession(){
if(ball.owner!==-1)return;

players.forEach(p=>{
if(ball.owner!==-1)return;

const headDist=Math.hypot(
ball.x-p.x,
ball.y-(FLOOR-75-p.jump)
);

const feetDist=Math.abs(ball.x-p.x);
const ballAtFeet=
feetDist<55 &&
ball.y>FLOOR-65-p.jump &&
ball.y<FLOOR+15-p.jump;

if(headDist<ball.r+30 || ballAtFeet){
ball.owner=p.index;
p.hasBall=true;
players.forEach(other=>{
if(other!==p)other.hasBall=false;
});
}
});
}

/* PASE Y DISPARO */
function kickBall(p,power){
if(!p || p.cooldown>0)return;

const distance=Math.hypot(
ball.x-p.x,
ball.y-(FLOOR-65-p.jump)
);

if(ball.owner!==p.index && distance>95)return;

ball.owner=-1;
p.hasBall=false;

// La pelota sale desde su posición real, no desde el cuerpo.
ball.vx=p.facing*power+p.vx*.45;
ball.vy=-300-p.jumpV*.15;
p.cooldown=.35;
}

/* FÍSICA Y ARRASTRE */
function updateBall(dt){

if(ball.owner>=0){
const owner=players[ball.owner];
const defender=players[1-ball.owner];

// El rival puede recuperar la pelota al acercarse.
if(Math.abs(owner.x-defender.x)<62 &&
owner.jump<65 && defender.jump<65 &&
defender.cooldown<=0){
ball.owner=defender.index;
owner.hasBall=false;
defender.hasBall=true;
defender.cooldown=.5;
}

// La pelota rueda delante del jugador.
// No se teletransporta ni queda pegada.
const targetX=owner.x+owner.facing*48;
const targetY=FLOOR-ball.r;

const difference=targetX-ball.x;

// Impulso suave para acompañar al jugador.
let acceleration=difference*13-ball.vx*3.2;
acceleration=Math.max(-650,Math.min(650,acceleration));

ball.vx+=acceleration*dt;

// La pelota también recibe el movimiento del jugador.
ball.vx+=owner.vx*.35*dt;

// Física vertical normal.
ball.vy+=1000*dt;

ball.x+=ball.vx*dt;
ball.y+=ball.vy*dt;
ball.rotation+=ball.vx*dt*.025;

// Rebote y rozamiento en el suelo.
if(ball.y+ball.r>FLOOR){
ball.y=FLOOR-ball.r;
ball.vy=-Math.abs(ball.vy)*.18;
if(Math.abs(ball.vy)<35)ball.vy=0;
ball.vx*=Math.pow(.975,dt*60);
}

// Limitar altura del campo.
if(ball.y-ball.r<300){
ball.y=300+ball.r;
ball.vy=Math.abs(ball.vy)*.65;
}

// Comprobar goles incluso mientras se conduce.
if(ball.x<25&&ball.y>355&&ball.y<470){
goal(1);
return;
}
if(ball.x>W-25&&ball.y>355&&ball.y<470){
goal(0);
return;
}

ball.x=Math.max(8,Math.min(W-8,ball.x));
return;
}

// PELOTA LIBRE
ball.x+=ball.vx*dt;
ball.y+=ball.vy*dt;
ball.vy+=1000*dt;
ball.rotation+=ball.vx*dt*.025;
ball.vx*=Math.pow(.994,dt*60);

if(ball.y+ball.r>FLOOR){
ball.y=FLOOR-ball.r;
ball.vy=-Math.abs(ball.vy)*.68;
if(Math.abs(ball.vy)<70)ball.vy=0;
ball.vx*=Math.pow(.975,dt*60);
}

if(ball.y-ball.r<300){
ball.y=300+ball.r;
ball.vy=Math.abs(ball.vy)*.65;
}

if(ball.x<8&&(ball.y<355||ball.y>470)){
ball.x=8;ball.vx=Math.abs(ball.vx)*.7;
}
if(ball.x>W-8&&(ball.y<355||ball.y>470)){
ball.x=W-8;ball.vx=-Math.abs(ball.vx)*.7;
}

if(ball.x<25&&ball.y>355&&ball.y<470){goal(1);return;}
if(ball.x>W-25&&ball.y>355&&ball.y<470){goal(0);return;}

// Choques con los jugadores.
players.forEach(p=>{
const hx=p.x,hy=FLOOR-75-p.jump;
const dx=ball.x-hx,dy=ball.y-hy;
const dist=Math.hypot(dx,dy);

if(dist<ball.r+38&&dist>0){
const nx=dx/dist,ny=dy/dist;
ball.x=hx+nx*(ball.r+39);
ball.y=hy+ny*(ball.r+39);
ball.vx=nx*430+p.vx*.5;
ball.vy=ny*430-100;
}
});

checkPossession();
}

/* GOLES */
function goal(team){
if(!running)return;
score[team]++;
$("score").textContent=score[0]+" - "+score[1];
$("message").textContent="¡GOOOOOOL!";
ball.owner=-1;
players.forEach(p=>p.hasBall=false);
ball.x=W/2;ball.y=FLOOR-110;ball.vx=0;ball.vy=0;
players[0].x=230;players[1].x=770;
players.forEach(p=>{p.jump=0;p.jumpV=0;p.cooldown=0;});
botJumpTimer=0;
botDirection=-1;
clearTimeout(messageTimeout);
messageTimeout=setTimeout(()=>{$("message").textContent="";},1400);
}

function finishGame(){
if(!running)return;
running=false;
clearInterval(timerInterval);
cancelAnimationFrame(animation);
let result="¡EMPATE!";
if(score[0]>score[1])result="¡GANASTE!";
if(score[1]>score[0])result=mode==="bot"?"GANÓ EL BOT":"GANÓ EL JUGADOR 2";
$("result").textContent=result+"  "+score[0]+" - "+score[1];
showScreen("endScreen");
}

function loop(now){
if(!running)return;
const dt=Math.min((now-lastTime)/1000,.035);
lastTime=now;
controlPlayer(dt);
botAI(dt);
players.forEach(p=>updateJump(p,dt));
updateBall(dt);
draw();
animation=requestAnimationFrame(loop);
}

/* MENÚ Y REVANCHA */
window.restartGame=function(){
clearInterval(timerInterval);
startGame();
};

window.backToMenu=function(){
running=false;
clearInterval(timerInterval);
cancelAnimationFrame(animation);
clearTimeout(messageTimeout);
keys={};
showScreen("mainScreen");
};

/* CONTROLES TÁCTILES */
document.querySelectorAll("[data-key]").forEach(button=>{
const key=button.dataset.key;
button.addEventListener("pointerdown",e=>{
e.preventDefault();
keys[key]=true;
});
["pointerup","pointercancel","pointerleave"].forEach(event=>{
button.addEventListener(event,e=>{
e.preventDefault();
keys[key]=false;
});
});
});

function actionButton(id,action){
$(id).addEventListener("pointerdown",e=>{
e.preventDefault();
if(!running)return;
action();
});
}
actionButton("jumpBtn",()=>jumpPlayer(players[0]));
actionButton("passBtn",()=>kickBall(players[0],470));
actionButton("shootBtn",()=>kickBall(players[0],900));

/* TECLADO */
document.addEventListener("keydown",e=>{
const k=e.key.toLowerCase();
if(["arrowleft","arrowright","arrowup"," "].includes(k))e.preventDefault();

if(k==="arrowleft"||k==="a")keys.left=true;
if(k==="arrowright"||k==="d")keys.right=true;
if(k==="e"&&!e.repeat&&players[0])kickBall(players[0],470);
if(k===" "&&!e.repeat&&players[0])kickBall(players[0],900);
if((k==="w"||k==="arrowup")&&!e.repeat&&players[0])jumpPlayer(players[0]);

if(k==="j")keys.left2=true;
if(k==="l")keys.right2=true;
if(k==="i"&&!e.repeat&&players[1])jumpPlayer(players[1]);
if(k==="u"&&!e.repeat&&players[1])kickBall(players[1],470);
if(k==="o"&&!e.repeat&&players[1])kickBall(players[1],900);
});

document.addEventListener("keyup",e=>{
const k=e.key.toLowerCase();
if(k==="arrowleft"||k==="a")keys.left=false;
if(k==="arrowright"||k==="d")keys.right=false;
if(k==="j")keys.left2=false;
if(k==="l")keys.right2=false;
});

window.addEventListener("blur",()=>keys={});

showScreen("mainScreen");

})();
</script>
</body>
</html>
