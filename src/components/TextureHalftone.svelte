<script>
  let canvas = $state(null);
  let imageSrc = $state('https://picsum.photos/800/1000');
  let cellSize = $state(6);
  let bgThreshold = $state(235);
  let showGraphics = $state(true);

  function hash2D(x, y) {
    const sin = Math.sin(x * 12.9898 + y * 78.233);
    return sin - Math.floor(sin);
  }

  // Organic Brain/Marble noise generator
  function getFluidPattern(x, y) {
    const scale = 0.045;
    const warp = Math.sin(x * scale + Math.sin(y * scale) * 2.5);
    return Math.sin(y * scale * 1.5 + warp * 3.0) > 0.1;
  }

  // Interlocking Circle Check
  function isInsidePlanetGraphic(x, y, w, h) {
    const cx = w * 0.73;
    const cy = h * 0.47;
    const radius = w * 0.09;
    return Math.hypot(x - cx, y - cy) <= radius;
  }

  function renderArt() {
    if (!canvas || !imageSrc) return;
    const ctx = canvas.getContext('2d');
    const img = new Image();
    img.crossOrigin = 'Anonymous';
    img.src = imageSrc;

    img.onload = () => {
      const w = img.width;
      const h = img.height;
      canvas.width = w;
      canvas.height = h;

      const offCanvas = document.createElement('canvas');
      offCanvas.width = w;
      offCanvas.height = h;
      const offCtx = offCanvas.getContext('2d');
      offCtx.drawImage(img, 0, 0);

      const imgData = offCtx.getImageData(0, 0, w, h);
      const d = imgData.data;

      // Base paper background fill
      ctx.fillStyle = '#E3D4C4';
      ctx.fillRect(0, 0, w, h);

      // --- LAYER 1: Background Geometric Graphics ---
      if (showGraphics) {
        ctx.strokeStyle = '#141414';
        ctx.fillStyle = '#141414';

        // Top-Left Frame Box with Stippling
        const boxX = w * 0.1, boxY = h * 0.08, boxSize = w * 0.32;
        ctx.strokeRect(boxX, boxY, boxSize, boxSize);
        ctx.lineWidth = 3;
        for (let py = boxY; py < boxY + boxSize; py += 3) {
          for (let px = boxX; px < boxX + boxSize; px += 3) {
            if (hash2D(px, py) < 0.25) ctx.fillRect(px, py, 2, 2);
          }
        }

        // Circular Planet Graphic
        const cx = w * 0.73, cy = h * 0.47, radius = w * 0.09;
        ctx.beginPath();
        ctx.arc(cx, cy, radius, 0, Math.PI * 2);
        ctx.lineWidth = 2;
        ctx.stroke();

        // Bottom Accent Filmstrip
        const fy = h * 0.78;
        ctx.fillRect(w * 0.16, fy, w * 0.6, 4);
        for (let i = 0; i < 14; i++) {
          ctx.fillRect(w * 0.17 + i * 30, fy - 6, 10, 14);
        }
      }

      // --- LAYER 2: Subject Texture Mapping ---
      for (let y = 0; y < h; y += cellSize) {
        for (let x = 0; x < w; x += cellSize) {
          const idx = (y * w + x) * 4;
          const lum = 0.299 * d[idx] + 0.587 * d[idx + 1] + 0.114 * d[idx + 2];
          const alpha = d[idx + 3];

          const isSubject = alpha >= 50 && lum < bgThreshold;
          if (!isSubject) continue;

          ctx.save();
          ctx.translate(x + cellSize / 2, y + cellSize / 2);

          // Color inversion when overlapping background planet graphic
          const inPlanet = showGraphics && isInsidePlanetGraphic(x, y, w, h);
          const inkColor = inPlanet ? '#E3D4C4' : '#141414';
          ctx.fillStyle = inkColor;
          ctx.strokeStyle = inkColor;

          // 1. SHADOWS: Solid Fill
          if (lum < 45) {
            ctx.fillRect(-cellSize / 2, -cellSize / 2, cellSize, cellSize);
          }
          // 2. DARK MIDTONES: Organic Fluid / Marble Pattern
          else if (lum >= 45 && lum < 95) {
            if (getFluidPattern(x, y)) {
              ctx.fillRect(-cellSize / 2, -cellSize / 2, cellSize, cellSize);
            }
          }
          // 3. MIDTONES: Interlocking Vertical & Diagonal Hatching
          else if (lum >= 95 && lum < 165) {
            if ((Math.floor(x / 24) % 2) === 0) {
              ctx.fillRect(-cellSize / 4, -cellSize / 2, cellSize / 2, cellSize);
            } else {
              ctx.rotate(Math.PI / 4);
              ctx.fillRect(-cellSize / 2, -cellSize / 4, cellSize, cellSize / 2);
            }
          }
          // 4. HIGHLIGHTS: Rotated Halftone Dots
          else {
            ctx.beginPath();
            ctx.arc(0, 0, cellSize * 0.25, 0, Math.PI * 2);
            ctx.fill();
          }

          ctx.restore();
        }
      }
    };
  }

  function handleUpload(e) {
    const file = e.target.files?.[0];
    if (file) {
      const reader = new FileReader();
      reader.onload = (evt) => { imageSrc = evt.target.result; };
      reader.readAsDataURL(file);
    }
  }

  $effect(() => {
    if (
      canvas &&
      imageSrc &&
      cellSize &&
      bgThreshold !== undefined &&
      showGraphics !== undefined
    ) {
      renderArt();
    }
  });
</script>

<div class="workspace">
  <div class="toolbar">
    <input type="file" accept="image/*" onchange={handleUpload} />

    <label>
      Grid Size: {cellSize}px
      <input type="range" min="3" max="12" bind:value={cellSize} />
    </label>

    <label>
      Ignore BG: {bgThreshold}
      <input type="range" min="150" max="255" bind:value={bgThreshold} />
    </label>

    <label class="checkbox">
      <input type="checkbox" bind:checked={showGraphics} />
      Show Background Graphics
    </label>
  </div>

  <canvas bind:this={canvas}></canvas>
</div>

<style>
  .workspace {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 1.5rem;
    background: #0d0d0d;
    padding: 2rem;
    color: #e3d4c4;
    font-family: monospace;
  }
  .toolbar {
    display: flex;
    flex-wrap: wrap;
    gap: 1.25rem;
    align-items: center;
    background: #1a1a1a;
    padding: 1rem 1.5rem;
    border-radius: 8px;
    border: 1px solid #333;
  }
  label {
    display: flex;
    flex-direction: column;
    gap: 0.3rem;
    font-size: 0.85rem;
  }
  .checkbox {
    flex-direction: row;
    align-items: center;
    cursor: pointer;
  }
  canvas {
    max-width: 100%;
    height: auto;
    border-radius: 4px;
    box-shadow: 0 15px 35px rgba(0, 0, 0, 0.7);
  }
</style>