# Concise ToyGrid Slides: Speaker Script

---

## Slide 1: Title

```
ToyGrid: Distributed Collaborative Editor  
JJ Salley
```

*Speaker:*

*“Hello everyone, and welcome to my presentation on ToyGrid: Building a Distributed Collaborative Editor. I’m JJ Salley. Today I will walk you through ToyGrid’s architecture, its core technologies, and how we handle persistence and real-time sync.”*

*Demo Files:*  
*No files demoed on this slide — this is an introductory title page.*

---

## Slide 2: What Is ToyGrid?

```
- A React-based, multi-user collaborative editor demo  
- Uses Yjs (CRDT) for real-time merging  
- Persists state locally with IndexedDB  
- Syncs live edits via a WebSocket server  
- Testing in-browser IPFS storage via Helia  
```

*Speaker:*

*“On this slide, I’ll give a high-level overview of what ToyGrid is and what it does.*  
- *First, ToyGrid is a web-based editor built with React. It allows multiple users to edit the same document at once.*  
- *Under the hood, we use Yjs, a Conflict-free Replicated Data Type library (CRDT), to merge edits in real time without conflicts.*  
- *We persist the editor’s state locally in the browser using IndexedDB, so if you close your browser and come back later, you’ll still see your last edits.*  
- *For real-time collaboration across multiple devices, we sync edits via a WebSocket server using Yjs’s WebSocket provider.*  
- *Finally, we are experimenting with Helia, an in-browser IPFS implementation, to store every Yjs update in a decentralized, content-addressed blockstore. This slide lists that as a ‘testing’ item because we have the write path implemented, but not yet the read/retrieve path.”*

*Demo Files:*  
*- To illustrate “React-based,” you could open `editor/src/App.jsx`.*  
*- To show IndexedDB persistence, demo the browser’s DevTools → Application → IndexedDB and point to keys under “toygrid”.*

---

## Slide 3: Front-End Stack

```
- React  
  - Manages UI components (in `editor/src/`)  
- Tiptap + ProseMirror  
  - Rich-text editor core  
  - Extensions used:  
    - `StarterKit` (basic formatting)  
    - `Highlight`, `TaskList`, `TaskItem`, `CharacterCount`  
    - `Collaboration` & `CollaborationCursor` (tie into Yjs)  
```

*Speaker:*

*“Now let’s dive into the front-end stack. ToyGrid is built using React, which manages all of our UI components. You can see the React code in `editor/src/` — `App.jsx` is the main entry point.*  
*Within React, we use Tiptap, which is a headless editor built on ProseMirror. ProseMirror powers the rich-text editing experience.*  
*We included several Tiptap extensions:*  
- *`StarterKit` provides basic formatting commands like bold, italics, and headings.*  
- *`Highlight` lets you highlight text.*  
- *`TaskList` and `TaskItem` let you create checkboxes and to-do lists.*  
- *`CharacterCount` shows the current character count (we cap it at 10,000).*  
- *Most importantly for collaboration, we include `@tiptap/extension-collaboration` and `@tiptap/extension-collaboration-cursor`. These two tie the editor to Yjs and let us show collaborator cursors with names and colors.*  
*Everything you see in the browser is rendered by React and Tiptap together.”*

*Demo Files:*  
*- Open `editor/src/App.jsx` to show the React component structure.*  
*- Open `package.json` to show the dependencies for Tiptap and its extensions.*  

---

## Slide 4: Yjs (CRDT) Integration

```
- Yjs Document (`ydoc = new Y.Doc()`)  
  - In-memory shared state for the editor  
  - Merges simultaneous edits without conflicts  
- Tiptap Collaboration Extension  
  - Hooks ProseMirror → Yjs  
  - Emits `ydoc.on('update', …)` events on every local change  
```

*Speaker:*

*“This slide explains how we integrate Yjs, the CRDT library.*  
*First, we instantiate a Yjs document in code by calling `new Y.Doc()`. This `ydoc` represents the entire shared document state, in memory.*  
*Yjs automatically manages merging concurrent edits. If two users type at the same time, Yjs resolves any conflicts behind the scenes.*  
*Next, we connect Tiptap to Yjs by using the `@tiptap/extension-collaboration` extension. Under the hood, that extension listens for ProseMirror transactions and applies them to `ydoc`, and also applies Yjs updates back into ProseMirror’s document.*  
*We also attach a listener: `ydoc.on('update', callback)` that fires whenever the local `ydoc` receives an update (either from local typing or from remote peers). We use that to write changes to IndexedDB and to Helia.”*

*Demo Files:*  
*- Show the line in `editor/src/App.jsx`:*  
  ```js
  const ydoc = new Y.Doc()
  ```  
*- Show where `Collaboration.configure({ document: ydoc })` is passed into Tiptap.*  

---

## Slide 5: Local Persistence (IndexedDB)

```
- IndexedDBPersistence (`y-indexeddb`)  
  - Instantiated as `new IndexeddbPersistence('cswg-demo', ydoc)`  
  - Automatically stores each Yjs update in browser’s IndexedDB  
  - On page load, `once('synced', …)` restores prior state  
- Benefit: Offline resilience and instant reload of last edits  
```

*Speaker:*

*“Next, we persist the Yjs document locally using IndexedDB. We use the `y-indexeddb` package, which exports `IndexeddbPersistence`. In code, we simply write:*  
```js
const indexeddbProvider = new IndexeddbPersistence('cswg-demo', ydoc);
indexeddbProvider.once('synced', () => {
  console.log('[IndexedDB] Document loaded from IndexedDB');
});
```  
*Once that runs, every Yjs update is automatically written to a database named `cswg-demo` in the browser’s IndexedDB.*  
*When the page reloads, `IndexeddbPersistence` reads from IndexedDB, applies those updates to `ydoc`, and then fires the `synced` event, ensuring the editor starts with exactly the same content as before.*  
*This gives us offline resilience—if I lose my network, I can still keep editing and come back later.”*

*Demo Files:*  
*- Open the browser’s DevTools → Application → IndexedDB and show the `cswg-demo` database entries.*  
*- Point to the code in `App.jsx` where we create `new IndexeddbPersistence('cswg-demo', ydoc)`.  

---

## Slide 6: Real-Time Sync (Y-WebSocket)

```
- WebsocketProvider (`y-websocket`)  
  - Connects `ydoc` to a central Y-WebSocket server (e.g. `ws://localhost:3000/cswg-demo`)  
  - On each update, broadcasts to all connected clients in the same “room”  
  - Tracks “awareness” (cursor positions, user names/colors) via `CollaborationCursor`  
- Server: run `npx y-websocket` on port 3000 (or use `x/tools/start-dev-toygrid-yjs-server.sh`)  
```

*Speaker:*

*“To enable multiple users to see each other’s edits in real time, we use a Yjs WebSocket provider.*  
*In `App.jsx`, we write:*  
```js
const websocketProvider = new WebsocketProvider(
  websocketUrl,    // e.g., 'ws://localhost:3000'
  'cswg-demo',     // room name
  ydoc
);
websocketProvider.on('status', event => {
  setStatus(event.status); // updates our UI to show “connected” or “offline”
});
```  
*This provider connects to a standalone Y-WebSocket server running on port 3000. Whenever Yjs generates an update locally, `WebsocketProvider` sends it to the server. The server then broadcasts it to all other clients in the same room (`cswg-demo`).*  
*We also use the `CollaborationCursor` extension which hooks into `websocketProvider.awareness` to show user cursors and names in real time.*  
*To run the server locally, you can simply `cd yjs-websocket-server && npx y-websocket --port 3000`, or use the helper script at `x/tools/start-dev-toygrid-yjs-server.sh` that automates those steps.”*

*Demo Files:*  
*- Show `x/tools/start-dev-toygrid-yjs-server.sh` and demonstrate how to uncomment lines to start the server.*  
*- Point to `editor/src/App.jsx` where `new WebsocketProvider(...)` is instantiated.*  

---

## Slide 7: Helia (IPFS) Integration (Testing)

```
- Motivation: Long-term, decentralized persistence of Yjs updates  
- `helia.js` Helper  
  ```js
  import { createHelia } from 'helia';
  let heliaNode = null;
  export async function initHelia(groupHash) {
    if (heliaNode) return heliaNode;
    heliaNode = await createHelia();
    return heliaNode;
  }
  ```  
- Usage (in `App.jsx`):  
  1. `initHelia('TEST_GROUP_HASH')` on mount  
  2. `ydoc.on('update', async update => { … await heliaNode.blockstore.put(update) … })`  
  3. Stores latest update CID under `TEST_GROUP_HASH` in LocalStorage  
- Current Status:  
  - Writes each update to Helia’s in-memory blockstore  
  - Reading from Helia on startup (fetch + `Y.applyUpdate`) is not yet implemented  
```

*Speaker:*

*“Now let’s talk about Helia integration—our in-browser IPFS solution.*  
*First, we created a file `editor/src/helia.js` with this helper function:*  
```js
import { createHelia } from 'helia';
let heliaNode = null;

export async function initHelia(groupHash) {
  if (heliaNode) return heliaNode;
  heliaNode = await createHelia();
  return heliaNode;
}
```  
*This function, `initHelia`, initializes Helia in the browser and returns the Helia node instance. We pass in a `groupHash`—for now it’s a placeholder string `'TEST_GROUP_HASH'`.*  

*Then in `App.jsx` inside our `useEffect`, we call `initHelia(placeholderGroupHash)` on mount. Once Helia is ready, we attach a listener to Yjs updates:*  
```js
ydoc.on('update', async update => {
  const cid = await heliaNode.blockstore.put(update);
  localStorage.setItem(placeholderGroupHash, cid.toString());
  console.log('Stored update in Helia with CID:', cid.toString());
});
```  
*That code writes each Yjs update into Helia’s content-addressed blockstore and records the returned CID in LocalStorage under our placeholder key.*  

*Right now, we can see in the browser console that Helia initializes and stores each update under `'TEST_GROUP_HASH'`. However, we haven’t yet implemented retrieving that data on startup. We’ll do that in the next phase.”*

*Demo Files:*  
*- Open `editor/src/helia.js` to show how `createHelia()` is called.*  
*- Show the code snippet in `App.jsx` where `ydoc.on('update', …)` is inserted.*  

---

## Slide 8: Next Steps & Roadmap

```
1. Replace Placeholder Hash  
   - Use the real “group hash” instead of 'TEST_GROUP_HASH'  
2. Fetch from Helia on Load  
   - If LocalStorage has a CID, `await heliaNode.blockstore.get(cid)` → `Y.applyUpdate(ydoc, data)`  
3. Enable P2P Discovery  
   - Share CIDs between browser peers without a central server  
4. Optional UI Toggle  
   - Let users choose “WebSocket Only” vs. “WebSocket + Helia” modes  
5. Finalize Code Review & Deployment  
```

*Speaker:*

*“Finally, here’s our roadmap for moving Helia from testing into production:*  
1. *Replace the placeholder hash (`'TEST_GROUP_HASH'`) with a real group identifier agreed upon by our team.*  
2. *When the editor loads, check LocalStorage for a stored CID. If found, call `heliaNode.blockstore.get(cid)` to fetch the latest update from IPFS, then apply it to `ydoc` with `Y.applyUpdate(ydoc, data)`. That way the document always starts from the most recent Helia snapshot.*  
3. *Implement peer discovery in IPFS or a simple gossip protocol so that if a new user opens the editor, they can fetch the latest CIDs from someone else’s Helia node.*  
4. *Consider adding a UI toggle so users can choose to collaborate over WebSocket only (fast, low-latency) or use WebSocket + Helia (persist to IPFS for archival and P2P sharing).*  
5. *Once these features are added, we’ll finalize code reviews and deploy ToyGrid to production.”*

*Demo Files:*  
*- Highlight the placeholder line in `App.jsx`: `const placeholderGroupHash = 'TEST_GROUP_HASH'`.*  
*- Point out where we would insert the `heliaNode.blockstore.get(...) + Y.applyUpdate(...)` logic in the same `useEffect`.*  

---

# End of Speaker Script
