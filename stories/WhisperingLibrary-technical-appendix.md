# Whispering Library: Technical Appendix

This appendix provides a deep dive into the core technologies and components referenced throughout the Whispering Library narrative. It maps story elements to underlying implementations, offers chapter‐by‐chapter technical commentary, and details key libraries and protocols used in the ToyGrid and PromiseGrid projects.

---

## A. Mapping Story Elements to Technologies

1. **Library as a Distributed Data Store**  
   - **Story**: Books are more than paper and ink; they hold memories of all who touched them.  
   - **Tech**: Each “book” is represented as a CBOR (Concise Binary Object Representation) object wrapped in a COSE (CBOR Object Signing and Encryption) envelope.  
   - **Implementation**:  
     ```js
     const payload = cbor.encode({ title: "My Book", data: Uint8Array([...]) });
     const signed = await cose.sign.create({ p: { alg: "ES256" } }, payload, privateKey);
     const bookCID = await helia.blockstore.put(signed);
     ```
   - **Explanation**:  
     - CBOR serializes structured data (e.g., Yjs document snapshots or deltas).  
     - COSE provides cryptographic signatures to verify authenticity.  
     - Helia/IPFS stores the signed CBOR block and returns a CID to reference it.

2. **Branches and Forks as CID Chains and CRDT Updates**  
   - **Story**: Birthing a branch of "The Flight of Dragons" creates an independent copy.  
   - **Tech**: Yjs CRDT snapshot and update deltas are serialized to CBOR, signed via COSE, and published to Helia to generate a new CID “branch.”  
   - **Implementation**:  
     ```js
     // Fork operation (snapshot)
     const snapshot = Y.encodeStateAsUpdate(ydoc);
     const cborPayload = cbor.encode({ branch: "Mira", update: Array.from(snapshot), timestamp: Date.now() });
     const signedBranch = await cose.sign.create({ p: { alg: "ES256" } }, cborPayload, miraPrivateKey);
     const miraCID = await helia.blockstore.put(signedBranch);
     ```
   - **Explanation**:  
     - `Y.encodeStateAsUpdate(ydoc)` captures the entire CRDT state.  
     - Wrapping in CBOR+COSE creates a verifiable, content-addressed block.  
     - Publishing to Helia yields a unique CID representing the branch’s root.

3. **ChronoLens as Historical CRDT Snapshots**  
   - **Story**: ChronoLens reveals every page’s evolution over time.  
   - **Tech**: The ChronoLens fetches all pinned CIDs, verifies COSE signatures, decodes CBOR payloads, and replays Yjs updates in order.  
   - **Implementation**:  
     ```js
     for await (const { cid } of helia.pin.ls({ namespace: "whispering-library" })) {
       const block = await helia.blockstore.get(cid);
       const { payload } = await cose.sign.verify(block, trustedPublicKey);
       const decoded = cbor.decode(payload);
       const update = new Uint8Array(decoded.update);
       Y.applyUpdate(ydoc, update);
     }
     ```
   - **Explanation**:  
     - `helia.pin.ls(...)` lists all CIDs in the “whispering-library” namespace.  
     - Each block’s payload is verified and decoded to reconstruct historical CRDT updates.

4. **Fracture Fork as Malformed CBOR Blocks**  
   - **Story**: A corrupted book loops its text endlessly.  
   - **Tech**: A broken CBOR or invalid COSE signature causes Helia to return a block that fails decoding or verification.  
   - **Implementation**:  
     ```js
     try {
       const block = await helia.blockstore.get(fractureCid);
       await cose.sign.verify(block, trustedPublicKey); // throws if invalid
       const decoded = cbor.decode(block);
       Y.applyUpdate(ydoc, new Uint8Array(decoded.update));
     } catch (err) {
       // Handle corrupted block
     }
     ```
   - **Explanation**:  
     - A failed `cbor.decode` or `cose.sign.verify` indicates corruption.  
     - The invalid CID is quarantined and unpinned to prevent further errors.

5. **Merge Tribunal as Conflict Resolution and Verification**  
   - **Story**: Scribes gather to detach corrupted lines and merge valid content.  
   - **Tech**: Batch verify COSE signatures, prune invalid CIDs, and use Yjs’s merge algorithm to reapply valid updates in timestamp order.  
   - **Implementation**:  
     ```js
     async function runMergeTribunal(rootCIDs) {
       const validUpdates = [];
       for (const cid of rootCIDs) {
         try {
           const block = await helia.blockstore.get(cid);
           const { payload } = await cose.sign.verify(block, trustedPublicKey);
           const decoded = cbor.decode(payload);
           validUpdates.push({ update: new Uint8Array(decoded.update), timestamp: decoded.timestamp });
         } catch {
           // Quarantine invalid CID
           await helia.pin.rm(cid, { namespace: "whispering-library" });
           await helia.pin.add(cid, { namespace: "garden-of-forgotten" });
         }
       }
       validUpdates.sort((a, b) => a.timestamp - b.timestamp);
       validUpdates.forEach(({ update }) => Y.applyUpdate(ydoc, update));
       const sanitized = Y.encodeStateAsUpdate(ydoc);
       const sanitizedPayload = cbor.encode({ update: Array.from(sanitized), timestamp: Date.now() });
       const signedSanitized = await cose.sign.create({ p: { alg: "ES256" } }, sanitizedPayload, librarianPrivateKey);
       const newRootCID = await helia.blockstore.put(signedSanitized);
       await helia.pin.add(newRootCID, { namespace: "whispering-library" });
       return newRootCID;
     }
     ```
   - **Explanation**:  
     - Valid updates are replayed to reconstruct a clean CRDT state.  
     - The new sanitized snapshot is again CBOR‐encoded, COSE‐signed, and pinned to Helia.

6. **Garden of Forgotten Drafts as IPFS Pinset Garbage Collection**  
   - **Story**: A hidden garden stores “lost” pages awaiting revival.  
   - **Tech**: Deleted or quarantined CIDs are moved to a “garden-of-forgotten” Helia namespace, where they remain until selectively re‐pinned.  
   - **Implementation**:  
     ```js
     // Move corrupted CID to garden-of-forgotten
     await helia.pin.rm(corruptCid, { namespace: "whispering-library" });
     await helia.pin.add(corruptCid, { namespace: "garden-of-forgotten" });

     // Recover a forgotten draft
     const block = await helia.blockstore.get(draftCid);
     const { payload } = await cose.sign.verify(block, trustedPublicKey);
     const decoded = cbor.decode(payload);
     Y.applyUpdate(ydoc, new Uint8Array(decoded.update));
     await helia.pin.add(draftCid, { namespace: "whispering-library" });
     ```
   - **Explanation**:  
     - The “garden-of-forgotten” namespace preserves quarantined blocks.  
     - Individuals can choose to restore a draft by verifying and reapplying its CBOR‐encoded Yjs update.

---

## B. Chapter‑by‑Chapter Technical Commentary

### Chapter 1: Arrival at the Library
- **Library Initialization**:
  1. On application load, the front end calls `initHelia()` to instantiate a Helia node (IPFS) via WebAssembly:
     ```js
     const helia = await createHelia();
     ```
  2. Check for an existing root CID in `localStorage` (e.g., `localStorage.getItem("whispering-library-root")`). If found:
     ```js
     const block = await helia.blockstore.get(storedRootCID);
     const { payload } = await cose.sign.verify(block, trustedPublicKey);
     const decoded = cbor.decode(payload);
     const updateUint8 = new Uint8Array(decoded.update);
     Y.applyUpdate(ydoc, updateUint8);
     ```
  3. If no root CID exists, create an empty Yjs document:
     ```js
     const ydoc = new Y.Doc();
     const initialSnapshot = Y.encodeStateAsUpdate(ydoc);
     const payload = cbor.encode({ update: Array.from(initialSnapshot), timestamp: Date.now() });
     const signed = await cose.sign.create({ p: { alg: "ES256" } }, payload, librarianPrivateKey);
     const newCID = await helia.blockstore.put(signed);
     await helia.pin.add(newCID, { namespace: "whispering-library" });
     localStorage.setItem("whispering-library-root", newCID.toString());
     ```

- **Persistence and Real‐Time Sync**:
  - **IndexedDBPersistence** listens to `ydoc.on("update", callback)` and stores updates in IndexedDB:
    ```js
    const indexeddb = new IndexeddbPersistence("whispering-library", ydoc);
    indexeddb.on("synced", () => console.log("Loaded from IndexedDB"));
    ```
  - **WebSocketProvider** broadcasts raw Yjs updates for real-time collaboration:
    ```js
    const wsProvider = new WebsocketProvider(serverUrl, "whispering-library", ydoc);
    wsProvider.on("status", event => console.log("Status:", event.status));
    ydoc.on("update", updateUint8 => {
      wsProvider.send({ type: "yjs-update", update: Array.from(updateUint8) });
    });
    ```

---

### Chapter 2: Forking a Tale
- **Fork Operation**:
  1. **Snapshot**: `const snapshotUpdate = Y.encodeStateAsUpdate(ydoc);`
  2. **CBOR‐encode & COSE‐sign**:
     ```js
     const branchPayload = cbor.encode({
       branch: "Mira",
       user: "Mira",
       timestamp: Date.now(),
       update: Array.from(snapshotUpdate)
     });
     const signedBranch = await cose.sign.create({ p: { alg: "ES256" } }, branchPayload, miraPrivateKey);
     ```
  3. **Publish to Helia**: `const miraCID = await helia.blockstore.put(signedBranch);`
  4. **Pin & Store**: 
     ```js
     await helia.pin.add(miraCID, { namespace: "whispering-library" });
     localStorage.setItem("flight-of-dragons-mira-cid", miraCID.toString());
     ```
- **Branch Evolution**:
  - Subsequent Mira edits produce new Yjs update events:
    ```js
    ydoc.on("update", async updateUint8 => {
      const payload = cbor.encode({ branch: "Mira", update: Array.from(updateUint8), timestamp: Date.now() });
      const signedUpdate = await cose.sign.create({ p: { alg: "ES256" } }, payload, miraPrivateKey);
      const updateCID = await helia.blockstore.put(signedUpdate);
      await helia.pin.add(updateCID, { namespace: "whispering-library" });
    });
    ```
  - Rafi does the same with his own private key and namespace, resulting in parallel CID chains.

---

### Chapter 3: The Archivist’s Lens
- **Listing CIDs**:
  ```js
  for await (const { cid } of helia.pin.ls({ namespace: "whispering-library" })) {
    // Process each CID
  }
  ```
- **Verification & Replay**:
  ```js
  const block = await helia.blockstore.get(cid);
  const { payload } = await cose.sign.verify(block, publicKey);
  const decoded = cbor.decode(payload);
  const update = new Uint8Array(decoded.update);
  Y.applyUpdate(ydoc, update);
  ```
- **Handling Tombstones**:
  - Deleted CRDT entries remain as “tombstone” objects in the Yjs causal tree. The ChronoLens still fetches them to illustrate deletion history.

---

### Chapter 4: The Fracture in the Archives
- **Detecting Corrupted Blocks**:
  ```js
  try {
    const block = await helia.blockstore.get(fractureCid);
    await cose.sign.verify(block, trustedPublicKey);
    const decoded = cbor.decode(block);
    Y.applyUpdate(ydoc, new Uint8Array(decoded.update));
  } catch(err) {
    // Corrupted: cbor.decode or cose.verify failed
    await helia.pin.rm(fractureCid, { namespace: "whispering-library" });
    await helia.pin.add(fractureCid, { namespace: "garden-of-forgotten" });
  }
  ```
- **Effect on CRDT State**:
  - Invalid updates disrupt the Yjs merge algorithm, potentially causing repeated or missing text. Quarantine prevents these issues from propagating.

---

### Chapter 5: The Merge Tribunal
- **Batch Integrity Check**:
  ```js
  async function verifyAndCollect(rootCIDs) {
    const valid = [];
    for (const cid of rootCIDs) {
      try {
        const block = await helia.blockstore.get(cid);
        const { payload } = await cose.sign.verify(block, trustedPublicKey);
        const decoded = cbor.decode(payload);
        valid.push({ update: new Uint8Array(decoded.update), timestamp: decoded.timestamp });
      } catch {
        await helia.pin.rm(cid, { namespace: "whispering-library" });
        await helia.pin.add(cid, { namespace: "garden-of-forgotten" });
      }
    }
    return valid;
  }
  ```
- **Deterministic Merge**:
  ```js
  const validUpdates = await verifyAndCollect(rootCIDs);
  validUpdates.sort((a, b) => a.timestamp - b.timestamp);
  validUpdates.forEach(({ update }) => Y.applyUpdate(ydoc, update));
  const sanitized = Y.encodeStateAsUpdate(ydoc);
  const payloadSanitized = cbor.encode({ update: Array.from(sanitized), timestamp: Date.now() });
  const signedSanitized = await cose.sign.create({ p: { alg: "ES256" } }, payloadSanitized, librarianPrivateKey);
  const newRootCID = await helia.blockstore.put(signedSanitized);
  await helia.pin.add(newRootCID, { namespace: "whispering-library" });
  localStorage.setItem("whispering-library-root", newRootCID.toString());
  ```

---

### Chapter 6: The Garden of Forgotten Drafts
- **Listing Forgotten CIDs**:
  ```js
  const forgotten = [];
  for await (const { cid } of helia.pin.ls({ namespace: "garden-of-forgotten" })) {
    forgotten.push(cid);
  }
  ```
- **Recovery Workflow**:
  ```js
  async function recoverDraft(draftCID) {
    const block = await helia.blockstore.get(draftCID);
    const { payload } = await cose.sign.verify(block, trustedPublicKey);
    const decoded = cbor.decode(payload);
    const update = new Uint8Array(decoded.update);
    Y.applyUpdate(ydoc, update);
    await helia.pin.add(draftCID, { namespace: "whispering-library" });
    const merged = Y.encodeStateAsUpdate(ydoc);
    const mergedPayload = cbor.encode({ update: Array.from(merged), timestamp: Date.now() });
    const signedMerged = await cose.sign.create({ p: { alg: "ES256" } }, mergedPayload, librarianPrivateKey);
    const newCID = await helia.blockstore.put(signedMerged);
    await helia.pin.add(newCID, { namespace: "whispering-library" });
    localStorage.setItem("whispering-library-root", newCID.toString());
  }
  ```

---

## C. Underlying Component Details

### 1. Yjs: CRDT Engine
- **Document Model**:  
  Yjs uses CRDTs (Conflict‐free Replicated Data Types) to manage concurrent edits. Each operation (insert, delete, format) is captured as a binary `Uint8Array` update.  
- **Serialization**:  
  ```js
  const update = Y.encodeStateAsUpdate(ydoc); // Uint8Array
  ```
- **Applying Updates**:  
  ```js
  Y.applyUpdate(ydoc, updateUint8Array);
  ```
- **Garbage Collection**:  
  Yjs periodically consolidates past updates into a snapshot and prunes obsolete deltas via:
  ```js
  const snapshot = Y.encodeStateAsUpdate(ydoc);
  ydoc.gc(); // Clean up old tombstones and merge snapshots
  ```

### 2. Helia: IPFS in the Browser
- **Blockstore API**:  
  - `helia.blockstore.put(bytes)` → returns a `CID` (multihash).  
  - `helia.blockstore.get(cid)` → returns raw block bytes.  
- **Pinning API**:  
  - `helia.pin.add(cid, { namespace })` → prevents garbage collection for that CID in the specified namespace.  
  - `helia.pin.rm(cid, { namespace })` → removes the pin, allowing GC if no other references exist.  
- **Namespaces**:  
  - `"whispering-library"`: Active live pinset containing valid, current CIDs.  
  - `"garden-of-forgotten"`: Archived pinset for quarantined or deleted CIDs.

### 3. COSE & CWT: Integrity and Authentication
- **Signing with COSE Sign1**:
  ```js
  const protectedHeader = { p: { alg: "ES256" } };
  const signed = await cose.sign.create(protectedHeader, payloadBytes, privateKey);
  ```
- **Verifying COSE Sign1**:
  ```js
  const { payload } = await cose.sign.verify(signedBytes, publicKey);
  ```
- **CWT (CBOR Web Token)**:  
  In this context, a COSE‐signed CBOR payload containing metadata claims (e.g., `sub`, `iat`, `branch`, `user`) serves as a lightweight CWT.

### 4. WebsocketProvider: Real-Time Collaboration
- **Initialization**:
  ```js
  const wsProvider = new WebsocketProvider("wss://<server>/yjs", "whispering-library", ydoc);
  wsProvider.on("status", event => console.log("WebSocket status:", event.status));
  ```
- **Update Broadcasting**:
  ```js
  ydoc.on("update", updateUint8 => {
    wsProvider.send({ type: "yjs-update", update: Array.from(updateUint8) });
  });
  ```
- **Receiving Remote Updates**:
  ```js
  wsProvider.on("yjs-update", data => {
    const incoming = new Uint8Array(data.update);
    Y.applyUpdate(ydoc, incoming);
  });
  ```

### 5. IndexedDBPersistence: Browser Local Store
- **Initialization**:
  ```js
  const indexeddbProvider = new IndexeddbPersistence("whispering-library", ydoc);
  indexeddbProvider.once("synced", () => {
    console.log("Loaded from IndexedDB");
  });
  ```
- **Storage Flow**:
  1. On `ydoc` update, IndexedDBPersistence serializes the update and stores it in the `objectStore` named `"whispering-library"`.  
  2. On page load, it reads all stored updates and replays them via `Y.applyUpdate`.

### 6. Yjs Merge & Garbage Collection
- **Causal Tree**:
  - Yjs internally maintains a directed acyclic graph (DAG) of CRDT operations.  
  - When updates arrive from different peers, Yjs merges them by computing a three‐way CRDT merge, guaranteeing eventual consistency.
- **Snapshot & Cleanup**:
  - Snapshot creation:
    ```js
    const snapshot = Y.encodeStateAsUpdate(ydoc);
    ```
  - Garbage collection:
    ```js
    ydoc.gc(); // Removes old tombstones and reduces memory footprint
    ```

---

## D. End‑to‑End Flow

### 1. User Creates a New Edit
```js
// a) Generate local Yjs update
const localUpdate = Y.encodeStateAsUpdate(ydoc);

// b) CBOR‐encode and COSE‐sign
const payload = cbor.encode({ type: "delta-update", update: Array.from(localUpdate), timestamp: Date.now(), user: currentUser });
const signed = await cose.sign.create({ p: { alg: "ES256" } }, payload, currentUserPrivateKey);

// c) Publish to Helia and pin
const updateCID = await helia.blockstore.put(signed);
await helia.pin.add(updateCID, { namespace: "whispering-library" });

// d) Broadcast raw Yjs update for real-time sync
wsProvider.send({ type: "yjs-update", update: Array.from(localUpdate) });
```

### 2. Another User Joins Later
```js
// a) Restore from root CID if exists
const storedRoot = localStorage.getItem("whispering-library-root");
if (storedRoot) {
  const block = await helia.blockstore.get(storedRoot);
  const { payload } = await cose.sign.verify(block, trustedPublicKey);
  const decoded = cbor.decode(payload);
  Y.applyUpdate(ydoc, new Uint8Array(decoded.update));
}

// b) Otherwise initialize new state as in Chapter 1

// c) Establish WebSocket connection
wsProvider.connect();

// d) Apply any missed remote updates received via WebSocket
wsProvider.on("yjs-update", data => {
  Y.applyUpdate(ydoc, new Uint8Array(data.update));
});
```

### 3. Handling Corruption and Quarantine
```js
async function quarantineCID(cid) {
  try {
    const block = await helia.blockstore.get(cid);
    await cose.sign.verify(block, trustedPublicKey);
  } catch {
    await helia.pin.rm(cid, { namespace: "whispering-library" });
    await helia.pin.add(cid, { namespace: "garden-of-forgotten" });
  }
}
```

### 4. Recovering Forgotten Drafts
```js
const forgottenCIDs = [];
for await (const { cid } of helia.pin.ls({ namespace: "garden-of-forgotten" })) {
  forgottenCIDs.push(cid);
}
await recoverDraft(forgottenCIDs[0]); // from Chapter 6 example
```

---

## Glossary: Story Terms to Technology Mappings

- **Whisper Quills** → COSE Sign1 keys used to sign CBOR payloads.  
- **Books (Tomes)** → CBOR‐serialized Yjs document snapshots or updates.  
- **CIDs** → IPFS Content Identifiers (multihashes) for CBOR blocks in Helia.  
- **ChronoLens** → A UI that replays historical CRDT updates by fetching and verifying each CID.  
- **Fracture Fork** → A corrupted chain of CIDs indicating invalid CBOR or COSE signature errors.  
- **Merge Tribunal** → Automated process of verifying CIDs, pruning invalid blocks, and merging valid CRDT updates.  
- **Garden of Forgotten Drafts** → Secondary Helia namespace for quarantined or archived CIDs.  
- **Helia** → IPFS implementation in browser (WebAssembly) providing content-addressed storage.  
- **Yjs** → CRDT engine for real-time collaborative editing.  
- **WebsocketProvider** → Yjs adapter for broadcasting and receiving updates via WebSocket.  
- **IndexedDBPersistence** → Yjs adapter for storing updates in browser IndexedDB for offline resilience.  
- **COSE** → Cryptographic envelope standard for signing and verifying CBOR payloads.  
- **CBOR** → Concise Binary Object Representation, a compact binary data format.

This technical appendix lays out the full set of underlying mechanisms behind the Whispering Library narrative, illustrating how ToyGrid and PromiseGrid components interoperate to create a robust, distributed, collaborative editing experience.
