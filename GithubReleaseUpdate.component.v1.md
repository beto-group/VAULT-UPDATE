

# ViewComponent

```jsx
const { useEffect, useRef, useState } = dc;

// --- UTILITY FUNCTIONS & COMPONENTS (Unchanged) ---
function findNearestAncestorWithClass(element, className) { if (!element) return null; let current = element.parentNode; while (current) { if (current.classList && current.classList.contains(className)) { return current; } current = current.parentNode; } return null; }
function findDirectChildByClass(parent, className) { if (!parent) return null; for (const child of parent.children) { if (child.classList && child.classList.contains(className)) { return child; } } return null; }
const ICONS = [{ name: "Notification Bell", svg: `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path class="svg-elem-1" d="M18 8A6 6 0 0 0 6 8c0 7-3 9-3 9h18s-3-2-3-9"></path><path class="svg-elem-2" d="M13.73 21a2 2 0 0 1-3.46 0"></path><path class="svg-elem-3" d="M19 8a3 3 0 0 0-6 0"></path></svg>` }];
const AnimatedIcon = ({ svgString, isActive, isInView }) => { const iconRef = useRef(null); const intervalRef = useRef(null); const timeoutRef = useRef(null); const pathsRef = useRef([]); const [hasRevealed, setHasRevealed] = useState(false); const DURATION = 1.0; useEffect(() => { const container = iconRef.current; if (!container || !svgString) return; container.innerHTML = svgString; const svgElement = container.querySelector('svg'); if (!svgElement) return; pathsRef.current = svgElement.querySelectorAll('[class*="svg-elem-"]'); pathsRef.current.forEach((path, index) => { const delay = 0.1 * index; const length = path.getTotalLength(); if (length > 0) { path.style.strokeDasharray = length; path.style.strokeDashoffset = length; path.style.stroke = 'var(--text-normal)'; path.style.strokeWidth = '1.5px'; path.style.fill = 'transparent'; path.style.transition = `stroke-dashoffset ${DURATION}s ease ${delay}s, fill ${DURATION * 0.7}s ease ${delay + (DURATION * 0.2)}s`; } }); }, [svgString]); useEffect(() => { if (isInView && !hasRevealed) { const paths = pathsRef.current; if (!paths || paths.length === 0) return; const maxDelay = 0.1 * (paths.length - 1); const totalRevealTime = (DURATION + maxDelay) * 1000; paths.forEach(path => { path.style.strokeDashoffset = '0'; path.style.fill = 'var(--interactive-accent-tint)'; }); setTimeout(() => setHasRevealed(true), totalRevealTime); } }, [isInView, hasRevealed]); useEffect(() => { if (!hasRevealed) return; const paths = pathsRef.current; if (!paths || paths.length === 0) return; const maxDelay = 0.1 * (paths.length - 1); const totalAnimationTime = (DURATION + maxDelay) * 1000; const runAnimationCycle = () => { paths.forEach(path => { path.style.strokeDashoffset = path.getTotalLength(); path.style.fill = 'transparent'; }); timeoutRef.current = setTimeout(() => { paths.forEach(path => { path.style.strokeDashoffset = '0'; path.style.fill = 'var(--interactive-accent-tint)'; }); }, totalAnimationTime * 0.88); }; if (isActive) { runAnimationCycle(); intervalRef.current = setInterval(runAnimationCycle, totalAnimationTime * 2); } else { paths.forEach(path => { path.style.strokeDashoffset = '0'; path.style.fill = 'var(--interactive-accent-tint)'; }); } return () => { clearInterval(intervalRef.current); clearTimeout(timeoutRef.current); }; }, [isActive, hasRevealed]); return <div ref={iconRef} style={{ width: '100%', height: '100%' }} />; };
const sleep = (ms) => new Promise(resolve => setTimeout(resolve, ms));

// =================================================================================
// --- UPDATER LOGIC (With Cache-Busting) ---
// =================================================================================

const GITHUB_OWNER = 'beto-group';
const GITHUB_REPO = 'VAULT-UPDATE';
const GITHUB_BRANCH = 'main';
const LOG_PREFIX = '[VaultUpdater]';

function compareSemVer(a, b) { const partsA = a.split('.').map(Number); const partsB = b.split('.').map(Number); const len = Math.max(partsA.length, partsB.length); for (let i = 0; i < len; i++) { const numA = partsA[i] || 0; const numB = partsB[i] || 0; if (numA > numB) return 1; if (numA < numB) return -1; } return 0; }
function parseVersionFromYaml(markdownContent) { const yamlMatch = markdownContent.match(/^---\s*([\s\S]*?)\s*---/); if (!yamlMatch) return null; const yaml = yamlMatch[1]; const versionMatch = yaml.match(/^version:\s*["']?(.+?)["']?$/m); return versionMatch ? versionMatch[1] : null; }

async function checkForUpdates() {
    console.log(`${LOG_PREFIX} Starting update check...`);
    try {
        const remoteReadmeUrl = `https://raw.githubusercontent.com/${GITHUB_OWNER}/${GITHUB_REPO}/${GITHUB_BRANCH}/README.md?cache-bust=${new Date().getTime()}`;
        console.log(`${LOG_PREFIX} Fetching remote README from: ${remoteReadmeUrl}`);
        const response = await requestUrl({ url: remoteReadmeUrl, method: 'GET' });
        if (response.status !== 200) { throw new Error(`Failed to fetch README.md. Status: ${response.status}`); }
        const remoteContent = response.text;
        const remoteVersion = parseVersionFromYaml(remoteContent);
        if (!remoteVersion) throw new Error("Could not find version in remote README.md");
        let localVersion = '0.0.0';
        const localReadmeFile = dc.app.vault.getAbstractFileByPath('README.md');
        if (localReadmeFile) { localVersion = dc.app.metadataCache.getFileCache(localReadmeFile)?.frontmatter?.version || '0.0.0'; }
        const updateAvailable = compareSemVer(remoteVersion, localVersion) > 0;
        console.log(`${LOG_PREFIX} Comparison: remote='${remoteVersion}', local='${localVersion}'. Update available: ${updateAvailable}`);
        return { updateAvailable, remoteVersion, localVersion };
    } catch (error) {
        console.error(`${LOG_PREFIX} Error during update check:`, error);
        new Notice("Could not check for updates. See console for details.", 4000);
        return { updateAvailable: false, remoteVersion: 'unknown', localVersion: 'unknown' };
    }
}

async function downloadLatestFromRepo() {
    console.log(`${LOG_PREFIX} Starting download of latest version...`);
    const treeUrl = `https://api.github.com/repos/${GITHUB_OWNER}/${GITHUB_REPO}/git/trees/${GITHUB_BRANCH}?recursive=1&cache-bust=${new Date().getTime()}`;
    let treeData;
    try {
        const treeResponse = await requestUrl({ url: treeUrl, method: 'GET' });
        if (treeResponse.status !== 200) throw new Error(`Failed to fetch file tree. Status: ${treeResponse.status}`);
        treeData = treeResponse.json;
        if (!treeData || !treeData.tree) throw new Error("Invalid tree data from GitHub API.");
    } catch (e) {
        console.error(`${LOG_PREFIX} Failed to fetch repo file tree.`, e);
        throw new Error("Could not retrieve file list from repository.");
    }
    const filesToDownload = treeData.tree.filter(item => item.type === 'blob');
    const filePromises = filesToDownload.map(async (file) => {
        try {
            const contentUrl = `https://raw.githubusercontent.com/${GITHUB_OWNER}/${GITHUB_REPO}/${GITHUB_BRANCH}/${file.path}?cache-bust=${new Date().getTime()}`;
            const contentResponse = await requestUrl({ url: contentUrl, method: 'GET' });
            if (contentResponse.status !== 200) { console.error(`${LOG_PREFIX} Failed to download ${file.path}. Status: ${contentResponse.status}`); return null; }
            return { path: file.path, content: contentResponse.text };
        } catch (error) { console.error(`${LOG_PREFIX} Failed to download file: ${file.path}`, error); return null; }
    });
    const downloadedFiles = (await Promise.all(filePromises)).filter(Boolean);
    if (downloadedFiles.length !== filesToDownload.length) new Notice("Warning: Some files failed to download.", 4000);
    return downloadedFiles;
}

async function getCurrentVaultState(trackedFilePaths) { const vaultFiles = []; for (const path of trackedFilePaths) { try { const content = await dc.app.vault.adapter.read(path); vaultFiles.push({ path, content }); } catch (error) { /* File doesn't exist locally, which is fine */ } } return vaultFiles; }
function compareVersions(latestFiles, currentFiles) { const latestFileMap = new Map(latestFiles.map(f => [f.path, f.content])); const currentFileMap = new Map(currentFiles.map(f => [f.path, f.content])); const newFiles = []; const updatedFiles = []; for (const [path, content] of latestFileMap.entries()) { if (!currentFileMap.has(path)) { newFiles.push({ path, content }); } else if (currentFileMap.get(path) !== content) { updatedFiles.push({ path, content }); } } return { newFiles, updatedFiles, userModified: [] }; }

// =================================================================================
// --- Main View Component ---
// =================================================================================
function BasicView({ onReloadRequest }) { // <-- Accepts the reload function as a prop
  const uniqueWrapperClass = "interactive-wrapper-" + useRef(Math.random().toString(36).substr(2, 9)).current;
  const STYLES = {
    injectedStyles: `
      .${uniqueWrapperClass}:hover .subtle-icon { opacity: 0.7; transform: scale(1); }
      .${uniqueWrapperClass} .promo-banner:hover { transform: scale(1.02); box-shadow: 0 0 90px -15px rgba(200, 160, 255, 0.4); }
      .reload-button:hover { background-color: var(--background-modifier-hover); transform: scale(1.05); }
      .reload-button:active { transform: scale(0.95); }
      @keyframes fadeIn { from { opacity: 0 } to { opacity: 1 } }
      @keyframes scaleIn { from { transform: scale(.96); opacity: 0 } to { transform: scale(1); opacity: 1 } }
      .modal-fade-in { animation: fadeIn 0.4s cubic-bezier(0.25, 1, 0.5, 1); }
      .modal-scale-in { animation: scaleIn 0.4s cubic-bezier(0.25, 1, 0.5, 1); }
      @keyframes betReveal { 0% { opacity: 0; transform: scale(0.7) rotate(-10deg); } 70% { opacity: 1; transform: scale(1.1) rotate(5deg); } 100% { opacity: 1; transform: scale(1) rotate(0deg); } }
      .bet-reveal { animation: betReveal 0.6s cubic-bezier(0.25, 1, 0.5, 1) forwards; }`,
    fullTabWrapper: { position: 'relative', height: "100%", width: "100%", padding: "20px", boxSizing: "border-box", display: "flex", flexDirection: "column", alignItems: "center", justifyContent: "center", gap: "25px", backgroundColor: "var(--background-secondary)", border: "1px solid var(--background-modifier-border)", borderRadius: "8px", color: "var(--text-normal)", transition: "background-color 0.2s ease", },
    icon: { position: "absolute", top: "15px", right: "20px", fontFamily: "monospace", fontSize: "14px", color: "var(--text-faint)", userSelect: "none", cursor: "pointer", opacity: 0, transform: "scale(0.9)", transition: "opacity 0.2s ease-in-out, transform 0.2s ease-in-out", zIndex: 10, },
    // --- NEW --- Styles for the reload button
    reloadButton: {
        position: "absolute", top: "12px", right: "50px", zIndex: 10,
        width: "30px", height: "30px", borderRadius: "50%", border: "none",
        display: "flex", justifyContent: "center", alignItems: "center",
        cursor: "pointer", color: "var(--text-faint)", outline: "none",
        padding: 0, opacity: 1, transform: "scale(1)",
        backgroundColor: 'transparent', transition: 'background-color 0.2s ease, transform 0.2s ease',
    },
    title: { fontSize: "2em", fontWeight: "600", color: "var(--text-normal)" },
    subtitle: { fontSize: "1em", color: "var(--text-muted)", maxWidth: "400px", textAlign: "center" },
    promoBanner: { display: 'flex', alignItems: 'center', justifyContent: 'center', gap: '20px', padding: "20px 30px", borderRadius: "16px", width: 'min(100%, 500px)', background: 'rgba(24, 15, 28, 0.5)', border: '1px solid rgba(255, 255, 255, 0.1)', boxShadow: '0 0 80px -20px rgba(200, 160, 255, 0.3)', cursor: 'pointer', transition: 'transform 0.3s ease, box-shadow 0.3s ease', },
    promoIconContainer: { width: '48px', height: '48px', flexShrink: 0, },
    bannerTextContainer: { textAlign: 'left', },
    bannerTitle: { margin: '0 0 5px 0', fontSize: '1.2em', fontWeight: 600, color: 'var(--text-normal)' },
    bannerText: { margin: 0, color: 'var(--text-muted)', fontSize: '0.9em' },
    modalOverlay: { position: 'fixed', inset: 0, display: 'flex', alignItems: 'center', justifyContent: 'center', backdropFilter: 'blur(12px) saturate(1.2)', zIndex: 9999, },
    modalContent: { display: 'flex', flexDirection: 'column', gap: '15px', alignItems: 'center', width: 'min(100%, 95vw)', maxWidth: '450px', minHeight: '220px', justifyContent: 'center', padding: '30px', boxSizing: 'border-box', background: 'rgba(24, 15, 28, 0.7)', border: '1px solid rgba(255, 255, 255, 0.1)', borderRadius: '16px', boxShadow: '0 0 80px -20px rgba(200, 160, 255, 0.3)', overflow: 'hidden', textAlign: 'left' },
    modalTitle: { margin: 0, fontSize: '1.5em', fontWeight: 600, color: 'var(--text-normal)', textAlign: 'center' },
    modalText: { margin: '0 0 10px 0', color: 'var(--text-muted)', textAlign: 'center' },
    hoverRevealText: { color: 'var(--text-faint)', fontSize: '12px', fontStyle: 'italic', height: '15px', textAlign: 'center', transition: 'opacity 0.3s ease, transform 0.3s ease', opacity: 0, transform: 'translateY(5px)', },
    betText: { fontSize: '3.5em', fontWeight: 'bold', color: 'var(--text-normal)', userSelect: 'none', },
    resultList: { listStyle: 'none', padding: 0, margin: '5px 0', fontSize: '14px' },
    resultItem: { color: 'var(--text-muted)' },
    resultItemNew: { color: 'var(--text-success)' },
    resultItemUpdate: { color: 'var(--text-warning)' },
    compactWrapper: { padding: "16px", boxSizing: "border-box", display: "flex", flexDirection: "column", alignItems: "center", justifyContent: "center", gap: "12px", border: "1px dashed var(--background-modifier-border)", borderRadius: "8px", backgroundColor: "var(--background-primary-alt)", },
    compactText: { margin: 0, color: "var(--text-muted)", fontSize: "14px" },
    buttonGroup: { display: "flex", gap: "10px", flexWrap: "wrap", justifyContent: "center" },
    button: { padding: "8px 16px", fontSize: "12px", fontWeight: "500", color: "var(--text-on-accent)", backgroundColor: "var(--interactive-accent)", border: "none", borderRadius: "6px", cursor: "pointer", },
    secondaryButton: { backgroundColor: "var(--background-modifier-hover)", color: "var(--text-muted)", },
  };

  const CONTENT = { title: "VAULT UPDATER", subtitle: "This component keeps your vault structure up-to-date.", modalTitle: "Confirm Update", };
  const [isFullTab, setIsFullTab] = useState(true);
  const [isModalOpen, setIsModalOpen] = useState(false);
  const [modalStage, setModalStage] = useState('confirmation');
  const [isConfirmHovered, setIsConfirmHovered] = useState(false);
  const [updateStatus, setUpdateStatus] = useState('idle');
  const [updateResult, setUpdateResult] = useState(null);
  const [updateCheck, setUpdateCheck] = useState({ status: 'checking', info: null });
  const containerRef = useRef(null);
  const stateRefs = useRef({}).current;
  useEffect(() => { const container = containerRef.current; if (!container) return; if (isFullTab) { if (!container.parentNode) { setTimeout(() => setIsFullTab(true), 50); return; } const targetPaneContent = findNearestAncestorWithClass(container, 'workspace-leaf-content'); if (!targetPaneContent) { console.error("[BasicView] Full tab mode failed."); setIsFullTab(false); return; } const contentWrapper = findDirectChildByClass(targetPaneContent, 'view-content') || targetPaneContent; stateRefs.originalParent = container.parentNode; stateRefs.placeholder = document.createElement('div'); stateRefs.placeholder.style.display = 'none'; container.parentNode.insertBefore(stateRefs.placeholder, container); const computedParentPosition = window.getComputedStyle(contentWrapper).position; stateRefs.parentPositionInfo = { element: contentWrapper, originalInlinePosition: contentWrapper.style.position }; if (computedParentPosition === 'static') { contentWrapper.style.position = "relative"; } contentWrapper.appendChild(container); Object.assign(container.style, { position: "absolute", top: "0px", left: "0px", width: "100%", height: "100%", zIndex: "9998", overflow: "auto" }); } return () => { if (!stateRefs.originalParent) return; if (stateRefs.placeholder?.parentNode) { stateRefs.placeholder.parentNode.replaceChild(container, stateRefs.placeholder); } else { stateRefs.originalParent.appendChild(container); } if (stateRefs.parentPositionInfo?.element) { stateRefs.parentPositionInfo.element.style.position = stateRefs.parentPositionInfo.originalInlinePosition || ''; } container.removeAttribute("style"); Object.keys(stateRefs).forEach(key => stateRefs[key] = null); }; }, [isFullTab]);
  useEffect(() => { if (!isModalOpen) return; const handleKeyDown = (event) => { if (event.key === 'Escape') { handleCloseModal(); } }; document.addEventListener('keydown', handleKeyDown); return () => document.removeEventListener('keydown', handleKeyDown); }, [isModalOpen]);
  useEffect(() => { async function doUpdateCheck() { const result = await checkForUpdates(); setUpdateCheck({ status: 'checked', info: result }); } doUpdateCheck(); }, []);
  const handleOpenModal = () => { if (!updateCheck.info?.updateAvailable) return; setModalStage('confirmation'); setIsModalOpen(true); };
  const handleCloseModal = () => setIsModalOpen(false);
  const handleStartUpdate = async () => { setModalStage('bet'); await sleep(1000); try { setModalStage('processing'); setUpdateStatus('downloading'); const latestFiles = await downloadLatestFromRepo(); setUpdateStatus('comparing'); const trackedFilePaths = latestFiles.map(f => f.path); const currentFiles = await getCurrentVaultState(trackedFilePaths); const results = compareVersions(latestFiles, currentFiles); setUpdateResult(results); setUpdateStatus('writing'); const allFilesToWrite = [...results.newFiles, ...results.updatedFiles]; for (const file of allFilesToWrite) { const parentDir = file.path.substring(0, file.path.lastIndexOf('/')); if (parentDir && !await dc.app.vault.adapter.exists(parentDir)) { await dc.app.vault.createFolder(parentDir); } await dc.app.vault.adapter.write(file.path, file.content); } new Notice(`Update to v${updateCheck.info.remoteVersion} complete!`, 5000); setUpdateStatus('success'); setModalStage('results'); } catch (error) { console.error("Update process failed:", error); setUpdateStatus('error'); new Notice("Update process failed. Check console for details.", 5000); handleCloseModal(); } };
  const handleExitFullTab = (e) => { e.stopPropagation(); setIsFullTab(false); };
  const handleEnterFullTab = () => setIsFullTab(true);
  const handleCopyPath = () => { try { const activeFile = dc.app.workspace.getActiveFile(); if (activeFile) { navigator.clipboard.writeText(activeFile.path); new Notice(`Path copied`, 3000); } else { new Notice("Could not determine path.", 3000); } } catch (error) { new Notice("Error copying path.", 3000); } };
  const renderBannerContent = () => { if (updateCheck.status === 'checking') { return { title: "Checking for Updates...", text: "Please wait a moment." }; } if (updateCheck.info?.updateAvailable) { return { title: `Update Available: v${updateCheck.info.remoteVersion}`, text: `You are on v${updateCheck.info.localVersion}. Click to upgrade.` }; } return { title: "You are Up-to-Date!", text: `You have the latest version: v${updateCheck.info.localVersion}` }; };
  const renderModalContent = () => { switch (modalStage) { case 'confirmation': return (<> <h3 style={STYLES.modalTitle}>{CONTENT.modalTitle}</h3> <p style={STYLES.modalText}>You are about to update from <b>v{updateCheck.info.localVersion}</b> to <b>v{updateCheck.info.remoteVersion}</b>. This will overwrite existing files.</p> <div style={STYLES.buttonGroup}> <button style={{...STYLES.button, ...STYLES.secondaryButton}} onClick={handleCloseModal}>Cancel</button> <button style={STYLES.button} onClick={handleStartUpdate} onMouseEnter={() => setIsConfirmHovered(true)} onMouseLeave={() => setIsConfirmHovered(false)}>Confirm Update</button> </div> <p style={{ ...STYLES.hoverRevealText, opacity: isConfirmHovered ? 1 : 0, transform: isConfirmHovered ? 'translateY(0)' : 'translateY(5px)' }}>bro you trust me ?</p> </>); case 'bet': return <div className="bet-reveal"><h1 style={STYLES.betText}>BET 🫡</h1></div>; case 'processing': return (<> <h3 style={STYLES.modalTitle}>Upgrading...</h3> <p style={STYLES.modalText}>{updateStatus === 'downloading' ? 'Downloading latest version...' : (updateStatus === 'comparing' ? 'Comparing files...' : 'Applying changes...')}</p> </>); case 'results': return (<> <h3 style={STYLES.modalTitle}>Update Complete!</h3> <p style={STYLES.modalText}>Applied version <b>v{updateCheck.info.remoteVersion}</b>.</p> <div> {updateResult.newFiles.length > 0 && <> <p style={{...STYLES.bannerTitle, fontSize: '1em', margin: '0 0 5px 0' }}>New Files Added:</p> <ul style={STYLES.resultList}>{updateResult.newFiles.map(f => <li key={f.path} style={{...STYLES.resultItem, ...STYLES.resultItemNew}}>+ {f.path}</li>)}</ul> </>} {updateResult.updatedFiles.length > 0 && <> <p style={{...STYLES.bannerTitle, fontSize: '1em', margin: '10px 0 5px 0' }}>Files Updated:</p> <ul style={STYLES.resultList}>{updateResult.updatedFiles.map(f => <li key={f.path} style={{...STYLES.resultItem, ...STYLES.resultItemUpdate}}>~ {f.path}</li>)}</ul> </>} </div> <div style={{...STYLES.buttonGroup, marginTop: '15px'}}><button style={STYLES.button} onClick={handleCloseModal}>Finish</button></div> </>); default: return null; } }
  const bannerContent = renderBannerContent();
  return ( <div ref={containerRef}> <style>{STYLES.injectedStyles}</style> {isModalOpen && <div style={STYLES.modalOverlay} className="modal-fade-in"><div style={STYLES.modalContent} className="modal-scale-in">{renderModalContent()}</div></div>} {isFullTab ? ( <div style={STYLES.fullTabWrapper} className={uniqueWrapperClass}>
    <span style={STYLES.icon} className="subtle-icon" title="Exit Full Tab" onClick={handleExitFullTab}>&lt;/&gt;</span>
    {/* --- NEW --- Reload button is here */}
    <button onClick={onReloadRequest} className="reload-button" style={STYLES.reloadButton} aria-label="Reload Component" title="Reload Component">
        <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="currentColor">
            <path d="M12 4V1L8 5l4 4V6c3.31 0 6 2.69 6 6s-2.69 6-6 6-6-2.69-6-6H4c0 4.42 3.58 8 8 8s8-3.58 8-8-3.58-8-8-8z"/>
        </svg>
    </button>
    <h2 style={STYLES.title}>{CONTENT.title}</h2><p style={STYLES.subtitle}>{CONTENT.subtitle}</p> <div style={{...STYLES.promoBanner, cursor: updateCheck.info?.updateAvailable ? 'pointer' : 'default', opacity: updateCheck.status === 'checking' ? 0.7 : 1}} className="promo-banner" onClick={handleOpenModal}> <div style={STYLES.promoIconContainer}><AnimatedIcon svgString={ICONS[0].svg} isActive={updateCheck.info?.updateAvailable} isInView={true} /></div> <div style={STYLES.bannerTextContainer}> <h3 style={STYLES.bannerTitle}>{bannerContent.title}</h3><p style={STYLES.bannerText}>{bannerContent.text}</p> </div> </div> </div> ) : ( <div style={STYLES.compactWrapper}> <p style={STYLES.compactText}>Component is in compact mode.</p> <div style={STYLES.buttonGroup}> <button style={STYLES.button} onClick={handleEnterFullTab}>Enter Full Tab</button> <button style={{...STYLES.button, ...STYLES.secondaryButton}} onClick={handleCopyPath}>Find Codeblock</button> </div> </div> )} </div> );
};

// =================================================================================
// --- NEW --- Container Component (Manages the hard reset logic)
// =================================================================================
function VaultUpdaterContainer() {
    const [refreshKey, setRefreshKey] = useState(0);

    const handleHardReset = () => {
        console.log(`${LOG_PREFIX} Hard reset triggered. Forcing component to remount.`);
        setRefreshKey(prevKey => prevKey + 1);
    };

    // Every time handleHardReset is called, the `key` prop changes,
    // forcing React to destroy the old BasicView and create a new one.
    return <BasicView key={refreshKey} onReloadRequest={handleHardReset} />;
}

// The final export is the container, which seamlessly handles the reset logic.
return { BasicView: VaultUpdaterContainer };
```


