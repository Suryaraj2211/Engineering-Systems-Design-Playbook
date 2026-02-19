# Chapter 10 — Sorting Algorithms

## Simple Explanation (ELI5)
Sorting = arranging items in order (small→big or A→Z). Like organizing cards in your hand or alphabetizing books on a shelf.

## Why It's Needed
- Enables binary search (O(log n) vs O(n))
- Duplicates become adjacent → easy to detect
- Many problems become simpler on sorted data
- Databases, file systems, and search engines rely on sorting

---

## 1. Bubble Sort — O(n²)
Repeatedly swap adjacent elements if out of order. Largest elements "bubble" to the end.

```javascript
function bubbleSort(arr) {
  for (let i = 0; i < arr.length - 1; i++) {
    let swapped = false;
    for (let j = 0; j < arr.length - 1 - i; j++) {
      if (arr[j] > arr[j+1]) {
        [arr[j], arr[j+1]] = [arr[j+1], arr[j]];
        swapped = true;
      }
    }
    if (!swapped) break;
  }
  return arr;
}
```

```python
def bubble_sort(arr):
    for i in range(len(arr) - 1):
        swapped = False
        for j in range(len(arr) - 1 - i):
            if arr[j] > arr[j+1]:
                arr[j], arr[j+1] = arr[j+1], arr[j]
                swapped = True
        if not swapped: break
    return arr
```

---

## 2. Selection Sort — O(n²)
Find minimum, swap to front. Repeat for remaining.

```javascript
function selectionSort(arr) {
  for (let i = 0; i < arr.length - 1; i++) {
    let minIdx = i;
    for (let j = i + 1; j < arr.length; j++) {
      if (arr[j] < arr[minIdx]) minIdx = j;
    }
    if (minIdx !== i) [arr[i], arr[minIdx]] = [arr[minIdx], arr[i]];
  }
  return arr;
}
```

---

## 3. Insertion Sort — O(n²)
Pick each element and insert it into its correct position in the sorted portion. Like sorting cards.

```javascript
function insertionSort(arr) {
  for (let i = 1; i < arr.length; i++) {
    let key = arr[i], j = i - 1;
    while (j >= 0 && arr[j] > key) {
      arr[j + 1] = arr[j];
      j--;
    }
    arr[j + 1] = key;
  }
  return arr;
}
```

**Best for:** Small arrays, nearly sorted data. O(n) best case.

---

## 4. Merge Sort — O(n log n) ⭐

**Divide & Conquer:** Split in half, sort each half, merge.

```javascript
function mergeSort(arr) {
  if (arr.length <= 1) return arr;
  let mid = Math.floor(arr.length / 2);
  let left = mergeSort(arr.slice(0, mid));
  let right = mergeSort(arr.slice(mid));
  return merge(left, right);
}

function merge(a, b) {
  let result = [], i = 0, j = 0;
  while (i < a.length && j < b.length) {
    if (a[i] <= b[j]) result.push(a[i++]);
    else result.push(b[j++]);
  }
  return result.concat(a.slice(i)).concat(b.slice(j));
}
```

```python
def merge_sort(arr):
    if len(arr) <= 1: return arr
    mid = len(arr) // 2
    left = merge_sort(arr[:mid])
    right = merge_sort(arr[mid:])
    return merge(left, right)

def merge(a, b):
    result, i, j = [], 0, 0
    while i < len(a) and j < len(b):
        if a[i] <= b[j]: result.append(a[i]); i += 1
        else: result.append(b[j]); j += 1
    return result + a[i:] + b[j:]
```

**Stable ✅, Space O(n), Guaranteed O(n log n).**

---

## 5. Quick Sort — O(n log n) avg ⭐

Pick pivot, partition into smaller/larger, recurse.

```javascript
function quickSort(arr, lo = 0, hi = arr.length - 1) {
  if (lo < hi) {
    let p = partition(arr, lo, hi);
    quickSort(arr, lo, p - 1);
    quickSort(arr, p + 1, hi);
  }
  return arr;
}

function partition(arr, lo, hi) {
  let pivot = arr[hi], i = lo - 1;
  for (let j = lo; j < hi; j++) {
    if (arr[j] < pivot) { i++; [arr[i], arr[j]] = [arr[j], arr[i]]; }
  }
  [arr[i+1], arr[hi]] = [arr[hi], arr[i+1]];
  return i + 1;
}
```

**In-place ✅, Space O(log n), Worst O(n²) with bad pivot.**

---

## 6. Heap Sort — O(n log n)

Build max heap, extract max repeatedly.

```javascript
function heapSort(arr) {
  let n = arr.length;
  for (let i = Math.floor(n/2) - 1; i >= 0; i--) heapify(arr, n, i);
  for (let i = n - 1; i > 0; i--) {
    [arr[0], arr[i]] = [arr[i], arr[0]];
    heapify(arr, i, 0);
  }
  return arr;
}

function heapify(arr, size, root) {
  let largest = root, l = 2*root+1, r = 2*root+2;
  if (l < size && arr[l] > arr[largest]) largest = l;
  if (r < size && arr[r] > arr[largest]) largest = r;
  if (largest !== root) {
    [arr[root], arr[largest]] = [arr[largest], arr[root]];
    heapify(arr, size, largest);
  }
}
```

---

## 7. Counting Sort — O(n + k)
Non-comparison sort. Counts occurrences. Only for non-negative integers.

```javascript
function countingSort(arr) {
  let max = Math.max(...arr);
  let count = new Array(max + 1).fill(0);
  for (let num of arr) count[num]++;
  let result = [];
  for (let i = 0; i < count.length; i++)
    for (let j = 0; j < count[i]; j++) result.push(i);
  return result;
}
```

---

## 8. Radix Sort — O(d × (n + k))
Sort by individual digits, from least significant to most.

```javascript
function radixSort(arr) {
  let max = Math.max(...arr);
  for (let exp = 1; Math.floor(max / exp) > 0; exp *= 10) {
    countSortByDigit(arr, exp);
  }
  return arr;
}

function countSortByDigit(arr, exp) {
  let output = new Array(arr.length), count = new Array(10).fill(0);
  for (let n of arr) count[Math.floor(n/exp) % 10]++;
  for (let i = 1; i < 10; i++) count[i] += count[i-1];
  for (let i = arr.length - 1; i >= 0; i--) {
    let d = Math.floor(arr[i]/exp) % 10;
    output[count[d] - 1] = arr[i];
    count[d]--;
  }
  for (let i = 0; i < arr.length; i++) arr[i] = output[i];
}
```

---

## 9. Stability
**Stable = equal elements keep their original order.**

| Stable ✅ | Unstable ❌ |
|-----------|-------------|
| Bubble, Insertion, Merge | Selection, Quick, Heap |
| Counting, Radix | |

---

## 10. Complete Comparison

| Algorithm | Best | Average | Worst | Space | Stable |
|-----------|------|---------|-------|-------|--------|
| Bubble | O(n) | O(n²) | O(n²) | O(1) | ✅ |
| Selection | O(n²) | O(n²) | O(n²) | O(1) | ❌ |
| Insertion | O(n) | O(n²) | O(n²) | O(1) | ✅ |
| Merge | O(n log n) | O(n log n) | O(n log n) | O(n) | ✅ |
| Quick | O(n log n) | O(n log n) | O(n²) | O(log n) | ❌ |
| Heap | O(n log n) | O(n log n) | O(n log n) | O(1) | ❌ |
| Counting | O(n+k) | O(n+k) | O(n+k) | O(k) | ✅ |
| Radix | O(d(n+k)) | O(d(n+k)) | O(d(n+k)) | O(n+k) | ✅ |

### When to Use Which
| Situation | Best Choice |
|-----------|-------------|
| Small n (<50) | Insertion Sort |
| Nearly sorted | Insertion Sort |
| Guaranteed O(n log n) | Merge Sort |
| General purpose | Quick Sort |
| Low memory | Heap Sort / Quick Sort |
| Integers in small range | Counting Sort |
| Need stable sort | Merge Sort |

### JavaScript Built-in Sort
```javascript
// ALWAYS provide comparator
arr.sort((a, b) => a - b);   // ascending
arr.sort((a, b) => b - a);   // descending
// V8 uses TimSort (Merge + Insertion hybrid): O(n log n)
```

---

## 11. Practice Problems

| Problem | Concept | Difficulty |
|---------|---------|-----------|
| Sort an Array | Any sort | Medium |
| Sort Colors | Dutch Flag | Medium |
| Merge Intervals | Sort + merge | Medium |
| Kth Largest Element | Quick Select | Medium |
| Largest Number | Custom sort | Medium |
| Count Inversions | Merge Sort variant | Hard |
