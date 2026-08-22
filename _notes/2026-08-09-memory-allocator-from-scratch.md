---
title: "What does a memory allocator do?"
date: 2026-08-09
tags: [systems, memory-management, c]
---

I wanted to understand how `malloc` actually works. Not as a black box, but at the actual mechanical level — how does it keep track of which blocks are free, how does it avoid turning the heap into unusable fragments, when does it ask the OS for more memory?

So I built a simple implicit free list allocator with ust the core ideas: headers and footers to track blocks, coalescing to merge adjacent free space, splitting to not waste memory on oversized allocations.

## basic setup

The allocator needs to:
- Handle random sequences of malloc/free calls.
- Return aligned memory (16-byte boundaries).
- Avoid fragmenting the heap into scattered holes.
- Only ask the OS for more memory when actually needed.

## block structure

Every block (allocated or free) looks like:

```
[Header (8 bytes)] [Payload (user data)] [Footer (8 bytes)]
                   ^ malloc returns pointer here
```

Header and footer are identical — they store the block size and whether it's allocated (1) or free (0). Both packed into one 64-bit value:

```c
#define SET_HEADER(size, flag) ((size << 0x4) | flag)
```

Size goes in the upper 60 bits (shifted left 4), flag in the lower 4 bits.

To extract:
```c
uint64_t get_size(block_t* block) {
    return (block->header >> 0x4);
}
uint64_t get_alloc_status(block_t* block) {
    return (block->header & 0xF);
}
```

**The footer matters because:** when you free a block, you need to check if the previous block is also free (to merge them). You can't walk backward without knowing the previous block's size. The footer sits right before your block's header and tells you that size. That's the whole point of duplicating the header into a footer.

All blocks are 16-byte aligned:

```c
#define ALIGN_UP(size, align) ((size + (align-1)) & ~(align-1))
```

This rounds size up to the nearest multiple of 16. Minimum block size is 16 bytes (header + footer), anything smaller doesn't make sense as a standalone block.

## heap layout

```
[Prologue (16 bytes)] [Free/Allocated Blocks...] [Epilogue (0 size)]
```

The prologue is a dummy block at the start. The epilogue marks the end. This simplifies things — you never have to check "does a previous block exist?" because the prologue is always there.

## finding and allocating

When malloc is called, the allocator scans the heap from the start — **first-fit**: find the first free block big enough for the request.

```c
block_t* find_alloc(uint64_t size) {
    block_t* head = alloc_head;
    
    while(get_size(head) > 0) {
        int alloc_size = ALIGN_UP(size + HEADER_SIZE + FOOTER_SIZE, 16);
        
        if (!get_alloc_status(head) && get_size(head) >= alloc_size) {
            allocate(head, alloc_size);
            return head;
        }
        head = next_block(head);
    }
    
    // No free block found, extend the heap
    int alloc_size = ALIGN_UP(size + HEADER_SIZE + FOOTER_SIZE + 8, 16);
    block_t* new_alloc = extend_heap(alloc_size);
    if (!new_alloc) return NULL;
    
    allocate(new_alloc, alloc_size);
    return new_alloc;
}
```

Why first-fit? It's simple and fast — O(n) scan, nothing to track. Downside: if the heap gets fragmented with scattered small free blocks, you might scan through a lot of them before finding one that fits.

## splitting: not wasting space

When a free block is larger than needed, split it into two: one allocated, one free.

```c
void split(block_t* head, uint64_t old_size, uint64_t new_size) {
    uint64_t remaining_size = old_size - new_size;
    
    if (remaining_size < 16) {
        // Don't split if remainder is too small
        head->header = SET_HEADER(old_size, 1);
        block_t* footer = (block_t*)((char*)head + old_size - FOOTER_SIZE);
        footer->header = SET_HEADER(old_size, 1);
        return;
    }
    
    // Allocate what was requested
    head->header = SET_HEADER(new_size, 1);
    block_t* footer = (block_t*)((char*)head + new_size - FOOTER_SIZE);
    footer->header = SET_HEADER(new_size, 1);
    
    // Create new free block with the rest
    block_t* new_block = (block_t*)((char*)head + new_size);
    new_block->header = SET_HEADER(remaining_size, 0);
    footer = (block_t*)((char*)new_block + remaining_size - FOOTER_SIZE);
    footer->header = SET_HEADER(remaining_size, 0);
}
```

The 16-byte minimum is important. If the remainder is less than 16 bytes, it's useless (header + footer alone are 16 bytes). Better to waste space than create a fragmentation trap.

## coalescing: merging free blocks

When you free a block, check if neighboring blocks are also free and merge them. This is what keeps the heap from turning into swiss cheese.

Three cases:

**Both neighbors free:**
```c
case p_n: {
    block_t* prev = prev_block(block);
    block_t* next = next_block(block);
    uint64_t new_size = get_size(prev) + get_size(next) + curr_size;
    prev->header = SET_HEADER(new_size, 0);
    next_footer(next)->header = SET_HEADER(new_size, 0);
    break;
}
```

**Only next neighbor free:**
```c
case o_n: {
    block_t* next = next_block(block);
    uint64_t new_size = get_size(next) + curr_size;
    block->header = SET_HEADER(new_size, 0);
    next_footer(next)->header = SET_HEADER(new_size, 0);
    break;
}
```

**Only previous neighbor free:**
```c
case o_p: {
    block_t* prev = prev_block(block);
    uint64_t new_size = get_size(prev) + curr_size;
    prev->header = SET_HEADER(new_size, 0);
    next_footer(block)->header = SET_HEADER(new_size, 0);
    break;
}
```

Use the footer of the block to your left to find its size, walk backward to its header, merge, and update the rightmost footer in the merged range. That's why the footer is essential — without it you'd need explicit backward pointers everywhere, which costs space and complexity.

## extending the heap

When no free block is large enough, ask the OS for more via `sbrk()`:

```c
block_t* extend_heap(uint64_t size) {
    block_t* new_alloc = el;  // Current epilogue becomes the new block
    
    if (sbrk(size) == (void*)-1)
        return NULL;  // OS said no
    
    new_alloc->header = SET_HEADER(size, 0);
    block_t* footer = (block_t*)((char*)new_alloc + size - FOOTER_SIZE);
    footer->header = SET_HEADER(size, 0);
    
    // Move epilogue to the new end
    el = (block_t*)((char*)footer + 8);
    el->header = SET_HEADER(0, 1);
    
    return new_alloc;
}
```

Each extend, the old epilogue becomes a new free block. Then place a fresh epilogue at the end.

## what i learned

**Pointer arithmetic depends on type.** `ptr + 1` on a `char*` advances 1 byte. On an `int*`, 4 bytes. On `void*` it's undefined (which is why you cast to `char*` to walk the heap). Matters a lot.

**The alignment bitmask is clever.** `~(align-1)` masks the lower bits. `(size + (align-1))` ensures you round up not down. No branching needed.

**`sbrk(0)` doesn't allocate.** It just returns the current program break. The pointer it returns is at the edge of valid memory — dereferencing it segfaults. Use it only to check where the break is.

**gdb beats printf debugging.** Live debugging memory issues is so much better than trying to print your way through it. You can watch the heap in real time, see coalescing happen, step through splits. Printf debugging of malloc is rough because libc affects the heap through printf itself.

**Using printf for debug changes the heap.** I thought my sbrk behavior was broken — turns out the program break was moving because printf was allocating buffers internally. Lesson: be careful what debug techniques you use when debugging memory allocation.

**The minimum block size prevents thrashing.** If you allow blocks smaller than header+footer, you fragment the heap with useless tiny holes that can never be allocated. The 16-byte minimum trades some space waste for stability.

## problems with this implementation

**First-fit fragments heavily.** Scatter allocations and frees across the heap and you end up with free space split across many small blocks. Best-fit (pick the smallest block that fits) or explicit free lists (linked list of only free blocks) would help, but cost more in speed or space.

**No heap shrinking.** The program break only moves forward. Free everything and the heap memory stays allocated — the OS doesn't get it back. A real allocator would try to coalesce backward and shrink the heap.

**Minimum block size wastes space.** A 1-byte allocation becomes 16 bytes due to alignment and minimum requirements. Workloads with many tiny allocations waste a lot this way.

**First-fit is simplistic.** It finds the first block that fits, doesn't optimize for locality or fragmentation. Best-fit reduces fragmentation. Next-fit reduces repeated scanning. Segregated size classes speed up common allocations.

For now this allocator works and helped me understand the basic principles behind memory allocation.

**Code:** [antoash/dynamic-memory-allocation](https://github.com/antoash/dynamic-memory-allocation)

---

## references that helped me along the way

- [CS 208: Allocator Design and the Implicit Free List](https://www.youtube.com/watch?v=xiPew2TmsDE) — the Excel demo in this really helped visualize how blocks move during allocation/freeing
- [Dynamic Memory Allocation: Basic Concepts (CMU CS 213)](https://www.cs.cmu.edu/~213/lectures/13-malloc-basic.pdf) — foundational stuff on fragmentation, coalescing, splitting
- [Generating Aligned Memory - Embedded Artistry](https://embeddedartistry.com/blog/2017/02/22/generating-aligned-memory/) — alignment and why it works
- [GDB Tutorial](https://www.youtube.com/watch?v=svG6OPyKsrw) — how to debug with GDB
- [sbrk/brk System Calls and Optimistic Allocation Explained](https://www.youtube.com/watch?v=vEXRpiI4Dhk) — understanding when the heap actually extends
- [How processes get more memory (mmap, brk)](https://www.youtube.com/watch?v=XV5sRaSVtXQ)