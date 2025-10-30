# Code Style

## Naming convention

The project strictly follows naming convention of the [Zig language](https://ziglang.org/documentation/master/#Names). You should study this if you are going to contribute to the project and are not yet familiar with it.

Additionally, the project has more specific naming rules. These rules govern the usage of terms and expressions, not the syntax.

There are many commonly used general terms for naming functions. The most frequent ones in our project are: `init`, `deinit`, `alloc`, `free`, `new`, `delete`, `make`. Functions with these keywords are found for almost every declared structure, in almost every file. Therefore, to avoid ambiguity in interpreting the meaning of these words, clear definitions of their meaning within the project context are provided.

### `alloc`
The primary meaning is to allocate a resource for temporary use, usually implying the possibility of releasing such a resource via a call to `free`. The return value is preferably `null`/error or a resource, like `?MyResource`/`!MyResource`.

If memory allocation is implied: it exclusively expects the allocation of an uninitialized memory region of a certain size with correct alignment. 

### `free`
Releases a resource previously allocated via `alloc`.

If used for freeing memory, the function must not perform deinitialization of the memory area, but only release that area.

### `init`
1. For kernel subsystems: it means to bring into an operational state, perform initial setup, make the subsystem valid within the system boot process.
2. For various structures, such as `File`, `Inode`, `Process`, `Device`, etc.: it initializes and returns a **local** instance of a structure. This function can take any additional arguments and return an error.

Example:
```zig
pub fn init(...) !MyStruct { ... }
```

### `deinit`
1. For kernel subsystems: it brings the subsystem into a correct state before a full or partial system shutdown.
2. For structures: it deinitializes the structure, releases all resources used by it. After this, the state of the structure is **undefined** and its further use is prohibited unless explicitly stated otherwise.

### `new`
General meaning: `alloc` + `init`.

### `delete`
The reverse analog of `new`, meaning `deinit` + `free`.

### `setup`
Initializes a structure **by pointer**. This is an analog of `init`, used when it's necessary to have a pointer to the structure instance beforehand, during the initialization. The function should typically return an error, a status, or simply `void`.

Example:

```zig
pub fn setup(self: *Self) !void { ... }
```

### `make`
Typically means to assemble or create an instance of a structure, without strict requirements for arguments and return value. This is a more flexible form of `init` and `new` that is rarely used within the definitions of the structures themselves, but is utilized by other namespaces.

For example: `Drive.makeRequest`, but not `IoRequest.make`.

### `create`
An analog of `new`, used for higher-level functions where it's necessary not only to initialize the structure's fields but also to perform more complex actions during object creation. For example: `createDirectory`, `createFile`, or `Process.create`.