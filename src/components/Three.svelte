<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

  let canvas;
  let model;

  onMount(() => {
    if (!canvas) return;

    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x111111);

    const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
    camera.position.set(0, 0, 4);

    const renderer = new THREE.WebGLRenderer({
      canvas: canvas,
      antialias: true,
    });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(window.devicePixelRatio);

    const ambientLight = new THREE.AmbientLight(0xffffff, 0.2);
    scene.add(ambientLight);

    const directionalLight = new THREE.DirectionalLight(0xffffff, 0.5);
    directionalLight.position.set(5, 10, 7.5);
    scene.add(directionalLight);

    const pointLight = new THREE.PointLight(0xffffff, 1.5, 100);
    scene.add(pointLight);

    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
    controls.screenSpacePanning = false;
    controls.enableZoom = false;
    controls.enablePan = false;
    controls.autoRotate = false;

    // ── Aurora Effect ──────────────────────────────────────────────
    const auroraGroup = new THREE.Group();
    scene.add(auroraGroup);

    const colorBlue = new THREE.Color(0x0052B4);
    const colorRed  = new THREE.Color(0xD80027);

    const auroraVertexShader = `
      attribute float bandAngle;
      attribute float bandOffset;
      attribute float bandPhase;
      uniform float time;
      varying float vAlpha;
      varying vec3 vColor;
      uniform vec3 colorA;
      uniform vec3 colorB;

      void main() {
        float speed  = 0.4 + bandPhase * 0.2;
        float angle  = bandAngle + time * speed;

        float phi    = acos(bandOffset);
        float theta  = angle;
        float radius = 0.52;
        float wave   = sin(angle * 3.0 + time * 1.2 + bandPhase) * 0.04;

        vec3 pos = vec3(
          (radius + wave) * sin(phi) * cos(theta),
          (radius + wave) * cos(phi),
          (radius + wave) * sin(phi) * sin(theta)
        );

        vAlpha = sin(phi) * 0.9;

        float blend = sin(bandPhase + time * 0.5) * 0.5 + 0.5;
        vColor = mix(colorA, colorB, blend);

        gl_Position  = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
        gl_PointSize = 5.5;
      }
    `;

    const auroraFragmentShader = `
      varying float vAlpha;
      varying vec3 vColor;

      void main() {
        float d = length(gl_PointCoord - vec2(0.5));
        if (d > 0.5) discard;
        float alpha = (1.0 - d * 2.0) * vAlpha;
        gl_FragColor = vec4(vColor, alpha);
      }
    `;

    const BANDS       = 8;
    const PER_BAND    = 2500;
    const TOTAL       = BANDS * PER_BAND;

    const positions   = new Float32Array(TOTAL * 3);
    const bandAngles  = new Float32Array(TOTAL);
    const bandOffsets = new Float32Array(TOTAL);
    const bandPhases  = new Float32Array(TOTAL);

    for (let b = 0; b < BANDS; b++) {
      const phaseShift = (b / BANDS) * Math.PI * 2;
      for (let i = 0; i < PER_BAND; i++) {
        const idx = b * PER_BAND + i;
        bandAngles[idx]  = (i / PER_BAND) * Math.PI * 2;
        bandOffsets[idx] = (i / PER_BAND) * 2 - 1;
        bandPhases[idx]  = phaseShift;
        positions[idx * 3]     = 0;
        positions[idx * 3 + 1] = 0;
        positions[idx * 3 + 2] = 0;
      }
    }

    const auroraGeo = new THREE.BufferGeometry();
    auroraGeo.setAttribute('position',   new THREE.BufferAttribute(positions,   3));
    auroraGeo.setAttribute('bandAngle',  new THREE.BufferAttribute(bandAngles,  1));
    auroraGeo.setAttribute('bandOffset', new THREE.BufferAttribute(bandOffsets, 1));
    auroraGeo.setAttribute('bandPhase',  new THREE.BufferAttribute(bandPhases,  1));

    const auroraMat = new THREE.ShaderMaterial({
      vertexShader:   auroraVertexShader,
      fragmentShader: auroraFragmentShader,
      uniforms: {
        time:   { value: 0 },
        colorA: { value: colorBlue },
        colorB: { value: colorRed  },
      },
      transparent: true,
      depthWrite:  false,
      blending:    THREE.AdditiveBlending,
    });

    const auroraParticles = new THREE.Points(auroraGeo, auroraMat);
    auroraGroup.add(auroraParticles);
    // ── End Aurora ─────────────────────────────────────────────────

    const loader = new GLTFLoader();
    const modelURL = 'https://jadiehm.github.io/three-test/assets/ball.gltf';

    loader.load(
      modelURL,
      (gltf) => {
        model = gltf.scene;
        model.scale.set(3, 3, 3);

        const box = new THREE.Box3().setFromObject(model);
        const center = box.getCenter(new THREE.Vector3());
        model.position.sub(center);

        scene.add(model);
        auroraGroup.scale.set(5, 5, 5);
      },
      (xhr) => {
        console.log((xhr.loaded / xhr.total * 100) + '% loaded');
      },
      (error) => {
        console.error('An error happened while loading the GLTF model:', error);
      }
    );

    const mouse = new THREE.Vector2();
    const handleMouseMove = (event) => {
      mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
      mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
    };
    window.addEventListener('mousemove', handleMouseMove);

    const clock = new THREE.Clock();

    const animate = () => {
      requestAnimationFrame(animate);

      const elapsed = clock.getElapsedTime();

      if (model) {
        model.rotation.y += 0.01;
        auroraGroup.rotation.y = model.rotation.y;
      }

      auroraMat.uniforms.time.value = elapsed;

      pointLight.position.set(mouse.x * 5, mouse.y * 5, 2);

      controls.update();
      renderer.render(scene, camera);
    };

    animate();

    const handleResize = () => {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
      renderer.setPixelRatio(window.devicePixelRatio);
    };

    window.addEventListener('resize', handleResize);

    return () => {
      window.removeEventListener('resize', handleResize);
      window.removeEventListener('mousemove', handleMouseMove);
      auroraGeo.dispose();
      auroraMat.dispose();
    };
  });
</script>

<canvas bind:this={canvas}></canvas>

<style>
  :global(body, html) {
    margin: 0;
    padding: 0;
    width: 100%;
    height: 100%;
    overflow: hidden;
  }

  canvas {
    display: block;
    width: 100%;
    height: 100%;
  }
</style>