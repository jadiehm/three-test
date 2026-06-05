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
    scene.background = new THREE.Color(0x060608);

    const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
    camera.position.set(0, 0, 4);

    const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(window.devicePixelRatio);

    const ambientLight = new THREE.AmbientLight(0xffffff, 0.04);
    scene.add(ambientLight);

    const pointLight = new THREE.PointLight(0xffffff, 2.0, 100);
    scene.add(pointLight);

    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
    controls.enableZoom = false;
    controls.enablePan = false;

    // ── Volumetric interior blob shader ───────────────────────────
    // A sphere slightly smaller than the ball, rendered with a
    // ray-marched volume shader so color fills the interior as
    // smooth swirling blobs — contained entirely inside the glass.

    const blobVert = `
      varying vec3 vPos;
      varying vec3 vNormal;
      void main() {
        vPos = position;
        vNormal = normal;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
      }
    `;

    const blobFrag = `
      uniform float time;
      varying vec3 vPos;
      varying vec3 vNormal;

      // Smooth noise via value noise
      vec3 hash3(vec3 p) {
        p = fract(p * vec3(443.8975, 397.2973, 491.1871));
        p += dot(p, p.yxz + 19.19);
        return fract((p.xxy + p.yzz) * p.zyx);
      }

      float noise(vec3 p) {
        vec3 i = floor(p);
        vec3 f = fract(p);
        vec3 u = f * f * (3.0 - 2.0 * f);
        return mix(
          mix(mix(dot(hash3(i + vec3(0,0,0)), f - vec3(0,0,0)),
                  dot(hash3(i + vec3(1,0,0)), f - vec3(1,0,0)), u.x),
              mix(dot(hash3(i + vec3(0,1,0)), f - vec3(0,1,0)),
                  dot(hash3(i + vec3(1,1,0)), f - vec3(1,1,0)), u.x), u.y),
          mix(mix(dot(hash3(i + vec3(0,0,1)), f - vec3(0,0,1)),
                  dot(hash3(i + vec3(1,0,1)), f - vec3(1,0,1)), u.x),
              mix(dot(hash3(i + vec3(0,1,1)), f - vec3(0,1,1)),
                  dot(hash3(i + vec3(1,1,1)), f - vec3(1,1,1)), u.x), u.y),
          u.z
        );
      }

      float fbm(vec3 p) {
        float v = 0.0;
        float a = 0.5;
        for (int i = 0; i < 5; i++) {
          v += a * noise(p);
          p = p * 2.1 + vec3(1.7, 9.2, 3.4);
          a *= 0.5;
        }
        return v;
      }

      vec3 colorA() { return vec3(0.05, 0.12, 0.9); }   // deep blue
      vec3 colorB() { return vec3(0.9, 0.08, 0.15); }   // vivid red
      vec3 colorC() { return vec3(0.6, 0.05, 0.7); }    // purple accent

      void main() {
        vec3 p = vPos * 2.2;
        float t = time * 0.18;

        // Two fbm layers drifting in different directions
        float n1 = fbm(p + vec3(t * 0.7, t * 0.4, t * 0.5));
        float n2 = fbm(p * 0.8 - vec3(t * 0.5, t * 0.8, t * 0.3) + 4.3);
        float n3 = fbm(p * 1.4 + vec3(t * 0.3, -t * 0.6, t * 0.7) + 8.7);

        // Blend into color regions
        float blend1 = smoothstep(0.3, 0.7, n1 + n2 * 0.4);
        float blend2 = smoothstep(0.2, 0.8, n2 * n3);

        vec3 col = mix(colorA(), colorB(), blend1);
        col = mix(col, colorC(), blend2 * 0.45);

        // Brighten center, darken edge (subsurface feel)
        float rim = 1.0 - pow(abs(dot(normalize(vNormal), vec3(0.0, 0.0, 1.0))), 0.6);
        col = mix(col * 1.4, col * 0.5, rim * 0.5);

        // Edge glow — blue-red chromatic rim
        vec3 rimCol = mix(vec3(0.2, 0.3, 1.0), vec3(1.0, 0.1, 0.2), sin(time * 0.3) * 0.5 + 0.5);
        col += rimCol * pow(rim, 2.5) * 1.2;

        // Slight overall luminance boost so it glows through glass
        col = pow(col, vec3(0.85));

        gl_FragColor = vec4(col, 0.92);
      }
    `;

    // Sphere sized to sit just inside the glass ball (radius ~0.47 in model space)
    // The GLTF is scaled 3x so this inner sphere is scaled to match
    const innerSphere = new THREE.Mesh(
      new THREE.SphereGeometry(0.47, 64, 64),
      new THREE.ShaderMaterial({
        vertexShader: blobVert,
        fragmentShader: blobFrag,
        uniforms: { time: { value: 0 } },
        transparent: true,
        depthWrite: false,
        side: THREE.FrontSide,
        blending: THREE.NormalBlending,
      })
    );
    // Will be added after model loads so we can position correctly

    // Soft colored point lights to tint the glass surface itself
    const lightDefs = [
      { color: 0x0033ff, speed: 0.22, phase: 0.0 },
      { color: 0xff0022, speed: 0.31, phase: 2.1 },
      { color: 0x8800cc, speed: 0.19, phase: 4.2 },
    ];
    const innerLights = lightDefs.map(({ color, speed, phase }) => {
      const light = new THREE.PointLight(color, 4.0, 12);
      scene.add(light);
      return { light, speed, phase };
    });
    // ── End volumetric ─────────────────────────────────────────────

    const loader = new GLTFLoader();
    loader.load(
      'https://jadiehm.github.io/three-test/assets/ball.gltf',
      (gltf) => {
        model = gltf.scene;
        model.scale.set(3, 3, 3);
        const box = new THREE.Box3().setFromObject(model);
        const center = box.getCenter(new THREE.Vector3());
        model.position.sub(center);
        scene.add(model);

        // Place inner sphere at the same center, scaled to match model
        innerSphere.scale.set(3, 3, 3);
        innerSphere.position.copy(model.position);
        // Render before the glass so it shows through
        innerSphere.renderOrder = -1;
        scene.add(innerSphere);
      },
      undefined,
      (e) => console.error(e)
    );

    const mouse = new THREE.Vector2();
    window.addEventListener('mousemove', (e) => {
      mouse.x = (e.clientX / window.innerWidth) * 2 - 1;
      mouse.y = -(e.clientY / window.innerHeight) * 2 + 1;
    });

    const clock = new THREE.Clock();
    const animate = () => {
      requestAnimationFrame(animate);
      const elapsed = clock.getElapsedTime();

      if (model) {
        model.rotation.y += 0.006;
        innerSphere.rotation.y = model.rotation.y * 0.4;
        innerSphere.rotation.x = Math.sin(elapsed * 0.15) * 0.3;
      }

      innerSphere.material.uniforms.time.value = elapsed;

      innerLights.forEach(({ light, speed, phase }) => {
        const t = elapsed * speed + phase;
        light.position.set(
          Math.cos(t) * 2.0,
          Math.sin(t * 0.7) * 2.0,
          Math.sin(t) * 2.0
        );
      });

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