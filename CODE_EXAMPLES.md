# Representative source excerpts

These excerpts are copied from the supplied source snapshot. Full files, dependencies and tests are in the source ZIP.

## app/hoop-lab.tsx

```tsx
"use client";

import { useCallback, useEffect, useRef, useState } from "react";

type Params = { m: number; R: number; Omega: number; g: number; damping: number };
type State = { theta: number; omega: number; phase: number; t: number };
type Sample = { t: number; theta: number; energy: number };
type P3 = { x: number; y: number; z: number };

const DEFAULTS: Params = { m: .35, R: 1.3, Omega: 3.4, g: 9.81, damping: .04 };
const INITIAL: State = { theta: 22 * Math.PI / 180, omega: 0, phase: 0, t: 0 };

function acceleration(s: State, p: Params) {
  return Math.sin(s.theta) * (p.Omega ** 2 * Math.cos(s.theta) - p.g / p.R) - p.damping * s.omega;
}

function rk4(s: State, dt: number, p: Params): State {
  const f = (q: State) => ({ theta: q.omega, omega: acceleration(q,p), phase: p.Omega });
  const sh = (q: State, d: ReturnType<typeof f>, h: number): State => ({theta:q.theta+d.theta*h,omega:q.omega+d.omega*h,phase:q.phase+d.phase*h,t:q.t+h});
  const a=f(s),b=f(sh(s,a,dt/2)),c=f(sh(s,b,dt/2)),d=f(sh(s,c,dt));
  return {
    theta:s.theta+dt*(a.theta+2*b.theta+2*c.theta+d.theta)/6,
    omega:s.omega+dt*(a.omega+2*b.omega+2*c.omega+d.omega)/6,
    phase:(s.phase+dt*(a.phase+2*b.phase+2*c.phase+d.phase)/6)%(Math.PI*2),
    t:s.t+dt
  };
}

function energy(s: State, p: Params) {
  return .5*p.m*p.R**2*s.omega**2 + p.m*p.g*p.R*(1-Math.cos(s.theta)) - .5*p.m*p.R**2*p.Omega**2*Math.sin(s.theta)**2;
}

function Slider({label,value,min,max,step,unit,onChange}:{label:string;value:number;min:number;max:number;step:number;unit:string;onChange:(n:number)=>void}) {
  const pct=(value-min)/(max-min)*100;
  return <label className="sliderRow"><span>{label}</span><output>{value.toFixed(step<.01?3:step<.1?2:1)} {unit}</output><input aria-label={label} type="range" min={min} max={max} step={step} value={value} style={{"--fill":pct+"%"} as React.CSSProperties} onChange={e=>onChange(Number(e.target.value))}/></label>;
}

function Scene({sim,params,running,onSetState}:{sim:State;params:Params;running:boolean;onSetState:(p:Partial<State>)=>void}) {
  const ref=useRef<HTMLCanvasElement>(null);
  const view=useRef({yaw:-.68,pitch:.25,zoom:1,drag:false,px:0,py:0});
  const drag=useRef<"orbit"|"bead"|null>(null);
  const hit=useRef({x:0,y:0,cx:0,cy:0});

  const draw=useCallback(()=>{
    const canvas=ref.current;if(!canvas)return;
    const rect=canvas.getBoundingClientRect(),dpr=Math.min(devicePixelRatio||1,2);
    if(canvas.width!==Math.round(rect.width*dpr)||canvas.height!==Math.round(rect.height*dpr)){canvas.width=Math.round(rect.width*dpr);canvas.height=Math.round(rect.height*dpr)}
    const ctx=canvas.getContext("2d")!;ctx.setTransform(dpr,0,0,dpr,0,0);
    const W=rect.width,H=rect.height,v=view.current,size=Math.max(1,params.R/1.3),scale=Math.min(W/(5.8*size),H/(4.7*size))*v.zoom;
    const cy=Math.cos(v.yaw),sy=Math.sin(v.yaw),cp=Math.cos(v.pitch),sp=Math.sin(v.pitch);
    const project=(p:P3)=>{const x1=cy*p.x-sy*p.z,z1=sy*p.x+cy*p.z,y2=cp*p.y-sp*z1,z2=sp*p.y+cp*z1,persp=1/Math.max(.6,1+z2*.06);return{x:W*.52+x1*scale*persp,y:H*.52-y2*scale*persp,s:persp,z:z2}};
    const line=(pts:P3[],color:string,width=1,dash:number[]=[] )=>{const q=pts.map(project);ctx.beginPath();ctx.moveTo(q[0].x,q[0].y);q.slice(1).forEach(a=>ctx.lineTo(a.x,a.y));ctx.strokeStyle=color;ctx.lineWidth=width;ctx.setLineDash(dash);ctx.stroke();ctx.setLineDash([])};
    const poly=(pts:P3[],fill:string,stroke="rgba(255,255,255,.12)")=>{const q=pts.map(project);ctx.beginPath();ctx.moveTo(q[0].x,q[0].y);q.slice(1).forEach(a=>ctx.lineTo(a.x,a.y));ctx.closePath();ctx.fillStyle=fill;ctx.fill();ctx.strokeStyle=stroke;ctx.stroke()};
    const bg=ctx.createLinearGradient(0,0,0,H);bg.addColorStop(0,"#101a20");bg.addColorStop(1,"#06090b");ctx.fillStyle=bg;ctx.fillRect(0,0,W,H);
    const ground=.3-params.R-.3;
    for(let gx=-3;gx<=3;gx+=.4)line([{x:gx,y:ground,z:-2.2},{x:gx,y:ground,z:2.2}],"rgba(255,255,255,.045)");
    for(let gz=-2.2;gz<=2.2;gz+=.4)line([{x:-3,y:ground,z:gz},{x:3,y:ground,z:gz}],"rgba(255,255,255,.045)");

    const envelope:P3[]=[];for(let i=0;i<=120;i++){const a=i/120*Math.PI*2;envelope.push({x:params.R*Math.cos(a),y:.3,z:params.R*Math.sin(a)})}line(envelope,"rgba(101,216,243,.11)",1,[4,7]);
    line([{x:0,y:ground,z:0},{x:0,y:.3+params.R+.72,z:0}],"#738891",4);
    line([{x:0,y:ground,z:0},{x:0,y:.3+params.R+.72,z:0}],"rgba(101,216,243,.52)",1);

    poly([{x:-.55,y:ground,z:-.42},{x:.55,y:ground,z:-.42},{x:.55,y:ground+.24,z:-.42},{x:-.55,y:ground+.24,z:-.42}],"#263a43");
    poly([{x:.55,y:ground,z:-.42},{x:.55,y:ground,z:.42},{x:.55,y:ground+.24,z:.42},{x:.55,y:ground+.24,z:-.42}],"#14252c");
    poly([{x:-.55,y:ground+.24,z:-.42},{x:.55,y:ground+.24,z:-.42},{x:.55,y:ground+.24,z:.42},{x:-.55,y:ground+.24,z:.42}],"#36525e");

    const arc:P3[]=[];for(let i=0;i<=45;i++){const a=-.4+i/45*1.55;arc.push({x:.54*Math.cos(a),y:.3+params.R+.52,z:.54*Math.sin(a)})}line(arc,"#65d8f3",2.5);
    const ae=arc.at(-1)!,ap=arc.at(-4)!;line([ae,{x:ae.x+(ae.x-ap.x)*1.8,y:ae.y,z:ae.z+(ae.z-ap.z)*1.8}],"#65d8f3",5);

    const ph=sim.phase,hoop:P3[]=[];for(let i=0;i<=180;i++){const a=i/180*Math.PI*2;hoop.push({x:params.R*Math.sin(a)*Math.cos(ph),y:.3-params.R*Math.cos(a),z:params.R*Math.sin(a)*Math.sin(ph)})}
    line(hoop,"#aebdc4",7);line(hoop,"#e8f0f3",2);
    line([{x:0,y:.3-params.R,z:0},{x:0,y:.3+params.R,z:0}],"rgba(243,173,67,.34)",2);
    const bead:P3={x:params.R*Math.sin(sim.theta)*Math.cos(ph),y:.3-params.R*Math.cos(sim.theta),z:params.R*Math.sin(sim.theta)*Math.sin(ph)};
    const qb=project(bead),qc=project({x:0,y:.3,z:0});hit.current={x:qb.x,y:qb.y,cx:qc.x,cy:qc.y};
    const trail:P3[]=[];for(let i=0;i<=80;i++){const a=i/80*Math.PI*2;trail.push({x:params.R*Math.sin(sim.theta)*Math.cos(a),y:bead.y,z:params.R*Math.sin(sim.theta)*Math.sin(a)})}line(trail,"rgba(243,173,67,.16)",1,[3,6]);
    const rad=Math.max(13,scale*.16*qb.s),g=ctx.createRadialGradient(qb.x-rad*.35,qb.y-rad*.4,1,qb.x,qb.y,rad);g.addColorStop(0,"#fff0a5");g.addColorStop(.35,"#f3ad43");g.addColorStop(1,"#833f10");ctx.fillStyle=g;ctx.strokeStyle="#ffd06c";ctx.lineWidth=2;ctx.beginPath();ctx.arc(qb.x,qb.y,rad,0,Math.PI*2);ctx.fill();ctx.stroke();
    const top=project({x:0,y:.3+params.R+.82,z:0});ctx.fillStyle="#65d8f3";ctx.font="700 12px ui-monospace,monospace";ctx.fillText("z",top.x+8,top.y+4);
    ctx.fillStyle="rgba(225,238,245,.72)";ctx.font="500 11px ui-monospace,monospace";ctx.fillText("DRAG BEAD TO SET θ · DRAG SPACE TO ORBIT",16,H-17);
  },[sim,params,running]);

  useEffect(()=>{draw();const resize=()=>draw();window.addEventListener("resize",resize);return()=>window.removeEventListener("resize",resize)},[draw]);
  const point=(e:React.PointerEvent<HTMLCanvasElement>)=>{const r=e.currentTarget.getBoundingClientRect();return{x:e.clientX-r.left,y:e.clientY-r.top}};
  return <canvas ref={ref} className="scene" aria-label="Interactive three-dimensional bead on a vertically rotating hoop"
    onPointerDown={e=>{const p=point(e);drag.current=Math.hypot(p.x-hit.current.x,p.y-hit.current.y)<36?"bead":"orbit";view.current.drag=true;view.current.px=p.x;view.current.py=p.y;e.currentTarget.setPointerCapture(e.pointerId)}}
    onPointerMove={e=>{if(!view.current.drag)return;const p=point(e),dx=p.x-view.current.px,dy=p.y-view.current.py;if(drag.current==="orbit"){view.current.yaw+=dx*.008;view.current.pitch=Math.max(-.08,Math.min(1.05,view.current.pitch+dy*.006));draw()}else{const h=hit.current;onSetState({theta:Math.max(-Math.PI+.03,Math.min(Math.PI-.03,Math.atan2(p.x-h.cx,p.y-h.cy))),omega:0})}view.current.px=p.x;view.current.py=p.y}}
    onPointerUp={()=>{view.current.drag=false;drag.current=null}}
    onWheel={e=>{e.preventDefault();view.current.zoom=Math.max(.68,Math.min(1.5,view.current.zoom*Math.exp(-e.deltaY*.001)));draw()}}/>;
}

function Plot({history,equilibrium}:{history:Sample[];equilibrium:number}) {
  const ref=useRef<HTMLCanvasElement>(null);
  useEffect(()=>{const c=ref.current;if(!c)return;const r=c.getBoundingClientRect(),d=Math.min(devicePixelRatio||1,2);c.width=r.width*d;c.height=r.height*d;const x=c.getContext("2d")!;x.setTransform(d,0,0,d,0,0);x.clearRect(0,0,r.width,r.height);const p={l:34,r:10,t:12,b:24},W=r.width-p.l-p.r,H=r.height-p.t-p.b;x.strokeStyle="rgba(255,255,255,.07)";for(let i=0;i<=4;i++){const yy=p.t+i*H/4;x.beginPath();x.moveTo(p.l,yy);x.lineTo(p.l+W,yy);x.stroke()}if(history.length<2)return;const start=history[0].t,end=history.at(-1)!.t,max=Math.max(.45,equilibrium*1.25,...history.map(s=>Math.abs(s.theta)));if(equilibrium>0){for(const sign of [-1,1]){const yy=p.t+H/2-sign*equilibrium/max*H*.44;x.beginPath();x.moveTo(p.l,yy);x.lineTo(p.l+W,yy);x.strokeStyle="rgba(243,173,67,.6)";x.setLineDash([5,5]);x.stroke();x.setLineDash([])}}x.beginPath();history.forEach((s,i)=>{const px=p.l+(s.t-start)/Math.max(.001,end-start)*W,py=p.t+H/2-s.theta/max*H*.44;i?x.lineTo(px,py):x.moveTo(px,py)});x.strokeStyle="#65d8f3";x.lineWidth=2;x.stroke();x.fillStyle="#73818a";x.font="10px ui-monospace,monospace";x.fillText(Math.max(0,end-start).toFixed(0)+" s window",p.l,r.height-7)},[history,equilibrium]);
  return <canvas ref={ref} className="plot" aria-label="Live plot of bead angle and stable equilibrium"/>;
}

export default function HoopLab() {
  const [params,setParams]=useState(DEFAULTS);const pRef=useRef(params);pRef.current=params;
  const [sim,setSim]=useState(INITIAL);const sRef=useRef(sim);sRef.current=sim;
  const [running,setRunning]=useState(false);const runRef=useRef(running);runRef.current=running;
  const [speed,setSpeed]=useState(1);const speedRef=useRef(speed);speedRef.current=speed;
```

## app/page.tsx

```tsx
"use client";

import { useCallback, useEffect, useRef, useState } from "react";
import HoopLab from "./hoop-lab";

type Params = { M: number; m: number; k: number; L: number; g: number; cartDamping: number; pivotDamping: number };
type State = { x: number; vx: number; theta: number; omega: number; t: number };
type Sample = { t: number; x: number; theta: number; energy: number };

const DEFAULTS: Params = { M: 2, m: 0.45, k: 8, L: 1.25, g: 9.81, cartDamping: 0.12, pivotDamping: 0.025 };
const INITIAL: State = { x: 0.42, vx: 0, theta: 28 * Math.PI / 180, omega: 0, t: 0 };

function accelerations(s: State, p: Params) {
  const { x, vx, theta: q, omega: w } = s;
  const c = Math.cos(q), sn = Math.sin(q);
  const A = p.M + p.m;
  const B = p.m * p.L * c;
  const D = p.L * c;
  const E = p.L * p.L;
  const C = -p.k * x - p.cartDamping * vx + p.m * p.L * w * w * sn;
  const F = -p.g * p.L * sn - (p.pivotDamping / p.m) * w;
  const det = A * E - B * D;
  return { ax: (C * E - B * F) / det, alpha: (A * F - D * C) / det };
}

function derivative(s: State, p: Params) {
  const a = accelerations(s, p);
  return { x: s.vx, vx: a.ax, theta: s.omega, omega: a.alpha, t: 1 };
}

function shifted(s: State, d: ReturnType<typeof derivative>, h: number): State {
  return { x: s.x + d.x * h, vx: s.vx + d.vx * h, theta: s.theta + d.theta * h, omega: s.omega + d.omega * h, t: s.t + h };
}

function rk4(s: State, dt: number, p: Params): State {
  const a = derivative(s, p);
  const b = derivative(shifted(s, a, dt / 2), p);
  const c = derivative(shifted(s, b, dt / 2), p);
  const d = derivative(shifted(s, c, dt), p);
  return {
    x: s.x + dt * (a.x + 2 * b.x + 2 * c.x + d.x) / 6,
    vx: s.vx + dt * (a.vx + 2 * b.vx + 2 * c.vx + d.vx) / 6,
    theta: s.theta + dt * (a.theta + 2 * b.theta + 2 * c.theta + d.theta) / 6,
    omega: s.omega + dt * (a.omega + 2 * b.omega + 2 * c.omega + d.omega) / 6,
    t: s.t + dt,
  };
}

function energy(s: State, p: Params) {
  const kinetic = 0.5 * (p.M + p.m) * s.vx ** 2 + p.m * p.L * s.vx * s.omega * Math.cos(s.theta) + 0.5 * p.m * p.L ** 2 * s.omega ** 2;
  return kinetic + 0.5 * p.k * s.x ** 2 + p.m * p.g * p.L * (1 - Math.cos(s.theta));
}

type P3 = { x: number; y: number; z: number };

function Scene({ sim, params, running, onSetState }: { sim: State; params: Params; running: boolean; onSetState: (patch: Partial<State>) => void }) {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const view = useRef({ yaw: -0.62, pitch: 0.42, zoom: 1, dragging: false, px: 0, py: 0 });
  const dragMode = useRef<"orbit" | "cart" | "bob" | null>(null);
  const hit = useRef({ cartX: 0, cartY: 0, bobX: 0, bobY: 0 });

  const draw = useCallback(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;
    const rect = canvas.getBoundingClientRect();
    const dpr = Math.min(devicePixelRatio || 1, 2);
    if (canvas.width !== Math.round(rect.width * dpr) || canvas.height !== Math.round(rect.height * dpr)) {
      canvas.width = Math.round(rect.width * dpr); canvas.height = Math.round(rect.height * dpr);
    }
    const ctx = canvas.getContext("2d")!;
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
    const W = rect.width, H = rect.height;
    ctx.clearRect(0, 0, W, H);
    const v = view.current;
    const scale = Math.min(W / 7.5, H / 4.8) * v.zoom;
    const cy = Math.cos(v.yaw), sy = Math.sin(v.yaw), cp = Math.cos(v.pitch), sp = Math.sin(v.pitch);
    const project = (p: P3) => {
      const x1 = cy * p.x - sy * p.z;
      const z1 = sy * p.x + cy * p.z;
      const y2 = cp * p.y - sp * z1;
      const z2 = sp * p.y + cp * z1;
      const persp = 1 / Math.max(0.58, 1 + z2 * 0.055);
      return { x: W * 0.54 + x1 * scale * persp, y: H * 0.53 - y2 * scale * persp, z: z2, s: persp };
    };
    const line = (pts: P3[], color: string, width = 1, dash: number[] = []) => {
      const q = pts.map(project); ctx.beginPath(); ctx.moveTo(q[0].x, q[0].y); q.slice(1).forEach(a => ctx.lineTo(a.x, a.y));
      ctx.strokeStyle = color; ctx.lineWidth = width; ctx.setLineDash(dash); ctx.stroke(); ctx.setLineDash([]);
    };
    const poly = (pts: P3[], fill: string, stroke = "rgba(255,255,255,.12)") => {
      const q = pts.map(project); ctx.beginPath(); ctx.moveTo(q[0].x, q[0].y); q.slice(1).forEach(a => ctx.lineTo(a.x, a.y)); ctx.closePath(); ctx.fillStyle = fill; ctx.fill(); ctx.strokeStyle = stroke; ctx.lineWidth = 1; ctx.stroke();
    };

    const bg = ctx.createLinearGradient(0, 0, 0, H); bg.addColorStop(0, "#11181d"); bg.addColorStop(1, "#070a0c"); ctx.fillStyle = bg; ctx.fillRect(0, 0, W, H);
    for (let gx = -4; gx <= 4; gx += .5) line([{x:gx,y:-.48,z:-2},{x:gx,y:-.48,z:2}], gx === 0 ? "rgba(93,216,255,.18)" : "rgba(255,255,255,.045)", gx === 0 ? 1.4 : 1);
    for (let gz = -2; gz <= 2; gz += .5) line([{x:-4,y:-.48,z:gz},{x:4,y:-.48,z:gz}], "rgba(255,255,255,.045)");

    poly([{x:-3.35,y:-.48,z:-.75},{x:-3.35,y:1.42,z:-.75},{x:-3.35,y:1.42,z:.75},{x:-3.35,y:-.48,z:.75}], "#151d22", "#30404a");
    line([{x:-3.35,y:.34,z:-.62},{x:-3.35,y:.34,z:.62}], "#94a3ad", 4);
    line([{x:-3.25,y:-.27,z:-.4},{x:3.55,y:-.27,z:-.4}], "#75838c", 5);
    line([{x:-3.25,y:-.27,z:.4},{x:3.55,y:-.27,z:.4}], "#3d4b53", 5);
```
