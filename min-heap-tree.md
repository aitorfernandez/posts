# Min Heap Tree

I've been experimenting with doing a turn-based system where each character acts based on their frequency. For example, a character with a frequency of 4 will act twice as many times as a character with a frequency of 2.

At the heart of the turn-based system lies a min-heap. A min-heap is a tree that follows a special rule: the smallest value is always at the top. The children of a node in the tree are always greater than the parent.

The min-heap data structure is ideal for the turn-base system, because "popping" characters from the min-heap tree always will be characters with a higher frequency.

To represent the min-heap tree, we will use an array. Each node in the tree, along with its children, is represented linearly.

For example a tree with the next values:

```plaintext
     1
   /   \
  3     2
 /  \   /
9    8 5
```

Will be represented as:

```plaintext
+---+---+---+---+---+---+
| 1 | 3 | 2 | 9 | 8 | 5 |
+---+---+---+---+---+---+
```

And their array indexes:

```plaintext
+---+---+---+---+---+---+
| 1 | 3 | 2 | 9 | 8 | 5 |
+---+---+---+---+---+---+
  0   1   2   3   4   5
```

With the previous ASCII diagrams, we can say the next rule to know the parent is true:

```plaintext
floor((index - 1) / 2)
```

An for the children:

```plaintext
2 * ( index parent ) + 1
2 * ( index parent ) + 2
```

Each element in the array will be a `MinNode` object, with two main components: the `key` and the `nonce`. The key corresponds to the frequency associated with the character mentioned at the beginning of the post. The nonce is particularly valuable in situations where characters possess equal frequency.

```typescript
interface MinNode<T> {
    key: number
    nonce: number
    item: T // Characters in the turn-based system
}
```

We can now begin implementing the MinHeap.

```typescript
interface MinHeap<T = any> {
    push(item: T, key: number): void
    remove(item: T): void
}

function createMinHeap<T = any>() {
    let heap: MinNode<T>[] = []
    let nonce: number = 0

    function push(item: T, key: number) {}

    function remove(item: T) {}

    return { push, remove }
}
```

The `push` method will add a value to the `heap` array increasing the nonce for each element. This part is relatively straightforward.

```typescript
function createMinHeap<T = any>() {
    // [...]

    function push(item: T, key: number) {
        nonce += 1
        heap.push({ item, key, nonce })
    }
}
```

Now that the object is at the bottom, it's time to perform a *heapify-up* to place it in the correct position.

```typescript
function createMinHeap<T = any>() {
    // [...]

    function push(item: T, key: number) {
        // [...]
        heapifyUp(heap.length - 1)
    }

    function heapifyUp(index: number) {}
}
```

With the element at the bottom, we search for the parent node using the rule mentioned before.

```typescript
function createMinHeap<T = any>() {
    // [...]

    function getParentIndex(index: number): number {
        return Math.floor((index - 1) / 2)
    }

    function heapifyUp(index: number) {
        if (index === 0) {
            return
        }

        const parentIndex = getParentIndex(index)
    }
}
```

Once we have found the parent, we compare the keys or nonce if necessary. If the parent's key is smaller, we stop. Otherwise, we swap the elements and repeat the *heapify-up* action.

```typescript
function createMinHeap<T = any>() {
    // [...]

    function less(a: MinNode<T>, b: MinNode<T>): boolean {
        return a.key === b.key ? a.nonce < b.nonce : a.key < b.key
    }

    function swap(a: number, b: number): void {
        const t = heap[a]
        heap[a] = heap[b]
        heap[b] = t
    }

    function heapifyUp(index: number) {
        // [...]

        if (parentIndex >= 0 && less(heap[index], heap[parentIndex])) {
            swap(index, parentIndex)
            heapifyUp(parentIndex)
        }
    }
}
```

For insertion, we perform a *heapify-up* operation, for removing items, we execute the opposite, a *heapify-down* action.

First, we need to find the item to be removed.

```typescript
function createMinHeap<T = any>() {
    // [...]

    function remove(item: T) {
        const index = heap.findIndex((e) => e.item === item);
        if (index === -1) {
            return
        }
    }
}
```

Once we have the index of the element to be removed, we copy the last element into that index and discard it.

```typescript
function createMinHeap<T = any>() {
    // [...]

    function remove(item: T) {
        // [...]

        heap[index] = heap[heap.length - 1]
        heap.pop()
    }
}
```

Now it's time to move down to the correct position.

```typescript
function createMinHeap<T = any>() {

    function remove(item: T) {
        // [...]

        heapifyDown(index)
    }

    function heapifyDown(index: number) {}
}
```

To move down correctly, we need to find the children for the index that was moved in the removed position. To find the children, we utilize the rules mentioned earlier.

```typescript
function createMinHeap<T = any>() {
    // [...]

    function getLeftChildIndex(index: number): number {
        return (2 * index) + 1
    }

    function getRightChildIndex(index: number): number {
        return (2 * index) + 2
    }

    function heapifyDown(index: number) {
        const left = getLeftChildIndex(index)
        const right = getRightChildIndex(index)

        let smallest = index
    }
}
```

By using the indexes, we compare the keys of each element to find the smallest key or nonce.

```typescript
function createMinHeap<T = any>() {
    // [...]

    function heapifyDown(index: number) {
        // [...]

        if (left < heap.length && less(heap[left], heap[smallest])) {
            smallest = left
        }

        if (right < heap.length && less(heap[right], heap[smallest])) {
            smallest = right
        }
    }
}
```

If the smallest key is different from the key of the element at the index we are trying to move down, we should continue with the *heapify-down* action.

```typescript
function createMinHeap<T = any>() {
    // [...]

    function heapifyDown(index: number) {
        // [...]

        if (smallest !== index) {
            swap(index, smallest)
            heapifyDown(smallest)
        }
    }
}
```

We can use now the MinHeap

```typescript
const mh = createMinHeap()

const a = { character: 'a' }
const b = { character: 'b' }
const c = { character: 'c' }

mh.push(a, 5)
mh.push(b, 3)
mh.push(c, 2)

// []
```

And that's it, although the turn-based system is more complicated, the *heapify-up* and *heapify-down* actions play an important role at its core.
