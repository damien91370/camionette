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
<canvas id="game" width="480" height="700"></canvas>

<audio id="bgMusic" src="bossfight-Vextron.mp3" loop autoplay></audio>

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

// ---------------- TEXTURES DESSINÉES ----------------
function makeVan(){
    const s = document.createElement("canvas");
    s.width=60; s.height=90;
    const c=s.getContext("2d");
    c.fillStyle="#dddddd"; c.fillRect(5,5,50,80);
    c.fillStyle="#88c"; c.fillRect(10,8,40,18);
    c.fillStyle="#abe"; c.fillRect(10,40,40,20);
    c.fillStyle="#ff0"; c.beginPath(); c.arc(15,5,4,0,Math.PI*2); c.fill();
    c.beginPath(); c.arc(45,5,4,0,Math.PI*2); c.fill();
    c.fillStyle="#222"; c.fillRect(0,20,5,18);
    c.fillRect(55,20,5,18); c.fillRect(0,55,5,18); c.fillRect(55,55,5,18);
    return s;
}
function makeChild(){
    const s=document.createElement("canvas"); s.width=40; s.height=55;
    const c=s.getContext("2d");
    c.fillStyle="#ffcdaa"; c.beginPath(); c.arc(20,12,10,0,Math.PI*2); c.fill();
    c.fillStyle="#000"; c.beginPath(); c.arc(16,12,2,0,Math.PI*2); c.fill();
    c.beginPath(); c.arc(24,12,2,0,Math.PI*2); c.fill();
    c.fillStyle="#f05050"; c.fillRect(10,22,20,22);
    c.strokeStyle="#3c3c3c"; c.lineWidth=3;
    c.beginPath(); c.moveTo(15,44); c.lineTo(15,54); c.stroke();
    c.beginPath(); c.moveTo(25,44); c.lineTo(25,54); c.stroke();
    return s;
}
function makePolice(){
    const s=document.createElement("canvas"); s.width=40; s.height=55;
    const c=s.getContext("2d");
    c.fillStyle="#b4b4ff"; c.beginPath(); c.arc(20,12,10,0,Math.PI*2); c.fill();
    c.fillStyle="#000"; c.beginPath(); c.arc(16,12,2,0,Math.PI*2); c.fill();
    c.beginPath(); c.arc(24,12,2,0,Math.PI*2); c.fill();
    c.fillStyle="#004080"; c.fillRect(10,22,20,22);
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
    c.fillStyle="#fffc78"; c.beginPath(); c.arc(10,80,5,0,Math.PI*2); c.fill();
    c.beginPath(); c.arc(50,80,5,0,Math.PI*2); c.fill();
    return s;
}
function makeTrash(){
    const s=document.createElement("canvas"); s.width=50; s.height=70;
    const c=s.getContext("2d");
    c.fillStyle="#46bf46"; c.fillRect(5,10,40,55);
    c.fillStyle="#329632"; c.fillRect(0,0,50,15);
    for(let y=20;y<60;y+=10){ c.strokeStyle="#288228"; c.beginPath(); c.moveTo(8,y); c.lineTo(42,y); c.stroke();}
    return s;
}

const VAN_IMG = makeVan();
const CHILD_IMG = makeChild();
const POLICE_IMG = makePolice();
const CAR_IMG = makeCar();
const TRASH_IMG = makeTrash();

// ---------------- PLAYER ----------------
const player = { rect:{x:LANES[playerLane],y:PLAYER_Y,w:60,h:90} };

// ---------------- SPAWN ----------------
function spawnObjects(){
    for(let i=0;i<2;i++) children.push({img:CHILD_IMG,x:randLane(),y:-40});
    if(Math.random()<0.8) police.push({img:POLICE_IMG,x:randLane(),y:-40});
    if(Math.random()<0.7) obstacles.push({img:Math.random()<0.5?CAR_IMG:TRASH_IMG,x:randLane(),y:-60});
}

function randLane(){ return LANES[Math.floor(Math.random()*3)]; }

// ---------------- RESTART ----------------
function restart(){
    children=[]; police=[]; obstacles=[]; score=0; gameOver=false; playerLane=1; player.rect.x=LANES[playerLane];
}

// ---------------- LOOP ----------------
let bgOffset=0;
function loop(){
    let speed=BASE_SPEED+score*0.03;
    bgOffset+=speed;

    if(!gameOver){
        let now=Date.now();
        if(now-lastSpawn>SPAWN_DELAY){ spawnObjects(); lastSpawn=now; }

        children.forEach((c,i)=>{ c.y+=speed; if(Math.abs(c.x-player.rect.x)<50 && Math.abs(c.y-player.rect.y)<50){score++; children.splice(i,1);} });
        police.forEach((p,i)=>{ p.y+=speed; if(Math.abs(p.x-player.rect.x)<50 && Math.abs(p.y-player.rect.y)<50){gameOver=true;} });
        obstacles.forEach((o,i)=>{ o.y+=speed; if(Math.abs(o.x-player.rect.x)<50 && Math.abs(o.y-player.rect.y)<50){gameOver=true;} });
    }

    // ---------------- DRAW ----------------
    ctx.fillStyle="#303030"; ctx.fillRect(0,0,W,H);
    ctx.fillStyle="#606060"; ctx.fillRect(0,0,80,H);
    ctx.fillRect(W-80,0,80,H);
    const lineSpacing = Math.floor(160*1.65);
    for(let y=-100;y<H;y+=lineSpacing){ ctx.fillStyle="#777"; ctx.fillRect(10,(y+bgOffset)%H,60,120); ctx.fillRect(W-70,(y+bgOffset)%H,60,120);}
    LANES.forEach(x=>{ ctx.strokeStyle="#888"; ctx.lineWidth=2; ctx.beginPath(); ctx.moveTo(x,0); ctx.lineTo(x,H); ctx.stroke(); });

    children.forEach(c=>ctx.drawImage(c.img,c.x-c.img.width/2,c.y-c.img.height/2));
    police.forEach(p=>ctx.drawImage(p.img,p.x-p.img.width/2,p.y-p.img.height/2));
    obstacles.forEach(o=>ctx.drawImage(o.img,o.x-o.img.width/2,o.y-o.img.height/2));
    ctx.drawImage(VAN_IMG,player.rect.x-30,player.rect.y-45);

    ctx.fillStyle="white"; ctx.font="20px Arial";
    ctx.fillText("Score: "+score,10,25);
    ctx.fillStyle="gold";
    ctx.fillText("Best: "+bestScore,10,50);
    if(gameOver){ ctx.fillStyle="red"; ctx.fillText("GAME OVER - R pour restart",60,H/2); }

    requestAnimationFrame(loop);
}

document.addEventListener("keydown",e=>{
    if(!gameOver){
        if(e.key==="ArrowLeft" && playerLane>0) playerLane--;
        if(e.key==="ArrowRight" && playerLane<2) playerLane++;
        player.rect.x=LANES[playerLane];
    } else if(e.key==="r") restart();
});

requestAnimationFrame(loop);
</script>
</body>
</html>
