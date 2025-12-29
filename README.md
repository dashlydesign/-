<!doctype html>
<html>
<head>
<meta charset="utf-8">
<style>
html,body{
  margin:0;
  background:#121212;
  color:#fff;
  font-family:Segoe UI, system-ui, sans-serif;
  height:100%;
}

body{
  display:flex;
  justify-content:center;
  align-items:center;
}

.wrapper{
  display:flex;
  flex-direction:column;
  align-items:center;
  gap:36px;
}

.clocks{
  display:flex;
  gap:120px;
}

canvas{display:block}

.sessions{
  display:grid;
  grid-template-columns:repeat(4,1fr);
  gap:20px;
  width:680px;
}

.session{
  background:#1a1a1a;
  border-radius:10px;
  padding:12px 0;
  text-align:center;
  font-size:14px;
  opacity:.45;
  transition:background .25s, opacity .25s;
}

.session.active{
  opacity:1;
}

.session span{
  display:block;
  font-size:12px;
  opacity:.6;
  margin-top:4px;
}

.countdowns{
  display:flex;
  gap:48px;
  font-size:14px;
  flex-wrap:wrap;
  justify-content:center;
}
</style>
</head>

<body>

<div class="wrapper">

  <div class="clocks">
    <canvas id="mt5" width="160" height="160"></canvas>
    <canvas id="ist" width="160" height="160"></canvas>
  </div>

  <div class="sessions" id="sessions"></div>

  <div class="countdowns" id="countdowns"></div>

</div>

<script>
// ---------- COLORS ----------
const sessionColors={
  "Sydney":"#1f3d2b",
  "Tokyo":"#1e344a",
  "London":"#3d2f1f",
  "New York":"#35203f"
};

// ---------- FETCH TIME ----------
let utcBase=0, istBase=0;

fetch("https://worldtimeapi.org/api/timezone/Etc/UTC")
  .then(r=>r.json())
  .then(d=>utcBase=new Date(d.datetime).getTime()-Date.now());

fetch("https://worldtimeapi.org/api/timezone/Asia/Kolkata")
  .then(r=>r.json())
  .then(d=>istBase=new Date(d.datetime).getTime()-Date.now());

// ---------- SESSIONS (MT5 UTC+2) ----------
const sessions=[
  {name:"Sydney",start:0,end:9},
  {name:"Tokyo",start:2,end:11},
  {name:"London",start:9,end:18},
  {name:"New York",start:14,end:23}
];

const sessionsEl=document.getElementById("sessions");
sessions.forEach(s=>{
  const d=document.createElement("div");
  d.className="session";
  d.dataset.name=s.name;
  d.innerHTML=`${s.name}<span>${String(s.start).padStart(2,"0")}–${String(s.end).padStart(2,"0")}</span>`;
  sessionsEl.appendChild(d);
});

// ---------- CLOCK ----------
function drawClock(canvas,date){
  const ctx=canvas.getContext("2d");
  const r=canvas.width/2;
  ctx.setTransform(1,0,0,1,0,0);
  ctx.clearRect(0,0,canvas.width,canvas.height);
  ctx.translate(r,r);

  ctx.strokeStyle="rgba(255,255,255,.25)";
  for(let i=0;i<12;i++){
    const a=i*Math.PI/6;
    ctx.beginPath();
    ctx.moveTo(Math.cos(a)*r*0.82,Math.sin(a)*r*0.82);
    ctx.lineTo(Math.cos(a)*r*0.88,Math.sin(a)*r*0.88);
    ctx.stroke();
  }

  const sec=date.getSeconds()+date.getMilliseconds()/1000;
  const min=date.getMinutes()+sec/60;
  const hr=date.getHours()%12+min/60;

  function hand(a,l,w){
    ctx.beginPath();
    ctx.lineWidth=w;
    ctx.lineCap="round";
    ctx.strokeStyle="#fff";
    ctx.moveTo(0,0);
    ctx.lineTo(Math.cos(a-Math.PI/2)*l,Math.sin(a-Math.PI/2)*l);
    ctx.stroke();
  }

  hand(hr*Math.PI/6,r*0.48,4.2);
  hand(min*Math.PI/30,r*0.68,2.7);
  hand(sec*Math.PI/30,r*0.83,1.6);
}

// ---------- MULTI-ACTIVE LOGIC ----------
function updateSessions(mt5){
  const nowSec=mt5.getHours()*3600+mt5.getMinutes()*60+mt5.getSeconds();
  const active=[], inactive=[];

  sessions.forEach(s=>{
    if(nowSec>=s.start*3600 && nowSec<s.end*3600) active.push(s);
    else inactive.push(s);
  });

  [...sessionsEl.children].forEach(el=>{
    const on=active.some(a=>a.name===el.dataset.name);
    el.classList.toggle("active",on);
    el.style.background=on?sessionColors[el.dataset.name]:"#1a1a1a";
  });

  const cd=document.getElementById("countdowns");
  cd.innerHTML="";
  const f=v=>String(v).padStart(2,"0");

  active
    .map(s=>{
      let t=s.end*3600-nowSec;
      if(t<0)t+=86400;
      return {name:s.name,t};
    })
    .sort((a,b)=>a.t-b.t)
    .forEach(e=>{
      cd.innerHTML+=`<div>${e.name} ends in ${f(e.t/3600|0)}:${f((e.t%3600)/60|0)}:${f(e.t%60)}</div>`;
    });

  if(inactive.length){
    const n=inactive
      .map(s=>{
        let t=s.start*3600-nowSec;
        if(t<0)t+=86400;
        return {name:s.name,t};
      })
      .sort((a,b)=>a.t-b.t)[0];

    cd.innerHTML+=`<div>${n.name} starts in ${f(n.t/3600|0)}:${f((n.t%3600)/60|0)}:${f(n.t%60)}</div>`;
  }
}

// ---------- LOOP ----------
function loop(){
  const now=Date.now();
  const utc=new Date(now+utcBase);
  const mt5=new Date(utc.getTime()+2*3600000);
  const ist=new Date(now+istBase);

  drawClock(document.getElementById("mt5"),mt5);
  drawClock(document.getElementById("ist"),ist);
  updateSessions(mt5);

  requestAnimationFrame(loop);
}
loop();
</script>

</body>
</html>
