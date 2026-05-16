<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';

  let canvas = $state();

  onMount(() => {
    if (!canvas) return;

    // ── Geometry ────────────────────────────────────────────────────────────
    const t = (1 + Math.sqrt(5)) / 2;
    const rawVerts = [];

    function addEvenPerms(a, b, c) {
      for (const [x, y, z] of [[a, b, c], [b, c, a], [c, a, b]])
        for (const sx of [1, -1]) for (const sy of [1, -1]) for (const sz of [1, -1])
          rawVerts.push([sx * x, sy * y, sz * z]);
    }
    addEvenPerms(0, 1, 3 * t);
    addEvenPerms(1, 2 + t, 2 * t);
    addEvenPerms(2, 1 + 2 * t, t);

    const vMap = new Map(), V = [];
    rawVerts.forEach(v => {
      const key = v.map(x => Math.round(x * 1000)).join(',');
      if (!vMap.has(key)) { vMap.set(key, V.length); V.push(v); }
    });

    const Vn = V.map(v => {
      const len = Math.sqrt(v[0] ** 2 + v[1] ** 2 + v[2] ** 2);
      return new THREE.Vector3(v[0] / len, v[1] / len, v[2] / len);
    });

    const allDists = [];
    for (let i = 0; i < V.length; i++) for (let j = i + 1; j < V.length; j++) {
      const dx = V[i][0] - V[j][0], dy = V[i][1] - V[j][1], dz = V[i][2] - V[j][2];
      allDists.push(Math.sqrt(dx * dx + dy * dy + dz * dz));
    }
    allDists.sort((a, b) => a - b);
    const THRESH = allDists[0] * 1.1;

    const adj = Array.from({ length: 60 }, () => []);
    for (let i = 0; i < V.length; i++) for (let j = i + 1; j < V.length; j++) {
      const dx = V[i][0] - V[j][0], dy = V[i][1] - V[j][1], dz = V[i][2] - V[j][2];
      if (Math.sqrt(dx * dx + dy * dy + dz * dz) < THRESH) { adj[i].push(j); adj[j].push(i); }
    }

    function turnRight(prev, cur) {
      const ns = adj[cur].filter(n => n !== prev);
      const inDir = Vn[cur].clone().sub(Vn[prev]).normalize();
      const normal = Vn[cur];
      let bestAngle = -Infinity, bestN = -1;
      for (const n of ns) {
        const outDir = Vn[n].clone().sub(Vn[cur]).normalize();
        const cross = new THREE.Vector3().crossVectors(inDir, outDir);
        const angle = Math.atan2(cross.dot(normal), inDir.dot(outDir));
        if (angle > bestAngle) { bestAngle = angle; bestN = n; }
      }
      return bestN;
    }

    const faceSet = new Set(), faces = [];
    for (let i = 0; i < V.length; i++) {
      for (const j of adj[i]) {
        let prev = i, cur = j;
        const fv = [i];
        for (let step = 0; step < 7; step++) {
          fv.push(cur);
          const nxt = turnRight(prev, cur);
          if (nxt === i || nxt === -1) break;
          prev = cur; cur = nxt;
        }
        if (fv.length >= 5 && fv.length <= 6) {
          const key = [...fv].sort((a, b) => a - b).join(',');
          if (!faceSet.has(key)) { faceSet.add(key); faces.push([...fv]); }
        }
      }
    }

    const pentagons = faces.filter(f => f.length === 5);
    const hexagons  = faces.filter(f => f.length === 6);

    function orderFaceCCW(vIdxs) {
      const cen = new THREE.Vector3();
      vIdxs.forEach(i => cen.add(Vn[i])); cen.normalize();
      const ref = new THREE.Vector3(0, 1, 0.01).sub(cen.clone().multiplyScalar(cen.dot(new THREE.Vector3(0, 1, 0.01)))).normalize();
      const up  = new THREE.Vector3().crossVectors(cen, ref).normalize();
      return [...vIdxs].sort((a, b) => {
        const da = Vn[a].clone().sub(cen), db = Vn[b].clone().sub(cen);
        return Math.atan2(da.dot(up), da.dot(ref)) - Math.atan2(db.dot(up), db.dot(ref));
      });
    }

    function faceCentroid(vIdxs) {
      const c = new THREE.Vector3(); vIdxs.forEach(i => c.add(Vn[i])); return c.normalize();
    }

    function buildInsetGeo(vIdxs, inset = 0.018, subdivs = 8) {
      const ord = orderFaceCCW(vIdxs);
      const cen = new THREE.Vector3(); ord.forEach(i => cen.add(Vn[i])); cen.normalize();
      const insetVerts = ord.map(i => Vn[i].clone().lerp(cen, inset).normalize());
      const N = ord.length;
      const pos = [], nm = [], idx = [];
      const sphPos = [];
      let vtxCount = 0;
      const vtxMap = new Map();

      // Local tangent frame for UV projection
      const tang = Math.abs(cen.y) < 0.9
        ? new THREE.Vector3().crossVectors(new THREE.Vector3(0, 1, 0), cen).normalize()
        : new THREE.Vector3().crossVectors(new THREE.Vector3(1, 0, 0), cen).normalize();
      const bita = new THREE.Vector3().crossVectors(cen, tang).normalize();

      function addVtx(v) {
        const key = `${Math.round(v.x * 1e5)},${Math.round(v.y * 1e5)},${Math.round(v.z * 1e5)}`;
        if (vtxMap.has(key)) return vtxMap.get(key);
        const id = vtxCount++;
        vtxMap.set(key, id);
        const p = v.clone().multiplyScalar(2.2);
        pos.push(p.x, p.y, p.z); nm.push(v.x, v.y, v.z);
        sphPos.push(v.clone());
        return id;
      }

      for (let fi = 0; fi < N; fi++) {
        const A = cen.clone();
        const B = insetVerts[fi].clone();
        const C = insetVerts[(fi + 1) % N].clone();
        const grid = [];
        for (let i = 0; i <= subdivs; i++) {
          grid.push([]);
          for (let j = 0; j <= subdivs - i; j++) {
            const k = subdivs - i - j;
            const pt = A.clone().multiplyScalar(i / subdivs)
              .add(B.clone().multiplyScalar(j / subdivs))
              .add(C.clone().multiplyScalar(k / subdivs))
              .normalize();
            grid[i].push(addVtx(pt));
          }
        }
        const gIdx = (i, j) => grid[i][j];
        for (let i = 0; i < subdivs; i++) {
          for (let j = 0; j < subdivs - i; j++) {
            idx.push(gIdx(i, j), gIdx(i + 1, j), gIdx(i, j + 1));
            if (j < subdivs - i - 1) idx.push(gIdx(i + 1, j), gIdx(i + 1, j + 1), gIdx(i, j + 1));
          }
        }
      }

      // Project sphere positions onto face tangent plane, normalize to [0,1] UV space
      const u2d = sphPos.map(v => v.dot(tang));
      const v2d = sphPos.map(v => v.dot(bita));
      let uMin = Infinity, uMax = -Infinity, vMin = Infinity, vMax = -Infinity;
      for (let i = 0; i < vtxCount; i++) {
        if (u2d[i] < uMin) uMin = u2d[i]; if (u2d[i] > uMax) uMax = u2d[i];
        if (v2d[i] < vMin) vMin = v2d[i]; if (v2d[i] > vMax) vMax = v2d[i];
      }
      const span = Math.max(uMax - uMin, vMax - vMin) || 1;
      const uCen = (uMin + uMax) / 2, vCen = (vMin + vMax) / 2;
      // < 1 zooms into the texture so the circular flag fills more of the polygon
      const UV_SCALE = 0.72;
      const uvArr = new Float32Array(vtxCount * 2);
      for (let i = 0; i < vtxCount; i++) {
        uvArr[i * 2]     = (u2d[i] - uCen) / span * UV_SCALE + 0.5;
        uvArr[i * 2 + 1] = (v2d[i] - vCen) / span * UV_SCALE + 0.5;
      }

      const geo = new THREE.BufferGeometry();
      geo.setAttribute('position', new THREE.BufferAttribute(new Float32Array(pos), 3));
      geo.setAttribute('normal',   new THREE.BufferAttribute(new Float32Array(nm), 3));
      geo.setAttribute('uv',       new THREE.BufferAttribute(uvArr, 2));
      geo.setIndex(idx);
      return geo;
    }

    // ── Scene ───────────────────────────────────────────────────────────────
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x0a0a0a);

    const camera = new THREE.PerspectiveCamera(55, canvas.clientWidth / canvas.clientHeight, 0.1, 100);
    camera.position.set(0, 0, 6);

    const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setSize(canvas.clientWidth, canvas.clientHeight);
    renderer.setPixelRatio(Math.min(devicePixelRatio, 2));

    scene.add(new THREE.AmbientLight(0xffffff, 0.3));
    const kl = new THREE.DirectionalLight(0xffffff, 1.1); kl.position.set(6, 8, 6); scene.add(kl);
    const fl = new THREE.DirectionalLight(0x7799ff, 0.3); fl.position.set(-5, -2, 4); scene.add(fl);
    const rl = new THREE.DirectionalLight(0xffffff, 0.35); rl.position.set(-3, 4, -6); scene.add(rl);
    const mp = new THREE.PointLight(0xfff4ee, 2.0, 30); scene.add(mp);

    const group = new THREE.Group(); scene.add(group);
    const panels = [];

    // 48 images mapped to 48 faces (alphabetical order)
    const imageNames = ['ar','at','au','ba','be','br','ca','cd','ch','ci','co','cr','cv','cw','cz','de','dz','ec','eg','es','fr','gb','gh','ht','iq','ir','jo','jp','kr','ma','mx','ne','no','nz','pa','pt','py','qa','sa','scot','se','sn','tn','tr','us','uy','uz','za'];
    const texLoader = new THREE.TextureLoader();
    const textures = imageNames.map(n => {
      const t = texLoader.load(`/assets/images/${n}.png`);
      t.colorSpace = THREE.SRGBColorSpace;
      t.wrapS = THREE.ClampToEdgeWrapping;
      t.wrapT = THREE.ClampToEdgeWrapping;
      return t;
    });
    let texIdx = 0;

    function addPanel(vIdxs) {
      const geo = buildInsetGeo(vIdxs, 0.018, 8);
      const mat = new THREE.MeshStandardMaterial({
        color: 0xffffff,
        roughness: 0.3,
        metalness: 0.05,
        map: textures[texIdx++],
      });
      const mesh = new THREE.Mesh(geo, mat);
      mesh.userData.normal = faceCentroid(vIdxs);
      group.add(mesh); panels.push(mesh);
    }

    // 12 pentagons → textures[0..11]
    pentagons.forEach(f => addPanel(orderFaceCCW(f)));

    // 16 split hexagons → textures[12..43], 4 whole hexagons → textures[44..47]
    hexagons.forEach((f, hi) => {
      const ord = orderFaceCCW(f);
      if (hi >= 16) {
        addPanel(ord);
      } else {
        const [v0, v1, v2, v3, v4, v5] = ord;
        addPanel([v0, v1, v2, v3]);
        addPanel([v0, v3, v4, v5]);
      }
    });

    // Dark base sphere (shows through gaps)
    group.add(new THREE.Mesh(
      new THREE.SphereGeometry(2.19, 64, 64),
      new THREE.MeshStandardMaterial({ color: 0x050505, roughness: 0.8, metalness: 0.1 })
    ));

    // Gloss overlay
    group.add(new THREE.Mesh(
      new THREE.SphereGeometry(2.21, 64, 64),
      new THREE.MeshStandardMaterial({
        color: 0xffffff, transparent: true, opacity: 0.04,
        roughness: 0.0, metalness: 0.2, side: THREE.FrontSide, depthWrite: false,
      })
    ));

    // ── Interaction ─────────────────────────────────────────────────────────
    const ZOOM_NEAR = 2.5, ZOOM_FAR = 6, ASPEED = 0.018;
    let zoomState = 'idle', animProg = 0;
    const sQ = new THREE.Quaternion(), tQ = new THREE.Quaternion();
    let sCZ = ZOOM_FAR, tCZ = ZOOM_FAR;
    const ease = t => t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t;
    const rc = new THREE.Raycaster(), ndc = new THREE.Vector2();

    function handleClick(cx, cy) {
      if (zoomState === 'zoomed' || zoomState === 'zooming-in') {
        sCZ = camera.position.z; tCZ = ZOOM_FAR; animProg = 0; zoomState = 'zooming-out'; return;
      }
      if (zoomState !== 'idle') return;
      const rect = canvas.getBoundingClientRect();
      ndc.x = (cx - rect.left) / rect.width * 2 - 1;
      ndc.y = -((cy - rect.top) / rect.height) * 2 + 1;
      rc.setFromCamera(ndc, camera);
      const hits = rc.intersectObjects(panels);
      if (!hits.length) return;
      sQ.copy(group.quaternion);
      tQ.setFromUnitVectors(hits[0].object.userData.normal, new THREE.Vector3(0, 0, 1));
      sCZ = camera.position.z; tCZ = ZOOM_NEAR; animProg = 0; zoomState = 'zooming-in'; vel = { x: 0, y: 0 };
    }

    let drag = false, pm = { x: 0, y: 0 }, md = { x: 0, y: 0 }, vel = { x: 0, y: 0 }, theta = 0, phi = 0;

    const onMouseDown = e => {
      md = { x: e.clientX, y: e.clientY };
      if (zoomState !== 'idle') return;
      drag = true; pm = { x: e.clientX, y: e.clientY }; vel = { x: 0, y: 0 };
    };
    const onMouseUp = e => {
      drag = false;
      if (Math.hypot(e.clientX - md.x, e.clientY - md.y) < 5) handleClick(e.clientX, e.clientY);
    };
    const onMouseMove = e => {
      const rect = canvas.getBoundingClientRect();
      mp.position.set(
        ((e.clientX - rect.left) / rect.width * 2 - 1) * 6,
        -((e.clientY - rect.top) / rect.height * 2 - 1) * 6, 5
      );
      if (zoomState === 'idle') {
        ndc.x = (e.clientX - rect.left) / rect.width * 2 - 1;
        ndc.y = -((e.clientY - rect.top) / rect.height) * 2 + 1;
        rc.setFromCamera(ndc, camera);
        canvas.style.cursor = rc.intersectObjects(panels).length ? 'pointer' : drag ? 'grabbing' : 'grab';
      } else if (zoomState === 'zoomed') {
        canvas.style.cursor = 'zoom-out';
      } else {
        canvas.style.cursor = 'default';
      }
      if (!drag || zoomState !== 'idle') return;
      const dx = e.clientX - pm.x, dy = e.clientY - pm.y;
      vel = { x: dx, y: dy };
      theta += dx * 0.005;
      phi = Math.max(-Math.PI / 2, Math.min(Math.PI / 2, phi + dy * 0.005));
      pm = { x: e.clientX, y: e.clientY };
    };

    let lt = null, td = null;
    const onTouchStart = e => { lt = e.touches[0]; td = { x: e.touches[0].clientX, y: e.touches[0].clientY }; };
    const onTouchEnd = e => {
      if (!td) return;
      const tt = e.changedTouches[0];
      if (Math.hypot(tt.clientX - td.x, tt.clientY - td.y) < 10) handleClick(tt.clientX, tt.clientY);
      td = null;
    };
    const onTouchMove = e => {
      e.preventDefault();
      if (zoomState !== 'idle') return;
      const tt = e.touches[0];
      theta += (tt.clientX - lt.clientX) * 0.006;
      phi = Math.max(-Math.PI / 2, Math.min(Math.PI / 2, phi + (tt.clientY - lt.clientY) * 0.006));
      lt = tt; td = null;
    };
    const onWheel = e => {
      camera.position.z = Math.max(3, Math.min(12, camera.position.z + e.deltaY * 0.01));
    };

    canvas.addEventListener('mousedown', onMouseDown);
    window.addEventListener('mouseup', onMouseUp);
    window.addEventListener('mousemove', onMouseMove);
    canvas.addEventListener('touchstart', onTouchStart);
    canvas.addEventListener('touchend', onTouchEnd);
    canvas.addEventListener('touchmove', onTouchMove, { passive: false });
    canvas.addEventListener('wheel', onWheel, { passive: true });

    // ── Resize ──────────────────────────────────────────────────────────────
    const ro = new ResizeObserver(() => {
      const w = canvas.clientWidth, h = canvas.clientHeight;
      camera.aspect = w / h;
      camera.updateProjectionMatrix();
      renderer.setSize(w, h);
    });
    ro.observe(canvas);

    // ── Animation loop ───────────────────────────────────────────────────────
    let rafId;
    const animate = () => {
      rafId = requestAnimationFrame(animate);
      if (zoomState === 'zooming-in' || zoomState === 'zooming-out') {
        animProg = Math.min(1, animProg + ASPEED);
        const et = ease(animProg);
        if (zoomState === 'zooming-in') group.quaternion.slerpQuaternions(sQ, tQ, et);
        camera.position.z = sCZ + (tCZ - sCZ) * et;
        if (animProg >= 1) {
          if (zoomState === 'zooming-in') {
            zoomState = 'zoomed';
          } else {
            const eu = new THREE.Euler().setFromQuaternion(group.quaternion, 'XYZ');
            theta = eu.y; phi = Math.max(-Math.PI / 2, Math.min(Math.PI / 2, eu.x));
            zoomState = 'idle';
          }
        }
      } else if (zoomState === 'idle') {
        if (!drag) { vel.x *= 0.92; theta += vel.x * 0.003; theta += 0.004; }
        group.rotation.y = theta;
        group.rotation.x = phi;
      }
      renderer.render(scene, camera);
    };
    animate();

    // ── Cleanup ──────────────────────────────────────────────────────────────
    return () => {
      cancelAnimationFrame(rafId);
      ro.disconnect();
      canvas.removeEventListener('mousedown', onMouseDown);
      window.removeEventListener('mouseup', onMouseUp);
      window.removeEventListener('mousemove', onMouseMove);
      canvas.removeEventListener('touchstart', onTouchStart);
      canvas.removeEventListener('touchend', onTouchEnd);
      canvas.removeEventListener('touchmove', onTouchMove);
      canvas.removeEventListener('wheel', onWheel);
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
    color: rgba(255, 255, 255, 0.25);
    font-size: 12px;
    line-height: 1;
    pointer-events: none;
    white-space: nowrap;
    margin: 0;
  }
</style>