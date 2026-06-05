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
  let model;        
  let fluidSphere;  
  let fluidMaterial;

  // ── 1. SVELTE 5 RUNES ─────────────────────────────────────────────
  // Change fluidOpacity to values like 0.4 or 0.6 to see it turn translucent!
  let { 
    fluidColors = ['#338AF3', '#D80027', '#013C83'],
    fluidOpacity = 0.6 
  } = $props();

  $effect(() => {
    if (fluidMaterial) {
      fluidMaterial.uniforms.uColor1.value.set(fluidColors[0]);
      fluidMaterial.uniforms.uColor2.value.set(fluidColors[1]);
      fluidMaterial.uniforms.uColor3.value.set(fluidColors[2]);
      fluidMaterial.uniforms.uOpacity.value = fluidOpacity;
    }
  });
  // ─────────────────────────────────────────────────────────────────

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
    uniform vec3 uColor1;
    uniform vec3 uColor2;
    uniform vec3 uColor3;
    uniform float uOpacity; // Incoming opacity control

    varying vec3 vPosition;
    varying vec3 vNormal;

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
      vec4 p = permute(permute(permute(i.z + vec4(0.0, i1.z, i2.z, 1.0))+ i.y + vec4(0.0, i1.y, i2.y, 1.0))+ i.x + vec4(0.0, i1.x, i2.x, 1.0));
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
      p0 *= norm.x; p1 *= norm.y; p2 *= norm.z; p3 *= norm.w;
      vec4 m = max(0.6 - vec4(dot(x0, x0), dot(x1, x1), dot(x2, x2), dot(x3, x3)), 0.0);
      m = m * m;
      return 42.0 * dot(m * m, vec4(dot(p0, x0), dot(p1, x1), dot(p2, x2), dot(p3, x3)));
    }

    void main() {
      vec3 noiseCoord1 = vPosition * 1.8 + vec3(0.0, uTime * 0.3, 0.0);
      vec3 noiseCoord2 = vPosition * 3.5 - vec3(uTime * 0.1, 0.0, uTime * 0.2);

      float n1 = smoothstep(0.2, 0.8, snoise(noiseCoord1) * 0.5 + 0.5);
      float n2 = smoothstep(0.3, 0.7, snoise(noiseCoord2) * 0.5 + 0.5);

      vec3 cFlagColor1 = uColor1;
      vec3 cFlagColor2 = uColor2;
      vec3 cFlagColor3 = uColor3;

      vec3 mixedColor = mix(cFlagColor1, cFlagColor2, n1);
      mixedColor = mix(mixedColor, cFlagColor3, n2 * 0.6);

      float fresnel = pow(1.0 - clamp(dot(vNormal, vec3(0.0, 0.0, 1.0)), 0.0, 1.0), 3.0);
      mixedColor = mix(mixedColor, mixedColor * 0.2, fresnel * 0.7);

      // FIX: Blend the fluid colors smoothly into a dark backdrop inside the shader.
      // This mimics real volumetric alpha transparency perfectly without breaking glass passes.
      vec3 darkVoid = vec3(0.008, 0.008, 0.015); 
      vec3 translucentColor = mix(darkVoid, mixedColor, uOpacity);

      gl_FragColor = vec4(translucentColor * 2.2, 1.0);
    }
  `;

  onMount(() => {
    if (!canvas) return;

    const W = canvas.clientWidth  || window.innerWidth;
    const H = canvas.clientHeight || window.innerHeight;

    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x020204);

    const camera = new THREE.PerspectiveCamera(50, W / H, 0.1, 100);
    camera.position.set(0, 0, 2);

    const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setSize(W, H);
    renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.toneMappingExposure = 1.0;

    new HDRLoader().load(
      'https://dl.polyhaven.org/file/ph-assets/HDRIs/hdr/1k/studio_small_08_1k.hdr',
      (hdr) => {
        hdr.mapping = THREE.EquirectangularReflectionMapping;
        scene.environment = hdr;
      }
    );

    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;

    const composer = new EffectComposer(renderer);
    composer.setSize(W, H);
    composer.addPass(new RenderPass(scene, camera));

    const bloomPass = new UnrealBloomPass(new THREE.Vector2(W, H), 0.5, 0.4, 0.25); 
    composer.addPass(bloomPass);
    composer.addPass(new OutputPass());

    const glassMaterial = new THREE.MeshPhysicalMaterial({
      transmission: 1.0, // Fixed: Kept at max realistic value 1.0 (1.1 breaks depth buffers)
      ior: 1.45,
      roughness: 0.08,
      clearcoat: 1.0,
      thickness: 0.15,
      side: THREE.FrontSide
    });

    // FIX: Kept transparent: false and depthWrite: true so it stays in the opaque pipeline
    fluidMaterial = new THREE.ShaderMaterial({
      vertexShader: fluidVertShader,
      fragmentShader: fluidFragShader,
      transparent: false, 
      depthWrite: true,
      uniforms: {
        uTime:   { value: 0 },
        uColor1: { value: new THREE.Color(fluidColors[0]) },
        uColor2: { value: new THREE.Color(fluidColors[1]) },
        uColor3: { value: new THREE.Color(fluidColors[2]) },
        uOpacity: { value: fluidOpacity }
      }
    });

    const loader = new GLTFLoader();
    loader.load(
      'https://jadiehm.github.io/three-test/assets/soccer_ball_orm_normal.glb',
      (gltf) => {
        model = gltf.scene;
        const box    = new THREE.Box3().setFromObject(model);
        const center = box.getCenter(new THREE.Vector3());
        const size   = box.getSize(new THREE.Vector3());
        const radius = Math.max(size.x, size.y, size.z) / 2;

        model.position.sub(center);

        model.traverse((child) => {
          if (child.isMesh) {
            if (child.name.toLowerCase().includes('wire') || child.scale.x < 0.9) {
              child.visible = false; 
            } else {
              child.material = glassMaterial;
              child.renderOrder = 2; 
            }
          }
        });
        scene.add(model);

        const fluidGeo = new THREE.SphereGeometry(radius * 0.92, 64, 64);
        fluidSphere = new THREE.Mesh(fluidGeo, fluidMaterial);
        fluidSphere.renderOrder = 1; 
        scene.add(fluidSphere);
      }
    );

    const clock = new THREE.Clock();
    const animate = () => {
      requestAnimationFrame(animate);
      const elapsed = clock.getElapsedTime();
      
      if (fluidMaterial) fluidMaterial.uniforms.uTime.value = elapsed;

      if (model) model.rotation.y = elapsed * 0.12;
      if (fluidSphere) {
        fluidSphere.rotation.y = -elapsed * 0.22;
        fluidSphere.rotation.z = Math.sin(elapsed * 0.15) * 0.15;
      }

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
      glassMaterial.dispose();
      if (fluidMaterial) fluidMaterial.dispose();
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