<script>
  // Svelte 5 Runes
  let started = $state(false);
  let dotStarted = $state(false);
  let showStudioI = $state(false);

  let drawProgress = $state(0);
  let dotTime = $state(0); 

  // Frame loop tracking tokens to prevent animation fighting
  let drawRafId = null;
  let dotRafId = null;

  // Element Bindings
  let pathEl0 = $state(null);
  let pathEl1 = $state(null);
  let pathEl2 = $state(null);
  let puddingDotEl = $state(null);
  let studioDotEl = $state(null);

  // Individual Path Measurement Metrics
  let len0 = $state(0);
  let len1 = $state(0);
  let len2 = $state(0);
  
  let totalLen = $derived(len0 + len1 + len2);

  let dx = $state(0), dy = $state(0);
  let sx = $state(1), sy = $state(1);
  let cxP = $state(0), cyP = $state(0);
  let halfH = $state(0); 

  // Smooth drawing ease
  function cubicOut(t) {
    return 1 - Math.pow(1 - t, 3);
  }

  // Classical Bounce Easing Curve
  function bounceOut(t) {
    const n1 = 7.5625;
    const d1 = 2.75;
    if (t < 1 / d1) {
      return n1 * t * t;
    } else if (t < 2 / d1) {
      return n1 * (t -= 1.5 / d1) * t + 0.75;
    } else if (t < 2.5 / d1) {
      return n1 * (t -= 2.25 / d1) * t + 0.9375;
    } else {
      return n1 * (t -= 2.625 / d1) * t + 0.984375;
    }
  }

  // Initialization Effect
  $effect(() => {
    if (pathEl0 && len0 === 0) len0 = pathEl0.getTotalLength();
    if (pathEl1 && len1 === 0) len1 = pathEl1.getTotalLength();
    if (pathEl2 && len2 === 0) len2 = pathEl2.getTotalLength();

    if (puddingDotEl && studioDotEl) {
      const boxP = puddingDotEl.getBBox();
      const boxS = studioDotEl.getBBox();

      cxP = boxP.x + boxP.width / 2;
      cyP = boxP.y + boxP.height / 2;
      halfH = boxP.height / 2;

      dx = (boxS.x + boxS.width / 2) - cxP;
      dy = (boxS.y + boxS.height / 2) - cyP;

      sx = boxS.width / boxP.width;
      sy = boxS.height / boxP.height;
    }
  });

  // Trigger drop effect at 85% completion
  $effect(() => {
    if (drawProgress >= 0.25 && !dotStarted) {
      dotStarted = true;
      animateDot();
    }
  });

  // Target spatial animation timeline
  const targetProgress = $derived(dotTime < 0.32 ? 0 : bounceOut((dotTime - 0.32) / 0.78));

  // --- Falling Droplet Scale Curves ---
  const liquidScaleX = $derived.by(() => {
    if (dotTime < 0.32) {
      return 1 - 0.15 * Math.sin((dotTime / 0.32) * Math.PI / 2);
    } else {
      const remT = (dotTime - 0.32) / 0.78;
      const floorImpact = Math.max(0, targetProgress - 0.75) / 0.25; 
      return 1 + 0.45 * floorImpact * (1 - remT);
    }
  });

  const liquidScaleY = $derived.by(() => {
    if (dotTime < 0.32) {
      return 1 + 0.15 * Math.sin((dotTime / 0.32) * Math.PI / 2);
    } else {
      const remT = (dotTime - 0.32) / 0.78;
      const floorImpact = Math.max(0, targetProgress - 0.75) / 0.25;
      const decay = 1 - remT;
      return 1 + 0.12 * (1 - floorImpact) * decay - 0.35 * floorImpact * decay;
    }
  });

  const extraY = $derived.by(() => {
    if (dotTime < 0.32) {
      return halfH * (liquidScaleY - 1);
    } else {
      const flightProgress = Math.min((dotTime - 0.32) / 0.15, 1);
      const snapDownToCenter = (halfH * 0.85) * (1 - flightProgress);
      const remT = (dotTime - 0.32) / 0.78;
      const floorImpact = Math.max(0, targetProgress - 0.75) / 0.25;
      const squashShift = -halfH * (1 - liquidScaleY) * floorImpact * (1 - remT);
      return snapDownToCenter + squashShift;
    }
  });

  const dotTransform = $derived(`
    translate(${cxP} ${cyP}) 
    translate(${dx * targetProgress} ${dy * targetProgress + extraY}) 
    scale(${(1 + (sx - 1) * targetProgress) * liquidScaleX} ${(1 + (sy - 1) * targetProgress) * liquidScaleY}) 
    translate(${-cxP} ${-cyP})
  `);


  // --- Duplicate Base Dot (Fluid surface tension snap-back) ---
  const dupeScaleY = $derived.by(() => {
    if (dotTime < 0.32) {
      return 1 + 0.15 * Math.sin((dotTime / 0.32) * Math.PI / 2);
    } else {
      const snapT = Math.min((dotTime - 0.32) / 0.18, 1);
      const baseSnap = 1 - snapT;
      const wobble = Math.sin(snapT * Math.PI * 3) * 0.10 * baseSnap;
      return 1 + 0.15 * baseSnap + wobble;
    }
  });

  const dupeScaleX = $derived.by(() => {
    if (dotTime < 0.32) {
      return 1 - 0.15 * Math.sin((dotTime / 0.32) * Math.PI / 2);
    } else {
      const snapT = Math.min((dotTime - 0.32) / 0.18, 1);
      const baseSnap = 1 - snapT;
      const wobble = Math.sin(snapT * Math.PI * 3) * 0.04 * baseSnap;
      return 1 - 0.15 * baseSnap - wobble;
    }
  });

  const dupeExtraY = $derived(halfH * (dupeScaleY - 1));

  const dupeTransform = $derived(`
    translate(${cxP} ${cyP})
    translate(0 ${dupeExtraY})
    scale(${dupeScaleX} ${dupeScaleY})
    translate(${-cxP} ${-cyP})
  `);


  // --- Reversed Drawing Timeline Offsets ---
  let currentDist = $derived(drawProgress * totalLen);
  let offset2 = $derived(len2 - Math.max(0, Math.min(len2, currentDist)));
  let offset1 = $derived(len1 - Math.max(0, Math.min(len1, currentDist - len2)));
  let offset0 = $derived(len0 - Math.max(0, Math.min(len0, currentDist - len2 - len1)));

  function animateDraw() {
    if (pathEl0 && len0 === 0) len0 = pathEl0.getTotalLength();
    if (pathEl1 && len1 === 0) len1 = pathEl1.getTotalLength();
    if (pathEl2 && len2 === 0) len2 = pathEl2.getTotalLength();

    const duration = 900; 
    const startTime = performance.now();

    function update(now) {
      const elapsed = now - startTime;
      const t = Math.min(elapsed / duration, 1);
      drawProgress = cubicOut(t);

      if (t < 1) {
        drawRafId = requestAnimationFrame(update);
      }
    }
    drawRafId = requestAnimationFrame(update);
  }

  function animateDot() {
    const duration = 2000; 
    const startTime = performance.now();

    function update(now) {
      const elapsed = now - startTime;
      dotTime = Math.min(elapsed / duration, 1);

      if (dotTime < 1) {
        dotRafId = requestAnimationFrame(update);
      } else {
        showStudioI = true;
      }
    }
    dotRafId = requestAnimationFrame(update);
  }

  // Dual-state interactive handler logic
  function handlePageClick() {
    if (!started) {
      // Step 1: Fire Animation
      started = true;
      animateDraw();
    } else {
      // Step 2: Instantly Kill Animation and Clean State Values
      if (drawRafId) cancelAnimationFrame(drawRafId);
      if (dotRafId) cancelAnimationFrame(dotRafId);
      
      started = false;
      dotStarted = false;
      showStudioI = false;
      drawProgress = 0;
      dotTime = 0;
    }
  }
</script>

<main onclick={handlePageClick} class="interactive-container">
  <svg id="script-logo" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 240 102.8">
    <defs>
      <clipPath id="studio-clip">
        <path d="M179.4,86c.1-.4,0-1-.5-1.2-.7-.4-1.3-.1-1.2-1,.2-1.9.8-4.2-.2-5.9-.8-1.3-2.4-1.5-3.7-1-5.4,1.5-6.7,9.8-10.1,12.7-2,1.5-2.2,0-1.6-1.7.4-.9.9-1.8,1.6-2.7,1.1-1.6,3.5-4.2,3.1-6.1-.4-1-1.7-1-2.4-.5-2,1.3-4.2,7.3-5.7,9.8-.6.9-1.4,1.9-2.6,2-.7,0-.9-.4-.7-1,1.1-5.2,5.7-9,8-13.7.6-1.2,2.1-4.2,.4-5.1-3.8-1.4-6.1,7.2-7.2,9.8-.8.6-.7-1.8-2.8-1.7-1.1,0-2.2.7-3,1.4-3.4,2.7-3.8,6.5-6.1,9.7-.6.8-1.8,2.2-2.9,2.1-.8-.4,0-2,.2-2.6,1.2-2.4,4.6-6,4.2-8.5,0-.6-.7-1-1.3-.9-3.2.6-5.1,9.4-7.8,11.7-2.7,2-2.5-1.1-1.3-2.7,1.1-1.9,3.9-4.5,4.2-6.7,0-.6-.2-1.3-.8-1.6-1.1-.6-2.7.7-3.5,1.7-2.2,2.5-3.5,8.6-6.3,10.1-5.5,1.9-.6-6.8.8-7.9.5-.2.9,0,1.5.1.5,0,1,0,1.4-.3.6-.5,1.4-1.6,.7-2.3-.4-.3-1.1,0-1.5-.2-.3,0-.2-.3,0-.5.6-.5,2.4-1.8,2.3-3.1-.4-3.3-4.5-.7-5.2,1.2-.3.5-.4,1.1-.9,1.4-.7.4-2.4.2-2.7,1.2,0,.3,0,.6.2.9.2.3.8.2.9.5.1.3,0,.6-.1.8-.5,1.2-1.5,3-2.3,4.4-1,2.1-2.3,4.8-2.3,7.2,0,1.8,1.8,2.3,3.3,2.1,1.9-.3,3.2-1.7,4.4-3.1.3-.4.6-.7.7,0,.5,4,3.8,2.6,6.2,0,.1-.1.4-.4.5-.4.2,0,.2.2.3.4.9,4.2,4.4,1.2,6.4-1.5.2-.3.4-.4.4,0,.1,4.6,3.3,3.8,6,.2.1-.2.4-.5.5-.5.2,0,.2.5.3.7.6,3.6,3.9,1.9,5.9-.7,1-1.3.5.7.8,1.2,1.6,3.3,5.5-1.4,7-2.9.1-.1.2-.2.4,0,0,0,0,0,.1.1,0,0,0,0,0,.1.7,2.4,2.1,3.5,4.1,2.7,2.7-1,5.4-4.2,6.6-6.8.1-.1.3-.1.6-.1.8,0,2.5-.1,2.8-1.1h0s0,0,0,0ZM149.7,91c-3.7.9.3-7.8,3.1-8.8.6-.2,1.4,0,1.5.7.2,1.2-.4,2.6-1,3.8-.5.9-1.1,1.9-1.7,2.6-.8.9-1.2,1.5-1.9,1.8h0ZM173,87.1c-.4,1-2.4,2.9-3.7,2-1.2-1.2.2-3.9.7-5.4,0-.1,0-.2.1-.1,0,.1.2.5.4,1,.3.5.6,1,1,1.4.5.4,1.4.6,1.4,1.2h0s0,0,0,0ZM174.5,84.6c-2.7.2-2-5,0-5.3,1.8-.1.9,2.9.6,3.9-.2.6-.2,1.1-.6,1.4h0ZM115.8,80.6c0-1.6,2-5,3.9-3.5,1.6,1.1,1.1,3.3-.9,4.1-.4.2-.9.5-.8,1,.1,1.2,1.5,2,2.7,1.9,2.5-.2,4.1-2.4,4.1-4.7,0-3.1-3-5.2-6-4.8-2.7.3-5.6,1.9-7,4.3-2.8,4.9,3.2,8.8,2.8,13.8-.1,3-4.1,4.9-6.6,3.8-.8-.4-1.3-1.2-1.1-2.5,0-.2,0-.5.2-.7.3-.4.8-.4,1.2-.5,2.1-.7,2.7-2.1,2.2-3.5-.9-2.9-4.8-3.4-7.2-1.4-1.3,1.1-2,2.7-2.1,4.4-.2,2.9,1.2,6.1,3.8,7.5,7.3,4.3,19.5-3.2,15.5-11.6-1.2-2.5-4.5-4.6-4.8-7.6h0s0,0,0,0Z"/>
      </clipPath>
    </defs>

    <path bind:this={studioDotEl} id="studio_x5F_i" fill="none" d="M166.7,76.6c1.2.6,2.7-.3,3.3-1.4.5-1,.7-2.3.3-3.1-.6-.9-2-.8-2.8-.1-1.3,1-2.3,3.4-.8,4.6h0s0,0,0,0Z"/>
    <path id="pudding" class="base" d="M239.6,22.8c-3.2-5.6-10.4-2.1-12.4,2.9-.3.4-.4.8-.8.6-.6-.4-1.5-1.9-2.5-2.2-2.6-1.3-6-.4-8.4,1.1-9.6,5.6-10.8,18.6-17,25.5-1.3,1.5-5.2,4.3-6.6,1.8-1.8-6.3,11.7-13.1,13.2-21,1.4-5.7-3.1-8.7-8.3-7-4,1.2-7.8,4.6-10.7,7.5-.3.2-.5.6-.9.7,0,0-.1,0-.1-.1,0-.4.5-1.9.5-2.7.2-1.6-.7-3.4-2.2-4-2.1-.7-4.5.6-6.1,1.9-8.3,7.5-6.4,22.6-17.4,27.8-2.3,1-4.2,0-4-2.5.6-7.4,9-12.2,11.7-19,1-2.3,1.7-6-1.5-6.8h0c-7.3-.7-10.9,16.3-14.3,21.4-1.5,2.5-7.3,12.5-10.2,7.9-.6-2.9,1.7-6.8,3.2-9.3,4.9-7.8,12.1-14.5,14.7-23.5.5-2,.6-4.3-1.1-5.7-1.2-1-3.3-1.4-4.8-.9-5.9,2.4-5.9,12.3-8.6,17.3-.1.1-.3.1-.4,0-.3-.3-.6-1.4-1.1-2.1-1.1-1.7-3.2-2.3-5.2-2-12.2,2.5-13.6,17.2-19.8,25.7-1.4,2.1-6.7,7.6-8.2,3-.8-9.9,19.7-26.2,18.3-35.5-.4-2.1-2.2-3.4-4.2-3.6-7.8-.9-8,12-10.8,17.5-.3.5-.7-.4-1-1-2.6-5.5-9.1-3.5-12.9,0-7.5,6.6-10,22.9-17.8,27.1-1.4.6-3.2.5-3.6-1.2-1.4-8.8,12.4-15.6,12.7-24.5,0-1.3-.5-2.7-1.8-3.2-1.7-.6-3.8.5-5.1,1.6-4.9,4.2-6.5,11.7-9.3,17.3-2,3.8-5.2,8.9-8.1,11.2-.9.7-2.3,1.3-3.5.9-2.1-1.2-.3-5.3.8-7,3.3-5.9,11.9-12.8,11.6-19.8-.3-3.1-4.2-3.1-6.2-1.7-6.4,4-11.2,16.9-13.6,24-1.4,4.6-4.7,14,1,16,6,1.4,10.3-6.7,14-10.8.2-.2.7-.8.6-.3-1,2.6-2.4,6.1-.7,8.5,1.5,2.2,4.6,1.5,6.5.1,3.1-2,5.4-5.2,7.8-8.2.1-.2.3-.3.4-.5.2-.2.4-.3.5-.3.3.4,0,1.1,0,1.9,0,2.3.3,5.2,2.6,6.3,5.8,2.1,10.7-5.9,13.7-10.1.5-.7,1.1-1.7.5.1-.4,1.5-1.2,3.5-1.3,4.8-.3,3.1,1.5,5.3,4.9,4.2,4.8-1.6,7.6-6,10.3-10,.2-.3.6-.8.8-.8.2-.1.2.9.1,1.5-.3,2.7-.5,6.1,2.2,7.6,4.5,2.4,10.2-4,13.1-7.8.4-.5.7-.6.6.4-.7,3.5-.8,7.3,3.8,6.8,3.9-.9,7.4-6.1,9.8-9.5,1-1.4.3,1.1.2,1.6-.4,2.3-.2,5.4,2.3,6.3,5.9,1.2,10.4-7.6,13.7-11.8,1.4-1.6-.4,2.5-.5,2.8-.9,2.6-1.4,6.4,1.4,7.3,2.7.9,5.5-.6,6.8-3,1.7-3,2.2-6.8,3.7-9.9,2.6-5.6,11.7-22.6,18.1-21.9,2.1.9.6,4.7-.2,6.3-2,4.1-5.7,7.5-8.5,11.1-2.5,3.3-5.1,7.5-5.6,11.6-.4,2.7,1.3,4.8,3.9,5.1,6.4.5,12.4-7.1,16.2-11.4,1.4-1.6.6,1,.5,1.9-.2,1.4-.2,2.9,0,4.2.6,3.6,4.3,4.6,7.4,3.6,5.7-1.9,11-10.5,14.6-15.4.2-.3.4-.3.4-.1,0,.2-.1.9-.3,1.2-3,10.5-11.7,26.1-24.3,23.4-2-.5-4-1.6-6-2.4-3.2-1.3-6.2-.8-8.3,2.2-2.9,3.9-1.9,9.6,2,12.4,4.5,3.5,10.8,2.9,15.9,1.1,16.9-5.9,19.6-25.2,26.2-39.6,3-6.6,8.2-11.9,11.7-18.2,1.2-2.2,2.3-4.9,1.2-7.1h0s0,0,0,0ZM95.6,59.9c-1.2-5.4,7.1-19.6,12.8-19.4h0c2.2.3,2.3,3.2,1.8,4.9-.5,3.3-11,22.3-14.7,14.5ZM131.3,58.1c-1.1.7-2.8.9-3.9,0h0c-1.4-1.6-.3-4.4.4-6.6,1.8-4.4,5.5-10.9,9.6-13.1.8-.4,1.8-.5,2.6-.3,3.8.8.8,7-.4,9.5-1.7,3.1-5.6,8.8-8.3,10.5h0ZM221.7,41.4c-2.2,4.2-6,10.2-10.2,11.1-1.3.2-2.8-.4-3.1-1.7-.9-3.6,1.6-9.1,3.5-12.1,2.2-3.2,4.9-7.2,8.2-8h0c6-.8,3.1,7.6,1.5,10.8h0ZM54.3,17c-.7-.3-1.5-.7-2.2-1-.6-.3-.8-.4-.8-.5h0c.3,0,.9,0,1.3,0,1.8,0,3.7,0,5.1-1,2.7-1.9,1.7-4.4-1.4-4.7-3.5-.4-6.6,1.6-9.2,3.7-.5.3-1,.5-1.6.4-15.6-2.5-40.7,9.9-33.7,28.2,3.7,9.1,17.4,6,15.5-4-.5-2.3-2.2-3.7-2.8-5.6-2.1-7.1,10.5-14,15.8-13.4.7.2.5.8.1,1.3-3.3,4.5-5.2,9.8-7.6,14.8-4.8,9.5-12.2,24.6-20.1,29.4-2.4,1.3-5.3,1.6-7.7,2.7-3.6,1.4-5.7,5.7-4.8,9.4,1.2,5.9,8,8.3,13.2,6.5,1.4-.5,2.8-1.2,4.1-2.1,5.2-3.9,8.3-10.1,10.9-15.7,2.2-4.7,3.8-9.9,5.3-14.9.4-.5,1,1.5,1.4,2,.2.3.4.6.7.8,3.8,4.1,9.8,4,14.6,1.8,14.7-6.7,20.3-30.4,4-38.1h0s0,0,0,0ZM39,47.1c-.9-.4-1.9-.9-2.5-1.6-1-1.3-.7-2.6-.3-4.1.8-3.2,2.1-6.6,3.2-9.7.9-2.5,4.1-11.1,6.3-12,1.7-.4,4.8,1.7,5.9,2.8h0c7.8,7.9-.4,28.9-12.7,24.6h0Z"/>
    <path id="the" class="base" d="M86.3,19.2c-1.3,2.2-4.4,4.7-4.9,6.9-.2,1.7,1.2,2.7,2.7,2.1,2.4-1,3.1-4.7,4.2-7,1.3-2.5,3.6-5.9,5.9-7,.5-.2,1.1-.3,1.3.3.6,3-5.3,6.2-6,10.1-.3,1.2,0,2.9,1.4,3,2.5.2,4.4-2.6,6-4.1.3-.3.5-.2.6.3.5,3.5,3.5,3.9,6.4,2.2,2.4-1.5,6.6-6,7.1-8.7.1-.8,0-1.6-1-1.5-1.1,0-2,1.4-2.2,2.3-.1.9.2,1.7-.3,2.5-1,1.9-4.1,3-5.8,1.6-.5-.4-.6-.8,0-1.2,3-1.9,7.3-5.9,8.4-9.2.2-.7.2-1.6-.2-2.3-.8-1.4-2.6-1-4-.4-3.6,1.9-6.9,7.5-8.2,11.4-.1.4-.1.8-.4,1.1-.7.4-2,.5-2.5.2-.3-.2-.4-.6-.4-1,0-1.5,1.7-2.7,2.6-3.8,1-1.2,2.2-3,2.5-4.4.2-1-.6-1.9-1.7-1.9-2.5,0-4.6,2.7-6.4,4.2-.5.4-1.4,1.1-1.8,1.4,0,0-.3.2-.2.1h0c.7-.9,1.5-1.9,2.4-2.7,2.2-2.2,6.5-4.8,6.3-8.2,0-.5-.3-.9-.7-1.1-4.5-1.6-8.8,11.5-11,14.8h0ZM103.7,13.6c.6-.5,1.9-1.3,2.6-.8,1.3,1.8-2.6,5-4,6.1-.4.3-1.2.8-1.6,1-.1,0-.3.1-.4,0h0c-.5-2.4,1.6-4.8,3.3-6.2h0ZM75.5,28.3c1.7-.4,2.8-1.3,3.9-2.7,2.4-3.1,4.4-8,5.9-10.9,1.3-2.3,2.7-4.7,4.3-6.9.4-.5.7-1.1,1.1-1.6.2-.2.3-.4.6-.5.4-.2,1-.3,1.5-.5,1.2-.5,2.5-1.6,2.8-2.8,0-.4,0-.9-.2-1.3-.3-.5-.8-.9-1.4-1.1-2.4-.4-4,2.2-5,4.1,0,.2-.2.3-.4.4-.3.2-.7,0-1.1,0-2.9-.7-5.9-2.1-9-1.2-2.7.8-5.4,2.8-5.7,5.7-.1,1.5.7,2.9,2.3,3,2.7.2,3.8-2.7,5.7-4,1.4-1,3-1.5,4.7-1.6.5,0,.9,0,1.4,0,.2,0,.4,0,.5.1h0c0,.1,0,.4-.2.5-1.7,2.8-3.4,5.6-5.3,8.3-1.3,1.6-3,4.7-5.2,4.9-1.1,0-2.2-.5-3.3,0-1.6.7-2.7,2.5-2.7,3.9-.2,2.7,2.2,4.9,4.9,4.2h0Z"/>

    <g clip-path="url(#studio-clip)" style="visibility: {started ? 'visible' : 'hidden'};">
      <path 
        bind:this={pathEl0}
        id="studio-path-0" 
        class="st0" 
        stroke-dasharray={len0 || 9999}
        stroke-dashoffset={len0 ? offset0 : 9999}
        d="M133.4,76.4s-7.2,11.6-7.6,12.9-1.4,5-1.8,6.1,1.3.6,2.7.2,4.3-2.4,5.2-3.2c1.7-1.4,6.9-12.5,6.9-12.5,0,0-6.3,13.8-6.3,14.9s5-1.9,6.8-4,5.9-11.5,5.9-11.5c0,0-5.8,14-5.8,15.1s4.3-2,5-2.1,1.1,2.2,2.1,2.1,4-2,5.1-3.1,3.8-6,4.9-7.5,7.1-13.8,7.2-14.2-5.2,10.4-5.6,10.8-2.1,1-3.3.5-3.1-1.1-4.1,0-3.3,4.5-3.5,5.7-.8,5.1-.3,6.2,4.2-1.8,4.5-2.1c.9,1.2.8,3.9,2.4,3s4.7-3.7,5.7-5,6.5-10.6,6.7-10.9-8.8,12.2-8.9,12.5,2,4,3.8,2.8,7.7-9.5,8.4-11.3c-1.2,3-3.8,9.2-3.3,10.2s5.3,0,7-2.3,3.7-7.7,3.7-9.4c0-3.2-2.6-3.3-6.3,1.2s8.1,4.7,9.8,4.3"
      />
      <path 
        bind:this={pathEl1}
        id="studio-path-1" 
        class="st0" 
        stroke-dasharray={len1 || 9999}
        stroke-dashoffset={len1 ? offset1 : 9999}
        d="M125.8,80.6c1.6.5,7.6,3.9,7.6,3.9"
      />
      <path 
        bind:this={pathEl2}
        id="studio-path-2" 
        class="st0" 
        stroke-dasharray={len2 || 9999}
        stroke-dashoffset={len2 ? offset2 : 9999}
        d="M118.3,83.5c.4-.3,3.4-1.7,3.7-2.4s.6-3.5.2-4.1-3-1.6-4.1-1.3-3.8,2.5-4.5,4.2,1.1,5.3,1.8,6,2.2,3.7,2.4,4.4c.6,1.6-.4,4.8-1,5.8s-4.3,2.4-5.4,2.6-4.8-.5-5.6-1-1.8-4.3-1.6-5.8c.5-2.2,6-3.3,6.8-3.4"
      />
    </g>

    <path 
      bind:this={puddingDotEl}
      id="pudding_x5F_i" 
      class="base"
      transform={dotTransform}
      style="visibility: {showStudioI ? 'hidden' : 'visible'}"
      d="M169.6,25.1c2.5,0,5-2.8,5.7-6.2.7-3.5-.7-6.3-3.2-6.3-2.5,0-5,2.8-5.7,6.2-.7,3.5.7,6.3,3.2,6.3Z"
    />

    <path 
      id="pudding_x5F_dupe" 
      class="base"
      transform={dupeTransform}
      d="M169.6,25.1c2.5,0,5-2.8,5.7-6.2.7-3.5-.7-6.3-3.2-6.3-2.5,0-5,2.8-5.7,6.2-.7,3.5.7,6.3,3.2,6.3Z"
    />

    <path 
      id="studio_x5F_i_final" 
      class="base"
      style="visibility: {showStudioI ? 'visible' : 'hidden'}"
      d="M166.7,76.6c1.2.6,2.7-.3,3.3-1.4.5-1,.7-2.3.3-3.1-.6-.9-2-.8-2.8-.1-1.3,1-2.3,3.4-.8,4.6h0s0,0,0,0Z"
    />
  </svg>
</main>

<style>
  :global(body) {
    margin: 0;
    background-color: #000000;
  }

  .interactive-container {
    width: 100vw;
    height: 100vh;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    cursor: pointer;
    user-select: none;
  }

  svg {
    width: 80%;
    max-width: 800px;
    height: auto;
  }

  .st0 {
        fill: none;
        stroke: #FFFEFB;
        stroke-miterlimit: 10;
        stroke-width: 11px; 
        stroke-linecap: round;
        stroke-linejoin: round;
    }

    .base {
        fill: #FFFEFB;
    }
</style>