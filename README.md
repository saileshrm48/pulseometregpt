<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1, user-scalable=no" />
  <title>PulseCam — Stable BPM via Phone Camera (v2)</title>
  <meta name="theme-color" content="#0ea5e9" />
  <style>
    :root {
      --bg: #0b1220; --card: #0f172a; --muted: #94a3b8; --text: #e2e8f0; --accent: #22d3ee;
      --good: #34d399; --warn: #fbbf24; --bad: #f87171;
    }
    * { box-sizing: border-box; }
    html, body { height: 100%; }
    body { margin: 0; font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, "Helvetica Neue", Arial; background: radial-gradient(1200px 800px at 70% -10%, #10315955, transparent), var(--bg); color: var(--text); }
    .wrap { max-width: 1080px; margin: 0 auto; padding: 16px; }
    header { display: flex; align-items: center; justify-content: space-between; gap: 12px; }
    h1 { font-size: clamp(20px, 3vw, 28px); margin: 8px 0; letter-spacing: 0.3px; }
    .sub { color: var(--muted); font-size: 14px; }
    .grid { display: grid; grid-template-columns: 1fr; gap: 12px; }
    @media (min-width: 980px) { .grid { grid-template-columns: 1.2fr 1fr; } }

    .card { background: linear-gradient(180deg, #0e172a, #0b1220); border: 1px solid #1f2b47; border-radius: 18px; box-shadow: 0 10px 30px rgba(0,0,0,0.25); }
    .card > .hd { display:flex; align-items:center; justify-content:space-between; padding: 14px 16px; border-bottom: 1px solid #1f2b47; }
    .card > .bd { padding: 14px 16px; }
    .btn { border: 1px solid #1f2b47; background: #0b2033; color: var(--text); border-radius: 10px; padding: 10px 14px; font-weight: 600; cursor: pointer; transition: all .2s; display: inline-flex; align-items: center; gap: 10px; }
    .btn:hover { transform: translateY(-1px); box-shadow: 0 6px 20px rgba(0,0,0,0.25); }
    .btn[disabled] { opacity: .6; cursor: not-allowed; filter: grayscale(.2); }
    .stat { display:flex; gap: 16px; flex-wrap: wrap; }
    .pill { background: #0b2033; border: 1px solid #1f2b47; border-radius: 999px; padding: 6px 10px; font-size: 12px; color: var(--muted); }

    #bpm { font-size: clamp(48px, 8vw, 76px); font-weight: 800; margin: 12px 0 4px; }
    #qual { font-size: 13px; color: var(--muted); }

    .row { display:flex; align-items:center; gap:10px; flex-wrap: wrap; }
    .row > * { flex: 1 1 auto; }

    canvas { width: 100%; height: 240px; background: #07101d; border-radius: 12px; border: 1px solid #1f2b47; display: block; }
    #video, #preview { width: 100%; max-height: 220px; border-radius: 12px; border:1px solid #1f2b47; background:#000; display:block; }
    #video { position: absolute; width: 1px; height: 1px; opacity: 0; pointer-events:none; }

    .hint { font-size: 13px; color: var(--muted); line-height: 1.45; }
    .log { font-family: ui-monospace, Menlo, Consolas, monospace; font-size: 12px; color: #9fb3c8; max-height: 120px; overflow: auto; background: #07101d; border: 1px solid #1f2b47; border-radius: 10px; padding: 8px; }

    .badge { font-size: 11px; padding: 2px 8px; border-radius: 999px; border:1px solid #1f2b47; background:#0b2033; color:var(--muted); }
    .ok { color: var(--good); }
    .warn { color: var(--warn); }
    .bad { color: var(--bad); }
  </style>
</head>
<body>
  <div class="wrap">
    <header>
      <div>
        <h1>PulseCam <span class="badge">PPG</span> <span class="badge">v2</span></h1>
        <div class="sub">Live preview + stabilized BPM with spectral autocorrelation and confidence scoring.</div>
      </div>
      <div class="row" style="justify-content:flex-end; max-width: 520px;">
        <button id="startBtn" class="btn">▶ Start</button>
        <button id="stopBtn" class="btn" disabled>■ Stop</button>
        <button id="torchBtn" class="btn" disabled>💡 Torch</button>
      </div>
    </header>

    <div class="grid" style="margin-top: 14px;">
      <section class="card">
        <div class="hd">
          <strong>Signal</strong>
          <div class="stat">
            <span class="pill" id="fr">FPS: —</span>
            <span class="pill" id="torchState">Torch: —</span>
            <span class="pill" id="coverage">Coverage: —</span>
            <span class="pill" id="sat">Saturation: —</span>
          </div>
        </div>
        <div class="bd">
          <video id="preview" playsinline muted></video>
          <canvas id="wave" style="margin-top: 10px;"></canvas>
          <div class="hint" style="margin-top: 10px;">
            Tip: Fully cover the rear camera with your fingertip. Use gentle pressure—avoid squeezing. Keep still.
          </div>
        </div>
      </section>

      <section class="card">
        <div class="hd"><strong>Heart Rate</strong><span class="pill" id="quality">Quality: —</span><span class="pill" id="conf">Confidence: —</span></div>
        <div class="bd">
          <div id="bpm">—</div>
          <div id="qual">Waiting for a stable signal…</div>
          <div class="hint" style="margin-top: 8px;">
            Stabilized using 8–12s window, autocorrelation (40–180 BPM), median filter + exponential smoothing, and outlier rejection.
          </div>
          <div class="log" id="log" style="margin-top: 10px;"></div>
        </div>
      </section>
    </div>

    <video id="video" playsinline muted></video>
  </div>

  <script>
    // ========== Utilities ==========
    const log = (...a) => { const el = document.getElementById('log'); el.textContent = [new Date().toLocaleTimeString(), ...a].join(' ') + '
' + el.textContent; };
    const clamp = (x,a,b)=>Math.max(a,Math.min(b,x));

    // ========== State ==========
    let stream, track, imageCapture, torchAvailable = false, torchOn = false;
    let rafId = null, procId = null, estId = null;
    let ctx, cvs, v, preview, drawCtx, drawCvs;
    let lastFrameTs = performance.now();

    // Buffers
    const TARGET_FPS = 30;             // analysis rate target
    const BUFFER_SECS = 12;            // rolling window length
    const MAX_SAMPLES = BUFFER_SECS * TARGET_FPS;
    let times = new Float64Array(MAX_SAMPLES);
    let samples = new Float32Array(MAX_SAMPLES);
    let wr = 0, count = 0;

    // BPM history for median smoothing
    const bpmHist = []; const BPM_HIST_N = 5; // last 5 estimates

    // ========== Filters ==========
    // Biquad band-pass (second order). Coeffs from RBJ Audio EQ Cookbook.
    class Biquad {
      constructor(type, fs, f0, Q){ this.type=type; this.fs=fs; this.set(f0,Q); this.x1=this.x2=this.y1=this.y2=0; }
      set(f0,Q){
        const fs=this.fs; const w0=2*Math.PI*f0/fs; const alpha=Math.sin(w0)/(2*Q); const cos=Math.cos(w0);
        let b0,b1,b2,a0,a1,a2;
        if(this.type==='bandpass'){
          b0 = Q*alpha; b1 = 0; b2 = -Q*alpha; a0 = 1 + alpha; a1 = -2*cos; a2 = 1 - alpha;
        } else if (this.type==='lowpass'){
          b0 = (1 - cos)/2; b1 = 1 - cos; b2 = (1 - cos)/2; a0 = 1 + alpha; a1 = -2*cos; a2 = 1 - alpha;
        } else if (this.type==='highpass'){
          b0 = (1 + cos)/2; b1 = -(1 + cos); b2 = (1 + cos)/2; a0 = 1 + alpha; a1 = -2*cos; a2 = 1 - alpha;
        }
        this.b0=b0/a0; this.b1=b1/a0; this.b2=b2/a0; this.a1=a1/a0; this.a2=a2/a0;
      }
      process(x){ const y = this.b0*x + this.b1*this.x1 + this.b2*this.x2 - this.a1*this.y1 - this.a2*this.y2; this.x2=this.x1; this.x1=x; this.y2=this.y1; this.y1=y; return y; }
    }

    // Detrend via exponential moving average
    const makeEMA = (alpha)=>{ let y=null; return x=>{ if(y===null) y=x; y = y + alpha*(x - y); return x - y; }; };

    // ========== Quality metrics ==========
    function coverageMetric(frame){ let redDominant=0; const step=4*8; for(let i=0;i<frame.data.length;i+=step){ const r=frame.data[i], g=frame.data[i+1], b=frame.data[i+2]; if(r>g*1.2 && r>b*1.2) redDominant++; } return redDominant/(frame.data.length/step); }
    function saturationMetric(frame){ // fraction of pixels near clipping
      let sat=0; const step=4*8; for(let i=0;i<frame.data.length;i+=step){ const r=frame.data[i]; if(r>250) sat++; }
      return sat/(frame.data.length/step);
    }

    function updateStatus(id, text){ document.getElementById(id).textContent = text; }
    function setQuality(status, msg){ const el=document.getElementById('quality'); el.textContent=`Quality: ${status}`; el.classList.remove('ok','warn','bad'); el.classList.add(status==='Good'?'ok':status==='Fair'?'warn':'bad'); document.getElementById('qual').textContent=msg; }

    // ========== Measurement Pipeline ==========
    let detrend, bp1, bp2;

    function initDSP(){
      detrend = makeEMA(1 - Math.exp(-2*Math.PI*0.5 / TARGET_FPS)); // ~0.5 Hz baseline removal
      // 0.7–3.0 Hz passband (~42–180 BPM) using two cascaded bandpass filters for steeper rolloff
      bp1 = new Biquad('bandpass', TARGET_FPS, 1.8, 0.75);
      bp2 = new Biquad('bandpass', TARGET_FPS, 1.8, 0.75);
    }

    function pushSample(t, x){ times[wr]=t; samples[wr]=x; wr=(wr+1)%MAX_SAMPLES; count=Math.min(count+1, MAX_SAMPLES); }

    function processFrame(){
      if (!v.videoWidth) return;
      ctx.drawImage(v, 0, 0, cvs.width, cvs.height);
      const frame = ctx.getImageData(0,0,cvs.width,cvs.height);

      const cov = coverageMetric(frame); updateStatus('coverage', `Coverage: ${(cov*100|0)}%`);
      const sat = saturationMetric(frame); updateStatus('sat', `Saturation: ${(sat*100|0)}%`);

      // Mean red and green (use ratio to reduce illumination dependency)
      let sumR=0, sumG=0; const data=frame.data; for(let i=0;i<data.length;i+=4){ sumR+=data[i]; sumG+=data[i+1]; }
      const meanR = sumR / (data.length/4); const meanG = sumG / (data.length/4);
      const raw = meanR / (meanG+1e-6);

      // Detrend + bandpass (two-stage for sharper band)
      const d = detrend(raw);
      const y1 = bp1.process(d);
      const y2 = bp2.process(y1);

      const now = performance.now()/1000;
      pushSample(now, y2);

      // FPS
      const dt = now - (processFrame.lastTs || now); processFrame.lastTs = now; const fps = 1/Math.max(dt,1/120); updateStatus('fr', 'FPS: '+fps.toFixed(0));

      // Quality hints
      if (cov < 0.25) setQuality('Poor','Cover camera fully with fingertip.');
      else if (sat > 0.15) setQuality('Fair','Too bright; ease pressure or move finger slightly.');
      else if (fps < 20) setQuality('Fair','Low frame rate. Keep device cool/close other apps.');
      else setQuality('Good','Hold steady. Breathe normally.');
    }

    // Autocorrelation-based BPM every 1s from last 8–12s window
    function estimateBPM(){
      const n = Math.min(count, MAX_SAMPLES);
      if (n < TARGET_FPS*4) return; // need at least 4s
      const W = Math.min(n, TARGET_FPS*10); // use last up to 10s
      const buf = new Float32Array(W);
      for (let i=0;i<W;i++){ const idx=(wr - 1 - i + MAX_SAMPLES)%MAX_SAMPLES; buf[W-1-i]=samples[idx]; }

      // Normalize
      let mean=0; for(let i=0;i<W;i++) mean+=buf[i]; mean/=W; let varr=0; for(let i=0;i<W;i++){ buf[i]-=mean; varr+=buf[i]*buf[i]; }
      const norm = Math.sqrt(varr/W)+1e-6; for(let i=0;i<W;i++) buf[i]/=norm;

      // Autocorrelation via direct method (W small, ok)
      const minBPM=40, maxBPM=180; const minLag=Math.floor(TARGET_FPS*60/maxBPM), maxLag=Math.floor(TARGET_FPS*60/minBPM);
      let bestLag=-1, bestVal=-1, second=0;
      for(let lag=minLag; lag<=maxLag; lag++){
        let s=0; for(let i=lag;i<W;i++) s += buf[i]*buf[i-lag];
        if (s>bestVal){ second=bestVal; bestVal=s; bestLag=lag; } else if (s>second) { second=s; }
      }
      if (bestLag<0) return;

      // Parabolic interpolation around best lag
      const L=bestLag; const f=(lag)=>{ let s=0; for(let i=lag;i<W;i++) s += buf[i]*buf[i-lag]; return s; };
      const y1=f(Math.max(minLag,L-1)), y2=bestVal, y3=f(Math.min(maxLag,L+1));
      const denom = (y1 - 2*y2 + y3);
      const delta = denom!==0 ? 0.5*(y1 - y3)/denom : 0; // -1..1
      const refinedLag = clamp(L + delta, minLag, maxLag);

      let bpm = 60 * TARGET_FPS / refinedLag;

      // Confidence: peak height vs zero-lag and vs second best
      const conf = clamp((bestVal - second) / Math.max(1e-6, 1 - second), 0, 1);
      document.getElementById('conf').textContent = `Confidence: ${(conf*100|0)}%`;

      // Median + exponential smoothing
      bpmHist.push(bpm); if (bpmHist.length > BPM_HIST_N) bpmHist.shift();
      const median = [...bpmHist].sort((a,b)=>a-b)[Math.floor(bpmHist.length/2)];
      const alpha=0.2; estimateBPM.smooth = (estimateBPM.smooth??median)*(1-alpha) + median*alpha;
      const finalBPM = Math.round(estimateBPM.smooth);

      // Sanity + outlier gate
      if (finalBPM<40 || finalBPM>180 || conf<0.2){ document.getElementById('bpm').textContent='—'; setQuality('Poor','Unstable signal; adjust finger/lighting.'); return; }
      document.getElementById('bpm').textContent = finalBPM;
    }

    // Drawing waveform
    function draw(){
      const W=drawCvs.width, H=drawCvs.height; const ctx=drawCtx;
      ctx.clearRect(0,0,W,H); ctx.fillStyle='#081325'; ctx.fillRect(0,0,W,H);
      ctx.strokeStyle='#1f2b47'; ctx.lineWidth=1; ctx.beginPath();
      for(let x=0;x<W;x+=Math.floor(W/12)){ ctx.moveTo(x,0); ctx.lineTo(x,H); }
      for(let y=0;y<H;y+=Math.floor(H/6)){ ctx.moveTo(0,y); ctx.lineTo(W,y); }
      ctx.stroke();
      ctx.beginPath(); ctx.lineWidth=2; ctx.strokeStyle='#7dd3fc';
      const n=Math.min(count,MAX_SAMPLES);
      if(n>2){ const start=(wr-n+MAX_SAMPLES)%MAX_SAMPLES; const t0=times[start];
        for(let i=0;i<n;i++){ const idx=(start+i)%MAX_SAMPLES; const x=(times[idx]-t0)/BUFFER_SECS; const y=samples[idx]; const px=x*W; const py=H*0.5 - y*(H*0.35); if(i===0) ctx.moveTo(px,py); else ctx.lineTo(px,py); }
        ctx.stroke();
      }
      rafId=requestAnimationFrame(draw);
    }

    // ========== Camera ==========
    async function start(){
      if (stream) return;
      try{
        stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: {ideal:'environment'}, width:{ideal:640}, height:{ideal:480}, frameRate:{ideal:60, max:60} }, audio:false });
        v.srcObject = stream; preview.srcObject = stream; track = stream.getVideoTracks()[0]; await v.play(); await preview.play();

        imageCapture = ('ImageCapture' in window) ? new ImageCapture(track) : null;
        torchAvailable=false;
        if (track.getCapabilities){ const caps=track.getCapabilities(); torchAvailable=!!caps.torch; }
        updateStatus('torchState', torchAvailable? 'Torch: available' : 'Torch: not available');
        document.getElementById('torchBtn').disabled = !torchAvailable;

        // Try continuous exposure/white-balance if supported
        try{ await track.applyConstraints({ advanced:[{ focusMode:'continuous' }, { exposureMode:'continuous' }, { whiteBalanceMode:'continuous' }] }); } catch(e){}

        if (torchAvailable) { try{ await track.applyConstraints({ advanced:[{ torch:true }] }); torchOn=true; updateStatus('torchState','Torch: on'); } catch(e){ log('Torch enable failed:', e.message); } }

        initDSP(); runProcessing();
        document.getElementById('startBtn').disabled=true; document.getElementById('stopBtn').disabled=false;
      } catch(err){ log('Camera error:', err.message); alert('Could not access the camera. Please allow camera permissions and use HTTPS (or localhost).'); }
    }

    async function toggleTorch(){ if(!track || !track.applyConstraints) return; try{ torchOn=!torchOn; await track.applyConstraints({ advanced:[{ torch: torchOn }] }); updateStatus('torchState', 'Torch: '+(torchOn?'on':'off')); } catch(e){ log('Torch toggle failed:', e.message); }}

    function stop(){ if(rafId) cancelAnimationFrame(rafId); if(procId) cancelAnimationFrame(procId); if(estId) clearInterval(estId); rafId=procId=null; if(stream){ stream.getTracks().forEach(t=>t.stop()); stream=null; } document.getElementById('startBtn').disabled=false; document.getElementById('stopBtn').disabled=true; updateStatus('fr','FPS: —'); updateStatus('coverage','Coverage: —'); updateStatus('torchState', torchOn?'Torch: on':'Torch: —'); document.getElementById('bpm').textContent='—'; setQuality('Poor','Stopped. Tap Start to measure again.'); }

    function initCanvas(){ drawCvs=document.getElementById('wave'); drawCtx=drawCvs.getContext('2d'); const dpr=Math.min(2, window.devicePixelRatio||1); const rect=drawCvs.getBoundingClientRect(); drawCvs.width=Math.floor(rect.width*dpr); drawCvs.height=Math.floor(rect.height*dpr); cvs=document.createElement('canvas'); ctx=cvs.getContext('2d',{willReadFrequently:true}); cvs.width=160; cvs.height=120; }

    function runProcessing(){ const loop=()=>{ processFrame(); procId=requestAnimationFrame(loop); }; loop(); draw(); estId = setInterval(estimateBPM, 1000); }

    // ========== DOM ==========
    window.addEventListener('load',()=>{ v=document.getElementById('video'); preview=document.getElementById('preview'); initCanvas(); window.addEventListener('resize', initCanvas); document.getElementById('startBtn').addEventListener('click', start); document.getElementById('stopBtn').addEventListener('click', stop); document.getElementById('torchBtn').addEventListener('click', toggleTorch); setQuality('Poor','Tap Start, then place your fingertip gently over the rear camera.'); });
  </script>
</body>
</html>
