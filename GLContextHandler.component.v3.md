

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

// --- WebGL Context Logger Utility ---
function wrapWebGLContext(gl, options = {}) {
    // ... (WebGL Context Logger Utility - unchanged, kept for completeness)
    const defaultOptions = {
        logArgs: false, // Default to false to reduce noise
        logResult: false, // Default to false
        contextName: 'WebGL',
        maxArrayLength: 8, // Reduced from 16 for less verbose array logs if enabled
        traceFunctions: null,
        logGetError: true, // Keep this true, it's very useful
        checkErrorAfterEachCall: false, // Set to false for less noise, true for deep debugging
    };
    const finalOptions = { ...defaultOptions, ...options };

    const {
        logArgs, logResult, contextName, maxArrayLength,
        traceFunctions, logGetError, checkErrorAfterEachCall
    } = finalOptions;

    if (!gl || typeof gl.getExtension !== 'function') {
        console.error(`[${contextName}] Invalid WebGL context provided. Aborting wrap.`, gl);
        return gl;
    }

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
        else if (window.WebGLVertexArrayObjectOES) { prototypes.push({ proto: window.WebGLVertexArrayObjectOES, name: "WebGLVertexArrayObjectOES" }); }

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
    if (!glPrototype) {
        console.error(`[${contextName}] Could not get prototype of WebGL context. Aborting wrap.`, gl);
        return gl;
    }

    let functionsToInstrumentList = [];
    if (traceFunctions && traceFunctions.length > 0) {
        functionsToInstrumentList = traceFunctions.filter(name => typeof gl[name] === 'function');
    } else {
        for (const name in gl) {
            if (typeof gl[name] === 'function' && Object.prototype.hasOwnProperty.call(glPrototype, name) && !name.startsWith('_')) {
                 functionsToInstrumentList.push(name);
            }
        }
    }
    functionsToInstrumentList = [...new Set(functionsToInstrumentList)];

    let instrumentedCount = 0;
    functionsToInstrumentList.forEach(functionName => {
        if (typeof gl[functionName] !== 'function') { return; }
        const originalFunction = gl[functionName];
        if (originalFunction._isWrappedByLogger === contextName) { return; }

        gl[functionName] = function(...args) {
            if (logArgs || logResult) { 
                const prefix = `[${contextName} CALL] ${functionName}`;
                if (logArgs) { const formattedArgs = args.map(formatArg); console.log(`${prefix}(${formattedArgs.join(', ')})`); }
                else { console.log(prefix); } 
            }
            const result = originalFunction.apply(gl, args);
            if (functionName === 'getError' && logGetError) {
                 if (result !== gl.NO_ERROR) { console.error(` L> [${contextName} GET_ERROR_RESULT] -> ${glConstants[result] || result} (${result})`); }
                 else if (logResult) { console.log(` L> [${contextName} RSP (NoError)] ${functionName} -> ${glConstants[result] || 'NO_ERROR (0)'}`); }
            } else if (logResult) { console.log(` L> [${contextName} RSP] ${functionName} ->`, formatArg(result)); }
            if (checkErrorAfterEachCall && functionName !== 'getError') {
                const error = gl.getError();
                if (error !== gl.NO_ERROR) { console.error(`[${contextName} ERROR] After ${functionName}: ${glConstants[error] || error} (${error})`);}
            }
            return result;
        };
        gl[functionName]._isWrappedByLogger = contextName;
        instrumentedCount++;
    });
    console.log(`%c[${contextName}] Context wrapped. ${instrumentedCount} functions instrumented. Opts: logArgs=${logArgs}, logRes=${logResult}, chkErr=${checkErrorAfterEachCall}`, "color: #2E8B57; font-weight: bold;");
    return gl;
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
// ... (FreshPip Component - unchanged, kept for completeness)
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
    return () => {
        // console.log(`[FreshPip ${pipId}] Unmounting.`);
    };
  }, [pipId]);

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
        const componentModuleLink = dc.headerLink(filePath, header);
        const dynamicModule = await dc.require(componentModuleLink);

        if (!isMounted) return;
        const Comp = dynamicModule[functionName];
        if (Comp) {
          setLoadedComponent(() => Comp);
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


const MAIN_SCENE_PANEL_ID = "worldview-main-scene-context";
const MAIN_SCENE_PANEL_TITLE = "Main 3D World";

const worldViewInstanceIdCounter = { count: 0 };

function WorldView() {
  const canvasRef = useRef(null);
  const [engine, setEngine] = useState(null);
  const [scene, setScene] = useState(null);
  const [refreshKey, setRefreshKey] = useState(0);
  const pipHostDivsRef = useRef(new Map());
  const spawnedPipCountRef = useRef(0);
  const [activePipsInfo, setActivePipsInfo] = useState([]);
  const instanceIdRef = useRef(`WorldView-${worldViewInstanceIdCounter.count++}`);

  useEffect(() => {
    const C_myInstanceId = instanceIdRef.current;
    console.log(`%c${C_myInstanceId}: Initializing instance.`, "color: green");
    return () => {
      console.log(`%c${C_myInstanceId}: Destroying instance.`, "color: red");
    };
  }, []); // instanceIdRef.current is stable after first assignment

  const closePip = useCallback((pipIdToClose) => {
    const C_myInstanceId = instanceIdRef.current;
    if (!preactRender) return;
    const hostDiv = pipHostDivsRef.current.get(pipIdToClose);
    if (hostDiv) {
      // console.log(`${C_myInstanceId}: Closing PiP ${pipIdToClose}.`);
      preactRender(null, hostDiv); // Unmount Preact component
      if (hostDiv.parentNode) hostDiv.parentNode.removeChild(hostDiv); // Remove host div from DOM
      pipHostDivsRef.current.delete(pipIdToClose);
      setActivePipsInfo(prevPips => prevPips.filter(p => p.id !== pipIdToClose));
    }
  }, [instanceIdRef]); // Removed setActivePipsInfo, pipHostDivsRef as they are stable (state setter/ref)

  const spawnPip = useCallback((pipConfig) => {
    const C_myInstanceId = instanceIdRef.current;
    if (!preactH || !preactRender) {
        console.error(`${C_myInstanceId}: Cannot spawn PiP, Preact not available.`); return;
    }
    const {
        pipId: providedPipId,
        filePath,
        header,
        functionName,
        componentProps = {},
        initialStyle = {},
        isDraggable = true,
    } = pipConfig;

    if (!filePath || !header || !functionName) {
        console.error(`${C_myInstanceId}: spawnPip requires filePath, header, and functionName.`, pipConfig);
        return;
    }
    const pipIdToUse = providedPipId || `pip-${Date.now()}-${Math.random().toString(36).substring(2, 7)}`;

    if (providedPipId && pipHostDivsRef.current.has(providedPipId)) {
        const existingHost = pipHostDivsRef.current.get(providedPipId);
        if (document.body.contains(existingHost)) {
            bringToFront(existingHost);
            return;
        } else { // Host div exists in ref but not in DOM, clean up
            pipHostDivsRef.current.delete(providedPipId);
            setActivePipsInfo(prevPips => prevPips.filter(p => p.id !== providedPipId));
        }
    }

    const hostDiv = document.createElement("div");
    hostDiv.id = `pip-host-wrapper-${pipIdToUse}`;
    document.body.appendChild(hostDiv);
    pipHostDivsRef.current.set(pipIdToUse, hostDiv);

    const freshPipDirectProps = {
        key: pipIdToUse,
        pipId: pipIdToUse,
        filePath, header, functionName,
        componentProps,
        initialStyle, isDraggable,
        onClose: () => closePip(pipIdToUse),
    };
    preactRender(preactH(FreshPip, freshPipDirectProps), hostDiv);

    const titleForPanel = componentProps.titleText || functionName || "PiP Window";
    setActivePipsInfo(prevPips => {
        if (prevPips.find(p => p.id === pipIdToUse)) return prevPips; // Should not happen with check above
        return [...prevPips, { id: pipIdToUse, title: titleForPanel }];
    });
    // console.log(`${C_myInstanceId}: Spawned PiP: ${pipIdToUse} ('${titleForPanel}')`);
  }, [closePip, instanceIdRef]);

  const loadScript = useCallback((src) => {
    return new Promise((resolve, reject) => {
      const C_myInstanceId = instanceIdRef.current;
      const existingScript = document.querySelector(`script[src="${src}"]`);
      if (existingScript) {
        if (existingScript.dataset.loaded === "true") { resolve(existingScript); }
        else {
            let attempts = 0; const FAILED_TIMEOUT = 5000;
            const interval = setInterval(() => {
                attempts++;
                if (existingScript.dataset.loaded === "true") { clearInterval(interval); resolve(existingScript); }
                else if (attempts * 100 > FAILED_TIMEOUT) { clearInterval(interval); console.warn(`${C_myInstanceId}: Timed out waiting for existing script ${src}`); reject(new Error(`Timed out ${src}`));}
            }, 100);
        }
        return;
      }
      const script = document.createElement("script"); script.src = src; script.async = true;
      script.onload = () => { script.dataset.loaded = "true"; /*console.log(`${C_myInstanceId}: Script loaded: ${src}`);*/ resolve(script); };
      script.onerror = (e) => { console.error(`${C_myInstanceId}: Error loading script: ${src}`, e); reject(new Error(`Failed to load script: ${src}`));};
      document.body.appendChild(script);
    });
  }, [instanceIdRef]);

  const initBabylon = useCallback(async () => {
    const C_myInstanceId = instanceIdRef.current;
    if (!canvasRef.current) {
        console.error(`${C_myInstanceId}: Canvas ref missing. Skipping Babylon init.`);
        return () => { /* console.log(`${C_myInstanceId}: Babylon init skipped (no canvas)...`); */ };
    }
    if (!window.BABYLON || !window.BABYLON.SceneLoader) {
      console.error(`${C_myInstanceId}: Babylon.js core or loaders not loaded. Skipping Babylon init.`);
      return () => { /* console.log(`${C_myInstanceId}: Babylon init failed (libs missing)...`); */ };
    }
    // console.log(`${C_myInstanceId}: Initializing Babylon.js engine and scene...`);
    let babylonEngine;
    try {
        babylonEngine = new window.BABYLON.Engine(canvasRef.current, true, { preserveDrawingBuffer: true, stencil: true });
    } catch (e) {
        console.error(`${C_myInstanceId}: Error creating Babylon Engine:`, e);
        return () => { /* console.log(`${C_myInstanceId}: Babylon engine creation failed...`); */ };
    }

    const loggerContextName = `BabylonGL-${C_myInstanceId}`;
    if (babylonEngine._gl && typeof wrapWebGLContext === 'function') {
        try {
            wrapWebGLContext(babylonEngine._gl, { contextName: loggerContextName });
        } catch (wrapError) {
            console.error(`${C_myInstanceId}: Error during wrapWebGLContext for ${loggerContextName}:`, wrapError);
        }
    } else if (!babylonEngine._gl) {
        // console.warn(`${C_myInstanceId}: babylonEngine._gl context not found. Cannot wrap.`);
    }

    const babylonScene = new window.BABYLON.Scene(babylonEngine);
    setEngine(babylonEngine); setScene(babylonScene);

    const camera = new window.BABYLON.ArcRotateCamera("Camera", -Math.PI / 2, Math.PI / 2.5, 10, window.BABYLON.Vector3.Zero(), babylonScene);
    camera.attachControl(canvasRef.current, true); camera.minZ = 0.1;
    const rotationSpeed = 0.008;
    babylonScene.clearColor = new window.BABYLON.Color4(0, 0, 0, 1);
    try {
      babylonScene.createDefaultEnvironment({ createSkybox: false, enableGround: false, createGround: false, environmentTexture: "https://assets.babylonjs.com/environments/studio.env", intensity: 1.2 });
    } catch(e) { console.error(`${C_myInstanceId}: Error creating default environment:`, e); }
    const directionalLight = new window.BABYLON.DirectionalLight("directionalLight", new window.BABYLON.Vector3(0.5, -1, 0.5), babylonScene);
    directionalLight.intensity = 1.5; directionalLight.diffuse = new window.BABYLON.Color3(1.0, 0.95, 0.9);

    const modelPath = "_setup/_babylon/b26.card.888.glb";
    try {
      const assetUrl = dc.app.vault.adapter.getResourcePath(modelPath);
      const result = await window.BABYLON.SceneLoader.ImportMeshAsync(null, "", assetUrl, babylonScene);
      if (result.meshes && result.meshes.length > 0) {
        let mainModelMesh = result.meshes.find(m => m.getTotalVertices() > 0 && m.name !== "__root__") || result.meshes[0];
        mainModelMesh.position = window.BABYLON.Vector3.Zero();
        mainModelMesh.scaling = new window.BABYLON.Vector3(2.5, 3.5, 3.5);
        const boundingInfo = mainModelMesh.getBoundingInfo();
        if (boundingInfo) {
            camera.setTarget(boundingInfo.boundingSphere.center);
            camera.radius = boundingInfo.boundingSphere.radius * 7.7;
        }
      } else { console.warn(`${C_myInstanceId}: GLB loaded, but no meshes found.`); }
    } catch (error) { console.error(`${C_myInstanceId}: Error loading GLB model:`, error); }

    babylonEngine.runRenderLoop(() => { if (babylonScene && babylonScene.activeCamera && babylonScene.isReady() && !babylonEngine.isDisposed) { camera.alpha += rotationSpeed; babylonScene.render(); }});
    const onResize = () => { if (babylonEngine && !babylonEngine.isDisposed) babylonEngine.resize(); }
    window.addEventListener("resize", onResize);
    const currentCanvas = canvasRef.current;
    const handleWheel = (e) => e.preventDefault();
    if (currentCanvas) currentCanvas.addEventListener("wheel", handleWheel, { passive: false });

    return () => {
      const C_myInstanceId_cleanup = instanceIdRef.current;
      console.log(`%c${C_myInstanceId_cleanup}: Disposing Babylon.js engine and scene.`, "color: orange; font-weight: bold;");
      window.removeEventListener("resize", onResize);
      if (currentCanvas) currentCanvas.removeEventListener("wheel", handleWheel);
      if (babylonEngine && !babylonEngine.isDisposed) {
        babylonEngine.stopRenderLoop();
        if (babylonScene && !babylonScene.isDisposed) {
            try { babylonScene.dispose(); } catch (e) { console.error(`${C_myInstanceId_cleanup}: Error scene.dispose():`, e); }
        }
        try { babylonEngine.dispose(); } catch (e) { console.error(`${C_myInstanceId_cleanup}: Error engine.dispose():`, e); }
      }
      setEngine(null); setScene(null);
    };
   }, [instanceIdRef, setEngine, setScene]); // canvasRef is stable


  // Callback to check and synchronize PiP list with DOM reality
  const checkAndSyncPips = useCallback(() => {
    const C_myInstanceId = instanceIdRef.current;
    const currentPipHostMap = pipHostDivsRef.current;
    
    const validatedPipInfos = []; // Will hold pipInfos for pips that are still valid
    const validatedPipHostMap = new Map(); // Will hold hostDivs for pips that are still valid
    
    let pipsInfoStateNeedsUpdate = false;
    let pipHostRefNeedsUpdate = false;

    // Phase 1: Validate entries from activePipsInfo against the DOM
    activePipsInfo.forEach(pipInfo => {
        const hostDiv = currentPipHostMap.get(pipInfo.id);
        if (hostDiv && hostDiv.isConnected) {
            validatedPipInfos.push(pipInfo); // Keep this pipInfo
            validatedPipHostMap.set(pipInfo.id, hostDiv); // Keep this hostDiv in our new validated map
        } else {
            pipsInfoStateNeedsUpdate = true; // This pipInfo from state is no longer valid or its div is gone
        }
    });

    // If the number of pips in validatedPipInfos is different from activePipsInfo, state needs update
    if (activePipsInfo.length !== validatedPipInfos.length) {
        pipsInfoStateNeedsUpdate = true;
    }

    // Phase 2: Validate entries from pipHostDivsRef.current against the DOM
    // This ensures the ref itself is clean and only contains connected divs.
    // We build a new map (finalValidatedHostMap) for what pipHostDivsRef.current should be.
    const finalValidatedHostMapForRef = new Map();
    currentPipHostMap.forEach((hostDiv, pipId) => {
        if (hostDiv && hostDiv.isConnected) {
            finalValidatedHostMapForRef.set(pipId, hostDiv);
        } else {
            // A div in the current ref is disconnected, so it won't be in the new map
            pipHostRefNeedsUpdate = true; 
        }
    });
    
    // If the size of the newly built map for the ref is different from the current ref's size
    if (currentPipHostMap.size !== finalValidatedHostMapForRef.size) {
        pipHostRefNeedsUpdate = true;
    }

    if (pipsInfoStateNeedsUpdate || pipHostRefNeedsUpdate) {
        // console.log(`%c${C_myInstanceId}: PiP list sync: activePipsInfo ${pipsInfoStateNeedsUpdate ? 'needs update' : 'OK'}, pipHostRef ${pipHostRefNeedsUpdate ? 'needs update' : 'OK'}.`, "color: DodgerBlue");
        
        if (pipHostRefNeedsUpdate) {
            pipHostDivsRef.current = finalValidatedHostMapForRef;
        }
        if (pipsInfoStateNeedsUpdate) {
            // It's possible validatedPipInfos is already correct from Phase 1.
            // If pipHostRefNeedsUpdate was true, some pips might have been removed from pipHostDivsRef.
            // We should ensure validatedPipInfos only contains pips present in the *new* pipHostDivsRef.current.
            const finalPipsToShow = validatedPipInfos.filter(p => pipHostDivsRef.current.has(p.id));
            setActivePipsInfo(finalPipsToShow);
        } else if (pipHostRefNeedsUpdate) {
            // activePipsInfo was fine, but pipHostDivsRef was cleaned.
            // Ensure activePipsInfo still aligns with the cleaned ref.
             setActivePipsInfo(prevPips => prevPips.filter(p => pipHostDivsRef.current.has(p.id)));
        }
    }
  }, [activePipsInfo, setActivePipsInfo, pipHostDivsRef, instanceIdRef]);

  // Effect for periodic synchronization
  useEffect(() => {
    const initialCheckTimeout = setTimeout(checkAndSyncPips, 250); // Initial check
    const intervalId = setInterval(checkAndSyncPips, 3000); // Check every 3 seconds
    return () => {
        clearTimeout(initialCheckTimeout);
        clearInterval(intervalId);
    };
  }, [checkAndSyncPips]);


  useEffect(() => {
    const C_myInstanceId = instanceIdRef.current;
    // console.log(`${C_myInstanceId}: Main useEffect triggered. RefreshKey:`, refreshKey);
    let cleanupBabylon = () => {}; const loadedScripts = [];
    
    const setupEnvironment = async () => {
      try {
        if (!window.BABYLON || !window.BABYLON.SceneLoader) {
            loadedScripts.push(await loadScript("https://cdn.babylonjs.com/babylon.js"));
            loadedScripts.push(await loadScript("https://cdn.babylonjs.com/loaders/babylonjs.loaders.min.js"));
        }
        if (window.BABYLON && window.BABYLON.SceneLoader) {
             cleanupBabylon = await initBabylon();
        } else { console.error(`${C_myInstanceId}: Babylon.js still NOT available after attempting load.`); }
      } catch (error) {
        console.error(`${C_myInstanceId}: Top-level error in setupEnvironment:`, error);
      }
    };
    setupEnvironment();

    return () => {
      const C_myInstanceId_cleanup = instanceIdRef.current;
      console.log(`%c${C_myInstanceId_cleanup}: Main useEffect cleanup (RefreshKey: ${refreshKey}). Cleaning resources.`, "color: red; font-weight: bold;");
      if (typeof cleanupBabylon === 'function') {
        cleanupBabylon();
      }
      
      const pipsToClose = Array.from(pipHostDivsRef.current.keys());
      pipsToClose.forEach(pipId => {
          closePip(pipId); // This will modify pipHostDivsRef.current and activePipsInfo
      });
      // States and refs should be cleared by the series of closePip calls.
      // Explicit reset of counter for this instance.
      spawnedPipCountRef.current = 0;
      
      loadedScripts.forEach(script => { if (script?.parentElement) { document.body.removeChild(script); }});
      // console.log(`${C_myInstanceId_cleanup}: Main useEffect cleanup finished.`);
    };
  }, [refreshKey, closePip, initBabylon, loadScript, instanceIdRef]);

  return (
    <div style={{ position: "relative", width: "100%", height: "500px", overflow: "hidden", background: "#121212", border: "1px solid #444" }}>
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
          max-height: 120px; 
          font-size: 12px; 
          background-color: rgba(30, 30, 30, 0.85);
          color: #e0e0e0;
          border-radius: 6px;
          border: 1px solid #444;
          padding: 8px; 
          z-index: 15000; 
          font-family: sans-serif;
          overflow-y: auto;
          box-shadow: 0 2px 8px rgba(0,0,0,0.3);
        }
        .active-pips-panel h4 {
          margin-top: 0;
          margin-bottom: 6px; 
          font-size: 13px; 
          color: #00aaff;
          border-bottom: 1px solid #555;
          padding-bottom: 4px; 
        }
        .active-pips-panel ul { list-style: none; padding: 0; margin: 0; }
        .active-pips-panel li {
          display: flex; justify-content: space-between; align-items: center; 
          margin-bottom: 4px; 
          padding: 2px 0; 
        }
        .active-pips-panel li .pip-info { 
            white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
            flex-grow: 1; margin-right: 5px; 
        }
         .active-pips-panel li .pip-id-snippet { font-size: 9px; color: #999; margin-left: 4px; }
        .active-pips-panel .no-pips { color: #888; font-style: italic; }
        .panel-pip-close-button {
            background: none; border: none; color: #ff6b6b; cursor: pointer;
            font-size: 14px; padding: 0 4px; line-height: 1; font-weight: bold;
        }
        .panel-pip-close-button:hover { color: #ff4757; }
      `}</style>

      <button onClick={() => setRefreshKey(prev => prev + 1)} className="refresh-button"
        style={{ position: "absolute", top: "5px", right: "5px", zIndex: 20000, width: "30px", height: "30px", 
                  borderRadius: "50%", border: "none", display: "flex", justifyContent: "center", alignItems: "center",
                  cursor: "pointer", boxShadow: "0px 2px 5px rgba(0,0,0,0.4)", color: "white", outline: "none" }}
        aria-label="Refresh This Scene" title="Refresh This Scene" >
        <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="currentColor">
          <path d="M12 4V1L8 5l4 4V6c3.31 0 6 2.69 6 6s-2.69 6-6 6-6-2.69-6-6H4c0 4.42 3.58 8 8 8s8-3.58 8-8-3.58-8-8-8z"/>
        </svg>
      </button>

      <div style={{position: "absolute", top: "5px", left: "5px", zIndex: 20000}}>
        <button
          className="pip-spawn-button"
          style={{padding: "4px 8px", fontSize: "12px"}}
          onClick={() => {
            spawnedPipCountRef.current += 1;
            const spawnOrder = spawnedPipCountRef.current;
            const offset = (spawnedPipCountRef.current -1) * 20;
            spawnPip({
              filePath: "BabylonGLContextAware.component.v1.md", // Assuming this file exports WorldView
              header: "ViewComponent", // Assuming WorldView is under ViewComponent export
              functionName: "WorldView",
              componentProps: {
                  titleText: `Nested World #${spawnOrder}`,
              },
              initialStyle: {
                width: "250px",
                height: "200px",
                top: `${10 + offset}px`,
                right: `${40 + offset}px`,
                left: "auto",
                border: "2px solid purple"
              }
            });
          }}
          title="Spawn Nested WorldView PiP"
        >
          Spawn Nested World
        </button>
      </div>

      <div className="active-pips-panel">
        <h4 title={`Instance ID: ${instanceIdRef.current}`}>{instanceIdRef.current} Contexts</h4>
        {(!engine || !scene) && activePipsInfo.length === 0 ? (
          <p className="no-pips">No contexts active.</p>
        ) : (
          <ul>
            {engine && scene && (
              <li key={MAIN_SCENE_PANEL_ID}>
                <span className="pip-info" title="Main rendering context for this instance">
                  {MAIN_SCENE_PANEL_TITLE}
                </span>
              </li>
            )}
            {activePipsInfo.map(pip => (
              <li key={pip.id}>
                <span className="pip-info" title={`ID: ${pip.id}`}>
                  {pip.title}
                  <span className="pip-id-snippet">(ID: {pip.id.substring(0, 12)}...)</span>
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