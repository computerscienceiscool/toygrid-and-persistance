# The Whispering Library (Fullest Technical Version with Code)

Welcome to the Whispering Library—an immersive distributed system where every **book** is a living CBOR‐encoded object, every **edit** is a CRDT update, and every revision is preserved via content addressing (CIDs), COSE signing, and IPFS/Helia storage. Below is the full story, enriched with detailed technical explanations and inline code examples to illustrate core PromiseGrid and ToyGrid concepts.

---

## Chapter 1: Arrival at the Library

In the heart of a bustling city, the Whispering Library stands as a grand testament to the power of words—and to the distributed systems that preserve them. Its towering spires reach toward the sky, and its vast halls are lined with shelves that seem to stretch into infinity. Few truly understand how this Library works, but those who enter soon realize it is unlike any other: each tome is a living **Content Addressed Document Object (CIDO)** implemented via **Yjs** CRDTs, persisted locally via **IndexedDBPersistence**, and shared across peers via **Helia** (an IPFS implementation for browsers).

### Code Example: Initializing Helia and Yjs

```javascript
import { createHelia } from 'helia'
import * as Y from 'yjs'
import { WebsocketProvider } from 'y-websocket'
import { IndexeddbPersistence } from 'y-indexeddb'
import * as cose from 'cose-js'
import * as cbor from '@ipld/dag-cbor'

// 1. Initialize a Yjs document
const ydoc = new Y.Doc()

// 2. Initialize Helia node (IPFS in browser)
const helia = await createHelia()

// 3. Setup IndexedDB persistence for offline support
const indexeddbProvider = new IndexeddbPersistence('whispering-library', ydoc)
indexeddbProvider.once('synced', () => {
  console.log('[IndexedDB] Loaded document state from browser storage')
})

// 4. Setup WebSocket provider for real-time collaboration
const wsProvider = new WebsocketProvider(
  process.env.REACT_APP_YJS_WEBSOCKET_SERVER_URL || 'ws://localhost:3000',
  'whispering-library',
  ydoc
)
wsProvider.on('status', event => {
  console.log('[WebSocket] Status:', event.status)
})

// 5. Function to publish a Yjs snapshot to Helia
async function publishSnapshot(snapshotBytes, privateKey) {
  // CBOR‐encode the snapshot
  const cborPayload = cbor.encode({
    type: 'snapshot',
    data: Array.from(snapshotBytes),
    timestamp: Date.now()
  })
  // COSE‐sign the CBOR payload
  const protectedHeader = { p: { alg: 'ES256' } }
  const signed = await cose.sign.create(protectedHeader, cborPayload, privateKey)

  // Publish to Helia blockstore and return its CID
  const cid = await helia.blockstore.put(signed)
  return cid // multihash identifier
}

// 6. Function to restore Yjs document from Helia by CID
async function restoreFromCID(rootCID, publicKey) {
  const block = await helia.blockstore.get(rootCID)
  const { payload } = await cose.sign.verify(block, publicKey)
  const decoded = cbor.decode(payload)
  const updateUint8 = new Uint8Array(decoded.data)
  Y.applyUpdate(ydoc, updateUint8)
}

// Example usage on app startup:
let rootCID = localStorage.getItem('whispering-library-root')
if (rootCID) {
  await restoreFromCID(rootCID, trustedPublicKey)
} else {
  // Create initial empty snapshot
  const initialBytes = Y.encodeStateAsUpdate(ydoc)
  const newCID = await publishSnapshot(initialBytes, librarianPrivateKey)
  localStorage.setItem('whispering-library-root', newCID.toString())
}
```

---

## Chapter 2: Forking a Tale

A week later, Elya assists Mira (folklorist) and Rafi (cartographer) to expand **“The Flight of Dragons.”** Mira and Rafi each branch from the main CRDT state, creating separate CIDs that they will update independently. This “branch” operation parallels Git forking but uses CRDT snapshots and Helia blockstore.

### Code Example: Forking a Yjs Document Branch

```javascript
// Assume `ydoc` is the current Yjs document state.

// 1. Capture current snapshot as an update
const snapshotUpdate = Y.encodeStateAsUpdate(ydoc)

// 2. Wrap in CBOR with branch metadata
const branchPayload = cbor.encode({
  branch: 'Mira',
  user: 'Mira',
  timestamp: Date.now(),
  update: Array.from(snapshotUpdate)
})

// 3. COSE‐sign the CBOR payload
const protectedHeader = { p: { alg: 'ES256' } }
const signedBranch = await cose.sign.create(protectedHeader, branchPayload, miraPrivateKey)

// 4. Publish new branch snapshot to Helia and get new CID
const miraCID = await helia.blockstore.put(signedBranch)
localStorage.setItem('flight-of-dragons-mira-cid', miraCID.toString())

// 5. Later, Mira can restore her branch by retrieving the CID:
const branchBlock = await helia.blockstore.get(miraCID)
const { payload } = await cose.sign.verify(branchBlock, miraPublicKey)
const decoded = cbor.decode(payload)
const updateUint8 = new Uint8Array(decoded.update)
Y.applyUpdate(ydoc, updateUint8)
```

Rafi’s workflow is similar but uses his own private key and CID. Each branch’s updates are appended in a DAG structure, enabling later merges.

---

## Chapter 3: The Archivist’s Lens

Elya’s new role brings her to the **Chart Alcove**, where the **ChronoLens** allows her to replay the entire history of “The Flight of Dragons” by pulling all relevant CIDs and applying their updates in chronological order.

### Code Example: Using ChronoLens to Replay History

```javascript
// 1. List all pinned CIDs in the "whispering-library" namespace
for await (const { cid } of helia.pin.ls({ namespace: 'whispering-library' })) {
  try {
    // 2. Retrieve block and verify COSE signature
    const block = await helia.blockstore.get(cid)
    const { payload } = await cose.sign.verify(block, trustedPublicKey)

    // 3. Decode CBOR payload to get Yjs update
    const decoded = cbor.decode(payload)
    const updateUint8 = new Uint8Array(decoded.update)

    // 4. Apply the update to reconstruct state
    Y.applyUpdate(ydoc, updateUint8)
  } catch(err) {
    console.error(`Failed to replay CID ${cid}`, err)
  }
}

// At this point, `ydoc` holds the full reconstructed state of the document.
```

This process mirrors how the ChronoLens “sees” every version, showing Mira’s and Rafi’s edits from their respective CIDs.

---

## Chapter 4: The Fracture in the Archives

One moonless night, Elya discovers a corrupted branch—the **Fracture Fork**—where the CBOR payload or COSE signature has become invalid. This corresponds to a broken CID chain in Helia, resulting from partial writes or tampering.

### Code Example: Detecting and Handling a Corrupted CID

```javascript
async function verifyCID(cid) {
  try {
    const block = await helia.blockstore.get(cid)
    await cose.sign.verify(block, trustedPublicKey)
    return true
  } catch(err) {
    return false
  }
}

const fractureCID = 'bafy...corruptedCid'
const isValid = await verifyCID(fractureCID)
if (!isValid) {
  console.warn(`CID ${fractureCID} is invalid. Pruning from pinset.`)
  // Remove from main pinset
  await helia.pin.rm(fractureCID, { namespace: 'whispering-library' })
  // Move to "garden-of-forgotten"
  await helia.pin.add(fractureCID, { namespace: 'garden-of-forgotten' })
}
```

Upon detecting such a failure, the system quarantines the corrupted content by unpinning from the main namespace and pinning to the “garden-of-forgotten” namespace, preventing it from affecting live merges.

---

## Chapter 5: The Merge Tribunal

To restore a clean state, the **Merge Tribunal** runs a comprehensive integrity check across all CIDs, prunes invalid ones, and merges the remaining Yjs updates to form a sanitized CRDT state.

### Code Example: Merge Tribunal Workflow

```javascript
async function runMergeTribunal(rootCIDs) {
  const validUpdates = []

  // 1. Verify every root CID
  for (const cid of rootCIDs) {
    try {
      const block = await helia.blockstore.get(cid)
      const { payload } = await cose.sign.verify(block, trustedPublicKey)
      const decoded = cbor.decode(payload)
      const updateUint8 = new Uint8Array(decoded.update)
      validUpdates.push({ update: updateUint8, timestamp: decoded.timestamp })
    } catch(err) {
      console.warn(`CID ${cid} is invalid, quarantining.`)
      await helia.pin.rm(cid, { namespace: 'whispering-library' })
      await helia.pin.add(cid, { namespace: 'garden-of-forgotten' })
    }
  }

  // 2. Sort valid updates by timestamp for deterministic merge
  validUpdates.sort((a, b) => a.timestamp - b.timestamp)

  // 3. Apply each update in order
  for (const { update } of validUpdates) {
    Y.applyUpdate(ydoc, update)
  }

  // 4. Create a new sanitized snapshot
  const sanitizedSnapshot = Y.encodeStateAsUpdate(ydoc)
  const cborPayload = cbor.encode({
    type: 'sanitized-snapshot',
    update: Array.from(sanitizedSnapshot),
    timestamp: Date.now()
  })
  const signedSanitized = await cose.sign.create({ p: { alg: 'ES256' } }, cborPayload, librarianPrivateKey)
  const newRootCID = await helia.blockstore.put(signedSanitized)

  // 5. Pin new root snapshot and update localStorage
  await helia.pin.add(newRootCID, { namespace: 'whispering-library' })
  localStorage.setItem('whispering-library-root', newRootCID.toString())

  return newRootCID
}

// Example invocation:
const allRootCIDs = []
for await (const { cid } of helia.pin.ls({ namespace: 'whispering-library' })) {
  allRootCIDs.push(cid)
}
const sanitizedCID = await runMergeTribunal(allRootCIDs)
console.log('Sanitized root CID:', sanitizedCID.toString())
```

This merges only valid CBOR payloads into a fresh CRDT snapshot, which becomes the new authoritative root for the document.

---

## Chapter 6: The Garden of Forgotten Drafts

Behind a tapestry of ivy and dust lies the **Garden of Forgotten Drafts**, a secondary Helia namespace for archived, unpinned CIDs. Elya uses her **Regrower** to restore any worthy fragment.

### Code Example: Recovering Forgotten Drafts

```javascript
async function recoverDraft(draftCID) {
  try {
    // 1. Retrieve the draft block
    const block = await helia.blockstore.get(draftCID)

    // 2. Verify COSE signature
    const { payload } = await cose.sign.verify(block, trustedPublicKey)

    // 3. Decode CBOR payload to get Yjs update
    const decoded = cbor.decode(payload)
    const updateUint8 = new Uint8Array(decoded.update)

    // 4. Merge update into live document
    Y.applyUpdate(ydoc, updateUint8)

    // 5. Pin draft CID back to main namespace
    await helia.pin.add(draftCID, { namespace: 'whispering-library' })

    // 6. Create a new merged snapshot
    const mergedSnapshot = Y.encodeStateAsUpdate(ydoc)
    const mergedPayload = cbor.encode({
      type: 'merged-snapshot',
      update: Array.from(mergedSnapshot),
      timestamp: Date.now()
    })
    const signedMerged = await cose.sign.create({ p: { alg: 'ES256' } }, mergedPayload, librarianPrivateKey)
    const newCID = await helia.blockstore.put(signedMerged)
    await helia.pin.add(newCID, { namespace: 'whispering-library' })
    localStorage.setItem('whispering-library-root', newCID.toString())

    console.log('Recovered and merged draft to new root CID:', newCID.toString())
  } catch(err) {
    console.error('Failed to recover draft', err)
  }
}

// Example retrieval of all forgotten CIDs:
const forgottenCIDs = []
for await (const { cid } of helia.pin.ls({ namespace: 'garden-of-forgotten' })) {
  forgottenCIDs.push(cid)
}

// If Elya wants to recover a specific one:
const selectedCID = forgottenCIDs[0]
await recoverDraft(selectedCID)
```

This process demonstrates how archived CRDT updates, once quarantined, can rejoin the live document through verification and merging.

---

## Epilogue: A Living, Distributed Legacy

In the Whispering Library, every reader is a guardian of memory. Each **CBOR-encoded Yjs CRDT update**, each **COSE-signed block**, each **Helia-pinned CID** is more than a collection of words—it is a living testament to the many hands that shaped it. Elya, Mira, Rafi, and the Scribes of Glass stand as humble stewards of this wondrous realm.

### Code Example: End-to-End Flow

```javascript
async function handleLocalEdit(yUpdateUint8) {
  // 1. CBOR-serialize and COSE-sign the Yjs update
  const cborPayload = cbor.encode({
    type: 'delta-update',
    update: Array.from(yUpdateUint8),
    timestamp: Date.now(),
    user: currentUser.id
  })
  const signedDelta = await cose.sign.create({ p: { alg: 'ES256' } }, cborPayload, currentUserPrivateKey)

  // 2. Publish to Helia and get a new CID for this update
  const deltaCID = await helia.blockstore.put(signedDelta)
  await helia.pin.add(deltaCID, { namespace: 'whispering-library' })

  // 3. Broadcast raw Yjs update over WebSocket for real-time sync
  wsProvider.send({
    type: 'yjs-update',
    update: Array.from(yUpdateUint8)
  })

  // 4. IndexedDBPersistence automatically stores the update locally
}

// Listen for remote WebSocket updates:
wsProvider.on('yjs-update', data => {
  const incomingUpdate = new Uint8Array(data.update)
  Y.applyUpdate(ydoc, incomingUpdate)
})

// On initial load:
(async () => {
  // a) Restore from localStorage root CID if exists
  const storedRootCID = localStorage.getItem('whispering-library-root')
  if (storedRootCID) {
    const block = await helia.blockstore.get(storedRootCID)
    const { payload } = await cose.sign.verify(block, trustedPublicKey)
    const decoded = cbor.decode(payload)
    const updateUint8 = new Uint8Array(decoded.update)
    Y.applyUpdate(ydoc, updateUint8)
  }
  // b) Or create new empty snapshot
  else {
    const initialUpdate = Y.encodeStateAsUpdate(ydoc)
    const initialPayload = cbor.encode({
      type: 'initial-snapshot',
      update: Array.from(initialUpdate),
      timestamp: Date.now()
    })
    const signedInitial = await cose.sign.create({ p: { alg: 'ES256' } }, initialPayload, librarianPrivateKey)
    const initialCID = await helia.blockstore.put(signedInitial)
    await helia.pin.add(initialCID, { namespace: 'whispering-library' })
    localStorage.setItem('whispering-library-root', initialCID.toString())
  }
})()
```

And so, the Whispering Library endures, ever ready to welcome the next dreamer who steps across its marble threshold, ready to change its pages and leave their own CIDs for future generations.

---

## Glossary of Terms

- **CBOR (Concise Binary Object Representation)**: A binary data format optimized for small storage and efficient decoding.
- **COSE (CBOR Object Signing and Encryption)**: A set of standards for digitally signing and encrypting CBOR data.
- **CID (Content Identifier)**: A multihash that uniquely identifies a piece of content stored in IPFS/Helia.
- **Yjs**: A CRDT (Conflict-free Replicated Data Type) library for building real-time collaborative applications.
- **Helia**: An IPFS implementation that runs within the browser (via WebAssembly), providing content-addressed storage.
- **IndexedDBPersistence**: A Yjs adapter that saves document updates in the browser’s IndexedDB for offline resilience.
- **WebsocketProvider**: A Yjs adapter that broadcasts and receives CRDT updates over a WebSocket server for real-time sync.
- **DAG (Directed Acyclic Graph)**: The data structure underlying IPFS/Helia, linking content by CIDs.
- **ChronoLens**: A specialized UI component that replays historical CRDT updates by fetching and verifying each CID.
- **Garden of Forgotten Drafts**: A secondary Helia namespace where quarantined or archived CIDs are stored for potential recovery.

By integrating these code examples into the story, this document serves as both a narrative and a technical reference for how PromiseGrid’s and ToyGrid’s core technologies work together to create a fully distributed, collaborative editing environment. 
