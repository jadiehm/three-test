<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

  let canvas;
  let model; // Make model accessible in the component scope

  onMount(() => {
    if (!canvas) return;

    // 1. Scene Setup
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x111111); // Darker background for better contrast

    // 2. Camera
    const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
    camera.position.set(0, 1, 4); // Adjusted camera position slightly

    // 3. Renderer
    const renderer = new THREE.WebGLRenderer({
      canvas: canvas,
      antialias: true,
    });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(window.devicePixelRatio);

    // 4. Lighting
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.2); // Dimmed the ambient light
    scene.add(ambientLight);

    const directionalLight = new THREE.DirectionalLight(0xffffff, 0.5);
    directionalLight.position.set(5, 10, 7.5);
    scene.add(directionalLight);
    
    // New Interactive Point Light
    const pointLight = new THREE.PointLight(0xffffff, 1.5, 100);
    scene.add(pointLight);

    // 5. Orbit Controls (for interactivity)
    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
    controls.screenSpacePanning = false;
    controls.minDistance = 2;
    controls.maxDistance = 10;
    controls.autoRotate = false; // We will handle rotation manually

    // 6. GLTF Loader
    const loader = new GLTFLoader();
    // Path to your local model
    const modelURL = '/assets/splat.gltf';

    loader.load(
      modelURL,
      (gltf) => {
        model = gltf.scene; // Assign to the component-scoped variable
        model.scale.set(5, 5, 5);
        model.position.set(0, 0, 0);
        scene.add(model);
        console.log('GLTF model loaded successfully!');
      },
      (xhr) => {
        console.log((xhr.loaded / xhr.total * 100) + '% loaded');
      },
      (error) => {
        console.error('An error happened while loading the GLTF model:', error);
      }
    );

    // Mouse tracking for light movement
    const mouse = new THREE.Vector2();
    const handleMouseMove = (event) => {
      // Normalize mouse position from -1 to 1 for both axes
      mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
      mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
    };
    window.addEventListener('mousemove', handleMouseMove);

    // 7. Animation Loop
    const animate = () => {
      requestAnimationFrame(animate);
      
      // Auto-rotate the model if it exists
      if (model) {
        model.rotation.y += 0.01;
      }
      
      // Update the point light position based on mouse
      // We can translate the 2D mouse position to a 3D point in the scene
      pointLight.position.set(mouse.x * 5, mouse.y * 5, 2);

      controls.update();
      renderer.render(scene, camera);
    };

    animate();

    // 8. Handle Window Resizing
    const handleResize = () => {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
      renderer.setPixelRatio(window.devicePixelRatio);
    };

    window.addEventListener('resize', handleResize);

    // Svelte's onMount cleanup function
    return () => {
      window.removeEventListener('resize', handleResize);
      window.removeEventListener('mousemove', handleMouseMove); // Clean up mouse listener
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
    overflow: hidden; /* Prevents scrollbars */
  }

  canvas {
    display: block;
    width: 100%;
    height: 100%;
  }
</style>

