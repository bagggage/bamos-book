# Virtual Memory Management Subsystem

**Subsystem: `vm`**

The subsystem implements all necessary functionality for memory management within the system. It also provides implementations of various allocators and some APIs for working with page tables.

The system has some specific characteristics.

## Mappings

- Upper Half

  This portion of virtual memory is reserved for kernel needs. The kernel code and data are all located in the upper half.

- Lower Half

  This is used for user space. The kernel does not store any data in this section.

## Linear Memory Access Region (LMA)

This is an important feature of the kernel architecture.

The Linear Memory Access region is a large area of virtual memory in the upper half with a predefined address and size (specific to each platform), which is linearly mapped to physical pages starting from address 0x0. The region size should be large enough to cover the potential range of physical addresses. For the **x86-64** architecture, it is `256 GiB`.

The main idea is the ability to access any physical address without the need for additional page mapping.

Mapping virtual to physical addresses is a costly operation, which may also require additional page table allocations.

The LMA region solves this problem, allowing significant optimization of memory handling within the kernel. The region is mapped only once during kernel initialization.

To convert any physical address to virtual and vice versa, you can use:

```zig
vm.getVirtLma(address: anytype) @TypeOf(address)
```
- `phys`: the physical address/pointer.

```zig
vm.getPhysLma(address: anytype) @TypeOf(address)
```
- `virt`: the virtual address/pointer.

  Only addresses obtained from `vm.getVirtLma` are allowed.

## Memory Allocation

All memory allocation starts with physical page allocation, managed by `vm.PageAllocator`.

### Page Allocator `vm.PageAllocator`

> [!NOTE]
> The page allocator is **thread-safe**.

This allocator is fast and efficient, based on a buddy algorithm.

The main goal during its design was to achieve the highest performance and minimize memory overhead. Avoiding additional memory allocation for the allocator itself is impossible, but this allocation is a one-time event during system initialization and is minimized in size.

Due to the chosen algorithm, the implementation allows allocation of page quantities that are powers of two, designated as `rank`.

The only drawback is the need to remember the size of the allocated region for subsequent freeing.

#### API

> [!NOTE]
> More detailed information is available in the [code documentation](https://bagggage.github.io/bamos/#bamos.vm.PageAllocator).

```zig
vm.PageAllocator.alloc(rank: u32) ?usize
```
Allocates a linear block of physical pages of the specified rank (size).
Returns the physical address of the allocated pages, or `null` if allocation fails.

- `rank`: Determines the number of pages as `2^rank`.

```zig
vm.PageAllocator.free(base: usize, rank: u32) void
```
Frees a physical memory of the specified rank (size).

- `base`: Physical address of the first page in a linear block returned by `vm.PageAllocator.alloc`.
- `rank`: Determines the number of pages as `2^rank`, must be the same as in the `vm.PageAllocator.alloc` call.

### Object Allocator `vm.ObjectAllocator`

> [!NOTE]
> The object allocator **is not thread-safe**. However, there is a simple wrapper, `vm.SafeOma`, which includes `utils.Spinlock` and is **thread-safe**.

> [!TIP]
> The allocator uses the LMA region for fast physical page mapping. Therefore, `vm.getPhysLma` can safely be used for allocated objects.

The object allocator was introduced for very fast allocation of memory sections of the same size. It’s useful in cases of frequent allocation of identical structures, such as descriptors for processes, files, devices, and more.

This allocator is based on the allocation of arenas (i.e., large blocks of one or more pages), which are then divided into multiple objects. When freed, objects are added to a singly linked list, allowing immediate reallocation: complexity `O(1)`.

Fragmentation is either non-existent or amounts to only a few bytes per arena, depending on the object size.

#### API

> [!NOTE]
> More detailed information is available in the [code documentation](https://bagggage.github.io/bamos/#bamos.vm.ObjectAllocator).

```zig
vm.ObjectAllocator.init(T: type) vm.ObjectAllocator
```
Initializes an allocator for a specific object type.
Can be used at `comptime`.

- `T`: The type of objects to allocate.

```zig
vm.ObjectAllocator.deinit(self) void
```
Deinitializes the allocator, freeing all allocated memory.

```zig
vm.ObjectAllocator.alloc(self, T: type) ?*T
```
Allocates memory for an object and casts it to a pointer of type `T`.
Returns a pointer to the allocated object, or `null` if allocation fails.

- `T`: type of the pointer.

```zig
vm.ObjectAllocator.free(self, obj_ptr: anytype) void
```
Frees the memory of an object. Invalid object pointers cause undefined behavior.

- `obj_ptr`: pointer to the object to free.

### Universal Allocator `vm.UniversalAllocator`

> [!NOTE]
> The universal allocator is **thread-safe**.

> [!TIP]
> The allocator uses the LMA region for fast physical page mapping. Therefore, `vm.getPhysLma` can safely be used for allocated sections.

The universal allocator allows allocation of memory sections of various sizes. It’s not as efficient as the page allocator or object allocator. However, there is often a need to allocate memory sections of unknown size, such as strings or other arrays. In these cases, this allocator is very useful.

It is based on multiple object allocators with predefined different sizes. When allocating memory, the size is rounded up to the nearest fit, which may cause some local fragmentation.

For large memory sections, it directly uses `vm.PageAllocator`, allocating the required number of pages. Sizes are also rounded, so it is best to allocate blocks in powers of two, matching page sizes, to avoid fragmentation.

#### API

> [!NOTE]
> More detailed information about the allocator’s implementation and limitations is available in the [code documentation](https://bagggage.github.io/bamos/#bamos.vm.UniversalAllocator).

The `vm` subsystem provides convenient shortcuts for direct calls:

```zig
vm.malloc(size: usize) ?*anyopaque
```
```zig
vm.free(ptr: ?*anyopaque)
```

It also has a shortcut for allocating a specific object:

```zig
vm.alloc(T: type) ?*T
```

> [!TIP]
> When working with Zig’s standard library (`std`), you can also use the ready-made `std.mem.Allocator` implementation based on `vm.UniversalAllocator`:
> 
> - `vm.std_allocator`

## Working with Page Tables

The subsystem aims to remove platform dependency. It provides a set of functions for interacting with page tables and mapping virtual to physical addresses.

`vm.PageTable` represents the platform-dependent page table structure defined in `arch.vm.PageTable`.

```zig
vm.allocPt() ?*vm.PageTable
```
Allocates a new page table and zeroes all entries.
Returns `null` if allocation fails.

```zig
vm.newPt() ?*vm.PageTable
```
Allocates a new page table and maps all necessary kernel units.
Returns `null` if allocation fails.

```zig
vm.freePt(pt: *vm.PageTable) void
```
Frees a page table.

```zig
vm.getPt() *vm.PageTable
```
Gets the current page table from a specific CPU register.

```zig
vm.setPt(pt: *const vm.PageTable) void
```
Sets the given page table to a specific CPU register.

```zig
vm.mmap(
  virt: usize,
  phys: usize,
  pages: u32,
  flags: vm.MapFlags,
  page_table: *PageTable
) vm.Error!void
```

Maps a virtual memory range to a physical memory range.
Returns `vm.Error` if mapping fails.

- `virt`: base virtual address to which physicall region must be mapped.
- `phys`: region base physical address.
- `pages`: number of pages to map.
- `flags`: flags to specify (see `vm.MapFlags` structure).
- `page_table`: target page table.

```zig
vm.getPhys(address: anytype) ?@TypeOf(address)
```
Translates the virtual address into a physical address via the current page table.
Returns the physical address or `null` if the address isn’t mapped.

```zig
vm.getPhysPt(address: anytype, pt: *const vm.PageTable) ?@TypeOf(address)
```
Translates the virtual address into a physical address via a specific page table.
Returns the physical address or `null` if the address isn’t mapped.

---

More detailed information is provided in the [code documentation](https://bagggage.github.io/bamos/#bamos.vm).