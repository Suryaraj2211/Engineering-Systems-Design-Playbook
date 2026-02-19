# TypeScript Roadmap: Part 6 — Advanced Concepts

---

## 1. Type Guards & Narrowing

```typescript
// typeof guard (primitives)
function process(val: string | number) {
    if (typeof val === "string") {
        val.toUpperCase(); // TS narrows to string
    } else {
        val.toFixed(2);    // TS narrows to number
    }
}

// instanceof guard (classes)
function handle(err: Error | TypeError) {
    if (err instanceof TypeError) {
        console.log("Type error:", err.message);
    }
}

// "in" guard (check if property exists)
interface Fish { swim(): void; }
interface Bird { fly(): void; }

function move(animal: Fish | Bird) {
    if ("swim" in animal) {
        animal.swim(); // TS narrows to Fish
    } else {
        animal.fly();  // TS narrows to Bird
    }
}

// Custom type guard (user-defined)
function isString(val: unknown): val is string {
    return typeof val === "string";
}
// After calling isString(x), TS knows x is string inside the if block.
```

---

## 2. Type Assertions

```typescript
// Assert when you know more than TS
const input = document.getElementById("email") as HTMLInputElement;
input.value = "surya@example.com";

// Double assertion (DANGEROUS — last resort only)
const x = "hello" as unknown as number; // Forces TS to shut up. Almost always wrong.

// Non-null assertion (!)
function getUser(): User | null { /* ... */ }
const user = getUser()!; // "I KNOW this isn't null." (Crashes if it IS null!)

// RULE: Prefer narrowing over assertions. Assertions bypass safety.
```

---

## 3. Conditional Types

```typescript
// Syntax: T extends U ? X : Y
type IsString<T> = T extends string ? "yes" : "no";
type A = IsString<string>;   // "yes"
type B = IsString<number>;   // "no"

// Practical: Extract array element type
type ElementType<T> = T extends (infer E)[] ? E : never;
type X = ElementType<string[]>;  // string
type Y = ElementType<number[]>;  // number
type Z = ElementType<boolean>;   // never (not an array)

// Practical: Extract Promise result type
type Awaited<T> = T extends Promise<infer R> ? R : T;
type Result = Awaited<Promise<string>>; // string
```

---

## 4. Template Literal Types

```typescript
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";
type ApiRoute = `/api/${string}`;
type Endpoint = `${HttpMethod} ${ApiRoute}`;

// "GET /api/users" | "POST /api/users" | "GET /api/..." | ...
const endpoint: Endpoint = "GET /api/users"; // ✓
// const bad: Endpoint = "PATCH /api/users";  // ERROR! PATCH not in HttpMethod

// String manipulation types
type Upper = Uppercase<"hello">;      // "HELLO"
type Lower = Lowercase<"HELLO">;      // "hello"
type Cap = Capitalize<"hello">;       // "Hello"
type Uncap = Uncapitalize<"Hello">;   // "hello"
```

---

## 5. Mapped Types

```typescript
// Make all properties of T optional
type MyPartial<T> = {
    [K in keyof T]?: T[K];
};

// Make all properties of T readonly
type MyReadonly<T> = {
    readonly [K in keyof T]: T[K];
};

// Make all properties nullable
type Nullable<T> = {
    [K in keyof T]: T[K] | null;
};

// Rename keys with template literals
type Getters<T> = {
    [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};
type UserGetters = Getters<{ name: string; age: number }>;
// { getName: () => string; getAge: () => number; }
```

---

## 6. Declaration Files (.d.ts)

```typescript
// When using a JS library without types, create a .d.ts file:
// types/mylib.d.ts
declare module "mylib" {
    export function doSomething(value: string): number;
    export interface Config {
        debug: boolean;
        timeout: number;
    }
}

// For global variables (e.g., injected by a script tag):
declare const API_KEY: string;
declare function analytics(event: string, data: object): void;
```
