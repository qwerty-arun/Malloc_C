### Full implementation (from the tutorial)

```c
/* malloc.c - simple malloc / free / realloc / calloc implementation
   from "Malloc tutorial" by Dan Luu (https://danluu.com/malloc-tutorial/) */

#include <assert.h>
#include <string.h>
#include <sys/types.h>
#include <unistd.h>

struct block_meta {
  size_t size;
  struct block_meta *next;
  int free;
  int magic; // For debugging only.
};

#define META_SIZE sizeof(struct block_meta)

void *global_base = NULL;

struct block_meta *find_free_block(struct block_meta **last, size_t size) {
  struct block_meta *current = global_base;
  while (current && !(current->free && current->size >= size)) {
    *last = current;
    current = current->next;
  }
  return current;
}

struct block_meta *request_space(struct block_meta* last, size_t size) {
  struct block_meta *block;
  block = sbrk(0);
  void *request = sbrk(size + META_SIZE);
  assert((void*)block == request); // Not thread safe.
  if (request == (void*) -1) {
    return NULL; // sbrk failed.
  }
  if (last) { // NULL on first request.
    last->next = block;
  }
  block->size = size;
  block->next = NULL;
  block->free = 0;
  block->magic = 0x12345678;
  return block;
}

void *malloc(size_t size) {
  struct block_meta *block;
  // TODO: align size?
  if (size <= 0) {
    return NULL;
  }
  if (!global_base) { // First call.
    block = request_space(NULL, size);
    if (!block) {
      return NULL;
    }
    global_base = block;
  } else {
    struct block_meta *last = global_base;
    block = find_free_block(&last, size);
    if (!block) { // Failed to find free block.
      block = request_space(last, size);
      if (!block) {
        return NULL;
      }
    } else {      // Found free block
      // TODO: consider splitting block here.
      block->free = 0;
      block->magic = 0x77777777;
    }
  }
  return(block+1);
}

struct block_meta *get_block_ptr(void *ptr) {
  return (struct block_meta*)ptr - 1;
}

void free(void *ptr) {
  if (!ptr) {
    return;
  }
  // TODO: consider merging blocks once splitting blocks is implemented.
  struct block_meta* block_ptr = get_block_ptr(ptr);
  assert(block_ptr->free == 0);
  assert(block_ptr->magic == 0x77777777 || block_ptr->magic == 0x12345678);
  block_ptr->free = 1;
  block_ptr->magic = 0x55555555;
}

void *realloc(void *ptr, size_t size) {
  if (!ptr) {
    // NULL ptr. realloc should act like malloc.
    return malloc(size);
  }

  struct block_meta* block_ptr = get_block_ptr(ptr);
  if (block_ptr->size >= size) {
    // We have enough space. Could free some once we implement split.
    return ptr;
  }
  // Need to really realloc. Malloc new space and free old space.
  // Then copy old data to new space.
  void *new_ptr;
  new_ptr = malloc(size);
  if (!new_ptr) {
    return NULL; // TODO: set errno on failure.
  }
  memcpy(new_ptr, ptr, block_ptr->size);
  free(ptr);
  return new_ptr;
}

void *calloc(size_t nelem, size_t elsize) {
  size_t size = nelem * elsize; // TODO: check for overflow.
  void *ptr = malloc(size);
  memset(ptr, 0, size);
  return ptr;
}
```

---

## Line-by-line explanation and concept walkthrough

I'll go through the file top-to-bottom, explaining every line and the concepts used.

---

#### Header comment

```c
/* malloc.c - simple malloc / free / realloc / calloc implementation
   from "Malloc tutorial" by Dan Luu (https://danluu.com/malloc-tutorial/) */
```

* Plain comment. Indicates purpose and source. Not executed.

---

#### Standard includes

```c
#include <assert.h>
#include <string.h>
#include <sys/types.h>
#include <unistd.h>
```

* `assert.h`: provides `assert(expr)` which aborts the program if `expr` is false; useful for internal sanity checks.
* `string.h`: includes `memcpy`, `memset`, and other memory/string helpers used later.
* `sys/types.h`: often included for type definitions (e.g., `size_t`), though `stddef.h` also has `size_t`.
* `unistd.h`: declares POSIX functions such as `sbrk()` which we use to change the process heap size.

**Concept: `sbrk()`** — `sbrk(0)` returns the current program break (top of heap). `sbrk(n)` increments the break by `n` bytes and returns the previous break on success, or `(void*)-1` on failure. This code uses `sbrk` to ask the OS for more heap memory. Modern systems may discourage `sbrk` (mmap is common), but for tutorial simplicity `sbrk` is fine.

---

#### Block metadata struct

```c
struct block_meta {
  size_t size;
  struct block_meta *next;
  int free;
  int magic; // For debugging only.
};
```

* This `struct` stores metadata for each allocated block on the heap.

  * `size`: number of usable bytes in this block (the size requested by the user).
  * `next`: pointer to next block's metadata — used to build a singly linked list of all blocks.
  * `free`: boolean-ish (0 or 1) indicating whether this block is currently free (available for reuse).
  * `magic`: an integer value used to help detect memory corruption / double-free / invalid pointers during debugging. Different magic values are written at different lifecycle points.

**Concepts:**

* We store metadata *adjacent* to the user-visible memory (usually immediately before it). This lets `free` and other routines find the metadata from a user pointer.
* The linked list gives us a simple way to track all blocks and search for free ones (`O(n)` search).

---

```c
#define META_SIZE sizeof(struct block_meta)
```

* Macro for convenience: the size in bytes of the metadata structure. It's used when allocating space with `sbrk` and when doing pointer arithmetic (e.g., returning `block + 1` to get the user pointer).

---

```c
void *global_base = NULL;
```

* `global_base` is the head (start) of the linked list of `block_meta` blocks. Initially `NULL`. This is the global allocator state.

**Concept: Global allocator state** — real allocators have complex state; here we use a single global pointer and a linked list for simplicity. This is *not thread-safe*.

---

#### find_free_block

```c
struct block_meta *find_free_block(struct block_meta **last, size_t size) {
  struct block_meta *current = global_base;
  while (current && !(current->free && current->size >= size)) {
    *last = current;
    current = current->next;
  }
  return current;
}
```

* Purpose: walk the linked list looking for a free block with `size >= requested size`.
* Args:

  * `last`: output parameter; if the function fails to find a block, `*last` will point to the last node visited (used to append a new block).
  * `size`: requested user size.
* Implementation:

  * Start at `global_base`.
  * Loop until `current` is `NULL` or until we find a `current` where `current->free` is true and `current->size >= size`.
  * In each iteration, set `*last = current` and `current = current->next`.
  * Return the found block pointer (or `NULL` if none found).

**Concepts:**

* Linear search: simple but `O(n)`. Real allocators use segregated free lists, trees, or bins for performance.
* The condition `current && !(current->free && current->size >= size)` is expressive: it continues until a suitable block is found.

---

#### request_space

```c
struct block_meta *request_space(struct block_meta* last, size_t size) {
  struct block_meta *block;
  block = sbrk(0);
  void *request = sbrk(size + META_SIZE);
  assert((void*)block == request); // Not thread safe.
  if (request == (void*) -1) {
    return NULL; // sbrk failed.
  }
  if (last) { // NULL on first request.
    last->next = block;
  }
  block->size = size;
  block->next = NULL;
  block->free = 0;
  block->magic = 0x12345678;
  return block;
}
```

* Purpose: ask the OS for more heap via `sbrk` and initialize a new `block_meta` there.
* Steps:

  1. `block = sbrk(0);` — get the current top-of-heap pointer; this is where the new block metadata will begin.
  2. `sbrk(size + META_SIZE)` — expand the heap by metadata + user area. `sbrk` returns the previous break (should equal `block`). We store it in `request`.
  3. `assert((void*)block == request);` — sanity check that `sbrk(0)` and the returned pointer match; if not, something odd happened (or another thread changed break).
  4. If `request == (void*)-1`, `sbrk` failed — return `NULL`.
  5. If there's a `last` (we're not the first block), link `last->next` to the new `block`.
  6. Initialize fields: `size`, `next`, `free = 0` (in use), and set a debug magic number.
  7. Return pointer to the metadata (`block`), not the user area.

**Concepts:**

* `sbrk` expands the process heap contiguously. We allocate `META_SIZE` + `size` so metadata is stored "immediately before" the user memory.
* This implementation **never returns memory to the OS** (it only expands). Real allocators may `munmap` or shrink the break when possible.
* `assert` is used to detect race conditions — this implementation is **not thread-safe**: if another thread calls `malloc`/`sbrk` concurrently break values may change.

---

#### malloc

```c
void *malloc(size_t size) {
  struct block_meta *block;
  // TODO: align size?
  if (size <= 0) {
    return NULL;
  }
  if (!global_base) { // First call.
    block = request_space(NULL, size);
    if (!block) {
      return NULL;
    }
    global_base = block;
  } else {
    struct block_meta *last = global_base;
    block = find_free_block(&last, size);
    if (!block) { // Failed to find free block.
      block = request_space(last, size);
      if (!block) {
        return NULL;
      }
    } else {      // Found free block
      // TODO: consider splitting block here.
      block->free = 0;
      block->magic = 0x77777777;
    }
  }
  return(block+1);
}
```

Now the main allocator function — step-by-step:

1. `struct block_meta *block;` — temporary pointer for the chosen block.
2. `// TODO: align size?` — alignment note: real allocators round requested sizes up to alignment (e.g., 8 or 16 bytes) to ensure returned pointers meet platform ABI alignment. This tutorial leaves it as an exercise.
3. `if (size <= 0) return NULL;` — defensive: zero or negative (negative wrapped to huge) sizes are not allocated here; the C standard allows `malloc(0)` to either return `NULL` or a unique pointer that can be freed later — this implementation chooses `NULL`.
4. If `!global_base` (first call):

   * `block = request_space(NULL, size);` — get new block from OS.
   * If `request_space` returned `NULL` → `malloc` returns `NULL` (allocation failure).
   * `global_base = block;` — set head pointer to this first block.
5. Else (not the first call):

   * `struct block_meta *last = global_base;` — initialize `last` for `find_free_block`.
   * `block = find_free_block(&last, size);` — search for reusable free block.
   * If none found (`!block`):

     * `block = request_space(last, size);` — append new block and initialize.
     * If `NULL` → return `NULL`.
   * Else (found free block):

     * `block->free = 0;` — mark it allocated.
     * `block->magic = 0x77777777;` — new magic number for allocated state.
     * Comment `// TODO: consider splitting block here.` — if the block is much larger than requested, a better allocator may split it and return only needed bytes, leaving the remainder as a smaller free block.
6. `return(block+1);`

   * Pointer arithmetic: `block` is `struct block_meta *`. `block + 1` points to memory immediately after the metadata (i.e., user area). This is the pointer returned to user code.

**Important concepts:**

* The returned pointer is to the usable memory, not the metadata. The metadata sits at a negative offset from the returned pointer.
* No alignment is enforced here; on some architectures that will still work often, but incorrect alignment can cause crashes or performance issues.
* No splitting or coalescing — this leads to fragmentation: large free blocks may be left unused if they are slightly too small or not split.
* Not thread-safe: multiple threads can corrupt `global_base` or call `sbrk` concurrently — real `malloc` implementations include locks or thread-local arenas.

---

#### get_block_ptr

```c
struct block_meta *get_block_ptr(void *ptr) {
  return (struct block_meta*)ptr - 1;
}
```

* Given a user pointer `ptr` returned by `malloc`, get the pointer to the metadata struct.
* `(struct block_meta*)ptr` casts the user pointer to a `block_meta *`, then `-1` moves it back by one `block_meta` unit — i.e., to the metadata header.

**Pointer arithmetic concept:** Subtracting 1 from a typed pointer moves the address back by `sizeof(type)` bytes.

---

#### free

```c
void free(void *ptr) {
  if (!ptr) {
    return;
  }
  // TODO: consider merging blocks once splitting blocks is implemented.
  struct block_meta* block_ptr = get_block_ptr(ptr);
  assert(block_ptr->free == 0);
  assert(block_ptr->magic == 0x77777777 || block_ptr->magic == 0x12345678);
  block_ptr->free = 1;
  block_ptr->magic = 0x55555555;
}
```

* `free` marks the block as available for reuse.
* Steps:

  1. If `ptr` is `NULL`, do nothing (standard says free(NULL) is a no-op).
  2. `block_ptr = get_block_ptr(ptr);` — locate metadata.
  3. `assert(block_ptr->free == 0);` — ensure it wasn't already freed (double-free detection in debug builds).
  4. `assert(block_ptr->magic == 0x77777777 || block_ptr->magic == 0x12345678);` — sanity check that the magic matches an allocated or freshly created block; helps catch invalid pointers.
  5. `block_ptr->free = 1;` — mark free.
  6. `block_ptr->magic = 0x55555555;` — mark magic value for freed block (detect reuse/double free).

**Concepts:**

* This `free` does **not** return memory to OS; it only toggles `free` flag so future `malloc` calls can reuse the chunk.
* No coalescing: adjacent free blocks are left separate, causing fragmentation. A better `free` would merge (coalesce) neighboring free blocks into a larger free block.
* The `assert`s help detect bugs in test/demo scenarios. In production `assert` is often disabled (by `NDEBUG`) — real allocators must be robust without `assert`.

---

#### realloc

```c
void *realloc(void *ptr, size_t size) {
  if (!ptr) {
    // NULL ptr. realloc should act like malloc.
    return malloc(size);
  }

  struct block_meta* block_ptr = get_block_ptr(ptr);
  if (block_ptr->size >= size) {
    // We have enough space. Could free some once we implement split.
    return ptr;
  }
  // Need to really realloc. Malloc new space and free old space.
  // Then copy old data to new space.
  void *new_ptr;
  new_ptr = malloc(size);
  if (!new_ptr) {
    return NULL; // TODO: set errno on failure.
  }
  memcpy(new_ptr, ptr, block_ptr->size);
  free(ptr);
  return new_ptr;
}
```

* `realloc` changes the size of an existing allocation.
* Behavior:

  * If `ptr == NULL` — behave like `malloc(size)` (standard requirement).
  * Get the metadata with `get_block_ptr`.
  * If the current block is already big enough (`block_ptr->size >= size`), simply return the same pointer (no shrinking logic is implemented — could split).
  * If not enough space:

    * `malloc` a new block of requested `size`.
    * If `malloc` returns `NULL`, propagate `NULL` (no `errno` set here).
    * `memcpy` the old contents up to `old_size` into `new_ptr`. Note: only `old_size` bytes are copied, not the new `size`.
    * `free` the old block.
    * Return the new pointer.

**Concepts & caveats:**

* This `realloc` does not try to expand the block in-place (e.g., by checking next block if free). Real allocators often can expand in-place if adjacent memory is free.
* No attempt to shrink or split blocks; so `realloc` can waste space.
* The copy length is `block_ptr->size` (old size) — correct: you copy existing data, never more than old size.
* Not setting `errno` on failure is a small omission.

---

#### calloc

```c
void *calloc(size_t nelem, size_t elsize) {
  size_t size = nelem * elsize; // TODO: check for overflow.
  void *ptr = malloc(size);
  memset(ptr, 0, size);
  return ptr;
}
```

* `calloc` allocates `nelem * elsize` bytes and zeroes them.
* Steps:

  * Compute `size = nelem * elsize`. **Potential overflow:** product can overflow `size_t` and produce a smaller value — must be checked in robust implementations.
  * Call `malloc(size)`.
  * `memset(ptr, 0, size)` to zero memory (standard behavior).
  * Return pointer.

**Concepts:**

* `calloc` differs from `malloc`+`memset` in semantics but can be implemented that way if overflow is properly handled.
* Real `calloc` must check for integer overflow and fail if `nelem * elsize` would not fit in `size_t`.

---

## High-level limitations & concepts (why this is a *tutorial* allocator)

I’ll summarize the important limitations and related concepts so you understand what this code *doesn’t* do and why real allocators are complex:

* **Thread safety:** This implementation has global shared state (`global_base`) and uses `sbrk`. Concurrent `malloc`/`free` calls would race. Real allocators use locks, thread-local caches/arenas, or lock-free structures.
* **Alignment:** The code does not explicitly align returned pointers. Proper alignment (8/16 bytes depending on ABI) must be enforced to avoid undefined behavior for some types.
* **Fragmentation:** It does not split large blocks when returning a small piece, nor does it coalesce adjacent free blocks. Both choices lead to internal and external fragmentation.
* **Performance:** It uses a single linked list and linear search (`O(n)` find). Real allocators use bins, trees, or size-segregated lists for near-constant-time allocation.
* **No returning memory to OS:** Memory allocated via `sbrk` is never `brk`-shrunk or `munmap`ped back, so once program asks memory it's not returned — real allocators sometimes release large unused regions back to OS.
* **No security/hardening:** No guard pages, no canaries, limited checks; real allocators include protections (ASLR, guard pages, hardened free checks).
* **No overflow checks:** `calloc` multiplication not checked; `malloc` doesn't handle extremely large inputs robustly.
* **`sbrk` usage:** `sbrk` is an older interface; modern allocations often use `mmap` and maintain separate arenas.
* **Debugging aids:** `magic` integer + `assert` help detect misuse during development.

(These limitations are discussed in the tutorial; the code is intentionally simple to teach core ideas.) ([danluu.com][1])

---

## Quick notes on testing & using this code

* Compile as a shared library on Linux and `LD_PRELOAD` it into programs to test without recompiling them (Dan Luu shows how). Example:
  `clang -O0 -g -W -Wall -Wextra -shared -fPIC malloc.c -o malloc.so`
  `export LD_PRELOAD=/abs/path/malloc.so` then run target binary.
* Use `gdb`/`lldb` with debug symbols to step through allocations, inspect `global_base`, block list, etc.
* Add alignment and locking to approach real-world robustness.

---

## **To do next:**

* Add 8-byte alignment
* Demonstrate **splitting** and **coalescing** free blocks (step-by-step code)
* Convert this to be **thread-safe** by adding a mutex / spinlock.