<!DOCTYPE html>
<html lang="fr">
<head>
<meta charset="UTF-8" />
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

const W = 480;
const H = 700;

const LANES = [W/4, W/2, 3*W/4];
const PLAYER_Y = H - 120;

const BASE_SPEED = 6;
const SPAWN_DELAY = 700;

let playerLane = 1;
let score = 0;
let bestScore = Number(localStorage.getItem("bestScore") || 0);
let gameOver = false;

let lastSpawn = 0;

const children = [];
const police = [];
const obstacles = [];

//////////////////////////////////////////////////////////
// PLAYER
//////////////////////////////////////////////////////////

const player = {
    w:60,
    h:90,
    x:LANES[playerLane],
    y:PLAYER_Y
};

//////////////////////////////////////////////////////////
// INPUT
//////////////////////////////////////////////////////////

document.addEventListener("keydown", e => {

    if(!gameOver){
        if(e.key === "ArrowLeft" && playerLane > 0) playerLane--;
        if(e.key === "ArrowRight" && playerLane < 2) playerLane++;
        player.x = LANES[playerLane];
    }
    else if(e.key === "r" || e.key === "R"){
        restart();
    }
});

//////////////////////////////////////////////////////////
// COLLISION
//////////////////////////////////////////////////////////

function collide(a,b){
    return (
        a.x-a.w/2 < b.x+b.w/2 &&
        a.x+a.w/2 > b.x-b.w/2 &&
        a.y-a.h/2 < b.y+b.h/2 &&
        a.y+a.h/2 > b.y-b.h/2
    );
}

//////////////////////////////////////////////////////////
// DRAW SHAPES (IDENTIQUES À PYGAME)
//////////////////////////////////////////////////////////

function drawRect(x,y,w,h,color){
    ctx.fillStyle=color;
    ctx.fillRect(x-w/2,y-h/2,w,h);
}

function drawVan(x,y){
    drawRect(x,y,60,90,"#e6e6e6");
    drawRect(x,y-25,40,18,"#78b4ff");
    drawRect(x,y+5,40,20,"#aad2ff");
}

function drawChild(x,y){
    ctx.fillStyle="#ff5555";
    ctx.fillRect(x-20,y-10,40,30);
    ctx.beginPath();
    ctx.arc(x,y-22,10,0,Math.PI*2);
    ctx.fillStyle="#ffddb4";
    ctx.fill();
}

function drawPolice(x,y){
    ctx.fillStyle="#002288";
    ctx.fillRect(x-20,y-10,40,30);
    ctx.beginPath();
    ctx.arc(x,y-22,10,0,Math.PI*2);
    ctx.fillStyle="#aaccff";
    ctx.fill();
}

function drawCar(x,y){
    drawRect(x,y,60,90,"#1e8cff");
}

function drawTrash(x,y){
    drawRect(x,y,50,70,"#46b046");
}

//////////////////////////////////////////////////////////
// SPAWN
//////////////////////////////////////////////////////////

function randLane(){
    return LANES[Math.floor(Math.random()*3)];
}

function spawn(){

    for(let i=0;i<2;i++){
        children.push({x:randLane(),y:-40,w:40,h:55});
    }

    if(Math.random()<0.8){
        police.push({x:randLane(),y:-40,w:40,h:55});
    }

    if(Math.random()<0.7){
        if(Math.random()<0.5)
            obstacles.push({x:randLane(),y:-60,w:60,h:90,type:"car"});
        else
            obstacles.push({x:randLane(),y:-60,w:50,h:70,type:"trash"});
    }
}

//////////////////////////////////////////////////////////
// RESTART
//////////////////////////////////////////////////////////

function restart(){
    children.length=0;
    police.length=0;
    obstacles.length=0;
    score=0;
    gameOver=false;
    playerLane=1;
    player.x=LANES[playerLane];
}

//////////////////////////////////////////////////////////
// LOOP
//////////////////////////////////////////////////////////

let lastTime=0;

function loop(time){

    const dt = time-lastTime;
    lastTime=time;

    const speed = BASE_SPEED + score*0.03;

    ctx.fillStyle="#303030";
    ctx.fillRect(0,0,W,H);

    ctx.strokeStyle="#777";
    ctx.lineWidth=2;
    LANES.forEach(x=>{
        ctx.beginPath();
        ctx.moveTo(x,0);
        ctx.lineTo(x,H);
        ctx.stroke();
    });

    if(!gameOver && time-lastSpawn>SPAWN_DELAY){
        spawn();
        lastSpawn=time;
    }

    //////////////////////////////////////////////////
    // UPDATE OBJECTS
    //////////////////////////////////////////////////

    function update(list,type){

        for(let i=list.length-1;i>=0;i--){

            const o=list[i];
            o.y+=speed;

            if(collide(player,o)){
                if(type==="child"){
                    score++;
                    list.splice(i,1);
                }else{
                    gameOver=true;
                }
            }
            else if(o.y>H+100){
                list.splice(i,1);
            }
        }
    }

    if(!gameOver){
        update(children,"child");
        update(police,"bad");
        update(obstacles,"bad");
    }

    //////////////////////////////////////////////////
    // DRAW OBJECTS
    //////////////////////////////////////////////////

    children.forEach(o=>drawChild(o.x,o.y));
    police.forEach(o=>drawPolice(o.x,o.y));

    obstacles.forEach(o=>{
        if(o.type==="car") drawCar(o.x,o.y);
        else drawTrash(o.x,o.y);
    });

    drawVan(player.x,player.y);

    //////////////////////////////////////////////////
    // UI
    //////////////////////////////////////////////////

    ctx.fillStyle="white";
    ctx.font="20px Arial";
    ctx.fillText("Score : "+score,10,25);
    ctx.fillText("Best : "+bestScore,10,50);

    if(gameOver){
        ctx.fillStyle="red";
        ctx.fillText("GAME OVER - R pour restart",80,H/2);
    }

    if(score>bestScore){
        bestScore=score;
        localStorage.setItem("bestScore",bestScore);
    }

    requestAnimationFrame(loop);
}

requestAnimationFrame(loop);
</script>
</body>
</html>
