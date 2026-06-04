<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  
  import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
  import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
  import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
  import { ShaderPass } from 'three/addons/postprocessing/ShaderPass.js';

  let canvas = $state();

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
      
      surface += vec3(1.0, 0.12, 0.15) * specRed * 0.45;
      surface += vec3(0.12, 0.45, 1.0) * specBlue * 0.45;

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

      vec3 redStudioGlint  = vec3(1.0, 0.15, 0.2) * smoothstep(0.4, 0.85, sphereMapUv.y) * 0.14;
      vec3 blueStudioGlint = vec3(0.15, 0.5, 1.0) * smoothstep(0.6, 0.2, sphereMapUv.x) * 0.14;
      surface += redStudioGlint + blueStudioGlint;

      gl_FragColor = vec4(surface, 1.0);
    }
  `;

  // ── 4. Unified Screen Post-Pass (Maintains Sharp Text on Edges) ──────
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
        
        // Tilt-Shift blur driver radius mask
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

  // ── Helper: High-Res Dynamic Canvas Texture ──────────────────────────
  function createTextTexture(text) {
    const textCanvas = document.createElement('canvas');
    textCanvas.width = 2048; 
    textCanvas.height = 1024;
    const ctx = textCanvas.getContext('2d');

    ctx.clearRect(0, 0, textCanvas.width, textCanvas.height);
    ctx.fillStyle = '#ffffff';
    ctx.font = '900 240px sans-serif';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    
    const words = text.split(' ');
    if (words.length > 2) {
      ctx.fillText(words.slice(0, 2).join(' '), 1024, 400);
      ctx.fillText(words.slice(2).join(' '), 1024, 624);
    } else {
      ctx.fillText(text, 1024, 512);
    }

    const tex = new THREE.CanvasTexture(textCanvas);
    tex.needsUpdate = true;
    return tex;
  }

  onMount(() => {
    if (!canvas) return;

    let width = canvas.clientWidth || 300;
    let height = canvas.clientHeight || 300;

    const textureLoader = new THREE.TextureLoader();
    const flagTexture = textureLoader.load('assets/images/fr.png');
    flagTexture.wrapS = THREE.RepeatWrapping;
    flagTexture.wrapT = THREE.RepeatWrapping;
    flagTexture.repeat.set(0.75, 0.75); 
    flagTexture.offset.set(0.12, 0.12);

    const tau=(1+Math.sqrt(5))/2; const raw=[];
    function evp(a,b,c){for(const[x,y,z]of[[a,b,c],[b,c,a],[c,a,b]])for(const sx of[1,-1])for(const sy of[1,-1])for(const sz of[1,-1])raw.push([sx*x,sy*y,sz*z]);}
    evp(0,1,3*tau);evp(1,2+tau,2*tau);evp(2,1+2*tau,tau);
    const vMap=new Map(),V=[];
    raw.forEach(v=>{const k=v.map(x=>Math.round(x*1000)).join(',');if(!vMap.has(k)){vMap.set(k,V.length);V.push(v);}});
    const Vn=V.map(v=>{const l=Math.sqrt(v[0]**2+v[1]**2+v[2]**2);return new THREE.Vector3(v[0]/l,v[1]/l,v[2]/l);});
    const ds=[];
    for(let i=0;i<V.length;i++)for(let j=i+1;j<V.length;j++){const dx=V[i][0]-V[j][0],dy=V[i][1]-V[j][1],dz=V[i][2]-V[j][2];ds.push(Math.sqrt(dx*dx + dy*dy + dz*dz));}
    ds.sort((a,b)=>a-b);const THR=ds[0]*1.1;
    const adj=Array.from({length:60},()=>[]);
    for(let i=0; i<V.length; i++)for(let j=i+1; j<V.length; j++){const dx=V[i][0]-V[j][0],dy=V[i][1]-V[j][1],dz=V[i][2]-V[j][2];if(Math.sqrt(dx*dx+dy*dy+dz*dz)<THR){adj[i].push(j);adj[j].push(i);}}
    function turnRight(prev,cur){const ns=adj[cur].filter(n=>n!==prev);const iD=Vn[cur].clone().sub(Vn[prev]).normalize();const nm=Vn[cur];let ba=-Infinity,bn=-1;for(const n of ns){const oD=Vn[n].clone().sub(Vn[cur]).normalize();const cr=new THREE.Vector3().crossVectors(iD,oD);const ang=Math.atan2(cr.dot(nm),iD.dot(oD));if(ang>ba){ba=ang;bn=n;}}return bn;}
    const faceSet=new Set(),faces=[];
    for(let i=0;i<V.length;i++)for(const j of adj[i]){let prev=i,cur=j;const fv=[i];for(let s=0;s<7;s++){fv.push(cur);const nx=turnRight(prev,cur);if(nx===i||nx===-1)break;prev=cur;cur=nx;}if(fv.length>=5&&fv.length<=6){const k=[...fv].sort((a,b)=>a-b).join(',');if(!faceSet.has(k)){faceSet.add(k);faces.push([...fv]);}}}
    const pentagons=faces.filter(f=>f.length===5),hexagons=faces.filter(f=>f.length===6);

    function orderCCW(vi){const c=new THREE.Vector3();vi.forEach(i=>c.add(Vn[i]));c.normalize();const r=new THREE.Vector3(0,1,0.01).sub(c.clone().multiplyScalar(c.dot(new THREE.Vector3(0,1,0.01)))).normalize();const u=new THREE.Vector3().crossVectors(c,r).normalize();return[...vi].sort((a,b)=>{const da=Vn[a].clone().sub(c),db=Vn[b].clone().sub(c);return Math.atan2(da.dot(u),da.dot(r))-Math.atan2(db.dot(u),db.dot(r));});}
    function centroid(vi){const c=new THREE.Vector3();vi.forEach(i=>c.add(Vn[i]));return c.normalize();}

    function buildGeo(vi, inset=0.055, sub=14){
      const ord=orderCCW(vi);
      const cn=new THREE.Vector3();ord.forEach(i=>cn.add(Vn[i]));cn.normalize();
      const iv=ord.map(i=>Vn[i].clone().lerp(cn,inset).normalize());
      const N=ord.length,pos=[],nm=[],pw=[],idx=[];
      let vc=0;const vm2=new Map();
      function av(v,p){
        const k=`${Math.round(v.x*1e5)},${Math.round(v.y*1e5)},${Math.round(v.z*1e5)}`;
        if(vm2.has(k))return vm2.get(k);
        const id=vc++;vm2.set(k,id);
        const pt=v.clone().multiplyScalar(2.26);
        pos.push(pt.x,pt.y,pt.z);nm.push(v.x,v.y,v.z);pw.push(p);return id;
      }
      for(let fi=0;fi<N;fi++){
        const A=cn.clone(),B=iv[fi].clone(),C=iv[(fi+1)%N].clone();
        const grid=[];
        for(let i=0; i<=sub; i++){
          grid.push([]);
          for(let j=0; j<=sub-i; j++){
            const ba = i/sub;
            const puffCurve = Math.pow(Math.sin(ba * Math.PI * 0.5), 0.35);
            const pt=A.clone().multiplyScalar(ba).add(B.clone().multiplyScalar(j/sub)).add(C.clone().multiplyScalar((sub-i-j)/sub)).normalize();
            grid[i].push(av(pt, puffCurve));
          }
        }
        const gi=(i,j)=>grid[i][j];
        for(let i=0; i<sub; i++)for(let j=0; j<sub-i; j++){
          idx.push(gi(i,j),gi(i+1,j),gi(i,j+1));
          if(j<sub-i-1)idx.push(gi(i+1,j),gi(i+1,j+1),gi(i,j+1));
        }
      }
      const geo=new THREE.BufferGeometry();
      geo.setAttribute('position',    new THREE.BufferAttribute(new Float32Array(pos),3));
      geo.setAttribute('normal',      new THREE.BufferAttribute(new Float32Array(nm),3));
      geo.setAttribute('aPuffWeight', new THREE.BufferAttribute(new Float32Array(pw),1));
      geo.setIndex(idx);
      return geo;
    }

    const scene=new THREE.Scene();
    scene.background = new THREE.Color(0x010102); 
    
    const camera=new THREE.PerspectiveCamera(50, width/height, 0.1, 100);
    camera.position.set(0, 0, 6);
    
    const renderer=new THREE.WebGLRenderer({canvas, antialias:true});
    renderer.setSize(width, height);
    renderer.setPixelRatio(Math.min(devicePixelRatio, 2));

    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.toneMappingExposure = 0.82; 

    const group=new THREE.Group(); scene.add(group);
    const panels=[], panelMats=[];
    const sharedTime={value:0};
    const sharedTheta={value:0};
    const sharedPhi={value:0};

    // ── FIXED SINGLE-SCENE TEXT SHEET ────────────────────────────────────
    // Mapping the canvas texture directly behind the ball inside the perspective pipeline.
    // Setting depthWrite to false stops it from interfering with glass transparencies.
    const textTexture = createTextTexture("Bosnia and Herzegovina");
    const textMesh = new THREE.Mesh(
      new THREE.PlaneGeometry(15.0, 7.5), 
      new THREE.MeshBasicMaterial({ 
        map: textTexture, 
        transparent: true,
        depthWrite: false 
      })
    );
    textMesh.position.set(0, 0, -3.0); // Anchored safely in back volume space
    scene.add(textMesh);

    // ── LAYER 1: SOLID GLASS CORE SPHERE ──────────────────────────────────
    const coreMat = new THREE.ShaderMaterial({
      vertexShader: coreVert, fragmentShader: coreFrag,
      uniforms: {
        uTime: sharedTime, uTheta: sharedTheta, uPhi: sharedPhi,
        uFlagTex: { value: flagTexture }
      }
    });
    const coreMesh = new THREE.Mesh(new THREE.SphereGeometry(2.175, 64, 64), coreMat);
    group.add(coreMesh);

    // ── LAYER 2: THE BEVELED PANEL JOINTS OVERLAY ────────────────────────
    function addPanel(vi){
      const geo=buildGeo(vi, 0.055, 14);
      const mat=new THREE.ShaderMaterial({
        vertexShader:panelVert, fragmentShader:panelFrag,
        uniforms:{ uTime: sharedTime, uPuff: { value: -0.085 } }, 
        transparent: true, depthWrite: true, blending: THREE.NormalBlending
      });
      panelMats.push(mat);
      const mesh=new THREE.Mesh(geo,mat);
      mesh.userData.normal=centroid(vi);
      group.add(mesh); panels.push(mesh);
    }

    pentagons.forEach(f=>addPanel(orderCCW(f)));
    hexagons.forEach((f,hi)=>{
      const o=orderCCW(f);
      if(hi>=16){addPanel(o);}
      else{const[v0,v1,v2,v3,v4,v5]=o;addPanel([v0,v1,v2,v3]);addPanel([v0,v3,v4,v5]);}
    });

    // ── POST-PROCESSING PIPELINE STACK ───────────────────────────────────
    const composer = new EffectComposer(renderer);
    composer.setSize(width, height);
    composer.addPass(new RenderPass(scene, camera));

    const bloomPass = new UnrealBloomPass(new THREE.Vector2(width, height), 0.55, 1.35, 0.0);
    composer.addPass(bloomPass);

    const photoFilterPass = new ShaderPass(postProcessShader);
    composer.addPass(photoFilterPass);

    // ── Interaction Logistics ──────────────────────────────────────────────
    const ZOOM_NEAR=2.5, ZOOM_FAR=6, ASPEED=0.018;
    let zoomState='idle', animProg=0;
    const sQ=new THREE.Quaternion(), tQ=new THREE.Quaternion();
    let sCZ=ZOOM_FAR, tCZ=ZOOM_FAR;
    const ease=x=>x<0.5?2*x*x:-1+(4-2*x)*x;
    const rc=new THREE.Raycaster(), ndc=new THREE.Vector2();

    function handleClick(cx,cy){
      if(zoomState==='zoomed'||zoomState==='zooming-in'){sCZ=camera.position.z;tCZ=ZOOM_FAR;animProg=0;zoomState='zooming-out';return;}
      if(zoomState!=='idle')return;
      const rect=canvas.getBoundingClientRect();
      ndc.x=(cx-rect.left)/rect.width*2-1;ndc.y=-((cy-rect.top)/rect.height)*2+1;
      rc.setFromCamera(ndc,camera);
      const hits=rc.intersectObjects(panels);if(!hits.length)return;
      sQ.copy(group.quaternion);
      tQ.setFromUnitVectors(hits[0].object.userData.normal,new THREE.Vector3(0,0,1));
      sCZ=camera.position.z;tCZ=ZOOM_NEAR;animProg=0;zoomState='zooming-in';vel={x:0,y:0};
    }

    let drag=false, pm={x:0,y:0}, md={x:0,y:0}, vel={x:0,y:0}, theta=0, phi=0;
    const onMouseDown=e=>{md={x:e.clientX,y:e.clientY};if(zoomState!=='idle')return;drag=true;pm={x:e.clientX,y:e.clientY};vel={x:0,y:0};};
    const onMouseUp=e=>{drag=false;if(Math.hypot(e.clientX-md.x,e.clientY-md.y)<5)handleClick(e.clientX,e.clientY);};
    const onMouseMove=e=>{
      if(zoomState==='idle'){
        const rect=canvas.getBoundingClientRect();
        ndc.x=(e.clientX-rect.left)/rect.width*2-1;ndc.y=-((e.clientY-rect.top)/rect.height)*2+1;
        rc.setFromCamera(ndc,camera);
        canvas.style.cursor=rc.intersectObjects(panels).length?'pointer':drag?'grabbing':'grab';
      } else{canvas.style.cursor=zoomState==='zoomed'?'zoom-out':'default';}
      if(!drag||zoomState!=='idle')return;
      const dx=e.clientX-pm.x, dy=e.clientY-pm.y;
      vel={x:dx,y:dy}; theta+=dx*0.005;
      phi=Math.max(-Math.PI/2, Math.min(Math.PI/2, phi+dy*0.005));
      pm={x:e.clientX,y:e.clientY};
    };
    let lt=null, td=null;
    const onTouchStart=e=>{lt=e.touches[0];td={x:e.touches[0].clientX,y:e.touches[0].clientY};};
    const onTouchEnd=e=>{if(!td)return;const tt=e.changedTouches[0];if(Math.hypot(tt.clientX-td.x,tt.clientY-td.y)<10)handleClick(tt.clientX,tt.clientY);td=null;};
    const onTouchMove=e=>{e.preventDefault();if(zoomState!=='idle')return;const tt=e.touches[0];theta+=(tt.clientX-lt.clientX)*0.006;phi=Math.max(-Math.PI/2,Math.min(Math.PI/2,phi+(tt.clientY-lt.clientY)*0.006));lt=tt;td=null;};
    const onWheel=e=>{camera.position.z=Math.max(3,Math.min(12,camera.position.z+e.deltaY*0.01));};

    canvas.addEventListener('mousedown',onMouseDown);
    window.addEventListener('mouseup',onMouseUp);
    window.addEventListener('mousemove',onMouseMove);
    canvas.addEventListener('touchstart',onTouchStart);
    canvas.addEventListener('touchend',onTouchEnd);
    canvas.addEventListener('touchmove',onTouchMove,{passive:false});
    canvas.addEventListener('wheel',onWheel,{passive:true});

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
    const animate=()=>{
      rafId=requestAnimationFrame(animate);
      sharedTime.value+=0.004;
      
      photoFilterPass.uniforms.uNoiseSeed.value = Math.random() * 100.0;

      if(zoomState==='zooming-in'||zoomState==='zooming-out'){
        animProg=Math.min(1,animProg+ASPEED);
        const et=ease(animProg);
        if(zoomState==='zooming-in')group.quaternion.slerpQuaternions(sQ,tQ,et);
        camera.position.z=sCZ+(tCZ-sCZ)*et;
        if(animProg>=1){if(zoomState==='zooming-in'){zoomState='zoomed';}else{const eu=new THREE.Euler().setFromQuaternion(group.quaternion,'XYZ');theta=eu.y;phi=Math.max(-Math.PI/2,Math.min(Math.PI/2,eu.x));zoomState='idle';}}
      } else if(zoomState==='idle'){
        if(!drag){vel.x*=0.92;theta+=vel.x*0.003;theta+=0.004;}
        group.rotation.y=theta;group.rotation.x=phi;
      }
      
      sharedTheta.value = theta;
      sharedPhi.value = phi;

      composer.render();
    };
    animate();

    return()=>{
      cancelAnimationFrame(rafId);ro.disconnect();
      canvas.removeEventListener('mousedown',onMouseDown);window.removeEventListener('mouseup',onMouseUp);window.addEventListener('mousemove',onMouseMove);
      canvas.removeEventListener('touchstart',onTouchStart);canvas.removeEventListener('touchend',onTouchEnd);canvas.removeEventListener('touchmove',onTouchMove);canvas.removeEventListener('wheel',onWheel);
      panelMats.forEach(m=>m.dispose());textTexture.dispose();coreMat.dispose();flagTexture.dispose();composer.dispose();renderer.dispose();
    };
  });
</script>

<div class="wrapper">
  <div class="canvas-container">
    <canvas bind:this={canvas}></canvas>
  </div>
  <p class="hint">Click a panel to zoom in · Click again to zoom out · Drag to rotate · Scroll to zoom</p>
</div>

<style>
  .wrapper { 
    position: relative; 
    width: 100%; 
    height: 100vh; 
    background: #010102; 
    overflow: hidden; 
  }
  
  .canvas-container {
    position: absolute;
    top: 0; left: 0; bottom: 0; right: 0;
    z-index: 1;
  }
  
  canvas { 
    display: block; 
    width: 100%; 
    height: 100%; 
  }
  
  .hint {
    position: absolute; bottom: 16px; left: 50%; transform: translateX(-50%);
    color: rgba(255,255,255,0.18); font-size: 12px; pointer-events: none;
    white-space: nowrap; margin: 0; line-height: 1; z-index: 3;
  }
</style>