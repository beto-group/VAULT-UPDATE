---
tags:
  - test/bob
file.tags:
---



# ViewComponent

```jsx
// Using the hooks as you specified
const { useState, useEffect } = dc;

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


/**
 * A component that provides simple, git-like version control for a single file
 * and displays all of its code blocks in a navigable tabbed interface.
 * The function is named GitControl to match the required export.
 */
function GitControl() {
  // ====================================================================
  //                        -- YOUR SETTINGS --
  // The file to track and display code blocks from.
  const fileName = "GitControl.component.v1.md";
  // ====================================================================

  // --- INTERNAL SETUP (Unchanged) ---
  const safeDirName = sanitizePathForDirName(fileName);
  const fileGitBaseDir = `.datacore/.git/${safeDirName}`;
  const objectsDir = `${fileGitBaseDir}/objects`;
  const headFile = `${fileGitBaseDir}/HEAD`;

  // --- STATE MANAGEMENT ---
  // State for multiple blocks and tabs.
  const [codeBlocks, setCodeBlocks] = useState([]);
  const [activeTabIndex, setActiveTabIndex] = useState(0);
  const [statusMessage, setStatusMessage] = useState("Initializing version control...");

  useEffect(() => {
    const processFileAndCommitChanges = async () => {
      try {
        const adapter = app.vault.adapter;
        let fullFileContent; 

        const file = app.vault.getAbstractFileByPath(fileName);
        if (!file) {
          setStatusMessage(`Error: Source file not found.`);
          setCodeBlocks([]);
          return;
        }
        fullFileContent = await app.vault.read(file);

        // --- VERSIONING LOGIC (Largely Unchanged) ---
        const currentHash = await calculateHash(fullFileContent);
        const objectPath = `${objectsDir}/${currentHash}.json`;

        if (await adapter.exists(objectPath)) {
          setStatusMessage(`No changes detected. (v: ${currentHash.slice(0, 7)})`);
        } else {
          setStatusMessage(`New version found. Committing...`);
          
          // Ensure base directories exist
          const stat = await adapter.stat(fileGitBaseDir);
          if (stat && stat.type === 'file') {
            await adapter.remove(fileGitBaseDir);
            await adapter.mkdir(fileGitBaseDir);
          } else if (!stat) {
            await adapter.mkdir(fileGitBaseDir);
          }
          if (!(await adapter.exists(objectsDir))) {
            await adapter.mkdir(objectsDir);
          }
          
          const commitObject = {
            source: fileName,
            timestamp: new Date().toISOString(),
            content: fullFileContent,
          };

          await adapter.write(objectPath, JSON.stringify(commitObject, null, 2));
          await adapter.write(headFile, currentHash);
          
          setStatusMessage(`Successfully committed v: ${currentHash.slice(0, 7)}`);
        }

        // --- PARSING LOGIC (from AceFileExplorer) ---
        // This regex captures both markdown headers (#) and fenced code blocks (```).
        const regex = /(?:^# (.*))|(?:^```([^\r\n]*)\r?\n([\s\S]*?)\r?\n```)/gm;
        const allMatches = [...fullFileContent.matchAll(regex)];

        const parsedBlocks = [];
        let lastHeader = null;
        let untitledBlockCounter = 1;

        for (const match of allMatches) {
            const headerMatch = match[1];
            // langMatch (match[2]) is available but not used in this simple viewer
            const codeMatch = match[3];

            if (headerMatch !== undefined) {
                // We found a header, store it for the next code block.
                lastHeader = headerMatch.trim();
            } else if (codeMatch !== undefined) {
                // We found a code block.
                const title = lastHeader ? lastHeader : `Block ${untitledBlockCounter++}`;
                parsedBlocks.push({
                    title: title,
                    content: codeMatch.trim()
                });
                lastHeader = null; // Reset header after it's been used.
            }
        }
        
        if (parsedBlocks.length === 0) {
          setStatusMessage(prev => `${prev} | Note: No code blocks were found in "${fileName}".`);
          setCodeBlocks([]);
        } else {
          setStatusMessage(prev => `${prev} | Loaded ${parsedBlocks.length} code block(s).`);
          setCodeBlocks(parsedBlocks);
          setActiveTabIndex(0); // Reset to the first tab on load
        }

      } catch (error) {
        console.error("GitControl Critical Error:", error);
        setStatusMessage(`A critical error occurred.`);
        setCodeBlocks([{ title: "Error", content: `// ${error.message}` }]);
      }
    };

    processFileAndCommitChanges();
    
  }, [fileName]);

  // --- RENDER LOGIC with TABS ---
  return (
    <dc.Stack style={{ padding: "10px 15px", fontFamily: "monospace" }}>
      {/* Status Bar */}
      <div style={{
          padding: "5px 10px",
          marginBottom: "10px",
          background: "#eef",
          border: "1px solid #cce",
          borderRadius: "4px",
          fontSize: "0.9em",
          color: "#335"
      }}>
          Status: {statusMessage}
      </div>

      {/* Tab Navigation */}
      {codeBlocks.length > 0 && (
          <div style={{ display: 'flex', borderBottom: '1px solid #ddd', flexWrap: 'wrap', marginBottom: '10px' }}>
              {codeBlocks.map((block, index) => (
                  <button
                      key={`${block.title}-${index}`}
                      onClick={() => setActiveTabIndex(index)}
                      style={{
                          padding: '8px 12px',
                          cursor: 'pointer',
                          border: 'none',
                          borderBottom: index === activeTabIndex ? '3px solid #007acc' : '3px solid transparent',
                          background: index === activeTabIndex ? '#f0f8ff' : 'none',
                          fontWeight: index === activeTabIndex ? 'bold' : 'normal',
                          color: 'inherit',
                          fontSize: '0.9em'
                      }}
                  >
                      {block.title}
                  </button>
              ))}
          </div>
      )}

      {/* Code Block Display */}
      <pre style={{
        background: '#f4f4f4',
        border: '1px solid #ddd',
        padding: '1em',
        borderRadius: '5px',
        whiteSpace: 'pre-wrap',
        wordBreak: 'break-word',
        color: '#333',
        margin: "0"
      }}>
        <code>
          {codeBlocks.length > 0 && codeBlocks[activeTabIndex]
            ? codeBlocks[activeTabIndex].content
            : '// Select a file to load, or no code blocks were found.'}
        </code>
      </pre>
    </dc.Stack>
  );
}

// This now correctly matches the function name.
return { GitControl };
```


# Test

```jsx
const { WorldView } = await dc.require(dc.headerLink("B25MovingCat.component.v3.md", "ViewComponent"));
return <WorldView />;


```