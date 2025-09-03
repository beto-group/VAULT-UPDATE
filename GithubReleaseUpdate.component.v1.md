

# ViewComponent

```jsx
const { useEffect, useRef, useState } = dc;

// --- UTILITY FUNCTIONS (Core logic, no need to edit) ---
function findNearestAncestorWithClass(element, className) { if (!element) return null; let current = element.parentNode; while (current) { if (current.classList && current.classList.contains(className)) { return current; } current = current.parentNode; } return null; }
function findDirectChildByClass(parent, className) { if (!parent) return null; for (const child of parent.children) { if (child.classList && child.classList.contains(className)) { return child; } } return null; }

// Icon Data & AnimatedIcon Component (Unchanged)
const ICONS = [{ name: "Notification Bell", svg: `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path class="svg-elem-1" d="M18 8A6 6 0 0 0 6 8c0 7-3 9-3 9h18s-3-2-3-9"></path><path class="svg-elem-2" d="M13.73 21a2 2 0 0 1-3.46 0"></path><path class="svg-elem-3" d="M19 8a3 3 0 0 0-6 0"></path></svg>` }];
const AnimatedIcon = ({ svgString, isActive, isInView }) => { const iconRef = useRef(null); const intervalRef = useRef(null); const timeoutRef = useRef(null); const pathsRef = useRef([]); const [hasRevealed, setHasRevealed] = useState(false); const DURATION = 1.0; useEffect(() => { const container = iconRef.current; if (!container || !svgString) return; container.innerHTML = svgString; const svgElement = container.querySelector('svg'); if (!svgElement) return; pathsRef.current = svgElement.querySelectorAll('[class*="svg-elem-"]'); pathsRef.current.forEach((path, index) => { const delay = 0.1 * index; const length = path.getTotalLength(); if (length > 0) { path.style.strokeDasharray = length; path.style.strokeDashoffset = length; path.style.stroke = 'var(--text-normal)'; path.style.strokeWidth = '1.5px'; path.style.fill = 'transparent'; path.style.transition = `stroke-dashoffset ${DURATION}s ease ${delay}s, fill ${DURATION * 0.7}s ease ${delay + (DURATION * 0.2)}s`; } }); }, [svgString]); useEffect(() => { if (isInView && !hasRevealed) { const paths = pathsRef.current; if (!paths || paths.length === 0) return; const maxDelay = 0.1 * (paths.length - 1); const totalRevealTime = (DURATION + maxDelay) * 1000; paths.forEach(path => { path.style.strokeDashoffset = '0'; path.style.fill = 'var(--interactive-accent-tint)'; }); setTimeout(() => setHasRevealed(true), totalRevealTime); } }, [isInView, hasRevealed]); useEffect(() => { if (!hasRevealed) return; const paths = pathsRef.current; if (!paths || paths.length === 0) return; const maxDelay = 0.1 * (paths.length - 1); const totalAnimationTime = (DURATION + maxDelay) * 1000; const runAnimationCycle = () => { paths.forEach(path => { path.style.strokeDashoffset = path.getTotalLength(); path.style.fill = 'transparent'; }); timeoutRef.current = setTimeout(() => { paths.forEach(path => { path.style.strokeDashoffset = '0'; path.style.fill = 'var(--interactive-accent-tint)'; }); }, totalAnimationTime * 0.88); }; if (isActive) { runAnimationCycle(); intervalRef.current = setInterval(runAnimationCycle, totalAnimationTime * 2); } else { paths.forEach(path => { path.style.strokeDashoffset = '0'; path.style.fill = 'var(--interactive-accent-tint)'; }); } return () => { clearInterval(intervalRef.current); clearTimeout(timeoutRef.current); }; }, [isActive, hasRevealed]); return <div ref={iconRef} style={{ width: '100%', height: '100%' }} />; };

// A simple utility to simulate network delay
const sleep = (ms) => new Promise(resolve => setTimeout(resolve, ms));


// =================================================================================
// --- 3. UPDATER LOGIC (Scaffolding for GitHub update process) ---
// =================================================================================

/**
 * Simulates downloading and unzipping the latest version from a GitHub repo.
 * @returns {Promise<Array<{path: string, content: string}>>} A list of file objects.
 */
async function downloadLatestFromRepo() {
    await sleep(1200); // Simulate network delay
    // In a real app, this would fetch a zip, unzip it, and read the files.
    return [
        { path: 'README.md', content: 'This is the new version 2.0.' },
        { path: 'core/main.js', content: 'console.log("updated logic");' },
        { path: 'assets/icon.svg', content: '<svg>new icon</svg>' },
    ];
}

/**
 * Simulates getting the current state of files in the user's vault.
 * This represents the data you'd get by comparing `tree` output with the vault.
 * @returns {Promise<Array<{path: string, content: string}>>} A list of file objects.
 */
async function getCurrentVaultState() {
    await sleep(500); // Simulate reading local files
    return [
        // This file is outdated and will be updated.
        { path: 'README.md', content: 'This is the old version 1.0.' },
        // This file is unchanged in the new version, but the user modified it.
        { path: 'core/main.js', content: 'console.log("my custom logic");' },
        // This is a file the user created themselves.
        { path: 'my_notes/project_ideas.md', content: 'A great idea...' },
    ];
}

/**
 * Core comparison logic. Identifies new, updated, and user-modified files.
 * @param {Array} latestFiles - Files from the GitHub download.
 * @param {Array} currentFiles - Files currently in the user's vault.
 * @returns {{newFiles: Array, updatedFiles: Array, userModified: Array}}
 */
function compareVersions(latestFiles, currentFiles) {
    const latestFileMap = new Map(latestFiles.map(f => [f.path, f.content]));
    const currentFileMap = new Map(currentFiles.map(f => [f.path, f.content]));

    const newFiles = [];
    const updatedFiles = [];
    const userModified = [];

    // Check for new and updated files from the latest version
    for (const [path, content] of latestFileMap.entries()) {
        if (!currentFileMap.has(path)) {
            newFiles.push({ path });
        } else if (currentFileMap.get(path) !== content) {
            // It exists, but the content is different. This is a core update.
            updatedFiles.push({ path });
        }
    }
    
    // Check for user-added or user-modified files
    for (const [path, content] of currentFileMap.entries()) {
        if (!latestFileMap.has(path)) {
             // This file exists locally but not in the repo, so the user created it.
            userModified.push({ path, reason: 'User-created file' });
        }
        // This part is tricky. If a file is in `updatedFiles` but the user also
        // changed it, you might need a merging strategy. For now, we assume if the
        // repo version is different, it's an "update", and we'd overwrite.
        // A more advanced system would detect this as a conflict.
    }

    return { newFiles, updatedFiles, userModified };
}


// =================================================================================
// --- Main BasicView Component (Integrates the updater logic) ---
// =================================================================================
function BasicView() {
  const uniqueWrapperClass = "interactive-wrapper-" + useRef(Math.random().toString(36).substr(2, 9)).current;

  // STYLES object (condensed for brevity, unchanged)
  const STYLES = { injectedStyles: `.${uniqueWrapperClass}:hover .subtle-icon { opacity: 0.7; transform: scale(1); } .${uniqueWrapperClass} .promo-banner:hover { transform: scale(1.02); box-shadow: 0 0 90px -15px rgba(200, 160, 255, 0.4); } @keyframes fadeIn { from { opacity: 0 } to { opacity: 1 } } @keyframes scaleIn { from { transform: scale(.96); opacity: 0 } to { transform: scale(1); opacity: 1 } } .modal-fade-in { animation: fadeIn 0.4s cubic-bezier(0.25, 1, 0.5, 1); } .modal-scale-in { animation: scaleIn 0.4s cubic-bezier(0.25, 1, 0.5, 1); } @keyframes betReveal { 0% { opacity: 0; transform: scale(0.7) rotate(-10deg); } 70% { opacity: 1; transform: scale(1.1) rotate(5deg); } 100% { opacity: 1; transform: scale(1) rotate(0deg); } } .bet-reveal { animation: betReveal 0.6s cubic-bezier(0.25, 1, 0.5, 1) forwards; }`, fullTabWrapper: { position: 'relative', height: "100%", width: "100%", padding: "20px", boxSizing: "border-box", display: "flex", flexDirection: "column", alignItems: "center", justifyContent: "center", gap: "25px", backgroundColor: "var(--background-secondary)", border: "1px solid var(--background-modifier-border)", borderRadius: "8px", color: "var(--text-normal)", transition: "background-color 0.2s ease", }, icon: { position: "absolute", top: "15px", right: "20px", fontFamily: "monospace", fontSize: "14px", color: "var(--text-faint)", userSelect: "none", cursor: "pointer", opacity: 0, transform: "scale(0.9)", transition: "opacity 0.2s ease-in-out, transform 0.2s ease-in-out", zIndex: 10, }, title: { fontSize: "2em", fontWeight: "600", color: "var(--text-normal)" }, subtitle: { fontSize: "1em", color: "var(--text-muted)", maxWidth: "400px", textAlign: "center" }, promoBanner: { display: 'flex', alignItems: 'center', justifyContent: 'center', gap: '20px', padding: "20px 30px", borderRadius: "16px", width: 'min(100%, 500px)', background: 'rgba(24, 15, 28, 0.5)', border: '1px solid rgba(255, 255, 255, 0.1)', boxShadow: '0 0 80px -20px rgba(200, 160, 255, 0.3)', cursor: 'pointer', transition: 'transform 0.3s ease, box-shadow 0.3s ease', }, promoIconContainer: { width: '48px', height: '48px', flexShrink: 0, }, bannerTextContainer: { textAlign: 'left', }, bannerTitle: { margin: '0 0 5px 0', fontSize: '1.2em', fontWeight: 600, color: 'var(--text-normal)' }, bannerText: { margin: 0, color: 'var(--text-muted)', fontSize: '0.9em' }, modalOverlay: { position: 'fixed', inset: 0, display: 'flex', alignItems: 'center', justifyContent: 'center', backdropFilter: 'blur(12px) saturate(1.2)', zIndex: 9999, }, modalContent: { display: 'flex', flexDirection: 'column', gap: '15px', alignItems: 'center', width: 'min(100%, 95vw)', maxWidth: '450px', minHeight: '220px', justifyContent: 'center', padding: '30px', boxSizing: 'border-box', background: 'rgba(24, 15, 28, 0.7)', border: '1px solid rgba(255, 255, 255, 0.1)', borderRadius: '16px', boxShadow: '0 0 80px -20px rgba(200, 160, 255, 0.3)', overflow: 'hidden', textAlign: 'left' }, modalTitle: { margin: 0, fontSize: '1.5em', fontWeight: 600, color: 'var(--text-normal)', textAlign: 'center' }, modalText: { margin: '0 0 10px 0', color: 'var(--text-muted)', textAlign: 'center' }, hoverRevealText: { color: 'var(--text-faint)', fontSize: '12px', fontStyle: 'italic', height: '15px', textAlign: 'center', transition: 'opacity 0.3s ease, transform 0.3s ease', opacity: 0, transform: 'translateY(5px)', }, betText: { fontSize: '3.5em', fontWeight: 'bold', color: 'var(--text-normal)', userSelect: 'none', }, resultList: { listStyle: 'none', padding: 0, margin: '5px 0', fontSize: '14px' }, resultItem: { color: 'var(--text-muted)' }, resultItemNew: { color: 'var(--text-success)' }, resultItemUpdate: { color: 'var(--text-warning)' }, compactWrapper: { padding: "16px", boxSizing: "border-box", display: "flex", flexDirection: "column", alignItems: "center", justifyContent: "center", gap: "12px", border: "1px dashed var(--background-modifier-border)", borderRadius: "8px", backgroundColor: "var(--background-primary-alt)", }, compactText: { margin: 0, color: "var(--text-muted)", fontSize: "14px" }, buttonGroup: { display: "flex", gap: "10px", flexWrap: "wrap", justifyContent: "center" }, button: { padding: "8px 16px", fontSize: "12px", fontWeight: "500", color: "var(--text-on-accent)", backgroundColor: "var(--interactive-accent)", border: "none", borderRadius: "6px", cursor: "pointer", }, secondaryButton: { backgroundColor: "var(--background-modifier-hover)", color: "var(--text-muted)", }, };
  const CONTENT = { title: "FULL TAB VIEW", subtitle: "This component is in full-tab mode.", modalTitle: "Join the Inner Circle?", modalText: "Unlock monthly updates and arcane secrets.", };
  
  // --- STATE MANAGEMENT ---
  const [isFullTab, setIsFullTab] = useState(true);
  const [isModalOpen, setIsModalOpen] = useState(false);
  const [modalStage, setModalStage] = useState('confirmation'); // confirmation | bet | processing | results
  const [isConfirmHovered, setIsConfirmHovered] = useState(false);
  const [updateStatus, setUpdateStatus] = useState('idle'); // idle | downloading | comparing | success | error
  const [updateResult, setUpdateResult] = useState(null);
  
  const containerRef = useRef(null);
  const stateRefs = useRef({}).current;

  // Effects (Unchanged)
  useEffect(() => { const container = containerRef.current; if (!container) return; if (isFullTab) { if (!container.parentNode) { setTimeout(() => setIsFullTab(true), 50); return; } const targetPaneContent = findNearestAncestorWithClass(container, 'workspace-leaf-content'); if (!targetPaneContent) { console.error("[BasicView] Full tab mode failed."); setIsFullTab(false); return; } const contentWrapper = findDirectChildByClass(targetPaneContent, 'view-content') || targetPaneContent; stateRefs.originalParent = container.parentNode; stateRefs.placeholder = document.createElement('div'); stateRefs.placeholder.style.display = 'none'; container.parentNode.insertBefore(stateRefs.placeholder, container); const computedParentPosition = window.getComputedStyle(contentWrapper).position; stateRefs.parentPositionInfo = { element: contentWrapper, originalInlinePosition: contentWrapper.style.position }; if (computedParentPosition === 'static') { contentWrapper.style.position = "relative"; } contentWrapper.appendChild(container); Object.assign(container.style, { position: "absolute", top: "0px", left: "0px", width: "100%", height: "100%", zIndex: "9998", overflow: "auto" }); } return () => { if (!stateRefs.originalParent) return; if (stateRefs.placeholder?.parentNode) { stateRefs.placeholder.parentNode.replaceChild(container, stateRefs.placeholder); } else { stateRefs.originalParent.appendChild(container); } if (stateRefs.parentPositionInfo?.element) { stateRefs.parentPositionInfo.element.style.position = stateRefs.parentPositionInfo.originalInlinePosition || ''; } container.removeAttribute("style"); Object.keys(stateRefs).forEach(key => stateRefs[key] = null); }; }, [isFullTab]);
  useEffect(() => { if (!isModalOpen) return; const handleKeyDown = (event) => { if (event.key === 'Escape') { handleCloseModal(); } }; document.addEventListener('keydown', handleKeyDown); return () => document.removeEventListener('keydown', handleKeyDown); }, [isModalOpen]);

  // --- EVENT HANDLERS ---
  const handleOpenModal = () => { setModalStage('confirmation'); setIsModalOpen(true); };
  const handleCloseModal = () => setIsModalOpen(false);
  
  const handleConfirmSubscription = async () => {
    setModalStage('bet');
    await sleep(1000); // Let the "BET" animation be seen

    try {
        setModalStage('processing');
        
        setUpdateStatus('downloading');
        const latestFiles = await downloadLatestFromRepo();
        
        setUpdateStatus('comparing');
        const currentFiles = await getCurrentVaultState();
        
        const results = compareVersions(latestFiles, currentFiles);
        setUpdateResult(results);
        setUpdateStatus('success');
        setModalStage('results');
        
    } catch (error) {
        console.error("Update process failed:", error);
        setUpdateStatus('error');
    }
  };
  
  const handleExitFullTab = (e) => { e.stopPropagation(); setIsFullTab(false); };
  const handleEnterFullTab = () => setIsFullTab(true);
  const handleCopyPath = () => { try { const activeFile = dc.app.workspace.getActiveFile(); if (activeFile) { navigator.clipboard.writeText(activeFile.path); new Notice(`Path copied`, 3000); } else { new Notice("Could not determine path.", 3000); } } catch (error) { new Notice("Error copying path.", 3000); } };

  const renderModalContent = () => {
    switch (modalStage) {
        case 'confirmation': return (<>
            <h3 style={STYLES.modalTitle}>{CONTENT.modalTitle}</h3>
            <p style={STYLES.modalText}>{CONTENT.modalText}</p>
            <div style={STYLES.buttonGroup}>
                <button style={{...STYLES.button, ...STYLES.secondaryButton}} onClick={handleCloseModal}>Cancel</button>
                <button style={STYLES.button} onClick={handleConfirmSubscription} onMouseEnter={() => setIsConfirmHovered(true)} onMouseLeave={() => setIsConfirmHovered(false)}>Confirm</button>
            </div>
            <p style={{ ...STYLES.hoverRevealText, opacity: isConfirmHovered ? 1 : 0, transform: isConfirmHovered ? 'translateY(0)' : 'translateY(5px)' }}>bro you trust me ?</p>
        </>);
        case 'bet': return <div className="bet-reveal"><h1 style={STYLES.betText}>BET 🫡</h1></div>;
        case 'processing': return (<>
            <h3 style={STYLES.modalTitle}>Upgrading...</h3>
            <p style={STYLES.modalText}>{updateStatus === 'downloading' ? 'Downloading latest version...' : 'Comparing files...'}</p>
        </>);
        case 'results': return (<>
            <h3 style={STYLES.modalTitle}>Upgrade Complete</h3>
            <div>
                <p style={{...STYLES.bannerTitle, fontSize: '1em', margin: '0 0 5px 0' }}>New Files:</p>
                <ul style={STYLES.resultList}>{updateResult.newFiles.map(f => <li style={{...STYLES.resultItem, ...STYLES.resultItemNew}}>+ {f.path}</li>)}</ul>
                <p style={{...STYLES.bannerTitle, fontSize: '1em', margin: '10px 0 5px 0' }}>Updated Files:</p>
                <ul style={STYLES.resultList}>{updateResult.updatedFiles.map(f => <li style={{...STYLES.resultItem, ...STYLES.resultItemUpdate}}>~ {f.path}</li>)}</ul>
                <p style={{...STYLES.bannerTitle, fontSize: '1em', margin: '10px 0 5px 0' }}>User Files (Ignored):</p>
                <ul style={STYLES.resultList}>{updateResult.userModified.map(f => <li style={STYLES.resultItem}>  {f.path}</li>)}</ul>
            </div>
            <div style={{...STYLES.buttonGroup, marginTop: '15px'}}><button style={STYLES.button} onClick={handleCloseModal}>Finish</button></div>
        </>);
        default: return null;
    }
  }

  return (
    <div ref={containerRef}>
      <style>{STYLES.injectedStyles}</style>
      {isModalOpen && <div style={STYLES.modalOverlay} className="modal-fade-in"><div style={STYLES.modalContent} className="modal-scale-in">{renderModalContent()}</div></div>}
      {isFullTab ? (
        <div style={STYLES.fullTabWrapper} className={uniqueWrapperClass}>
          <span style={STYLES.icon} className="subtle-icon" title="Exit Full Tab" onClick={handleExitFullTab}>&lt;/&gt;</span>
          <h2 style={STYLES.title}>{CONTENT.title}</h2><p style={STYLES.subtitle}>{CONTENT.subtitle}</p>
          <div style={STYLES.promoBanner} className="promo-banner" onClick={handleOpenModal}>
            <div style={STYLES.promoIconContainer}><AnimatedIcon svgString={ICONS[0].svg} isActive={true} isInView={true} /></div>
            <div style={STYLES.bannerTextContainer}>
              <h3 style={STYLES.bannerTitle}>Subscribe to Monthly Upgrades</h3><p style={STYLES.bannerText}>Click here to get started.</p>
            </div>
          </div>
        </div>
      ) : (
        <div style={STYLES.compactWrapper}>
            <p style={STYLES.compactText}>Component is in compact mode.</p>
            <div style={STYLES.buttonGroup}>
                <button style={STYLES.button} onClick={handleEnterFullTab}>Enter Full Tab</button>
                <button style={{...STYLES.button, ...STYLES.secondaryButton}} onClick={handleCopyPath}>Find Codeblock</button>
            </div>
        </div>
      )}
    </div>
  );
};

return { BasicView };
```


