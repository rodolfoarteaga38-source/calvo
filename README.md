<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>El Calvo Supremo 5.0</title>

<style>
body{margin:0;background:black;overflow:hidden;font-family:Arial}
canvas{display:block;margin:auto;background:black}
#characterMenu{position:absolute;top:35%;left:50%;transform:translate(-50%,-50%);text-align:center;color:white;z-index:10;}
.character{width:60px;height:60px;border-radius:50%;display:inline-block;margin:10px;cursor:pointer;}
.diffBtn{padding:8px 15px;margin:5px;cursor:pointer;background:cyan;border:none;font-weight:bold;}
#musicControl{position:absolute;bottom:10px;left:10px;color:white;}
#endScreen{position:absolute;top:50%;left:50%;transform:translate(-50%,-50%);text-align:center;color:white;display:none;z-index:20;}
button{padding:12px 25px;font-size:16px;cursor:pointer;border:none;background:cyan;color:black;font-weight:bold;}
</style>
</head>
<body>

<canvas id="game"></canvas>

<div id="characterMenu">
<h1>ELIGE PERSONAJE</h1>
<img src="assets/player.png" width="80" onclick="selectCharacter('calvo')" style="cursor:pointer">
<br>
<div class="character" style="background:red" onclick="selectCharacter('red')"></div>
<div class="character" style="background:blue" onclick="selectCharacter('blue')"></div>
<div class="character" style="background:green" onclick="selectCharacter('green')"></div>
<div class="character" style="background:yellow" onclick="selectCharacter('yellow')"></div>

<h2>DIFICULTAD</h2>
<button class="diffBtn" onclick="setDifficulty('easy')">Fácil</button>
<button class="diffBtn" onclick="setDifficulty('medium')">Medio</button>
<button class="diffBtn" onclick="setDifficulty('hard')">Difícil</button>
</div>

<div id="musicControl">
Volumen:
<input type="range" min="0" max="1" step="0.01" value="0.5"
oninput="levelMusic.volume=this.value">
</div>

<div id="endScreen">
<h1 id="endText"></h1>
<button onclick="restartGame()">Intentar otra vez</button>
</div>

<script>

const canvas=document.getElementById("game");
const ctx=canvas.getContext("2d");

canvas.width=window.innerWidth;
canvas.height=window.innerHeight;

window.addEventListener("resize",()=>{
canvas.width=window.innerWidth;
canvas.height=window.innerHeight;
});

/* ASSETS */
const playerImg=new Image();
playerImg.src="assets/player.png";

const enemyImg=new Image();
enemyImg.src="assets/enemy.png";

const bossImg=new Image();
bossImg.src="assets/boss.png";

const levelMusic=new Audio("assets/level1.mp3");
levelMusic.loop=true;
levelMusic.volume=0.5;

/* VARIABLES */
let enemyMoveSpeed=2;
let enemyFireRate=0.02;
let enemyBulletSpeed=3;

let playerType="calvo";
let gameStarted=false;

let player;
let bullets=[];
let enemyBullets=[];
let enemies=[];
let missiles=[];
let bossBullets=[];
let score=0;

/* POWERUPS */
let powerups=[];
let shield=0;   // AHORA ACUMULATIVO
let damageMultiplier=1;
let rapidFire=false;

let keys={};
let shootCooldown=0;

/* BOSS */
let boss=null;
let bossActive=false;
let bossShootCooldown=0;

/* DIFICULTAD */
function setDifficulty(level){
if(level==="easy"){enemyMoveSpeed=1.5;}
if(level==="medium"){enemyMoveSpeed=2;}
if(level==="hard"){enemyMoveSpeed=3;}
}

/* INICIO */
function selectCharacter(type){
playerType=type;
document.getElementById("characterMenu").style.display="none";
gameStarted=true;
restartGame();
}

function restartGame(){
player={x:canvas.width/2,y:canvas.height-120,size:70,speed:8,hp:5};
bullets=[];enemyBullets=[];enemies=[];missiles=[];bossBullets=[];powerups=[];
score=0;boss=null;bossActive=false;
shield=0;
damageMultiplier=1;
rapidFire=false;
bossShootCooldown=0;
levelMusic.currentTime=0;
levelMusic.play().catch(()=>{});
}

/* CONTROLES */
document.addEventListener("keydown",e=>{
keys[e.key.toLowerCase()]=true;
if(e.key.toLowerCase()==="q")launchMissile();
});
document.addEventListener("keyup",e=>{
keys[e.key.toLowerCase()]=false;
});

canvas.addEventListener("mousemove",(e)=>{
if(!player)return;
player.x=e.clientX-player.size/2;
player.y=e.clientY-player.size/2;
});

/* SPAWN ENEMIGOS */
setInterval(()=>{
if(!gameStarted||bossActive)return;
enemies.push({
x:Math.random()*(canvas.width-60),
y:Math.random()*300,
size:60,
vx:(Math.random()-0.5)*enemyMoveSpeed,
vy:(Math.random()-0.5)*enemyMoveSpeed
});
},900);

/* POWERUPS MÁS FRECUENTES */
setInterval(()=>{
if(!gameStarted)return;

if(Math.random()<0.9){
let types=["shield","shield","damage","rapid"];

powerups.push({
x:Math.random()*(canvas.width-20),
y:Math.random()*(canvas.height/2),
type:types[Math.floor(Math.random()*types.length)]
});
}

},3000);

/* LOOP */
function update(){
ctx.clearRect(0,0,canvas.width,canvas.height);

if(!gameStarted){
requestAnimationFrame(update);
return;
}

if(keys["a"])player.x-=player.speed;
if(keys["d"])player.x+=player.speed;
if(keys["w"])player.y-=player.speed;
if(keys["s"])player.y+=player.speed;

if(keys[" "]&&shootCooldown<=0){
bullets.push({x:player.x+35,y:player.y});
shootCooldown= rapidFire ? 4 : 10;
}
if(shootCooldown>0)shootCooldown--;

drawPlayer();
updateBullets();
updateEnemies();
updateEnemyBullets();
updateMissiles();
updatePowerups();

/* ACTIVAR BOSS */
if(score>=50 && !bossActive){
spawnBoss();
}

updateBoss();
updateBossBullets();
drawHUD();

if(player.hp<=0){
showEnd("GAME OVER");
}

requestAnimationFrame(update);
}

/* POWERUPS */
function updatePowerups(){
powerups.forEach((p,i)=>{

if(p.type==="shield")ctx.fillStyle="cyan";
if(p.type==="damage")ctx.fillStyle="red";
if(p.type==="rapid")ctx.fillStyle="lime";

ctx.fillRect(p.x,p.y,20,20);

if(p.x<player.x+70 && p.x+20>player.x &&
p.y<player.y+70 && p.y+20>player.y){

if(p.type==="shield")shield++;  // AHORA SUMA ESCUDOS

if(p.type==="damage"){
damageMultiplier=2;
setTimeout(()=>damageMultiplier=1,8000);
}

if(p.type==="rapid"){
rapidFire=true;
setTimeout(()=>rapidFire=false,8000);
}

powerups.splice(i,1);
}

});
}

/* JUGADOR */
function drawPlayer(){

if(playerType==="calvo"){
ctx.drawImage(playerImg,player.x,player.y,player.size,player.size);
}else{
ctx.fillStyle=playerType;
ctx.beginPath();
ctx.arc(player.x+35,player.y+35,35,0,Math.PI*2);
ctx.fill();
}

/* ESCUDOS VISUALES ACUMULATIVOS */
for(let i=0;i<shield;i++){
ctx.strokeStyle="cyan";
ctx.lineWidth=3;
ctx.beginPath();
ctx.arc(player.x+35,player.y+35,45+(i*6),0,Math.PI*2);
ctx.stroke();
}

}

/* BALAS PLAYER */
function updateBullets(){
bullets.forEach((b,i)=>{
b.y-=10;
ctx.fillStyle="cyan";
ctx.fillRect(b.x,b.y,6,16);
if(b.y<0)bullets.splice(i,1);
});
}

/* ENEMIGOS */
function updateEnemies(){
if(bossActive)return;

enemies.forEach((e,ei)=>{
e.x+=e.vx;
e.y+=e.vy;

if(e.x<0||e.x>canvas.width-60)e.vx*=-1;
if(e.y<0||e.y>300)e.vy*=-1;

ctx.drawImage(enemyImg,e.x,e.y,e.size,e.size);

if(Math.random()<enemyFireRate){
for(let a=0;a<360;a+=60){
let rad=a*Math.PI/180;
enemyBullets.push({
x:e.x+30,
y:e.y+30,
dx:Math.cos(rad)*enemyBulletSpeed,
dy:Math.sin(rad)*enemyBulletSpeed
});
}
}

bullets.forEach((b,bi)=>{
if(b.x<e.x+60&&b.x+6>e.x&&b.y<e.y+60&&b.y+16>e.y){
enemies.splice(ei,1);
bullets.splice(bi,1);
score++;
}
});
});
}

/* BALAS ENEMIGAS */
function updateEnemyBullets(){
enemyBullets.forEach((b,i)=>{
b.x+=b.dx;
b.y+=b.dy;
ctx.fillStyle="orange";
ctx.fillRect(b.x,b.y,6,6);

if(b.x<player.x+70&&b.x+6>player.x&&
b.y<player.y+70&&b.y+6>player.y){

if(shield>0){shield--;}
else{player.hp--;}

enemyBullets.splice(i,1);
}
});
}

/* MISIL Q */
function launchMissile(){
if(enemies.length===0)return;
let target=enemies[0];
missiles.push({x:player.x+35,y:player.y,target});
}

function updateMissiles(){
missiles.forEach((m,i)=>{
if(!m.target){missiles.splice(i,1);return;}
let dx=m.target.x-m.x;
let dy=m.target.y-m.y;
let dist=Math.sqrt(dx*dx+dy*dy);
m.x+=dx/dist*6;
m.y+=dy/dist*6;
ctx.fillStyle="red";
ctx.fillRect(m.x,m.y,8,8);
if(dist<20){
let index=enemies.indexOf(m.target);
if(index>-1){
enemies.splice(index,1);
score++;
}
missiles.splice(i,1);
}
});
}

/* BOSS */
function spawnBoss(){
bossActive=true;
boss={x:canvas.width/2-150,y:40,width:300,height:180,hp:200,vx:2,vy:3};
}

function updateBoss(){
if(!bossActive||!boss)return;

boss.x+=boss.vx;
boss.y+=boss.vy;

if(boss.x<=0||boss.x+boss.width>=canvas.width)boss.vx*=-1;
if(boss.y<=0||boss.y>=canvas.height/2)boss.vy*=-1;

ctx.drawImage(bossImg,boss.x,boss.y,boss.width,boss.height);

ctx.fillStyle="red";
ctx.fillRect(canvas.width/2-150,20,300,15);
ctx.fillStyle="lime";
ctx.fillRect(canvas.width/2-150,20,(boss.hp/200)*300,15);

if(bossShootCooldown<=0){

for(let a=0;a<360;a+=20){
let rad=a*Math.PI/180;

bossBullets.push({
x:boss.x+boss.width/2,
y:boss.y+boss.height/2,
dx:Math.cos(rad)*1,
dy:Math.sin(rad)*1
});
}

bossShootCooldown=100;
}

if(bossShootCooldown>0)bossShootCooldown--;

bullets.forEach((b,bi)=>{
if(b.x<boss.x+boss.width&&b.x+6>boss.x&&
b.y<boss.y+boss.height&&b.y+16>boss.y){
boss.hp-=damageMultiplier;
bullets.splice(bi,1);
if(boss.hp<=0){
bossActive=false;
boss=null;
showEnd("¡GANASTE!");
}
}
});
}

function updateBossBullets(){
bossBullets.forEach((b,i)=>{
b.x+=b.dx;
b.y+=b.dy;

ctx.fillStyle="red";
ctx.fillRect(b.x,b.y,8,8);

if(b.x<0||b.x>canvas.width||b.y<0||b.y>canvas.height){
bossBullets.splice(i,1);
}

if(b.x<player.x+70&&b.x+8>player.x&&
b.y<player.y+70&&b.y+8>player.y){

if(shield>0){shield--;}
else{player.hp--;}

bossBullets.splice(i,1);
}
});
}

/* HUD */
function drawHUD(){
ctx.fillStyle="white";
ctx.fillText("Vida: "+player.hp,20,40);
ctx.fillText("Puntos: "+score,20,70);
ctx.fillText("Escudos: "+shield,20,100);
}

function showEnd(text){
document.getElementById("endText").innerText=text;
document.getElementById("endScreen").style.display="block";
levelMusic.pause();
}

update();
</script>
</body>
</html>
