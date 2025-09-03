

# ViewComponent


```jsx
// Ensure 'dc' is available in the environment where this component runs.
// In a typical React setup, this would be:
// import React, { useRef, useEffect, useState } from 'react';
// For 'dc' environment, this syntax is correct:
const { useRef, useEffect, useState } = dc;

// --- WebGL Context Logger Utility ---
/**
 * Wraps a WebGL rendering context to log its API calls to the console.
 * @param {WebGLRenderingContext | WebGL2RenderingContext} gl - The WebGL context to wrap.
 * @param {object} [options={}] - Configuration options for logging.
 * @param {boolean} [options.logArgs=true] - Whether to log arguments of GL calls.
 * @param {boolean} [options.logResult=false] - Whether to log results of GL calls.
 * @param {string} [options.contextName='WebGL'] - A name for the context, used in log messages.
 * @param {number} [options.maxArrayLength=16] - Max length of array elements to log in full. Longer arrays are summarized.
 * @param {string[] | null} [options.traceFunctions=null] - An array of specific function names to trace.
 *                                                       If null or empty, attempts to trace all discoverable functions on the context's prototype.
 *                                                       This can be very verbose. A curated list is recommended for most uses.
 * @param {boolean} [options.logGetError=true] - Whether to specifically log the result of gl.getError() calls.
 * @param {boolean} [options.checkErrorAfterEachCall=false] - If true, calls gl.getError() after each wrapped function call.
 *                                                          This is very performance-intensive and should only be used for deep debugging.
 */
function wrapWebGLContext(gl, options = {}) {
    const {
        logArgs = true,
        logResult = false,
        contextName = 'WebGL',
        maxArrayLength = 16,
        traceFunctions = null,
        logGetError = true, // Default to true, as it's useful
        checkErrorAfterEachCall = false
    } = options;

    const glConstants = {};
    // Create a reverse map for WebGL constants
    for (const key in gl) {
        if (typeof gl[key] === 'number' && key.toUpperCase() === key) {
            // Heuristic to prefer shorter/more common constant names if multiple keys map to the same value
            if (!glConstants[gl[key]] || glConstants[gl[key]].length > key.length) {
                glConstants[gl[key]] = key;
            }
        }
    }
    // Manually add/override common constants for clarity or if missed by the loop
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
    for(const val in commonConstants) {
        glConstants[Number(val)] = commonConstants[val]; // Ensure keys are numbers for lookup
    }

    function formatArg(arg) {
        if (arg === null) return "null";
        if (arg === undefined) return "undefined";

        if (glConstants[arg] !== undefined) {
            return `${glConstants[arg]} (${arg})`;
        }

        const prototypes = [
            { proto: window.WebGLBuffer, name: "WebGLBuffer" },
            { proto: window.WebGLTexture, name: "WebGLTexture" },
            { proto: window.WebGLProgram, name: "WebGLProgram" },
            { proto: window.WebGLShader, name: "WebGLShader" },
            { proto: window.WebGLFramebuffer, name: "WebGLFramebuffer" },
            { proto: window.WebGLRenderbuffer, name: "WebGLRenderbuffer" },
            { proto: window.WebGLUniformLocation, name: "WebGLUniformLocation" }
        ];
        if (window.WebGLVertexArrayObject) { // For WebGL2 or VAO extension
            prototypes.push({ proto: window.WebGLVertexArrayObject, name: "WebGLVertexArrayObject" });
        }
        for (const p of prototypes) {
            if (p.proto && arg instanceof p.proto) return p.name;
        }

        if (Array.isArray(arg) || (typeof arg === "object" && arg && typeof arg.length === 'number' && typeof arg !== 'string' && !(arg instanceof String))) {
            const arrSlice = Array.from(arg.slice(0, maxArrayLength));
            if (arg.length > maxArrayLength) {
                return `[${arg.constructor.name}, len: ${arg.length}, data: [${arrSlice.join(', ')}]...]`;
            }
            return `[${arg.constructor.name}, len: ${arg.length}, data: [${arrSlice.join(', ')}]`;
        }
        if (typeof arg === 'string') {
             if (arg.includes('\n') && arg.length > 100) { // Likely a shader source
                return `"${arg.substring(0, 50).replace(/\n/g, '\\n')}..." (len ${arg.length}, shader?)`;
             }
             if (arg.length > 50) return `"${arg.substring(0, 47)}..." (len ${arg.length})`;
        }
        return String(arg);
    }

    const glPrototype = Object.getPrototypeOf(gl);
    const functionsToInstrument = traceFunctions && traceFunctions.length > 0
        ? traceFunctions
        : Object.getOwnPropertyNames(glPrototype).filter(name =>
            typeof glPrototype[name] === 'function' && !name.startsWith('_')
        );

    let instrumentedCount = 0;
    functionsToInstrument.forEach(functionName => {
        if (typeof glPrototype[functionName] !== 'function') {
            // console.warn(`[${contextName}] Property ${functionName} is not a function on gl prototype. Skipping.`);
            return;
        }
        const originalFunction = gl[functionName];

        gl[functionName] = function(...args) {
            const prefix = `[${contextName} CALL] ${functionName}`;
            if (logArgs) {
                const formattedArgs = args.map(formatArg);
                console.log(`${prefix}(${formattedArgs.join(', ')})`);
            } else {
                console.log(prefix);
            }

            const result = originalFunction.apply(gl, args);

            if (functionName === 'getError' && logGetError) {
                 if (result !== gl.NO_ERROR) {
                    console.error(` L> [${contextName} GET_ERROR_RESULT] -> ${glConstants[result] || result} (${result})`);
                } else if (logResult) { // Only log NO_ERROR if general logResult is true
                    console.log(` L> [${contextName} RSP] ${functionName} -> ${glConstants[result] || 'NO_ERROR (0)'}`);
                }
            } else if (logResult) {
                console.log(` L> [${contextName} RSP] ${functionName} ->`, formatArg(result));
            }

            if (checkErrorAfterEachCall && functionName !== 'getError') {
                const error = gl.getError(); // Call original getError (now wrapped, but we need the native behavior for this check)
                if (error !== gl.NO_ERROR) {
                    console.error(`[${contextName} ERROR] After ${functionName}: ${glConstants[error] || error} (${error})`);
                }
            }
            return result;
        };
        instrumentedCount++;
    });

    console.log(`%c[${contextName}] Context wrapped for logging. ${instrumentedCount} functions instrumented.`, "color: #2E8B57; font-weight: bold;");
    console.log(`%c[${contextName}] Options: logArgs=${logArgs}, logResult=${logResult}, maxArrLen=${maxArrayLength}, checkErr=${checkErrorAfterEachCall}`, "color: #2E8B57;");
    if (traceFunctions && traceFunctions.length > 0) {
        console.log(`%c[${contextName}] Tracing: ${traceFunctions.join(', ')}`, "color: #2E8B57;");
    } else {
        console.log(`%c[${contextName}] Tracing all discoverable prototype functions (can be very verbose).`, "color: #2E8B57;");
    }
}
// --- End WebGL Context Logger Utility ---


function WorldView() {
  const canvasRef = useRef(null);
  const [engine, setEngine] = useState(null);
  const [scene, setScene] = useState(null);
  const [refreshKey, setRefreshKey] = useState(0);

  const loadScript = (src) => {
    return new Promise((resolve, reject) => {
      const script = document.createElement("script");
      script.src = src;
      script.async = true;
      script.onload = () => {
        console.log(`WorldView State: Script loaded successfully: ${src}`);
        resolve(script);
      };
      script.onerror = (e) => {
        console.error(`WorldView State: Error loading script: ${src}`, e);
        reject(new Error(`Failed to load script: ${src}`));
      };
      document.body.appendChild(script);
    });
  };

  const initBabylon = async () => {
    if (!canvasRef.current || !window.BABYLON || !window.BABYLON.SceneLoader) {
      console.error("WorldView State: initBabylon - Canvas ref missing or Babylon.js not loaded. Skipping.");
      return () => { console.log("WorldView State: Babylon init failed, no cleanup."); };
    }

    console.log("WorldView State: Initializing Babylon.js engine and scene...");

    const babylonEngine = new window.BABYLON.Engine(
      canvasRef.current,
      true, // enable antialiasing
      { preserveDrawingBuffer: true, stencil: true }
    );

    // --- ADD GL CONTEXT LOGGING ---
    // Configure WebGL logging for a higher-level overview, not per-frame details.
    if (babylonEngine._gl && typeof wrapWebGLContext === 'function') {
        console.log("WorldView State: Enabling WebGL context logging with a high-level filter.");
        wrapWebGLContext(babylonEngine._gl, {
            logArgs: true,
            logResult: false, // Set to true to see return values of even these calls
            contextName: 'BabylonGL-Setup', // Changed name for clarity
            maxArrayLength: 8,
            checkErrorAfterEachCall: false, // Keep false for performance unless deep debugging specific GL errors
            logGetError: true, // Specifically log getError results
            
            traceFunctions: [
                // Shader and Program lifecycle (shows setup, crucial for rendering)
                'createShader', 'shaderSource', 'compileShader', 'getShaderParameter', 'getShaderInfoLog',
                'createProgram', 'attachShader', 'linkProgram', 'getProgramParameter', 'getProgramInfoLog',
                'useProgram', // Shows which shader program becomes active
                'deleteShader', 'deleteProgram',

                // Texture loading (shows when image data is sent to GPU - typically at load time)
                'texImage2D', 'texSubImage2D', 'generateMipmap', 'pixelStorei', 'bindTexture', 'activeTexture',

                // Buffer creation and data upload (initial geometry setup)
                'createBuffer', 'bindBuffer', 'bufferData',
                // 'vertexAttribPointer', 'enableVertexAttribArray', // Can be noisy if many attributes

                // Framebuffer operations (less frequent, indicate render target changes)
                // 'bindFramebuffer', 'checkFramebufferStatus', 'framebufferTexture2D', // Uncomment if debugging FBOs

                // Explicit error checking by the application/library
                'getError' // Logs gl.getError() calls and their results if not NO_ERROR
            ],
            // TO DEBUG EVERYTHING (VERY VERBOSE - for deep GL issues):
            // traceFunctions: null,
            // logResult: true,
            // checkErrorAfterEachCall: true, // (Extremely slow)

            // TO DEBUG SPECIFIC PER-FRAME FUNCTIONS (example):
            // traceFunctions: ['drawElements', 'uniformMatrix4fv', 'clear', 'getError'],
        });
    } else {
        if (!babylonEngine._gl) console.warn("WorldView State: WebGL context (_gl) not found on engine. Logging unavailable.");
        if (typeof wrapWebGLContext !== 'function') console.warn("WorldView State: wrapWebGLContext function not defined. Logging unavailable.");
    }
    // --- END GL CONTEXT LOGGING ---

    const babylonScene = new window.BABYLON.Scene(babylonEngine);
    console.log("WorldView State: Babylon.js Scene created.");

    setEngine(babylonEngine);
    setScene(babylonScene);

    const camera = new window.BABYLON.ArcRotateCamera(
      "Camera",
      -Math.PI / 2,
      Math.PI / 2.5,
      10,
      window.BABYLON.Vector3.Zero(),
      babylonScene
    );
    camera.attachControl(canvasRef.current, true);
    camera.minZ = 0.1;
    console.log("WorldView State: Camera created and controls attached.");

    const rotationSpeed = 0.008;
    babylonScene.clearColor = new window.BABYLON.Color4(0, 0, 0, 1);

    babylonScene.createDefaultEnvironment({
        createSkybox: false,
        enableGround: false,
        createGround: false,
        environmentTexture: "https://assets.babylonjs.com/environments/studio.env",
        intensity: 1.2,
    });
    console.log("WorldView State: Default environment created.");

    const directionalLight = new window.BABYLON.DirectionalLight(
      "directionalLight",
      new window.BABYLON.Vector3(0.5, -1, 0.5),
      babylonScene
    );
    directionalLight.intensity = 1.5;
    directionalLight.diffuse = new window.BABYLON.Color3(1.0, 0.95, 0.9);
    console.log("WorldView State: Directional light created.");

    const modelPath = "_setup/_babylon/b26.card.888.glb";
    try {
      console.log(`WorldView State: Attempting to load GLB model from: ${modelPath}`);
      const assetUrl = dc.app.vault.adapter.getResourcePath(modelPath);
      console.log("WorldView State: Resolved GLB asset URL:", assetUrl);

      const result = await window.BABYLON.SceneLoader.ImportMeshAsync(
        null, "", assetUrl, babylonScene,
        (evt) => {
            if (evt.lengthComputable) {
                const percentComplete = (evt.loaded / evt.total * 100).toFixed(2);
                console.log(`WorldView State: GLB Loading Progress: ${percentComplete}%`);
            } else {
                console.log(`WorldView State: GLB Loading Progress: ${evt.loaded} bytes loaded (total size unknown)`);
            }
        }
      );

      if (result.meshes && result.meshes.length > 0) {
        let mainModelMesh = result.meshes.find(m => m.getTotalVertices() > 0 && m.name !== "__root__") || result.meshes[0];
        mainModelMesh.position = window.BABYLON.Vector3.Zero();
        mainModelMesh.scaling = new window.BABYLON.Vector3(2.5, 3.5, 3.5);
        console.log(`WorldView State: GLB model loaded successfully: ${mainModelMesh.name} and scaled.`);

        const boundingInfo = mainModelMesh.getBoundingInfo();
        if (boundingInfo) {
            const center = boundingInfo.boundingSphere.center;
            const radius = boundingInfo.boundingSphere.radius;
            camera.setTarget(center);
            camera.radius = radius * 7.7;
            console.log(`WorldView State: Camera adjusted for model: target=${center.toString()}, radius=${camera.radius}`);
        } else {
            console.warn("WorldView State: Bounding info not found for main model mesh, using default camera settings.");
            camera.setTarget(window.BABYLON.Vector3.Zero());
            camera.radius = 10;
        }
      } else {
        console.warn("WorldView State: GLB loaded, but no meshes found in the result.");
      }
    } catch (error) {
      console.error("WorldView State: Error loading GLB model:", error);
    }

    console.log("WorldView State: Starting Babylon.js render loop.");
    babylonEngine.runRenderLoop(() => {
      if (babylonScene.activeCamera && babylonScene.isReady()) {
          camera.alpha += rotationSpeed;
          babylonScene.render();
      }
    });

    const onResize = () => {
        console.log("WorldView State: Window resized, resizing engine.");
        babylonEngine.resize();
    }
    window.addEventListener("resize", onResize);

    const canvas = canvasRef.current;
    const handleWheel = (e) => e.preventDefault();
    if (canvas) canvas.addEventListener("wheel", handleWheel, { passive: false });

    return () => {
      console.log("WorldView State: Cleaning up Babylon.js scene and engine...");
      window.removeEventListener("resize", onResize);
      babylonEngine.stopRenderLoop();
      console.log("WorldView State: Render loop stopped.");
      if (babylonScene) {
          babylonScene.dispose();
          console.log("WorldView State: Scene disposed.");
      }
      if (babylonEngine) {
          babylonEngine.dispose();
          console.log("WorldView State: Engine disposed.");
      }
      setEngine(null);
      setScene(null);

      if (canvas) canvas.removeEventListener("wheel", handleWheel);
      console.log("WorldView State: Babylon.js cleanup complete.");
    };
  };

  useEffect(() => {
    console.log("WorldView State: useEffect triggered (refreshKey changed or initial mount). Current refreshKey:", refreshKey);
    let cleanupBabylon = () => {};
    const loadedScripts = [];

    const setupEnvironment = async () => {
      try {
        if (!window.BABYLON || !window.BABYLON.SceneLoader) {
            console.log("WorldView State: Babylon.js core/loaders not found. Attempting to load from CDN...");
            loadedScripts.push(await loadScript("https://cdn.babylonjs.com/babylon.js"));
            loadedScripts.push(await loadScript("https://cdn.babylonjs.com/loaders/babylonjs.loaders.min.js"));
            console.log("WorldView State: Babylon.js core and loaders script loading initiated.");
        } else {
            console.log("WorldView State: Babylon.js and Loaders already present in global scope.");
        }

        if (window.BABYLON && window.BABYLON.SceneLoader) {
            console.log("WorldView State: Babylon.js ready, proceeding with initBabylon.");
            cleanupBabylon = await initBabylon();
        } else {
            console.error("WorldView State: Babylon.js or SceneLoader is NOT available after script loading attempts. Initialization cannot proceed.");
        }
      } catch (error) {
        console.error("WorldView State: Failed to load Babylon.js scripts or initialize the scene:", error);
      }
    };

    setupEnvironment();

    return () => {
      console.log("WorldView State: useEffect cleanup function running (component unmount or refreshKey changed).");
      if (typeof cleanupBabylon === 'function') {
        cleanupBabylon();
      }
      loadedScripts.forEach(script => {
        if (script && script.parentElement) {
          console.log(`WorldView State: Removing dynamically loaded script: ${script.src}`);
          document.body.removeChild(script);
        }
      });
      console.log("WorldView State: useEffect cleanup completed.");
    };
  }, [refreshKey]);

  console.log("WorldView State: Rendering component. Canvas ref valid:", !!(canvasRef && canvasRef.current));

  return (
    <div style={{ position: "relative", width: "100%", height: "500px", overflow: "hidden" }}>
      <canvas ref={canvasRef} style={{ width: "100%", height: "100%", display: "block" }} />
      <style>
      {`
        .refresh-button {
          background-color: #333;
          transition: background-color 0.3s ease, transform 0.1s ease;
          box-sizing: border-box;
        }
        .refresh-button:hover {
          background-color: #6A0DAD;
          transform: scale(1.05);
        }
        .refresh-button:active {
          transform: scale(0.95);
        }
      `}
      </style>
      <button
        onClick={() => {
            console.log("WorldView State: Refresh button clicked. Incrementing refreshKey.");
            setRefreshKey(prevKey => prevKey + 1);
        }}
        className="refresh-button"
        style={{
          position: "absolute", top: "10px", right: "10px", zIndex: 10,
          width: "44px", height: "44px", borderRadius: "50%", border: "none",
          display: "flex", justifyContent: "center", alignItems: "center",
          cursor: "pointer", boxShadow: "0px 4px 10px rgba(0, 0, 0, 0.4)",
          color: "white", outline: "none",
        }}
        aria-label="Refresh Scene" title="Refresh Scene"
      >
        <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="currentColor">
          <path d="M12 4V1L8 5l4 4V6c3.31 0 6 2.69 6 6s-2.69 6-6 6-6-2.69-6-6H4c0 4.42 3.58 8 8 8s8-3.58 8-8-3.58-8-8-8z"/>
        </svg>
      </button>
    </div>
  );
}

// In 'dc' environment, components are typically exported like this:
return { WorldView };
```