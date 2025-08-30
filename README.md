<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<title>Fresh System Map (Pan/Zoom + Draggable Split + Table)</title>
<meta name="viewport" content="width=device-width, initial-scale=1" />
<style>
  :root {
    --bg:#0b0f1a; --panel:#f7f9fc; --border:#cbd5e1; --head:#0f172a; --headtxt:#e5e7eb;
    --rowhl:#fff5b1; --rowhl2:#fde68a; --btn:#2563eb; --btn2:#334155;
  }
  * { box-sizing: border-box; }
  html, body { height: 100%; margin: 0; font-family: system-ui, Arial, sans-serif; overflow: hidden; }

  /* Split layout */
  #split {
    position: fixed; inset: 0; display: grid;
    grid-template-columns: 2fr 6px 1fr; /* left | gutter | right */
    grid-template-rows: 100%;
  }
  #mapWrap { position: relative; background: #0e1320; overflow: hidden; }
  #mapCanvas { width: 100%; height: 100%; display: block; background: #0e1320; }
  #gutter {
    background: #0b1220; cursor: col-resize; border-left: 1px solid #111827; border-right: 1px solid #111827;
  }
  #right {
    min-width: 420px; max-width: 60vw; background: var(--panel); display:flex; flex-direction:column; border-left: 1px solid #111827;
  }

  /* Toolbar pinned at top of right pane */
  #toolbar {
    padding: .5rem .6rem; background: var(--head); color: var(--headtxt);
    display: flex; gap: .5rem; align-items: center; flex-wrap: wrap;
    position: sticky; top: 0; z-index: 5;
  }
  .btn {
    border: 1px solid var(--border); background: white; padding: .4rem .6rem;
    border-radius: .5rem; cursor: pointer; font-size: .9rem;
  }
  .btn.primary { background: var(--btn); color: white; border-color: #1d4ed8; }
  .btn.ghost { background: #f1f5f9; }
  .sep { width:1px; height:24px; background:#334155; margin:0 .25rem; }

  /* Table area */
  #tableWrap { overflow: auto; flex: 1; border-top: 1px solid var(--border); }
  table { width: 100%; border-collapse: collapse; font-size: 12px; }
  thead th {
    position: sticky; top: 0; background: #1f2937; color: white; z-index: 2;
    border: 1px solid #0f172a; padding: 4px 6px; white-space: nowrap;
  }
  td, th { border: 1px solid #dde3ee; padding: 3px 4px; text-align: left; }
  tbody tr:nth-child(odd) { background: #f8fafc; }
  tbody tr.highlight { background: var(--rowhl) !important; }
  tbody tr:hover { background: var(--rowhl2); cursor: pointer; }

  /* Paste area */
  #pastePanel { padding: .6rem; border-top: 1px solid var(--border); background: #fdfdfd; }
  #pasteArea { width: 100%; min-height: 120px; resize: vertical; font-family: ui-monospace, Menlo, Consolas, monospace; }

  /* Map HUD */
  #hud { position: absolute; right: 10px; top: 10px; background: rgba(255,255,255,.9); padding: 6px 8px; border-radius: 8px; font-size: 12px; }
  #hud div { margin: 2px 0; }
  .legend { display:flex; gap:6px; align-items:center; }
  .dot { width:12px; height:12px; border-radius:50%; display:inline-block; border:1px solid #111; }
</style>
</head>
<body>
  <div id="split">
    <div id="mapWrap">
      <canvas id="mapCanvas"></canvas>
      <div id="hud">
        <div><b>Pan:</b> drag empty space · <b>Zoom:</b> wheel · <b>Drag node:</b> drag circle</div>
        <div class="legend"><span class="dot" style="background:#22c55e"></span> exact link</div>
        <div class="legend"><span class="dot" style="background:#ef4444"></span> stretched link</div>
      </div>
    </div>
    <div id="gutter" title="Drag to resize"></div>
    <div id="right">
      <div id="toolbar">
        <button class="btn primary" id="btnBuild">Build / Rebuild Map</button>
        <button class="btn" id="btnAdjust">Adjust Angles (±15°)</button>
        <span class="sep"></span>
        <button class="btn ghost" id="btnResetView">Reset View</button>
        <button class="btn ghost" id="btnFit">Fit to View</button>
        <span style="flex:1"></span>
        <button class="btn" id="btnExport">Export CSV</button>
      </div>

      <div id="tableWrap">
        <table id="sysTable">
          <thead>
            <tr id="hdr"></tr>
          </thead>
          <tbody id="tbody"></tbody>
        </table>
      </div>

      <div id="pastePanel">
        <div style="margin-bottom:6px;font-size:12px;color:#475569">
          Paste <b>tab-separated</b> data with headers exactly:
          <code>Bonus, System Name, Angle2one, Dist2one, Bonus2, Connects1, Angle2two, Dist2two, Bonus3, Connects2, Angle2three, Dist2three, Bonus4, Connects3, Angle2four, Dist2four, Bonus5, Connects4, Angle2five, Dist2five, Bonus6, Connects5</code>
        </div>
        <textarea id="pasteArea" placeholder="Paste your rows here..."></textarea>
        <div style="margin-top:8px;display:flex;gap:.5rem;flex-wrap:wrap">
          <button class="btn primary" id="btnApply">Apply Pasted Data</button>
        </div>
      </div>
    </div>
  </div>

<script>
/* ========================= Constants & Headers ========================= */
const HEADERS = [
  "Bonus","System Name","Angle2one","Dist2one","Bonus2","Connects1",
  "Angle2two","Dist2two","Bonus3","Connects2",
  "Angle2three","Dist2three","Bonus4","Connects3",
  "Angle2four","Dist2four","Bonus5","Connects4",
  "Angle2five","Dist2five","Bonus6","Connects5"
];
// groups per row
const GROUPS = [
  {ang:2, dist:3, bonus:0, conn:5},
  {ang:6, dist:7, bonus:4, conn:9},
  {ang:10,dist:11,bonus:8, conn:13},
  {ang:14,dist:15,bonus:12,conn:17},
  {ang:18,dist:19,bonus:16,conn:21}
];
const RADIUS = 5;               // node radius
const LINK_TOL = 5;             // px tolerance for "exact"

/* ========================= State ========================= */
let rows = [];                  // array of arrays (table)
let systems = new Map();        // name -> {x,y,connections:[]}
let selectedName = null;

/* ========================= Split / Resizer ========================= */
(function initSplit(){
  const split = document.getElementById('split');
  const gutter = document.getElementById('gutter');
  let dragging = false;

  gutter.addEventListener('mousedown', e => { dragging = true; e.preventDefault(); });
  window.addEventListener('mousemove', e => {
    if (!dragging) return;
    const total = window.innerWidth;
    const leftW = Math.min(Math.max(e.clientX, 280), total - 360); // bounds
    const rightW = total - leftW - gutter.offsetWidth;
    split.style.gridTemplateColumns = `${leftW}px ${gutter.offsetWidth}px ${rightW}px`;
    sizeCanvas();
    draw();
  });
  window.addEventListener('mouseup', ()=> dragging=false);
})();

/* ========================= Canvas & Viewport ========================= */
const canvas = document.getElementById('mapCanvas');
const ctx = canvas.getContext('2d', {alpha:false});
let view = {x: 0, y: 0, scale: 1}; // world -> screen: S = (W - view) * scale

function sizeCanvas(){
  const rect = canvas.parentElement.getBoundingClientRect();
  canvas.width = Math.max(100, rect.width);
  canvas.height = Math.max(100, rect.height);
}
window.addEventListener('resize', ()=>{ sizeCanvas(); draw(); });
sizeCanvas();

function worldToScreen(wx, wy){
  return {
    x: (wx - view.x) * view.scale + canvas.width/2,
    y: (wy - view.y) * view.scale + canvas.height/2
  };
}
function screenToWorld(sx, sy){
  return {
    x: (sx - canvas.width/2) / view.scale + view.x,
    y: (sy - canvas.height/2) / view.scale + view.y
  };
}

/* ========================= Map Interactions ========================= */
let isPanning=false, panStart=null, panOrigin=null;
let isDraggingNode=false, dragNode=null, dragOffset=null;

canvas.addEventListener('mousedown', e=>{
  const pos = screenToWorld(e.offsetX, e.offsetY);
  const n = findNodeAt(pos.x, pos.y, RADIUS * 2 / view.scale);
  if (n){
    isDraggingNode = true; dragNode = n;
    dragOffset = {dx: n.x - pos.x, dy: n.y - pos.y};
  } else {
    isPanning = true; panStart = {sx: e.clientX, sy: e.clientY}; panOrigin = {...view};
  }
});

canvas.addEventListener('mousemove', e=>{
  if (isPanning){
    const dx = (e.clientX - panStart.sx) / view.scale;
    const dy = (e.clientY - panStart.sy) / view.scale;
    view.x = panOrigin.x - dx; view.y = panOrigin.y - dy;
    draw();
  } else if (isDraggingNode && dragNode){
    const pos = screenToWorld(e.offsetX, e.offsetY);
    dragNode.x = pos.x + dragOffset.dx;
    dragNode.y = pos.y + dragOffset.dy;
    draw();
  }
});

window.addEventListener('mouseup', ()=>{
  isPanning=false; isDraggingNode=false; dragNode=null; dragOffset=null;
});

canvas.addEventListener('wheel', e=>{
  e.preventDefault();
  const mouse = screenToWorld(e.offsetX, e.offsetY);
  const factor = Math.pow(1.0015, -e.deltaY);
  const newScale = Math.min(6, Math.max(0.2, view.scale * factor));
  const k = newScale / view.scale;
  // zoom around mouse point
  view.x = mouse.x - (mouse.x - view.x) * k;
  view.y = mouse.y - (mouse.y - view.y) * k;
  view.scale = newScale;
  draw();
},{passive:false});

canvas.addEventListener('dblclick', e=>{
  const mouse = screenToWorld(e.offsetX, e.offsetY);
  const k = e.shiftKey ? 1/1.5 : 1.5;
  const newScale = Math.min(6, Math.max(0.2, view.scale * k));
  const ratio = newScale / view.scale;
  view.x = mouse.x - (mouse.x - view.x) * ratio;
  view.y = mouse.y - (mouse.y - view.y) * ratio;
  view.scale = newScale;
  draw();
});

canvas.addEventListener('click', e=>{
  const pos = screenToWorld(e.offsetX, e.offsetY);
  const n = findNodeAt(pos.x, pos.y, RADIUS * 2 / view.scale);
  if (n){
    selectSystem(n.name, true);
  }
});

/* ========================= Table Rendering & Events ========================= */
const hdrEl = document.getElementById('hdr');
const tbodyEl = document.getElementById('tbody');
HEADERS.forEach(h=>{ const th=document.createElement('th'); th.textContent=h; hdrEl.appendChild(th); });

function renderTable(){
  tbodyEl.innerHTML = "";
  rows.forEach((r, idx)=>{
    const tr = document.createElement('tr'); tr.dataset.name = (r[1]||"").trim();
    r.forEach((cell,i)=>{
      const td = document.createElement('td'); td.textContent = cell ?? ""; tr.appendChild(td);
    });
    // REMOVED: hover to center functionality
    // Only center on row click, not hover
    tr.addEventListener('click', ()=>{
      selectSystem(tr.dataset.name, true);
    });
    tbodyEl.appendChild(tr);
  });
}

function highlightRow(name){
  document.querySelectorAll('#tbody tr').forEach(tr=>tr.classList.remove('highlight'));
  const row = document.querySelector(`#tbody tr[data-name="${CSS.escape(name)}"]`);
  if (row){
    row.classList.add('highlight');
    // ensure visible
    const wrap = document.getElementById('tableWrap');
    const rb = row.getBoundingClientRect(), wb = wrap.getBoundingClientRect();
    if (rb.top < wb.top+40 || rb.bottom > wb.bottom-20){
      row.scrollIntoView({block:'center', behavior:'smooth'});
    }
  }
}

/* ========================= Data Ingest ========================= */
function parseTSV(tsv){
  const lines = tsv.split(/\r?\n/).filter(l=>l.trim()!=="");
  if (!lines.length) return [];
  const cols = lines[0].split(/\t/);
  const hasHeader = HEADERS.every((h,i)=> (cols[i]||"").trim().toLowerCase() === h.toLowerCase());
  const start = hasHeader ? 1 : 0;
  const out = [];
  for (let i=start;i<lines.length;i++){
    const cells = lines[i].split(/\t/);
    const row = HEADERS.map((_,ci)=> cells[ci] ?? "");
    out.push(row);
  }
  return out;
}

function buildGraphFromRows(){
  systems.clear();
  if (!rows.length) return;

  // helper
  const ensure = (name)=> {
    if (!name) return null;
    if (!systems.has(name)) systems.set(name, {name, x: null, y: null, connections: []});
    return systems.get(name);
  };

  rows.forEach(r=>{
    const srcName = (r[1]||"").trim(); if (!srcName) return;
    const src = ensure(srcName);
    GROUPS.forEach(g=>{
      const tname = (r[g.conn]||"").trim();
      const dist = toNumber(r[g.dist]);
      const ang  = toNumber(r[g.ang]);
      if (!tname) return;
      ensure(tname);
      src.connections.push({to: tname, dist, ang});
    });
  });

  // initial placement: first row center; neighbors around it
  const first = systems.get((rows[0][1]||"").trim());
  if (!first) return;
  first.x = 0; first.y = 0;

  // place direct neighbors using their angle/distance when present
  first.connections.forEach(c=>{
    const n = systems.get(c.to);
    if (n && (n.x==null || n.y==null) && c.dist!=null && c.ang!=null){
      const p = polar(c.ang, c.dist);
      n.x = first.x + p.x; n.y = first.y + p.y;
    }
  });

  // breadth placement using single known neighbor rule
  let progress = true, guard=0;
  while(progress && guard<400){
    progress = false; guard++;
    systems.forEach((sys)=>{
      if (sys.x==null || sys.y==null) return;
      sys.connections.forEach(c=>{
        const tgt = systems.get(c.to);
        if (!tgt || (tgt.x!=null && tgt.y!=null)) return;
        if (c.dist!=null && c.ang!=null){
          const p = polar(c.ang, c.dist);
          tgt.x = sys.x + p.x; tgt.y = sys.y + p.y;
          progress = true;
        }
      });
    });
  }

  // view reset
  view.x = 0; view.y = 0; view.scale = 1;
}

function toNumber(v){ const n=parseFloat(v); return Number.isFinite(n)?n:null; }
function polar(angleDeg, r){
  const th = (angleDeg*Math.PI)/180;
  return {x: r*Math.cos(th), y: -r*Math.sin(th)};
}

/* ========================= Drawing ========================= */
function draw(){
  ctx.setTransform(1,0,0,1,0,0);
  ctx.fillStyle = '#0e1320'; ctx.fillRect(0,0,canvas.width, canvas.height);

  // grid (subtle)
  const step = 100*view.scale;
  if (step>20){
    ctx.strokeStyle = 'rgba(255,255,255,.06)';
    ctx.lineWidth = 1;
    const ox = (canvas.width/2 - view.x*view.scale) % step;
    const oy = (canvas.height/2 - view.y*view.scale) % step;
    for (let x=ox; x<canvas.width; x+=step){ ctx.beginPath(); ctx.moveTo(x,0); ctx.lineTo(x,canvas.height); ctx.stroke(); }
    for (let y=oy; y<canvas.height; y+=step){ ctx.beginPath(); ctx.moveTo(0,y); ctx.lineTo(canvas.width,y); ctx.stroke(); }
  }

  // transform
  ctx.translate(canvas.width/2, canvas.height/2);
  ctx.scale(view.scale, view.scale);
  ctx.translate(-view.x, -view.y);

  // links
  systems.forEach(sys=>{
    if (sys.x==null) return;
    sys.connections.forEach(c=>{
      const tgt = systems.get(c.to); if (!tgt || tgt.x==null) return;
      const dx = tgt.x - sys.x, dy = tgt.y - sys.y;
      const d = Math.hypot(dx,dy);
      ctx.strokeStyle = (c.dist!=null && Math.abs(d - c.dist) <= LINK_TOL) ? '#22c55e' : '#ef4444';
      ctx.lineWidth = 2/view.scale;
      ctx.beginPath(); ctx.moveTo(sys.x, sys.y); ctx.lineTo(tgt.x, tgt.y); ctx.stroke();
    });
  });

  // nodes
  systems.forEach(sys=>{
    if (sys.x==null) return;
    ctx.fillStyle = (sys.name===selectedName)? '#60a5fa' : '#ffffff';
    ctx.strokeStyle = '#1f2937'; ctx.lineWidth = 2/view.scale;
    ctx.beginPath(); ctx.arc(sys.x, sys.y, RADIUS/view.scale, 0, Math.PI*2); ctx.fill(); ctx.stroke();
  });
}

/* ========================= Hit-testing & Selection ========================= */
function findNodeAt(wx, wy, tolWorld){
  let hit = null;
  systems.forEach(sys=>{
    if (sys.x==null) return;
    const d = Math.hypot(sys.x-wx, sys.y-wy);
    if (d <= tolWorld) hit = sys;
  });
  return hit;
}

function selectSystem(name, center=false){
  if (!systems.has(name)) return;
  selectedName = name;
  highlightRow(name);
  if (center) centerOnSystem(name, true);
  draw();
}

function centerOnSystem(name, animate=false){
  const sys = systems.get(name); if (!sys || sys.x==null) return;
  if (animate){
    const frames = 12;
    const stepx = (sys.x - view.x)/frames;
    const stepy = (sys.y - view.y)/frames;
    let i=0;
    (function tick(){
      view.x += stepx; view.y += stepy; draw();
      if(++i<frames) requestAnimationFrame(tick);
    })();
  } else {
    view.x = sys.x; view.y = sys.y; draw();
  }
}

/* ========================= Buttons ========================= */
document.getElementById('btnBuild').addEventListener('click', ()=>{
  buildGraphFromRows(); fitToView(); draw();
});

document.getElementById('btnAdjust').addEventListener('click', ()=>{
  // Small local adjustments on neighbors: try angle ±15° to minimize length error to intended distance
  systems.forEach(sys=>{
    if (sys.x==null) return;
    sys.connections.forEach(c=>{
      const tgt = systems.get(c.to); if (!tgt || tgt.x==null || c.dist==null) return;
      // Only adjust target, keep source fixed
      const base = Math.atan2(-(tgt.y - sys.y), (tgt.x - sys.x)) * 180/Math.PI;
      let best = {err: Infinity, ang: base};
      for (let a=-15; a<=15; a+=1){
        const th = (a + (c.ang ?? base)) * Math.PI/180;
        const px = sys.x + c.dist*Math.cos(th);
        const py = sys.y - c.dist*Math.sin(th);
        // error vs other already-connected neighbors' desired distances (just this one baseline)
        const err = Math.hypot(px - tgt.x, py - tgt.y); // move cost
        if (err < best.err){ best = {err, ang: a + (c.ang ?? base)}; }
      }
      const th = best.ang * Math.PI/180;
      tgt.x = sys.x + c.dist*Math.cos(th);
      tgt.y = sys.y - c.dist*Math.sin(th);
    });
  });
  draw();
});

document.getElementById('btnResetView').addEventListener('click', ()=>{
  view = {x:0,y:0,scale:1}; draw();
});

document.getElementById('btnFit').addEventListener('click', ()=>{ fitToView(); draw(); });

document.getElementById('btnExport').addEventListener('click', ()=>{
  // build CSV from current table (headers + rows)
  let out = HEADERS.join(",") + "\n";
  rows.forEach(r=>{
    const line = r.map(v => {
      const s = (v??"").toString();
      return s.includes(",") ? `"${s.replace(/"/g,'""')}"` : s;
    }).join(",");
    out += line + "\n";
  });
  const blob = new Blob([out],{type:"text/csv"});
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a'); a.href=url; a.download="systems.csv"; a.click();
  URL.revokeObjectURL(url);
});

document.getElementById('btnApply').addEventListener('click', ()=>{
  const text = document.getElementById('pasteArea').value.trim();
  if (!text) return;
  rows = parseTSV(text);
  renderTable();
});

function fitToView(){
  // compute world bounds of placed nodes
  let xs=[], ys=[];
  systems.forEach(s=>{ if (s.x!=null){ xs.push(s.x); ys.push(s.y); }});
  if (!xs.length){ view = {x:0,y:0,scale:1}; return; }
  const minX=Math.min(...xs)-40, maxX=Math.max(...xs)+40;
  const minY=Math.min(...ys)-40, maxY=Math.max(...ys)+40;
  const w = maxX - minX, h = maxY - minY;
  const sx = canvas.width / w, sy = canvas.height / h;
  const sc = Math.min(sx, sy, 4);
  view.scale = sc;
  view.x = (minX + maxX)/2;
  view.y = (minY + maxY)/2;
}

/* ========================= Demo starter ========================= */
const DEMO =
`Bonus	System Name	Angle2one	Dist2one	Bonus2	Connects1	Angle2two	Dist2two	Bonus3	Connects2	Angle2three	Dist2three	Bonus4	Connects3	Angle2four	Dist2four	Bonus5	Connects4	Angle2five	Dist2five	Bonus6	Connects5
GECg	Dandinoon	30	147	Arm+	Sallysah	300	295	HP+	Zedio	270	296	Arm+	Janislemy	90	185	RSU	Fedinan				
Arm+	Janislemy	105	296	GECg	Dandinoon		222	Mine	Bam		185	Acc+	Qangrend								
GECg	Ouruscant	0	259	Mine	Risentum	240	296	Fac	Lolipupos	180	259	Fac	Xem								
Arm+	Sallysah	90	222	Mine	Sawadikalos	330	185	Fac	Wakom	210	147	GECg	Dandinoon								
Fac	Wakom	0	259	Fac	Xem	240	185	Mine	Reid	150	185	Arm+	Sallysah								
Fac	Xem	60	185	Mine	Muloism	0	259	GECg	Ouruscant	180	259	Fac	Wakom								
HP+	Zedio		110	Mine	Reid		222	Fac	Trianguilc		258	Mine	Bam	120	295	GECg	Dandinoon				`;

rows = parseTSV(DEMO);
renderTable();
buildGraphFromRows();
fitToView();
draw();

/* ========================= Utility ========================= */
function clamp(v,min,max){ return Math.min(max, Math.max(min,v)); }
</script>
</body>
</html>
