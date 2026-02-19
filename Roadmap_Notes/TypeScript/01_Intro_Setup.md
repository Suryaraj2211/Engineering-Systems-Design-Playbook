# TypeScript Roadmap: Part 1 — Introduction & Setup

---

## 1. What Is TypeScript?

TypeScript is a **superset of JavaScript** that adds **static type checking** at compile time. All valid JS is valid TS. TS compiles to plain JS — browsers and Node.js never see TypeScript directly.

```
WHY TYPESCRIPT EXISTS:
══════════════════════
  JavaScript:
    function add(a, b) { return a + b; }
    add("5", 3);  // "53" — silent bug! No error until runtime.
    
  TypeScript:
    function add(a: number, b: number): number { return a + b; }
    add("5", 3);  // ERROR at COMPILE time! Caught before shipping.
    
  TS catches bugs BEFORE your code runs:
    - Typos in property names
    - Wrong argument types
    - Missing function arguments
    - Null/undefined access
    - Incomplete switch statements
```

---

## 2. Setup

```bash
# Install TypeScript globally
npm install -g typescript

# Initialize a tsconfig.json
tsc --init

# Compile TypeScript to JavaScript
tsc                   # Compile all .ts files per tsconfig.json
tsc index.ts          # Compile a single file
tsc --watch           # Watch mode: recompile on file changes

# Run TypeScript directly (without compiling):
npx tsx index.ts      # tsx = TypeScript Execute (fast, uses esbuild)
npx ts-node index.ts  # ts-node (older, slower)
```

---

## 3. tsconfig.json — Essential Settings

```jsonc
{
    "compilerOptions": {
        "target": "ES2022",          // JS version to compile to
        "module": "ESNext",          // Module system (ESM)
        "strict": true,              // ENABLE ALL STRICT CHECKS (always do this!)
        "noUncheckedIndexedAccess": true, // array[i] returns T | undefined (safer!)
        "esModuleInterop": true,     // Better CommonJS/ESM interop
        "outDir": "./dist",          // Compiled output directory
        "rootDir": "./src",          // Source directory
        "declaration": true,         // Generate .d.ts type declaration files
        "sourceMap": true            // Generate source maps for debugging
    },
    "include": ["src/**/*"],
    "exclude": ["node_modules", "dist"]
}
```

### Why `strict: true` Is Non-Negotiable
```
"strict": true enables:
  strictNullChecks:       null/undefined must be handled explicitly
  noImplicitAny:          Functions must have typed parameters
  strictFunctionTypes:    Correct function type compatibility
  strictPropertyInit:     Class properties must be initialized
  
Without strict, TypeScript is just JavaScript with extra syntax.
WITH strict, TypeScript catches 90% of common bugs at compile time.
```
