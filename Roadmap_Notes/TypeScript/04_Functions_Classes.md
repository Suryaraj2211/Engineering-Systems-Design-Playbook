# TypeScript Roadmap: Part 4 — Functions & Classes

---

## 1. Function Types

```typescript
// Parameter and return types
function add(a: number, b: number): number {
    return a + b;
}

// Arrow function with type
const multiply = (a: number, b: number): number => a * b;

// Optional & default parameters
function greet(name: string, greeting: string = "Hello"): string {
    return `${greeting}, ${name}!`;
}
greet("Surya");          // "Hello, Surya!"
greet("Surya", "Hey");   // "Hey, Surya!"

// Rest parameters
function sum(...nums: number[]): number {
    return nums.reduce((a, b) => a + b, 0);
}

// Function type alias
type MathFn = (a: number, b: number) => number;
const divide: MathFn = (a, b) => a / b; // Parameters inferred from type!

// Overloads (one function, multiple signatures)
function format(value: string): string;
function format(value: number): string;
function format(value: string | number): string {
    if (typeof value === "string") return value.trim();
    return value.toFixed(2);
}
```

---

## 2. Classes

```typescript
class User {
    // Property declarations with access modifiers
    public name: string;        // Accessible everywhere (default)
    private password: string;    // Only inside this class
    protected email: string;     // This class + subclasses
    readonly id: number;         // Cannot change after constructor
    
    // Shorthand: Declare + initialize in constructor parameters!
    constructor(
        public name: string,     // Creates this.name automatically
        private password: string,
        readonly id: number
    ) {}
    
    // Method
    greet(): string {
        return `Hi, I'm ${this.name}`;
    }
    
    // Getter & Setter
    get displayName(): string {
        return this.name.toUpperCase();
    }
    set displayName(value: string) {
        this.name = value.trim();
    }
    
    // Static (belongs to class, not instances)
    static create(name: string): User {
        return new User(name, "default", Date.now());
    }
}

const user = User.create("Surya");
user.name;           // "Surya" (public)
// user.password;    // ERROR! Private.
user.displayName;    // "SURYA" (getter)
```

---

## 3. Abstract Classes

```typescript
abstract class Shape {
    abstract area(): number;     // MUST be implemented by subclasses
    abstract perimeter(): number;
    
    describe(): string {         // Can have concrete methods too
        return `Area: ${this.area()}, Perimeter: ${this.perimeter()}`;
    }
}

class Circle extends Shape {
    constructor(private radius: number) { super(); }
    
    area(): number { return Math.PI * this.radius ** 2; }
    perimeter(): number { return 2 * Math.PI * this.radius; }
}

// const shape = new Shape();  // ERROR! Cannot instantiate abstract class.
const circle = new Circle(5);
circle.describe(); // "Area: 78.54, Perimeter: 31.42"
```

---

## 4. Implementing Interfaces

```typescript
interface Serializable {
    serialize(): string;
    deserialize(data: string): void;
}

interface Printable {
    print(): void;
}

class Document implements Serializable, Printable {
    constructor(public title: string, public content: string) {}
    
    serialize(): string {
        return JSON.stringify({ title: this.title, content: this.content });
    }
    
    deserialize(data: string): void {
        const parsed = JSON.parse(data);
        this.title = parsed.title;
        this.content = parsed.content;
    }
    
    print(): void {
        console.log(`[${this.title}]\n${this.content}`);
    }
}
```
