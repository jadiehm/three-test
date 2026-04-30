<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';

  let canvas = $state();

  onMount(() => {
    if (!canvas) return;

    // --- Scene ---
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x111111);

    // --- Camera ---
    const camera = new THREE.PerspectiveCamera(60, canvas.clientWidth / canvas.clientHeight, 0.1, 100);
    camera.position.set(0, 0, 6);

    // --- Renderer ---
    const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setSize(canvas.clientWidth, canvas.clientHeight);
    renderer.setPixelRatio(Math.min(devicePixelRatio, 2));

    // --- Lighting ---
    scene.add(new THREE.AmbientLight(0xffffff, 0.4));

    const key = new THREE.DirectionalLight(0xffffff, 0.9);
    key.position.set(6, 8, 6);
    scene.add(key);

    const fill = new THREE.DirectionalLight(0x8899ff, 0.25);
    fill.position.set(-5, -2, 4);
    scene.add(fill);

    const rim = new THREE.DirectionalLight(0xffffff, 0.3);
    rim.position.set(-3, 4, -6);
    scene.add(rim);

    const mousePt = new THREE.PointLight(0xfff8ee, 1.8, 30);
    scene.add(mousePt);

    // --- Ball ---
    const R = 2.2;
    const SEAM_R = R + 0.012;
    const SUB = 6;
    const group = new THREE.Group();
    scene.add(group);

    const octFaces = [
      [[ 1,0,0],[0,1,0],[0,0,1]],
      [[ 1,0,0],[0,0,1],[0,-1,0]],
      [[ 1,0,0],[0,-1,0],[0,0,-1]],
      [[ 1,0,0],[0,0,-1],[0,1,0]],
      [[-1,0,0],[0,0,1],[0,1,0]],
      [[-1,0,0],[0,-1,0],[0,0,1]],
      [[-1,0,0],[0,0,-1],[0,-1,0]],
      [[-1,0,0],[0,1,0],[0,0,-1]],
    ].map(f => f.map(v => new THREE.Vector3(...v)));

    function sphPoint(a, b, c, u, v, w) {
      return a.clone().multiplyScalar(u)
        .add(b.clone().multiplyScalar(v))
        .add(c.clone().multiplyScalar(w))
        .normalize();
    }

    function buildPanelGeo(A, B, C) {
      const positions = [];
      const normals = [];
      const indices = [];

      const verts = [];
      for (let i = 0; i <= SUB; i++) {
        for (let j = 0; j <= SUB - i; j++) {
          const k = SUB - i - j;
          const pt = sphPoint(A, B, C, i / SUB, j / SUB, k / SUB).multiplyScalar(R);
          const nm = sphPoint(A, B, C, i / SUB, j / SUB, k / SUB);
          verts.push({ pt, nm });
        }
      }

      verts.forEach(({ pt, nm }) => {
        positions.push(pt.x, pt.y, pt.z);
        normals.push(nm.x, nm.y, nm.z);
      });

      function idx(i, j) {
        let start = 0;
        for (let m = 0; m < i; m++) start += (SUB - m + 1);
        return start + j;
      }

      for (let i = 0; i < SUB; i++) {
        for (let j = 0; j < SUB - i; j++) {
          const a = idx(i, j);
          const b = idx(i + 1, j);
          const c = idx(i, j + 1);
          indices.push(a, b, c);
          if (j < SUB - i - 1) {
            indices.push(b, idx(i + 1, j + 1), c);
          }
        }
      }

      const geo = new THREE.BufferGeometry();
      geo.setAttribute('position', new THREE.BufferAttribute(new Float32Array(positions), 3));
      geo.setAttribute('normal', new THREE.BufferAttribute(new Float32Array(normals), 3));
      geo.setIndex(indices);
      return geo;
    }

    function buildSeamLine(a, b, steps = 16) {
      const pts = [];
      for (let i = 0; i <= steps; i++) {
        pts.push(a.clone().lerp(b, i / steps).normalize().multiplyScalar(SEAM_R));
      }
      return new THREE.BufferGeometry().setFromPoints(pts);
    }

    const panels = [];
    let panelIdx = 0;

    octFaces.forEach(([A, B, C]) => {
      const MAB = A.clone().add(B).normalize();
      const MBC = B.clone().add(C).normalize();
      const MCA = C.clone().add(A).normalize();
      const CEN = A.clone().add(B).add(C).normalize();

      const subs = [
        [A, MAB, CEN], [MAB, B, CEN],
        [B, MBC, CEN], [MBC, C, CEN],
        [C, MCA, CEN], [MCA, A, CEN],
      ];

      subs.forEach(([p, q, r]) => {
        const dark = panelIdx % 2 === 0;

        const mat = new THREE.MeshStandardMaterial({
          color: dark ? 0x111111 : 0xeeeeee,
          roughness: dark ? 0.55 : 0.35,
          metalness: dark ? 0.15 : 0.05,
        });
        const mesh = new THREE.Mesh(buildPanelGeo(p, q, r), mat);
        // Store the panel's outward normal (centroid of its spherical triangle)
        // so raycasting can find which way to rotate the group to center this panel.
        mesh.userData.normal = p.clone().add(q).add(r).normalize();
        group.add(mesh);
        panels.push(mesh);

        const seamMat = new THREE.LineBasicMaterial({
          color: dark ? 0x555555 : 0x999999,
          transparent: true,
          opacity: 0.5,
        });
        [[p, q], [q, r], [r, p]].forEach(([s, e]) => {
          group.add(new THREE.Line(buildSeamLine(s, e), seamMat));
        });

        panelIdx++;
      });
    });

    // Glass skin to smooth the silhouette
    group.add(new THREE.Mesh(
      new THREE.SphereGeometry(R + 0.01, 64, 64),
      new THREE.MeshStandardMaterial({
        color: 0xffffff,
        transparent: true,
        opacity: 0.035,
        roughness: 0.0,
        metalness: 0.1,
        side: THREE.FrontSide,
        depthWrite: false,
      })
    ));

    // --- Zoom state machine ---
    // 'idle'        — auto-rotating, draggable
    // 'zooming-in'  — animating toward selected panel
    // 'zoomed'      — panel centered, camera close, no rotation
    // 'zooming-out' — animating camera back out
    const ZOOM_NEAR = 2.5;
    const ZOOM_FAR  = 6;
    const ANIM_SPEED = 0.01;

    let zoomState = 'idle';
    let animProgress = 0;
    const startQ  = new THREE.Quaternion();
    const targetQ = new THREE.Quaternion();
    let startCamZ  = ZOOM_FAR;
    let targetCamZ = ZOOM_FAR;

    // Cubic ease-in-out
    const ease = t => t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t;

    const raycaster = new THREE.Raycaster();
    const ndcClick  = new THREE.Vector2();

    function handleClick(clientX, clientY) {
      // While zoomed (or mid-zoom-in), any click zooms back out
      if (zoomState === 'zoomed' || zoomState === 'zooming-in') {
        startCamZ    = camera.position.z;
        targetCamZ   = ZOOM_FAR;
        animProgress = 0;
        zoomState    = 'zooming-out';
        return;
      }
      if (zoomState !== 'idle') return;

      const rect = canvas.getBoundingClientRect();
      ndcClick.x = (clientX - rect.left) / rect.width * 2 - 1;
      ndcClick.y = -((clientY - rect.top) / rect.height) * 2 + 1;

      raycaster.setFromCamera(ndcClick, camera);
      const hits = raycaster.intersectObjects(panels);
      if (hits.length === 0) return;

      // Rotate the group so this panel's outward normal faces +Z (the camera)
      const panelNormal = hits[0].object.userData.normal;
      startQ.copy(group.quaternion);
      targetQ.setFromUnitVectors(panelNormal, new THREE.Vector3(0, 0, 1));
      startCamZ    = camera.position.z;
      targetCamZ   = ZOOM_NEAR;
      animProgress = 0;
      zoomState    = 'zooming-in';
      velocity     = { x: 0, y: 0 };
    }

    // --- Input ---
    let isDragging   = false;
    let prevMouse    = { x: 0, y: 0 };
    let mouseDownPos = { x: 0, y: 0 };
    let velocity     = { x: 0, y: 0 };
    let theta = 0, phi = 0;

    const onMouseDown = (e) => {
      mouseDownPos = { x: e.clientX, y: e.clientY };
      if (zoomState !== 'idle') return;
      isDragging = true;
      prevMouse  = { x: e.clientX, y: e.clientY };
      velocity   = { x: 0, y: 0 };
    };

    const onMouseUp = (e) => {
      isDragging = false;
      const dx = e.clientX - mouseDownPos.x;
      const dy = e.clientY - mouseDownPos.y;
      if (Math.hypot(dx, dy) < 5) handleClick(e.clientX, e.clientY);
    };

    const onMouseMove = (e) => {
      const rect = canvas.getBoundingClientRect();
      mousePt.position.set(
        ((e.clientX - rect.left) / rect.width * 2 - 1) * 6,
        -((e.clientY - rect.top) / rect.height * 2 - 1) * 6,
        5
      );

      // Update cursor: pointer when hovering a panel (idle only)
      if (zoomState === 'idle') {
        ndcClick.x = (e.clientX - rect.left) / rect.width * 2 - 1;
        ndcClick.y = -((e.clientY - rect.top) / rect.height) * 2 + 1;
        raycaster.setFromCamera(ndcClick, camera);
        canvas.style.cursor = raycaster.intersectObjects(panels).length
          ? 'pointer'
          : isDragging ? 'grabbing' : 'grab';
      } else if (zoomState === 'zoomed') {
        canvas.style.cursor = 'zoom-out';
      } else {
        canvas.style.cursor = 'default';
      }

      if (!isDragging || zoomState !== 'idle') return;
      const dx = e.clientX - prevMouse.x;
      const dy = e.clientY - prevMouse.y;
      velocity  = { x: dx, y: dy };
      theta    += dx * 0.005;
      phi       = Math.max(-Math.PI / 2, Math.min(Math.PI / 2, phi + dy * 0.005));
      prevMouse = { x: e.clientX, y: e.clientY };
    };

    let lastTouch   = null;
    let touchDownPos = null;
    const onTouchStart = (e) => {
      lastTouch    = e.touches[0];
      touchDownPos = { x: e.touches[0].clientX, y: e.touches[0].clientY };
    };
    const onTouchEnd = (e) => {
      if (!touchDownPos) return;
      const t = e.changedTouches[0];
      if (Math.hypot(t.clientX - touchDownPos.x, t.clientY - touchDownPos.y) < 10) {
        handleClick(t.clientX, t.clientY);
      }
      touchDownPos = null;
    };
    const onTouchMove = (e) => {
      e.preventDefault();
      if (zoomState !== 'idle') return;
      const t = e.touches[0];
      theta += (t.clientX - lastTouch.clientX) * 0.006;
      phi    = Math.max(-Math.PI / 2, Math.min(Math.PI / 2, phi + (t.clientY - lastTouch.clientY) * 0.006));
      lastTouch    = t;
      touchDownPos = null; // moved → not a tap
    };
    const onWheel = (e) => {
      camera.position.z = Math.max(3, Math.min(12, camera.position.z + e.deltaY * 0.01));
    };

    canvas.addEventListener('mousedown', onMouseDown);
    window.addEventListener('mouseup',   onMouseUp);
    window.addEventListener('mousemove', onMouseMove);
    canvas.addEventListener('touchstart', onTouchStart);
    canvas.addEventListener('touchend',   onTouchEnd);
    canvas.addEventListener('touchmove',  onTouchMove, { passive: false });
    canvas.addEventListener('wheel',      onWheel,     { passive: true });

    // --- Resize ---
    const ro = new ResizeObserver(() => {
      const w = canvas.clientWidth, h = canvas.clientHeight;
      camera.aspect = w / h;
      camera.updateProjectionMatrix();
      renderer.setSize(w, h);
    });
    ro.observe(canvas);

    // --- Animation loop ---
    let rafId;
    const animate = () => {
      rafId = requestAnimationFrame(animate);

      if (zoomState === 'zooming-in' || zoomState === 'zooming-out') {
        animProgress = Math.min(1, animProgress + ANIM_SPEED);
        const t = ease(animProgress);

        if (zoomState === 'zooming-in') {
          group.quaternion.slerpQuaternions(startQ, targetQ, t);
        }
        camera.position.z = startCamZ + (targetCamZ - startCamZ) * t;

        if (animProgress >= 1) {
          if (zoomState === 'zooming-in') {
            zoomState = 'zoomed';
          } else {
            // Re-sync theta/phi from the quaternion so dragging resumes correctly
            const euler = new THREE.Euler().setFromQuaternion(group.quaternion, 'XYZ');
            theta = euler.y;
            phi   = Math.max(-Math.PI / 2, Math.min(Math.PI / 2, euler.x));
            zoomState = 'idle';
          }
        }
      } else if (zoomState === 'idle') {
        if (!isDragging) {
          velocity.x *= 0.92;
          theta      += velocity.x * 0.003;
          theta      += 0.004;
        }
        group.rotation.y = theta;
        group.rotation.x = phi;
      }
      // 'zoomed': group and camera are already at their targets — nothing to update

      renderer.render(scene, camera);
    };
    animate();

    // --- Cleanup ---
    return () => {
      cancelAnimationFrame(rafId);
      ro.disconnect();
      canvas.removeEventListener('mousedown', onMouseDown);
      window.removeEventListener('mouseup',   onMouseUp);
      window.removeEventListener('mousemove', onMouseMove);
      canvas.removeEventListener('touchstart', onTouchStart);
      canvas.removeEventListener('touchend',   onTouchEnd);
      canvas.removeEventListener('touchmove',  onTouchMove);
      canvas.removeEventListener('wheel',      onWheel);
      renderer.dispose();
    };
  });
</script>

<div class="wrapper">
  <canvas bind:this={canvas}></canvas>
  <p class="hint">Click a panel to zoom in · Click again to zoom out · Drag to rotate · Scroll to zoom</p>
</div>

<style>
  .wrapper {
    position: relative;
    width: 100%;
    height: 100%;
  }

  canvas {
    display: block;
    width: 100%;
    height: 100%;
  }

  .hint {
    position: absolute;
    bottom: 16px;
    left: 50%;
    transform: translateX(-50%);
    color: rgba(255, 255, 255, 0.35);
    font-size: 12px;
    line-height: 1;
    pointer-events: none;
    white-space: nowrap;
    margin: 0;
  }
</style>
