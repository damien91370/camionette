<!DOCTYPE html>
<html lang="fr">
<head>
<meta charset="UTF-8">
<title>Rue aux Enfants 🚐</title>
<style>
body{margin:0;background:#222;display:flex;justify-content:center;align-items:center;height:100vh;font-family:Arial,sans-serif;}
canvas{background:#333;border-radius:12px;touch-action:none;}
</style>
</head>
<body>
<canvas id="game" width="480" height="700"></canvas>
<script>
const canvas=document.getElementById("game");
const ctx=canvas.getContext("2d");

const W=canvas.width,H=canvas.height;
const LANES=[W/4,W/2,3*W/4];
const PLAYER_Y=H-120;

let BASE_SPEED=6,SPAWN_DELAY=900;

let playerLane=1;
let playerRect={x:LANES[playerLane],y:PLAYER_Y,w:60,h:90};

let children=[],police=[],obstacles=[];
let score=0,gameOver=false,lastSpawn=0;
let bestScore=Number(localStorage.getItem("bestScore")||0);

// ---------- TEXTURES ----------
function makeVan(){let s=document.createElement("canvas");s.width=60;s.height=90;let c=s.getContext("2d");c.fillStyle="#dddddd";c.fillRect(5,5,50,80);c.fillStyle="#78b4ff";c.fillRect(10,8,40,18);c.fillStyle="#aad2ff";c.fillRect(10,40,40,20);c.fillStyle="#ffff78";c.beginPath();c.arc(15,5,4,0,2*Math.PI);c.fill();c.beginPath();c.arc(45,5,4,0,2*Math.PI);c.fill();c.fillStyle="#1e1e1e";c.fillRect(0,20,5,18);c.fillRect(55,20,5,18);c.fillRect(0,55,5,18);c.fillRect(55,55,5,18);return s;}
function makeChild(){let s=document.createElement("canvas");s.width=40;s.height=55;let c=s.getContext("2d");c.fillStyle="#ffdcb4";c.beginPath();c.arc(20,12,10,0,2*Math.PI);c.fill();c.fillStyle="#000";c.beginPath();c.arc(16,12,2,0,2*Math.PI);c.fill();c.beginPath();c.arc(24,12,2,0,2*Math.PI);c.fill();c.fillStyle="#ff5050";c.fillRect(10,22,20,22);c.strokeStyle="#3c3c3c";c.lineWidth=3;c.beginPath();c.moveTo(15,44);c.lineTo(15,54);c.stroke();c.beginPath();c.moveTo(25,44);c.lineTo(25,54);c.stroke();return s;}
function makePolice(){let s=document.createElement("canvas");s.width=40;s.height=55;let c=s.getContext("2d");c.fillStyle="#b4b4ff";c.beginPath();c.arc(20,12,10,0,2*Math.PI);c.fill();c.fillStyle="#000";c.beginPath();c.arc(16,12,2,0,2*Math.PI);c.fill();c.beginPath();c.arc(24,12,2,0,2*Math.PI);c.fill();c.fillStyle="#000078";c.fillRect(10,22,20,22);c.strokeStyle="#3c3c3c";c.lineWidth=3;c.beginPath();c.moveTo(15,44);c.lineTo(15,54);c.stroke();c.beginPath();c.moveTo(25,44);c.lineTo(25,54);c.stroke();return s;}
function makeCar(){let s=document.createElement("canvas");s.width=60;s.height=90;let c=s.getContext("2d");c.fillStyle="#1e8cff";c.fillRect(0,20,60,60);c.fillStyle="#b4e6ff";c.fillRect(10,30,40,25);c.fillStyle="#ffff78";c.beginPath();c.arc(10,80,5,0,2*Math.PI);c.fill();c.beginPath();c.arc(50,80,5,0,2*Math.PI);c.fill();return s;}
function makeTrash(){let s=document.createElement("canvas");s.width=50;s.height=70;let c=s.getContext("2d");c.fillStyle="#46be46";c.fillRect(5,10,40,55);c.fillStyle="#329632";c.fillRect(0,0,50,15);for(let y=20;y<60;y+=10){c.strokeStyle="#28822a";c.beginPath();c.moveTo(8,y);c.lineTo(42,y);c.stroke();}return s;}

const VAN_IMG=makeVan(),CHILD_IMG=makeChild(),POLICE_IMG=makePolice(),CAR_IMG=makeCar(),TRASH_IMG=makeTrash();

// ---------- SPAWN ----------
function spawnObjects(){
    for(let i=0;i<2;i++){
        let x=LANES[Math.floor(Math.random()*3)];
        children.push({img:CHILD_IMG,x:x,y:-40,w:40,h:55});
    }
    if(Math.random()<0.8){
        let x=LANES[Math.floor(Math.random()*3)];
        police.push({img:POLICE_IMG,x:x,y:-40,w:40,h:55});
    }
    if(Math.random()<0.7){
        let x=LANES[Math.floor(Math.random()*3)];
        let img=Math.random()<0.5?CAR_IMG:TRASH_IMG;
        obstacles.push({img:img,x:x,y:-60,w:img.width,h:img.height});
    }
}

// ---------- COLLISIONS ----------
function isColliding(a,b){return Math.abs(a.x-b.x)<30 && Math.abs(a.y-b.y)<60;}

// ---------- DRAW ----------
function drawRect(img,x,y){ctx.drawImage(img,x-img.width/2,y-img.height/2);}
function drawBackground(offset){
    ctx.fillStyle="#303030";ctx.fillRect(0,0,W,H);
    ctx.fillStyle="#606060";ctx.fillRect(0,0,80,H);ctx.fillRect(W-80,0,80,H);
    ctx.strokeStyle="#888";ctx.lineWidth=2;
    LANES.forEach(x=>{ctx.beginPath();ctx.moveTo(x,0);ctx.lineTo(x,H);ctx.stroke();});
    for(let y=-100;y<H;y+=160*1.65){ctx.fillStyle="#777";ctx.fillRect(10,(y+offset)%H,60,120);ctx.fillStyle="#777";ctx.fillRect(W-70,(y+offset)%H,60,120);}
}
function drawUI(){
    ctx.fillStyle="white";ctx.font="20px Arial";ctx.fillText("Score : "+score,10,25);
    ctx.fillStyle="gold";ctx.fillText("Best : "+bestScore,10,50);
    if(gameOver){ctx.fillStyle="red";ctx.font="20px Arial";ctx.fillText("GAME OVER - appuyez sur R ou touchez pour restart",40,H/2);}
}

// ---------- RESTART ----------
function restart(){children=[];police=[];obstacles=[];score=0;playerLane=1;playerRect.x=LANES[playerLane];gameOver=false;}

// ---------- LOOP ----------
let lastTime=0,bgOffset=0;
function loop(time){
    let dt=time-lastTime;lastTime=time;
    let speed=BASE_SPEED+score*0.03;
    bgOffset+=speed;

    if(!gameOver){
        if(time-lastSpawn>SPAWN_DELAY){spawnObjects();lastSpawn=time;}

        children.forEach((c,i)=>{c.y+=speed;if(c.y>H) children.splice(i,1);if(isColliding(playerRect,c)){score++;children.splice(i,1);}});
        police.forEach((p,i)=>{p.y+=speed;if(p.y>H) police.splice(i,1);if(isColliding(playerRect,p)){gameOver=true;}});
        obstacles.forEach((o,i)=>{o.y+=speed;if(o.y>H) obstacles.splice(i,1);if(isColliding(playerRect,o)){gameOver=true;}});

        if(score>bestScore){bestScore=score;localStorage.setItem("bestScore",bestScore);}
    }

    drawBackground(bgOffset);
    children.forEach(c=>drawRect(c.img,c.x,c.y));
    police.forEach(c=>drawRect(c.img,c.x,c.y));
    obstacles.forEach(c=>drawRect(c.img,c.x,c.y));
    drawRect(VAN_IMG,playerRect.x,playerRect.y);
    drawUI();

    requestAnimationFrame(loop);
}
requestAnimationFrame(loop);

// ---------- CONTROLES PC ----------
document.addEventListener("keydown",e=>{
    if(!gameOver){
        if(e.key==="ArrowLeft" && playerLane>0) playerLane--;
        if(e.key==="ArrowRight" && playerLane<2) playerLane++;
        playerRect.x=LANES[playerLane];
    }else if(e.key==="r"){restart();}
});

// ---------- TOUCH & SWIPE MOBILE ----------
let touchStartX=null, swipeLocked=false;

canvas.addEventListener("touchstart", e => { 
    e.preventDefault();
    if(gameOver){ restart(); return; }
    touchStartX=e.touches[0].clientX;
    swipeLocked=false;
});

canvas.addEventListener("touchmove", e => {
    if(touchStartX===null || swipeLocked) return;
    let diff=e.touches[0].clientX-touchStartX;
    if(diff>40 && playerLane<2){playerLane++;playerRect.x=LANES[playerLane];swipeLocked=true;}
    if(diff<-40 && playerLane>0){playerLane--;playerRect.x=LANES[playerLane];swipeLocked=true;}
});

canvas.addEventListener("touchend", ()=>{touchStartX=null; swipeLocked=false;});
</script>
</body>
</html>
