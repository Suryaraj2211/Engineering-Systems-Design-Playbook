# JavaScript Roadmap: Part 6 — Modern JS & Tooling

---

## 1. ES Modules (ESM)

```javascript
// math.js — EXPORTING
export const PI = 3.14159;
export function add(a, b) { return a + b; }
export default class Calculator { /* ... */ } // One default export per file

// app.js — IMPORTING
import Calculator from "./math.js";            // Default import (any name)
import { add, PI } from "./math.js";           // Named imports (exact names)
import { add as sum } from "./math.js";        // Rename on import
import * as math from "./math.js";             // Import everything as namespace

// Dynamic import (code splitting — lazy load when needed)
const module = await import("./heavy-module.js"); // Returns a promise!
module.default; // Access default export
```

### CommonJS vs ESM
```
CommonJS (Node.js legacy):        ESM (Modern standard):
  const fs = require("fs");         import fs from "fs";
  module.exports = { add };         export { add };
  Synchronous                       Async (can be tree-shaken!)
  Runtime resolution                Static analysis at build time
```

---

## 2. Bundlers & Build Tools

```
WHY BUNDLERS EXIST:
═══════════════════
  Problem: Your app has 200 .js files, each needing its own HTTP request.
  Bundler: Combines them into 1-3 optimized files.

  MODERN TOOLS:
  ─────────────
  Vite:      Dev server with HMR. Uses Rollup for production builds. FAST.
  Webpack:   Oldest, most configurable. Complex config. Used by legacy projects.
  Rollup:    Best tree-shaking. Used for libraries.
  esbuild:   Written in Go, blazing fast. Used internally by Vite.
  
  WHAT THEY DO:
  1. Bundle:      Combine 200 files → 1-3 files
  2. Minify:      Remove whitespace, shorten variable names
  3. Tree-shake:  Remove unused exports (dead code elimination)
  4. Transpile:   Convert modern JS → older JS (via Babel/SWC)
  5. Code-split:  Split into chunks, load on demand
```

---

## 3. Package Managers

```bash
# npm (default, comes with Node.js)
npm init -y               # Create package.json
npm install lodash         # Add dependency
npm install -D jest        # Add dev dependency
npm run build              # Run script from package.json

# pnpm (faster, disk-efficient — uses hard links)
pnpm install
pnpm add lodash

# Important files:
# package.json       — Project metadata + dependencies + scripts
# package-lock.json  — Exact versions of all installed packages (commit this!)
# node_modules/      — Installed packages (NEVER commit this! Add to .gitignore)
```

---

## 4. Linting & Formatting

```javascript
// ESLint — catches bugs and enforces code style
// .eslintrc.json
{
    "extends": ["eslint:recommended"],
    "rules": {
        "no-unused-vars": "warn",
        "no-console": "off",
        "eqeqeq": "error"  // Force === instead of ==
    }
}

// Prettier — auto-formats code (no more style debates!)
// .prettierrc
{
    "semi": true,
    "singleQuote": true,
    "tabWidth": 4,
    "trailingComma": "es5"
}
```

---

## 5. Testing

```javascript
// Jest (most popular testing framework)
// math.test.js
import { add } from "./math.js";

describe("add", () => {
    test("adds two positive numbers", () => {
        expect(add(2, 3)).toBe(5);
    });
    
    test("handles negative numbers", () => {
        expect(add(-1, 1)).toBe(0);
    });
    
    test("handles floating point", () => {
        expect(add(0.1, 0.2)).toBeCloseTo(0.3); // Float precision!
    });
});

// Run: npx jest
```

---

## 6. Git Essentials

```bash
git init                        # Initialize repo
git add .                       # Stage all changes
git commit -m "feat: add auth"  # Commit with message
git push origin main            # Push to remote
git pull origin main            # Pull latest changes
git branch feature/login        # Create branch
git checkout feature/login      # Switch to branch
git merge feature/login         # Merge branch into current
git stash                       # Temporarily save uncommitted changes
git log --oneline -10           # View last 10 commits

# Conventional Commit Messages:
# feat:     New feature
# fix:      Bug fix
# docs:     Documentation changes
# refactor: Code change that doesn't fix/add
# test:     Adding tests
# chore:    Build process or tooling changes
```
