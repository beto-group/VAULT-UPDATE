

# ViewComponent

```jsx
const { useEffect, useRef, useState, useMemo } = dc;



// This code will be placed inside the imageWorkerCode template string

// --- CUSTOM SVG PARSER AND RENDERER ---

/**
 * Parses an SVG path data string ("d" attribute).
 * Supports a limited set of absolute commands: M, L, H, V, Z.
 * @param {string} pathData The "d" attribute string.
 * @returns {Array} An array of command objects.
 */
function parsePathData(pathData) {
    // This regex splits the path data into commands and their coordinates.
    // It finds a letter (the command) followed by numbers.
    const commands = pathData.match(/[a-z][^a-z]*/ig) || [];
    let currentPoint = { x: 0, y: 0 };

    return commands.map(commandString => {
        const type = commandString.charAt(0);
        const args = (commandString.substring(1).match(/-?\d*\.?\d+/g) || []).map(parseFloat);
        
        // For this simple parser, we treat relative commands as absolute  simplicity
        // A full parser would need to track the current point meticulously.
        // H and V commands are converted to L commands.
        switch (type.toUpperCase()) {
            case 'H': // Horizontal LineTo
                currentPoint.x = args[0];
                return { type: 'L', args: [currentPoint.x, currentPoint.y] };
            case 'V': // Vertical LineTo
                currentPoint.y = args[0];
                return { type: 'L', args: [currentPoint.x, currentPoint.y] };
            case 'M': // MoveTo
                currentPoint = { x: args[0], y: args[1] };
                return { type, args };
            case 'L': // LineTo
                currentPoint = { x: args[0], y: args[1] };
                 return { type, args };
            case 'Z': // ClosePath
                return { type: 'Z', args: [] };
            default:
                // Unsupported command
                return { type: 'UNSUPPORTED', args: [] };
        }
    });
}


/**
 * Renders a parsed SVG element onto the canvas context.
 * @param {CanvasRenderingContext2D} ctx The canvas context.
 * @param {Element} element The SVG element to render.
 * @param {object} scaleInfo Scaling info from the viewBox.
 */
function renderElement(ctx, element, scaleInfo) {
    const fill = element.getAttribute('fill') || '#000'; // Default to black fill
    const stroke = element.getAttribute('stroke') || 'none';
    const strokeWidth = parseFloat(element.getAttribute('stroke-width') || '1');

    ctx.fillStyle = fill;
    ctx.strokeStyle = stroke;
    // Scale the stroke width based on the average scaling factor
    ctx.lineWidth = strokeWidth * Math.min(scaleInfo.scaleX, scaleInfo.scaleY);

    const transformPoint = (x, y) => ({
        x: (x - scaleInfo.vb.x) * scaleInfo.scaleX,
        y: (y - scaleInfo.vb.y) * scaleInfo.scaleY,
    });

    ctx.beginPath();

    switch (element.tagName.toLowerCase()) {
        case 'rect': {
            const p = transformPoint(parseFloat(element.getAttribute('x') || 0), parseFloat(element.getAttribute('y') || 0));
            const w = parseFloat(element.getAttribute('width')) * scaleInfo.scaleX;
            const h = parseFloat(element.getAttribute('height')) * scaleInfo.scaleY;
            ctx.rect(p.x, p.y, w, h);
            break;
        }
        case 'circle': {
            const p = transformPoint(parseFloat(element.getAttribute('cx') || 0), parseFloat(element.getAttribute('cy') || 0));
            const r = parseFloat(element.getAttribute('r')) * Math.min(scaleInfo.scaleX, scaleInfo.scaleY);
            ctx.arc(p.x, p.y, r, 0, 2 * Math.PI);
            break;
        }
        case 'line': {
            const p1 = transformPoint(parseFloat(element.getAttribute('x1') || 0), parseFloat(element.getAttribute('y1') || 0));
            const p2 = transformPoint(parseFloat(element.getAttribute('x2') || 0), parseFloat(element.getAttribute('y2') || 0));
            ctx.moveTo(p1.x, p1.y);
            ctx.lineTo(p2.x, p2.y);
            break;
        }
        case 'path': {
            const pathData = element.getAttribute('d');
            if (!pathData) break;
            const commands = parsePathData(pathData);
            
            for (const cmd of commands) {
                if (cmd.type === 'M') {
                    const p = transformPoint(cmd.args[0], cmd.args[1]);
                    ctx.moveTo(p.x, p.y);
                } else if (cmd.type === 'L') {
                    const p = transformPoint(cmd.args[0], cmd.args[1]);
                    ctx.lineTo(p.x, p.y);
                } else if (cmd.type === 'Z') {
                    ctx.closePath();
                }
            }
            break;
        }
    }

    if (fill !== 'none') {
        ctx.fill();
    }
    if (stroke !== 'none') {
        ctx.stroke();
    }
}

/**
 * Main custom function to parse and render SVG text to a canvas.
 * Replaces the need for the canvg library for basic SVGs.
 * @param {string} svgText The raw SVG file content.
 * @param {number} width The target canvas width.
 * @param {number} height The target canvas height.
 * @returns {Promise<ImageBitmap>} A promise that resolves with the rendered ImageBitmap.
 */
async function parseAndRenderSvg(svgText, width, height) {
    const canvas = new OffscreenCanvas(width, height);
    const ctx = canvas.getContext('2d');

    try {
        // Use the browser's built-in DOM parser to turn the SVG string into a traversable structure
        const parser = new DOMParser();
        const doc = parser.parseFromString(svgText, "image/svg+xml");
        const svgNode = doc.documentElement;
        
        if (svgNode.tagName.toLowerCase() !== 'svg' || svgNode.querySelector('parsererror')) {
             throw new Error("Invalid SVG file.");
        }

        const viewBox = svgNode.getAttribute('viewBox');
        let vbParts = viewBox ? viewBox.split(/[ ,]+/).map(parseFloat) : [0, 0, width, height];
        if (vbParts.length !== 4) vbParts = [0, 0, width, height];
        
        const scaleInfo = {
            vb: { x: vbParts[0], y: vbParts[1], w: vbParts[2], h: vbParts[3] },
            scaleX: width / vbParts[2],
            scaleY: height / vbParts[3],
        };

        // Render each top-level element in the SVG
        for (const element of svgNode.children) {
            renderElement(ctx, element, scaleInfo);
        }

        return self.createImageBitmap(canvas);

    } catch (err) {
        console.error("Custom SVG parser failed:", err);
        // Draw an error state on the canvas
        ctx.fillStyle = '#401010';
        ctx.fillRect(0, 0, width, height);
        ctx.fillStyle = 'white';
        ctx.textAlign = 'center';
        ctx.fillText('SVG Parse Error', width / 2, height / 2);
        return self.createImageBitmap(canvas); // Return the error bitmap
    }
}


// --- UTILITY FUNCTIONS (Unchanged) ---
function findNearestAncestorWithClass(element, className) {
  if (!element) return null;
  let current = element.parentNode;
  while (current) {
    if (current.classList && current.classList.contains(className)) return current;
    current = current.parentNode;
  }
  return null;
}
function findDirectChildByClass(parent, className) {
  if (!parent) return null;
  for (const child of parent.children) {
    if (child.classList && child.classList.contains(className)) return child;
  }
  return null;
}

// --- CUSTOM HOOK for Fullscreen "Portal" Effect (Unchanged) ---
function useFullscreenEffect(containerRef, isFullTab) {
  const stateRefs = useRef({}).current;
  useEffect(() => {
    const container = containerRef.current;
    if (!container || !isFullTab) return;
    const timer = setTimeout(() => {
      const t = findNearestAncestorWithClass(container, "workspace-leaf-content");
      if (!t) return;
      const contentWrapper = findDirectChildByClass(t, "view-content") || t;
      stateRefs.originalParent = container.parentNode;
      stateRefs.placeholder = document.createElement("div");
      if (container.parentNode) { container.parentNode.insertBefore(stateRefs.placeholder, container); }
      const originalPosition = window.getComputedStyle(contentWrapper).position;
      stateRefs.parentPositionInfo = { element: contentWrapper, originalInlinePosition: contentWrapper.style.position };
      if (originalPosition === "static") { contentWrapper.style.position = "relative"; }
      contentWrapper.appendChild(container);
      container.classList.add('fullscreen-active');
    }, 50);
    return () => {
      clearTimeout(timer);
      if (!stateRefs.originalParent || !container) return;
      if (stateRefs.placeholder?.parentNode) {
        stateRefs.placeholder.parentNode.replaceChild(container, stateRefs.placeholder);
      } else if (stateRefs.originalParent) {
        stateRefs.originalParent.appendChild(container);
      }
      if (stateRefs.parentPositionInfo?.element) {
        stateRefs.parentPositionInfo.element.style.position = stateRefs.parentPositionInfo.originalInlinePosition || "";
      }
      container.classList.remove('fullscreen-active');
      Object.keys(stateRefs).forEach(k => delete stateRefs[k]);
    };
  }, [isFullTab, containerRef]);
}


// --- WEB WORKER with CUSTOM SVG PARSER ---
const imageWorkerCode = `
    // --- CUSTOM SVG PARSER AND RENDERER ---
    
    function parsePathData(pathData) {
        const commands = pathData.match(/[a-z][^a-z]*/ig) || [];
        let currentPoint = { x: 0, y: 0 };
        const results = [];

        for (const commandString of commands) {
            const type = commandString.charAt(0);
            const args = (commandString.substring(1).match(/-?\\d*\\.?\\d+/g) || []).map(parseFloat);
            
            let cmd = { type: 'UNSUPPORTED', args: [] };
            switch (type) { // Note: No toUpperCase(), this simple parser is case-sensitive for M/L/H/V
                case 'H': currentPoint.x = args[0]; cmd = { type: 'L', args: [currentPoint.x, currentPoint.y] }; break;
                case 'h': currentPoint.x += args[0]; cmd = { type: 'L', args: [currentPoint.x, currentPoint.y] }; break;
                case 'V': currentPoint.y = args[0]; cmd = { type: 'L', args: [currentPoint.x, currentPoint.y] }; break;
                case 'v': currentPoint.y += args[0]; cmd = { type: 'L', args: [currentPoint.x, currentPoint.y] }; break;
                case 'M': currentPoint = { x: args[0], y: args[1] }; cmd = { type, args }; break;
                case 'm': currentPoint.x += args[0]; currentPoint.y += args[1]; cmd = { type: 'M', args: [currentPoint.x, currentPoint.y] }; break;
                case 'L': currentPoint = { x: args[0], y: args[1] }; cmd = { type, args }; break;
                case 'l': currentPoint.x += args[0]; currentPoint.y += args[1]; cmd = { type: 'L', args: [currentPoint.x, currentPoint.y] }; break;
                case 'Z': case 'z': cmd = { type: 'Z', args: [] }; break;
            }
            results.push(cmd);
        }
        return results;
    }

    function renderElement(ctx, element, scaleInfo) {
        const fill = element.getAttribute('fill') || '#000';
        const stroke = element.getAttribute('stroke') || 'none';
        const strokeWidth = parseFloat(element.getAttribute('stroke-width') || '1');
        ctx.fillStyle = fill;
        ctx.strokeStyle = stroke;
        ctx.lineWidth = strokeWidth * Math.min(scaleInfo.scaleX, scaleInfo.scaleY);
        const transformPoint = (x, y) => ({ x: (x - scaleInfo.vb.x) * scaleInfo.scaleX, y: (y - scaleInfo.vb.y) * scaleInfo.scaleY });
        ctx.beginPath();
        switch (element.tagName.toLowerCase()) {
            case 'rect': {
                const p = transformPoint(parseFloat(element.getAttribute('x') || 0), parseFloat(element.getAttribute('y') || 0));
                const w = parseFloat(element.getAttribute('width')) * scaleInfo.scaleX;
                const h = parseFloat(element.getAttribute('height')) * scaleInfo.scaleY;
                ctx.rect(p.x, p.y, w, h);
                break;
            }
            case 'circle': {
                const p = transformPoint(parseFloat(element.getAttribute('cx') || 0), parseFloat(element.getAttribute('cy') || 0));
                const r = parseFloat(element.getAttribute('r')) * Math.min(scaleInfo.scaleX, scaleInfo.scaleY);
                ctx.arc(p.x, p.y, r, 0, 2 * Math.PI);
                break;
            }
            case 'line': {
                const p1 = transformPoint(parseFloat(element.getAttribute('x1') || 0), parseFloat(element.getAttribute('y1') || 0));
                const p2 = transformPoint(parseFloat(element.getAttribute('x2') || 0), parseFloat(element.getAttribute('y2') || 0));
                ctx.moveTo(p1.x, p1.y);
                ctx.lineTo(p2.x, p2.y);
                break;
            }
            case 'path': {
                const pathData = element.getAttribute('d');
                if (!pathData) break;
                const commands = parsePathData(pathData);
                for (const cmd of commands) {
                    if (cmd.type === 'M') { const p = transformPoint(cmd.args[0], cmd.args[1]); ctx.moveTo(p.x, p.y); } 
                    else if (cmd.type === 'L') { const p = transformPoint(cmd.args[0], cmd.args[1]); ctx.lineTo(p.x, p.y); } 
                    else if (cmd.type === 'Z') { ctx.closePath(); }
                }
                break;
            }
        }
        if (fill !== 'none') { ctx.fill(); }
        if (stroke !== 'none') { ctx.stroke(); }
    }

    async function parseAndRenderSvg(svgText, width, height) {
        const canvas = new OffscreenCanvas(width, height);
        const ctx = canvas.getContext('2d');
        try {
            const parser = new DOMParser();
            const doc = parser.parseFromString(svgText, "image/svg+xml");
            const svgNode = doc.documentElement;
            if (svgNode.tagName.toLowerCase() !== 'svg' || svgNode.querySelector('parsererror')) { throw new Error("Invalid SVG file."); }
            const viewBox = svgNode.getAttribute('viewBox');
            let vbParts = viewBox ? viewBox.split(/[ ,]+/).map(parseFloat) : [0, 0, width, height];
            if (vbParts.length !== 4) vbParts = [0, 0, width, height];
            const scaleInfo = { vb: { x: vbParts[0], y: vbParts[1], w: vbParts[2], h: vbParts[3] }, scaleX: width / vbParts[2], scaleY: height / vbParts[3] };
            for (const element of svgNode.children) { renderElement(ctx, element, scaleInfo); }
            return self.createImageBitmap(canvas);
        } catch (err) {
            console.error("Custom SVG parser failed:", err);
            ctx.fillStyle = '#401010'; ctx.fillRect(0, 0, width, height); ctx.fillStyle = 'white'; ctx.textAlign = 'center'; ctx.fillText('SVG Parse Error', width / 2, height / 2);
            return self.createImageBitmap(canvas);
        }
    }
    
    // --- Worker Message Handler ---
    self.onmessage = async function(e) {
        const { type, imagesToLoad } = e.data;
        if (type === 'generate') {
            const results = {};
            const transferable = [];
            for (const image of imagesToLoad) {
                const { path, svgText } = image;
                try {
                    // *** THIS IS THE KEY CHANGE: We call our custom function now ***
                    const bitmap = await parseAndRenderSvg(svgText, 320, 420);
                    results[path] = { bitmap };
                    transferable.push(bitmap);
                } catch (error) {
                    console.error(\`Worker with custom parser failed to process SVG: \${path}\`, error);
                    results[path] = { error: error.message };
                }
            }
            self.postMessage({ type: 'generated', results }, transferable);
        }
    };
`;

// --- ASYNCHRONOUS & NON-BLOCKING CANVAS HOOK (Unchanged) ---
function useInteractiveCanvas(refs, isFullTab, onCardClick, imageFiles, worker) {
  const { containerRef, canvasRef, tooltipRef } = refs;
  const imageCache = useRef(new Map()).current;
  const requestedImages = useRef(new Set()).current;

  useEffect(() => {
    if (!isFullTab || !imageFiles || imageFiles.length === 0 || !worker) return;
    
    worker.onmessage = (e) => {
      const { type, results } = e.data;
      if (type === 'generated') {
        for (const path in results) {
          const { bitmap, error } = results[path];
          if (bitmap) {
            imageCache.set(path, { bitmap, url: null });
          } else {
            imageCache.set(path, { error: true });
          }
          requestedImages.delete(path);
        }
        requestRender();
      }
    };
    
    const root = containerRef.current;
    const canvas = canvasRef.current;
    const tooltip = tooltipRef.current;
    if (!canvas || !tooltip || !root) return;

    // --- All rendering logic from here is unchanged ---
    const ctx = canvas.getContext("2d", { alpha: false, desynchronized: true });
    let rafId = 0, isRunning = false, lastFrameTime = performance.now();
    let W = 1, H = 1, DPR = 1;
    let camX = 0, camY = 0, vX = 0, vY = 0;
    let zoom = 1, zTarget = 1;
    let mx = 0, my = 0, isPointerDown = false, downStart = { x: 0, y: 0 };
    const CARD_W = 232, CARD_H = 288, GAP = 26;
    const TILE_W = CARD_W + GAP, TILE_H = CARD_H + GAP;
    
    const imageFor = (i, j) => {
      const hash = Math.abs(((i * 73856093) ^ (j * 19349663)) >>> 0);
      const index = hash % imageFiles.length;
      return imageFiles[index];
    };

    const worldFromScreen = (sx, sy) => ({ x: (sx - W / 2) / zoom + camX, y: (sy - H / 2) / zoom + camY });
    const getTileFromWorldPoint = (wx, wy) => { const i = Math.floor(wx/TILE_W); const j = Math.floor(wy/TILE_H); const localX=wx-(i*TILE_W), localY=wy-(j*TILE_H); const overCard = localX>=GAP/2 && localX<=GAP/2+CARD_W && localY>=GAP/2 && localY<=GAP/2+CARD_H; return { i, j, overCard }; };
    const requestRender = () => { if (!isRunning) { isRunning = true; lastFrameTime = performance.now(); rafId = requestAnimationFrame(frame); } };

    const frame = () => {
      const now = performance.now(); const dt = Math.min(0.1, (now - lastFrameTime) / 1000); lastFrameTime = now;
      vX *= Math.exp(-7*dt); vY *= Math.exp(-7*dt); camX += vX*60*dt; camY += vY*60*dt; zoom += (zTarget - zoom) * (1 - Math.exp(-15*dt));
      ctx.fillStyle = "#0a0f0a"; ctx.fillRect(0,0,W,H);
      const halfW = W/(2*zoom), halfH=H/(2*zoom); const view = {left:camX-halfW, right:camX+halfW, top:camY-halfH, bottom:camY+halfH};
      const i0=Math.floor(view.left/TILE_W)-1, i1=Math.floor(view.right/TILE_W)+1, j0=Math.floor(view.top/TILE_H)-1, j1=Math.floor(view.bottom/TILE_H)+1;
      ctx.save(); ctx.translate(W/2,H/2); ctx.scale(zoom,zoom); ctx.translate(-camX,-camY);
      const projCardWidth = CARD_W * zoom * DPR; const imagesToLoad = new Set();
      for (let j=j0; j<=j1; j++) for(let i=i0; i<=i1; i++) {
          const file=imageFor(i,j); if(!file) continue;
          const path = file.path; const image = imageCache.get(path);
          if(!image && !requestedImages.has(path)) { imagesToLoad.add(file); requestedImages.add(path); }
          const x=i*TILE_W+GAP/2, y=j*TILE_H+GAP/2;
          if(image?.bitmap && projCardWidth >= 40) { ctx.drawImage(image.bitmap, x, y, CARD_W, CARD_H); }
          else { ctx.fillStyle = image?.error ? "#401010" : "#1a2b20";
              if(projCardWidth < 5) { ctx.fillRect(x+CARD_W/2-1, y+CARD_H/2-1, 2, 2); }
              else { ctx.fillRect(x,y,CARD_W,CARD_H); }
          }
      }
      if(imagesToLoad.size > 0) { Promise.all(Array.from(imagesToLoad).map(async file => ({ path: file.path, svgText: await dc.app.vault.read(file) }))).then(imageData => { worker.postMessage({ type: 'generate', imagesToLoad: imageData }); }); }
      ctx.restore();
      const isMoving = Math.abs(vX)>0.001 || Math.abs(vY)>0.001 || Math.abs(zTarget-zoom)>0.0001; if(isMoving) { rafId = requestAnimationFrame(frame); } else { isRunning = false; }
    };

    const stopRender = () => { isRunning=false; cancelAnimationFrame(rafId); };
    const sizeToContainer = () => { const r=root.getBoundingClientRect(); const dpr=Math.min(1.5, window.devicePixelRatio||1); if(W!==r.width||H!==r.height||DPR!==dpr){ W=r.width;H=r.height;DPR=dpr; canvas.width=W*DPR; canvas.height=H*DPR; ctx.setTransform(DPR,0,0,DPR,0,0); requestRender(); } };
    const onPointerDown = e => { if (e.target!==canvas || !!document.querySelector('.panel-wrap')) return; const r=canvas.getBoundingClientRect(); mx=e.clientX-r.left; my=e.clientY-r.top; downStart={x:mx,y:my}; isPointerDown=true; canvas.setPointerCapture?.(e.pointerId); canvas.style.cursor="grabbing"; vX*=0.2; requestRender(); };
    const onPointerMove = e => { const r=canvas.getBoundingClientRect(); const pMx=mx, pMy=my; mx=e.clientX-r.left; my=e.clientY-r.top; if (isPointerDown) { const dx=mx-pMx, dy=my-pMy, k=1/zoom; camX-=dx*k; camY-=dy*k; vX=-(dx*k)*0.5; vY=-(dy*k)*0.5; } requestRender(); };
    const onPointerUp = e => { if (!isPointerDown) return; isPointerDown=false; canvas.releasePointerCapture?.(e.pointerId); requestRender(); };
    const onWheel = e => { if (!!document.querySelector('.panel-wrap')) return; e.preventDefault(); const wpb=worldFromScreen(mx,my); const factor=Math.exp(-e.deltaY*0.002); zTarget=Math.max(0.05, Math.min(5, zTarget*factor)); const wpa=worldFromScreen(mx,my); camX+=wpb.x-wpa.x; camY+=wpb.y-wpa.y; requestRender(); };
    const onClick = (e) => {
        if (!!document.querySelector('.panel-wrap') || Math.hypot(mx-downStart.x, my-downStart.y)>5) return;
        const clicked = getTileFromWorldPoint(worldFromScreen(mx, my).x, worldFromScreen(mx,my).y);
        if (clicked.overCard) {
            const file = imageFor(clicked.i, clicked.j);
            const image = imageCache.get(file.path);
            if (!image?.bitmap) return;
            if (!image.url) {
                const tempCanvas=document.createElement('canvas'); tempCanvas.width=image.bitmap.width; tempCanvas.height=image.bitmap.height;
                tempCanvas.getContext('2d').drawImage(image.bitmap, 0, 0);
                image.url = tempCanvas.toDataURL('image/jpeg', 0.8);
            }
            onCardClick({ path: file.path, url: image.url, i: clicked.i, j: clicked.j });
        }
    };
    
    sizeToContainer();
    const resizeObserver = new ResizeObserver(sizeToContainer);
    resizeObserver.observe(root);
    canvas.addEventListener("pointerdown", onPointerDown);
    window.addEventListener("pointermove", onPointerMove, { passive: true });
    window.addEventListener("pointerup", onPointerUp, { passive: true });
    canvas.addEventListener("wheel", onWheel, { passive: false });
    canvas.addEventListener("click", onClick);
    requestRender();

    return () => {
      stopRender();
      resizeObserver.disconnect();
      if (worker) worker.onmessage = null;
      window.removeEventListener("pointermove", onPointerMove);
      window.removeEventListener("pointerup", onPointerUp);
    };
  }, [isFullTab, onCardClick, imageCache, requestedImages, imageFiles, worker]);
}

// --- Main React Component ---
function BasicView() {
  const [isFullTab, setIsFullTab] = useState(true);
  const [panel, setPanel] = useState(null);
  const [worker, setWorker] = useState(null);
  const [workerError, setWorkerError] = useState(null);
  
  const containerRef = useRef(null);
  const canvasRef = useRef(null);
  const tooltipRef = useRef(null);

  // --- CONFIGURATION ---
  const FOLDER_PATH = "_RESOURCES/ASSETS/888/ASSETS_.A";

  const imageFiles = useMemo(() => {
    try {
      const allFiles = dc.app.vault.getFiles();
      return allFiles.filter(file => 
        file.path.startsWith(FOLDER_PATH) && file.extension === 'svg'
      );
    } catch (e) {
      console.error("Failed to get SVG files from vault:", e);
      new Notice("Could not load SVG files.", 5000);
      return [];
    }
  }, []);

  useEffect(() => {
    let workerInstance;
    
    function initializeWorker() {
      try {
        // We no longer need to find any external library. We just create the worker.
        const blob = new Blob([imageWorkerCode], { type: 'application/javascript' });
        workerInstance = new Worker(URL.createObjectURL(blob));
        setWorker(workerInstance);
      } catch (err) {
        console.error("Worker Initialization Failed:", err);
        new Notice(`CRITICAL ERROR: ${err.message}`, 15000);
        setWorkerError(err.message);
      }
    }
    
    initializeWorker();

    return () => {
      if (workerInstance) {
        workerInstance.terminate();
      }
    };
  }, []);

  useFullscreenEffect(containerRef, isFullTab);
  useInteractiveCanvas({ containerRef, canvasRef, tooltipRef }, isFullTab, setPanel, imageFiles, worker);
  
  useEffect(() => {
    const handleKeydown = (e) => { if (e.key === "Escape" && panel) { setPanel(null); } };
    window.addEventListener("keydown", handleKeydown);
    return () => window.removeEventListener("keydown", handleKeydown);
  }, [panel]);

  const handleExitFullTab = (e) => { e.stopPropagation(); setIsFullTab(false); };
  const handleEnterFullTab = () => setIsFullTab(true);
  
  if (workerError) {
      return (
          <div style={{ padding: '16px', textAlign: 'center' }}>
              <p style={{color: '#ff8a8a'}}>Worker Failed to Initialize</p>
              <p style={{color: '#aaa', fontSize: '12px' }}>{workerError}</p>
          </div>
      )
  }
  if (imageFiles.length === 0 && isFullTab) {
    return (
        <div style={{ padding: '16px', textAlign: 'center' }}>
            <p>No SVG files found in the specified folder.</p>
            <p style={{ color: '#888', fontSize: '12px' }}>Checked: {FOLDER_PATH}</p>
        </div>
    )
  }

  return (
    <div ref={containerRef}>
      <style>{`
        /* All styles are unchanged and complete */
        .full-tab-wrapper { position: relative; height: 100%; width: 100%; padding: 0; box-sizing: border-box; display: flex; align-items: stretch; justify-content: stretch; background: #0a0f0a; border: 1px solid var(--background-modifier-border); border-radius: 10px; color: var(--text-normal); overflow: hidden; }
        .interactive-canvas { display: block; width: 100%; height: 100%; cursor: grab; touch-action: none; background-color: #0a0f0a; }
        .overlay { position: absolute; inset: 0; pointer-events: none; }
        .subtle-icon { position: absolute; top: 14px; right: 18px; font-family: ui-monospace, monospace; font-size: 13px; color: rgba(180,220,200,.6); user-select: none; cursor: pointer; opacity: 0; transform: scale(.95); transition: opacity .2s, transform .2s; z-index: 10; pointer-events: auto; }
        .full-tab-wrapper:hover .subtle-icon { opacity: .75; transform: scale(1); }
        .fullscreen-active { position: absolute; top: 0; left: 0; width: 100%; height: 100%; z-index: 9998; overflow: hidden; }
        .tooltip { position: absolute; top: 0; left: 0; padding: 6px 10px; font-family: ui-monospace, monospace; font-size: 12px; color: rgb(0,255,140); background: rgba(10,18,12,.95); border: 1px solid rgba(0,255,140,.35); border-radius: 10px; box-shadow: 0 8px 24px rgba(0,0,0,.35); display: none; white-space: nowrap; z-index: 9999; }
        @keyframes fadeIn { from { opacity: 0; } to { opacity: 1; } }
        @keyframes scaleIn { from { transform: scale(0.97); } to { transform: scale(1); } }
        .panel-wrap { position: absolute; inset: 0; display: flex; align-items: center; justify-content: center; background: rgba(0,0,0,.28); backdrop-filter: blur(6px); pointer-events: auto; animation: fadeIn 0.3s cubic-bezier(0.25, 1, 0.5, 1); z-index: 100; }
        .panel { width: min(96vw, 980px); background: #0b120c; border: 1px solid rgba(0,255,140,.25); border-radius: 16px; box-shadow: 0 24px 80px rgba(0,0,0,.5); overflow: hidden; display: grid; grid-template-columns: 1.2fr 1fr; gap: 0; animation: scaleIn 0.3s cubic-bezier(0.25, 1, 0.5, 1); }
        .panel-img-box { position: relative; background: radial-gradient(1200px 800px at 50% 50%, rgba(0,255,140,.06), transparent); display: grid; place-items: center; padding: 16px;}
        .panel-img { display: block; width: 100%; height: 100%; object-fit: contain; }
        .panel-meta { padding: 24px; display: grid; gap: 12px; color: rgba(205,255,228,.92); align-content: start;}
        .panel-title { font-size: 18px; font-weight: 700; color: rgb(0,255,140); letter-spacing: .2px; word-break: break-all; }
        .panel-row { font-size: 13px; line-height: 1.45; color: rgba(190,255,215,.9); word-break: break-all; }
        .btn-row { display: flex; gap: 10px; margin-top: 6px; }
        .btn { padding: 10px 14px; font-size: 12px; font-weight: 700; letter-spacing: .2px; border-radius: 12px; border: 1px solid rgba(0,255,140,.35); background: rgba(10,18,12,.9); color: rgb(0,255,140); cursor: pointer; transition: background-color 0.2s, color 0.2s; }
        .btn:hover { background: rgba(0,255,140,.2); }
        .btn.ghost { border-color: rgba(0,255,140,.2); background: transparent; color: rgba(0,255,140,.85); }
        .btn.ghost:hover { background: rgba(0,255,140,.1); color: rgb(0,255,140); }
        .compact-wrapper { padding: 16px; box-sizing: border-box; display: flex; flex-direction: column; align-items: center; justify-content: center; gap: 12px; border: 1px dashed var(--background-modifier-border); border-radius: 8px; background-color: var(--background-primary-alt); }
        .compact-text { margin: 0; color: var(--text-muted); font-size: 14px; }
        .button-group { display: flex; gap: 10px; }
        .button { padding: 8px 16px; font-size: 12px; font-weight: 500; color: var(--text-on-accent); background-color: var(--interactive-accent); border: none; border-radius: 6px; cursor: pointer; }
        .button.secondary { background-color: var(--background-modifier-hover); color: var(--text-muted); }
      `}</style>
      
      {isFullTab ? (
        <div className="full-tab-wrapper">
          <span className="subtle-icon" title="Exit" onClick={handleExitFullTab}>&lt;/&gt;</span>
          <canvas ref={canvasRef} className="interactive-canvas"/>
          <div className="overlay">
            <div ref={tooltipRef} className="tooltip"></div>
            {panel && (
              <div className="panel-wrap" onClick={(e) => { if (e.target === e.currentTarget) setPanel(null); }}>
                <div className="panel">
                  <div className="panel-img-box">
                    <img className="panel-img" src={panel.url} alt={panel.path} />
                  </div>
                  <div className="panel-meta">
                    <div className="panel-title">{panel.path.split('/').pop()}</div>
                    <div className="panel-row">Path: {panel.path}</div>
                    <div className="panel-row">Grid position: ({panel.i}, {panel.j})</div>
                    <div className="btn-row">
                      <button className="btn" onClick={() => setPanel(null)}>Close</button>
                      <button className="btn ghost" onClick={() => new Notice("Action placeholder", 1500)}>Action</button>
                    </div>
                  </div>
                </div>
              </div>
            )}
          </div>
        </div>
      ) : (
        <div className="compact-wrapper">
          <p className="compact-text">Component is in compact mode.</p>
          <div className="button-group">
            <button className="button" onClick={handleEnterFullTab}>Enter Full Tab</button>
          </div>
        </div>
      )}
    </div>
  );
}

return { BasicView };
```


