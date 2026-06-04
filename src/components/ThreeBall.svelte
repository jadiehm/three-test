<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  
  import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
  import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
  import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
  import { ShaderPass } from 'three/addons/postprocessing/ShaderPass.js';

  // ── DYNAMIC GLOBAL CONFIGURATIONS ────────────────────────────────────
  let primaryColor = $state('#0052B4');   
  let secondaryColor = $state('#FFDA44'); 
  let flagImage = $state('assets/images/ba.png'); 
  let countryName = $state('Bosnia and Herzegovina');

  let canvas = $state();
  let textureLoader;
  let flagTexture;

  const getThreeColor = (hex) => new THREE.Color(hex);

  // ── 1. Panel Vertex Shader ───────────────────────────────────────────
  const panelVert = `
    uniform float uPuff;
    attribute float aPuffWeight;
    varying vec3  vWorldNormal;
    varying vec3  vNormal;
    varying vec3  vViewPosition;
    varying float vPuffW;

    void main() {
      vPuffW = aPuffWeight;
      vec3 puffed     = position + normal * aPuffWeight * uPuff;
      vec4 mvPosition = modelViewMatrix * vec4(puffed, 1.0);
      vViewPosition   = -mvPosition.xyz;
      vWorldNormal    = normalize((modelMatrix * vec4(normal, 0.0)).xyz);
      vNormal         = normalize(normalMatrix * normal);
      gl_Position     = projectionMatrix * mvPosition;
    }
  `;

  // ── 2. Panel Fragment Shader ─────────────────────────────────────────
  const panelFrag = `
    uniform float uTime;
    uniform vec3 uPrimaryColor;
    uniform vec3 uSecondaryColor;
    varying vec3  vNormal;
    varying vec3  vViewPosition;
    varying float vPuffW;

    void main() {
      vec3  N     = normalize(vNormal);
      vec3  V     = normalize(vViewPosition);
      float NdotV = clamp(dot(N, V), 0.0, 1.0);
      float fresnel = pow(1.0 - NdotV, 3.0);

      float edgeSlope = pow(1.0 - vPuffW, 4.0);
      vec3 surface = vec3(0.002, 0.003, 0.004);
      
      vec3 L_Red  = normalize(vec3(-4.0, -5.0, 3.0));
      vec3 L_Blue = normalize(vec3(4.0, 5.0, 4.0));
      
      float specRed  = pow(max(dot(N, normalize(L_Red + V)), 0.0), 150.0);
      float specBlue = pow(max(dot(N, normalize(L_Blue + V)), 0.0), 150.0);
      
      surface += uPrimaryColor * specRed * 0.45;
      surface += uSecondaryColor * specBlue * 0.45;

      vec3 ridgeColor = vec3(0.22, 0.25, 0.32) * (0.1 + 0.9 * fresnel);
      surface = mix(surface, ridgeColor + vec3((specRed + specBlue) * 0.15), edgeSlope);

      float alpha = mix(0.06 + fresnel * 0.35, 0.96, edgeSlope);
      gl_FragColor = vec4(surface, alpha);
    }
  `;

  const coreVert = `
    varying vec3 vWorldNormal;
    varying vec3 vNormal;
    varying vec3 vViewPosition;

    void main() {
      vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
      vViewPosition   = -mvPosition.xyz;
      vWorldNormal    = normalize((modelMatrix * vec4(position, 0.0)).xyz);
      vNormal         = normalize(normalMatrix * normal);
      gl_Position     = projectionMatrix * mvPosition;
    }
  `;

  // ── 3. Solid Glass Core Fragment Shader ──────────────────────────────
  const coreFrag = `
    uniform sampler2D uFlagTex;
    uniform float uTime;
    uniform float uTheta;
    uniform float uPhi;
    uniform vec3 uPrimaryColor;
    uniform vec3 uSecondaryColor;

    varying vec3 vWorldNormal;
    varying vec3 vNormal;
    varying vec3 vViewPosition;

    float hash3(vec3 p) {
      p = fract(p * vec3(443.8975, 397.2973, 491.1871));
      p += dot(p.xyz, p.yzx + 19.19);
      return fract(p.x * p.y * p.z);
    }

    vec2 rotateUV(vec2 uv, float rotation) {
      float mid = 0.5;
      return vec2(
        cos(rotation) * (uv.x - mid) + sin(rotation) * (uv.y - mid) + mid,
        cos(rotation) * (uv.y - mid) - sin(rotation) * (uv.x - mid) + mid
      );
    }

    vec2 getFluidOffset(vec3 p, float t, float spinOffset) {
      vec3 q = vec3(sin(p.x * 1.5 + t * 0.8 + spinOffset), cos(p.y * 1.2 + t * 0.6 - spinOffset), sin(p.z * 1.4 + t * 0.7));
      vec3 r = vec3(sin(p.z + q.x * 2.0 + t * 0.4), cos(p.x + q.y * 1.8 + t * 0.5), sin(p.y + q.z * 2.2 + t * 0.3));
      return vec2(sin(p.x + r.y * 2.5 + t * 0.6), cos(p.y * r.x * 2.5 + t * 0.5)) * 0.35;
    }

    void main() {
      vec3  N     = normalize(vNormal);
      vec3  V     = normalize(vViewPosition);
      float NdotV = clamp(dot(N, V), 0.0, 1.0);
      float fresnel = pow(1.0 - NdotV, 2.5);
      vec3 wp = vWorldNormal;

      float noise = hash3(wp * 450.0 + sin(uTime * 0.05)) * 0.10;

      float totalSpin = uTheta * 1.2 + uPhi * 0.8;
      vec2 fluidWarp = getFluidOffset(wp * 1.5, uTime * 1.2, totalSpin);
      
      vec2 baseUv = vec2(atan(wp.z, wp.x) / 6.283185 + 0.5, wp.y * 0.5 + 0.5);
      baseUv = rotateUV(baseUv, uTheta * 0.5);
      vec2 warpedUv = baseUv + fluidWarp;
      
      float blurScale = 0.035 + (fresnel * 0.05); 
      vec2 b1 = vec2(blurScale, blurScale); vec2 b2 = vec2(-blurScale, blurScale);
      
      vec3 flagSample = vec3(0.0);
      flagSample.r += texture2D(uFlagTex, warpedUv + b1).r * 0.25;
      flagSample.r += texture2D(uFlagTex, warpedUv - b1).r * 0.25;
      flagSample.g += texture2D(uFlagTex, warpedUv + b2).g * 0.25;
      flagSample.g += texture2D(uFlagTex, warpedUv - b2).g * 0.25;
      flagSample.b += texture2D(uFlagTex, warpedUv + vec2(blurScale, 0.0)).b * 0.5;
      flagSample.b += texture2D(uFlagTex, warpedUv - vec2(blurScale, 0.0)).b * 0.5;

      float iridPhase = fresnel * 8.0 + wp.y * 1.5 + uTime * 0.1;
      vec3 iridSpectrum = 0.5 + 0.5 * cos(6.28318 * (iridPhase + vec3(0.0, 0.33, 0.67)));

      vec3 surface = vec3(0.003, 0.005, 0.008) * NdotV + (flagSample * 0.70) + (iridSpectrum * fresnel * 0.60);
      surface += vec3(noise * (0.25 + fresnel * 0.75));

      float thicknessFactor = 1.0 - NdotV;
      surface = mix(surface, surface * vec3(0.20, 0.25, 0.35), thicknessFactor * 0.65);

      vec3 reflectDir = reflect(-V, N);
      float m = 2.0 * sqrt(pow(reflectDir.x, 2.0) + pow(reflectDir.y, 2.0) + pow(reflectDir.z + 1.0, 2.0));
      vec2 sphereMapUv = reflectDir.xy / m + 0.5;

      vec3 redStudioGlint  = uPrimaryColor * smoothstep(0.4, 0.85, sphereMapUv.y) * 0.14;
      vec3 blueStudioGlint = uSecondaryColor * smoothstep(0.6, 0.2, sphereMapUv.x) * 0.14;
      surface += redStudioGlint + blueStudioGlint;

      gl_FragColor = vec4(surface, 1.0);
    }
  `;

  // ── 4. Screen-Space Blur Filter Pass ─────────────────────────────────
  const postProcessShader = {
    uniforms: {
      tDiffuse: { value: null },
      uNoiseSeed: { value: 0.0 },
      uDispersion: { value: 0.0045 } 
    },
    vertexShader: `
      varying vec2 vUv;
      void main() {
        vUv = uv;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
      }
    `,
    fragmentShader: `
      uniform sampler2D tDiffuse;
      uniform float uNoiseSeed;
      uniform float uDispersion;
      varying vec2 vUv;

      float screenNoise(vec2 co) {
        return fract(sin(dot(co, vec2(12.9898, 78.233) + uNoiseSeed)) * 43758.5453);
      }

      void main() {
        vec2 centerOffset = vUv - 0.5;
        float distFromCenter = length(centerOffset);
        
        float tiltShiftBlur = smoothstep(0.32, 0.52, distFromCenter) * 0.015;

        vec3 color = vec3(0.0);
        float totalWeight = 0.0;
        
        for (float x = -2.0; x <= 2.0; x += 1.0) {
          for (float y = -2.0; y <= 2.0; y += 1.0) {
            vec2 offset = vec2(x, y) * tiltShiftBlur;
            vec2 sampleUv = vUv + offset;
            
            vec3 sampledColor;
            sampledColor.r = texture2D(tDiffuse, 0.5 + (sampleUv - 0.5) * (1.0 + uDispersion * 1.5)).r;
            sampledColor.g = texture2D(tDiffuse, sampleUv).g;
            sampledColor.b = texture2D(tDiffuse, 0.5 + (sampleUv - 0.5) * (1.0 - uDispersion * 1.5)).b;
            
            float weight = 1.0 - (length(vec2(x,y)) / 4.0);
            color += sampledColor * weight;
            totalWeight += weight;
          }
        }
        color /= totalWeight;

        float grain = (screenNoise(vUv) - 0.5) * 0.055;
        color += vec3(grain);

        float vignette = smoothstep(0.75, 0.35, distFromCenter);
        color *= mix(0.35, 1.0, vignette);

        gl_FragColor = vec4(color, 1.0);
      }
    `
  };

  onMount(() => {
    if (!canvas) return;

    let width = canvas.clientWidth || 300;
    let height = canvas.clientHeight || 300;

    textureLoader = new THREE.TextureLoader();
    
    const configureTexture = (tex) => {
      tex.wrapS = THREE.RepeatWrapping;
      tex.wrapT = THREE.RepeatWrapping;
      tex.repeat.set(0.75, 0.75); 
      tex.offset.set(0.12, 0.12);
    };

    flagTexture = textureLoader.load(flagImage);
    configureTexture(flagTexture);

    // ── GENERATE TRUNCATED ICOSAHEDRON (32-PANEL SOCCER BALL) ────────────
    const phiGold = (1 + Math.sqrt(5)) / 2; 
    const raw = [];

    // Truncated Icosahedron base point configurations
    function evp(a, b, c) {
      for (const [x, y, z] of [[a, b, c], [b, c, a], [c, a, b]]) {
        for (const sx of [1, -1]) {
          for (const sy of [1, -1]) {
            for (const sz of [1, -1]) {
              raw.push([sx * x, sy * y, sz * z]);
            }
          }
        }
      }
    }
    // Standard coordinates defining the 60 vertices of a classic soccer ball
    evp(0, 1, 3 * phiGold);
    evp(2, 1 + 2 * phiGold, phiGold);
    evp(1, 2 + phiGold, 2 * phiGold);

    const vMap = new Map(), V = [];
    raw.forEach(v => {
      const k = v.map(x => Math.round(x * 1000)).join(',');
      if (!vMap.has(k)) {
        vMap.set(k, V.length);
        V.push(v);
      }
    });

    const Vn = V.map(v => {
      const l = Math.sqrt(v[0]**2 + v[1]**2 + v[2]**2);
      return new THREE.Vector3(v[0]/l, v[1]/l, v[2]/l);
    });

    const ds = [];
    for (let i = 0; i < V.length; i++) {
      for (let j = i + 1; j < V.length; j++) {
        const dx = V[i][0] - V[j][0], dy = V[i][1] - V[j][1], dz = V[i][2] - V[j][2];
        ds.push(Math.sqrt(dx*dx + dy*dy + dz*dz));
      }
    }
    ds.sort((a, b) => a - b);
    const THR = ds[0] * 1.1; // Find adjacency threshold across edge links

    const adj = Array.from({ length: 60 }, () => []);
    for (let i = 0; i < V.length; i++) {
      for (let j = i + 1; j < V.length; j++) {
        const dx = V[i][0] - V[j][0], dy = V[i][1] - V[j][1], dz = V[i][2] - V[j][2];
        if (Math.sqrt(dx*dx + dy*dy + dz*dz) < THR) {
          adj[i].push(j);
          adj[j].push(i);
        }
      }
    }

    function turnRight(prev, cur) {
      const ns = adj[cur].filter(n => n !== prev);
      const iD = Vn[cur].clone().sub(Vn[prev]).normalize();
      const nm = Vn[cur];
      let ba = -Infinity, bn = -1;
      for (const n of ns) {
        const oD = Vn[n].clone().sub(Vn[cur]).normalize();
        const cr = new THREE.Vector3().crossVectors(iD, oD);
        const ang = Math.atan2(cr.dot(nm), iD.dot(oD));
        if (ang > ba) { ba = ang; bn = n; }
      }
      return bn;
    }

    const faceSet = new Set(), faces = [];
    for (let i = 0; i < V.length; i++) {
      for (const j of adj[i]) {
        let prev = i, cur = j;
        const fv = [i];
        for (let s = 0; s < 7; s++) {
          fv.push(cur);
          const nx = turnRight(prev, cur);
          if (nx === i || nx === -1) break;
          prev = cur;
          cur = nx;
        }
        if (fv.length >= 5 && fv.length <= 6) {
          const k = [...fv].sort((a, b) => a - b).join(',');
          if (!faceSet.has(k)) {
            faceSet.add(k);
            faces.push([...fv]);
          }
        }
      }
    }
    
    // Splits exactly into 12 pentagons and 20 hexagons
    const pentagons = faces.filter(f => f.length === 5);
    const hexagons = faces.filter(f => f.length === 6);

    function orderCCW(vi) {
      const c = new THREE.Vector3(); vi.forEach(i => c.add(Vn[i])); c.normalize();
      const r = new THREE.Vector3(0, 1, 0.01).sub(c.clone().multiplyScalar(c.dot(new THREE.Vector3(0, 1, 0.01)))).normalize();
      const u = new THREE.Vector3().crossVectors(c, r).normalize();
      return [...vi].sort((a, b) => {
        const da = Vn[a].clone().sub(c), db = Vn[b].clone().sub(c);
        return Math.atan2(da.dot(u), da.dot(r)) - Math.atan2(db.dot(u), db.dot(r));
      });
    }
    function centroid(vi) { const c = new THREE.Vector3(); vi.forEach(i => c.add(Vn[i])); return c.normalize(); }

    function buildGeo(vi, inset = 0.045, sub = 14) {
      const ord = orderCCW(vi);
      const cn = new THREE.Vector3(); ord.forEach(i => cn.add(Vn[i])); cn.normalize();
      const iv = ord.map(i => Vn[i].clone().lerp(cn, inset).normalize());
      const N = ord.length, pos = [], nm = [], pw = [], idx = [];
      let vc = 0; const vm2 = new Map();
      function av(v, p) {
        const k = `${Math.round(v.x * 1e5)},${Math.round(v.y * 1e5)},${Math.round(v.z * 1e5)}`;
        if (vm2.has(k)) return vm2.get(k);
        const id = vc++; vm2.set(k, id);
        const pt = v.clone().multiplyScalar(2.26);
        pos.push(pt.x, pt.y, pt.z); nm.push(v.x, v.y, v.z); pw.push(p); return id;
      }
      for (let fi = 0; fi < N; fi++) {
        const A = cn.clone(), B = iv[fi].clone(), C = iv[(fi + 1) % N].clone();
        const grid = [];
        for (let i = 0; i <= sub; i++) {
          grid.push([]);
          for (let j = 0; j <= sub - i; j++) {
            const ba = i / sub;
            const puffCurve = Math.pow(Math.sin(ba * Math.PI * 0.5), 0.35);
            const pt = A.clone().multiplyScalar(ba).add(B.clone().multiplyScalar(j / sub)).add(C.clone().multiplyScalar((sub - i - j) / sub)).normalize();
            grid[i].push(av(pt, puffCurve));
          }
        }
        const gi = (i, j) => grid[i][j];
        for (let i = 0; i < sub; i++) for (let j = 0; j < sub - i; j++) {
          idx.push(gi(i, j), gi(i + 1, j), gi(i, j + 1));
          if (j < sub - i - 1) idx.push(gi(i + 1, j), gi(i + 1, j + 1), gi(i, j + 1));
        }
      }
      const geo = new THREE.BufferGeometry();
      geo.setAttribute('position',    new THREE.BufferAttribute(new Float32Array(pos), 3));
      geo.setAttribute('normal',      new THREE.BufferAttribute(new Float32Array(nm), 3));
      geo.setAttribute('aPuffWeight', new THREE.BufferAttribute(new Float32Array(pw), 1));
      geo.setIndex(idx);
      return geo;
    }

    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x010102); 
    
    const camera = new THREE.PerspectiveCamera(50, width / height, 0.1, 100);
    camera.position.set(0, 0, 8);
    
    const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setSize(width, height);
    renderer.setPixelRatio(Math.min(devicePixelRatio, 2));

    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.toneMappingExposure = 0.82; 

    const group = new THREE.Group(); scene.add(group);
    const panels = [], panelMats = [];
    const sharedTime = { value: 0 };
    const sharedTheta = { value: 0 };
    const sharedPhi = { value: 0 };

    // ── LAYER 1: SOLID GLASS CORE SPHERE ──────────────────────────────────
    const coreMat = new THREE.ShaderMaterial({
      vertexShader: coreVert, fragmentShader: coreFrag,
      uniforms: {
        uTime: sharedTime, uTheta: sharedTheta, uPhi: sharedPhi,
        uFlagTex: { value: flagTexture },
        uPrimaryColor: { value: getThreeColor(primaryColor) },
        uSecondaryColor: { value: getThreeColor(secondaryColor) }
      }
    });
    const coreMesh = new THREE.Mesh(new THREE.SphereGeometry(2.175, 64, 64), coreMat);
    group.add(coreMesh);

    // ── LAYER 2: THE BEVELED PANEL JOINTS OVERLAY ────────────────────────
    function addPanel(vi) {
      const geo = buildGeo(vi, 0.055, 14); // Cleaner, slightly wider gaps for 32-panel layout
      const mat = new THREE.ShaderMaterial({
        vertexShader: panelVert, fragmentShader: panelFrag,
        uniforms: { 
          uTime: sharedTime, 
          uPuff: { value: -0.085 },
          uPrimaryColor: { value: getThreeColor(primaryColor) },
          uSecondaryColor: { value: getThreeColor(secondaryColor) }
        }, 
        transparent: true, depthWrite: true, blending: THREE.NormalBlending
      });
      panelMats.push(mat);
      const mesh = new THREE.Mesh(geo, mat);
      mesh.userData.normal = centroid(vi);
      group.add(mesh); panels.push(mesh);
    }

    // Directly maps panels without complex sub-splitting rules
    pentagons.forEach(f => addPanel(orderCCW(f)));
    hexagons.forEach(f => addPanel(orderCCW(f)));

    // ── POST-PROCESSING PIPELINE STACK ───────────────────────────────────
    const composer = new EffectComposer(renderer);
    composer.setSize(width, height);
    composer.addPass(new RenderPass(scene, camera));

    const bloomPass = new UnrealBloomPass(new THREE.Vector2(width, height), 0.55, 1.35, 0.0);
    composer.addPass(bloomPass);

    const photoFilterPass = new ShaderPass(postProcessShader);
    composer.addPass(photoFilterPass);

    // ── INTERACTION LOGISTICS (Auto-Rotation + Mouse Tilt) ───────────────
    let autoTheta = 0;
    let targetTiltX = 0, targetTiltY = 0;
    let currentTiltX = 0, currentTiltY = 0;
    
    const onMouseMove = e => {
      const rect = canvas.getBoundingClientRect();
      const x = ((e.clientX - rect.left) / rect.width) * 2 - 1; 
      const y = -((e.clientY - rect.top) / rect.height) * 2 + 1; 
      
      targetTiltX = x * (Math.PI * 0.35); 
      targetTiltY = y * (Math.PI * 0.35); 
    };

    const onTouchMove = e => {
      const rect = canvas.getBoundingClientRect();
      const t = e.touches[0];
      const x = ((t.clientX - rect.left) / rect.width) * 2 - 1;
      const y = -((t.clientY - rect.top) / rect.height) * 2 + 1;
      
      targetTiltX = x * (Math.PI * 0.35);
      targetTiltY = y * (Math.PI * 0.35);
    };

    window.addEventListener('mousemove', onMouseMove);
    window.addEventListener('touchmove', onTouchMove, { passive: true });

    // ── REACTIVE RUNE WATCHERS ───────────────────────────────────────────
    $effect(() => {
      const pCol = getThreeColor(primaryColor);
      coreMat.uniforms.uPrimaryColor.value.copy(pCol);
      panelMats.forEach(m => m.uniforms.uPrimaryColor.value.copy(pCol));
    });

    $effect(() => {
      const sCol = getThreeColor(secondaryColor);
      coreMat.uniforms.uSecondaryColor.value.copy(sCol);
      panelMats.forEach(m => m.uniforms.uSecondaryColor.value.copy(sCol));
    });

    $effect(() => {
      if (textureLoader) {
        textureLoader.load(flagImage, (newTex) => {
          configureTexture(newTex);
          const oldTex = coreMat.uniforms.uFlagTex.value;
          coreMat.uniforms.uFlagTex.value = newTex;
          if (oldTex) oldTex.dispose(); 
        });
      }
    });

    const ro = new ResizeObserver(() => {
      const w = canvas.clientWidth;
      const h = canvas.clientHeight;
      if (w === 0 || h === 0) return; 

      camera.aspect = w / h;
      camera.updateProjectionMatrix();
      renderer.setSize(w, h);
      composer.setSize(w, h);
    });
    ro.observe(canvas);

    let rafId;
    const animate = () => {
      rafId = requestAnimationFrame(animate);
      sharedTime.value += 0.004;
      
      photoFilterPass.uniforms.uNoiseSeed.value = Math.random() * 100.0;

      autoTheta += 0.004; 

      currentTiltX += (targetTiltX - currentTiltX) * 0.08;
      currentTiltY += (targetTiltY - currentTiltY) * 0.08;

      group.rotation.y = autoTheta + currentTiltX;
      group.rotation.x = currentTiltY;
      
      sharedTheta.value = autoTheta + currentTiltX;
      sharedPhi.value = currentTiltY;

      composer.render();
    };
    animate();

    return () => {
      cancelAnimationFrame(rafId); ro.disconnect();
      window.removeEventListener('mousemove', onMouseMove);
      window.removeEventListener('touchmove', onTouchMove);
      panelMats.forEach(m => m.dispose()); coreMat.dispose(); if (flagTexture) flagTexture.dispose(); composer.dispose(); renderer.dispose();
    };
  });
</script>

<div class="header-text-container" style="--primary-color: {primaryColor}; --secondary-color: {secondaryColor};">
    <h1 class="gradient-text" data-text="How the world searches for soccer">How the world searches for soccer</h1>
</div>
<div class="wrapper">
  <div class="canvas-container">
    <canvas bind:this={canvas}></canvas>
  </div>
</div>
<h3 class="country-name">{countryName}</h3>

<style>
  .wrapper { 
    position: absolute;
    top: -4rem; left: 0; bottom: 0; right: 0;
    width: 100%; 
    height: 100svh; 
    overflow: hidden; 
     z-index: 1;
  }
  
  .canvas-container {
    position: relative;
    width: 100%;
    height: 100%;
    mix-blend-mode: screen; 
    z-index: 1;
  }
  
  canvas { 
    display: block; 
    width: 100%; 
    height: 100%; 
  }

  .header-text-container {
    width: 100%;
    display: flex;
    justify-content: center;
    align-items: center;
    padding: 2rem 2rem 0 2rem;
    position: relative;
    z-index: 1000;
  }

  .gradient-text {
    font-family: var(--sans);
    font-size: 2rem;
    font-weight: 500;
    letter-spacing: -0.03em;
    margin: 0;
    background-clip: text;
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-image: linear-gradient(
      135deg,
      var(--primary-color) 0%,
      var(--secondary-color) 25%,
      var(--primary-color) 50%,
      var(--secondary-color) 75%,
      var(--primary-color) 100%
    );
    background-size: 400% 100%;
    animation: fluid-flow 8s linear infinite;
    position: relative;
  }

  .gradient-text::after {
    content: attr(data-text);
    position: absolute;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
    z-index: -1;
    
    /* 1. Explicitly clear out all vector clipping fields */
    background-clip: unset;
    -webkit-background-clip: unset;
    background-image: none;
    
    /* 2. Set text structural fill to clear so only the shadow draws */
    -webkit-text-fill-color: transparent;
    color: transparent;
    
    /* 3. Cast high-intensity neon blurs via CSS shadow vectors using your colors */
    text-shadow: 
      0 0 10px #ffffff,
      0 0 25px #ffffff,
      0 0 50px #ffffff;
      
    /* 4. Soften the overall light bleed and drop opacity so it stays elegant */
    filter: blur(4px);
    opacity: 0.2;
    
    /* Keep animation rules to maintain precise sync matching main tag bounds */
    animation: fluid-flow 8s linear infinite;
  }

  @keyframes fluid-flow {
    0% {
      background-position: 0% 50%;
    }
    50% {
      background-position: 100% 50%;
    }
    100% {
      background-position: 0% 50%;
    }
  }

  .country-name {
    position: absolute;
    bottom: 0;
    left: 50%;
    transform: translateX(-50%);
    font-family: var(--sans);
    font-weight: 500;
    font-size: 200px;
    text-align: center;
    color: #ffffff;
    z-index: 3;
    padding:0;
    margin: 0;
    line-height: 1;
  }
</style>