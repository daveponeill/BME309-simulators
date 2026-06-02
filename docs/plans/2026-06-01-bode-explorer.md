# Bode Explorer Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build `bode_explorer.html` — an interactive s-plane pole/zero editor with live Bode magnitude + phase plots and asymptotic approximation overlay.

**Architecture:** Single self-contained HTML file, no dependencies. Mirrors the structure and style of `zplane_explorer_3.html`. Canvas rendering via 2D context; all math in vanilla JS. Two-column layout: s-plane canvas left, stacked Bode plots right.

**Tech Stack:** HTML5, CSS3 (CSS variables, grid), vanilla JavaScript, Canvas 2D API.

**Reference file:** `zplane_explorer_3.html` — copy CSS variables, card/button/slider patterns, dark mode logic, `drawAxes`, `drawCurve` helpers verbatim.

---

### Task 1: HTML skeleton + CSS

**Files:**
- Create: `bode_explorer.html`

**Step 1: Create the file with boilerplate**

Copy the full `<head>` and `<style>` block from `zplane_explorer_3.html`. Make these changes:
- Title: `Z-Plane Pole-Zero Explorer — BME 309` → `Bode Plot Explorer — BME 309`
- Header h1: `Bode Plot Explorer`
- Remove `--phase` CSS variable (unused)
- Change `.plots` grid from `repeat(3, 1fr)` to `grid-template-columns: 1fr 1fr`
- Add `.bode-stack` class: `display:flex; flex-direction:column; gap:12px`
- The s-plane canvas should be taller: use `width="400" height="400"`
- Each Bode canvas: `width="400" height="190"`

**Step 2: Add HTML structure (no JS yet)**

```html
<header>
  <h1>Bode Plot Explorer</h1>
  <span class="sub">BME 309 — Signals &amp; Systems</span>
</header>
<main>
  <!-- Presets card -->
  <div class="card">
    <div class="section-label">Presets</div>
    <div class="preset-row">
      <button onclick="setPreset('butter1lp')">Butterworth 1st LP</button>
      <button onclick="setPreset('butter2lp')">Butterworth 2nd LP</button>
      <button onclick="setPreset('butter2hp')">Butterworth 2nd HP</button>
      <button onclick="setPreset('ecgbp')">ECG Bandpass</button>
      <button onclick="setPreset('notch60')">60 Hz Notch</button>
      <button onclick="setPreset('reset')">Reset</button>
    </div>
  </div>

  <!-- Pole/Zero controls -->
  <div class="pz-grid">
    <!-- Poles panel (same structure as zplane_explorer_3.html) -->
    <!-- id: pn0/pn1/pn2, p1s/p1sL (sigma), p1w/p1wL (omega), pole1ctrl, pole2ctrl -->
    <!-- Zeros panel -->
    <!-- id: zn0/zn1/zn2, z1s/z1sL, z1w/z1wL, zero1ctrl, zero2ctrl -->
  </div>

  <!-- Toggle card -->
  <div class="card">
    <div class="toggle-row">
      <span class="tog-label">Frequency:</span>
      <button id="togRad" class="tog active" onclick="setFreqUnit('rad')">rad/s</button>
      <button id="togHz" class="tog" onclick="setFreqUnit('hz')">Hz</button>
      <div class="sep"></div>
      <span class="tog-label">Magnitude:</span>
      <button id="togLin" class="tog" onclick="setDb(false)">Linear</button>
      <button id="togDb" class="tog active" onclick="setDb(true)">dB</button>
      <div class="sep"></div>
      <span class="tog-label">Bode curve:</span>
      <button id="togReal" class="tog active" onclick="setBodeMode('real')">Real</button>
      <button id="togAsymp" class="tog" onclick="setBodeMode('asymp')">Asymptotic</button>
      <button id="togBoth" class="tog" onclick="setBodeMode('both')">Both</button>
    </div>
  </div>

  <!-- Plots -->
  <div class="plots">
    <div class="plot-box">
      <div class="plot-title">s-plane</div>
      <canvas id="splane" width="400" height="400"></canvas>
    </div>
    <div class="bode-stack">
      <div class="plot-box">
        <div class="plot-title" id="magTitle">|H(jΩ)| — dB</div>
        <canvas id="bodemag" width="400" height="190"></canvas>
      </div>
      <div class="plot-box">
        <div class="plot-title">∠H(jΩ) — degrees</div>
        <canvas id="bodephase" width="400" height="190"></canvas>
      </div>
    </div>
  </div>

  <div id="noteBox" class="note-box"></div>

  <!-- How to read this card -->
</main>
<footer>BME 309 · Northwestern University · Interactive tool generated with Claude</footer>
```

**Step 3: Open in browser and verify layout renders (no JS)**

Expected: header, preset buttons, empty plot boxes in 2-column layout, toggle row visible.

---

### Task 2: State, pole/zero sliders, and getPoles/getZeros

**Files:**
- Modify: `bode_explorer.html` — add `<script>` block

**Step 1: Add color constants and state variables**

```js
const isDark = matchMedia('(prefers-color-scheme: dark)').matches;
const textCol  = isDark ? '#a0a09a' : '#5a5a56';
const gridCol  = isDark ? 'rgba(255,255,255,0.07)' : 'rgba(0,0,0,0.06)';
const unitCol  = isDark ? 'rgba(255,255,255,0.22)' : 'rgba(0,0,0,0.16)';
const poleCol  = isDark ? '#F07050' : '#D85A30';
const zeroCol  = isDark ? '#5BAFF5' : '#185FA5';
const lineCol  = isDark ? '#8880EE' : '#534AB7';
const asympCol = isDark ? 'rgba(255,200,80,0.7)' : 'rgba(180,120,0,0.65)';

let poleCount = 1, zeroCount = 0;
let useDb = true, freqUnit = 'rad', bodeMode = 'real';

function sv(id) { const el = document.getElementById(id); return el ? parseFloat(el.value) : 0; }
function si(id, v) { const el = document.getElementById(id); if (el) el.value = v; }
```

**Step 2: Add getPoles / getZeros**

Poles/zeros are defined by σ (sigma, ≤0) and Ω (omega, ≥0).
- If Ω === 0: single real pole/zero at s = σ (push once, not a conjugate pair)
- If Ω > 0: conjugate pair at s = σ ± jΩ (push two)

```js
function getPoles() {
  const list = [];
  if (poleCount >= 1) {
    const s = sv('p1s'), w = sv('p1w');
    if (w === 0) list.push([s, 0]); else list.push([s, w], [s, -w]);
  }
  if (poleCount >= 2) {
    const s = sv('p2s'), w = sv('p2w');
    if (w === 0) list.push([s, 0]); else list.push([s, w], [s, -w]);
  }
  return list; // each entry: [sigma, omega_imag]
}
function getZeros() {
  const list = [];
  if (zeroCount >= 1) {
    const s = sv('z1s'), w = sv('z1w');
    if (w === 0) list.push([s, 0]); else list.push([s, w], [s, -w]);
  }
  if (zeroCount >= 2) {
    const s = sv('z2s'), w = sv('z2w');
    if (w === 0) list.push([s, 0]); else list.push([s, w], [s, -w]);
  }
  return list;
}
```

**Step 3: Add update() stub and slider HTML**

Fill in the pole/zero slider panels in the HTML (replacing the comments from Task 1).
Sliders: `p1s` (sigma 1, min=-2000, max=0, step=10, value=-100), `p1w` (omega 1, min=0, max=2000, step=10, value=0).
Same for p2s/p2w, z1s/z1w, z2s/z2w.

```js
function update() {
  // update label displays
  [['p1s',''],['p1w',''],['p2s',''],['p2w',''],
   ['z1s',''],['z1w',''],['z2s',''],['z2w','']].forEach(([id])=>{
    const el=document.getElementById(id), lbl=document.getElementById(id+'L');
    if(el&&lbl) lbl.textContent = parseFloat(el.value).toFixed(0);
  });
  // placeholder — draw functions added in later tasks
}
update();
```

**Step 4: Verify in browser**

Sliders appear, values update in labels when dragged. No plots yet.

**Step 5: Commit**

```bash
git add bode_explorer.html
git commit -m "Add Bode explorer skeleton with sliders and state"
```

---

### Task 3: Bode computation (exact)

**Files:**
- Modify: `bode_explorer.html`

**Step 1: Add frequency axis helper**

```js
const N_FREQ = 500;
const OMEGA_MIN = 0.1, OMEGA_MAX = 10000; // rad/s

function freqAxis() {
  // returns array of N_FREQ log-spaced omega values
  const arr = [];
  const logMin = Math.log10(OMEGA_MIN), logMax = Math.log10(OMEGA_MAX);
  for (let i = 0; i < N_FREQ; i++) {
    arr.push(Math.pow(10, logMin + (i/(N_FREQ-1))*(logMax-logMin)));
  }
  return arr;
}
```

**Step 2: Add computeExact()**

Evaluates H(jΩ) = ∏(jΩ − zᵢ) / ∏(jΩ − pᵢ) at each frequency point.
Each pole/zero is [sigma, omega_imag], so s = sigma + j*omega_imag.
At s = jΩ: (jΩ − (σ + jω)) = (−σ) + j(Ω − ω).

```js
function computeExact(omegas) {
  const poles = getPoles(), zeros = getZeros();
  const mags = [], phases = [];
  for (const om of omegas) {
    let re = 1, im = 0; // running product for numerator
    // zeros: multiply by (jom - z) = (-z.sigma) + j*(om - z.omega)
    for (const [sig, omZ] of zeros) {
      const zr = -sig, zi = om - omZ;
      const [nr, ni] = [re*zr - im*zi, re*zi + im*zr];
      re = nr; im = ni;
    }
    let dr = 1, di = 0; // denominator
    for (const [sig, omP] of poles) {
      const pr = -sig, pi = om - omP;
      const d2 = pr*pr + pi*pi;
      if (d2 < 1e-20) { dr = 1e-10; di = 0; break; }
      const [nr, ni] = [dr*pr + di*pi, di*pr - dr*pi];
      dr = nr/d2; di = ni/d2; // multiply by 1/(p)
      // actually: divide running denom by (jom-pole)
      // Simpler: accumulate magnitude and phase separately
    }
    // Redo with magnitude/phase accumulation (cleaner):
    let magNum = 1, phNum = 0, magDen = 1, phDen = 0;
    for (const [sig, omZ] of zeros) {
      const a = -sig, b = om - omZ;
      magNum *= Math.sqrt(a*a + b*b);
      phNum  += Math.atan2(b, a);
    }
    for (const [sig, omP] of poles) {
      const a = -sig, b = om - omP;
      const d = Math.sqrt(a*a + b*b);
      if (d < 1e-10) { magDen = 1e-10; } else { magDen *= d; phDen += Math.atan2(b, a); }
    }
    mags.push(magNum / magDen);
    phases.push(phNum - phDen);
  }
  // Unwrap phase
  for (let i = 1; i < phases.length; i++) {
    let diff = phases[i] - phases[i-1];
    while (diff >  Math.PI) diff -= 2*Math.PI;
    while (diff < -Math.PI) diff += 2*Math.PI;
    phases[i] = phases[i-1] + diff;
  }
  return { mags, phases };
}
```

**Step 3: Verify in browser console**

Open DevTools console, type `computeExact(freqAxis())` — should return object with `mags` and `phases` arrays of length 500.

---

### Task 4: Asymptotic Bode approximation

**Files:**
- Modify: `bode_explorer.html`

**Step 1: Add computeAsymp()**

The asymptotic magnitude approximation:
- Start at 0 dB
- For each real pole at corner frequency ωc = |σ|: slope decreases by −20 dB/dec above ωc
- For each complex conjugate pole pair at ωn = √(σ²+ω²): slope decreases by −40 dB/dec above ωn
- Same logic inverted for zeros

The asymptotic phase approximation (per pole/zero):
- Real pole at ωc: 0° for Ω < ωc/10, −45° at ωc, −90° for Ω > 10·ωc (linear interpolation in log space)
- Complex pair at ωn: 0° → −180° spanning one decade around ωn

```js
function computeAsymp(omegas) {
  const poles = getPoles(), zeros = getZeros();

  // Collect corner frequencies with their type and sign
  // type 'real': single pole/zero; type 'complex': conjugate pair (count once per pair)
  const corners = [];
  // poles counted by pair: iterate unique pairs only
  const seenPoles = new Set();
  for (const [sig, omP] of poles) {
    const key = `${sig},${Math.abs(omP)}`;
    if (!seenPoles.has(key)) {
      seenPoles.add(key);
      const wn = Math.sqrt(sig*sig + omP*omP) || Math.abs(sig);
      corners.push({ wc: wn, slope: omP === 0 ? -20 : -40, phShift: omP === 0 ? -90 : -180 });
    }
  }
  const seenZeros = new Set();
  for (const [sig, omZ] of zeros) {
    const key = `${sig},${Math.abs(omZ)}`;
    if (!seenZeros.has(key)) {
      seenZeros.add(key);
      const wn = Math.sqrt(sig*sig + omZ*omZ) || Math.abs(sig);
      corners.push({ wc: wn, slope: omZ === 0 ? +20 : +40, phShift: omZ === 0 ? +90 : +180 });
    }
  }

  const magsdB = [], phases = [];
  for (const om of omegas) {
    let dB = 0;
    let ph = 0;
    for (const { wc, slope, phShift } of corners) {
      // magnitude: flat below wc, then slope dB/dec above
      if (om > wc) dB += slope * Math.log10(om / wc);
      // phase: linear in log10(omega) from wc/10 to wc*10
      const logRatio = Math.log10(om / wc);
      if (logRatio < -1)      ph += 0;
      else if (logRatio > 1)  ph += phShift;
      else                    ph += phShift * (logRatio + 1) / 2;
    }
    magsdB.push(dB);
    phases.push(ph * Math.PI / 180); // convert to radians for consistent plotting
  }
  return { magsdB, phases };
}
```

**Step 2: Verify in console**

`computeAsymp(freqAxis())` returns object with `magsdB` and `phases` of length 500.

---

### Task 5: Draw s-plane canvas

**Files:**
- Modify: `bode_explorer.html`

**Step 1: Add drawSPlane()**

```js
function drawSPlane() {
  const c = document.getElementById('splane');
  const ctx = c.getContext('2d');
  const W = c.width, H = c.height;
  ctx.clearRect(0, 0, W, H);

  // Map s-plane coords to canvas:
  // sigma (x): -2200 to 200 rad/s → 0 to W
  // omega (y): -2200 to 2200 rad/s → H to 0
  const sigMin = -2200, sigMax = 200, omRange = 2200;
  const toX = (sig) => (sig - sigMin) / (sigMax - sigMin) * W;
  const toY = (om)  => H/2 - om / omRange * (H/2);

  // Imaginary axis (stability boundary)
  ctx.strokeStyle = unitCol; ctx.lineWidth = 1.5;
  ctx.beginPath(); ctx.moveTo(toX(0), 0); ctx.lineTo(toX(0), H); ctx.stroke();

  // Real axis
  ctx.strokeStyle = gridCol; ctx.lineWidth = 0.5;
  ctx.beginPath(); ctx.moveTo(0, toY(0)); ctx.lineTo(W, toY(0)); ctx.stroke();

  // Stable region shading
  ctx.fillStyle = isDark ? 'rgba(255,255,255,0.03)' : 'rgba(0,0,0,0.025)';
  ctx.fillRect(0, 0, toX(0), H);

  // Axis labels
  ctx.fillStyle = textCol; ctx.font = '10px system-ui,sans-serif';
  ctx.textAlign = 'center';
  ctx.fillText('jΩ axis', toX(0), 10);
  ctx.fillText('stable →', toX(-1100), H - 6);

  // Draw poles (×)
  getPoles().forEach(([sig, om]) => {
    const x = toX(sig), y = toY(om);
    ctx.strokeStyle = poleCol; ctx.lineWidth = 2.2;
    ctx.beginPath(); ctx.moveTo(x-7,y-7); ctx.lineTo(x+7,y+7); ctx.stroke();
    ctx.beginPath(); ctx.moveTo(x+7,y-7); ctx.lineTo(x-7,y+7); ctx.stroke();
  });

  // Draw zeros (○)
  getZeros().forEach(([sig, om]) => {
    const x = toX(sig), y = toY(om);
    ctx.strokeStyle = zeroCol; ctx.lineWidth = 2.2;
    ctx.beginPath(); ctx.arc(x, y, 7, 0, 2*Math.PI); ctx.stroke();
  });

  // Legend
  ctx.font = '10px system-ui,sans-serif'; ctx.textAlign = 'left';
  ctx.fillStyle = poleCol; ctx.fillText('×  pole', 8, H-14);
  ctx.fillStyle = zeroCol; ctx.fillText('○  zero', 8, H-4);
}
```

**Step 2: Call drawSPlane() from update(), verify poles/zeros appear on canvas.**

---

### Task 6: Draw Bode magnitude plot

**Files:**
- Modify: `bode_explorer.html`

**Step 1: Add drawBodeMag(omegas, exact, asymp)**

`exact` = `{ mags, phases }` from computeExact. `asymp` = `{ magsdB, phases }` from computeAsymp.

```js
function drawBodeMag(omegas, exact, asymp) {
  const c = document.getElementById('bodemag');
  const ctx = c.getContext('2d');
  const W = c.width, H = c.height;
  ctx.clearRect(0, 0, W, H);
  const pad = { l: 44, r: 10, t: 10, b: 26 };

  // Build data arrays in dB or linear
  let realData, asympData, yMin, yMax;
  if (useDb) {
    realData  = exact.mags.map(m => m > 1e-12 ? 20*Math.log10(m) : -120);
    asympData = asymp.magsdB;
    const allFin = [...(bodeMode!=='asymp'?realData:[]),
                   ...(bodeMode!=='real'?asympData:[])].filter(isFinite);
    const rawMax = Math.max(...allFin, 0), rawMin = Math.min(...allFin, 0);
    yMax = Math.ceil(rawMax/20)*20 || 20;
    yMin = Math.max(Math.floor(rawMin/20)*20, yMax - 120);
  } else {
    realData  = exact.mags;
    asympData = asymp.magsdB.map(db => Math.pow(10, db/20));
    const fin = realData.filter(isFinite);
    yMax = Math.ceil(Math.min(Math.max(...fin), 20)*10)/10 || 1;
    yMin = 0;
  }

  // x-axis labels in rad/s or Hz
  const xLabels = freqUnit === 'rad'
    ? ['0.1','1','10','100','1k','10k']
    : ['0.016','0.16','1.6','16','159','1592'];

  drawAxes(ctx, W, H, pad, xLabels, yMin, yMax, 4);

  // Helper: map omega index to x pixel (log scale)
  const gW = W - pad.l - pad.r, gH = H - pad.t - pad.b;
  const omToX = i => pad.l + (Math.log10(omegas[i]/OMEGA_MIN) /
                               Math.log10(OMEGA_MAX/OMEGA_MIN)) * gW;

  function plotLine(data, col, dash=[]) {
    ctx.strokeStyle = col; ctx.lineWidth = 2; ctx.setLineDash(dash);
    ctx.beginPath(); let started = false;
    data.forEach((v, i) => {
      const x = omToX(i);
      const clamped = Math.max(yMin, Math.min(yMax, v));
      const y = pad.t + gH - (clamped - yMin)/(yMax - yMin)*gH;
      if (!isFinite(v) || !started) { ctx.moveTo(x,y); started=true; }
      else ctx.lineTo(x,y);
    });
    ctx.stroke(); ctx.setLineDash([]);
  }

  if (bodeMode === 'real' || bodeMode === 'both') plotLine(realData, lineCol);
  if (bodeMode === 'asymp' || bodeMode === 'both') plotLine(asympData, asympCol, [5,4]);

  // Corner frequency markers
  const allCorners = [
    ...getPoles().map(([s,w])=>Math.sqrt(s*s+w*w)),
    ...getZeros().map(([s,w])=>Math.sqrt(s*s+w*w))
  ].filter(wc => wc > 0);
  allCorners.forEach(wc => {
    const frac = Math.log10(wc/OMEGA_MIN)/Math.log10(OMEGA_MAX/OMEGA_MIN);
    const x = pad.l + frac*gW;
    ctx.strokeStyle = isDark?'rgba(255,255,255,0.15)':'rgba(0,0,0,0.12)';
    ctx.lineWidth=0.8; ctx.setLineDash([3,3]);
    ctx.beginPath(); ctx.moveTo(x,pad.t); ctx.lineTo(x,pad.t+gH); ctx.stroke();
    ctx.setLineDash([]);
  });
}
```

**Step 2: Wire into update(), verify magnitude plot renders with log x-axis.**

---

### Task 7: Draw Bode phase plot

**Files:**
- Modify: `bode_explorer.html`

**Step 1: Add drawBodePhase(omegas, exact, asymp)**

```js
function drawBodePhase(omegas, exact, asymp) {
  const c = document.getElementById('bodephase');
  const ctx = c.getContext('2d');
  const W = c.width, H = c.height;
  ctx.clearRect(0, 0, W, H);
  const pad = { l: 44, r: 10, t: 10, b: 26 };

  const toDeg = arr => arr.map(r => r * 180 / Math.PI);
  const realDeg  = toDeg(exact.phases);
  const asympDeg = toDeg(asymp.phases);

  const allDeg = [...(bodeMode!=='asymp'?realDeg:[]),
                  ...(bodeMode!=='real'?asympDeg:[])].filter(isFinite);
  const rawMin = Math.min(...allDeg, 0), rawMax = Math.max(...allDeg, 0);
  const yMin = Math.floor(rawMin/45)*45 - 45;
  const yMax = Math.ceil(rawMax/45)*45  + 45;

  const xLabels = freqUnit === 'rad'
    ? ['0.1','1','10','100','1k','10k']
    : ['0.016','0.16','1.6','16','159','1592'];

  drawAxes(ctx, W, H, pad, xLabels, yMin, yMax, 4);

  const gW = W - pad.l - pad.r, gH = H - pad.t - pad.b;
  const omToX = i => pad.l + (Math.log10(omegas[i]/OMEGA_MIN) /
                               Math.log10(OMEGA_MAX/OMEGA_MIN)) * gW;

  function plotLine(data, col, dash=[]) {
    ctx.strokeStyle = col; ctx.lineWidth = 2; ctx.setLineDash(dash);
    ctx.beginPath(); let started = false;
    data.forEach((v, i) => {
      const x = omToX(i);
      const clamped = Math.max(yMin, Math.min(yMax, v));
      const y = pad.t + gH - (clamped - yMin)/(yMax - yMin)*gH;
      if (!isFinite(v) || !started) { ctx.moveTo(x,y); started=true; }
      else ctx.lineTo(x,y);
    });
    ctx.stroke(); ctx.setLineDash([]);
  }

  if (bodeMode === 'real' || bodeMode === 'both') plotLine(realDeg, lineCol);
  if (bodeMode === 'asymp' || bodeMode === 'both') plotLine(asympDeg, asympCol, [5,4]);

  // y-axis phase labels in degrees (override numeric with degree symbols)
  ctx.fillStyle = textCol; ctx.font = '10px system-ui,sans-serif'; ctx.textAlign = 'right';
  const ticks = 4;
  for (let i = 0; i <= ticks; i++) {
    const v = yMax - i*(yMax-yMin)/ticks;
    const y = pad.t + i*gH/ticks;
    ctx.fillText(v.toFixed(0)+'°', pad.l-4, y+3);
  }
}
```

**Step 2: Wire into update(), verify phase plot renders.**

---

### Task 8: Canvas drag interaction

**Files:**
- Modify: `bode_explorer.html`

**Step 1: Add drag state and event listeners to s-plane canvas**

```js
let dragging = null; // { type:'pole'|'zero', idx:0|1, which:'pair' }

function sPlaneCoords(canvas, evt) {
  const rect = canvas.getBoundingClientRect();
  const px = (evt.clientX - rect.left) * (canvas.width / rect.width);
  const py = (evt.clientY - rect.top)  * (canvas.height / rect.height);
  const sigMin = -2200, sigMax = 200, omRange = 2200;
  const W = canvas.width, H = canvas.height;
  const sig = sigMin + (px / W) * (sigMax - sigMin);
  const om  = (H/2 - py) / (H/2) * omRange;
  return { sig: Math.max(-2000, Math.min(0, sig)), om: Math.max(0, om) };
}

function initDrag() {
  const c = document.getElementById('splane');
  c.addEventListener('mousedown', e => {
    const { sig, om } = sPlaneCoords(c, e);
    // Find nearest pole/zero within 15px
    const sigMin=-2200,sigMax=200,omRange=2200,W=c.width,H=c.height;
    const toX = s => (s-sigMin)/(sigMax-sigMin)*W;
    const toY = o => H/2 - o/omRange*(H/2);
    const hit = (s,o) => Math.hypot(toX(s)-toX(sig), toY(o)-toY(om)) < 15;

    if (poleCount >= 1 && hit(sv('p1s'), sv('p1w'))) dragging = {id:'p1s',wid:'p1w'};
    else if (poleCount >= 2 && hit(sv('p2s'), sv('p2w'))) dragging = {id:'p2s',wid:'p2w'};
    else if (zeroCount >= 1 && hit(sv('z1s'), sv('z1w'))) dragging = {id:'z1s',wid:'z1w'};
    else if (zeroCount >= 2 && hit(sv('z2s'), sv('z2w'))) dragging = {id:'z2s',wid:'z2w'};
  });
  c.addEventListener('mousemove', e => {
    if (!dragging) return;
    const { sig, om } = sPlaneCoords(c, e);
    si(dragging.id, Math.round(sig/10)*10);
    si(dragging.wid, Math.round(om/10)*10);
    update();
  });
  c.addEventListener('mouseup', () => dragging = null);
  c.addEventListener('mouseleave', () => dragging = null);
}
```

**Step 2: Call `initDrag()` after `update()` at bottom of script.**

**Step 3: Verify dragging poles/zeros on s-plane updates Bode plots live.**

---

### Task 9: Toggle functions + setPoleCount/setZeroCount + presets

**Files:**
- Modify: `bode_explorer.html`

**Step 1: Add toggle and count functions**

```js
function setPoleCount(n) {
  poleCount = n;
  ['pn0','pn1','pn2'].forEach((id,i)=>document.getElementById(id).classList.toggle('active',i===n));
  document.getElementById('pole1ctrl').style.display = n>=1?'':'none';
  document.getElementById('pole2ctrl').style.display = n>=2?'':'none';
  update();
}
function setZeroCount(n) {
  zeroCount = n;
  ['zn0','zn1','zn2'].forEach((id,i)=>document.getElementById(id).classList.toggle('active',i===n));
  document.getElementById('zero1ctrl').style.display = n>=1?'':'none';
  document.getElementById('zero2ctrl').style.display = n>=2?'':'none';
  update();
}
function setFreqUnit(u) {
  freqUnit = u;
  document.getElementById('togRad').classList.toggle('active', u==='rad');
  document.getElementById('togHz').classList.toggle('active',  u==='hz');
  update();
}
function setDb(v) {
  useDb = v;
  document.getElementById('togLin').classList.toggle('active', !v);
  document.getElementById('togDb').classList.toggle('active',   v);
  document.getElementById('magTitle').innerHTML = v ? '|H(jΩ)| — dB' : '|H(jΩ)| — linear';
  update();
}
function setBodeMode(m) {
  bodeMode = m;
  ['togReal','togAsymp','togBoth'].forEach((id,i)=>
    document.getElementById(id).classList.toggle('active', ['real','asymp','both'][i]===m));
  update();
}
```

**Step 2: Add presets**

```js
function setPreset(t) {
  if (t === 'butter1lp') {
    setPoleCount(1); setZeroCount(0);
    si('p1s',-100); si('p1w',0);
  } else if (t === 'butter2lp') {
    setPoleCount(1); setZeroCount(0);
    si('p1s',-71); si('p1w',71);
  } else if (t === 'butter2hp') {
    setPoleCount(1); setZeroCount(1);
    si('p1s',-71); si('p1w',71);
    si('z1s',0);   si('z1w',0);   // zero at origin (s=0)
  } else if (t === 'ecgbp') {
    // Bandpass: poles near 100 Hz (628 rad/s), zeros at origin
    setPoleCount(1); setZeroCount(1);
    si('p1s',-3);  si('p1w',628);
    si('z1s',0);   si('z1w',0);
  } else if (t === 'notch60') {
    // 60 Hz notch: zeros on jΩ axis at ±377, poles slightly inside
    setPoleCount(1); setZeroCount(1);
    si('p1s',-38); si('p1w',377);
    si('z1s',0);   si('z1w',377);
  } else {
    setPoleCount(1); setZeroCount(0);
    si('p1s',-100); si('p1w',0);
  }
  update();
}
```

**Step 3: Verify all presets produce sensible Bode plots.**

---

### Task 10: Note box + final wiring + commit

**Files:**
- Modify: `bode_explorer.html`

**Step 1: Add classify()**

```js
function classify() {
  const poles = getPoles(), zeros = getZeros();
  if (!poleCount && !zeroCount) return 'No poles or zeros — H(s) = 1, flat unity response.';
  const bits = [];
  // Unique pairs only
  const seenP = new Set();
  for (const [sig, om] of poles) {
    const key = `${sig},${Math.abs(om)}`;
    if (seenP.has(key)) continue; seenP.add(key);
    const wn = Math.sqrt(sig*sig+om*om).toFixed(1);
    const stable = sig < 0 ? 'stable' : sig === 0 ? 'marginally stable' : 'UNSTABLE';
    const type = om === 0 ? `real pole at s = ${sig}` : `complex pair at σ=${sig}, Ω=±${om}`;
    bits.push(`<strong>Pole:</strong> ${type} — ωn = ${wn} rad/s — <em>${stable}</em>`);
  }
  const seenZ = new Set();
  for (const [sig, om] of zeros) {
    const key = `${sig},${Math.abs(om)}`;
    if (seenZ.has(key)) continue; seenZ.add(key);
    const wn = Math.sqrt(sig*sig+om*om).toFixed(1);
    const type = om === 0 ? `real zero at s = ${sig}` : `complex pair at σ=${sig}, Ω=±${om}`;
    bits.push(`<strong>Zero:</strong> ${type} — corner freq ≈ ${wn} rad/s`);
  }
  return bits.join('<br>');
}
```

**Step 2: Update the full update() function**

```js
function update() {
  [['p1s',''],['p1w',''],['p2s',''],['p2w',''],
   ['z1s',''],['z1w',''],['z2s',''],['z2w','']].forEach(([id])=>{
    const el=document.getElementById(id), lbl=document.getElementById(id+'L');
    if(el&&lbl) lbl.textContent = parseFloat(el.value).toFixed(0);
  });
  const omegas = freqAxis();
  const exact  = computeExact(omegas);
  const asymp  = computeAsymp(omegas);
  drawSPlane();
  drawBodeMag(omegas, exact, asymp);
  drawBodePhase(omegas, exact, asymp);
  document.getElementById('noteBox').innerHTML = classify();
}
```

**Step 3: Open in browser, test all presets, toggle all switches, drag poles.**

**Step 4: Commit and push**

```bash
git add bode_explorer.html docs/plans/2026-06-01-bode-explorer.md docs/plans/2026-06-01-bode-explorer-design.md
git commit -m "Add Bode Plot Explorer simulator for BME 309"
git push
```

**Step 5: Verify GitHub Pages deploys**

Visit: `https://daveponeill.github.io/BME309-simulators/bode_explorer.html`
