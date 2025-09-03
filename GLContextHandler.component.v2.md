

# ViewComponent


```jsx
// Ensure 'dc' is available in the environment where this component runs.
// For React: import React, { useRef, useEffect, useState, useCallback } from 'react';
const { useRef, useEffect, useState, useCallback } = dc;
// Ensure Preact is available for FreshPip
const { h: preactH, render: preactRender } = dc.preact || { h: null, render: null };

if (!preactH || !preactRender) {
    console.error("WorldView PiP: dc.preact.h or dc.preact.render is not available. PiP functionality will not work.");
}

// --- WebGL Context Logger Utility (Full function as provided before) ---
function wrapWebGLContext(gl, options = {}) {
    const {
        logArgs = true, logResult = false, contextName = 'WebGL', maxArrayLength = 16,
        traceFunctions = null, logGetError = true, checkErrorAfterEachCall = false
    } = options;
    const glConstants = {};
    for (const key in gl) {
        if (typeof gl[key] === 'number' && key.toUpperCase() === key) {
            if (!glConstants[gl[key]] || glConstants[gl[key]].length > key.length) {
                glConstants[gl[key]] = key;
            }
        }
    }
    const commonConstants = {
        [gl.ZERO]: "ZERO", [gl.ONE]: "ONE", [gl.SRC_ALPHA]: "SRC_ALPHA", [gl.ONE_MINUS_SRC_ALPHA]: "ONE_MINUS_SRC_ALPHA",
        [gl.DEPTH_TEST]: "DEPTH_TEST", [gl.BLEND]: "BLEND", [gl.CULL_FACE]: "CULL_FACE", [gl.TEXTURE_2D]: "TEXTURE_2D",
        [gl.UNSIGNED_BYTE]: "UNSIGNED_BYTE", [gl.FLOAT]: "FLOAT", [gl.TRIANGLES]: "TRIANGLES", [gl.LINES]: "LINES",
        [gl.COLOR_BUFFER_BIT]: "COLOR_BUFFER_BIT", [gl.DEPTH_BUFFER_BIT]: "DEPTH_BUFFER_BIT", [gl.STENCIL_BUFFER_BIT]: "STENCIL_BUFFER_BIT",
        [gl.ARRAY_BUFFER]: "ARRAY_BUFFER", [gl.ELEMENT_ARRAY_BUFFER]: "ELEMENT_ARRAY_BUFFER", [gl.STATIC_DRAW]: "STATIC_DRAW",
        [gl.DYNAMIC_DRAW]: "DYNAMIC_DRAW", [gl.STREAM_DRAW]: "STREAM_DRAW", [gl.FRAGMENT_SHADER]: "FRAGMENT_SHADER", [gl.VERTEX_SHADER]: "VERTEX_SHADER",
        [gl.COMPILE_STATUS]: "COMPILE_STATUS", [gl.LINK_STATUS]: "LINK_STATUS", [gl.ACTIVE_TEXTURE]: "ACTIVE_TEXTURE",
        [gl.TEXTURE0]: "TEXTURE0", [gl.CLAMP_TO_EDGE]: "CLAMP_TO_EDGE", [gl.LINEAR]: "LINEAR", [gl.NEAREST]: "NEAREST",
        [gl.NO_ERROR]: "NO_ERROR", [gl.INVALID_ENUM]: "INVALID_ENUM", [gl.INVALID_VALUE]: "INVALID_VALUE",
        [gl.INVALID_OPERATION]: "INVALID_OPERATION", [gl.OUT_OF_MEMORY]: "OUT_OF_MEMORY", [gl.INVALID_FRAMEBUFFER_OPERATION]: "INVALID_FRAMEBUFFER_OPERATION",
        [gl.CONTEXT_LOST_WEBGL]: "CONTEXT_LOST_WEBGL"
    };
    for(const val in commonConstants) { glConstants[Number(val)] = commonConstants[val]; }
    function formatArg(arg) {
        if (arg === null) return "null"; if (arg === undefined) return "undefined";
        if (glConstants[arg] !== undefined) return `${glConstants[arg]} (${arg})`;
        const prototypes = [
            { proto: window.WebGLBuffer, name: "WebGLBuffer" }, { proto: window.WebGLTexture, name: "WebGLTexture" },
            { proto: window.WebGLProgram, name: "WebGLProgram" }, { proto: window.WebGLShader, name: "WebGLShader" },
            { proto: window.WebGLFramebuffer, name: "WebGLFramebuffer" }, { proto: window.WebGLRenderbuffer, name: "WebGLRenderbuffer" },
            { proto: window.WebGLUniformLocation, name: "WebGLUniformLocation" }
        ];
        if (window.WebGLVertexArrayObject) { prototypes.push({ proto: window.WebGLVertexArrayObject, name: "WebGLVertexArrayObject" }); }
        for (const p of prototypes) { if (p.proto && arg instanceof p.proto) return p.name; }
        if (Array.isArray(arg) || (typeof arg === "object" && arg && typeof arg.length === 'number' && typeof arg !== 'string' && !(arg instanceof String))) {
            const arrSlice = Array.from(arg.slice(0, maxArrayLength));
            if (arg.length > maxArrayLength) return `[${arg.constructor.name}, len: ${arg.length}, data: [${arrSlice.join(', ')}]...]`;
            return `[${arg.constructor.name}, len: ${arg.length}, data: [${arrSlice.join(', ')}]`;
        }
        if (typeof arg === 'string') {
             if (arg.includes('\n') && arg.length > 100) return `"${arg.substring(0, 50).replace(/\n/g, '\\n')}..." (len ${arg.length}, shader?)`;
             if (arg.length > 50) return `"${arg.substring(0, 47)}..." (len ${arg.length})`;
        }
        return String(arg);
    }
    const glPrototype = Object.getPrototypeOf(gl);
    const functionsToInstrument = traceFunctions && traceFunctions.length > 0 ? traceFunctions
        : Object.getOwnPropertyNames(glPrototype).filter(name => typeof glPrototype[name] === 'function' && !name.startsWith('_'));
    let instrumentedCount = 0;
    functionsToInstrument.forEach(functionName => {
        if (typeof glPrototype[functionName] !== 'function') return;
        const originalFunction = gl[functionName];
        gl[functionName] = function(...args) {
            const prefix = `[${contextName} CALL] ${functionName}`;
            if (logArgs) { const formattedArgs = args.map(formatArg); console.log(`${prefix}(${formattedArgs.join(', ')})`); }
            else { console.log(prefix); }
            const result = originalFunction.apply(gl, args);
            if (functionName === 'getError' && logGetError) {
                 if (result !== gl.NO_ERROR) { console.error(` L> [${contextName} GET_ERROR_RESULT] -> ${glConstants[result] || result} (${result})`); }
                 else if (logResult) { console.log(` L> [${contextName} RSP] ${functionName} -> ${glConstants[result] || 'NO_ERROR (0)'}`); }
            } else if (logResult) { console.log(` L> [${contextName} RSP] ${functionName} ->`, formatArg(result)); }
            if (checkErrorAfterEachCall && functionName !== 'getError') {
                const error = gl.getError();
                if (error !== gl.NO_ERROR) { console.error(`[${contextName} ERROR] After ${functionName}: ${glConstants[error] || error} (${error})`);}
            }
            return result;
        };
        instrumentedCount++;
    });
    console.log(`%c[${contextName}] Context wrapped. ${instrumentedCount} functions instrumented. Opts: logArgs=${logArgs}, logRes=${logResult}, maxArr=${maxArrayLength}, chkErr=${checkErrorAfterEachCall}`, "color: #2E8B57; font-weight: bold;");
    if (traceFunctions?.length > 0) console.log(`%c[${contextName}] Tracing: ${traceFunctions.join(', ')}`, "color: #2E8B57;");
    else console.log(`%c[${contextName}] Tracing all discoverable prototype functions.`, "color: #2E8B57;");
}


/*==============================================================================
  PiP GLOBAL Z-INDEX MANAGEMENT & UTILITIES
==============================================================================*/
let highestZIndex = 10000;
const DEFAULT_FALLBACK_ZINDEX = 10000;

function updateHighestZIndex() {
  let max = 0;
  document.querySelectorAll('.fresh-pip').forEach((el) => {
    let computedZStr = window.getComputedStyle(el).zIndex;
    let z = (computedZStr === "auto" || computedZStr === "")
      ? (parseInt(el.style.zIndex, 10) || DEFAULT_FALLBACK_ZINDEX)
      : (parseInt(computedZStr, 10) || 0);
    if (z > max) max = z;
  });
  if (max < DEFAULT_FALLBACK_ZINDEX) max = DEFAULT_FALLBACK_ZINDEX;
  highestZIndex = max;
  return highestZIndex;
}

function bringToFront(container, fallback = 0) {
  updateHighestZIndex();
  if (fallback && highestZIndex < fallback) highestZIndex = fallback;
  highestZIndex++;
  container.style.setProperty("z-index", highestZIndex, "important");
}

/*==============================================================================
  FreshPip COMPONENT
==============================================================================*/
const PIP_HEADER_HEIGHT_NUM = 55; 
const PIP_HEADER_HEIGHT = `${PIP_HEADER_HEIGHT_NUM}px`;
const DEFAULT_PIP_WIDTH = "400px";
const DEFAULT_PIP_HEIGHT = "300px";
const DEFAULT_PIP_TOP = `calc(50vh - 150px)`;
const DEFAULT_PIP_LEFT = "50px";
const DEFAULT_PIP_BORDER_RADIUS = "8px";

function FreshPip({
  onClose,
  pipId,
  filePath,
  header, 
  functionName,
  component,
  componentProps = {},
  initialStyle = {},
  isDraggable = true,
}) {
  const containerRef = useRef(null);
  const headerRef = useRef(null);
  const [LoadedComponent, setLoadedComponent] = useState(() => component || null);
  const [isBeingDraggedInternal, setIsBeingDraggedInternal] = useState(false);
  const originalStylesRef = useRef({});
  const dragStartDataRef = useRef({});

  const defaultPipBaseStyleRef = useRef({
    position: "fixed", backgroundColor: "#222222", border: "2px solid #444444",
    boxSizing: "border-box", padding: "0px", overflow: "visible",
    zIndex: DEFAULT_FALLBACK_ZINDEX, display: 'flex', flexDirection: 'column',
    transition: 'width 0.2s ease-out, height 0.2s ease-out, border-radius 0.2s ease-out, box-shadow 0.2s ease-out, top 0.2s ease-out, left 0.2s ease-out, right 0.2s ease-out',
  });

  useEffect(() => {
    if (component) {
      setLoadedComponent(() => component);
      return;
    }
    if (!filePath || !header || !functionName) {
      let missing = [];
      if (!filePath) missing.push("filePath");
      if (!header) missing.push("header (for dc.headerLink)");
      if (!functionName) missing.push("functionName");
      console.warn(`[FreshPip ${pipId}] Missing required props for dynamic loading: ${missing.join(', ')}.`);
      setLoadedComponent(() => () => preactH('div',{style:{color:'#777',padding:'20px', textAlign:'center'}}, `(Invalid Config: Missing ${missing.join(', ')})`));
      return;
    }

    let isMounted = true;
    (async () => {
      try {
        console.log(`[FreshPip ${pipId}] Attempting to load: ${functionName} from ${filePath} with header section: ${header}`);
        const componentModuleLink = dc.headerLink(filePath, header);
        const dynamicModule = await dc.require(componentModuleLink);

        if (!isMounted) return;
        const Comp = dynamicModule[functionName];
        if (Comp) {
          setLoadedComponent(() => Comp);
          console.log(`[FreshPip ${pipId}] Component '${functionName}' loaded successfully.`);
        } else {
          console.error(`[FreshPip ${pipId}] Component '${functionName}' not found in module from ${filePath} (header: ${header}). Available:`, Object.keys(dynamicModule));
          setLoadedComponent(() => () => preactH('div',{style:{color:'red',padding:'20px'}},`Error: Component ${functionName} not found.`));
        }
      } catch (error) {
        if (!isMounted) return;
        console.error(`[FreshPip ${pipId}] Error loading component '${functionName}' from ${filePath} (header: ${header}):`, error);
        setLoadedComponent(() => () => preactH('div',{style:{color:'red',padding:'20px'}},`Error loading: ${error.message}`));
      }
    })();
    return () => { isMounted = false; };
  }, [component, filePath, header, functionName, pipId]);

  useEffect(() => {
    const container = containerRef.current;
    if (container) {
      bringToFront(container, DEFAULT_FALLBACK_ZINDEX);
      requestAnimationFrame(() => {
          if (containerRef.current && Object.keys(originalStylesRef.current).length === 0) {
                const computed = getComputedStyle(containerRef.current);
                originalStylesRef.current = {
                    width: computed.width, height: computed.height,
                    top: computed.top, left: computed.left,
                    right: computed.right, borderRadius: computed.borderRadius,
                };
            }
      });
      const handlePointerDownBringToFront = (e) => {
         if (!e.target.closest('button')) { bringToFront(container); }
      };
      container.addEventListener("pointerdown", handlePointerDownBringToFront, true);
      return () => { container.removeEventListener("pointerdown", handlePointerDownBringToFront, true); };
    }
  }, [pipId]);

  useEffect(() => {
    if (!isDraggable || !preactH) return;
    const container = containerRef.current;
    const dragHeaderEl = headerRef.current;
    if (!container || !dragHeaderEl) return;
    const handleMouseDownOnHeader = (e) => {
      if (e.target.closest('button')) return; e.preventDefault();
      bringToFront(container); setIsBeingDraggedInternal(true);
      const computedContainerStyle = getComputedStyle(container);
      const rect = container.getBoundingClientRect();
      dragStartDataRef.current = {
        initialCursorX: e.clientX, initialCursorY: e.clientY,
        clickOffsetXOnPip: e.clientX - rect.left, clickOffsetYOnPip: e.clientY - rect.top,
        initialTop: parseFloat(computedContainerStyle.top), initialLeft: parseFloat(computedContainerStyle.left),
        initialRight: parseFloat(computedContainerStyle.right),
        isRightAnchored: computedContainerStyle.left === 'auto' && computedContainerStyle.right !== 'auto',
      };
      container.style.transition = 'none';
    };
    const handleMouseMove = (e) => {
      if (!isBeingDraggedInternal) return;
      const { clickOffsetXOnPip, clickOffsetYOnPip, isRightAnchored } = dragStartDataRef.current;
      const newTop = e.clientY - clickOffsetYOnPip;
      if (isRightAnchored) {
        const currentWidth = parseFloat(container.style.width || originalStylesRef.current?.width || DEFAULT_PIP_WIDTH);
        const newRight = window.innerWidth - e.clientX - (currentWidth - clickOffsetXOnPip);
        container.style.left = 'auto'; container.style.right = `${newRight}px`;
      } else {
        const newLeft = e.clientX - clickOffsetXOnPip;
        container.style.left = `${newLeft}px`; container.style.right = 'auto';
      }
      container.style.top = `${newTop}px`;
    };
    const handleMouseUp = () => {
      if (!isBeingDraggedInternal) return;
      setIsBeingDraggedInternal(false);
      container.style.transition = defaultPipBaseStyleRef.current.transition;
       if (containerRef.current) {
            const newComputed = getComputedStyle(containerRef.current);
            originalStylesRef.current = { ...originalStylesRef.current, top: newComputed.top, left: newComputed.left, right: newComputed.right };
        }
    };
    dragHeaderEl.addEventListener("mousedown", handleMouseDownOnHeader);
    window.addEventListener("mousemove", handleMouseMove);
    window.addEventListener("mouseup", handleMouseUp);
    return () => {
      dragHeaderEl.removeEventListener("mousedown", handleMouseDownOnHeader);
      window.removeEventListener("mousemove", handleMouseMove);
      window.removeEventListener("mouseup", handleMouseUp);
    };
  }, [isDraggable, pipId, defaultPipBaseStyleRef]);

  let currentPipStyle = { ...defaultPipBaseStyleRef.current };
  currentPipStyle.width = initialStyle.width || DEFAULT_PIP_WIDTH;
  currentPipStyle.height = initialStyle.height || DEFAULT_PIP_HEIGHT;
  currentPipStyle.top = initialStyle.top || (originalStylesRef.current.top || DEFAULT_PIP_TOP);
  currentPipStyle.left = initialStyle.left || (originalStylesRef.current.left !== 'auto' ? originalStylesRef.current.left : DEFAULT_PIP_LEFT);
  currentPipStyle.right = initialStyle.right || (originalStylesRef.current.right !== 'auto' ? originalStylesRef.current.right : 'auto');
  currentPipStyle.borderRadius = initialStyle.borderRadius || DEFAULT_PIP_BORDER_RADIUS;
  currentPipStyle = { ...currentPipStyle, ...initialStyle }; 
   if (isBeingDraggedInternal) { currentPipStyle.transition = 'none'; }
   else if (originalStylesRef.current.top && !initialStyle.top) { 
        currentPipStyle.top = originalStylesRef.current.top;
        currentPipStyle.left = originalStylesRef.current.left;
        currentPipStyle.right = originalStylesRef.current.right;
   }
  const headerBarStyle = {
    height: PIP_HEADER_HEIGHT, width: '100%', backgroundColor: "#333333", display: "flex",
    alignItems: "center", justifyContent: "space-between", padding: "0 10px",
    cursor: isDraggable ? "move" : "default", flexShrink: 0, userSelect: 'none',
    borderBottom: '1px solid #444444', boxSizing: 'border-box',
  };
  const pipContentStyle = {
    flexGrow: 1, overflow: "auto", display: "flex", flexDirection: 'column',
    width: '100%', height: '100%', padding: "10px", boxSizing: 'border-box',
    backgroundColor: '#1e1e1e', color: '#f0f0f0',
  };
  const pipTitle = componentProps.titleText || functionName || "PiP Window";

  if (!preactH) return null;

  return (
    preactH("div", { ref: containerRef, className: "fresh-pip", style: currentPipStyle },
      preactH("div", { ref: headerRef, className: "fresh-pip-header", style: headerBarStyle },
        preactH("span", { style: { color: "#ccc", fontSize: "14px", overflow: "hidden", textOverflow: "ellipsis", whiteSpace: "nowrap", marginRight: '10px'} }, pipTitle),
        preactH("button", { style: { cursor: "pointer", background: "#555", border: "1px solid #777", color: "white", fontSize: "14px", borderRadius: "3px", padding: "2px 8px", lineHeight: '1' },
            onClick: (e) => { e.stopPropagation(); if (onClose) onClose(); } }, "✕")
      ),
      preactH("div", { className: "fresh-pip-content", style: pipContentStyle },
        LoadedComponent
          ? preactH(LoadedComponent, { ...componentProps })
          : preactH("div", { style: { width: "100%", height: "100%", display: "flex", alignItems: "center", justifyContent: "center", color: "#888", userSelect: "none", fontSize: '12px' } }, "Loading Content...")
      )
    )
  );
}
// --- End FreshPip Component ---


// Constants for the main scene's representation in the panel
const MAIN_SCENE_PANEL_ID = "worldview-main-scene-context"; // Unique key for list item
const MAIN_SCENE_PANEL_TITLE = "Main 3D World";

function WorldView() {
  const canvasRef = useRef(null);
  const [engine, setEngine] = useState(null);
  const [scene, setScene] = useState(null);
  const [refreshKey, setRefreshKey] = useState(0);
  const pipHostDivsRef = useRef(new Map());
  const enigmaPipSpawnCountRef = useRef(0);
  const [activePipsInfo, setActivePipsInfo] = useState([]); // ONLY for dynamic, closable PiPs

  const closePip = useCallback((pipIdToClose) => {
    if (!preactRender) return;
    const hostDiv = pipHostDivsRef.current.get(pipIdToClose);
    if (hostDiv) {
      preactRender(null, hostDiv);
      if (hostDiv.parentNode) hostDiv.parentNode.removeChild(hostDiv);
      pipHostDivsRef.current.delete(pipIdToClose);
      setActivePipsInfo(prevPips => prevPips.filter(p => p.id !== pipIdToClose));
      console.log(`WorldView PiP: Closed and removed PiP: ${pipIdToClose}`);
    }
  }, []); 

  const spawnPip = useCallback((pipConfig) => {
    if (!preactH || !preactRender) {
        console.error("WorldView PiP: Cannot spawn PiP, Preact not available."); return;
    }
    const {
        pipId: providedPipId,
        filePath,
        header, 
        functionName,
        componentProps = {}, initialStyle = {}, isDraggable = true,
    } = pipConfig;

    if (!filePath || !header || !functionName) { 
        console.error("WorldView PiP: spawnPip requires filePath, header, and functionName.", pipConfig);
        return;
    }

    const pipIdToUse = providedPipId || `pip-${Date.now()}-${Math.random().toString(36).substring(2, 7)}`;

    if (providedPipId && pipHostDivsRef.current.has(providedPipId)) {
        const existingHost = pipHostDivsRef.current.get(providedPipId);
        if (document.body.contains(existingHost)) {
            bringToFront(existingHost);
            console.log(`WorldView PiP: Brought existing PiP to front: ${providedPipId}`);
            return;
        } else { 
            pipHostDivsRef.current.delete(providedPipId); 
            setActivePipsInfo(prevPips => prevPips.filter(p => p.id !== providedPipId));
        }
    }

    const hostDiv = document.createElement("div");
    hostDiv.id = `pip-host-wrapper-${pipIdToUse}`;
    document.body.appendChild(hostDiv);
    pipHostDivsRef.current.set(pipIdToUse, hostDiv);

    const pipElementProps = {
      key: pipIdToUse,
      pipId: pipIdToUse, 
      filePath, header, functionName,
      componentProps, initialStyle, isDraggable,
      onClose: () => closePip(pipIdToUse),
    };
    preactRender(preactH(FreshPip, pipElementProps), hostDiv);

    const title = componentProps.titleText || functionName || "Unnamed PiP";
    setActivePipsInfo(prevPips => {
        if (prevPips.find(p => p.id === pipIdToUse)) return prevPips;
        return [...prevPips, { id: pipIdToUse, title }];
    });

    console.log(`WorldView PiP: Spawned PiP: ${pipIdToUse} for ${functionName} from ${filePath} (header: ${header})`);
  }, [closePip]); 

  const loadScript = (src) => {
    return new Promise((resolve, reject) => {
      const existingScript = document.querySelector(`script[src="${src}"]`);
      if (existingScript) {
        if (existingScript.dataset.loaded === "true") { resolve(existingScript); }
        else {
            let attempts = 0; const FAILED_TIMEOUT = 5000;
            const interval = setInterval(() => {
                attempts++;
                if (existingScript.dataset.loaded === "true") { clearInterval(interval); resolve(existingScript); }
                else if (attempts * 100 > FAILED_TIMEOUT) { clearInterval(interval); console.warn(`Timed out for ${src}`); reject(new Error(`Timed out ${src}`));}
            }, 100);
        }
        return;
      }
      const script = document.createElement("script"); script.src = src; script.async = true;
      script.onload = () => { script.dataset.loaded = "true"; console.log(`Script loaded: ${src}`); resolve(script); };
      script.onerror = (e) => { console.error(`Error script: ${src}`, e); reject(new Error(`Failed script: ${src}`));};
      document.body.appendChild(script);
    });
  };
  const initBabylon = async () => {
    if (!canvasRef.current || !window.BABYLON || !window.BABYLON.SceneLoader) {
      console.error("WorldView State: initBabylon - Canvas ref missing or Babylon.js not loaded. Skipping.");
      return () => { console.log("WorldView State: Babylon init failed, no cleanup."); };
    }
    console.log("WorldView State: Initializing Babylon.js engine and scene...");
    const babylonEngine = new window.BABYLON.Engine(canvasRef.current, true, { preserveDrawingBuffer: true, stencil: true });

    if (babylonEngine._gl && typeof wrapWebGLContext === 'function') {
        console.log("WorldView State: Enabling WebGL context logging with a high-level filter.");
        wrapWebGLContext(babylonEngine._gl, {
            logArgs: true, logResult: false, contextName: 'BabylonGL-Setup', maxArrayLength: 8,
            checkErrorAfterEachCall: false, logGetError: true,
            traceFunctions: [
                'createShader', 'shaderSource', 'compileShader', 'getShaderParameter', 'getShaderInfoLog',
                'createProgram', 'attachShader', 'linkProgram', 'getProgramParameter', 'getProgramInfoLog',
                'useProgram', 'deleteShader', 'deleteProgram', 'texImage2D', 'texSubImage2D',
                'generateMipmap', 'pixelStorei', 'bindTexture', 'activeTexture', 'createBuffer',
                'bindBuffer', 'bufferData', 'getError'
            ],
        });
    }

    const babylonScene = new window.BABYLON.Scene(babylonEngine);
    console.log("WorldView State: Babylon.js Scene created.");
    setEngine(babylonEngine); setScene(babylonScene); // This will trigger panel update via `engine` state
    const camera = new window.BABYLON.ArcRotateCamera("Camera", -Math.PI / 2, Math.PI / 2.5, 10, window.BABYLON.Vector3.Zero(), babylonScene);
    camera.attachControl(canvasRef.current, true); camera.minZ = 0.1;
    console.log("WorldView State: Camera created and controls attached.");
    const rotationSpeed = 0.008;
    babylonScene.clearColor = new window.BABYLON.Color4(0, 0, 0, 1);
    babylonScene.createDefaultEnvironment({ createSkybox: false, enableGround: false, createGround: false, environmentTexture: "https://assets.babylonjs.com/environments/studio.env", intensity: 1.2 });
    console.log("WorldView State: Default environment created.");
    const directionalLight = new window.BABYLON.DirectionalLight("directionalLight", new window.BABYLON.Vector3(0.5, -1, 0.5), babylonScene);
    directionalLight.intensity = 1.5; directionalLight.diffuse = new window.BABYLON.Color3(1.0, 0.95, 0.9);
    console.log("WorldView State: Directional light created.");

    const modelPath = "_setup/_babylon/b26.card.888.glb";
    try {
      console.log(`WorldView State: Attempting to load GLB model from: ${modelPath}`);
      const assetUrl = dc.app.vault.adapter.getResourcePath(modelPath);
      const result = await window.BABYLON.SceneLoader.ImportMeshAsync(null, "", assetUrl, babylonScene);
      if (result.meshes && result.meshes.length > 0) {
        let mainModelMesh = result.meshes.find(m => m.getTotalVertices() > 0 && m.name !== "__root__") || result.meshes[0];
        mainModelMesh.position = window.BABYLON.Vector3.Zero();
        mainModelMesh.scaling = new window.BABYLON.Vector3(2.5, 3.5, 3.5);
        console.log(`WorldView State: GLB model loaded: ${mainModelMesh.name}.`);
        const boundingInfo = mainModelMesh.getBoundingInfo();
        if (boundingInfo) {
            camera.setTarget(boundingInfo.boundingSphere.center);
            camera.radius = boundingInfo.boundingSphere.radius * 7.7;
        }
      } else { console.warn("WorldView State: GLB loaded, but no meshes found."); }
    } catch (error) { console.error("WorldView State: Error loading GLB model:", error); }

    babylonEngine.runRenderLoop(() => { if (babylonScene.activeCamera && babylonScene.isReady()) { camera.alpha += rotationSpeed; babylonScene.render(); }});
    const onResize = () => { babylonEngine.resize(); }
    window.addEventListener("resize", onResize);
    const canvas = canvasRef.current;
    const handleWheel = (e) => e.preventDefault();
    if (canvas) canvas.addEventListener("wheel", handleWheel, { passive: false });

    return () => {
      window.removeEventListener("resize", onResize);
      babylonEngine.stopRenderLoop();
      if (babylonScene) babylonScene.dispose();
      if (babylonEngine) babylonEngine.dispose();
      setEngine(null); setScene(null); // This will trigger panel update via `engine` state
      if (canvas) canvas.removeEventListener("wheel", handleWheel);
    };
   };

  useEffect(() => {
    console.log("WorldView State: useEffect triggered. Key:", refreshKey);
    let cleanupBabylon = () => {}; const loadedScripts = [];
    const setupEnvironment = async () => {
      try {
        if (!window.BABYLON || !window.BABYLON.SceneLoader) {
            loadedScripts.push(await loadScript("https://cdn.babylonjs.com/babylon.js"));
            loadedScripts.push(await loadScript("https://cdn.babylonjs.com/loaders/babylonjs.loaders.min.js"));
        }
        if (window.BABYLON && window.BABYLON.SceneLoader) { cleanupBabylon = await initBabylon(); }
        else { console.error("WorldView State: Babylon.js NOT available."); }
      } catch (error) { console.error("WorldView State: Failed to load/init Babylon.js:", error); }
    };
    setupEnvironment();
    return () => {
      console.log("WorldView State: useEffect cleanup. Key:", refreshKey);
      if (typeof cleanupBabylon === 'function') cleanupBabylon(); // This will set engine/scene to null
      pipHostDivsRef.current.forEach((hostDiv, pipId) => closePip(pipId)); 
      pipHostDivsRef.current.clear(); 
      enigmaPipSpawnCountRef.current = 0; 
      setActivePipsInfo([]); // Clear dynamic PiPs
      loadedScripts.forEach(script => { if (script?.parentElement) { document.body.removeChild(script); }});
    };
  }, [refreshKey, closePip]);

  return (
    <div style={{ position: "relative", width: "100%", height: "500px", overflow: "hidden", background: "#121212" }}>
      <canvas ref={canvasRef} style={{ width: "100%", height: "100%", display: "block" }} />
      <style>{`
        .refresh-button { background-color: #333; transition: background-color 0.3s ease, transform 0.1s ease; box-sizing: border-box; }
        .refresh-button:hover { background-color: #6A0DAD; transform: scale(1.05); }
        .refresh-button:active { transform: scale(0.95); }
        .pip-spawn-button { background-color: #007bff; color: white; border:none; padding: 8px 15px; margin: 5px; border-radius: 4px; cursor: pointer; }
        .pip-spawn-button:hover { background-color: #0056b3; }
        .active-pips-panel {
          position: absolute;
          bottom: 10px;
          left: 10px;
          width: 230px; 
          max-height: 150px; 
          background-color: rgba(30, 30, 30, 0.85);
          color: #e0e0e0;
          border-radius: 6px;
          border: 1px solid #444;
          padding: 10px;
          z-index: 15000; 
          font-family: sans-serif;
          font-size: 13px;
          overflow-y: auto;
          box-shadow: 0 2px 8px rgba(0,0,0,0.3);
        }
        .active-pips-panel h4 {
          margin-top: 0;
          margin-bottom: 8px;
          font-size: 14px;
          color: #00aaff;
          border-bottom: 1px solid #555;
          padding-bottom: 5px;
        }
        .active-pips-panel ul {
          list-style: none;
          padding: 0;
          margin: 0;
        }
        .active-pips-panel li {
          display: flex; 
          justify-content: space-between; 
          align-items: center; 
          margin-bottom: 6px;
          padding: 3px 0;
        }
        .active-pips-panel li .pip-info { 
            white-space: nowrap;
            overflow: hidden;
            text-overflow: ellipsis;
            flex-grow: 1; 
            margin-right: 5px; 
        }
         .active-pips-panel li .pip-id-snippet {
          font-size: 10px;
          color: #999;
          margin-left: 5px;
        }
        .active-pips-panel .no-pips {
          color: #888;
          font-style: italic;
        }
        .panel-pip-close-button {
            background: none;
            border: none;
            color: #ff6b6b; 
            cursor: pointer;
            font-size: 16px; 
            padding: 0 5px;
            line-height: 1;
            font-weight: bold;
        }
        .panel-pip-close-button:hover {
            color: #ff4757; 
        }
      `}</style>
      <button onClick={() => {
          setRefreshKey(prev => prev + 1);
        }} className="refresh-button"
        style={{ position: "absolute", top: "10px", right: "10px", zIndex: 20000, width: "44px", height: "44px",
                  borderRadius: "50%", border: "none", display: "flex", justifyContent: "center", alignItems: "center",
                  cursor: "pointer", boxShadow: "0px 4px 10px rgba(0,0,0,0.4)", color: "white", outline: "none" }}
        aria-label="Refresh Scene" title="Refresh Scene" >
        <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="currentColor">
          <path d="M12 4V1L8 5l4 4V6c3.31 0 6 2.69 6 6s-2.69 6-6 6-6-2.69-6-6H4c0 4.42 3.58 8 8 8s8-3.58 8-8-3.58-8-8-8z"/>
        </svg>
      </button>

      <div style={{position: "absolute", top: "10px", left: "10px", zIndex: 20000}}>
        <button
          className="pip-spawn-button"
          onClick={() => {
            enigmaPipSpawnCountRef.current += 1;
            const spawnOrder = enigmaPipSpawnCountRef.current;
            const offset = (spawnOrder -1) * 25; 

            spawnPip({
              filePath: "EnigmaViewer.component.v3.md",
              header: "ViewComponent", 
              functionName: "EnigmaView",
              componentProps: {
                  modelPath: "_setup/_babylon/minigame/b26.card.888.glb",
                  initialUrl: "https://www.youtube.com/embed/ePF_GYs4RrM",
                  titleText: `Enigma #${spawnOrder}`, 
                  descriptionText: "A test portal..."
              },
              initialStyle: { 
                width: "250px",
                height: "150px",
                top: `${10 + offset}px`,     
                right: `${70 + offset}px`,   
                left: "auto"
              }
            });
          }}
          title="Spawn EnigmaView PiP"
        >
          Spawn EnigmaView
        </button>
      </div>

      {/* Panel for Active Contexts */}
      <div className="active-pips-panel">
        <h4>Active Contexts</h4>
        {(!engine || !scene) && activePipsInfo.length === 0 ? (
          <p className="no-pips">No contexts currently active.</p>
        ) : (
          <ul>
            {engine && scene && ( /* Display Main Scene if active */
              <li key={MAIN_SCENE_PANEL_ID}>
                <span className="pip-info" title="Main rendering context">
                  {MAIN_SCENE_PANEL_TITLE}
                </span>
                {/* No close button for the main scene */}
              </li>
            )}
            {activePipsInfo.map(pip => ( /* Display dynamic PiPs */
              <li key={pip.id}>
                <span className="pip-info" title={`ID: ${pip.id}`}>
                  {pip.title}
                  <span className="pip-id-snippet">(ID: {pip.id.substring(4, 11)}...)</span>
                </span>
                <button
                  className="panel-pip-close-button"
                  onClick={() => closePip(pip.id)}
                  title={`Close ${pip.title}`}
                  aria-label={`Close ${pip.title}`}
                >
                  ✕
                </button>
              </li>
            ))}
          </ul>
        )}
      </div>
    </div>
  );
}

return { WorldView };
```