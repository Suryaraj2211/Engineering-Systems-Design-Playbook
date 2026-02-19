# TypeScript Roadmap: Part 5 — Generics & Utility Types

---

## 1. Generics — The Most Powerful Feature

Generics let you write code that works with ANY type while keeping type safety.

```typescript
// WITHOUT generics: Must write separate functions for each type
function firstNumber(arr: number[]): number { return arr[0]; }
function firstString(arr: string[]): string { return arr[0]; }

// WITH generics: One function works for ALL types
function first<T>(arr: T[]): T | undefined {
    return arr[0];
}
first([1, 2, 3]);         // Return type: number
first(["a", "b"]);        // Return type: string
first<boolean>([true]);   // Explicit: boolean
```

### Generic Constraints
```typescript
// "T must have a .length property"
function longest<T extends { length: number }>(a: T, b: T): T {
    return a.length >= b.length ? a : b;
}
longest("hello", "hi");    // "hello" (string has .length)
longest([1, 2], [1]);      // [1, 2]  (array has .length)
// longest(10, 20);         // ERROR! number has no .length

// "K must be a key of T"
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
    return obj[key];
}
const user = { name: "Surya", age: 25 };
getProperty(user, "name"); // Return type: string
getProperty(user, "age");  // Return type: number
// getProperty(user, "foo"); // ERROR! "foo" is not a key of user
```

### Generic Classes & Interfaces
```typescript
class Stack<T> {
    private items: T[] = [];
    
    push(item: T): void { this.items.push(item); }
    pop(): T | undefined { return this.items.pop(); }
    peek(): T | undefined { return this.items[this.items.length - 1]; }
    get size(): number { return this.items.length; }
}

const numStack = new Stack<number>();
numStack.push(42);
numStack.push("hello"); // ERROR! Expected number.

interface ApiResponse<T> {
    data: T;
    status: number;
    message: string;
}
type UserResponse = ApiResponse<User>;
type ProductResponse = ApiResponse<Product>;
```

---

## 2. Built-in Utility Types — Complete Reference

```typescript
interface User {
    id: number;
    name: string;
    email: string;
    age: number;
}

// PARTIAL — All properties become optional
type UpdateUser = Partial<User>;
// { id?: number; name?: string; email?: string; age?: number; }

// REQUIRED — All properties become required
type StrictUser = Required<User>;

// READONLY — All properties become readonly
type FrozenUser = Readonly<User>;

// PICK — Select specific properties
type UserPreview = Pick<User, "id" | "name">;
// { id: number; name: string; }

// OMIT — Remove specific properties
type UserWithoutEmail = Omit<User, "email">;
// { id: number; name: string; age: number; }

// RECORD — Create an object type with specific key-value types
type UserRoles = Record<"admin" | "user" | "guest", boolean>;
// { admin: boolean; user: boolean; guest: boolean; }

// EXTRACT / EXCLUDE — Filter union types
type A = "a" | "b" | "c" | "d";
type B = Extract<A, "a" | "b">;   // "a" | "b"
type C = Exclude<A, "a" | "b">;   // "c" | "d"

// RETURNTYPE — Get the return type of a function
function createUser() { return { id: 1, name: "Surya" }; }
type NewUser = ReturnType<typeof createUser>; // { id: number; name: string; }

// PARAMETERS — Get parameter types as a tuple
type CreateUserParams = Parameters<typeof createUser>; // []
```

---

## 3. Practical Patterns

```typescript
// API handler with generics
async function fetchData<T>(url: string): Promise<T> {
    const response = await fetch(url);
    return await response.json() as T;
}

const user = await fetchData<User>("/api/user/1");
// user is fully typed: user.name, user.email etc.

// Builder pattern with generics
class QueryBuilder<T> {
    private filters: Partial<T> = {};
    
    where<K extends keyof T>(key: K, value: T[K]): this {
        this.filters[key] = value;
        return this; // Return `this` for chaining
    }
    
    build(): Partial<T> { return this.filters; }
}

new QueryBuilder<User>()
    .where("name", "Surya")
    .where("age", 25)
    // .where("name", 42)  // ERROR! name must be string, not number
    .build();
```
