# ToyGrid: Distributed Collaborative Editor

---

## Slide 1: Title

**ToyGrid: Building a Distributed Collaborative Editor**  
*JJ Salley*

---

## Slide 2: Overview

**What is ToyGrid?**  
- A web-based, multi-user collaborative editor demo.  
- Uses CRDTs (Yjs) for real-time editing and merging.  
- Persists state locally (IndexedDB) and plans to add P2P storage (Helia/IPFS).  
- Syncs across clients via a Y-WebSocket server.  

**Goals of this Presentation**  
1. Explain ToyGrid’s architecture and key components.  
2. Show how client-side persistence is handled.  
3. Introduce Helia integration for broader replication.  
4. Provide background on common web state persistence techniques.  

---

## Slide 3: Big Picture Architecture

1. **Client (Browser) Components**  
   - **Tiptap Editor** (ProseMirror under the hood)  
   - **Yjs Document (`ydoc`)**: in-memory CRDT representing the shared document.  
   - **IndexedDBPersistence**: saves Yjs updates locally for offline recovery.  
   - **WebsocketProvider**: syncs Yjs updates in real time via WebSocket server.  
   - **Helia Integration (planned)**: stores Yjs updates in a local IPFS/Helia blockstore for P2P sharing.  

2. **Server Components**  
   - **Y-WebSocket Server**: relays Yjs updates between connected clients.  
   - (No Helia server needed—Helia runs entirely in the browser.)  

3. **Storage Layers**  
   - **IndexedDB**: fast local storage for immediate reload.  
   - **Helia (IPFS)**: content-addressed, browser-based blockstore for long-term, decentralized persistence.  

---

## Slide 4: ToyGrid Front-End Stack

- **React**: UI framework  
- **Tiptap** (`@tiptap/react` + `StarterKit` + extensions)  
  - Rich text editing powered by ProseMirror.  
  - Extensions used: Highlight, TaskList, TaskItem, CharacterCount.  
- **Yjs**: CRDT library  
  - Instantiated as `const ydoc = new Y.Doc()`.  
  - Provides `ydoc.on('update', ...)` event to capture and apply changes.  
- **@tiptap/extension-collaboration** & **@tiptap/extension-collaboration-cursor**  
  - Collaboration extension ties Tiptap to Yjs.  
  - CollaborationCursor manages awareness (user cursors, names, colors).  
- **IndexeddbPersistence** (`y-indexeddb`)  
  - Automatically stores Yjs updates to IndexedDB under a key (e.g. `"cswg-demo"`).  
  - On load, fires `once('synced', ...)` when local state is restored.  

---

## Slide 5: Real-Time Sync with Y-WebSocket

- **WebsocketProvider** (`y-websocket`)  
  - Connects to `ws://<server>:<port>/<room>` (e.g. `ws://localhost:3000/cswg-demo`).  
  - On each local Yjs update, sends the update to the server.  
  - Server broadcasts to all other clients in the same “room.”  
  - Handles **awareness** (who’s online, cursor positions).  

- **In ToyGrid’s Code**:  
  ```js
  const websocketProvider = new WebsocketProvider(
    websocketUrl,          // e.g. 'ws://localhost:3000'
    'cswg-demo',           // room name
    ydoc                   // the shared Yjs document
  );
  websocketProvider.on('status', event => {
    setStatus(event.status); // updates “connected” / “offline”
  });
  // CollaborationCursor extension uses websocketProvider.awareness
  ```

- **Why Y-WebSocket?**  
  - Ensures all clients see edits in real time.  
  - Provides awareness updates (cursor, name, color).  

---

## Slide 6: Local Persistence with IndexedDB

- **IndexedDBPersistence** (`y-indexeddb`)  
  - Stores each Yjs update in the browser’s IndexedDB.  
  - On page reload or offline, rebuilds `ydoc` from stored updates.  

- **In ToyGrid’s Code**:  
  ```js
  const indexeddbProvider = new IndexeddbPersistence('cswg-demo', ydoc);
  indexeddbProvider.once('synced', () => {
    console.log('[IndexedDB] Document loaded from IndexedDB');
  });
  ```
- **Advantages**  
  - **Offline Resilience**: If network is unavailable, edits still persist.  
  - **Fast Startup**: Recover state instantly from local IndexedDB.  
  - **No Server Round-Trip**: Immediate access to last known state without waiting on network.  

---

## Slide 7: Planning for Helia / IPFS Integration

- **Why Helia (IPFS) in the Browser?**  
  - Browser‐based blockstore → content-addressed storage.  
  - Peer-to-peer distribution of Yjs updates (beyond central WebSocket).  
  - Long-term archival of CRDT snapshots: each update yields a CID.  
  - Allows offline devices to fetch the latest state from peers, not just local cache.  

- **ToyGrid’s Helia Integration (currently testing)**  
  1. **`helia.js`** helper:  
     ```js
     import { createHelia } from 'helia';
     let heliaNode = null;
     export async function initHelia(groupHash) {
       console.log('initHelia called with groupHash:', groupHash);
       if (heliaNode) return heliaNode;
       heliaNode = await createHelia();
       console.log('Helia node initialized:', heliaNode);
       return heliaNode;
     }
     ```
  2. **Placeholder Group Hash** (for testing):  
     ```js
     const placeholderGroupHash = 'TEST_GROUP_HASH'; // FOR TESTING ONLY
     ```
  3. **Hooking into Yjs Updates**:  
     ```js
     ydoc.on('update', async update => {
       const cid = await node.blockstore.put(update);
       localStorage.setItem(placeholderGroupHash, cid.toString());
       console.log('Stored update in Helia with CID:', cid.toString());
     });
     ```
  4. **UI Status**:  
     ```jsx
     <div>
       {heliaNode
         ? 'Helia is initialized'
         : 'Initializing Helia…'}
     </div>
     ```
- **Next Steps**  
  - Replace `placeholderGroupHash` with real “group hash” once available.  
  - On startup, fetch latest state from Helia if present (`helia.blockstore.get(cid)` + `Y.applyUpdate`).  
  - Eventually, peer-to-peer discovery to fetch CIDs from other clients.  

---

## Slide 8: Client-Side Web State Persistence (Background)

### The Big Question – How does a website remember you?
- **Websites use HTTP, which is stateless.**  
- **Cookies, sessions, and tokens** help “remember” users.  
- These tools add memory to otherwise forgetful systems.  

---

## Slide 9: Cookie-Based Authentication – Cloakroom Ticket

- **Like a coat check ticket.**  
- Cookie = small identifier in your browser.  
- Sent with every request to identify you.  
- **Used for**: Staying logged in, saving cart contents, remembering preferences.  
- **Limitations**: Size constraints, security flags (HTTPOnly, Secure, SameSite).  

---

## Slide 10: Server-Side Sessions – Library Card

- **Like a library card pointing to your info.**  
- Browser holds a session ID; server stores session data.  
- **Use cases**: Banking, schools, e-commerce checkouts.  
- **Session management**: Server controls lifetime, logout, expiration, scalability.  

---

## Slide 11: Token-Based Authentication – Hotel Keycard

- **JWT (JSON Web Token) = digital keycard.**  
  - Header: type + algorithm  
  - Payload: user info + permissions  
  - Signature: prevents tampering  
- **Advantages**:  
  - Self-contained and portable  
  - Scales easily for mobile & API usage  
- **Challenges**:  
  - Token revocation (use short expiration, refresh tokens, track invalid tokens).  

---

## Slide 12: Client-Side Data Persistence Techniques

- **Cookies**: Small key–value, auto-sent with requests; good for authentication.  
- **LocalStorage**: Synchronous key–value store, persists across sessions; useful for preferences.  
- **SessionStorage**: Clears on tab close; short‐lived per‐tab persistence.  
- **IndexedDB**: Asynchronous, structured database; ideal for large, complex data (e.g. Yjs updates).  
- **Cache API**: Service worker–based response caching for offline use.  

---

## Slide 13: PromiseGrid Analogy

- **Local state = sticky notes on your device**  
- **Shared state = synced notebook in the cloud**  
- **PromiseGrid = flexible agents with secure server sync**  
- ToyGrid’s architecture parallels this:  
  - **Yjs + IndexedDB** = local “sticky notes” (instant, offline‐first edits).  
  - **WebSocket Provider** = “synced notebook” (live sharing).  
  - **Helia (IPFS)** = decentralized, peer‐sharable storage (global “notebook” that anyone can fetch).  

---

## Slide 14: Security & Persistence Best Practices

- **Always use HTTPS** for both WebSocket and HTTP requests.  
- **Encrypt cookies** and set secure flags.  
- **Use short‐lived tokens** and refresh tokens for authentication.  
- **Expire sessions** and revoke tokens as needed.  
- **Store only non‐sensitive data** in client‐side stores.  
- **Helia integration**:  
  - Only store content‐addressed binary updates.  
  - Clients fetch by CID; no sensitive user data is put directly into IPFS without encryption.  

---

## Slide 15: Wrap‐Up & Next Steps

1. **ToyGrid Today**  
   - Real‐time editing: Yjs + WebSocket + Tiptap.  
   - Local persistence: IndexedDBPersistence.  
   - Helia integration (in‐browser IPFS) for P2P backup.  

2. **What’s Left**  
   - Replace placeholder hash with real group hash once available.  
   - On load: fetch latest state from Helia (if any) and apply to `ydoc`.  
   - Enable peer discovery so browsers can share CIDs directly.  
   - Optionally add a UI toggle: “Use Helia” vs. “Use WebSocket only.”  

3. **Client‐Side Persistence Recap**  
   - **Cookies**, **Sessions**, and **JWTs** for authentication.  
   - **IndexedDB** for structured data (Yjs).  
   - **Helia/IPFS** for content‐addressed, decentralized storage.  

4. **Final Thoughts**  
   - Combining **local** (IndexedDB) + **P2P** (Helia) + **real‐time** (WebSocket) yields a robust, offline‐first, collaborative experience.  
   - Prioritize security (HTTPS, token management) when persisting user state.  
   - Choose the right persistence method based on data size, sensitivity, and offline requirements.  

---

## Slide 16: Questions & Discussion

- Thank you!  
- Let’s discuss any questions about ToyGrid’s architecture, persistence strategies, or Helia integration.
