# TypeScript Roadmap: Part 2 — Basic Types

---

## 1. Type Annotations

```typescript
// Primitives
let name: string = "Surya";
let age: number = 25;             // ALL numbers (int, float, NaN, Infinity)
let isActive: boolean = true;
let nothing: null = null;
let notSet: undefined = undefined;

// Type inference — TS infers the type from the value (no annotation needed!)
let city = "Chennai";  // TS knows: string
let count = 42;        // TS knows: number
// RULE: Let TS infer when the type is obvious. Annotate when it's not.
```

---

## 2. Arrays & Tuples

```typescript
// Arrays (all same type)
let nums: number[] = [1, 2, 3];
let names: Array<string> = ["a", "b"]; // Generic syntax (same thing)

// Tuples (fixed length, each position has a specific type)
let pair: [string, number] = ["Surya", 25];
pair[0].toUpperCase();  // TS knows index 0 is string ✓
pair[1].toFixed(2);     // TS knows index 1 is number ✓
// pair[2];             // ERROR! Tuple only has 2 elements.

// Readonly tuple (can't modify)
const point: readonly [number, number] = [10, 20];
// point[0] = 99;  // ERROR! Readonly.
```

---

## 3. Union & Literal Types

```typescript
// Union: Value can be one of several types
let id: string | number = "abc";
id = 42;     // OK
// id = true; // ERROR!

// Must narrow before using type-specific methods:
function printId(id: string | number) {
    if (typeof id === "string") {
        console.log(id.toUpperCase()); // TS knows: string here
    } else {
        console.log(id.toFixed(2));    // TS knows: number here
    }
}

// Literal types: Exact values, not just types
let direction: "north" | "south" | "east" | "west" = "north";
// direction = "up";  // ERROR! Not one of the allowed values.

// Combining with as const:
const config = {
    port: 3000,
    host: "localhost"
} as const;
// type: { readonly port: 3000; readonly host: "localhost" }
// Values are literal types, not just number/string!
```

---

## 4. Special Types

```typescript
// any — disables type checking (AVOID unless migrating JS → TS)
let x: any = 5;
x = "hello";     // No error
x.anything();    // No error — but will CRASH at runtime!

// unknown — safe version of any (must check type before using)
let y: unknown = 5;
// y.toFixed(2);  // ERROR! Can't use unknown directly.
if (typeof y === "number") {
    y.toFixed(2); // OK — narrowed to number ✓
}

// void — function returns nothing
function log(msg: string): void {
    console.log(msg);
}

// never — function NEVER returns (throws or infinite loop)
function throwError(msg: string): never {
    throw new Error(msg);
}

// RULE: any = "I give up on type safety" (bad)
//       unknown = "I don't know the type yet, but I'll check" (good)
```

---

## 5. Enums

```typescript
// Numeric enum (default)
enum Direction {
    Up = 0,    // If you don't assign, Up=0, Down=1, Left=2, Right=3
    Down = 1,
    Left = 2,
    Right = 3
}
let d: Direction = Direction.Up; // 0

// String enum (preferred — more readable in debug output)
enum Status {
    Active = "ACTIVE",
    Inactive = "INACTIVE",
    Pending = "PENDING"
}

// const enum — inlined at compile time (zero runtime cost!)
const enum Color {
    Red = "#FF0000",
    Green = "#00FF00"
}
let c = Color.Red; // Compiled JS: let c = "#FF0000"; (no enum object!)
```
