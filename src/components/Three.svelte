<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

  let canvas;
  let wrapper;
  let model;
  let loaded = false;

  onMount(() => {
    if (!canvas) return;

    // ---------- scene ----------
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x0a0a0a);

    const camera = new THREE.PerspectiveCamera(
      45,
      canvas.clientWidth / canvas.clientHeight,
      0.1,
      1000
    );
    camera.position.set(0, 0, 4.2);
    camera.lookAt(0, 0, 0);

    const renderer = new THREE.WebGLRenderer({
      canvas,
      antialias: true,
      alpha: true
    });
    renderer.setSize(canvas.clientWidth, canvas.clientHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));

    // ---------- lighting ----------
    const ambient = new THREE.AmbientLight(0x4a4438, 0.9);
    scene.add(ambient);

    const key = new THREE.DirectionalLight(0xfff8ec, 2.0);
    key.position.set(5, 8, 6);
    scene.add(key);

    const fill = new THREE.DirectionalLight(0xeaf0ff, 0.5);
    fill.position.set(-4, 2, 3);
    scene.add(fill);

    // cursor-driven point light — gives the glossy surface a moving highlight
    const cursorLight = new THREE.PointLight(0xfff4dc, 2.0, 12);
    cursorLight.position.set(0, 0, 2.5);
    scene.add(cursorLight);

    // ---------- glossy "vanilla pudding" material ----------
    const puddingMaterial = new THREE.MeshPhysicalMaterial({
      color: 0xfaf6ec,
      roughness: 0.2,
      metalness: 0.0,
      clearcoat: 1.0,
      clearcoatRoughness: 0.1,
      reflectivity: 0.5
    });

    // ---------- load model ----------
    const loader = new GLTFLoader();
    const modelURL = 'https://jadiehm.github.io/three-test/assets/splat.gltf';

    loader.load(
      modelURL,
      (gltf) => {
        model = gltf.scene;
        model.scale.set(5, 5, 5);

        // center on the model's actual visual bounds, not just its origin —
        // GLTF exports often have a pivot that isn't at the mesh's center
        const box = new THREE.Box3().setFromObject(model);
        const center = box.getCenter(new THREE.Vector3());
        model.position.sub(center);

        model.traverse((child) => {
          if (child.isMesh) {
            child.material = puddingMaterial;
            // cache the rest-state geometry so ripples have a stable base
            // to displace from each frame, rather than compounding drift
            const pos = child.geometry.attributes.position;
            if (pos) {
              child.userData.basePos = Float32Array.from(pos.array);
            }
          }
        });

        scene.add(model);
        loaded = true;
      },
      undefined,
      (error) => {
        console.error('Failed to load splat model:', error);
      }
    );

    // ---------- cursor tracking ----------
    // drives both the moving highlight and the ripple origin
    const mouse = { x: 0, y: 0 };
    const targetMouse = { x: 0, y: 0 };

    function handleMouseMove(event) {
      const rect = canvas.getBoundingClientRect();
      targetMouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
      targetMouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
    }
    window.addEventListener('mousemove', handleMouseMove);

    // ---------- scroll-tied rotation ----------
    // replaces free-orbit drag: rotation is driven by the page, not the visitor's
    // accidental cursor drag, so the brand moment stays controlled
    let scrollRotation = 0;
    function handleScroll() {
      const maxScroll = document.documentElement.scrollHeight - window.innerHeight;
      const progress = maxScroll > 0 ? window.scrollY / maxScroll : 0;
      scrollRotation = progress * Math.PI * 0.6;
    }
    window.addEventListener('scroll', handleScroll, { passive: true });

    // ---------- ripple deformation ----------
    // perturbs each mesh's vertices outward from the mouse-projected point,
    // same decay shape as a real ripple: sine wave damped by distance and time
    const rippleOrigin = new THREE.Vector3();
    const raycaster = new THREE.Raycaster();
    const rippleStrength = { current: 0 };

    function updateRippleOrigin() {
      raycaster.setFromCamera(mouse, camera);
      // project onto the z=0 plane the model sits on
      const t = -raycaster.ray.origin.z / raycaster.ray.direction.z;
      rippleOrigin.copy(raycaster.ray.origin).addScaledVector(raycaster.ray.direction, t);
    }

    function applyRipple(time) {
      if (!model) return;
      updateRippleOrigin();

      model.traverse((child) => {
        if (!child.isMesh || !child.userData.basePos) return;
        const pos = child.geometry.attributes.position;
        const base = child.userData.basePos;

        for (let i = 0; i < pos.array.length; i += 3) {
          const bx = base[i];
          const by = base[i + 1];
          const bz = base[i + 2];

          // distance from this vertex (in model local space, roughly) to the ripple origin
          const dx = bx - rippleOrigin.x / 5;
          const dy = by - rippleOrigin.y / 5;
          const dist = Math.sqrt(dx * dx + dy * dy);

          const wave = Math.sin(dist * 8 - time * 2) * Math.exp(-dist * 5) * 0.12;
          const ambientWobble = Math.sin(bx * 2 + time * 0.8) * 0.01;

          pos.array[i + 2] = bz + wave + ambientWobble;
        }
        pos.needsUpdate = true;
        child.geometry.computeVertexNormals();
      });
    }

    // ---------- fade in once loaded ----------
    function tickFade() {
      if (loaded && canvas.style.opacity !== '1') {
        canvas.style.opacity = '1';
      }
    }

    // ---------- animation loop ----------
    const clock = new THREE.Clock();

    function animate() {
      requestAnimationFrame(animate);
      const elapsed = clock.getElapsedTime();

      // smooth cursor easing so the light/ripple glide rather than snap
      mouse.x += (targetMouse.x - mouse.x) * 0.08;
      mouse.y += (targetMouse.y - mouse.y) * 0.08;

      cursorLight.position.set(mouse.x * 3, mouse.y * 3, 2.2);

      if (model) {
        model.rotation.y = scrollRotation + Math.sin(elapsed * 0.15) * 0.05;
        applyRipple(elapsed);
      }

      tickFade();
      renderer.render(scene, camera);
    }
    animate();

    // ---------- resize ----------
    function handleResize() {
      const w = canvas.clientWidth;
      const h = canvas.clientHeight;
      camera.aspect = w / h;
      camera.updateProjectionMatrix();
      renderer.setSize(w, h);
    }
    window.addEventListener('resize', handleResize);

    return () => {
      window.removeEventListener('resize', handleResize);
      window.removeEventListener('mousemove', handleMouseMove);
      window.removeEventListener('scroll', handleScroll);
      renderer.dispose();
    };
  });
</script>

<div class="wrapper" bind:this={wrapper}>
  <canvas bind:this={canvas}></canvas>
</div>

<style>
  /* fills its parent edge-to-edge — the parent element needs
     position: relative (or similar) and a real height, e.g.
     a hero section with min-height: 100vh */
  .wrapper {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    background: #0a0a0a;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  canvas {
    display: block;
    width: 100%;
    height: 100%;
    opacity: 0;
    transition: opacity 0.6s ease;
  }
</style>