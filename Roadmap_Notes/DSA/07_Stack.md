# Chapter 07 — Stack

## Simple Explanation (ELI5)
A stack of plates — you can only add or remove from the **top**. The last plate you put on is the first one you take off. **LIFO: Last In, First Out.**

## Technical Definition
A stack is an abstract data type with operations restricted to one end (top): push, pop, and peek, all in O(1).

## Real-world Analogy
- Browser back button (most recent page first)
- Undo in text editors (most recent action undone first)
- Function call stack in programming

---

## 1. Operations & Complexity

| Operation | Description | Time |
|-----------|-------------|------|
| `push(x)` | Add to top | O(1) |
| `pop()` | Remove from top | O(1) |
| `peek()/top()` | View top without removing | O(1) |
| `isEmpty()` | Check if empty | O(1) |

## 2. Implementation

```javascript
class Stack {
  constructor() { this.items = []; }
  push(x) { this.items.push(x); }
  pop() { return this.items.pop(); }
  peek() { return this.items[this.items.length - 1]; }
  isEmpty() { return this.items.length === 0; }
  size() { return this.items.length; }
}
```

```python
class Stack:
    def __init__(self):
        self.items = []
    def push(self, x): self.items.append(x)
    def pop(self): return self.items.pop()
    def peek(self): return self.items[-1]
    def is_empty(self): return len(self.items) == 0
```

---

## 3. The Call Stack

```javascript
function a() { b(); }
function b() { c(); }
function c() { console.log("done"); }
a();

// Call stack:
// [a()] → [a(), b()] → [a(), b(), c()] → c returns
// → [a(), b()] → b returns → [a()] → a returns → []
```

> **Stack Overflow** = recursion too deep → call stack runs out of memory.

---

## 4. Classic Problems

### Valid Parentheses
```javascript
function isValid(s) {
  let stack = [];
  let map = { ')': '(', '}': '{', ']': '[' };
  for (let ch of s) {
    if ('({['.includes(ch)) stack.push(ch);
    else {
      if (stack.pop() !== map[ch]) return false;
    }
  }
  return stack.length === 0;
}
```

### Min Stack — O(1) getMin
```javascript
class MinStack {
  constructor() { this.stack = []; this.minStack = []; }
  push(val) {
    this.stack.push(val);
    this.minStack.push(
      this.minStack.length === 0 ? val : Math.min(val, this.getMin())
    );
  }
  pop() { this.stack.pop(); this.minStack.pop(); }
  top() { return this.stack[this.stack.length - 1]; }
  getMin() { return this.minStack[this.minStack.length - 1]; }
}
```

### Next Greater Element (→ see Ch26 Monotonic Stack for deep dive)
```javascript
function nextGreater(arr) {
  let result = new Array(arr.length).fill(-1);
  let stack = [];
  for (let i = 0; i < arr.length; i++) {
    while (stack.length && arr[i] > arr[stack[stack.length - 1]]) {
      result[stack.pop()] = arr[i];
    }
    stack.push(i);
  }
  return result;
}
```

### Evaluate Reverse Polish Notation
```javascript
function evalRPN(tokens) {
  let stack = [];
  for (let t of tokens) {
    if ("+-*/".includes(t)) {
      let b = stack.pop(), a = stack.pop();
      if (t === '+') stack.push(a + b);
      else if (t === '-') stack.push(a - b);
      else if (t === '*') stack.push(a * b);
      else stack.push(Math.trunc(a / b));
    } else stack.push(Number(t));
  }
  return stack[0];
}
```

### Daily Temperatures
```javascript
function dailyTemperatures(temps) {
  let result = new Array(temps.length).fill(0);
  let stack = [];
  for (let i = 0; i < temps.length; i++) {
    while (stack.length && temps[i] > temps[stack[stack.length - 1]]) {
      let idx = stack.pop();
      result[idx] = i - idx;
    }
    stack.push(i);
  }
  return result;
}
```

---

## 5. Common Mistakes
- Popping from empty stack → always check `isEmpty()` first
- Using stack when queue is needed (BFS needs queue, not stack!)
- Not recognizing that recursion IS a stack

## 6. When NOT to Use
- Need FIFO order → Queue
- Need random access → Array
- BFS traversal → Queue

---

## 7. Practice Problems

| Problem | Concept | Difficulty |
|---------|---------|-----------|
| Valid Parentheses | Matching | Easy |
| Min Stack | Auxiliary stack | Medium |
| Evaluate RPN | Stack evaluation | Medium |
| Daily Temperatures | Monotonic Stack | Medium |
| Implement Queue using Stacks | Design | Easy |
| Decode String `3[a2[c]]` | Nested stack | Medium |
| Largest Rectangle in Histogram | Monotonic Stack | Hard |
| Basic Calculator | Stack + parsing | Hard |
| Asteroid Collision | Stack simulation | Medium |
