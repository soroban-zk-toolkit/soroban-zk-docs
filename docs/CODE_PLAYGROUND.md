# Interactive Code Playground for ZK Proofs

This document explains how the interactive browser-based playground for zero-knowledge proofs works, the tools it uses, and how to embed it in the documentation or any web page.

## Overview

The playground lets users write, compile, and verify ZK circuits directly in the browser — no local installation required. Users can experiment with simple proof systems, tweak circuit parameters, and observe proof generation and verification in real time.

## Architecture

```
+---------------------------+
|     Monaco Editor (UI)    |  <- user writes circom circuit here
+---------------------------+
           |
           v
+---------------------------+
|  snarkjs (WASM bundle)    |  <- compiles circuit, generates witness
+---------------------------+
           |
           v
+---------------------------+
|  Proof Generation (WASM)  |  <- Groth16 or PLONK via snarkjs.groth16.fullProve
+---------------------------+
           |
           v
+---------------------------+
|  Verification Output      |  <- snarkjs.groth16.verify / plonk.verify
+---------------------------+
```

## Tools Used

### Monaco Editor
- The same editor that powers VS Code, embedded via the `@monaco-editor/react` package.
- Provides syntax highlighting, error markers, and autocompletion for Circom syntax.
- Runs entirely in the browser with no server round-trips.

### snarkjs (WASM)
- `snarkjs` ships a WebAssembly build that runs natively in modern browsers.
- Handles:
  - Witness generation from circuit inputs
  - Proof generation (Groth16 and PLONK)
  - Proof verification against a verification key
- The WASM binary is loaded lazily on first use to keep initial page load fast.

### Circom WASM Compiler
- A WASM port of the Circom 2 compiler allows in-browser compilation of `.circom` source files.
- Compiles the user's circuit to R1CS and generates the WASM witness calculator.

### Trusted Setup (Pre-loaded)
- For demonstration purposes, a pre-computed Powers of Tau file (`pot12_final.ptau`) is bundled.
- Users can run a per-circuit setup (`snarkjs groth16 setup`) in the browser for small circuits (up to 2^12 constraints).

## How It Works — Step by Step

1. **Write a circuit** — The user types a Circom circuit in the Monaco editor pane.
2. **Compile** — Clicking "Compile" triggers the in-browser Circom compiler, producing R1CS and a witness calculator WASM.
3. **Provide inputs** — A JSON input panel lets the user specify the private and public signals.
4. **Generate witness** — snarkjs runs the witness calculator against the provided inputs.
5. **Prove** — snarkjs generates a Groth16 (or PLONK) proof using the pre-loaded trusted setup.
6. **Verify** — The proof and public signals are verified using the verification key; pass/fail is displayed.

## Embedding the Playground

### iframe Embed

```html
<iframe
  src="https://soroban-zk-toolkit.github.io/playground"
  width="100%"
  height="600px"
  style="border: none; border-radius: 8px;"
  allow="cross-origin-isolated"
></iframe>
```

> **Note:** The `allow="cross-origin-isolated"` attribute is required because snarkjs WASM uses `SharedArrayBuffer`, which requires cross-origin isolation headers (`COOP` / `COEP`).

### React Component

```jsx
import { useEffect, useRef } from 'react';

export function ZKPlayground({ initialCircuit }) {
  const iframeRef = useRef(null);

  useEffect(() => {
    const handler = (event) => {
      if (event.data?.type === 'PLAYGROUND_READY') {
        iframeRef.current.contentWindow.postMessage(
          { type: 'SET_CIRCUIT', circuit: initialCircuit },
          '*'
        );
      }
    };
    window.addEventListener('message', handler);
    return () => window.removeEventListener('message', handler);
  }, [initialCircuit]);

  return (
    <iframe
      ref={iframeRef}
      src="https://soroban-zk-toolkit.github.io/playground"
      width="100%"
      height="600"
      allow="cross-origin-isolated"
    />
  );
}
```

## Browser Requirements

| Requirement | Minimum |
|---|---|
| WebAssembly | All modern browsers (Chrome 57+, Firefox 52+, Safari 11+) |
| SharedArrayBuffer | Chrome 92+, Firefox 79+, Safari 15.2+ (requires COOP/COEP) |
| ES Modules | Chrome 61+, Firefox 60+, Safari 10.1+ |

## Performance Notes

- Circuits with fewer than 2^12 constraints compile and prove in under 5 seconds on a modern laptop.
- Larger circuits may hit browser memory limits; a warning is shown above 2^14 constraints.
- Proof generation runs in a Web Worker to keep the UI responsive.

## Local Development

```bash
git clone https://github.com/soroban-zk-toolkit/playground
cd playground
# No npm/node usage per project rules — open index.html directly with a static server
python3 -m http.server 8080
# Navigate to http://localhost:8080
```

## Related Issues and Docs

- Issue #17: Add interactive code playground for trying ZK proofs in the browser
- [snarkjs documentation](https://github.com/iden3/snarkjs)
- [Circom language reference](https://docs.circom.io)
