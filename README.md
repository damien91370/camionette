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

<audio id="bgMusic" src="bossfight-Vextron.mp3" loop></audio>

<canvas id="game" width="480" height="700"></canvas>
<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

const music = document.getElementById("bgMusic");
music.volume = 0.5;
music.play().catch(()=>{}); // démarrage musique (peut bloquer selon navigateur)

const W = canvas.width;
const H = canvas.height;

const LANES = [W/4, W/2, 3*W/4];
const PLAYER_Y = H - 120;
const BASE_SPEED = 6;
const SPAWN_DELAY = 700;
const ROW_HEIGHT = 5*1.65; // distance entre rangées 1,65 fois plus que 5

let playerLane = 1;
let score = 0;
let gameOver = false;
let lastSpawn = 0;

// ------------------- IMAGES DESSINÉES -------------------
function makeVan(){
    const s = document.createElement("canvas");
    s.width=60; s.height=90;
    const c = s.getContext("2d");
    c.fillStyle="#e6e6e6"; c.fillRect(5,5,50,80);
    c.fillStyle="#78b4ff"; c.fillRect(10,8,40,18);
    c.fillStyle="#aad2ff"; c.fillRect(10,40,40,20);
    c.fillStyle="#ffff78"; c.beginPath(); c.arc(15,5,4,0,2*Math.PI); c.fill();
    c.beginPath(); c.arc(45,5,4,0,2*Math.PI); c.fill();
    c.fillStyle="#1e1e1e";
    c.fillRect(0,20,5,18); c.fillRect(55,20,5,18);
    c.fillRect(0,55,5,18); c.fillRect(55,55,5,18);
    return s;
}

function makeChild(){
    const s = document.createElement("canvas"); s.width=40; s.height=55;
    const c = s.getContext("2d");
    c.fillStyle="#ffdcB4"; c.beginPath(); c.arc(20,12,10,0,2*Math.PI); c.fill();
    c.fillStyle="#000"; c.beginPath(); c.arc(16,12,2,0,2*Math.PI); c.fill();
    c.beginPath(); c.arc(24,12,2,0,2*Math.PI); c.fill();
    c.fillStyle="#ff5050"; c.fillRect(10,22,20,22);
    c.strokeStyle="#3c3c3c"; c.lineWidth=3;
    c.beginPath(); c.moveTo(15,44); c.lineTo(15,54); c.stroke();
    c.beginPath(); c.moveTo(25,44); c.lineTo(25,54); c.stroke();
    return s;
}

function makePolice(){
    const s = document.createElement("canvas"); s.width=40; s.height=55;
    const c = s.getContext("2d");
    c.fillStyle="#b4b4ff"; c.beginPath(); c.arc(20,12,10,0,2*Math.PI); c.fill();
    c.fillStyle="#000"; c.beginPath(); c.arc(16,12,2,0,2*Math.PI); c.fill();
    c.beginPath(); c.arc(24,12,2,0,2*Math.PI); c.fill();
    c.fillStyle="#000080"; c.fillRect(10,22,20,22);
    c.strokeStyle="#3c3c3c"; c.lineWidth=3;
    c.beginPath(); c.moveTo(15,44); c.lineTo(15,54); c.stroke();
    c.beginPath(); c.moveTo(25,44); c.lineTo(25,54); c.stroke();
    return s;
}

function makeCar(){
    const s=document.createElement("canvas"); s.width=60; s.height=90;
    const c=s.getContext("2d");
    c.fillStyle="#1e8cff"; c.fillRect(0,20,60,60);
    c.fillStyle="#b4e6ff"; c.fillRect(10,30,40,25);
    c.fillStyle="#ffff78"; c.beginPath(); c.arc(10,80,5,0,2*Math.PI); c.fill();
    c.beginPath(); c.arc(50,80,5,0,2*Math.PI); c.fill();
    return s;
}

function makeTrash(){
    const s=document.createElement("canvas"); s.width=50; s.height=70;
    const c=s.getContext("2d");
    c.fillStyle="#46be46"; c.fillRect(5,10,40,55);
    c.fillStyle="#329632"; c.fillRect(0,0,50,15);
    c.strokeStyle="#288228"; c.lineWidth=2;
    for(let y=20;y<60;y+=10){ c.beginPath(); c.moveTo(8,y); c.lineTo(42,y); c.stroke(); }
    return s;
}

const VAN_IMG = makeVan();
const CHILD_IMG = makeChild();
const POLICE_IMG = makePolice();
const CAR_IMG = makeCar();
const TRASH_IMG = makeTrash();

// ------------------- PLAYER -------------------
let player_rect = {x:LANES[playerLane], y:PLAYER_Y, w:60, h:90};

// ------------------- OBJECTS -------------------
let children = [], police = [], obstacles = [];
let bestScore = 0;

// ------------------- UTIL -------------------
function rectsCollide(a,b){
    return a.x-a.w/2 < b.x+b.w/2 &&
           a.x+a.w/2 > b.x-b.w/2 &&
           a.y-a.h/2 < b.y+b.h/2 &&
           a.y+a.h/2 > b.y-b.h/2;
}

function randLane(){ return LANES[Math.floor(Math.random()*3)]; }

function spawnObjects(){
    for(let i=0;i<2;i++){
        children.push({x:randLane(), y:-40, w:40, h:55, img:CHILD_IMG});
    }
    if(Math.random()<0.8) police.push({x:randLane(), y:-40, w:40, h:55, img:POLICE_IMG});
    if(Math.random()<0.7){
        let img = Math.random()<0.5 ? CAR_IMG : TRASH_IMG;
        obstacles.push({x:randLane(), y:-60, w:img.width, h:img.height, img:img});
    }
}

function drawBackground(offset){
    ctx.fillStyle="#303030"; ctx.fillRect(0,0,W,H);
    ctx.fillStyle="#606060"; ctx.fillRect(0,0,80,H);
    ctx.fillRect(W-80,0,80,H);
    LANES.forEach(x=>{ ctx.strokeStyle="#888"; ctx.beginPath(); ctx.moveTo(x,0); ctx.lineTo(x,H); ctx.stroke(); });
    for(let y=-100;y<H;y+=ROW_HEIGHT*32){ // ratio 1.65
        ctx.fillStyle="#777"; ctx.fillRect(10,(y+offset)%H,60,120);
        ctx.fillStyle="#888"; ctx.fillRect(W-70,(y+offset)%H,60,120);
    }
}

function drawObjects(){
    children.forEach(o=>ctx.drawImage(o.img,o.x-o.w/2,o.y-o.h/2));
    police.forEach(o=>ctx.drawImage(o.img,o.x-o.w/2,o.y-o.h/2));
    obstacles.forEach(o=>ctx.drawImage(o.img,o.x-o.w/2,o.y-o.h/2));
}

function drawPlayer(){
    ctx.drawImage(VAN_IMG, player_rect.x-30, player_rect.y-45);
}

function drawUI(){
    ctx.fillStyle="white"; ctx.font="20px Arial";
    ctx.fillText("Score : "+score,10,25);
    ctx.fillStyle="gold"; ctx.fillText("Best : "+bestScore,10,50);
    if(gameOver){ ctx.fillStyle="red"; ctx.fillText("GAME OVER - R pour restart",120,H/2); }
}

// ------------------- LOOP -------------------
let lastTime=0;
let bgOffset=0;

function loop(time){
    let dt=time-lastTime;
    lastTime=time;
    let speed=BASE_SPEED+score*0.03;
    bgOffset+=speed;

    if(!gameOver){
        if(time-lastSpawn>SPAWN_DELAY){
            spawnObjects();
            lastSpawn=time;
        }

        children.forEach((c,i)=>{ c.y+=speed; if(rectsCollide(player_rect,c)) { score++; children.splice(i,1); }});
        police.forEach((c,i)=>{ c.y+=speed; if(rectsCollide(player_rect,c)) { gameOver=true; }});
        obstacles.forEach((c,i)=>{ c.y+=speed; if(rectsCollide(player_rect,c)) { gameOver=true; }});

        if(score>bestScore) bestScore=score;
    }

    drawBackground(bgOffset);
    drawObjects();
    drawPlayer();
    drawUI();
    requestAnimationFrame(loop);
}

document.addEventListener("keydown", e=>{
    if(!gameOver){
        if(e.key==="ArrowLeft" && playerLane>0) playerLane--;
        if(e.key==="ArrowRight" && playerLane<2) playerLane++;
        player_rect.x=LANES[playerLane];
    } else if(e.key==="r"){ score=0; gameOver=false; children=[]; police=[]; obstacles=[]; playerLane=1; player_rect.x=LANES[playerLane]; bgOffset=0;}
});

requestAnimationFrame(loop);

</script>
</body>
</html>
