# New Developer Guide: ToyGrid Integration with PromiseGrid

Welcome to the team! This guide will help you understand:
1. **What ToyGrid is**, its components, and how it works.
2. **How ToyGrid fits into PromiseGrid** and how to integrate it.
3. **Key concepts** to get you started quickly as a new developer.

---

## Table of Contents

1. [Overview: What Is PromiseGrid?](#overview-what-is-promisegrid)
2. [Overview: What Is ToyGrid?](#overview-what-is-toygrid)
3. [ToyGrid Architecture and Components](#toygrid-architecture-and-components)
   - Simulator and Protocols
   - Front-End Editor (React + Yjs)
   - Persistence Layers (IndexedDB, WebSocket, Helia)
4. [How ToyGrid Fits into PromiseGrid](#how-toygrid-fits-into-promisegrid)
5. [Getting Started: Local Setup](#getting-started-local-setup)
6. [Running ToyGrid Independently](#running-toygrid-independently)
7. [Integrating ToyGrid with PromiseGrid](#integrating-toygrid-with-promisegrid)
8. [Conclusion and Next Steps](#conclusion-and-next-steps)

---

## Overview: What Is PromiseGrid?

**PromiseGrid** is the overarching system for decentralized, promise-based data exchange across a network of peers. It aims to:

- Encode promises (or data transactions) in CBOR Web Tokens (CWTs).
- Store and verify these tokens using COSE for cryptographic integrity.
- Route data through a distributed, decentralized protocol (e.g., hyper-tree or other overlays).

**Key Concepts for PromiseGrid**:
- **CWT (CBOR Web Token)**: A compact, binary token format for signed data (“claims”).
- **COSE**: A cryptographic envelope format used to sign CWTs.
- **Routing Layer**: A structured peer graph (e.g., hyper-tree) for message exchange.
- **Persistence and Replication**: Using IPFS/Helia or other decentralized storage for long-term data resilience.

You don’t need to know all details of PromiseGrid immediately, but understand that ToyGrid is a sub-module used for prototyping distributed message routing and collaborative editing within the larger system.

---

## Overview: What Is ToyGrid?

**ToyGrid** is a lightweight, “toy” proof-of-concept for:
1. **Simulating distributed message routing** (Priority 1 in our roadmap).
2. **Building a collaborative editor** with offline-first and peer-to-peer persistence (Priority 2).
3. Experiments in decentralized data exchange and CRDT merging (Priority 3).

For a new developer, focus on **Priority 1** and **Priority 2** features:
- **Priority 1**: Swap in a “random-message” protocol in the simulator, introduce a “ThreatAgent” for hyper-tree routing, and test with unit tests.
- **Priority 2**: Build a collaborative browser-based editor using Yjs, with offline persistence (IndexedDB), real-time sync (WebSockets), and IPFS/Helia integration.

ToyGrid helps illustrate how PromiseGrid’s routing and token-passing ideas can be applied to real collaborative applications.

---

## ToyGrid Architecture and Components

Below are the main components you’ll encounter in ToyGrid:

### 1. Simulator and Protocols
- **Location**: `x/simulator` and `x/experimental_protocols.js`
- **Purpose**: Simulate a network of nodes (A, B, C) to demonstrate message routing protocols.
- **Key Files**:
  - `x/simulator/registry_placeholder.js`: A simple registry to register protocol handlers (e.g., “hello”, “random”).
  - `x/simulator/index.js`: Bootstraps the simulator, sets up nodes A/B/C, and invokes the chosen protocol handler once.
  - `x/experimental_protocols.js`: Defines:
    - `SID_RANDOM`: A protocol identifier for random-message routing.
    - `randomMessageHandler(agent, message)`: Picks a random peer and forwards the message.
    - `ThreatAgent`: Builds a simple binary hyper-tree neighbor map and exposes `getPeers()`.
  - **Unit Tests** (in `x/tests`):
    - `experimental_protocols.test.js`: Tests ThreatAgent neighbor generation and protocol handler registration.
    - `edge_case_protocols.test.js`: Tests “hello” and “random” handlers when there are no peers or a single peer.
- **How It Works**:
  1. Choose protocol via CLI: `node x/simulator/index.js --protocol hello` or `--protocol random`.
  2. For “hello”: Node A forwards a “Hello from A” message to Node B (one-hop).
  3. For “random”: Node A uses ThreatAgent to pick a random neighbor (B or C), forwards once, and stops.
  4. Unit tests verify correct registration and edge behaviors.

### 2. Front-End Editor (React + Yjs)
- **Location**: `editor/src`
- **Purpose**: Provide a browser-based collaborative editor demo.
- **Key Files**:
  - `src/App.jsx`: Main React component that:
    1. Imports CRDT and persistence modules.
    2. Initializes a Yjs document (`ydoc`).
    3. Configures `WebsocketProvider` to sync in real-time with a Y-WebSocket server.
    4. Configures `IndexeddbPersistence` to save the Yjs document in local IndexedDB.
    5. Uses `@tiptap/react`, `StarterKit`, `Collaboration`, and `CollaborationCursor` extensions to render and handle editing.
  - `src/helia.js` (to add): Initializes a Helia (IPFS) node in the browser for peer-to-peer content-addressed storage.
  - `src/MenuBar.jsx`: A toolbar UI component for the editor (formatting, user actions).
- **How It Works**:
  1. On load, `App.jsx` sets up IndexedDBPersistence to load any saved document.
  2. It sets up `WebsocketProvider` to connect to a WebSocket server (e.g., `europa.d4.t7a.org:3000`).
  3. Yjs CRDT merges local edits and remote edits automatically.
  4. Cursor positions and usernames/colors are shared via `CollaborationCursor`.
  5. Helia (to be added) will store CRDT updates in a content-addressed blockstore.

### 3. Persistence Layers
- **IndexedDBPersistence**:
  - Stores Yjs updates locally so users can reload without losing work.
  - Fires an event `once('synced')` when local data is loaded into memory.
- **WebsocketProvider**:
  - Broadcasts Yjs updates to all connected clients in a “room” (using a shared server).
  - Provides “awareness” (cursor positions, user presence).
- **Helia (IPFS)**:
  - Will provide long-term, content-addressed storage in the browser.
  - Offers a peer-to-peer alternative to the WebSocket server.

---

## How ToyGrid Fits into PromiseGrid

ToyGrid serves as a **sandbox** to prototype the core concepts we’ll eventually use in PromiseGrid:

1. **Routing and Message Delivery**  
   - ToyGrid’s simulator and ThreatAgent illustrate how peer nodes can forward messages along a hyper-tree overlay.  
   - PromiseGrid will use a similar routing layer (possibly more robust) to deliver CWT payloads among peers.

2. **CRDT-Based State Synchronization**  
   - The Yjs-based editor in ToyGrid shows real-time document merging.  
   - In PromiseGrid, certain data (e.g., collaborative promises, stateful tasks) may also use CRDTs to merge concurrent updates.

3. **Offline Resilience and Long-Term Storage**  
   - IndexedDBPersistence demonstrates offline recovery.  
   - Helia integration in ToyGrid previews how PromiseGrid will store and retrieve signed CWTs on IPFS, allowing peers to come online later and fetch updates.

4. **Awareness and User Presence**  
   - CollaborationCursor in ToyGrid shares user presence and cursor colors.  
   - PromiseGrid may use a similar “awareness” mechanism for tracking who has accepted or is editing a promise.

By understanding ToyGrid’s architecture, the new developer will have a clearer sense of how to integrate these patterns into the broader PromiseGrid codebase.

---

## Getting Started: Local Setup

1. **Clone the Repository**  
   ```
   git clone https://<your-repo-url>/promisegrid.git
   cd promisegrid
   ```

2. **Install Root Dependencies**  
   In the project root, run:
   ```
   npm install
   ```

3. **Run the ToyGrid Simulator** (Priority 1)  
   ```
   cd x
   npm install   # if unit tests require additional packages
   node simulator/index.js --protocol hello
   node simulator/index.js --protocol random
   node tests/experimental_protocols.test.js
   node tests/edge_case_protocols.test.js
   ```

4. **Run the Editor Demo** (Priority 2)  
   ```
   cd editor
   npm install
   npm start   # starts React dev server
   ```
   - Open your browser to `http://localhost:3000` to see the collaborative editor.
   - Make edits in one browser tab, open a second tab to observe real-time sync.

---

## Running ToyGrid Independently

If you ever need to focus on ToyGrid without the rest of PromiseGrid:

1. **Navigate to the ToyGrid subdirectories**  
   ```
   cd promisegrid/x
   ```

2. **Simulator**  
   ```
   node simulator/index.js --protocol hello
   node simulator/index.js --protocol random
   npm test   # runs the two unit test files
   ```

3. **Editor**  
   ```
   cd ../editor
   npm install
   npm start
   ```

---

## Integrating ToyGrid with PromiseGrid

Once you understand how ToyGrid components work, here’s how to integrate them with the larger PromiseGrid system:

1. **Use a Shared Group Identifier**  
   - PromiseGrid will define a unique “group hash” for each collaborative document or promise context.  
   - Pass this group hash into ToyGrid’s `App.jsx` (e.g., as a prop or environment variable) to initialize Yjs and Helia under that namespace.  

2. **Replace WebSocket Persistence with PromiseGrid’s Network**  
   - Instead of using `y-websocket` to connect to a central server, PromiseGrid may provide a peer-to-peer messaging layer to broadcast CWTs or CRDT updates.  
   - Modify `WebsocketProvider` instantiation to use a `PromiseGridProvider` that leverages PromiseGrid’s message routing.

3. **Store and Fetch CWTs via IPFS**  
   - In ToyGrid’s Helia layer, after calling `helia.blockstore.put(update)`, you obtain a CID for each Yjs update.  
   - PromiseGrid can wrap that CID in a CWT (claiming the “promise” that document version `cid` is valid), sign it via COSE, and broadcast it using the routing layer.  
   - Other peers can fetch that CWT, verify its signature, then extract the CID and call `helia.blockstore.get(cid)` to retrieve the update and merge it into their local Yjs doc.

4. **Enable Cross-Origin or Peer Discovery**  
   - ToyGrid currently assumes a single origin (`localhost:3000` or `europa.d4.t7a.org`).  
   - When integrating, PromiseGrid may run across multiple domains or nodes. You might need to:
     - Implement cross-origin messaging (e.g., via `postMessage` between iframes or messaging APIs).  
     - Use a service discovery mechanism in PromiseGrid to locate peers that hold relevant CIDs.

5. **Testing and Code Reviews**  
   - Any modifications to `App.jsx`, the simulator, or Helia integration should go through code review.  
   - Use the line numbers and insertion points from this guide to propose minimal, incremental PRs.  
   - For each PR, include a clear explanation of why you added or changed code (e.g., “Now reading Helia CID from environment variable”, “Switching provider from WebSocket to PromiseGrid messaging layer”).

---

## Conclusion and Next Steps

1. **Review this Guide**  
   - Familiarize yourself with ToyGrid’s simulator, editor, and persistence approaches.  
   - Run the demo on your local machine to see how updates flow from the editor to IndexedDB and the WebSocket server.

2. **Learn PromiseGrid’s Core API**  
   - Read the PromiseGrid documentation on CWT generation, COSE signing, and the routing layer.  
   - Identify how to wrap Yjs updates into CWTs and broadcast them via the PromiseGrid network.

3. **Build Incremental Integrations**  
   1. Start by allowing ToyGrid to read a “group hash” from an environment variable (e.g., `REACT_APP_PROMISEGRID_NAMESPACE`).  
   2. Create a minimal `PromiseGridProvider` stub that logs messages instead of sending them.  
   3. Gradually replace `WebsocketProvider` calls with calls to this new provider.  
   4. Test end-to-end: Two clients share a document under a PromiseGrid namespace.

4. **Ask Questions**  
   - Whenever you’re unsure how a piece fits together, refer back to this guide or ask the team.  
   - Keep each change small and self-contained, and request code reviews explaining your intent.

---

**Welcome aboard!** We’re excited to have you contributing to ToyGrid and PromiseGrid. As you get more comfortable, you’ll help shape how these pieces come together into a robust, decentralized platform. Good luck, and feel free to reach out with any questions.
