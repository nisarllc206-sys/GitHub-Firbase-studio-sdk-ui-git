import { useState, useRef, useEffect, useCallback } from "react";

/* ─── Google Fonts ── */
const FontLoader = () => {
  useEffect(() => {
    const l = document.createElement("link");
    l.href = "https://fonts.googleapis.com/css2?family=Orbitron:wght@400;600;700;900&family=Rajdhani:wght@300;400;500;600;700&family=Share+Tech+Mono&display=swap";
    l.rel = "stylesheet";
    document.head.appendChild(l);
  }, []);
  return null;
};

/* ─── Constants ── */
const CAT_COLORS = {
  AI:"#00f5ff", PDF:"#a855f7", OCR:"#f59e0b", QR:"#10b981",
  MEDIA:"#f43f5e", FILES:"#6366f1", TEXT:"#fb923c", UTIL:"#34d399"
};

const TOOLS = [
  {id:"ai-chat",     icon:"🤖", label:"AI Chat",          cat:"AI",   desc:"Claude-powered assistant"},
  {id:"ai-text",     icon:"✍️", label:"AI Text Gen",      cat:"AI",   desc:"Generate any content"},
  {id:"ai-summary",  icon:"📋", label:"Doc Summary",       cat:"AI",   desc:"Summarize documents"},
  {id:"ai-improve",  icon:"💎", label:"AI Writing",        cat:"AI",   desc:"Improve your writing"},
  {id:"ocr",         icon:"👁️", label:"OCR Scanner",       cat:"OCR",  desc:"Extract text from image"},
  {id:"pdf-info",    icon:"📄", label:"PDF Reader",        cat:"PDF",  desc:"Read & analyze PDFs"},
  {id:"pdf-merge",   icon:"🔗", label:"Merge PDF",         cat:"PDF",  desc:"Combine PDF files"},
  {id:"pdf-compress",icon:"🗜️", label:"Compress PDF",      cat:"PDF",  desc:"Reduce PDF size"},
  {id:"qr-gen",      icon:"📱", label:"QR Generator",      cat:"QR",   desc:"Create QR codes"},
  {id:"qr-scan",     icon:"🔍", label:"QR Scanner",        cat:"QR",   desc:"Decode QR from image"},
  {id:"img-compress",icon:"🖼️", label:"Img Compressor",    cat:"MEDIA",desc:"Compress images"},
  {id:"img-convert", icon:"🔄", label:"Format Convert",    cat:"MEDIA",desc:"Convert image format"},
  {id:"file-mgr",    icon:"📁", label:"File Manager",      cat:"FILES",desc:"Manage your files"},
  {id:"zip",         icon:"📦", label:"Zip Tool",          cat:"FILES",desc:"Zip & unzip files"},
  {id:"word-count",  icon:"📊", label:"Word Counter",       cat:"TEXT", desc:"Analyze text stats"},
  {id:"base64",      icon:"🔐", label:"Base64 Codec",       cat:"UTIL", desc:"Encode / Decode Base64"},
  {id:"color-pick",  icon:"🎨", label:"Color Picker",       cat:"UTIL", desc:"Pick & generate palettes"},
  {id:"pass-gen",    icon:"🛡️", label:"Password Gen",       cat:"UTIL", desc:"Secure password generator"},
  {id:"pdf-split",   icon:"✂️", label:"Split PDF",          cat:"PDF",  desc:"Split PDF pages"},
  {id:"pdf-rotate",  icon:"🔃", label:"Rotate PDF",         cat:"PDF",  desc:"Rotate PDF pages"},
  {id:"watermark",   icon:"💧", label:"Watermark",          cat:"PDF",  desc:"Add watermark"},
  {id:"img-pdf",     icon:"🖼️", label:"Image→PDF",          cat:"PDF",  desc:"Convert images to PDF"},
  {id:"img-video",   icon:"🎬", label:"Image→Video",        cat:"MEDIA",desc:"Create slideshow video"},
  {id:"vid-img",     icon:"🎞️", label:"Video→Image",        cat:"MEDIA",desc:"Extract video frames"},
];

/* ─── Claude API ── */
async function callClaude(messages, system="") {
  const r = await fetch("https://api.anthropic.com/v1/messages",{
    method:"POST", headers:{"Content-Type":"application/json"},
    body:JSON.stringify({ model:"claude-sonnet-4-20250514", max_tokens:1000,
      system: system||"You are a helpful AI assistant inside the Nexus Ultra AI PDF Super Toolkit. Be concise and practical.",
      messages })
  });
  const d = await r.json();
  return d.content?.[0]?.text || "No response";
}

/* ─── QR Matrix (real-ish pattern) ── */
function drawQR(canvas, text, fg="#00f5ff", bg="#050810") {
  const S = 25, cell = Math.floor(canvas.width / S);
  const ctx = canvas.getContext("2d");
  ctx.fillStyle = bg; ctx.fillRect(0,0,canvas.width,canvas.height);
  let h = 5381;
  for (let i=0;i<text.length;i++) h = ((h<<5)+h)^text.charCodeAt(i);
  h = h>>>0;
  const m = Array.from({length:S},()=>Array(S).fill(0));
  // Finder patterns
  const fin=(r,c)=>{
    for(let i=0;i<7;i++) for(let j=0;j<7;j++) {
      if(i===0||i===6||j===0||j===6||(i>=2&&i<=4&&j>=2&&j<=4)) m[r+i][c+j]=1;
      else m[r+i][c+j]=0;
    }
  };
  fin(0,0); fin(0,S-7); fin(S-7,0);
  // Timing
  for(let i=8;i<S-8;i++) m[6][i]=m[i][6]=i%2===0?1:0;
  // Alignment pattern
  const ap=[[S-9,S-9]];
  ap.forEach(([r,c])=>{ for(let i=-2;i<=2;i++) for(let j=-2;j<=2;j++) m[r+i][c+j]=(Math.abs(i)===2||Math.abs(j)===2||i===0&&j===0)?1:0; });
  // Data
  let seed=h;
  for(let i=0;i<S;i++) for(let j=0;j<S;j++) {
    if(m[i][j]!==0) continue;
    const skip=(i<9&&(j<9||j>S-9))||(i>S-9&&j<9)||(i===6||j===6);
    if(!skip){ seed^=seed<<13; seed^=seed>>7; seed^=seed<<17; m[i][j]=(seed&1); }
  }
  // Draw
  for(let i=0;i<S;i++) for(let j=0;j<S;j++) {
    if(m[i][j]) {
      const isKey=(i<8&&(j<8||j>S-8))||(i>S-8&&j<8);
      ctx.fillStyle = isKey ? "#ffffff" : fg;
      ctx.shadowColor = isKey ? "#fff" : fg;
      ctx.shadowBlur = isKey ? 0 : 6;
      const x=j*cell+1, y=i*cell+1, s=cell-2;
      const r=Math.min(3,s*0.3);
      ctx.beginPath();
      ctx.moveTo(x+r,y); ctx.lineTo(x+s-r,y);
      ctx.quadraticCurveTo(x+s,y,x+s,y+r);
      ctx.lineTo(x+s,y+s-r); ctx.quadraticCurveTo(x+s,y+s,x+s-r,y+s);
      ctx.lineTo(x+r,y+s); ctx.quadraticCurveTo(x,y+s,x,y+s-r);
      ctx.lineTo(x,y+r); ctx.quadraticCurveTo(x,y,x+r,y);
      ctx.closePath(); ctx.fill();
    }
  }
  ctx.shadowBlur=0;
}

/* ─── Animated Hero Canvas ── */
function HeroCanvas() {
  const ref = useRef();
  useEffect(()=>{
    const c = ref.current; if(!c) return;
    const ctx = c.getContext("2d");
    const W=c.width=c.offsetWidth, H=c.height=80;
    const pts = Array.from({length:60},()=>({
      x:Math.random()*W, y:Math.random()*H,
      vx:(Math.random()-.5)*.4, vy:(Math.random()-.5)*.4,
      r:Math.random()*2+1
    }));
    let raf;
    const draw=()=>{
      ctx.clearRect(0,0,W,H);
      pts.forEach(p=>{ p.x+=p.vx; p.y+=p.vy; if(p.x<0||p.x>W) p.vx*=-1; if(p.y<0||p.y>H) p.vy*=-1; });
      pts.forEach((a,i)=>pts.slice(i+1).forEach(b=>{
        const d=Math.hypot(a.x-b.x,a.y-b.y);
        if(d<90){ ctx.strokeStyle=`rgba(0,245,255,${(1-d/90)*0.25})`; ctx.lineWidth=.5; ctx.beginPath(); ctx.moveTo(a.x,a.y); ctx.lineTo(b.x,b.y); ctx.stroke(); }
      }));
      pts.forEach(p=>{ ctx.fillStyle="#00f5ff88"; ctx.beginPath(); ctx.arc(p.x,p.y,p.r,0,Math.PI*2); ctx.fill(); });
      raf=requestAnimationFrame(draw);
    };
    draw();
    return ()=>cancelAnimationFrame(raf);
  },[]);
  return <canvas ref={ref} style={{position:"absolute",inset:0,width:"100%",height:"100%",opacity:.5}} />;
}

/* ─── CSS ── */
const CSS = `
*{box-sizing:border-box;margin:0;padding:0;}
:root{
  --bg:#030712;--surface:#0d1117;--surface2:#0f1520;--border:#1a2235;
  --cyan:#00f5ff;--purple:#a855f7;--gold:#f59e0b;--green:#10b981;
  --red:#f43f5e;--orange:#fb923c;--teal:#34d399;
  --text:#e2e8f0;--muted:#4a5568;
  --glow-c:0 0 24px rgba(0,245,255,0.45);
  --glow-p:0 0 24px rgba(168,85,247,0.45);
}
body{background:var(--bg);color:var(--text);font-family:'Rajdhani',sans-serif;}
::-webkit-scrollbar{width:4px;}::-webkit-scrollbar-track{background:#0a0f1a;}::-webkit-scrollbar-thumb{background:#00f5ff33;border-radius:4px;}

@keyframes scanline{0%{top:-4px;}100%{top:100vh;}}
@keyframes float{0%,100%{transform:translateY(0);}50%{transform:translateY(-7px);}}
@keyframes appear{from{opacity:0;transform:translateY(18px);}to{opacity:1;transform:translateY(0);}}
@keyframes pulse{0%,100%{opacity:.4;}50%{opacity:1;}}
@keyframes spin{from{transform:rotate(0)}to{transform:rotate(360deg)}}
@keyframes flicker{0%,100%{opacity:1;}91%{opacity:1;}92%{opacity:.3;}94%{opacity:1;}96%{opacity:.7;}98%{opacity:1;}}
@keyframes slideIn{from{opacity:0;transform:translateX(-16px);}to{opacity:1;transform:translateX(0);}}
@keyframes heroText{0%{background-position:0% 50%;}50%{background-position:100% 50%;}100%{background-position:0% 50%;}}
@keyframes blink{0%,100%{opacity:1;}50%{opacity:0;}}
@keyframes dotBounce{0%,80%,100%{transform:translateY(0);}40%{transform:translateY(-8px);}}
@keyframes glow-pulse{0%,100%{box-shadow:0 0 8px currentColor;}50%{box-shadow:0 0 24px currentColor,0 0 48px currentColor;}}

.scanline{position:fixed;left:0;right:0;height:3px;background:linear-gradient(transparent,rgba(0,245,255,.15),transparent);animation:scanline 10s linear infinite;pointer-events:none;z-index:999;}
.grid-bg{position:fixed;inset:0;background-image:linear-gradient(rgba(0,245,255,.025) 1px,transparent 1px),linear-gradient(90deg,rgba(0,245,255,.025) 1px,transparent 1px);background-size:44px 44px;pointer-events:none;z-index:0;}
.vignette{position:fixed;inset:0;background:radial-gradient(ellipse at center,transparent 60%,rgba(0,0,0,.7));pointer-events:none;z-index:0;}

/* Header */
.header{position:sticky;top:0;z-index:100;background:rgba(3,7,18,.94);backdrop-filter:blur(24px);border-bottom:1px solid rgba(0,245,255,.12);padding:0 20px;height:60px;display:flex;align-items:center;justify-content:space-between;gap:12px;}
.logo{font-family:'Orbitron',monospace;font-weight:900;font-size:17px;color:var(--cyan);text-shadow:var(--glow-c);animation:flicker 5s infinite;letter-spacing:2px;cursor:pointer;white-space:nowrap;}
.logo span{color:var(--purple);}
.nav-pills{display:flex;gap:3px;}
.nav-pill{padding:6px 14px;border-radius:6px;border:1px solid transparent;font-family:'Rajdhani',sans-serif;font-size:13px;font-weight:600;cursor:pointer;transition:all .2s;background:transparent;color:var(--muted);letter-spacing:1px;}
.nav-pill:hover{color:var(--cyan);border-color:rgba(0,245,255,.25);}
.nav-pill.active{color:var(--cyan);border-color:var(--cyan);background:rgba(0,245,255,.07);box-shadow:0 0 12px rgba(0,245,255,.15);}
.badge{font-family:'Share Tech Mono',monospace;font-size:10px;color:var(--green);background:rgba(16,185,129,.1);border:1px solid rgba(16,185,129,.3);padding:3px 9px;border-radius:4px;white-space:nowrap;}

/* Main */
.main{flex:1;position:relative;z-index:1;padding:20px;}

/* Hero */
.hero{position:relative;text-align:center;padding:36px 16px 28px;overflow:hidden;border-radius:16px;margin-bottom:24px;border:1px solid rgba(0,245,255,.1);background:linear-gradient(180deg,rgba(0,245,255,.04) 0%,transparent 100%);}
.hero-title{font-family:'Orbitron',monospace;font-size:clamp(22px,5vw,48px);font-weight:900;background:linear-gradient(270deg,#00f5ff,#a855f7,#f59e0b,#00f5ff);background-size:300% 300%;-webkit-background-clip:text;-webkit-text-fill-color:transparent;background-clip:text;animation:heroText 5s ease infinite;line-height:1.1;margin-bottom:10px;}
.hero-sub{font-size:14px;color:var(--muted);letter-spacing:1.5px;font-weight:400;}
.hero-stats{display:flex;justify-content:center;gap:24px;margin-top:20px;flex-wrap:wrap;}
.hero-stat{text-align:center;}
.hero-stat-n{font-family:'Orbitron',monospace;font-size:26px;font-weight:700;color:var(--cyan);text-shadow:0 0 16px rgba(0,245,255,.5);}
.hero-stat-l{font-size:11px;color:var(--muted);letter-spacing:1px;margin-top:2px;}

/* Filters */
.cat-filter{display:flex;gap:6px;flex-wrap:wrap;justify-content:center;margin:0 0 20px;}
.cat-btn{padding:5px 14px;border-radius:20px;border:1px solid var(--border);font-family:'Rajdhani',sans-serif;font-size:12px;font-weight:600;cursor:pointer;transition:all .2s;background:transparent;color:var(--muted);}
.cat-btn:hover{color:var(--text);border-color:var(--muted);}
.cat-btn.active{border-color:currentColor;background:rgba(255,255,255,.05);}

/* Tool Grid */
.tools-grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(150px,1fr));gap:10px;max-width:1200px;margin:0 auto;}
.tool-card{background:var(--surface);border:1px solid var(--border);border-radius:12px;padding:18px 14px;cursor:pointer;transition:all .25s;position:relative;overflow:hidden;animation:appear .4s ease both;display:flex;flex-direction:column;align-items:center;gap:8px;text-align:center;}
.tool-card::after{content:'';position:absolute;inset:0;border-radius:12px;opacity:0;transition:opacity .3s;background:radial-gradient(ellipse at 50% 0%,var(--hue,rgba(0,245,255,.08)),transparent 70%);}
.tool-card:hover{transform:translateY(-5px);border-color:var(--hue-solid,rgba(0,245,255,.5));}
.tool-card:hover::after{opacity:1;}
.tool-card:hover .tool-icon{animation:float .8s ease-in-out infinite;}
.tool-icon{font-size:26px;line-height:1;transition:transform .2s;}
.tool-name{font-family:'Rajdhani',sans-serif;font-weight:700;font-size:13px;color:var(--text);letter-spacing:.4px;}
.tool-desc{font-size:10px;color:var(--muted);}
.tool-dot{width:5px;height:5px;border-radius:50%;position:absolute;top:9px;right:9px;}

/* Panel */
.panel{max-width:880px;margin:0 auto;animation:appear .35s ease;}
.panel-hd{display:flex;align-items:center;gap:14px;margin-bottom:24px;}
.back-btn{width:36px;height:36px;border-radius:8px;border:1px solid var(--border);background:transparent;color:var(--cyan);cursor:pointer;display:flex;align-items:center;justify-content:center;font-size:18px;transition:all .2s;flex-shrink:0;}
.back-btn:hover{border-color:var(--cyan);background:rgba(0,245,255,.1);box-shadow:0 0 12px rgba(0,245,255,.2);}
.panel-title{font-family:'Orbitron',monospace;font-size:20px;font-weight:700;color:var(--cyan);}
.panel-sub{font-size:12px;color:var(--muted);margin-top:2px;}

/* Card */
.card{background:var(--surface);border:1px solid var(--border);border-radius:12px;padding:18px;}
.card-glow{border-color:rgba(0,245,255,.2);box-shadow:inset 0 0 48px rgba(0,245,255,.02),0 0 32px rgba(0,245,255,.04);}

/* Labels */
.slabel{font-family:'Share Tech Mono',monospace;font-size:10px;color:var(--cyan);letter-spacing:2px;text-transform:uppercase;margin-bottom:8px;display:block;}

/* Inputs */
.inp,.ta,select.inp{width:100%;background:var(--surface2);border:1px solid var(--border);border-radius:8px;padding:11px 15px;color:var(--text);font-family:'Rajdhani',sans-serif;font-size:14px;font-weight:500;outline:none;transition:border-color .2s;}
.inp:focus,.ta:focus{border-color:rgba(0,245,255,.4);box-shadow:0 0 0 3px rgba(0,245,255,.06);}
.ta{resize:vertical;min-height:110px;line-height:1.6;}
.inp::placeholder,.ta::placeholder{color:var(--muted);}
select.inp option{background:#0d1117;}

/* Buttons */
.btn{display:inline-flex;align-items:center;justify-content:center;gap:7px;padding:10px 22px;border-radius:8px;font-family:'Rajdhani',sans-serif;font-weight:700;font-size:14px;letter-spacing:.5px;cursor:pointer;transition:all .2s;border:none;outline:none;white-space:nowrap;}
.btn-p{background:linear-gradient(135deg,#00c8d4,#0090a0);color:#000;box-shadow:0 4px 18px rgba(0,245,255,.25);}
.btn-p:hover{transform:translateY(-2px);box-shadow:0 8px 28px rgba(0,245,255,.4);}
.btn-p:disabled{opacity:.45;cursor:not-allowed;transform:none;}
.btn-s{background:var(--surface2);border:1px solid var(--border);color:var(--text);}
.btn-s:hover{border-color:var(--cyan);color:var(--cyan);}
.btn-g{background:transparent;border:1px solid rgba(0,245,255,.3);color:var(--cyan);}
.btn-g:hover{background:rgba(0,245,255,.07);}
.btn-d{background:rgba(244,63,94,.12);border:1px solid rgba(244,63,94,.35);color:#f43f5e;}
.btn-sm{padding:6px 14px;font-size:12px;}

/* Chat */
.chat-win{height:400px;overflow-y:auto;display:flex;flex-direction:column;gap:10px;padding:14px;background:var(--surface2);border:1px solid var(--border);border-radius:12px;margin-bottom:14px;}
.msg{max-width:82%;padding:11px 15px;border-radius:12px;font-size:13px;line-height:1.65;animation:slideIn .3s ease;}
.msg-u{align-self:flex-end;background:linear-gradient(135deg,rgba(0,180,200,.22),rgba(0,100,120,.12));border:1px solid rgba(0,245,255,.18);border-radius:12px 12px 2px 12px;}
.msg-a{align-self:flex-start;background:var(--surface);border:1px solid var(--border);border-radius:12px 12px 12px 2px;}
.mlabel{font-family:'Share Tech Mono',monospace;font-size:9px;letter-spacing:1px;margin-bottom:5px;}
.typing-wrap{display:flex;gap:5px;align-items:center;height:16px;}
.dot{width:7px;height:7px;border-radius:50%;background:var(--cyan);display:inline-block;animation:dotBounce 1.2s ease-in-out infinite;}
.dot:nth-child(2){animation-delay:.2s;}
.dot:nth-child(3){animation-delay:.4s;}

/* Upload */
.upload-z{border:2px dashed var(--border);border-radius:12px;padding:36px;text-align:center;cursor:pointer;transition:all .2s;background:var(--surface2);}
.upload-z:hover,.upload-z.drag{border-color:var(--cyan);background:rgba(0,245,255,.03);}
.uicon{font-size:38px;margin-bottom:10px;display:block;}

/* Result */
.rbox{background:var(--surface2);border:1px solid rgba(0,245,255,.18);border-radius:10px;padding:15px;font-family:'Share Tech Mono',monospace;font-size:12px;line-height:1.75;white-space:pre-wrap;word-break:break-word;max-height:320px;overflow-y:auto;color:#9decec;}
.racts{display:flex;gap:7px;margin-top:10px;flex-wrap:wrap;}

/* QR */
.qr-wrap{display:flex;flex-direction:column;align-items:center;gap:14px;padding:22px;background:var(--surface2);border:1px solid rgba(0,245,255,.2);border-radius:12px;}
.qr-frame{padding:14px;background:#050810;border:2px solid var(--cyan);border-radius:14px;box-shadow:0 0 32px rgba(0,245,255,.3),inset 0 0 20px rgba(0,245,255,.05);position:relative;}
.qr-corner{position:absolute;width:14px;height:14px;border-color:var(--purple);border-style:solid;}
.qr-corner.tl{top:-1px;left:-1px;border-width:3px 0 0 3px;}
.qr-corner.tr{top:-1px;right:-1px;border-width:3px 3px 0 0;}
.qr-corner.bl{bottom:-1px;left:-1px;border-width:0 0 3px 3px;}
.qr-corner.br{bottom:-1px;right:-1px;border-width:0 3px 3px 0;}

/* Stats */
.stats-row{display:grid;grid-template-columns:repeat(auto-fit,minmax(110px,1fr));gap:10px;margin:14px 0;}
.scard{background:var(--surface2);border:1px solid var(--border);border-radius:10px;padding:13px;text-align:center;}
.scard-n{font-family:'Orbitron',monospace;font-size:20px;font-weight:700;}
.scard-l{font-size:10px;color:var(--muted);margin-top:4px;letter-spacing:.5px;}

/* File */
.fi{display:flex;align-items:center;gap:11px;padding:12px 15px;background:var(--surface2);border:1px solid var(--border);border-radius:10px;margin-bottom:7px;animation:slideIn .3s ease;}

/* Switch */
.sw-row{display:flex;align-items:center;justify-content:space-between;padding:11px 0;border-bottom:1px solid var(--border);}
.tog{width:42px;height:22px;background:var(--surface2);border:1px solid var(--border);border-radius:11px;cursor:pointer;position:relative;transition:all .2s;flex-shrink:0;}
.tog.on{background:linear-gradient(90deg,#00c8d4,#0090a0);border-color:var(--cyan);}
.tog::after{content:'';position:absolute;top:3px;left:3px;width:14px;height:14px;background:#fff;border-radius:50%;transition:transform .2s;}
.tog.on::after{transform:translateX(20px);}

/* Progress */
.pb-wrap{background:var(--surface2);border-radius:100px;height:6px;overflow:hidden;margin-top:6px;}
.pb-fill{height:100%;border-radius:100px;transition:width .5s ease;}

/* Tag */
.tag{display:inline-flex;align-items:center;padding:2px 9px;border-radius:100px;font-size:10px;font-weight:700;letter-spacing:.4px;}

/* Hex swatch */
.swatch{width:36px;height:36px;border-radius:8px;border:2px solid var(--border);cursor:pointer;transition:transform .15s;flex-shrink:0;}
.swatch:hover{transform:scale(1.15);}

/* Password */
.pass-display{font-family:'Share Tech Mono',monospace;font-size:18px;letter-spacing:3px;background:var(--surface2);border:1px solid rgba(0,245,255,.25);border-radius:10px;padding:16px;text-align:center;color:var(--cyan);text-shadow:0 0 12px rgba(0,245,255,.3);word-break:break-all;min-height:58px;display:flex;align-items:center;justify-content:center;}
.str-bar{height:8px;border-radius:100px;transition:all .4s;}

/* Word counter */
.wc-grid{display:grid;grid-template-columns:repeat(3,1fr);gap:8px;margin-top:14px;}
.wc-cell{background:var(--surface2);border:1px solid var(--border);border-radius:8px;padding:12px;text-align:center;}
.wc-num{font-family:'Orbitron',monospace;font-size:22px;font-weight:700;color:var(--cyan);}
.wc-lbl{font-size:10px;color:var