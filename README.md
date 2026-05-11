## Implementation Details

### Block Layout

Each heap block is stored inside a contiguous heap region and contains metadata used by the allocator to manage allocation and deallocation.

An allocated block is organized as:

[ header | payload ]

A free block is organized as:

[ header | payload / free-list pointers | footer ]

The header stores the block size and allocation status. Free blocks also use a footer so that the allocator can find the previous physical block during coalescing.

The block size includes metadata and payload space. Since block sizes are aligned, the lowest bit of the size field can be used as the allocation bit.

### Header and Footer Encoding

The allocator stores size and allocation status in a single metadata word.

- The block size is extracted by masking out the allocation bit.
- The allocation status is stored in the least significant bit.

This keeps metadata compact while allowing the allocator to quickly determine whether a block is free or allocated.

### Alignment

All allocation requests are rounded up to satisfy alignment requirements. The allocator also adds metadata overhead before searching for a suitable free block.

For example, a small user request may be expanded to include the header and any padding needed for alignment. This ensures that the returned payload pointer is properly aligned.

### Implicit Free List

The implicit free-list allocator does not store a separate list of free blocks. Instead, it traverses the heap block by block.

During allocation, it:

1. Starts at the beginning of the heap.
2. Reads each block’s header.
3. Checks whether the block is free and large enough.
4. Moves to the next block using the current block’s size.

This design is simple, but allocation can become slow because the allocator may need to scan many allocated blocks before finding a usable free block.

### Explicit Free List

The explicit free-list allocator maintains a doubly linked list of only free blocks. The `prev` and `next` pointers are stored inside the payload area of each free block.

A free block in the explicit allocator has the following layout:

[ header | prev pointer | next pointer | remaining free space | footer ]

When a block is allocated, it is removed from the free list. When a block is freed, it is inserted back into the free list. This reduces traversal cost because allocation only searches free blocks instead of every block in the heap.

The trade-off is that the explicit design requires more careful pointer management, especially during splitting and coalescing.

### Block Splitting

When the allocator finds a free block larger than the requested size, it may split the block.

The first part is marked as allocated and returned to the user. The remaining part becomes a smaller free block.

The allocator only performs splitting if the remaining space is large enough to form a valid free block with its own metadata. This avoids creating unusable fragments.

### Coalescing

The allocator performs immediate coalescing when a block is freed. It checks the neighboring physical blocks and merges adjacent free blocks to reduce external fragmentation.

There are four cases:

1. Both neighbors are allocated: no coalescing is needed.
2. The next block is free: merge with the next block.
3. The previous block is free: merge with the previous block.
4. Both neighbors are free: merge all three blocks.

Boundary tags make this efficient because the footer of the previous block stores its size, allowing the allocator to locate the previous block without scanning from the beginning of the heap.

### Free List Updates

For the explicit allocator, the free list must be updated whenever a block changes state.

- Allocated blocks are removed from the free list.
- Freed blocks are inserted into the free list.
- Coalesced neighboring blocks are removed before the merged block is inserted.
- Split remainders are inserted back as free blocks.

Incorrect free-list updates can cause corrupted pointers, cycles in the free list, or invalid memory accesses, so this part required careful debugging.

### Debugging and Testing

The allocator was tested using trace-driven workloads that simulate repeated calls to allocation and free operations.

During development, GDB was used to inspect heap layout, verify header and footer values, and debug corrupted free-list pointers.

Common issues encountered included:

- incorrect block-size calculations after alignment
- forgetting to update footers after splitting
- failing to remove blocks from the explicit free list before coalescing
- returning a pointer to the block header instead of the payload
- creating free blocks that were too small to hold required metadata
