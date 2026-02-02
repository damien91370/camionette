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

//////////////////////////////////////////////////
// SPRITES (canvas)
//////////////////////////////////////////////////

function makeVan(){
    const s = document.createElement("canvas");
    s.width=60; s.height=90;
    const c = s.getContext("2d");
    c.fillStyle="#e6e6e6"; c.fillRect(5,5,50,80);
    c.fillStyle="#78b4ff"; c.fillRect(10,8,40,18);
    c.fillStyle="#aad2ff"; c.fillRect(10,40,40,20);
    c.fillStyle="#ffff78"; c.beginPath(); c.arc(15,5,4,0,2*Math.PI); c.fill();
    c.beginPath(); c.arc(45,5,4,0,2*Math.PI); c.fill();
    c.fillStyle="#1e1e1e"; c.fillRect(0,20,5,18); c.fillRect(55,20,5,18);
    c.fillRect(0,55,5,18); c.fillRect(55,55,5,18);
    return s;
}

function makeChild(){
    const s=document.createElement("canvas"); s.width=40; s.height=55;
    const c=s.getContext("2d");
    c.fillStyle="#ffdcB4"; c.beginPath(); c.arc(20,12,10,0,2*Math.PI); c.fill();
    c.fillStyle="#000"; c.beginPath(); c.arc(16,12,2,0,2*Math.PI); c.fill();
    c.beginPath(); c.arc(24,12,2,0,2*Math.PI); c.fill();
    c.fillStyle="#ff5050"; c.fillRect(10,22,20,22);
    c.strokeStyle="#3c3c3c"; c.lineWidth=3; c.beginPath(); c.moveTo(15,44); c.lineTo(15,54); c.stroke();
    c.beginPath(); c.moveTo(25,44); c.lineTo(25,54); c.stroke();
    return s;
}

function makePolice(){
    const s=document.createElement("canvas"); s.width=40; s.height=55;
    const c=s.getContext("2d");
    c.fillStyle="#b4b4ff"; c.beginPath(); c.arc(20,12,10,0,2*Math.PI); c.fill();
    c.fillStyle="#000"; c.beginPath(); c.arc(16,12,2,0,2*Math.PI); c.fill();
    c.beginPath(); c.arc(24,12,2,0,2*Math.PI); c.fill();
    c.fillStyle="#000078"; c.fillRect(10,22,20,22);
    c.strokeStyle="#3c3c3c"; c.lineWidth=3; c.beginPath(); c.moveTo(15,44); c.lineTo(15,54); c.stroke();
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
    for(let y=20;y<60;y+=10){c.beginPath();c.moveTo(8,y);c.lineTo(42,y);c.stroke();}
    return s;
}

const VAN_IMG = makeVan();
const CHILD_IMG = makeChild();
const POLICE_IMG = makePolice();
const CAR_IMG = makeCar();
const TRASH_IMG = makeTrash();

//////////////////////////////////////////////////
// PLAYER
//////////////////////////////////////////////////

const player = {w:60,h:90,x:LANES[playerLane],y:PLAYER_Y};

//////////////////////////////////////////////////
// CONTROLES
//////////////////////////////////////////////////

document.addEventListener("keydown", e => {
    if(!gameOver){
        if(e.key==="ArrowLeft" && playerLane>0) playerLane--;
        if(e.key==="ArrowRight" && playerLane<2) playerLane++;
        player.x=LANES[playerLane];
    }else{
        if(e.key==="r"||e.key==="R") restart();
    }
});

//////////////////////////////////////////////////
// UTILS
//////////////////////////////////////////////////

function rectsCollide(a,b){
    return a.x-a.w/2<b.x+b.w/2 &&
           a.x+a.w/2>b.x-b.w/2 &&
           a.y-a.h/2<b.y+b.h/2 &&
           a.y+a.h/2>b.y-b.h/2;
}

function randLane(){return LANES[Math.floor(Math.random()*3)];}

//////////////////////////////////////////////////
// SPAWN
//////////////////////////////////////////////////

function spawn(){
    for(let i=0;i<2;i++){
        children.push({x:randLane(),y:-40,w:40,h:55,img:CHILD_IMG});
    }
    if(Math.random()<0.8) police.push({x:randLane(),y:-40,w:40,h:55,img:POLICE_IMG});
    if(Math.random()<0.7){
        let img=Math.random()<0.5?CAR_IMG:TRASH_IMG;
        obstacles.push({x:randLane(),y:-60,w:60,h:90,img:img});
    }
}

//////////////////////////////////////////////////
// DRAW
//////////////////////////////////////////////////

function drawRectImg(obj){ctx.drawImage(obj.img,obj.x-obj.w/2,obj.y-obj.h/2);}

function drawBackground(offset){
    ctx.fillStyle="#303030";
    ctx.fillRect(0,0,W,H);
    ctx.fillStyle="#606060";
    ctx.fillRect(0,0,80,H);
    ctx.fillRect(W-80,0,80,H);
    ctx.strokeStyle="#888"; ctx.lineWidth=2;
    LANES.forEach(x=>{ctx.beginPath();ctx.moveTo(x,0);ctx.lineTo(x,H);ctx.stroke();});

    let lineSpacing=160*7/5; // ratio 7/5
    for(let y=-100;y<H;y+=lineSpacing){
        ctx.fillStyle="#777"; ctx.fillRect(10,(y+offset)%H,60,120);
        ctx.fillRect(W-70,(y+offset)%H,60,120);
    }
}

function drawUI(){
    ctx.fillStyle="white"; ctx.font="20px Arial";
    ctx.fillText("Score : "+score,10,25);
    ctx.fillStyle="gold";
    ctx.fillText("Best : "+bestScore,10,50);
    if(gameOver){
        ctx.fillStyle="red"; ctx.font="30px Arial";
        ctx.fillText("GAME OVER - R pour restart",60,H/2);
    }
}

//////////////////////////////////////////////////
// RESTART
//////////////////////////////////////////////////

function restart(){
    children=[]; police=[]; obstacles=[]; score=0; playerLane=1; player.x=LANES[playerLane]; gameOver=false;
}

//////////////////////////////////////////////////
// LOOP
//////////////////////////////////////////////////

let lastTime=0;
let bgOffset=0;

function loop(time){
    let dt=time-lastTime;
    lastTime=time;
    let speed=BASE_SPEED+score*0.03;
    bgOffset+=speed;

    if(!gameOver){
        if(time-lastSpawn>SPAWN_DELAY){spawn();lastSpawn=time;}

        children.forEach((c,i)=>{c.y+=speed;if(rectsCollide(player,c)){score++;children.splice(i,1);}});
        police.forEach((p,i)=>{p.y+=speed;if(rectsCollide(player,p)){gameOver=true;}});
        obstacles.forEach((o,i)=>{o.y+=speed;if(rectsCollide(player,o)){gameOver=true;}});
    }

    drawBackground(bgOffset);
    children.forEach(drawRectImg);
    police.forEach(drawRectImg);
    obstacles.forEach(drawRectImg);

    ctx.drawImage(VAN_IMG,player.x-30,player.y-45);

    drawUI();
    requestAnimationFrame(loop);
}

requestAnimationFrame(loop);
</script>
</body>
</html>
