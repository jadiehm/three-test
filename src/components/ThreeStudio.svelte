<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
  import * as BufferGeometryUtils from 'three/examples/jsm/utils/BufferGeometryUtils.js';

  let canvas;
  let wrapper;
  let model;
  let singleMesh;
  let loaded = false;

  onMount(() => {
    if (!canvas) return;

    let animId;

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
    const ambient = new THREE.AmbientLight(0xfffdfa, 1.1);
    scene.add(ambient);

    const key = new THREE.DirectionalLight(0xfffdfa, 2.2);
    key.position.set(5, 8, 6);
    scene.add(key);

    const fill = new THREE.DirectionalLight(0xe8f0fe, 0.6);
    fill.position.set(-4, 2, 3);
    scene.add(fill);

    const backLight = new THREE.DirectionalLight(0xfff8ee, 1.2);
    backLight.position.set(0, -3, -5);
    scene.add(backLight);

    const cursorLight = new THREE.PointLight(0xfffefb, 2.5, 12);
    cursorLight.position.set(0, 0, 2.5);
    scene.add(cursorLight);

    // ---------- vanilla pudding material ----------
    const puddingMaterial = new THREE.MeshPhysicalMaterial({
      color: 0xfffefb,
      roughness: 0.25,
      metalness: 0.0,
      
      transmission: 0.18,
      thickness: 1.5,
      attenuationColor: new THREE.Color(0xfff8ee),
      attenuationDistance: 1.8,
      
      clearcoat: 0.85,
      clearcoatRoughness: 0.15,
      ior: 1.38,
      reflectivity: 0.5
    });

    // ---------- load model ----------
    const loader = new GLTFLoader();
    const modelURL = 'assets/studio.gltf';

    loader.load(
      modelURL,
      (gltf) => {
        model = gltf.scene;
        model.scale.set(5, 5, 8.5);

        const geometries = [];
        model.traverse((child) => {
          if (child.isMesh && child.geometry) {
            const geo = child.geometry.clone();
            geo.deleteAttribute('normal');
            geo.deleteAttribute('uv');
            geometries.push(geo);
          }
        });

        if (geometries.length > 0) {
          let combinedGeo = geometries.length === 1 
            ? geometries[0] 
            : BufferGeometryUtils.mergeGeometries(geometries, false);

          combinedGeo = BufferGeometryUtils.mergeVertices(combinedGeo, 0.001);
          combinedGeo.computeVertexNormals();

          singleMesh = new THREE.Mesh(combinedGeo, puddingMaterial);

          while (model.children.length > 0) {
            model.remove(model.children[0]);
          }
          model.add(singleMesh);

          const pos = combinedGeo.attributes.position;
          if (pos) {
            singleMesh.userData.basePos = Float32Array.from(pos.array);
          }
        }

        const box = new THREE.Box3().setFromObject(model);
        const center = box.getCenter(new THREE.Vector3());
        model.position.sub(center);

        scene.add(model);
        loaded = true;
      },
      undefined,
      (error) => {
        console.error('Failed to load model:', error);
      }
    );

    // ---------- mouse & canvas boundary tracking ----------
    const mouse = new THREE.Vector2(-999, -999);
    const raycaster = new THREE.Raycaster();
    const localPlane = new THREE.Plane(new THREE.Vector3(0, 0, 1), -0.2);
    const hitPoint = new THREE.Vector3();
    const invMatrix = new THREE.Matrix4();

    let mouseInCanvas = false;

    function handleMouseMove(event) {
      if (!canvas) return;
      const rect = canvas.getBoundingClientRect();
      mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
      mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
      mouseInCanvas = true;
    }

    function handleMouseLeave() {
      mouseInCanvas = false;
      mouse.set(-999, -999);
    }

    window.addEventListener('mousemove', handleMouseMove);
    canvas.addEventListener('mouseleave', handleMouseLeave);

    // ---------- scroll-tied rotation ----------
    let scrollRotation = 0;
    function handleScroll() {
      const maxScroll = document.documentElement.scrollHeight - window.innerHeight;
      const progress = maxScroll > 0 ? window.scrollY / maxScroll : 0;
      scrollRotation = progress * Math.PI * 0.6;
    }
    window.addEventListener('scroll', handleScroll, { passive: true });

    // ---------- lifecycle state tracking ----------
    let introTime = 0;
    let isIntroFinished = false;

    const localPoke = {
      target: new THREE.Vector3(),
      current: new THREE.Vector3(),
      intensity: 0,
      targetIntensity: 0
    };

    function updateLocalDent(delta) {
      if (!singleMesh || !singleMesh.userData.basePos) return;

      // STAGE 1: INTRO ANIMATION (Sweeps once across the logo on first load)
      if (!isIntroFinished) {
        introTime += delta;
        const introDuration = 2.2; // seconds
        const progress = Math.min(introTime / introDuration, 1.0);

        // Sweep path from left to right with a sine arch
        const introX = THREE.MathUtils.lerp(-2.2, 2.2, progress);
        const introY = Math.sin(progress * Math.PI) * 0.25;

        // Intensity bell curve (0 -> 1 -> 0) so it settles smoothly
        const introIntensity = Math.sin(progress * Math.PI) * 0.9;

        localPoke.target.set(introX, introY, 0.2);
        localPoke.targetIntensity = introIntensity;

        if (progress >= 1.0) {
          isIntroFinished = true;
          localPoke.targetIntensity = 0.0;
        }
      } 
      // STAGE 2 & 3: SETTLED / INTERACTIVE
      else {
        if (mouseInCanvas) {
          raycaster.setFromCamera(mouse, camera);
          invMatrix.copy(singleMesh.matrixWorld).invert();
          const localRay = raycaster.ray.clone().applyMatrix4(invMatrix);

          if (localRay.intersectPlane(localPlane, hitPoint)) {
            localPoke.target.copy(hitPoint);
            const inBounds = Math.abs(hitPoint.x) < 2.5 && Math.abs(hitPoint.y) < 1.0;
            localPoke.targetIntensity = inBounds ? 1.0 : 0.0;
          } else {
            localPoke.targetIntensity = 0.0;
          }
        } else {
          // Off canvas -> settle to 0 intensity (Default View)
          localPoke.targetIntensity = 0.0;
        }
      }

      // Smoothly lerp towards target
      localPoke.current.lerp(localPoke.target, 0.12);
      localPoke.intensity += (localPoke.targetIntensity - localPoke.intensity) * 0.1;

      const pos = singleMesh.geometry.attributes.position;
      const base = singleMesh.userData.basePos;

      // CLEAN RESET: When settled at zero intensity, restore exact base geometry
      if (localPoke.intensity < 0.001) {
        let needsUpdate = false;
        for (let i = 2; i < pos.array.length; i += 3) {
          if (pos.array[i] !== base[i]) {
            pos.array[i] = base[i];
            needsUpdate = true;
          }
        }
        if (needsUpdate) {
          pos.needsUpdate = true;
          singleMesh.geometry.computeVertexNormals();
        }
        return;
      }

      // Active deformation math
      const px = localPoke.current.x;
      const py = localPoke.current.y;

      const dentRadiusSq = 0.22; 
      const dentDepth = 0.32;    
      const bulgeHeight = 0.13;  

      for (let i = 0; i < pos.array.length; i += 3) {
        const bx = base[i];
        const by = base[i + 1];
        const bz = base[i + 2];

        const dx = bx - px;
        const dy = by - py;
        const distSq = dx * dx + dy * dy;

        if (distSq < 1.1) {
          const dist = Math.sqrt(distSq);

          // Gaussian indent + volume ring
          const dent = Math.exp(-distSq / dentRadiusSq) * dentDepth * localPoke.intensity;
          const bulge = (dist / 0.4) * Math.exp(-distSq / 0.35) * bulgeHeight * localPoke.intensity;

          const targetZ = bz - dent + bulge;
          pos.array[i + 2] += (targetZ - pos.array[i + 2]) * 0.18;
        } else {
          pos.array[i + 2] += (bz - pos.array[i + 2]) * 0.18;
        }
      }

      pos.needsUpdate = true;
      singleMesh.geometry.computeVertexNormals();
    }

    // ---------- fade in once loaded ----------
    function tickFade() {
      if (!canvas) return;
      if (loaded && canvas.style.opacity !== '1') {
        canvas.style.opacity = '1';
      }
    }

    // ---------- animation loop ----------
    const clock = new THREE.Clock();
    let lastTime = 0;

    function animate() {
      animId = requestAnimationFrame(animate);
      const elapsed = clock.getElapsedTime();
      const delta = Math.min(elapsed - lastTime, 0.1); // Cap delta to avoid jumps
      lastTime = elapsed;

      // Update cursor light position
      if (mouseInCanvas) {
        cursorLight.position.set(mouse.x * 3, mouse.y * 3, 2.2);
      } else {
        cursorLight.position.set(0, 0, 2.5);
      }

      if (model) {
        model.rotation.y = scrollRotation + Math.sin(elapsed * 0.15) * 0.05;
        if (loaded) {
          updateLocalDent(delta);
        }
      }

      tickFade();
      renderer.render(scene, camera);
    }
    animate();

    // ---------- resize ----------
    function handleResize() {
      if (!canvas) return;
      const w = canvas.clientWidth;
      const h = canvas.clientHeight;
      camera.aspect = w / h;
      camera.updateProjectionMatrix();
      renderer.setSize(w, h);
    }
    window.addEventListener('resize', handleResize);

    // ---------- cleanup ----------
    return () => {
      cancelAnimationFrame(animId);
      window.removeEventListener('resize', handleResize);
      window.removeEventListener('mousemove', handleMouseMove);
      window.removeEventListener('scroll', handleScroll);
      if (canvas) {
        canvas.removeEventListener('mouseleave', handleMouseLeave);
      }
      renderer.dispose();
    };
  });
</script>

<div class="wrapper" bind:this={wrapper}>
  <canvas bind:this={canvas}></canvas>
</div>

<style>
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