# 3dsvalor
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover, user-scalable=no">
<meta name="apple-mobile-web-app-capable" content="yes">
<title>3DS Player</title>
<style>
:root{--side:#d4325f;--key:#f7c6d9;--ink:#2a0f1a;--screen:#111}
*{box-sizing:border-box}
html,body{height:100%;margin:0;background:#1b1b1f;overflow:hidden;font-family:system-ui,-apple-system,Segoe UI,sans-serif;color:#fff;-webkit-user-select:none;user-select:none;-webkit-touch-callout:none;touch-action:none}
#app{position:fixed;inset:0;display:grid;grid-template-columns:1fr auto 1fr;padding:env(safe-area-inset-top) env(safe-area-inset-right) env(safe-area-inset-bottom) env(safe-area-inset-left)}
.side{background:var(--side);display:flex;flex-direction:column;justify-content:space-between;align-items:center;padding:7vh 0;position:relative}
.key{background:var(--key);color:var(--ink);border:.5vmin solid var(--ink);display:grid;place-items:center;font-weight:700;font-size:3.4vmin;cursor:pointer}
.key.on{background:#fff;transform:scale(.94)}
.shoulder{width:11vmin;height:11vmin;border-radius:1vmin}
.mid{display:flex;gap:2vmin;margin-bottom:2vh}
.small{width:8vmin;height:4vmin;border-radius:2vmin;font-size:1.8vmin}
#dpad{width:26vmin;height:26vmin;border-radius:50%;background:var(--key);border:.5vmin solid var(--ink);position:relative;display:grid;place-items:center}
#dpad i{position:absolute;width:30%;height:30%;background:var(--side);opacity:.0;border-radius:50%}
#dpad b{width:38%;height:38%;border-radius:50%;border:.5vmin solid var(--ink);display:grid;place-items:center;font-size:2.2vmin;text-align:center;line-height:1.1}
#dpad.u i.u,#dpad.d i.d,#dpad.l i.l,#dpad.r i.r{opacity:.55}
i.u{top:6%;left:35%}i.d{bottom:6%;left:35%}i.l{left:6%;top:35%}i.r{right:6%;top:35%}
#faces{width:30vmin;height:30vmin;position:relative}
#faces .key{position:absolute;width:10.5vmin;height:10.5vmin;border-radius:50%}
#bX{top:0;left:9.75vmin}#bB{bottom:0;left:9.75vmin}#bY{left:0;top:9.75vmin}#bA{right:0;top:9.75vmin}
#screens{align-self:center;width:min(60vw,calc(100dvh * 5 / 6));background:var(--screen)}
canvas{display:block;image-rendering:pixelated;background:#000}
#top{width:100%;aspect-ratio:400/240;pointer-events:none;border-bottom:2px solid #000}
#bot{width:80%;aspect-ratio:320/240;margin:0 auto;touch-action:none;cursor:crosshair}
#gate{position:fixed;inset:0;background:rgba(20,6,12,.92);display:grid;place-items:center;z-index:5;text-align:center;padding:24px}
#gate.hide{display:none}
#gate h1{font-size:28px;margin:0 0 8px}
#gate p{margin:0 0 20px;color:#e9b6c8;max-width:46ch}
#pick{background:var(--key);color:var(--ink);border:0;border-radius:12px;padding:14px 26px;font-size:18px;font-weight:700;cursor:pointer}
#status{margin-top:16px;min-height:1.4em;color:#fff}
#status.err{color:#ffb4b4}
#tools{position:fixed;top:calc(env(safe-area-inset-top) + 8px);right:calc(env(safe-area-inset-right) + 8px);z-index:4;display:flex;gap:8px}
#tools button{background:rgba(0,0,0,.35);color:#fff;border:1px solid rgba(255,255,255,.4);border-radius:8px;padding:6px 10px;font-size:13px}
</style>
</head>
<body>
<div id="app">
  <div class="side">
    <div class="key shoulder" data-btn="L">L</div>
    <div>
      <div class="mid"><div class="key small" data-btn="SELECT">Select</div></div>
      <div id="dpad"><i class="u"></i><i class="d"></i><i class="l"></i><i class="r"></i><b>D-Pad</b></div>
    </div>
  </div>
  <div id="screens">
    <canvas id="top" width="400" height="240"></canvas>
    <canvas id="bot" width="320" height="240"></canvas>
  </div>
  <div class="side">
    <div class="key shoulder" data-btn="R">R</div>
    <div>
      <div class="mid"><div class="key small" data-btn="START">Start</div><div class="key small" data-btn="HOME">Home</div></div>
      <div id="faces">
        <div class="key" id="bX" data-btn="X">X</div><div class="key" id="bY" data-btn="Y">Y</div>
        <div class="key" id="bA" data-btn="A">A</div><div class="key" id="bB" data-btn="B">B</div>
      </div>
    </div>
  </div>
</div>

<div id="tools"><button id="fs">Fullscreen</button><button id="again">Load game</button></div>

<div id="gate">
  <div>
    <h1>3DS Player</h1>
    <p>Choose a game file or drop it here. Supported: .3ds, .cci, .cia, .cxi, .3dsx, .elf, and .7z archives. Files stay on your device.</p>
    <button id="pick">Choose game file</button>
    <div id="status"></div>
  </div>
</div>
<input id="file" type="file" hidden>

<script>
"use strict";
/* ---------- Button bits (match the 3DS HID layout) ---------- */
const BIT={A:1,B:2,SELECT:4,START:8,RIGHT:16,LEFT:32,UP:64,DOWN:128,R:256,L:512,X:1024,Y:2048,HOME:4096};

/* ---------- Core contract ----------
   Plug a real emulator core (WASM) in with registerCore({...}):
     name                       string
     load(file, format, bytes)  Promise. `file` is the File (or null for .7z contents);
                                `bytes` is a Uint8Array when the game came out of a .7z.
     setInput(mask)             current button bitmask (see BIT)
     touch(x, y, down)          lower-screen touch in 320x240 space
     frame(topCtx, botCtx)      called every animation frame; draw 400x240 and 320x240
*/
const DemoCore={
  name:"Demo core", mask:0, t:null, game:"", n:0,
  async load(file,format,bytes){this.game=file?file.name:"(from archive)";this.format=format;},
  setInput(m){this.mask=m;},
  touch(x,y,d){this.t=d?{x,y}:null;},
  frame(tc,bc){
    this.n++;
    let g=tc.createLinearGradient(0,0,400,240);g.addColorStop(0,"#3a1d5c");g.addColorStop(1,"#d4325f");
    tc.fillStyle=g;tc.fillRect(0,0,400,240);
    tc.fillStyle="#fff";tc.font="bold 22px system-ui";tc.fillText("Upper screen (not touchable)",20,48);
    tc.font="15px system-ui";tc.fillText(this.game+"  ["+this.format+"]",20,78);
    tc.fillText("No emulator core attached yet. Frame "+this.n,20,104);
    const names=Object.keys(BIT).filter(k=>this.mask&BIT[k]);
    tc.fillText("Held: "+(names.join(", ")||"none"),20,140);
    bc.fillStyle="#f4eef1";bc.fillRect(0,0,320,240);
    bc.fillStyle="#2a0f1a";bc.font="bold 18px system-ui";bc.fillText("Lower screen (touch me)",56,36);
    if(this.t){bc.fillStyle="#d4325f";bc.beginPath();bc.arc(this.t.x,this.t.y,12,0,7);bc.fill();
      bc.fillStyle="#2a0f1a";bc.font="13px system-ui";bc.fillText(Math.round(this.t.x)+", "+Math.round(this.t.y),10,230);}
  }
};
let Core=DemoCore;
window.registerCore=c=>{Core=c;};

/* ---------- Input state ---------- */
const sources=new Map();           // source id -> bitmask
let mask=0;
function setSource(id,m){ if(m) sources.set(id,m); else sources.delete(id);
  let v=0; for(const x of sources.values()) v|=x;
  if(v!==mask){mask=v;Core.setInput(mask);} }

/* Face and shoulder buttons: multi-touch, one pointer per button */
document.querySelectorAll("[data-btn]").forEach(el=>{
  const bit=BIT[el.dataset.btn];
  el.addEventListener("pointerdown",e=>{e.preventDefault();el.setPointerCapture(e.pointerId);
    el.classList.add("on");setSource("b"+e.pointerId,bit);});
  const up=e=>{el.classList.remove("on");setSource("b"+e.pointerId,0);};
  el.addEventListener("pointerup",up);el.addEventListener("pointercancel",up);
});

/* D-pad: direction from the finger position, diagonals allowed */
const dpad=document.getElementById("dpad");
function dpadMask(e){
  const r=dpad.getBoundingClientRect(),dx=e.clientX-(r.left+r.width/2),dy=e.clientY-(r.top+r.height/2);
  const rad=r.width/2; if(Math.hypot(dx,dy)<rad*.22) return 0;
  let m=0,t=rad*.3;
  if(dx>t)m|=BIT.RIGHT; if(dx<-t)m|=BIT.LEFT; if(dy>t)m|=BIT.DOWN; if(dy<-t)m|=BIT.UP;
  return m;
}
function dpadClass(m){dpad.className=(m&BIT.UP?"u ":"")+(m&BIT.DOWN?"d ":"")+(m&BIT.LEFT?"l ":"")+(m&BIT.RIGHT?"r":"");}
dpad.addEventListener("pointerdown",e=>{e.preventDefault();dpad.setPointerCapture(e.pointerId);
  const m=dpadMask(e);setSource("d"+e.pointerId,m);dpadClass(m);});
dpad.addEventListener("pointermove",e=>{if(!dpad.hasPointerCapture(e.pointerId))return;
  const m=dpadMask(e);setSource("d"+e.pointerId,m);dpadClass(m);});
const dUp=e=>{setSource("d"+e.pointerId,0);dpadClass(0);};
dpad.addEventListener("pointerup",dUp);dpad.addEventListener("pointercancel",dUp);

/* Lower screen only: pointer position maps to the 320x240 touchscreen */
const bot=document.getElementById("bot");let touchId=null;
function toTouch(e){const r=bot.getBoundingClientRect();
  return [Math.min(319,Math.max(0,(e.clientX-r.left)/r.width*320)),Math.min(239,Math.max(0,(e.clientY-r.top)/r.height*240))];}
bot.addEventListener("pointerdown",e=>{if(touchId!==null)return;e.preventDefault();touchId=e.pointerId;
  bot.setPointerCapture(e.pointerId);const [x,y]=toTouch(e);Core.touch(x,y,true);});
bot.addEventListener("pointermove",e=>{if(e.pointerId!==touchId)return;const [x,y]=toTouch(e);Core.touch(x,y,true);});
const tEnd=e=>{if(e.pointerId!==touchId)return;touchId=null;Core.touch(0,0,false);};
bot.addEventListener("pointerup",tEnd);bot.addEventListener("pointercancel",tEnd);
/* The upper canvas has pointer-events:none, so it never reaches the core. */

/* Keyboard */
const KEYS={ArrowUp:BIT.UP,ArrowDown:BIT.DOWN,ArrowLeft:BIT.LEFT,ArrowRight:BIT.RIGHT,
  x:BIT.A,z:BIT.B,s:BIT.X,a:BIT.Y,q:BIT.L,w:BIT.R,Enter:BIT.START,Shift:BIT.SELECT,Escape:BIT.HOME};
addEventListener("keydown",e=>{const m=KEYS[e.key.length===1?e.key.toLowerCase():e.key];
  if(m){e.preventDefault();setSource("k"+e.key.toLowerCase(),m);}});
addEventListener("keyup",e=>{setSource("k"+e.key.toLowerCase(),0);});

/* Gamepad (standard mapping, Nintendo-style A/B) */
function pollPad(){
  const p=[...(navigator.getGamepads?navigator.getGamepads():[])].find(Boolean); let m=0;
  if(p){const b=i=>p.buttons[i]&&p.buttons[i].pressed;
    if(b(1))m|=BIT.A;if(b(0))m|=BIT.B;if(b(3))m|=BIT.X;if(b(2))m|=BIT.Y;
    if(b(4))m|=BIT.L;if(b(5))m|=BIT.R;if(b(8))m|=BIT.SELECT;if(b(9))m|=BIT.START;
    if(b(12))m|=BIT.UP;if(b(13))m|=BIT.DOWN;if(b(14))m|=BIT.LEFT;if(b(15))m|=BIT.RIGHT;}
  setSource("pad",m);
}

/* ---------- File loading ---------- */
const EXT={".3ds":"3ds",".cci":"cci",".cia":"cia",".cxi":"cxi",".3dsx":"3dsx",".elf":"elf"};
const extOf=n=>{const i=n.lastIndexOf(".");return i<0?"":n.slice(i).toLowerCase();};
function sniff(h){
  try{
    const u=new Uint8Array(h), s=(o,n)=>String.fromCharCode(...u.slice(o,o+n));
    if(u.length>=6&&[0x37,0x7A,0xBC,0xAF,0x27,0x1C].every((v,i)=>u[i]===v))return"7z";
    if(s(0,4)==="3DSX")return"3dsx";
    if(s(0,4)==="\x7fELF")return"elf";
    if(s(0x100,4)==="NCSD")return"cci";
    if(s(0x100,4)==="NCCH")return"cxi";
    if(new DataView(h).getUint32(0,true)===0x2020)return"cia";
  }catch(_){}
  return null;
}
const status=(t,err)=>{const s=document.getElementById("status");s.textContent=t;s.className=err?"err":"";};

/* .7z: unpack in the browser with 7z-wasm, then use the largest supported entry.
   The archive is held in memory while unpacking, so very large archives may fail on iPad. */
async function from7z(file){
  status("Loading 7z support...");
  const mod=await import("https://cdn.jsdelivr.net/npm/7z-wasm@1.0.2/+esm");
  const z=await mod.default({print(){},printErr(){}});
  status("Unpacking "+file.name+"...");
  z.FS.writeFile("/a.7z",new Uint8Array(await file.arrayBuffer()));
  z.FS.mkdir("/out");
  z.callMain(["x","/a.7z","-o/out","-y"]);
  const found=[];
  (function walk(dir){for(const n of z.FS.readdir(dir)){if(n==="."||n==="..")continue;
    const p=dir+"/"+n,st=z.FS.stat(p);
    if(z.FS.isDir(st.mode))walk(p);else if(EXT[extOf(n)])found.push({p,n,size:st.size});}})("/out");
  if(!found.length)throw new Error("No supported game file found inside the archive.");
  found.sort((a,b)=>b.size-a.size);
  const pick=found[0],data=z.FS.readFile(pick.p);
  return {name:pick.n,format:EXT[extOf(pick.n)],data};
}

async function openFile(file){
  try{
    status("Reading "+file.name+"...");
    let fmt=EXT[extOf(file.name)]||null,head=sniff(await file.slice(0,0x200).arrayBuffer());
    if(head==="7z"||extOf(file.name)===".7z"){
      const r=await from7z(file);
      await Core.load({name:r.name},r.format,r.data);
    }else{
      fmt=fmt||head;
      if(!fmt)throw new Error("Unsupported file. Choose a .3ds, .cci, .cia, .cxi, .3dsx, .elf or .7z file.");
      await Core.load(file,fmt,null);
    }
    document.getElementById("gate").classList.add("hide");status("");
  }catch(err){
    status(err.message||String(err),true);
    document.getElementById("gate").classList.remove("hide");
  }
}

const fileInput=document.getElementById("file");
document.getElementById("pick").onclick=()=>fileInput.click();
document.getElementById("again").onclick=()=>document.getElementById("gate").classList.remove("hide");
fileInput.onchange=()=>{if(fileInput.files[0])openFile(fileInput.files[0]);fileInput.value="";};
addEventListener("dragover",e=>e.preventDefault());
addEventListener("drop",e=>{e.preventDefault();const f=e.dataTransfer.files[0];if(f)openFile(f);});
document.getElementById("fs").onclick=()=>{const d=document.documentElement;(d.requestFullscreen||d.webkitRequestFullscreen||(()=>{})).call(d);};

/* Block pinch zoom and the context menu on iPad */
["gesturestart","gesturechange","contextmenu"].forEach(t=>addEventListener(t,e=>e.preventDefault()));

/* ---------- Main loop ---------- */
const tc=document.getElementById("top").getContext("2d"),bc=bot.getContext("2d");
(function loop(){pollPad();Core.frame(tc,bc);requestAnimationFrame(loop);})();
</script>
</body>
</html>
