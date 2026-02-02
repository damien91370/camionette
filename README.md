<!DOCTYPE html>
<html lang="fr">
<head>
<meta charset="UTF-8">
<title>Rue aux Enfants 🚐</title>
<style>
body{
    margin:0;
    background:#222;
    display:flex;
    justify-content:center;
    align-items:center;
    height:100vh;
    font-family:Arial, sans-serif;
}
canvas{
    background:#333;
    border-radius:12px;
}
</style>
</head>
<body>

<audio id="bgm" src="bossfight-Vextron.mp3" loop autoplay></audio>
<canvas id="game" width="480" height="700"></canvas>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

const W = canvas.width;
const H = canvas.height;

const LANES = [W/4, W/2, 3*W/4];
const PLAYER_Y = H - 120;

let BASE_SPEED = 6;
let SPAWN_DELAY = 700;

let playerLane = 1;
let children = [];
let police = [];
let obstacles = [];

let score = 0;
let gameOver = false;
let lastSpawn = 0;
let bestScore = Number(localStorage.getItem("bestScore") || 0);

// ----------- TEXTURES -----------

function makeVan(){let s=document.createElement("canvas");s.width=60;s.height=90;let c=s.getContext("2d");c.fillStyle="#dddddd";c.fillRect(5,5,50,80);c.fillStyle="#78b4ff";c.fillRect(10,8,40,18);c.fillStyle="#aad2ff";c.fillRect(10,40,40,20);c.fillStyle="#ffff78";c.beginPath();c.arc(15,5,4,0,Math.PI*2);c.fill();c.beginPath();c.arc(45,5,4,0,Math.PI*2);c.fill();c.fillStyle="#1e1e1e";c.fillRect(0,20,5,18);c.fillRect(55,20,5,18);c.fillRect(0,55,5,18);c.fillRect(55,55,5,18);return s;}
function makeChild(){let s=document.createElement("canvas");s.width=40;s.height=55;let c=s.getContext("2d");c.fillStyle="#ffdcb4";c.beginPath();c.arc(20,12,10,0,Math.PI*2);c.fill();c.fillStyle="#000";c.beginPath();c.arc(16,12,2,0,Math.PI*2);c.fill();c.beginPath();c.arc(24,12,2,0,Math.PI*2);c.fill();c.fillStyle="#ff5050";c.fillRect(10,22,20,22);c.strokeStyle="#3c3c3c";c.lineWidth=3;c.beginPath();c.moveTo(15,44);c.lineTo(15,54);c.stroke();c.beginPath();c.moveTo(25,44);c.lineTo(25,54);c.stroke();return s;}
function makePolice(){let s=document.createElement("canvas");s.width=40;s.height=55;let c=s.getContext("2d");c.fillStyle="#b4b4ff";c.beginPath();c.arc(20,12,10,0,Math.PI*2);c.fill();c.fillStyle="#000";c.beginPath();c.arc(16,12,2,0,Math.PI*2);c.fill();c.beginPath();c.arc(24,12,2,0,Math.PI*2);c.fill();c.fillStyle="#000078";c.fillRect(10,22,20,22);c.strokeStyle="#3c3c3c";c.lineWidth=3;c.beginPath();c.moveTo(15,44);c.lineTo(15,54);c.stroke();c.beginPath();c.moveTo(25,44);c.lineTo(25,54);c.stroke();return s;}
function makeCar(){let s=document.createElement("canvas");s.width=60;s.height=90;let c=s.getContext("2d");c.fillStyle="#1e8cff";c.fillRect(0,20,60,60);c.fillStyle="#b4e6ff";c.fillRect(10,30,40,25);c.fillStyle="#ffff78";c.beginPath();c.arc(10,80,5,0,Math.PI*2);c.fill();c.beginPath();c.arc(50,80,5,0,Math.PI*2);c.fill();return s;}
function makeTrash(){let s=document.createElement("canvas");s.width=50;s.height=70;let c=s.getContext("2d");c.fillStyle="#46be46";c.fillRect(5,10,40,55);c.fillStyle="#329632";c.fillRect(0,0,50,15);for(let y=20;y<60;y+=10){c.strokeStyle="#28822a";c.beginPath();c.moveTo(8,y);c.lineTo(42,y);c.stroke();}return s;}

const VAN_IMG = makeVan();
const CHILD_IMG = makeChild();
const POLICE_IMG = makePolice();
const CAR_IMG = makeCar();
const TRASH_IMG = makeTrash();

let player_rect = {x: LANES[playerLane], y:PLAYER_Y, w:60, h:90};

// ----------- SPAWN SAFE -----------

function spawn(){
    let occupied = [...children.map(c=>c.x), ...police.map(p=>p.x), ...obstacles.map(o=>o.x)];
    let freeLanes = LANES.filter(l=>!occupied.includes(l));

    // --- enfants (max 2) ---
    for(let i=0;i<Math.min(2, freeLanes.length); i++){
        let idx=Math.floor(Math.random()*freeLanes.length);
        let lane=freeLanes.splice(idx,1)[0];
        children.push({img:CHILD_IMG,x:lane,y:-40,w:40,h:55});
    }

    // --- police (max 1) ---
    if(Math.random()<0.8 && freeLanes.length>0){
        let idx=Math.floor(Math.random()*freeLanes.length);
        let lane=freeLanes.splice(idx,1)[0];
        police.push({img:POLICE_IMG,x:lane,y:-40,w:40,h:55});
    }

    // --- obstacles (tripler) ---
    let obstaclesToSpawn = Math.min(3, freeLanes.length); // 3x plus
    for(let i=0; i<obstaclesToSpawn; i++){
        let idx=Math.floor(Math.random()*freeLanes.length);
        let lane=freeLanes.splice(idx,1)[0];
        obstacles.push({img:Math.random()<0.5?CAR_IMG:TRASH_IMG,x:lane,y:-60,w:60,h:90});
    }
}


// ----------- COLLISION -----------

function isColliding(a,b){
    return a.x - a.w/2 < b.x + b.w/2 &&
           a.x + a.w/2 > b.x - b.w/2 &&
           a.y - a.h/2 < b.y + b.h/2 &&
           a.y + a.h/2 > b.y - b.h/2;
}

// ----------- DESSIN -----------

function drawRect(img,x,y){ ctx.drawImage(img,x-img.width/2,y-img.height/2); }

function drawBackground(offset){
    ctx.fillStyle="#303030"; ctx.fillRect(0,0,W,H);
    ctx.fillStyle="#606060"; ctx.fillRect(0,0,80,H); ctx.fillRect(W-80,0,80,H);
    ctx.strokeStyle="#888"; ctx.lineWidth=2;
    LANES.forEach(x=>{ ctx.beginPath(); ctx.moveTo(x,0); ctx.lineTo(x,H); ctx.stroke(); });
    let space=160*1.65;
    for(let y=-100;y<H;y+=space){
        ctx.fillStyle="#777"; ctx.fillRect(10,(y+offset)%H,60,120);
        ctx.fillStyle="#777"; ctx.fillRect(W-70,(y+offset)%H,60,120);
    }
}

function drawUI(){
    ctx.fillStyle="white"; ctx.font="20px Arial";
    ctx.fillText("Score : "+score,10,25);
    ctx.fillStyle="gold"; ctx.fillText("Best : "+bestScore,10,50);
    if(gameOver){ctx.fillStyle="red";ctx.font="30px Arial";ctx.fillText("GAME OVER",120,H/2);ctx.font="20px Arial";ctx.fillText("Appuie sur R / Touchez pour rejouer",80,H/2+30);}
}

// ----------- RESTART -----------

function restart(){
    children=[]; police=[]; obstacles=[]; score=0; playerLane=1; player_rect.x=LANES[playerLane]; gameOver=false;
}

// ----------- LOOP -----------

let lastTime=0,bgOffset=0;

function loop(time){
    let dt=time-lastTime; lastTime=time;
    let speed=BASE_SPEED+score*0.03;
    bgOffset+=speed;

    if(!gameOver){
        if(time-lastSpawn>SPAWN_DELAY){ spawn(); lastSpawn=time; }

        children.forEach(c=>c.y+=speed);
        police.forEach(p=>p.y+=speed);
        obstacles.forEach(o=>o.y+=speed);

        children=children.filter(c=>c.y<H+50);
        police=police.filter(p=>p.y<H+50);
        obstacles=obstacles.filter(o=>o.y<H+50);

        // collision enfants
        children=children.filter(c=>{ if(isColliding(player_rect,c)){score++; return false;} return true; });

        if(police.some(p=>isColliding(player_rect,p)) || obstacles.some(o=>isColliding(player_rect,o))) gameOver=true;

        if(score>bestScore){ bestScore=score; localStorage.setItem("bestScore",bestScore); }
    }

    drawBackground(bgOffset);
    children.forEach(c=>drawRect(c.img,c.x,c.y));
    police.forEach(p=>drawRect(p.img,p.x,p.y));
    obstacles.forEach(o=>drawRect(o.img,o.x,o.y));
    drawRect(VAN_IMG,player_rect.x,player_rect.y);
    drawUI();

    requestAnimationFrame(loop);
}

requestAnimationFrame(loop);

// ----------- CONTROLES -----------

document.addEventListener("keydown",e=>{
    if(!gameOver){
        if(e.key==="ArrowLeft" && playerLane>0) playerLane--;
        if(e.key==="ArrowRight" && playerLane<2) playerLane++;
        player_rect.x=LANES[playerLane];
    } else { if(e.key==="r") restart(); }
});

// ---------- SWIPE MOBILE ----------

let touchStartX=0,touchEndX=0;

canvas.addEventListener("touchstart",e=>{ touchStartX=e.touches[0].clientX; });
canvas.addEventListener("touchend",e=>{ touchEndX=e.changedTouches[0].clientX; handleSwipe(); if(gameOver) restart(); });

function handleSwipe(){
    let dx=touchEndX-touchStartX;
    if(Math.abs(dx)<30) return;
    if(!gameOver){
        if(dx>0 && playerLane<2) playerLane++;
        if(dx<0 && playerLane>0) playerLane--;
        player_rect.x=LANES[playerLane];
    }
}
</script>

</body>
</html>
