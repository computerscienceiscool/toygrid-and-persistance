# ToyGrid: Distributed Collaborative Editor (Concise Slide Deck)

---

## Slide 1: Title

**ToyGrid: Distributed Collaborative Editor**  
*JJ Salley*

---

## Slide 2: What Is ToyGrid?

- A React-based, multi-user collaborative editor demo  
- Uses Yjs (CRDT) for real-time merging  
- Persists state locally with IndexedDB  
- Syncs live edits via a WebSocket server  
- Testing in-browser IPFS storage via Helia  

---

## Slide 3: Front-End Stack

- **React**  
  - Manages UI components (in `editor/src/`)  
- **Tiptap** + **ProseMirror**  
  - Rich-text editor core  
  - Extensions used:  
    - `StarterKit` (basic formatting)  
    - `Highlight`, `TaskList`, `TaskItem`, `CharacterCount`  
    - `Collaboration` & `CollaborationCursor` (tie into Yjs)  

---

## Slide 4: Yjs (CRDT) Integration

- **Yjs Document** (`ydoc = new Y.Doc()`)  
  - In-memory shared state for the editor  
  - Merges simultaneous edits without conflicts  
- **Tiptap Collaboration Extension**  
  - Hooks ProseMirror → Yjs  
  - Emits `ydoc.on('update', …)` events on every local change  

---

## Slide 5: Local Persistence (IndexedDB)

- **IndexedDBPersistence** (`y-indexeddb`)  
  - Instantiated as `new IndexeddbPersistence('cswg-demo', ydoc)`  
  - Automatically stores each Yjs update in browser’s IndexedDB  
  - On page load, `once('synced', …)` restores prior state  
- **Benefit**: Offline resilience and instant reload of last edits  

---

## Slide 6: Real-Time Sync (Y-WebSocket)

- **WebsocketProvider** (`y-websocket`)  
  - Connects `ydoc` to a central Y-WebSocket server (e.g. `ws://localhost:3000/cswg-demo`)  
  - On each update, broadcasts to all connected clients in the same “room”  
  - Tracks “awareness” (cursor positions, user names/colors) via `CollaborationCursor`  
- **Server**: run `npx y-websocket` on port 3000 (or use `x/tools/start-dev-toygrid-yjs-server.sh`)  

---

## Slide 7: Helia (IPFS) Integration (Testing)

- **Motivation**: Long-term, decentralized persistence of Yjs updates  
- **`helia.js` Helper**  
  ```js
  import { createHelia } from 'helia';
  let heliaNode = null;
  export async function initHelia(groupHash) {
    if (heliaNode) return heliaNode;
    heliaNode = await createHelia();
    return heliaNode;
  }
  ```
- **Usage** (in `App.jsx`):  
  1. `initHelia('TEST_GROUP_HASH')` on mount  
  2. `ydoc.on('update', async update => { … await heliaNode.blockstore.put(update) … })`  
  3. Stores latest update CID under `TEST_GROUP_HASH` in LocalStorage  
- **Current Status**:  
  - Writes each update to Helia’s in-memory blockstore  
  - Reading from Helia on startup (fetch + `Y.applyUpdate`) is not yet implemented  

---

## Slide 8: Next Steps & Roadmap

1. **Replace Placeholder Hash**  
   - Use the real “group hash” instead of `'TEST_GROUP_HASH'`  
2. **Fetch from Helia on Load**  
   - If LocalStorage has a CID, `await heliaNode.blockstore.get(cid)` → `Y.applyUpdate(ydoc, data)`  
3. **Enable P2P Discovery**  
   - Share CIDs between browser peers without a central server  
4. **Optional UI Toggle**  
   - Let users choose “WebSocket Only” vs. “WebSocket + Helia” modes  
5. **Finalize Code Review & Deployment**  

---

# ToyGrid Technology Cheatsheet

| Technology                      | Location / File                          | Purpose                                                                |
|---------------------------------|------------------------------------------|------------------------------------------------------------------------|
| **React**                       | `editor/src/App.jsx`, `editor/src/…`     | Core UI framework                                                      |
| **Tiptap + ProseMirror**        | `editor/src/App.jsx`                     | Rich-text editing; ProseMirror under the hood                           |
| **StarterKit**                  | `@tiptap/starter-kit` (import in `App.jsx`) | Basic formatting (bold, italic, headings, lists)                         |
| **Highlight**                   | `@tiptap/extension-highlight`            | Text highlighting extension                                             |
| **TaskList** / **TaskItem**     | `@tiptap/extension-task-list` / `task-item` | Checkbox-list / to-do items                                              |
| **CharacterCount**              | `@tiptap/extension-character-count`      | Shows character count limit                                             |
| **Collaboration**               | `@tiptap/extension-collaboration`        | Connects Tiptap to Yjs CRDT document                                     |
| **CollaborationCursor**         | `@tiptap/extension-collaboration-cursor` | Shows collaborators’ cursors (with names/colors)                         |
| **Yjs (CRDT)**                  | `import * as Y from 'yjs'` in `App.jsx`  | In-memory conflict-free replicated data type (shared document state)     |
| **IndexeddbPersistence**        | `import { IndexeddbPersistence } from 'y-indexeddb'` | Persists Yjs updates to browser’s IndexedDB for offline recovery          |
| **WebsocketProvider**           | `import { WebsocketProvider } from 'y-websocket'` | Real-time Yjs update sync via central WebSocket server                    |
| **Y-WebSocket Server**          | `x/tools/start-dev-toygrid-yjs-server.sh` + `y-websocket` | Node-based server that relays Yjs updates between clients                 |
| **Helia (IPFS in Browser)**     | `editor/src/helia.js`                    | In-browser IPFS blockstore; stores Yjs updates as content-addressed blocks |
| **LocalStorage**                | Used in `App.jsx` for `placeholderGroupHash` →  latest CID | Stores the latest Helia CID under a group hash                            |
| **Browser’s IndexedDB**         | Automatically managed by `IndexeddbPersistence` | Stores and retrieves entire Yjs document state across page reloads         |
| **Environment Variables**       | `REACT_APP_YJS_WEBSOCKET_SERVER_URL`     | Overrides default WebSocket server URL (e.g., `ws://localhost:3000`)      |

---

**Cheatsheet Notes**  
- All “import” lines refer to code in `editor/src/App.jsx` unless otherwise specified.  
- Helia integration is currently in testing; once a real group hash is provided, the client will fetch updates from IPFS on load.  
- The Y-WebSocket server may run locally (e.g. `npx y-websocket`) or remotely (e.g. `ws://europa.d4.t7a.org:3000`).  
- **IndexedDB** and **LocalStorage** are the only direct client-side stores used today; no cookies or server-side sessions are part of ToyGrid.  
- Helia runs purely in the browser (no separate server component) and uses a placeholder hash for now.
