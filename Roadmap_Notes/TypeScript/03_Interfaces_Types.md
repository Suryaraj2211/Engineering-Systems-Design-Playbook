# TypeScript Roadmap: Part 3 — Interfaces & Type Aliases

---

## 1. Interfaces

```typescript
interface User {
    id: number;
    name: string;
    email: string;
    age?: number;           // Optional property (number | undefined)
    readonly createdAt: Date; // Cannot be changed after creation
}

const user: User = {
    id: 1,
    name: "Surya",
    email: "surya@example.com",
    createdAt: new Date()
    // age is optional, so we can omit it
};

// user.createdAt = new Date(); // ERROR! readonly
```

### Extending Interfaces
```typescript
interface Admin extends User {
    role: "admin" | "superadmin";
    permissions: string[];
}
// Admin has ALL of User's properties + role + permissions

// Multiple extension:
interface SuperAdmin extends Admin, Auditable {
    canDeleteUsers: boolean;
}
```

---

## 2. Type Aliases

```typescript
type ID = string | number;
type Point = { x: number; y: number };
type Callback = (data: string) => void;

// Intersection types (combine multiple types — like extends but for type)
type Employee = User & {
    department: string;
    salary: number;
};
```

---

## 3. Interface vs Type — When to Use Which

| Feature | Interface | Type |
|---------|-----------|------|
| Object shapes | ✅ Best for | ✅ Works |
| Extends/Implements | ✅ `extends` keyword | ✅ `&` intersection |
| Union types | ❌ Cannot | ✅ `string \| number` |
| Primitives | ❌ Cannot | ✅ `type ID = string` |
| Declaration merging | ✅ Auto-merges same name | ❌ Duplicate error |
| Classes implement | ✅ `implements` | ✅ `implements` |

```
RULE OF THUMB:
  Use interface for: Object shapes, classes, APIs, libraries
  Use type for:      Unions, primitives, complex computed types, utility types
  
  In practice: Either works for most cases. Be consistent in your project.
```

---

## 4. Index Signatures & Record

```typescript
// Index signature: Object with unknown keys
interface StringMap {
    [key: string]: number;
}
const scores: StringMap = { math: 95, science: 88 };

// Record utility type (cleaner syntax)
type ScoreMap = Record<string, number>;

// Specific keys only:
type Scores = Record<"math" | "science" | "english", number>;
const s: Scores = { math: 95, science: 88, english: 92 };
```

---

## 5. Discriminated Unions (Tagged Unions)

The most powerful pattern in TypeScript for handling multiple shapes:

```typescript
interface Circle {
    kind: "circle";   // Discriminant / tag
    radius: number;
}
interface Rectangle {
    kind: "rectangle";
    width: number;
    height: number;
}

type Shape = Circle | Rectangle;

function area(shape: Shape): number {
    switch (shape.kind) {
        case "circle":
            return Math.PI * shape.radius ** 2; // TS knows: Circle
        case "rectangle":
            return shape.width * shape.height;   // TS knows: Rectangle
        default:
            // Exhaustive check: If you add a new shape and forget to handle it,
            // this line will cause a compile error!
            const _exhaustive: never = shape;
            return _exhaustive;
    }
}
```
