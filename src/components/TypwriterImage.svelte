<script>
  let canvas = $state(null);
  let imageSrc = $state('https://picsum.photos/800/1000');
  let cellSize = $state(7);
  let density = $state(3);
  let edgeSensitivity = $state(35); // Sobel threshold for detecting edges
  let edgeBoost = $state(2.5);       // Stacking multiplier along edges
  let bgThreshold = $state(230);

  function pseudoRandom(seed) {
    const x = Math.sin(seed) * 10000;
    return x - Math.floor(x);
  }

  // Sobel Operator pass: returns grayscale, alpha, gradient magnitude, and directional angle
  function processImageData(ctx, width, height) {
    const imgData = ctx.getImageData(0, 0, width, height);
    const d = imgData.data;
    const gray = new Float32Array(width * height);
    const alpha = new Uint8Array(width * height);

    for (let i = 0; i < d.length; i += 4) {
      const idx = i / 4;
      gray[idx] = 0.299 * d[i] + 0.587 * d[i + 1] + 0.114 * d[i + 2];
      alpha[idx] = d[i + 3];
    }

    const mag = new Float32Array(width * height);
    const angle = new Float32Array(width * height);

    for (let y = 1; y < height - 1; y++) {
      for (let x = 1; x < width - 1; x++) {
        const idx = y * width + x;
        const gx =
          -gray[idx - width - 1] + gray[idx - width + 1] -
          2 * gray[idx - 1] + 2 * gray[idx + 1] -
          gray[idx + width - 1] + gray[idx + width + 1];

        const gy =
          -gray[idx - width - 1] - 2 * gray[idx - width] - gray[idx - width + 1] +
          gray[idx + width - 1] + 2 * gray[idx + width] + gray[idx + width + 1];

        mag[idx] = Math.sqrt(gx * gx + gy * gy);
        angle[idx] = Math.atan2(gy, gx); // Direction of edge gradient
      }
    }

    return { gray, alpha, mag, angle };
  }

  function renderTypewriterCollage() {
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

      const { gray, alpha, mag, angle } = processImageData(offCtx, w, h);

      // Cream paper background
      ctx.fillStyle = '#E8DFCE';
      ctx.fillRect(0, 0, w, h);

      const fontSize = cellSize * 1.35;
      ctx.font = `bold ${fontSize}px "Courier New", Courier, monospace`;
      ctx.textAlign = 'center';
      ctx.textBaseline = 'middle';

      for (let y = 0; y < h; y += cellSize) {
        for (let x = 0; x < w; x += cellSize) {
          let totalLum = 0;
          let totalAlpha = 0;
          let maxEdgeMag = 0;
          let dominantAngle = 0;
          let count = 0;

          // Sample block statistics
          for (let cy = 0; cy < cellSize && (y + cy) < h; cy++) {
            for (let cx = 0; cx < cellSize && (x + cx) < w; cx++) {
              const idx = (y + cy) * w + (x + cx);
              totalLum += gray[idx];
              totalAlpha += alpha[idx];

              if (mag[idx] > maxEdgeMag) {
                maxEdgeMag = mag[idx];
                dominantAngle = angle[idx];
              }
              count++;
            }
          }

          const avgLum = totalLum / count;
          const avgAlpha = totalAlpha / count;

          // Ignore background pixels
          if (avgAlpha < 50 || avgLum >= bgThreshold) {
            continue;
          }

          const isEdge = maxEdgeMag > edgeSensitivity;

          // Select character pool
          let charPool = [];
          let stackLayers = 1;

          if (isEdge) {
            // Edge Characters: Linear/heavy glyphs that define contours cleanly
            charPool = ['M', 'W', 'N', 'E', 'H', '|', '/', '\\', 'X', 'B'];
            stackLayers = Math.floor(density * edgeBoost);
          } else if (avgLum < 45) {
            charPool = ['M', 'W', 'B', '@', '#', 'N', 'K'];
            stackLayers = Math.floor(density * 2.5);
          } else if (avgLum < 90) {
            charPool = ['B', 'D', 'H', 'R', 'E', 'O'];
            stackLayers = Math.floor(density * 1.8);
          } else if (avgLum < 145) {
            charPool = ['A', 'O', 'U', 'P', 'Z', 'C'];
            stackLayers = Math.max(1, Math.floor(density * 1.2));
          } else if (avgLum < 190) {
            charPool = ['T', 'J', 'I', 'L', 'Y', '1'];
            stackLayers = Math.max(1, Math.floor(density * 0.7));
          } else {
            charPool = ['.', ':', ';', '!', '/', '-'];
            stackLayers = 1;
          }

          const centerX = x + cellSize / 2;
          const centerY = y + cellSize / 2;

          for (let layer = 0; layer < stackLayers; layer++) {
            const seed = x * 12.9898 + y * 78.233 + layer * 43.123;
            const char = charPool[Math.floor(pseudoRandom(seed) * charPool.length)];

            let offsetX, offsetY, rotation;

            if (isEdge) {
              // EDGE LOGIC: Align letters along edge tangent (dominantAngle + 90 deg)
              // Keep scatter small so lines remain ultra-crisp
              const edgeTangent = dominantAngle + Math.PI / 2;
              const jitterMagnitude = (pseudoRandom(seed + 1) - 0.5) * (cellSize * 0.2);
              
              offsetX = Math.cos(edgeTangent) * jitterMagnitude;
              offsetY = Math.sin(edgeTangent) * jitterMagnitude;
              rotation = edgeTangent + (pseudoRandom(seed + 3) - 0.5) * 0.15;
            } else {
              // Standard fill logic: random positional scatter and rotational jitter
              offsetX = (pseudoRandom(seed + 1) - 0.5) * (cellSize * 0.5);
              offsetY = (pseudoRandom(seed + 2) - 0.5) * (cellSize * 0.5);
              rotation = (pseudoRandom(seed + 3) - 0.5) * 0.3;
            }

            // Darker ink opacity on edges to define boundaries
            const baseOpacity = isEdge ? 0.95 : Math.min(0.85, 0.35 + (1 - avgLum / 255) * 0.5);
            ctx.fillStyle = `rgba(15, 12, 10, ${baseOpacity})`;

            ctx.save();
            ctx.translate(centerX + offsetX, centerY + offsetY);
            ctx.rotate(rotation);
            ctx.fillText(char, 0, 0);
            ctx.restore();
          }
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
      density !== undefined &&
      edgeSensitivity !== undefined &&
      edgeBoost !== undefined &&
      bgThreshold !== undefined
    ) {
      renderTypewriterCollage();
    }
  });
</script>

<div class="workspace">
  <div class="toolbar">
    <input type="file" accept="image/*" onchange={handleUpload} />

    <label>
      Grid Size: {cellSize}px
      <input type="range" min="4" max="16" bind:value={cellSize} />
    </label>

    <label>
      Edge Sensitivity: {edgeSensitivity}
      <input type="range" min="10" max="100" bind:value={edgeSensitivity} />
    </label>

    <label>
      Edge Ink Density: {edgeBoost}x
      <input type="range" min="1" max="5" step="0.5" bind:value={edgeBoost} />
    </label>

    <label>
      Base Density: {density}x
      <input type="range" min="1" max="5" bind:value={density} />
    </label>

    <label>
      Ignore BG: {bgThreshold}
      <input type="range" min="150" max="255" bind:value={bgThreshold} />
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
    color: #e8dfce;
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
    font-size: 0.8rem;
  }
  canvas {
    max-width: 100%;
    height: auto;
    border-radius: 4px;
    box-shadow: 0 15px 35px rgba(0, 0, 0, 0.8);
  }
</style>