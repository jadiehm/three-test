<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
  import { HDRLoader } from 'three/addons/loaders/HDRLoader.js';
  import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
  import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
  import { ShaderPass } from 'three/addons/postprocessing/ShaderPass.js';

  let canvas;
  let model;

  // ── EDIT THESE ────────────────────────────────────────────────────
  const HEX_A   = '#0052B4';
  const HEX_B   = '#D80027';
  const HEX_C   = '#ffffff';
  const HEX_RIM = '#1133ff';
  // ─────────────────────────────────────────────────────────────────

  const postProcessShader = {
    uniforms: {
      tDiffuse:    { value: null },
      uNoiseSeed:  { value: 0.0 },
      uDispersion: { value: 0.0045 },
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
        vec2  centerOffset   = vUv - 0.5;
        float distFromCenter = length(centerOffset);
        float tiltShiftBlur  = smoothstep(0.15, 0.55, distFromCenter) * 0.008;

        vec3  color       = vec3(0.0);
        float totalWeight = 0.0;

        for (float x = -2.0; x <= 2.0; x += 1.0) {
          for (float y = -2.0; y <= 2.0; y += 1.0) {
            vec2 offset   = vec2(x, y) * tiltShiftBlur;
            vec2 sampleUv = vUv + offset;

            vec3 sampledColor;
            sampledColor.r = texture2D(tDiffuse, 0.5 + (sampleUv - 0.5) * (1.0 + uDispersion * 1.5)).r;
            sampledColor.g = texture2D(tDiffuse, sampleUv).g;
            sampledColor.b = texture2D(tDiffuse, 0.5 + (sampleUv - 0.5) * (1.0 - uDispersion * 1.5)).b;

            float weight = 1.0 - (length(vec2(x, y)) / 4.0);
            color       += sampledColor * weight;
            totalWeight += weight;
          }
        }
        color /= totalWeight;

        float grain = (screenNoise(vUv) - 0.5) * 0.1;
        color += vec3(grain);

        float vignette = smoothstep(0.75, 0.35, distFromCenter);
        color *= mix(0.35, 1.0, vignette);

        gl_FragColor = vec4(color, 1.0);
      }
    `,
  };

  onMount(() => {
    if (!canvas) return;

    const W = canvas.clientWidth  || window.innerWidth;
    const H = canvas.clientHeight || window.innerHeight;

    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x060608);

    const camera = new THREE.PerspectiveCamera(75, W / H, 0.1, 1000);
    camera.position.set(1, 1, 1);

    const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setSize(W, H);
    renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.toneMappingExposure = 1.1;
    renderer.outputColorSpace = THREE.SRGBColorSpace;

    new HDRLoader().load(
      'https://dl.polyhaven.org/file/ph-assets/HDRIs/hdr/1k/studio_small_08_1k.hdr',
      (hdr) => {
        hdr.mapping = THREE.EquirectangularReflectionMapping;
        scene.environment = hdr;
      }
    );

    const mouseLight = new THREE.PointLight(0xffffff, 0.6, 100);
    scene.add(mouseLight);

    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
    controls.enableZoom = false;
    controls.enablePan = false;

    const composer = new EffectComposer(renderer);
    composer.setSize(W, H);
    composer.addPass(new RenderPass(scene, camera));
    const photoFilterPass = new ShaderPass(postProcessShader);
    composer.addPass(photoFilterPass);

    // ── Outer glow shell ─────────────────────────────────────────
    const glowVert = `
      varying vec3 vNormal;
      varying vec3 vViewDir;
      void main() {
        vNormal  = normalize(normalMatrix * normal);
        vec4 mv  = modelViewMatrix * vec4(position, 1.0);
        vViewDir = normalize(-mv.xyz);
        gl_Position = projectionMatrix * mv;
      }
    `;

    const glowFrag = `
      uniform vec3  glowColor;
      uniform float time;
      varying vec3 vNormal;
      varying vec3 vViewDir;

      void main() {
        float NdotV  = clamp(dot(normalize(vNormal), normalize(vViewDir)), 0.0, 1.0);
        // Pure rim — bright at edge, invisible in center
        float rim    = pow(1.0 - NdotV, 3.5);
        // Breathe the glow slightly
        float pulse  = 0.85 + 0.15 * sin(time * 0.8);
        float alpha  = rim * pulse * 0.5;
        gl_FragColor = vec4(glowColor * rim * pulse * 1.5, alpha);
      }
    `;

    const glowMat = new THREE.ShaderMaterial({
      vertexShader:   glowVert,
      fragmentShader: glowFrag,
      uniforms: {
        glowColor: { value: new THREE.Color(HEX_A) },
        time:      { value: 0 },
      },
      transparent: true,
      depthWrite:  false,
      side:        THREE.BackSide, // render inside-out so it surrounds the ball
      blending:    THREE.AdditiveBlending,
    });

    const ballVert = `
      varying vec3 vWorldNormal;
      varying vec3 vNormal;
      varying vec3 vViewDir;

      void main() {
        vWorldNormal    = normalize((modelMatrix * vec4(normal, 0.0)).xyz);
        vNormal         = normalize(normalMatrix * normal);
        vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
        vViewDir        = normalize(-mvPosition.xyz);
        gl_Position     = projectionMatrix * mvPosition;
      }
    `;

    const ballFrag = `
      uniform float time;
      uniform float uTheta;
      uniform vec3  colorA;
      uniform vec3  colorB;
      uniform vec3  colorC;
      uniform vec3  colorRim;

      varying vec3 vWorldNormal;
      varying vec3 vNormal;
      varying vec3 vViewDir;

      vec2 getFluidOffset(vec3 p, float t, float spinOffset) {
        vec3 q = vec3(
          sin(p.x * 1.5 + t * 0.8 + spinOffset),
          cos(p.y * 1.2 + t * 0.6 - spinOffset),
          sin(p.z * 1.4 + t * 0.7)
        );
        vec3 r = vec3(
          sin(p.z + q.x * 2.0 + t * 0.4),
          cos(p.x + q.y * 1.8 + t * 0.5),
          sin(p.y + q.z * 2.2 + t * 0.3)
        );
        return vec2(
          sin(p.x + r.y * 2.5 + t * 0.6),
          cos(p.y * r.x * 2.5 + t * 0.5)
        ) * 0.35;
      }

      vec2 rotateUV(vec2 uv, float rotation) {
        float mid = 0.5;
        return vec2(
          cos(rotation) * (uv.x - mid) + sin(rotation) * (uv.y - mid) + mid,
          cos(rotation) * (uv.y - mid) - sin(rotation) * (uv.x - mid) + mid
        );
      }

      vec3 threeColorFluid(vec3 wp, float t, float theta) {
        float totalSpin = theta * 1.2;
        vec2 fluidWarp  = getFluidOffset(wp * 1.5, t * 1.2, totalSpin);
        vec2 baseUV     = vec2(atan(wp.z, wp.x) / 6.28318 + 0.5, wp.y * 0.5 + 0.5);
        baseUV          = rotateUV(baseUV, theta * 0.5);
        vec2 warpedUV   = baseUV + fluidWarp;

        vec2 fluidWarp2 = getFluidOffset(wp * 1.1, t * 0.7, totalSpin + 2.1);
        vec2 baseUV2    = vec2(wp.y * 0.5 + 0.5, atan(wp.x, wp.z) / 6.28318 + 0.5);
        baseUV2         = rotateUV(baseUV2, theta * 0.3);
        vec2 warpedUV2  = baseUV2 + fluidWarp2;

        float blend1 = sin(warpedUV.x * 6.28318 * 1.5 + warpedUV.y * 3.14159) * 0.5 + 0.5;
        blend1 = smoothstep(0.18, 0.82, blend1);

        float blend2 = cos(warpedUV2.x * 6.28318 * 1.2 + warpedUV2.y * 4.0 + t * 0.15) * 0.5 + 0.5;
        blend2 = smoothstep(0.2, 0.8, blend2);

        vec3 ab = mix(colorA, colorB, blend1);
        return mix(ab, colorC, blend2 * 0.65);
      }

      void main() {
        vec3  N       = normalize(vNormal);
        vec3  V       = normalize(vViewDir);
        float NdotV   = clamp(dot(N, V), 0.0, 1.0);
        float fresnel = pow(1.0 - NdotV, 2.2);
        vec3  wp      = normalize(vWorldNormal);
        float t       = time * 0.38;

        vec3 fluidCol = threeColorFluid(wp, t, uTheta);

        float blurS = 0.025 + fresnel * 0.035;
        vec3 fluidR = threeColorFluid(normalize(vWorldNormal + vec3( blurS, 0.0, 0.0)), t, uTheta);
        vec3 fluidB = threeColorFluid(normalize(vWorldNormal + vec3(-blurS, 0.0, 0.0)), t, uTheta);
        fluidCol.r  = fluidR.r;
        fluidCol.b  = fluidB.b;

        float iridPhase = fresnel * 5.0 + wp.y * 1.5 + time * 0.07;
        vec3 irid = 0.5 + 0.5 * cos(6.28318 * (iridPhase + vec3(0.0, 0.33, 0.67)));
        irid = mix(irid, colorRim, 0.5);

        vec3  L1    = normalize(vec3(-3.0, -4.0, 3.0));
        vec3  L2    = normalize(vec3( 3.0,  4.0, 4.0));
        float spec1 = pow(max(dot(N, normalize(L1 + V)), 0.0), 120.0);
        float spec2 = pow(max(dot(N, normalize(L2 + V)), 0.0), 120.0);

        float groove = 1.0 - pow(NdotV, 0.5);

        vec3 col = fluidCol;
        col += irid * fresnel * 0.7;
        col += colorA * spec1 * 0.4;
        col += colorB * spec2 * 0.4;
        col += vec3(0.8, 0.9, 1.0) * (spec1 + spec2) * 0.3;
        col  = mix(col, col * 0.12, pow(groove, 6.0) * 0.85);
        col += colorRim * pow(fresnel, 3.0) * 1.1;

        float alpha = mix(0.82, 0.98, fresnel * 0.6 + groove * 0.4);
        gl_FragColor = vec4(col, alpha);
      }
    `;

    const fluidMat = new THREE.ShaderMaterial({
      vertexShader: ballVert,
      fragmentShader: ballFrag,
      uniforms: {
        time:     { value: 0 },
        uTheta:   { value: 0 },
        colorA:   { value: new THREE.Color(HEX_A) },
        colorB:   { value: new THREE.Color(HEX_B) },
        colorC:   { value: new THREE.Color(HEX_C) },
        colorRim: { value: new THREE.Color(HEX_RIM) },
      },
      transparent: true,
      depthWrite:  true,
      side:        THREE.FrontSide,
    });

    const lightDefs = [
      { color: HEX_A, speed: 0.22, phase: 0.0 },
      { color: HEX_B, speed: 0.31, phase: 2.1 },
      { color: HEX_C, speed: 0.19, phase: 4.2 },
    ];
    const dynLights = lightDefs.map(({ color, speed, phase }) => {
      const light = new THREE.PointLight(new THREE.Color(color), 3.5, 16);
      scene.add(light);
      return { light, speed, phase };
    });

    const loader = new GLTFLoader();
    loader.load(
      'https://jadiehm.github.io/three-test/assets/ball.gltf',
      (gltf) => {
        model = gltf.scene;
        model.scale.set(1, 1, 1);

        const box    = new THREE.Box3().setFromObject(model);
        const center = box.getCenter(new THREE.Vector3());
        const size   = box.getSize(new THREE.Vector3());
        const radius = Math.max(size.x, size.y, size.z) / 2;

        model.position.sub(center);

        model.traverse((child) => {
          if (child.isMesh) {
            child.material    = fluidMat;
            child.renderOrder = 0;
          }
        });

        scene.add(model);

        // Glow shell — 30% bigger than the ball, renders behind via BackSide
        const glowMesh = new THREE.Mesh(
          new THREE.SphereGeometry(radius * 1.02, 50, 50),
          glowMat
        );
        glowMesh.renderOrder = -1;
        scene.add(glowMesh);
      },
      undefined,
      (e) => console.error(e)
    );

    const mouse = new THREE.Vector2();
    window.addEventListener('mousemove', (e) => {
      mouse.x =  (e.clientX / window.innerWidth)  * 2 - 1;
      mouse.y = -(e.clientY / window.innerHeight) * 2 + 1;
    });

    let autoTheta = 0;
    const clock = new THREE.Clock();
    const animate = () => {
      requestAnimationFrame(animate);
      const elapsed = clock.getElapsedTime();

      autoTheta += 0.006;
      if (model) model.rotation.y = autoTheta;

      fluidMat.uniforms.time.value   = elapsed;
      fluidMat.uniforms.uTheta.value = autoTheta;
      glowMat.uniforms.time.value    = elapsed;

      // Cycle glow color between A and B over time
      const glowBlend = Math.sin(elapsed * 0.4) * 0.5 + 0.5;
      glowMat.uniforms.glowColor.value.set(0xffffff);

      photoFilterPass.uniforms.uNoiseSeed.value = Math.random() * 100.0;

      dynLights.forEach(({ light, speed, phase }) => {
        const t = elapsed * speed + phase;
        light.position.set(
          Math.cos(t) * 2.5,
          Math.sin(t * 0.7) * 2.5,
          Math.sin(t) * 2.5
        );
      });

      mouseLight.position.set(mouse.x * 5, mouse.y * 5, 3);
      controls.update();
      composer.render();
    };

    animate();

    const ro = new ResizeObserver(() => {
      const w = canvas.clientWidth;
      const h = canvas.clientHeight;
      if (!w || !h) return;
      camera.aspect = w / h;
      camera.updateProjectionMatrix();
      renderer.setSize(w, h);
      composer.setSize(w, h);
    });
    ro.observe(canvas);

    return () => {
      ro.disconnect();
      composer.dispose();
      renderer.dispose();
    };
  });
</script>

<canvas bind:this={canvas}></canvas>

<style>
  canvas {
    display: block;
    width: 100%;
    height: 100svh;
  }
</style>