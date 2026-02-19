# TypeScript Roadmap: Part 7 — Extras & Best Practices

---

## 1. Module Patterns

```typescript
// Barrel exports (re-export everything from one index.ts)
// src/models/index.ts
export { User } from "./User";
export { Product } from "./Product";
export { Order } from "./Order";

// Import from barrel:
import { User, Product, Order } from "./models";
```

---

## 2. Error Handling Patterns

```typescript
// Result type (no exceptions needed — inspired by Rust)
type Result<T, E = Error> = 
    | { ok: true; value: T }
    | { ok: false; error: E };

function parseJSON<T>(json: string): Result<T> {
    try {
        return { ok: true, value: JSON.parse(json) };
    } catch (e) {
        return { ok: false, error: e instanceof Error ? e : new Error(String(e)) };
    }
}

const result = parseJSON<User>('{"name":"Surya"}');
if (result.ok) {
    console.log(result.value.name); // TS knows: User
} else {
    console.error(result.error.message); // TS knows: Error
}
```

---

## 3. Type-Safe Event Emitter

```typescript
type EventMap = {
    login: { userId: string; timestamp: Date };
    logout: { userId: string };
    error: { code: number; message: string };
};

class TypedEmitter<T extends Record<string, any>> {
    private listeners: Partial<{ [K in keyof T]: ((data: T[K]) => void)[] }> = {};
    
    on<K extends keyof T>(event: K, callback: (data: T[K]) => void): void {
        (this.listeners[event] ??= []).push(callback);
    }
    
    emit<K extends keyof T>(event: K, data: T[K]): void {
        this.listeners[event]?.forEach(cb => cb(data));
    }
}

const emitter = new TypedEmitter<EventMap>();
emitter.on("login", (data) => {
    console.log(data.userId);     // TS knows: string ✓
    console.log(data.timestamp);  // TS knows: Date ✓
});
// emitter.emit("login", { userId: "abc" }); // ERROR! Missing timestamp.
```

---

## 4. Best Practices

```
TYPESCRIPT BEST PRACTICES:
══════════════════════════
  1. ALWAYS enable "strict": true in tsconfig.json.
  2. NEVER use `any` — use `unknown` if type is truly unknown.
  3. Prefer `interface` for object shapes, `type` for unions/utilities.
  4. Use discriminated unions instead of optional properties.
  5. Let TS infer return types (annotate only public API).
  6. Use `readonly` for data that shouldn't be mutated.
  7. Prefer `as const` over manual literal types.
  8. Use exhaustive switches with `never` to catch missing cases.
  9. Put shared types in a `types/` directory.
  10. Use generics to avoid code duplication.
```

---

## 5. TypeScript with Frameworks

```typescript
// REACT (most common TS usage)
interface Props {
    name: string;
    age?: number;
    onClick: (id: string) => void;
}

const UserCard: React.FC<Props> = ({ name, age = 0, onClick }) => {
    return <div onClick={() => onClick(name)}>{name}, {age}</div>;
};

// NODE.JS / EXPRESS
import express, { Request, Response } from "express";
const app = express();

interface CreateUserBody { name: string; email: string; }

app.post("/users", (req: Request<{}, {}, CreateUserBody>, res: Response) => {
    const { name, email } = req.body; // Fully typed!
    res.json({ id: 1, name, email });
});
```
