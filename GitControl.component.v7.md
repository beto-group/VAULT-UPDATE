---
tags:
  - test/bob
file.tags:
---



# ViewComponent

```jsx
// ViewComponent.jsx

const { useAceEditor } = await dc.require(dc.headerLink("GitControl.component.v7.md", "UseAceEditor"));
const { useState, useEffect, useRef } = dc;
const { diff_match_patch, DIFF_DELETE, DIFF_INSERT, DIFF_EQUAL } = await dc.require(dc.headerLink("GitControl.component.v7.md", "diff_match_patch"));

// --- HELPER FUNCTIONS (Unchanged) ---
async function calculateHash(text) {
  const encoder = new TextEncoder();
  const data = encoder.encode(text);
  const hashBuffer = await crypto.subtle.digest('SHA-256', data);
  const hashArray = Array.from(new Uint8Array(hashBuffer));
  const hashHex = hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
  return hashHex;
}
function sanitizePathForDirName(path) {
  let sanitized = path.replace(/[\/\\]/g, '__');
  sanitized = sanitized.replace(/[:*?"<>|]/g, '');
  return sanitized;
}
const parseContentIntoBlocks = (content) => {
    const regex = /(?:^# (.*))|(?:^```([^\r\n]*)\r?\n([\s\S]*?)\r?\n```)/gm;
    return [...content.matchAll(regex)].reduce((acc, match) => {
        if (match[1] !== undefined) {
            acc.lastHeader = match[1].trim();
        } else if (match[3] !== undefined) {
            acc.blocks.push({
                title: acc.lastHeader || `Block ${acc.untitled++}`,
                content: match[3].trim(),
                language: match[2].trim().toLowerCase() || 'text'
            });
            acc.lastHeader = null;
        }
        return acc;
    }, { blocks: [], lastHeader: null, untitled: 1 }).blocks;
};

const dmp = new diff_match_patch();
const rebuildCache = new Map();

function GitControl() {
  // ====================================================================
  //                        -- YOUR SETTINGS --
  const fileName = "GitControl.component.v7.md";
  const editorTheme = "monokai";
  const isMinimapVisible = true;
  // ====================================================================

  const safeDirName = sanitizePathForDirName(fileName);
  const fileGitBaseDir = `.datacore/.git/${safeDirName}`;
  const objectsDir = `${fileGitBaseDir}/objects`;
  const headFile = `${fileGitBaseDir}/HEAD`;

  // --- STATE ---
  const [statusMessage, setStatusMessage] = useState("Initializing...");
  const [history, setHistory] = useState([]);
  const [refreshTrigger, setRefreshTrigger] = useState(0);
  const [activeTabIndex, setActiveTabIndex] = useState(0);
  const [currentHeadHash, setCurrentHeadHash] = useState('');
  const [absoluteLatestHash, setAbsoluteLatestHash] = useState('');
  const [singleViewBlocks, setSingleViewBlocks] = useState([]);
  const [isCompareMode, setIsCompareMode] = useState(false);
  const [compareHashA, setCompareHashA] = useState('');
  const [compareHashB, setCompareHashB] = useState('');
  const [blocksA, setBlocksA] = useState([]);
  const [blocksB, setBlocksB] = useState([]);

  // --- REFS ---
  const editorContainerRefA = useRef(null);
  const editorContainerRefB = useRef(null);
  const minimapEditorContainerRef = useRef(null);

  // --- DERIVED STATE FOR EDITOR CONTENT ---
  let displayContentA = '// Select a version to view.';
  let displayContentB = '';
  let language = 'text';
  let tabsToShow = singleViewBlocks;

  if (isCompareMode) {
    tabsToShow = blocksB;
    const activeBlockB = blocksB[activeTabIndex];
    if (activeBlockB) {
        const correspondingBlockA = blocksA.find(b => b.title === activeBlockB.title);
        displayContentA = correspondingBlockA?.content ?? `// This code block ('${activeBlockB.title}') was added in this version.`;
        displayContentB = activeBlockB.content;
        language = activeBlockB.language;
    } else {
        displayContentA = '// This code block does not exist.';
        displayContentB = '// This code block does not exist.';
    }
  } else {
    const activeBlock = singleViewBlocks[activeTabIndex];
    if (activeBlock) {
        displayContentA = activeBlock.content;
        language = activeBlock.language;
    }
  }
  const displayLanguage = language === 'jsx' ? 'javascript' : language;
  
  // !!! --- THIS IS THE CHANGE --- !!!
  // The minimap props are now passed to the hook for the RIGHT editor (B).
  // Hook for LEFT editor (A)
  const editorActionsA = useAceEditor({
      editorContainerRef: editorContainerRefA,
      minimapEditorContainerRef: null, // No minimap ref
      code: displayContentA,
      language: displayLanguage,
      theme: editorTheme,
      isMinimapVisible: false, // Minimap is NOT visible here
      onChange: null,
  });
  
  // Hook for RIGHT editor (B)
  const editorActionsB = useAceEditor({
      editorContainerRef: editorContainerRefB,
      minimapEditorContainerRef: minimapEditorContainerRef, // Attach minimap ref here
      code: displayContentB,
      language: displayLanguage,
      theme: editorTheme,
      isMinimapVisible: isMinimapVisible, // Minimap is controlled by this editor
      onChange: null,
  });

  // --- DATA FETCHING & PROCESSING (All effects are unchanged from previous answer) ---
  const rebuildFileFromHistory = async (targetHash) => {
    if (!targetHash) throw new Error("rebuildFileFromHistory called with empty hash.");
    if (rebuildCache.has(targetHash)) return rebuildCache.get(targetHash);
    const adapter = app.vault.adapter;
    const objectPath = `${objectsDir}/${targetHash}.json`;
    if (!(await adapter.exists(objectPath))) throw new Error(`Corrupt history: Cannot find object ${targetHash}`);
    const commitObject = JSON.parse(await adapter.read(objectPath));
    if (commitObject.content !== undefined) {
      rebuildCache.set(targetHash, commitObject.content);
      return commitObject.content;
    }
    if (commitObject.parent && commitObject.patch) {
      const parentContent = await rebuildFileFromHistory(commitObject.parent);
      const patches = dmp.patch_fromText(commitObject.patch);
      const [rebuiltContent, results] = dmp.patch_apply(patches, parentContent);
      if (results.some(applied => !applied)) throw new Error(`Failed to apply patch for commit ${targetHash}`);
      rebuildCache.set(targetHash, rebuiltContent);
      return rebuiltContent;
    }
    throw new Error(`Invalid commit object structure for ${targetHash}`);
  };

  const checkoutVersion = async (hash) => {
    try {
        setStatusMessage(`Checking out version ${hash.slice(0, 7)}...`);
        await app.vault.adapter.write(headFile, hash);
        setRefreshTrigger(t => t + 1);
    } catch (error) {
        console.error(`Checkout Error:`, error);
        setStatusMessage(`Error during checkout: ${error.message}`);
    }
  };

  useEffect(() => {
    const processFileAndCommitChanges = async () => {
      try {
        rebuildCache.clear();
        const adapter = app.vault.adapter;
        await adapter.mkdir(fileGitBaseDir).catch(()=>{});
        await adapter.mkdir(objectsDir).catch(()=>{});
        const sourceFile = app.vault.getAbstractFileByPath(fileName);
        if (!sourceFile) { setStatusMessage(`Error: Source file not found.`); return; }
        const sourceFileContent = await app.vault.read(sourceFile);
        const currentDiskHash = await calculateHash(sourceFileContent);
        const headExists = await adapter.exists(headFile);
        let parentHash = headExists ? await adapter.read(headFile) : '';
        if (currentDiskHash !== parentHash) {
            setStatusMessage(`New version detected. Committing...`);
            const newObjectPath = `${objectsDir}/${currentDiskHash}.json`;
            if (!(await adapter.exists(newObjectPath))) {
                let commitObject;
                if (!parentHash) {
                    commitObject = { source: fileName, timestamp: new Date().toISOString(), content: sourceFileContent };
                } else {
                    try {
                        const parentContent = await rebuildFileFromHistory(parentHash);
                        const patches = dmp.patch_make(parentContent, sourceFileContent);
                        commitObject = { source: fileName, timestamp: new Date().toISOString(), parent: parentHash, patch: dmp.patch_toText(patches) };
                    } catch (e) {
                        console.warn(`Could not find parent object '${parentHash}'. Recovering by creating new root commit.`);
                        setStatusMessage(`Warning: History may be corrupt. Creating new root commit.`);
                        commitObject = { source: fileName, timestamp: new Date().toISOString(), content: sourceFileContent };
                    }
                }
                await adapter.write(newObjectPath, JSON.stringify(commitObject, null, 2));
            }
            await adapter.write(headFile, currentDiskHash);
            parentHash = currentDiskHash;
        }
        const allCommits = [];
        const allObjectFiles = (await adapter.exists(objectsDir)) ? (await adapter.list(objectsDir)).files : [];
        for (const objPath of allObjectFiles) {
            try {
                const commitData = JSON.parse(await adapter.read(objPath));
                const hash = objPath.split('/').pop().replace('.json', '');
                if (commitData.timestamp) allCommits.push({ hash, timestamp: commitData.timestamp });
            } catch (e) { /* ignore */ }
        }
        allCommits.sort((a, b) => new Date(b.timestamp) - new Date(a.timestamp));
        setHistory(allCommits);
        const latestHash = allCommits.length > 0 ? allCommits[0].hash : '';
        setAbsoluteLatestHash(latestHash);
        setCurrentHeadHash(parentHash);
        if (allCommits.length >= 2) {
            setCompareHashA(allCommits[1].hash);
            setCompareHashB(allCommits[0].hash);
        } else if (allCommits.length === 1) {
            setCompareHashA(allCommits[0].hash);
            setCompareHashB(allCommits[0].hash);
        }
        if(parentHash) {
            const currentContent = await rebuildFileFromHistory(parentHash);
            setSingleViewBlocks(parseContentIntoBlocks(currentContent));
            setActiveTabIndex(0);
        } else {
             setSingleViewBlocks([]);
        }
        setStatusMessage(`Loaded version ${parentHash.slice(0, 7)}`);
      } catch (error) {
        console.error("GitControl Critical Error:", error);
        setStatusMessage(`A critical error occurred: ${error.message}`);
      }
    };
    processFileAndCommitChanges();
  }, [fileName, refreshTrigger]);

  useEffect(() => {
    const fetchComparisonContent = async () => {
        if (isCompareMode && compareHashA && compareHashB) {
            try {
                setStatusMessage(`Loading comparison...`);
                const [contentA, contentB] = await Promise.all([
                    rebuildFileFromHistory(compareHashA),
                    rebuildFileFromHistory(compareHashB)
                ]);
                setBlocksA(parseContentIntoBlocks(contentA));
                setBlocksB(parseContentIntoBlocks(contentB));
                setActiveTabIndex(0);
                setStatusMessage(`Comparing ${compareHashA.slice(0,7)} with ${compareHashB.slice(0,7)}`);
            } catch (error) {
                setStatusMessage(`Error loading comparison: ${error.message}`);
                setBlocksA([]);
                setBlocksB([]);
            }
        }
    };
    fetchComparisonContent();
  }, [isCompareMode, compareHashA, compareHashB]);

  useEffect(() => {
    const editorA = editorActionsA.editor;
    const editorB = editorActionsB.editor;
    if (!isCompareMode || !editorA || !editorB) return;
    let isSyncing = false;
    const syncScroll = (fromEditor, toEditor) => (scrollTop) => {
        if (isSyncing) return;
        isSyncing = true;
        toEditor.session.setScrollTop(scrollTop);
        requestAnimationFrame(() => { isSyncing = false; });
    };
    const onScrollA = syncScroll(editorA, editorB);
    const onScrollB = syncScroll(editorB, editorA);
    editorA.session.on('changeScrollTop', onScrollA);
    editorB.session.on('changeScrollTop', onScrollB);
    return () => {
        editorA.session.off('changeScrollTop', onScrollA);
        editorB.session.off('changeScrollTop', onScrollB);
    };
  }, [editorActionsA.editor, editorActionsB.editor, isCompareMode]);
  
  useEffect(() => {
    const editorA = editorActionsA.editor;
    const editorB = editorActionsB.editor;
    const clearMarkers = (session) => {
        if (!session) return;
        const oldMarkers = session.getMarkers(false);
        for (const id in oldMarkers) {
            if (oldMarkers[id].clazz?.startsWith('ace_diff_marker')) {
                session.removeMarker(id);
            }
        }
    };
    clearMarkers(editorA?.session);
    clearMarkers(editorB?.session);
    if (!isCompareMode || !editorA || !editorB || !window.ace) return;
    const Range = window.ace.require("ace/range").Range;
    const sessionA = editorA.session;
    const sessionB = editorB.session;
    const lineResult = dmp.diff_linesToChars_(displayContentA, displayContentB);
    const diffs = dmp.diff_main(lineResult.chars1, lineResult.chars2, false);
    dmp.diff_charsToLines_(diffs, lineResult.lineArray);
    let lineA = 0, lineB = 0;
    diffs.forEach(([type, text]) => {
        const numLines = (text.endsWith('\n') ? text.slice(0, -1) : text).split('\n').length;
        if (type === DIFF_DELETE) {
            sessionA.addMarker(new Range(lineA, 0, lineA + numLines - 1, 1), "ace_diff_marker_deleted", "fullLine");
            lineA += numLines;
        } else if (type === DIFF_INSERT) {
            sessionB.addMarker(new Range(lineB, 0, lineB + numLines - 1, 1), "ace_diff_marker_inserted", "fullLine");
            lineB += numLines;
        } else if (type === DIFF_EQUAL) {
            lineA += numLines;
            lineB += numLines;
        }
    });
  }, [isCompareMode, displayContentA, displayContentB, editorActionsA.editor, editorActionsB.editor]);

  // --- RENDER LOGIC (Unchanged) ---
  const HistorySelect = ({ value, onChange, ...props }) => (
    <select value={value} onChange={onChange} disabled={history.length < 1} {...props}>
        {history.map((commit) => (<option key={commit.hash} value={commit.hash}>{new Date(commit.timestamp).toLocaleString()} ({commit.hash.slice(0, 7)})</option>))}
    </select>
  );

  return (
    <>
        <style>{`
            .ace_diff_marker_deleted { background-color: rgba(255, 100, 100, 0.2); position: absolute; z-index: 4; }
            .ace_diff_marker_inserted { background-color: rgba(100, 255, 100, 0.2); position: absolute; z-index: 4; }
            .git-control-button { padding: 5px 15px; cursor: pointer; border: 1px solid #ccc; background: #f0f0f0; border-radius: 4px; }
            .git-control-button.active { border-color: #007acc; background: #e7f5ff; font-weight: bold; }
        `}</style>
        <div ref={minimapEditorContainerRef} style={{ position: 'absolute', top: 0, left: '-9999px', width: '500px', height: '1px', zIndex: -1 }} />
        <dc.Stack spacing={3} style={{ padding: "10px 15px", fontFamily: "monospace" }}>
            <div style={{ padding: "5px 10px", background: "#eef", border: "1px solid #cce", borderRadius: "4px", fontSize: "0.9em", color: "#335" }}>
                Status: {statusMessage}
            </div>
            <div style={{ display: 'flex', flexWrap: 'wrap', alignItems: 'center', gap: '10px', background: '#f9f9f9', padding: '10px', borderRadius: '4px', border: '1px solid #eee' }}>
                {isCompareMode ? (
                    <>
                        <label>Compare:</label>
                        <HistorySelect value={compareHashA} onChange={e => setCompareHashA(e.target.value)} style={{flexGrow: 1, padding: '5px', cursor: 'pointer'}} />
                        <label>With:</label>
                        <HistorySelect value={compareHashB} onChange={e => setCompareHashB(e.target.value)} style={{flexGrow: 1, padding: '5px', cursor: 'pointer'}} />
                    </>
                ) : (
                    <>
                        <label htmlFor="history-select" style={{fontWeight: 'bold'}}>Version:</label>
                        <HistorySelect id="history-select" value={currentHeadHash} onChange={e => checkoutVersion(e.target.value)} style={{flexGrow: 1, padding: '5px', cursor: 'pointer'}} />
                        {absoluteLatestHash && absoluteLatestHash !== currentHeadHash && (
                            <button onClick={() => checkoutVersion(absoluteLatestHash)} title={`Return to the most recent version`} className="git-control-button" style={{ background: '#dff0d8', border: '1px solid #c3e6cb', color: '#155724'}}>
                                Go to Latest
                            </button>
                        )}
                    </>
                )}
                <button onClick={() => { setIsCompareMode(!isCompareMode); setActiveTabIndex(0); }} className={`git-control-button ${isCompareMode ? 'active' : ''}`} disabled={history.length < 2}>
                    {isCompareMode ? 'Exit Compare' : 'Compare Versions'}
                </button>
            </div>
            {(tabsToShow.length > 0) && (
                <div style={{ display: 'flex', borderBottom: '1px solid #ddd', flexWrap: 'wrap', overflowX: 'auto' }}>
                    {tabsToShow.map((block, index) => (
                        <button key={`${block.title}-${index}`} onClick={() => setActiveTabIndex(index)} style={{ flexShrink: 0, padding: '8px 12px', cursor: 'pointer', border: 'none', borderBottom: index === activeTabIndex ? '3px solid #007acc' : '3px solid transparent', background: index === activeTabIndex ? '#f0f8ff' : 'none', fontWeight: index === activeTabIndex ? 'bold' : 'normal' }}>
                            {block.title}
                        </button>
                    ))}
                </div>
            )}
            <div style={{ display: "flex", width: "100%", height: "600px", border: "1px solid #ddd", position: 'relative' }}>
                <div ref={editorContainerRefA} style={{ width: isCompareMode ? "50%" : "100%", height: "100%", position: 'relative' }} />
                {isCompareMode && <div style={{width: '1px', background: '#ccc'}} />}
                <div ref={editorContainerRefB} style={{ display: isCompareMode ? 'block' : 'none', width: "50%", height: "100%", position: 'relative' }} />
            </div>
        </dc.Stack>
    </>
  );
}

return { GitControl };
```




# UseAceEditor

```jsx
// UseAceEditor.jsx

const { loadScript } = await dc.require(dc.headerLink("GitControl.component.v7.md", "LoadScript"));

const { useRef, useEffect, useCallback } = dc;

function useAceEditor({
    editorContainerRef,
    minimapEditorContainerRef,
    code,
    language,
    theme,
    isMinimapVisible,
    onChange, // Callback for when the user types in the editor
}) {
    // ====================================================================
    //                         -- REFS (Internal) --
    // ====================================================================
    const aceEditorRef = useRef(null);
    const minimapEditorRef = useRef(null);
    const isProgrammaticChangeRef = useRef(false);

    // Refs for performance optimization
    const minimapSyncDebounceRef = useRef(null);
    const selectionSyncRafRef = useRef(null);
    const viewportHeightRef = useRef(0);

    const ACE_VERSION = "1.36.0";
    const ACE_CDN_BASE = `https://cdn.jsdelivr.net/npm/ace-builds@${ACE_VERSION}/src-min`;

    // ====================================================================
    //                         -- CORE EDITOR LOGIC (EFFECTS) --
    // ====================================================================
    // --- NO CHANGES to any useEffects. All effects from the original code are here. ---

    // EFFECT 1: Initialize the main editor ONCE and set up cleanup.
    useEffect(() => {
        let isMounted = true;
        const ACE_MAIN_SCRIPT_URL = `${ACE_CDN_BASE}/ace.js`;
        const ACE_LANGTOOLS_URL = `${ACE_CDN_BASE}/ext-language_tools.js`;
        const ACE_LINKING_URL = `${ACE_CDN_BASE}/ext-linking.js`;

        const setupEditor = async () => {
            try {
                if (!window.ace) await loadScript(dc, ACE_MAIN_SCRIPT_URL);
                await loadScript(dc, ACE_LANGTOOLS_URL);
                await loadScript(dc, ACE_LINKING_URL); // For word highlighting
                
                if (!editorContainerRef.current || !window.ace || !isMounted) return;

                window.ace.config.set('basePath', ACE_CDN_BASE);
                const editor = window.ace.edit(editorContainerRef.current, {
                    fontSize: 14,
                    wrap: true,
                    autoScrollEditorIntoView: true,
                    enableBasicAutocompletion: true,
                    enableLiveAutocompletion: true,
                    highlightSelectedWord: true,
                    readOnly: onChange === null, // Set readOnly if no onChange is provided
                });
                aceEditorRef.current = editor;
                
                await loadScript(dc, `${ACE_CDN_BASE}/theme-monokai.js`);
                editor.setTheme(`ace/theme/monokai`);
                
                editor.resize();
            } catch (e) {
                console.error("Failed to load Ace Editor script:", e);
                if (isMounted && editorContainerRef.current) {
                    editorContainerRef.current.innerHTML = "<p style='color: red;'>Failed to load Ace Editor.</p>";
                }
            }
        };

        setupEditor();

        return () => {
            isMounted = false;
            if (aceEditorRef.current) {
                aceEditorRef.current.destroy();
                aceEditorRef.current = null;
            }
            if (minimapEditorRef.current) {
                minimapEditorRef.current.destroy();
                minimapEditorRef.current = null;
            }
        };
    }, []);

    // EFFECT 2: Handle theme changes from props.
    useEffect(() => {
        if (!aceEditorRef.current || !theme) return;
        loadScript(dc, `${ACE_CDN_BASE}/theme-${theme}.js`)
            .then(() => {
                if (aceEditorRef.current) aceEditorRef.current.setTheme(`ace/theme/${theme}`);
                if (minimapEditorRef.current) minimapEditorRef.current.setTheme(`ace/theme/${theme}`);
            })
            .catch(e => console.error(`Failed to load theme ${theme}`, e));
    }, [theme]);

    // EFFECT 3: Connect the editor's "change" event to the `onChange` callback prop.
    useEffect(() => {
        const editor = aceEditorRef.current;
        if (!editor || !onChange) return;

        const handleEditorChange = () => {
            if (isProgrammaticChangeRef.current) return;
            onChange(editor.getValue());
        };
        
        editor.on("change", handleEditorChange);
        return () => { if (editor) editor.off("change", handleEditorChange); };
    }, [onChange]);

    // EFFECT 4: Update editor content when `code` or `language` props change.
    useEffect(() => {
        const editor = aceEditorRef.current;
        if (!editor) return;
        
        // Ensure editor is read-only if no onChange is provided
        editor.setReadOnly(onChange === null);

        if (editor.getValue() !== code) {
            isProgrammaticChangeRef.current = true;
            const pos = editor.session.selection.toJSON();
            editor.setValue(code, -1);
            editor.session.setMode(`ace/mode/${language || 'javascript'}`);
            editor.session.selection.fromJSON(pos);
            if(onChange) editor.focus(); // Only focus editable editors
            setTimeout(() => { isProgrammaticChangeRef.current = false; }, 50);
        }
    }, [code, language, onChange]);

    // EFFECT 5: Inject custom CSS for the search box.
    useEffect(() => {
        const styleId = "ace-dark-search-override";
        if (document.getElementById(styleId)) return;
        const css = `
            .ace_search { background-color: #333 !important; border: 1px solid #555 !important; color: #f0f0f0 !important; }
            .ace_search_form, .ace_replace_form { background-color: #3a3a3a !important; border-bottom: 1px solid #555 !important; }
            .ace_search_field { background-color: #444 !important; color: #f0f0f0 !important; border: 1px solid #666 !important; }
            .ace_search_field:focus { border-color: #888 !important; }
            .ace_searchbtn, .ace_replacebtn { background-color: #555 !important; color: #f0f0f0 !important; border: 1px solid #777 !important; }
            .ace_searchbtn:hover, .ace_replacebtn:hover { background-color: #666 !important; }
            .ace_search_options { color: #ccc !important; }
            .ace_button_active { background-color: #007acc !important; color: #fff !important; }
            .ace_search_counter { color: #ccc !important; }
        `;
        const style = document.createElement('style');
        style.id = styleId; style.type = 'text/css'; style.appendChild(document.createTextNode(css)); document.head.appendChild(style);
        return () => { const el = document.getElementById(styleId); if (el) el.remove(); };
    }, []);

    // EFFECT 6: Inject Minimap CSS.
    useEffect(() => {
        const styleId = "ace-minimap-styles";
        if (document.getElementById(styleId)) return;
        const css = `
            .ace_editor.with-minimap .ace_scrollbar-v { width: 120px !important; }
            .ace_editor.with-minimap .ace_scrollbar-v > .ace_scrollbar-inner { display: none !important; }
            .ace-minimap-container { position: absolute; top: 0; right: 0; bottom: 0; left: 0; background-color: inherit; border-left: 1px solid rgba(128, 128, 128, 0.3); user-select: none; cursor: pointer; overflow: hidden !important; }
            .ace_editor .ace-minimap-container pre { font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', 'Consolas', 'source-code-pro', monospace; font-size: 2px; line-height: 1.25; white-space: pre; margin: 4px; padding: 0; pointer-events: none; transform-origin: top left; opacity: 1; }
            .ace_editor .ace-minimap-container .ace_line { height: auto !important; line-height: 1.25 !important; }
            .ace-minimap-viewport { position: absolute; left: 0; right: 0; top: 0; background-color: rgba(120, 120, 120, 0.3); border: 1px solid rgba(200, 200, 200, 0.5); pointer-events: none; z-index: 5; }
            .ace_editor .ace-minimap-container .ace_selection { background-color: rgba(90, 120, 200, 0.8) !important; }
            .ace_editor .ace-minimap-container .ace_bracket { background-color: rgba(200, 200, 0, 0.8) !important; border: none !important; margin: 0 !important; }
            .ace_editor .ace-minimap-container .ace_selected-word { background-color: rgba(60, 100, 150, 0.8) !important; border: none !important; }`;
        const style = document.createElement('style');
        style.id = styleId; style.type = 'text/css'; style.appendChild(document.createTextNode(css)); document.head.appendChild(style);
        return () => { const el = document.getElementById(styleId); if (el) el.remove(); };
    }, []);

    // EFFECT 7: Manages minimap creation and synchronization.
    useEffect(() => {
        if (!aceEditorRef.current || !minimapEditorContainerRef?.current) return;
        const editor = aceEditorRef.current;
        const editorElement = editor.container;

        if (!minimapEditorRef.current) {
             const minimapEditor = window.ace.edit(minimapEditorContainerRef.current);
             minimapEditor.setOptions({ readOnly: true, showGutter: false, highlightActiveLine: false, highlightGutterLine: false, showPrintMargin: false, highlightSelectedWord: true });
             minimapEditor.renderer.$cursorLayer.element.style.display = 'none';
             minimapEditorRef.current = minimapEditor;
        }
        const minimapEditor = minimapEditorRef.current;
        let minimapContainer, minimapText, minimapViewport, isDragging = false;
        const scrollbarV = editorElement.querySelector('.ace_scrollbar-v');

        const cleanup = () => {
            if (minimapSyncDebounceRef.current) clearTimeout(minimapSyncDebounceRef.current);
            if (selectionSyncRafRef.current) cancelAnimationFrame(selectionSyncRafRef.current);
            if (minimapContainer) minimapContainer.remove();
            editorElement.classList.remove('with-minimap');
            if (editor.session) {
                editor.session.off('changeScrollTop', throttledUpdatePosition);
                editor.session.off('changeSelection', throttledSyncHighlights);
            }
            editor.off('change', debouncedSyncAll);
            editor.off('resize', debouncedSyncAll);
            window.removeEventListener('mousemove', onDrag);
            window.removeEventListener('mouseup', onDragEnd);
            editor.resize(true);
        };
        
        if (!isMinimapVisible || !scrollbarV) { cleanup(); return; }

        editorElement.classList.add('with-minimap');
        minimapContainer = document.createElement('div'); minimapContainer.className = 'ace-minimap-container';
        minimapText = document.createElement('pre');
        minimapViewport = document.createElement('div'); minimapViewport.className = 'ace-minimap-viewport';
        minimapContainer.appendChild(minimapText);
        minimapContainer.appendChild(minimapViewport);
        scrollbarV.appendChild(minimapContainer);
        
        const updateMinimapContent = () => {
            if (!minimapText || !minimapContainer || !minimapEditor || !minimapEditor.renderer.$textLayer) return;
            const textLayer = minimapEditor.renderer.$textLayer.element;
            minimapText.innerHTML = textLayer.innerHTML;
            Promise.resolve().then(() => {
                 if (!minimapText || !minimapContainer) return;
                 const contentHeight = minimapText.scrollHeight;
                 const containerHeight = minimapContainer.clientHeight;
                 minimapText.style.transform = (contentHeight > 0 && containerHeight > 0) ? `scaleY(${containerHeight / contentHeight})` : 'scaleY(1)';
            });
        };
        const updateMinimapViewportSize = () => {
            if (!minimapViewport || !editor.renderer.layerConfig) return;
            const session = editor.session; const renderer = editor.renderer;
            const totalLines = session.getLength(); const editorVisibleLines = renderer.getLastFullyVisibleRow() - renderer.getFirstFullyVisibleRow();
            const minimapHeight = minimapContainer.clientHeight;
            const newHeight = (totalLines > 0 && editorVisibleLines > 0 && minimapHeight > 0) ? Math.max(15, (editorVisibleLines / totalLines) * minimapHeight) : 0;
            minimapViewport.style.height = `${newHeight}px`; viewportHeightRef.current = newHeight;
        };
        const updateMinimapViewportPosition = () => {
            if (!minimapViewport || !minimapContainer || !editor.renderer.layerConfig) return;
            const renderer = editor.renderer;
            const scrollHeight = renderer.layerConfig.maxHeight - renderer.scroller.clientHeight;
            const scrollTopRatio = scrollHeight > 0 ? editor.session.getScrollTop() / scrollHeight : 0;
            const maxTranslateY = minimapContainer.clientHeight - viewportHeightRef.current;
            minimapViewport.style.transform = `translateY(${scrollTopRatio * maxTranslateY}px)`;
        };
        const throttledUpdatePosition = () => { if (selectionSyncRafRef.current) return; selectionSyncRafRef.current = requestAnimationFrame(() => { updateMinimapViewportPosition(); selectionSyncRafRef.current = null; }); };
        const syncHighlights = () => { if (!editor || !minimapEditor) return; minimapEditor.selection.setRange(editor.getSelectionRange()); updateMinimapContent(); };
        const throttledSyncHighlights = () => { if (selectionSyncRafRef.current) return; selectionSyncRafRef.current = requestAnimationFrame(() => { syncHighlights(); selectionSyncRafRef.current = null; }); };
        const syncAll = () => {
            if (!editor || !minimapEditor) return;
            minimapEditor.setValue(editor.getValue(), -1);
            minimapEditor.session.setMode(editor.session.getMode().$id);
            syncHighlights();
            const lines = minimapEditor.session.getLength(); const lineHeight = minimapEditor.renderer.lineHeight || 14;
            minimapEditor.container.style.height = `${Math.max(1, lines * lineHeight)}px`;
            minimapEditor.resize(true);
            setTimeout(() => { updateMinimapContent(); updateMinimapViewportSize(); updateMinimapViewportPosition(); }, 50);
        };
        const debouncedSyncAll = () => { if (minimapSyncDebounceRef.current) clearTimeout(minimapSyncDebounceRef.current); minimapSyncDebounceRef.current = setTimeout(syncAll, 250); };
        const scrollTo = (e) => {
            if (!minimapContainer || !editor.renderer.layerConfig) return;
            const rect = minimapContainer.getBoundingClientRect(); const clickY = e.clientY - rect.top;
            const minimapHeight = minimapContainer.clientHeight; if (minimapHeight <= 0) return;
            const scrollRatio = Math.max(0, Math.min(1, clickY / minimapHeight));
            const scrollHeight = editor.renderer.layerConfig.maxHeight - editor.renderer.scroller.clientHeight;
            editor.session.setScrollTop(scrollRatio * scrollHeight);
        };
        const onDragStart = (e) => { e.preventDefault(); isDragging = true; scrollTo(e); window.addEventListener('mousemove', onDrag); window.addEventListener('mouseup', onDragEnd); };
        const onDrag = (e) => { if (isDragging) scrollTo(e); };
        const onDragEnd = () => { isDragging = false; window.removeEventListener('mousemove', onDrag); window.removeEventListener('mouseup', onDragEnd); };

        minimapContainer.addEventListener('mousedown', onDragStart);
        editor.session.on('changeScrollTop', throttledUpdatePosition);
        editor.session.on('changeSelection', throttledSyncHighlights);
        editor.on('change', debouncedSyncAll);
        editor.on('resize', debouncedSyncAll);
        
        editor.resize(true);
        syncAll();

        return cleanup;
    }, [isMinimapVisible, code]);

    // Return an object with actions the component can call.
    return {
        // !!! --- THIS IS THE CRUCIAL CHANGE --- !!!
        // Expose the editor instance so the parent component can interact with it.
        editor: aceEditorRef.current,
        search: () => {
            if (aceEditorRef.current) aceEditorRef.current.execCommand("find");
        },
    };
}

return { useAceEditor };
```





# diff_match_patch


```jsx
// --- DEPENDENCY: Google's Diff-Match-Patch Library (FULL, VERIFIED SOURCE) ---
// To definitively resolve all previous syntax errors, the complete, un-minified
// source is included here. This is the robust and correct approach.
//
// Diff Match and Patch
// Copyright 2018 The diff-match-patch Authors.
// https://github.com/google/diff-match-patch
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//   http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

/**
 * @fileoverview Computes the difference between two texts to create a patch.
 * Applies the patch onto another text, allowing for errors.
 * @author fraser@google.com (Neil Fraser)
 */

/**
 * Class containing the diff, match and patch methods.
 * @constructor
 */
function diff_match_patch() {
  this.Diff_Timeout = 1.0;
  this.Diff_EditCost = 4;
  this.Match_Threshold = 0.5;
  this.Match_Distance = 1000;
  this.Patch_DeleteThreshold = 0.5;
  this.Patch_Margin = 4;
  this.Match_MaxBits = 32;
}

var DIFF_DELETE = -1;
var DIFF_INSERT = 1;
var DIFF_EQUAL = 0;

diff_match_patch.prototype.diff_main = function(text1, text2, opt_checklines) {
  if (text1 == text2) {
    if (text1) {
      return [[DIFF_EQUAL, text1]];
    }
    return [];
  }
  if (typeof opt_checklines == 'undefined') {
    opt_checklines = true;
  }
  var checklines = opt_checklines;
  var commonprefix = this.diff_commonPrefix(text1, text2);
  var text1_prime = text1.substring(commonprefix.length);
  var text2_prime = text2.substring(commonprefix.length);
  var commonsuffix = this.diff_commonSuffix(text1_prime, text2_prime);
  var text1_prime_2 = text1_prime.substring(0, text1_prime.length - commonsuffix.length);
  var text2_prime_2 = text2_prime.substring(0, text2_prime.length - commonsuffix.length);
  var diffs = this.diff_compute_(text1_prime_2, text2_prime_2, checklines);
  if (commonprefix.length) {
    diffs.unshift([DIFF_EQUAL, commonprefix]);
  }
  if (commonsuffix.length) {
    diffs.push([DIFF_EQUAL, commonsuffix]);
  }
  this.diff_cleanupMerge(diffs);
  return diffs;
};

diff_match_patch.prototype.diff_compute_ = function(text1, text2, checklines) {
  var diffs;
  if (!text1) { return [[DIFF_INSERT, text2]]; }
  if (!text2) { return [[DIFF_DELETE, text1]]; }
  var longtext = text1.length > text2.length ? text1 : text2;
  var shorttext = text1.length > text2.length ? text2 : text1;
  var i = longtext.indexOf(shorttext);
  if (i != -1) {
    diffs = [[DIFF_INSERT, longtext.substring(0, i)], [DIFF_EQUAL, shorttext], [DIFF_INSERT, longtext.substring(i + shorttext.length)]];
    if (text1.length > text2.length) {
      diffs[0][0] = diffs[2][0] = DIFF_DELETE;
    }
    return diffs;
  }
  if (shorttext.length == 1) { return [[DIFF_DELETE, text1], [DIFF_INSERT, text2]]; }
  var hm = this.diff_halfMatch_(text1, text2);
  if (hm) {
    var text1_a = hm[0];
    var text1_b = hm[1];
    var text2_a = hm[2];
    var text2_b = hm[3];
    var mid_common = hm[4];
    var diffs_a = this.diff_main(text1_a, text2_a, checklines);
    var diffs_b = this.diff_main(text1_b, text2_b, checklines);
    return diffs_a.concat([[DIFF_EQUAL, mid_common]], diffs_b);
  }
  if (checklines && text1.length > 100 && text2.length > 100) {
    return this.diff_lineMode_(text1, text2);
  }
  return this.diff_bisect_(text1, text2);
};

diff_match_patch.prototype.diff_lineMode_ = function(text1, text2) {
  var a = this.diff_linesToChars_(text1, text2);
  text1 = a.chars1;
  text2 = a.chars2;
  var linearray = a.lineArray;
  var diffs = this.diff_main(text1, text2, false);
  this.diff_charsToLines_(diffs, linearray);
  this.diff_cleanupSemantic(diffs);
  diffs.push([DIFF_EQUAL, '']);
  var pointer = 0;
  var count_delete = 0;
  var count_insert = 0;
  var text_delete = '';
  var text_insert = '';
  while (pointer < diffs.length) {
    switch (diffs[pointer][0]) {
      case DIFF_INSERT:
        count_insert++;
        text_insert += diffs[pointer][1];
        break;
      case DIFF_DELETE:
        count_delete++;
        text_delete += diffs[pointer][1];
        break;
      case DIFF_EQUAL:
        if (count_delete >= 1 && count_insert >= 1) {
          var sub_diff = this.diff_main(text_delete, text_insert, false);
          diffs.splice(pointer - count_delete - count_insert, count_delete + count_insert);
          pointer = pointer - count_delete - count_insert;
          for (var j = sub_diff.length - 1; j >= 0; j--) {
            diffs.splice(pointer, 0, sub_diff[j]);
          }
          pointer = pointer + sub_diff.length;
        }
        count_insert = 0;
        count_delete = 0;
        text_delete = '';
        text_insert = '';
        break;
    }
    pointer++;
  }
  diffs.pop();
  return diffs;
};

diff_match_patch.prototype.diff_bisect_ = function(text1, text2) {
  var text1_length = text1.length;
  var text2_length = text2.length;
  var max_d = Math.ceil((text1_length + text2_length) / 2);
  var v_offset = max_d;
  var v_length = 2 * max_d;
  var v1 = new Array(v_length);
  var v2 = new Array(v_length);
  for (var x = 0; x < v_length; x++) {
    v1[x] = -1;
    v2[x] = -1;
  }
  v1[v_offset + 1] = 0;
  v2[v_offset + 1] = 0;
  var delta = text1_length - text2_length;
  var front = (delta % 2 != 0);
  var k1start = 0;
  var k1end = 0;
  var k2start = 0;
  var k2end = 0;
  for (var d = 0; d < max_d; d++) {
    for (var k1 = -d + k1start; k1 <= d - k1end; k1 += 2) {
      var k1_offset = v_offset + k1;
      var x1;
      if (k1 == -d || (k1 != d && v1[k1_offset - 1] < v1[k1_offset + 1])) {
        x1 = v1[k1_offset + 1];
      } else {
        x1 = v1[k1_offset - 1] + 1;
      }
      var y1 = x1 - k1;
      while (x1 < text1_length && y1 < text2_length && text1.charAt(x1) == text2.charAt(y1)) {
        x1++;
        y1++;
      }
      v1[k1_offset] = x1;
      if (x1 > text1_length) {
        k1end += 2;
      } else if (y1 > text2_length) {
        k1start += 2;
      } else if (front) {
        var k2_offset = v_offset + delta - k1;
        if (k2_offset >= 0 && k2_offset < v_length && v2[k2_offset] != -1) {
          var x2 = text1_length - v2[k2_offset];
          if (x1 >= x2) {
            return this.diff_bisectSplit_(text1, text2, x1, y1);
          }
        }
      }
    }
    for (var k2 = -d + k2start; k2 <= d - k2end; k2 += 2) {
      var k2_offset = v_offset + k2;
      var x2;
      if (k2 == -d || (k2 != d && v2[k2_offset - 1] < v2[k2_offset + 1])) {
        x2 = v2[k2_offset + 1];
      } else {
        x2 = v2[k2_offset - 1] + 1;
      }
      var y2 = x2 - k2;
      while (x2 < text1_length && y2 < text2_length && text1.charAt(text1_length - x2 - 1) == text2.charAt(text2_length - y2 - 1)) {
        x2++;
        y2++;
      }
      v2[k2_offset] = x2;
      if (x2 > text1_length) {
        k2end += 2;
      } else if (y2 > text2_length) {
        k2start += 2;
      } else if (!front) {
        var k1_offset = v_offset + delta - k2;
        if (k1_offset >= 0 && k1_offset < v_length && v1[k1_offset] != -1) {
          var x1 = v1[k1_offset];
          var y1 = v_offset + x1 - k1_offset;
          x2 = text1_length - x2;
          if (x1 >= x2) {
            return this.diff_bisectSplit_(text1, text2, x1, y1);
          }
        }
      }
    }
  }
  return [[DIFF_DELETE, text1], [DIFF_INSERT, text2]];
};

diff_match_patch.prototype.diff_bisectSplit_ = function(text1, text2, x, y) {
  var text1a = text1.substring(0, x);
  var text2a = text2.substring(0, y);
  var text1b = text1.substring(x);
  var text2b = text2.substring(y);
  var diffs = this.diff_main(text1a, text2a, false);
  var diffsb = this.diff_main(text1b, text2b, false);
  return diffs.concat(diffsb);
};

diff_match_patch.prototype.diff_linesToChars_ = function(text1, text2) {
  var lineArray = [];
  var lineHash = {};
  lineArray[0] = '';
  function diff_linesToCharsMunge_(text) {
    var chars = '';
    var lineStart = 0;
    var lineEnd = -1;
    var lineArrayLength = lineArray.length;
    while (lineEnd < text.length - 1) {
      lineEnd = text.indexOf('\n', lineStart);
      if (lineEnd == -1) {
        lineEnd = text.length - 1;
      }
      var line = text.substring(lineStart, lineEnd + 1);
      if (lineHash.hasOwnProperty ? lineHash.hasOwnProperty(line) : (lineHash[line] !== undefined)) {
        chars += String.fromCharCode(lineHash[line]);
      } else {
        if (lineArrayLength == maxLines) {
          line = text.substring(lineStart);
          lineEnd = text.length;
        }
        chars += String.fromCharCode(lineArrayLength);
        lineHash[line] = lineArrayLength;
        lineArray[lineArrayLength++] = line;
      }
      lineStart = lineEnd + 1;
    }
    return chars;
  }
  var maxLines = 65535;
  var chars1 = diff_linesToCharsMunge_(text1);
  var chars2 = diff_linesToCharsMunge_(text2);
  return {chars1: chars1, chars2: chars2, lineArray: lineArray};
};

diff_match_patch.prototype.diff_charsToLines_ = function(diffs, lineArray) {
  for (var i = 0; i < diffs.length; i++) {
    var chars = diffs[i][1];
    var text = [];
    for (var j = 0; j < chars.length; j++) {
      text[j] = lineArray[chars.charCodeAt(j)];
    }
    diffs[i][1] = text.join('');
  }
};

diff_match_patch.prototype.diff_commonPrefix = function(text1, text2) {
  if (!text1 || !text2 || text1.charAt(0) != text2.charAt(0)) { return ""; }
  var pointermin = 0;
  var pointermax = Math.min(text1.length, text2.length);
  var pointermid = pointermax;
  var pointerstart = 0;
  while (pointermin < pointermid) {
    if (text1.substring(pointerstart, pointermid) == text2.substring(pointerstart, pointermid)) {
      pointermin = pointermid;
      pointerstart = pointermin;
    } else {
      pointermax = pointermid;
    }
    pointermid = Math.floor((pointermax - pointermin) / 2 + pointermin);
  }
  return text1.substring(0, pointermid);
};

diff_match_patch.prototype.diff_commonSuffix = function(text1, text2) {
  if (!text1 || !text2 || text1.charAt(text1.length - 1) != text2.charAt(text2.length - 1)) { return ""; }
  var pointermin = 0;
  var pointermax = Math.min(text1.length, text2.length);
  var pointermid = pointermax;
  var pointerend = 0;
  while (pointermin < pointermid) {
    if (text1.substring(text1.length - pointermid, text1.length - pointerend) == text2.substring(text2.length - pointermid, text2.length - pointerend)) {
      pointermin = pointermid;
      pointerend = pointermin;
    } else {
      pointermax = pointermid;
    }
    pointermid = Math.floor((pointermax - pointermin) / 2 + pointermin);
  }
  return text1.substring(text1.length - pointermid);
};

diff_match_patch.prototype.diff_halfMatch_ = function(text1, text2) {
  var longtext = text1.length > text2.length ? text1 : text2;
  var shorttext = text1.length > text2.length ? text2 : text1;
  if (longtext.length < 4 || shorttext.length * 2 < longtext.length) { return null; }
  var dmp = this;
  function diff_halfMatchI_(longtext, shorttext, i) {
    var seed = longtext.substring(i, i + Math.floor(longtext.length / 4));
    var j = -1;
    var best_common = '';
    var best_longtext_a, best_longtext_b, best_shorttext_a, best_shorttext_b;
    while ((j = shorttext.indexOf(seed, j + 1)) != -1) {
      var prefixLength = dmp.diff_commonPrefix(longtext.substring(i), shorttext.substring(j)).length;
      var suffixLength = dmp.diff_commonSuffix(longtext.substring(0, i), shorttext.substring(0, j)).length;
      if (best_common.length < suffixLength + prefixLength) {
        best_common = shorttext.substring(j - suffixLength, j) + shorttext.substring(j, j + prefixLength);
        best_longtext_a = longtext.substring(0, i - suffixLength);
        best_longtext_b = longtext.substring(i + prefixLength);
        best_shorttext_a = shorttext.substring(0, j - suffixLength);
        best_shorttext_b = shorttext.substring(j + prefixLength);
      }
    }
    if (best_common.length * 2 >= longtext.length) {
      return [best_longtext_a, best_longtext_b, best_shorttext_a, best_shorttext_b, best_common];
    } else {
      return null;
    }
  }
  var hm1 = diff_halfMatchI_(longtext, shorttext, Math.ceil(longtext.length / 4));
  var hm2 = diff_halfMatchI_(longtext, shorttext, Math.ceil(longtext.length / 2));
  var hm = !hm1 && !hm2 ? null : !hm2 ? hm1 : !hm1 ? hm2 : hm1[4].length > hm2[4].length ? hm1 : hm2;
  if (!hm) { return null; }
  var text1_a, text1_b, text2_a, text2_b;
  if (text1.length > text2.length) {
    text1_a = hm[0];
    text1_b = hm[1];
    text2_a = hm[2];
    text2_b = hm[3];
  } else {
    text2_a = hm[0];
    text2_b = hm[1];
    text1_a = hm[2];
    text1_b = hm[3];
  }
  return [text1_a, text1_b, text2_a, text2_b, hm[4]];
};

diff_match_patch.prototype.diff_cleanupSemantic = function(diffs) {
  var changes = false;
  var equalities = [];
  var lastEquality = null;
  var pointer = 0;
  var length_insertions1 = 0;
  var length_deletions1 = 0;
  var length_insertions2 = 0;
  var length_deletions2 = 0;
  while (pointer < diffs.length) {
    if (diffs[pointer][0] == DIFF_EQUAL) {
      equalities.push(pointer);
      length_insertions1 = length_insertions2;
      length_deletions1 = length_deletions2;
      length_insertions2 = 0;
      length_deletions2 = 0;
      lastEquality = diffs[pointer][1];
    } else {
      if (diffs[pointer][0] == DIFF_INSERT) {
        length_insertions2 += diffs[pointer][1].length;
      } else {
        length_deletions2 += diffs[pointer][1].length;
      }
      if (lastEquality && (lastEquality.length <= Math.max(length_insertions1, length_deletions1)) && (lastEquality.length <= Math.max(length_insertions2, length_deletions2))) {
        diffs.splice(equalities[equalities.length - 1], 0, [DIFF_DELETE, lastEquality]);
        diffs[equalities[equalities.length - 1] + 1][0] = DIFF_INSERT;
        equalities.pop();
        if (equalities.length) {
          equalities.pop();
        }
        pointer = equalities.length ? equalities[equalities.length - 1] : -1;
        length_insertions1 = 0;
        length_deletions1 = 0;
        length_insertions2 = 0;
        length_deletions2 = 0;
        lastEquality = null;
        changes = true;
      }
    }
    pointer++;
  }
  if (changes) {
    this.diff_cleanupMerge(diffs);
  }
  this.diff_cleanupSemanticLossless(diffs);
};

diff_match_patch.prototype.diff_cleanupSemanticLossless = function(diffs) {
  function diff_cleanupSemanticScore_(one, two) {
    if (!one || !two) { return 6; }
    var char1 = one.charAt(one.length - 1);
    var char2 = two.charAt(0);
    var nonAlphaNumeric1 = char1.match(/[^a-zA-Z0-9]/);
    var nonAlphaNumeric2 = char2.match(/[^a-zA-Z0-9]/);
    var whitespace1 = nonAlphaNumeric1 && char1.match(/\s/);
    var whitespace2 = nonAlphaNumeric2 && char2.match(/\s/);
    var lineBreak1 = whitespace1 && char1.match(/[\r\n]/);
    var lineBreak2 = whitespace2 && char2.match(/[\r\n]/);
    var blankLine1 = lineBreak1 && one.match(/\n\r?\n$/);
    var blankLine2 = lineBreak2 && two.match(/^\r?\n\r?\n/);
    if (blankLine1 || blankLine2) { return 5; }
    if (lineBreak1 || lineBreak2) { return 4; }
    if (nonAlphaNumeric1 && !whitespace1 && whitespace2) { return 3; }
    if (whitespace1 || whitespace2) { return 2; }
    if (nonAlphaNumeric1 || nonAlphaNumeric2) { return 1; }
    return 0;
  }
  var pointer = 1;
  while (pointer < diffs.length - 1) {
    if (diffs[pointer - 1][0] == DIFF_EQUAL && diffs[pointer + 1][0] == DIFF_EQUAL) {
      var equality1 = diffs[pointer - 1][1];
      var edit = diffs[pointer][1];
      var equality2 = diffs[pointer + 1][1];
      var commonOffset = this.diff_commonSuffix(equality1, edit).length;
      if (commonOffset) {
        var commonString = edit.substring(edit.length - commonOffset);
        equality1 = equality1.substring(0, equality1.length - commonOffset);
        edit = commonString + edit.substring(0, edit.length - commonOffset);
        equality2 = commonString + equality2;
      }
      var bestEquality1 = equality1;
      var bestEdit = edit;
      var bestEquality2 = equality2;
      var bestScore = diff_cleanupSemanticScore_(equality1, edit) + diff_cleanupSemanticScore_(edit, equality2);
      while (edit.charAt(0) === equality2.charAt(0)) {
        equality1 += edit.charAt(0);
        edit = edit.substring(1) + equality2.charAt(0);
        equality2 = equality2.substring(1);
        var score = diff_cleanupSemanticScore_(equality1, edit) + diff_cleanupSemanticScore_(edit, equality2);
        if (score >= bestScore) {
          bestScore = score;
          bestEquality1 = equality1;
          bestEdit = edit;
          bestEquality2 = equality2;
        }
      }
      if (diffs[pointer - 1][1] != bestEquality1) {
        if (bestEquality1) {
          diffs[pointer - 1][1] = bestEquality1;
        } else {
          diffs.splice(pointer - 1, 1);
          pointer--;
        }
        diffs[pointer][1] = bestEdit;
        if (bestEquality2) {
          diffs[pointer + 1][1] = bestEquality2;
        } else {
          diffs.splice(pointer + 1, 1);
          pointer--;
        }
      }
    }
    pointer++;
  }
};

diff_match_patch.prototype.diff_cleanupEfficiency = function(diffs) {
  var changes = false;
  var equalities = [];
  var lastEquality = '';
  var pointer = 0;
  var pre_ins = 0;
  var pre_del = 0;
  var post_ins = 0;
  var post_del = 0;
  while (pointer < diffs.length) {
    if (diffs[pointer][0] == DIFF_EQUAL) {
      if (diffs[pointer][1].length < this.Diff_EditCost && (post_ins || post_del)) {
        equalities.push(pointer);
        pre_ins = post_ins;
        pre_del = post_del;
        lastEquality = diffs[pointer][1];
      } else {
        equalities = [];
        lastEquality = '';
      }
      post_ins = 0;
      post_del = 0;
    } else {
      if (diffs[pointer][0] == DIFF_DELETE) {
        post_del += diffs[pointer][1].length;
      } else {
        post_ins += diffs[pointer][1].length;
      }
      if (lastEquality.length && (pre_ins + pre_del > post_ins + post_del)) {
        var L = pre_ins + pre_del - (post_ins + post_del);
        if (L * 2 > lastEquality.length) {
          var i = -1, j = -1;
          for (var k = 0; k < equalities.length; k++) {
            var equality = diffs[equalities[k]][1];
            if (i == -1) i = diffs[equalities[k] - 1][1].length;
            if (j == -1) j = diffs[equalities[k] + 1][1].length;
          }
          if (L * 2 > i + j) {
            if (i > j) {
              var edit = diffs[equalities[0] - 1];
              edit[1] = edit[1].substring(0, edit[1].length - lastEquality.length)
            } else {
              var edit = diffs[equalities[0] + 1];
              edit[1] = lastEquality + edit[1];
            }
            diffs[equalities[0]][1] = '';
            changes = true;
          }
        }
      }
    }
    pointer++;
  }
  if (changes) {
    this.diff_cleanupMerge(diffs);
  }
};

diff_match_patch.prototype.diff_cleanupMerge = function(diffs) {
  diffs.push([DIFF_EQUAL, '']);
  var pointer = 0;
  var count_delete = 0;
  var count_insert = 0;
  var text_delete = '';
  var text_insert = '';
  while (pointer < diffs.length) {
    switch (diffs[pointer][0]) {
      case DIFF_INSERT:
        count_insert++;
        text_insert += diffs[pointer][1];
        pointer++;
        break;
      case DIFF_DELETE:
        count_delete++;
        text_delete += diffs[pointer][1];
        pointer++;
        break;
      case DIFF_EQUAL:
        if (count_delete + count_insert > 1) {
          if (count_delete !== 0 && count_insert !== 0) {
            var commonlength = this.diff_commonPrefix(text_insert, text_delete).length;
            if (commonlength !== 0) {
              if ((pointer - count_delete - count_insert) > 0 && diffs[pointer - count_delete - count_insert - 1][0] == DIFF_EQUAL) {
                diffs[pointer - count_delete - count_insert - 1][1] += text_insert.substring(0, commonlength);
              } else {
                diffs.splice(0, 0, [DIFF_EQUAL, text_insert.substring(0, commonlength)]);
                pointer++;
              }
              text_insert = text_insert.substring(commonlength);
              text_delete = text_delete.substring(commonlength);
            }
            commonlength = this.diff_commonSuffix(text_insert, text_delete).length;
            if (commonlength !== 0) {
              diffs[pointer][1] = text_insert.substring(text_insert.length - commonlength) + diffs[pointer][1];
              text_insert = text_insert.substring(0, text_insert.length - commonlength);
              text_delete = text_delete.substring(0, text_delete.length - commonlength);
            }
          }
          pointer -= count_delete + count_insert;
          diffs.splice(pointer, count_delete + count_insert);
          if (text_delete.length) {
            diffs.splice(pointer, 0, [DIFF_DELETE, text_delete]);
            pointer++;
          }
          if (text_insert.length) {
            diffs.splice(pointer, 0, [DIFF_INSERT, text_insert]);
            pointer++;
          }
        } else if (pointer !== 0 && diffs[pointer - 1][0] == DIFF_EQUAL) {
          diffs[pointer - 1][1] += diffs[pointer][1];
          diffs.splice(pointer, 1);
        } else {
          pointer++;
        }
        count_insert = 0;
        count_delete = 0;
        text_delete = '';
        text_insert = '';
        break;
    }
  }
  if (diffs[diffs.length - 1][1] === '') {
    diffs.pop();
  }
  var changes = false;
  pointer = 1;
  while (pointer < diffs.length - 1) {
    if (diffs[pointer - 1][0] == DIFF_EQUAL && diffs[pointer + 1][0] == DIFF_EQUAL) {
      if (diffs[pointer][1].substring(diffs[pointer][1].length - diffs[pointer - 1][1].length) == diffs[pointer - 1][1]) {
        diffs[pointer][1] = diffs[pointer - 1][1] + diffs[pointer][1].substring(0, diffs[pointer][1].length - diffs[pointer - 1][1].length);
        diffs[pointer + 1][1] = diffs[pointer - 1][1] + diffs[pointer + 1][1];
        diffs.splice(pointer - 1, 1);
        changes = true;
      } else if (diffs[pointer][1].substring(0, diffs[pointer + 1][1].length) == diffs[pointer + 1][1]) {
        diffs[pointer - 1][1] += diffs[pointer + 1][1];
        diffs[pointer][1] = diffs[pointer][1].substring(diffs[pointer + 1][1].length) + diffs[pointer + 1][1];
        diffs.splice(pointer + 1, 1);
        changes = true;
      }
    }
    pointer++;
  }
  if (changes) {
    this.diff_cleanupMerge(diffs);
  }
};

diff_match_patch.patch_obj = function() {
  this.diffs = [];
  this.start1 = null;
  this.start2 = null;
  this.length1 = 0;
  this.length2 = 0;
};

diff_match_patch.patch_obj.prototype.toString = function() {
  var coords1, coords2;
  coords1 = this.length1 === 0 ? this.start1 + ',0' : this.length1 == 1 ? this.start1 + 1 : (this.start1 + 1) + ',' + this.length1;
  coords2 = this.length2 === 0 ? this.start2 + ',0' : this.length2 == 1 ? this.start2 + 1 : (this.start2 + 1) + ',' + this.length2;
  var text = '@@ -' + coords1 + ' +' + coords2 + ' @@\n';
  for (var i = 0; i < this.diffs.length; i++) {
    switch (this.diffs[i][0]) {
      case DIFF_INSERT:
        text += '+' + encodeURI(this.diffs[i][1]) + '\n';
        break;
      case DIFF_DELETE:
        text += '-' + encodeURI(this.diffs[i][1]) + '\n';
        break;
      case DIFF_EQUAL:
        text += ' ' + encodeURI(this.diffs[i][1]) + '\n';
        break;
    }
  }
  return text.replace(/%20/g, ' ');
};

diff_match_patch.prototype.patch_toText = function(patches) {
  var text = [];
  for (var i = 0; i < patches.length; i++) {
    text[i] = patches[i];
  }
  return text.join('');
};

diff_match_patch.prototype.patch_fromText = function(textline) {
  var patches = [];
  if (!textline) {
    return patches;
  }
  var text = textline.split('\n');
  var textPointer = 0;
  var patchHeader = /^@@ -(\d+),?(\d*) \+(\d+),?(\d*) @@$/;
  while (textPointer < text.length) {
    var m = text[textPointer].match(patchHeader);
    if (!m) { throw new Error('Invalid patch string: ' + text[textPointer]); }
    var patch = new diff_match_patch.patch_obj();
    patches.push(patch);
    patch.start1 = parseInt(m[1], 10);
    if (m[2] === '') {
      patch.start1--;
      patch.length1 = 1;
    } else if (m[2] == '0') {
      patch.length1 = 0;
    } else {
      patch.start1--;
      patch.length1 = parseInt(m[2], 10);
    }
    patch.start2 = parseInt(m[3], 10);
    if (m[4] === '') {
      patch.start2--;
      patch.length2 = 1;
    } else if (m[4] == '0') {
      patch.length2 = 0;
    } else {
      patch.start2--;
      patch.length2 = parseInt(m[4], 10);
    }
    textPointer++;
    while (textPointer < text.length) {
      var sign = text[textPointer].charAt(0);
      try {
        var line = decodeURI(text[textPointer].substring(1));
      } catch (ex) {
        throw new Error('Illegal escape in patch_fromText: ' + line);
      }
      if (sign == '-') {
        patch.diffs.push([DIFF_DELETE, line]);
      } else if (sign == '+') {
        patch.diffs.push([DIFF_INSERT, line]);
      } else if (sign == ' ') {
        patch.diffs.push([DIFF_EQUAL, line]);
      } else if (sign == '@') {
        break;
      } else if (sign !== '') {
        throw new Error('Invalid patch mode "' + sign + '" in: ' + line);
      }
      textPointer++;
    }
  }
  return patches;
};

diff_match_patch.prototype.patch_make = function(a, opt_b) {
  var text1, diffs;
  if (typeof a == 'string' && typeof opt_b == 'string') {
    text1 = a;
    diffs = this.diff_main(text1, opt_b, true);
    if (diffs.length > 2) {
      this.diff_cleanupSemantic(diffs);
      this.diff_cleanupEfficiency(diffs);
    }
  } else if (a && typeof a == 'object' && typeof opt_b == 'undefined') {
    diffs = a;
    text1 = this.diff_text1(diffs);
  } else if (typeof a == 'string' && opt_b && typeof opt_b == 'object') {
    text1 = a;
    diffs = opt_b;
  } else {
    throw new Error('Unknown call format to patch_make.');
  }
  if (diffs.length === 0) { return []; }
  var patches = [];
  var patch = new diff_match_patch.patch_obj();
  var char_count1 = 0;
  var char_count2 = 0;
  var prepatch_text = text1;
  for (var i = 0; i < diffs.length; i++) {
    var diff_type = diffs[i][0];
    var diff_text = diffs[i][1];
    if (!patch.diffs.length && diff_type != DIFF_EQUAL) {
      patch.start1 = char_count1;
      patch.start2 = char_count2;
    }
    switch (diff_type) {
      case DIFF_INSERT:
        patch.diffs.push(diffs[i]);
        patch.length2 += diff_text.length;
        break;
      case DIFF_DELETE:
        patch.length1 += diff_text.length;
        patch.diffs.push(diffs[i]);
        break;
      case DIFF_EQUAL:
        if (diff_text.length <= 2 * this.Patch_Margin && patch.diffs.length !== 0 && diffs.length != i + 1) {
          patch.diffs.push(diffs[i]);
          patch.length1 += diff_text.length;
          patch.length2 += diff_text.length;
        }
        if (diff_text.length >= 2 * this.Patch_Margin) {
          if (patch.diffs.length) {
            this.patch_addContext_(patch, prepatch_text);
            patches.push(patch);
            patch = new diff_match_patch.patch_obj();
          }
          prepatch_text = this.diff_text2(diffs.slice(0, i + 1));
          char_count1 = prepatch_text.length;
          char_count2 = prepatch_text.length;
        }
        break;
    }
    if (diff_type !== DIFF_INSERT) {
      char_count1 += diff_text.length;
    }
    if (diff_type !== DIFF_DELETE) {
      char_count2 += diff_text.length;
    }
  }
  if (patch.diffs.length) {
    this.patch_addContext_(patch, prepatch_text);
    patches.push(patch);
  }
  return patches;
};

diff_match_patch.prototype.patch_deepCopy = function(patches) {
  var patches_copy = [];
  for (var i = 0; i < patches.length; i++) {
    var patch = patches[i];
    var patch_copy = new diff_match_patch.patch_obj();
    patch_copy.diffs = [];
    for (var j = 0; j < patch.diffs.length; j++) {
      patch_copy.diffs[j] = patch.diffs[j].slice();
    }
    patch_copy.start1 = patch.start1;
    patch_copy.start2 = patch.start2;
    patch_copy.length1 = patch.length1;
    patch_copy.length2 = patch.length2;
    patches_copy[i] = patch_copy;
  }
  return patches_copy;
};

diff_match_patch.prototype.patch_apply = function(patches, text) {
  if (patches.length == 0) { return [text, []]; }
  patches = this.patch_deepCopy(patches);
  var nullPadding = this.patch_addPadding(patches);
  text = nullPadding + text + nullPadding;
  this.patch_splitMax(patches);
  var delta = 0;
  var results = [];
  for (var i = 0; i < patches.length; i++) {
    var expected_loc = patches[i].start2 + delta;
    var text1 = this.diff_text1(patches[i].diffs);
    var start_loc = -1;
    if (text1.length > this.Match_MaxBits) {
      var end_loc = -1;
      start_loc = this.match_main(text, text1.substring(0, this.Match_MaxBits), expected_loc);
      if (start_loc != -1) {
        end_loc = this.match_main(text, text1.substring(text1.length - this.Match_MaxBits), expected_loc + text1.length - this.Match_MaxBits);
        if (start_loc == -1 || end_loc == -1) {
          start_loc = -1;
        }
      }
    } else {
      start_loc = this.match_main(text, text1, expected_loc);
    }
    if (start_loc == -1) {
      results[i] = false;
      delta -= patches[i].length2 - patches[i].length1;
    } else {
      results[i] = true;
      delta = start_loc - expected_loc;
      var text2 = text.substring(start_loc, start_loc + text1.length);
      if (text1 == text2) {
        text = text.substring(0, start_loc) + this.diff_text2(patches[i].diffs) + text.substring(start_loc + text1.length);
      } else {
        var diffs = this.diff_main(text1, text2, false);
        if (text1.length > this.Match_MaxBits && this.diff_levenshtein(diffs) / text1.length > this.Patch_DeleteThreshold) {
          results[i] = false;
        } else {
          this.diff_cleanupSemanticLossless(diffs);
          var index1 = 0;
          for (var j = 0; j < patches[i].diffs.length; j++) {
            var mod = patches[i].diffs[j];
            if (mod[0] !== DIFF_EQUAL) {
              var index2 = this.diff_xIndex(diffs, index1);
            }
            if (mod[0] === DIFF_INSERT) {
              text = text.substring(0, start_loc + index2) + mod[1] + text.substring(start_loc + index2);
            } else if (mod[0] === DIFF_DELETE) {
              text = text.substring(0, start_loc + index2) + text.substring(start_loc + this.diff_xIndex(diffs, index1 + mod[1].length));
            }
            if (mod[0] !== DIFF_DELETE) {
              index1 += mod[1].length;
            }
          }
        }
      }
    }
  }
  text = text.substring(nullPadding.length, text.length - nullPadding.length);
  return [text, results];
};

diff_match_patch.prototype.patch_addPadding = function(patches) {
  var paddingLength = this.Patch_Margin;
  var nullPadding = '';
  for (var x = 1; x <= paddingLength; x++) {
    nullPadding += String.fromCharCode(x);
  }
  for (var i = 0; i < patches.length; i++) {
    patches[i].start1 += paddingLength;
    patches[i].start2 += paddingLength;
  }
  var patch = patches[0];
  var diffs = patch.diffs;
  if (diffs.length == 0 || diffs[0][0] != DIFF_EQUAL) {
    diffs.unshift([DIFF_EQUAL, nullPadding]);
    patch.start1 -= paddingLength;
    patch.start2 -= paddingLength;
    patch.length1 += paddingLength;
    patch.length2 += paddingLength;
  } else if (paddingLength > diffs[0][1].length) {
    var extraLength = paddingLength - diffs[0][1].length;
    diffs[0][1] = nullPadding.substring(diffs[0][1].length) + diffs[0][1];
    patch.start1 -= extraLength;
    patch.start2 -= extraLength;
    patch.length1 += extraLength;
    patch.length2 += extraLength;
  }
  patch = patches[patches.length - 1];
  diffs = patch.diffs;
  if (diffs.length == 0 || diffs[diffs.length - 1][0] != DIFF_EQUAL) {
    diffs.push([DIFF_EQUAL, nullPadding]);
    patch.length1 += paddingLength;
    patch.length2 += paddingLength;
  } else if (paddingLength > diffs[diffs.length - 1][1].length) {
    var extraLength = paddingLength - diffs[diffs.length - 1][1].length;
    diffs[diffs.length - 1][1] += nullPadding.substring(0, extraLength);
    patch.length1 += extraLength;
    patch.length2 += extraLength;
  }
  return nullPadding;
};

diff_match_patch.prototype.patch_splitMax = function(patches) {
  var patch_size = this.Match_MaxBits;
  for (var i = 0; i < patches.length; i++) {
    if (patches[i].length1 <= patch_size) {
      continue;
    }
    var bigpatch = patches[i];
    patches.splice(i--, 1);
    var start1 = bigpatch.start1;
    var start2 = bigpatch.start2;
    var precontext = '';
    while (bigpatch.diffs.length !== 0) {
      var patch = new diff_match_patch.patch_obj();
      var empty = true;
      patch.start1 = start1 - precontext.length;
      patch.start2 = start2 - precontext.length;
      if (precontext !== '') {
        patch.length1 = patch.length2 = precontext.length;
        patch.diffs.push([DIFF_EQUAL, precontext]);
      }
      while (bigpatch.diffs.length !== 0 && patch.length1 < patch_size - this.Patch_Margin) {
        var diff_type = bigpatch.diffs[0][0];
        var diff_text = bigpatch.diffs[0][1];
        if (diff_type === DIFF_INSERT) {
          patch.length2 += diff_text.length;
          start2 += diff_text.length;
          patch.diffs.push(bigpatch.diffs.shift());
          empty = false;
        } else if (diff_type === DIFF_DELETE && patch.diffs.length == 1 && patch.diffs[0][0] == DIFF_EQUAL && diff_text.length > 2 * patch_size) {
          patch.length1 += diff_text.length;
          start1 += diff_text.length;
          empty = false;
          patch.diffs.push([diff_type, diff_text]);
          bigpatch.diffs.shift();
        } else {
          diff_text = diff_text.substring(0, patch_size - patch.length1 - this.Patch_Margin);
          patch.length1 += diff_text.length;
          start1 += diff_text.length;
          if (diff_type === DIFF_EQUAL) {
            patch.length2 += diff_text.length;
            start2 += diff_text.length;
          } else {
            empty = false;
          }
          patch.diffs.push([diff_type, diff_text]);
          if (diff_text == bigpatch.diffs[0][1]) {
            bigpatch.diffs.shift();
          } else {
            bigpatch.diffs[0][1] = bigpatch.diffs[0][1].substring(diff_text.length);
          }
        }
      }
      precontext = this.diff_text2(patch.diffs);
      precontext = precontext.substring(precontext.length - this.Patch_Margin);
      var postcontext = this.diff_text1(bigpatch.diffs).substring(0, this.Patch_Margin);
      if (postcontext !== '') {
        patch.length1 += postcontext.length;
        patch.length2 += postcontext.length;
        if (patch.diffs.length !== 0 && patch.diffs[patch.diffs.length - 1][0] === DIFF_EQUAL) {
          patch.diffs[patch.diffs.length - 1][1] += postcontext;
        } else {
          patch.diffs.push([DIFF_EQUAL, postcontext]);
        }
      }
      if (!empty) {
        patches.splice(++i, 0, patch);
      }
    }
  }
};

diff_match_patch.prototype.patch_addContext_ = function(patch, text) {
  if (text.length == 0) { return; }
  var pattern = text.substring(patch.start2, patch.start2 + patch.length1);
  var padding = 0;
  while (text.indexOf(pattern) != text.lastIndexOf(pattern) && pattern.length < this.Match_MaxBits - this.Patch_Margin - this.Patch_Margin) {
    padding += this.Patch_Margin;
    pattern = text.substring(Math.max(0, patch.start2 - padding), patch.start2 + patch.length1 + padding);
  }
  padding += this.Patch_Margin;
  var prefix = text.substring(Math.max(0, patch.start2 - padding), patch.start2);
  if (prefix) {
    patch.diffs.unshift([DIFF_EQUAL, prefix]);
  }
  var suffix = text.substring(patch.start2 + patch.length1, patch.start2 + patch.length1 + padding);
  if (suffix) {
    patch.diffs.push([DIFF_EQUAL, suffix]);
  }
  patch.start1 -= prefix.length;
  patch.start2 -= prefix.length;
  patch.length1 += prefix.length + suffix.length;
  patch.length2 += prefix.length + suffix.length;
};

diff_match_patch.prototype.diff_text1 = function(diffs) {
  var text = [];
  for (var i = 0; i < diffs.length; i++) {
    if (diffs[i][0] !== DIFF_INSERT) {
      text[i] = diffs[i][1];
    }
  }
  return text.join('');
};

diff_match_patch.prototype.diff_text2 = function(diffs) {
  var text = [];
  for (var i = 0; i < diffs.length; i++) {
    if (diffs[i][0] !== DIFF_DELETE) {
      text[i] = diffs[i][1];
    }
  }
  return text.join('');
};

diff_match_patch.prototype.diff_levenshtein = function(diffs) {
  var levenshtein = 0;
  var insertions = 0;
  var deletions = 0;
  for (var i = 0; i < diffs.length; i++) {
    var op = diffs[i][0];
    var data = diffs[i][1];
    switch (op) {
      case DIFF_INSERT:
        insertions += data.length;
        break;
      case DIFF_DELETE:
        deletions += data.length;
        break;
      case DIFF_EQUAL:
        levenshtein += Math.max(insertions, deletions);
        insertions = 0;
        deletions = 0;
        break;
    }
  }
  levenshtein += Math.max(insertions, deletions);
  return levenshtein;
};

diff_match_patch.prototype.diff_xIndex = function(diffs, loc) {
  var chars1 = 0;
  var chars2 = 0;
  var last_chars1 = 0;
  var last_chars2 = 0;
  for (var x = 0; x < diffs.length; x++) {
    if (diffs[x][0] !== DIFF_INSERT) {
      chars1 += diffs[x][1].length;
    }
    if (diffs[x][0] !== DIFF_DELETE) {
      chars2 += diffs[x][1].length;
    }
    if (chars1 > loc) { break; }
    last_chars1 = chars1;
    last_chars2 = chars2;
  }
  if (diffs.length != x && diffs[x][0] === DIFF_DELETE) {
    return last_chars2;
  }
  return last_chars2 + (loc - last_chars1);
};

diff_match_patch.prototype.match_main = function(text, pattern, loc) {
  loc = Math.max(0, Math.min(loc, text.length));
  if (text == pattern) { return 0; }
  if (!text.length) { return -1; }
  if (text.substring(loc, loc + pattern.length) == pattern) { return loc; }
  return this.match_bitap_(text, pattern, loc);
};

diff_match_patch.prototype.match_bitap_ = function(text, pattern, loc) {
  if (pattern.length > this.Match_MaxBits) { throw new Error('Pattern too long for this browser.'); }
  var s = this.match_alphabet_(pattern);
  var dmp = this;
  function match_bitapScore_(e, x) {
    var accuracy = e / pattern.length;
    var proximity = Math.abs(loc - x);
    if (!dmp.Match_Distance) {
      return proximity ? 1.0 : accuracy;
    }
    return accuracy + (proximity / dmp.Match_Distance);
  }
  var score_threshold = this.Match_Threshold;
  var best_loc = text.indexOf(pattern, loc);
  if (best_loc != -1) {
    score_threshold = Math.min(match_bitapScore_(0, best_loc), score_threshold);
    best_loc = text.lastIndexOf(pattern, loc + pattern.length);
    if (best_loc != -1) {
      score_threshold = Math.min(match_bitapScore_(0, best_loc), score_threshold);
    }
  }
  var matchmask = 1 << (pattern.length - 1);
  best_loc = -1;
  var bin_min, bin_mid;
  var bin_max = pattern.length + text.length;
  var last_rd;
  for (var d = 0; d < pattern.length; d++) {
    bin_min = 0;
    bin_mid = bin_max;
    while (bin_min < bin_mid) {
      if (match_bitapScore_(d, loc + bin_mid) <= score_threshold) {
        bin_min = bin_mid;
      } else {
        bin_max = bin_mid;
      }
      bin_mid = Math.floor((bin_max - bin_min) / 2 + bin_min);
    }
    bin_max = bin_mid;
    var start = Math.max(1, loc - bin_mid + 1);
    var finish = Math.min(loc + bin_mid, text.length) + pattern.length;
    var rd = Array(finish + 2);
    rd[finish + 1] = (1 << d) - 1;
    for (var j = finish; j >= start; j--) {
      var charMatch;
      if (text.length <= j - 1 || !s.hasOwnProperty(text.charAt(j - 1))) {
        charMatch = 0;
      } else {
        charMatch = s[text.charAt(j - 1)];
      }
      if (d === 0) {
        rd[j] = ((rd[j + 1] << 1) | 1) & charMatch;
      } else {
        rd[j] = (((rd[j + 1] << 1) | 1) & charMatch) | (((last_rd[j + 1] | last_rd[j]) << 1) | 1) | last_rd[j + 1];
      }
      if (rd[j] & matchmask) {
        var score = match_bitapScore_(d, j - 1);
        if (score <= score_threshold) {
          score_threshold = score;
          best_loc = j - 1;
          if (best_loc > loc) {
            start = Math.max(1, 2 * loc - best_loc);
          } else {
            break;
          }
        }
      }
    }
    if (match_bitapScore_(d + 1, loc) > score_threshold) { break; }
    last_rd = rd;
  }
  return best_loc;
};

diff_match_patch.prototype.match_alphabet_ = function(pattern) {
  var s = {};
  for (var i = 0; i < pattern.length; i++) {
    s[pattern.charAt(i)] = 0;
  }
  for (var i = 0; i < pattern.length; i++) {
    s[pattern.charAt(i)] |= 1 << (pattern.length - i - 1);
  }
  return s;
};

// --- END of Diff-Match-Patch Library ---
return { diff_match_patch };
```





# LoadScript

```jsx
/**
 * Loads a script either from a URL (with caching) or a local vault path.
 * In a Datacore component context, this function requires the `dc` object
 * to access the vault's file system adapter for caching.
 *
 * @param {object} dc - The Datacore context object.
 * @param {string} src - The URL or local vault path of the script.
 * @param {Function} [onload] - Optional callback function to execute when the script loads successfully.
 * @param {Function} [onerror] - Optional callback function to execute if loading fails.
 * @returns {Promise<HTMLScriptElement>} A promise that resolves with the script element when loaded, or rejects on error.
 */
async function loadScript(dc, src, onload, onerror) {
  // Define a cache directory within Obsidian's hidden folder structure
  const cacheDir = ".datacore/script_cache";
  // Simple check for URL format
  const isUrl = /^https?:\/\//.test(src);

  // --- Helper Function to Execute Script Content ---
  const executeScriptContent = (scriptContent, resolve, reject, scriptElement) => {
    try {
      scriptElement.textContent = scriptContent;
      document.body.appendChild(scriptElement);
      console.log(`Script executed from ${isUrl ? 'cache/network' : 'local path'}: ${src}`);
      if (onload) {
        onload();
      }
      resolve(scriptElement);
    } catch (execError) {
      console.error(`Error executing script content from ${src}:`, execError);
      if (onerror) {
        onerror(execError);
      }
      reject(execError);
    }
  };

  return new Promise(async (resolve, reject) => {
    const scriptElement = document.createElement("script");
    scriptElement.async = true;

    // **CHANGE**: Get the adapter from the `dc` object, not the global `app`.
    if (!dc || !dc.app || !dc.app.vault || !dc.app.vault.adapter) {
        return reject(new Error("Datacore context 'dc' with vault adapter is required for loadScript."));
    }
    const adapter = dc.app.vault.adapter;

    try {
      if (isUrl) {
        // --- URL Handling (Fetch & Cache) ---
        const safeFilename = src
          .replace(/^https?:\/\//, '')
          .replace(/[\/\\?%*:|"<>]/g, '_') + ".js";
        const cachePath = `${cacheDir}/${safeFilename}`;

        let scriptText = null;

        // 1. Check if the cached file exists
        const cachedExists = await adapter.exists(cachePath);

        if (cachedExists) {
          console.log(`Loading script from cache: ${cachePath}`);
          try {
            scriptText = await adapter.read(cachePath);
          } catch (readError) {
            console.warn(`Failed to read cache file ${cachePath}, attempting refetch. Error:`, readError);
          }
        }

        if (scriptText === null) {
          console.log(`Fetching script from network: ${src}`);
          const response = await fetch(src);

          if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status} for ${src}`);
          }
          scriptText = await response.text();

          // 3. Write to cache
          try {
            if (!(await adapter.exists(cacheDir))) {
              console.log(`Creating script cache directory: ${cacheDir}`);
              await adapter.mkdir(cacheDir);
            }
            console.log(`Writing script to cache: ${cachePath}`);
            await adapter.write(cachePath, scriptText);
          } catch (writeError) {
            console.warn(`Failed to write script to cache ${cachePath}. Error:`, writeError);
          }
        }
        executeScriptContent(scriptText, resolve, reject, scriptElement);

      } else {
        // --- Local Vault Path Handling ---
        console.log(`Loading script from local vault path: ${src}`);
        const localFileExists = await adapter.exists(src);

        if (!localFileExists) {
           throw new Error(`Local script file not found: ${src}`);
        }

        const scriptText = await adapter.read(src);
        executeScriptContent(scriptText, resolve, reject, scriptElement);
      }
    } catch (error) {
      // --- General Error Handling ---
      console.error(`Failed to load script ${src}:`, error);
      if (scriptElement.parentNode) {
        scriptElement.parentNode.removeChild(scriptElement);
      }
      if (onerror) {
        onerror(error);
      }
      reject(error);
    }
  });
}

return { loadScript };
```




# Test

```jsx
const { WorldView } = await dc.require(dc.headerLink("B25MovingCat.component.v3.md", "ViewComponent"));
return <WorldView />;

test11111

```

