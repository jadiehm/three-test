<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
  import { HDRLoader } from 'three/addons/loaders/HDRLoader.js';
  import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
  import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
  import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
  import { OutputPass } from 'three/addons/postprocessing/OutputPass.js';

  let canvas;
  let model;        // Outer Glass Ball Container
  let fluidSphere;  // Inner Fluid Core Mesh

  // ── 2. Fluid Core GLSL Shaders ─────────────────────────────────────
  const fluidVertShader = `
    varying vec3 vPosition;
    varying vec3 vNormal;
    void main() {
      vPosition = position;
      vNormal = normalize(normalMatrix * normal);
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `;

  const fluidFragShader = `
    uniform float uTime;
    varying vec3 vPosition;
    varying vec3 vNormal;

    // Description : Array and textureless GLSL 2D/3D/4D simplex noise functions.
    //      Author : Ian McEwan, Ashima Arts.
    vec4 permute(vec4 x){return mod(((x*34.0)+1.0)*x, 289.0);}
    vec4 taylorInvSqrt(vec4 r){return 1.79284291400159 - 0.85373472095314 * r;}

    float snoise(vec3 v){
      const vec2 C = vec2(1.0/6.0, 1.0/3.0);
      const vec4 D = vec4(0.0, 0.5, 1.0, 2.0);

      vec3 i  = floor(v + dot(v, C.yyy));
      vec3 x0 = v - i + dot(i, C.xxx);

      vec3 g = step(x0.yzx, x0.xyz);
      vec3 l = 1.0 - g;
      vec3 i1 = min(g.xyz, l.zxy);
      vec3 i2 = max(g.xyz, l.zxy);

      vec3 x1 = x0 - i1 + 1.0 * C.xxx;
      vec3 x2 = x0 - i2 + 2.0 * C.xxx;
      vec3 x3 = x0 - D.yyy;

      i = mod(i, 289.0);
      vec4 p = permute(permute(permute(
                 i.z + vec4(0.0, i1.z, i2.z, 1.0))
               + i.y + vec4(0.0, i1.y, i2.y, 1.0))
               + i.x + vec4(0.0, i1.x, i2.x, 1.0));

      float n_ = 0.142857142857;
      vec3 ns = n_ * D.wyz - D.xzx;

      vec4 j = p - 49.0 * floor(p * ns.z * ns.z);

      vec4 x_ = floor(j * ns.z);
      vec4 y_ = floor(j - 7.0 * x_);

      vec4 x = x_ * ns.x + ns.yyyy;
      vec4 y = y_ * ns.x + ns.yyyy;
      vec4 h = 1.0 - abs(x) - abs(y);

      vec4 b0 = vec4(x.xy, y.xy);
      vec4 b1 = vec4(x.zw, y.zw);

      vec4 s0 = floor(b0) * 2.0 + 1.0;
      vec4 s1 = floor(b1) * 2.0 + 1.0;
      vec4 sh = -step(h, vec4(0.0));

      vec4 a0 = b0.xzyw + s0.xzyw * sh.xxyy;
      vec4 a1 = b1.xzyw + s1.xzyw * sh.zzww;

      vec3 p0 = vec3(a0.xy, h.x);
      vec3 p1 = vec3(a0.zw, h.y);
      vec3 p2 = vec3(a1.xy, h.z);
      vec3 p3 = vec3(a1.zw, h.w);

      vec4 norm = taylorInvSqrt(vec4(dot(p0, p0), dot(p1, p1), dot(p2, p2), dot(p3, p3)));
      p0 *= norm.x;
      p1 *= norm.y;
      p2 *= norm.z;
      p3 *= norm.w;

      vec4 m = max(0.6 - vec4(dot(x0, x0), dot(x1, x1), dot(x2, x2), dot(x3, x3)), 0.0);
      m = m * m;
      return 42.0 * dot(m * m, vec4(dot(p0, x0), dot(p1, x1), dot(p2, x2), dot(p3, x3)));
    }

    void main() {
      // Create morphing coordinate windows based on space and time
      vec3 noiseCoord1 = vPosition * 2.0 + vec3(0.0, uTime * 0.4, 0.0);
      vec3 noiseCoord2 = vPosition * 4.0 - vec3(uTime * 0.2, 0.0, uTime * 0.3);

      float n1 = snoise(noiseCoord1) * 0.5 + 0.5;
      float n2 = snoise(noiseCoord2) * 0.5 + 0.5;

      // Defining a vibrant palette (Deep Purple, Neon Magenta, Electric Cyan)
      vec3 cPurple  = vec3(0.24, 0.0, 0.51);
      vec3 cMagenta = vec3(1.0, 0.0, 0.43);
      vec3 cCyan    = vec3(0.0, 0.91, 1.0);

      // Interpolate colors dynamically using Simplex layers
      vec3 mixedColor = mix(cPurple, cMagenta, n1);
      mixedColor = mix(mixedColor, cCyan, n2 * 0.7);

      // Subtle edge darkening on the inner fluid core to accent depth
      float fresnel = pow(1.0 - clamp(dot(vNormal, vec3(0.0, 0.0, 1.0)), 0.0, 1.0), 2.0);
      mixedColor = mix(mixedColor, mixedColor * 0.4, fresnel * 0.5);

      // Overexpose the color properties (Factor 2.2) to push past 1.0 threshold for Bloom triggering
      gl_FragColor = vec4(mixedColor * 2.2, 1.0);
    }
  `;

  onMount(() => {
    if (!canvas) return;

    const W = canvas.clientWidth  || window.innerWidth;
    const H = canvas.clientHeight || window.innerHeight;

    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x030305);

    const camera = new THREE.PerspectiveCamera(50, W / H, 0.1, 100);
    camera.position.set(0, 0, 3.5);

    const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setSize(W, H);
    renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.toneMappingExposure = 1.0;

    // Environment Lighting Map
    new HDRLoader().load(
      'https://dl.polyhaven.org/file/ph-assets/HDRIs/hdr/1k/studio_small_08_1k.hdr',
      (hdr) => {
        hdr.mapping = THREE.EquirectangularReflectionMapping;
        scene.environment = hdr;
      }
    );

    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
    controls.enableZoom = true;

    // ── 3. Post-Processing Stack (Bloom Pass Configuration) ──────────
    const composer = new EffectComposer(renderer);
    composer.setSize(W, H);
    
    const renderPass = new RenderPass(scene, camera);
    composer.addPass(renderPass);

    const bloomPass = new UnrealBloomPass(
      new THREE.Vector2(W, H), 
      1.5,  // strength
      0.5,  // radius
      0.15  // threshold
    );
    composer.addPass(bloomPass);

    const outputPass = new OutputPass();
    composer.addPass(outputPass);

    // ── 1. Glass Material Design Setup ───────────────────────────────
    const glassMaterial = new THREE.MeshPhysicalMaterial({
      transmission: 1.0,
      ior: 1.52,
      roughness: 0.05,
      clearcoat: 1.0,
      clearcoatRoughness: 0.02,
      thickness: 0.35, // Simulates physical thickness and high refractions
      side: THREE.DoubleSide
    });

    const fluidMaterial = new THREE.ShaderMaterial({
      vertexShader: fluidVertShader,
      fragmentShader: fluidFragShader,
      uniforms: {
        uTime: { value: 0 }
      }
    });

    // Asset Loader
    const loader = new GLTFLoader();
    loader.load(
      'https://jadiehm.github.io/three-test/assets/ball.gltf',
      (gltf) => {
        model = gltf.scene;

        // Automatically position object to spatial center
        const box    = new THREE.Box3().setFromObject(model);
        const center = box.getCenter(new THREE.Vector3());
        const size   = box.getSize(new THREE.Vector3());
        const radius = Math.max(size.x, size.y, size.z) / 2;

        model.position.sub(center);

        // Assign Physical Glass properties across GLTF structure meshes
        model.traverse((child) => {
          if (child.isMesh) {
            child.material = glassMaterial;
          }
        });
        scene.add(model);

        // Dynamic fluid generation scaled to ~93% to cleanly nest inside
        const fluidGeo = new THREE.SphereGeometry(radius * 0.93, 64, 64);
        fluidSphere = new THREE.Mesh(fluidGeo, fluidMaterial);
        scene.add(fluidSphere);
      },
      undefined,
      (e) => console.error("Error loading GLTF asset:", e)
    );

    // ── 4. Animation Loop ────────────────────────────────────────────
    const clock = new THREE.Clock();
    
    const animate = () => {
      requestAnimationFrame(animate);
      
      const elapsed = clock.getElapsedTime();
      fluidMaterial.uniforms.uTime.value = elapsed;

      // Outer glass structure rotating gracefully on Y
      if (model) {
        model.rotation.y = elapsed * 0.12;
      }

      // Parallax Movement: Inner core counter-rotates and shifts off-axis
      if (fluidSphere) {
        fluidSphere.rotation.y = -elapsed * 0.22;
        fluidSphere.rotation.z = Math.sin(elapsed * 0.15) * 0.15;
      }

      controls.update();
      composer.render();
    };

    animate();

    // Window and Container Resize Handling
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
      glassMaterial.dispose();
      fluidMaterial.dispose();
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