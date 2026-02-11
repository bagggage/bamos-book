# Device's I/O access

**Subsystem**: `dev.io` (*hereafter `io`*)

To simplify and standardize input/output operations for devices, a specialized subsystem was implemented.

It offers a platform-independent API for working with **MMIO space** and **I/O ports**, as well as the ability to implement your own **I/O mechanism**.

Additionally, for security purposes, the system provides functionality for managing and monitoring the address ranges of input/output spaces, binding them to specific devices. This helps resolve conflicts that arise when multiple drivers attempt to access the same I/O addresses.

> [!CAUTION]
> Of course, you can bypass the I/O space reservation functions for your device
> and directly use the read/write functions. However,
> this is strongly discouraged and such code is generally not allowed in the kernel.

## I/O Mechanism `io.Mechanism`

An I/O mechanism is an abstraction that allows you to conveniently use different access methods, abstracting all specific code.

**Mechanisms are entirely `comptime` structures and do not add runtime overhead.**

Mechanisms are useful for dynamically switching between different ways of working with a device. Often, devices may support both I/O ports and MMIO. Using mechanisms, you can write code that is independent of the I/O method and dynamically select the appropriate mechanism at runtime.

Another use case for mechanisms is with the [Registers API](./registers.md).

Creating a mechanism (*all parameters must be known at compile time*):

```zig
io.Mechanism(AddrType, DataType, readFn, writeFn, ?initFn) type
```
- `AddrType`: the address type used by the mechanism, e.g., `u32` or `usize`.
- `DataType`: the data type, used to define the size and alignment for read/write operations, e.g., `u8`, `u16`, `u32`, etc.
- `readFn`: the read function implementation.

  Should look like: 
  ```zig
  fn read(address: AddrType) DataType
  ```

- `writeFn`: the write function implementation.

  Should look like:
  ```zig
  fn write(address: AddrType, data: DataType) void
  ```

- `initFn`: an initialization function used to run during runtime when creating an instance of the structure. This is an optional parameter and can be set to `null`.

  Should look like:
  ```zig
  fn init(base: AddrType, size: AddrType) anyerror!AddrType
  ```

  The function initializes and must return the base address (potentially modified) if successful.

### Example Usage

```zig
const MyIoType = io.Mechanism(u16, u8, read, write, init);

fn init(base: u16, size: u16) !u16 {
    _ = io.request("My I/O", base, size, .io_ports) orelse return error.IoBusy;
    return base;
}

fn read(addr: u16) u8 {
    // Read byte from I/O port
    return io.inb(addr);
}

fn write(addr: u16, data: u8) void {
    // Write byte to port
    io.outb(addr, data);
}

pub fn use() void {
    try MyIoType.init(0xCF8, 0x4);

    MyIoType.write(0xCF8, 0x8000_0000);
    _ = MyIoType.read(0xCFC);
}
```

## I/O Management API

```zig
io.request(
    comptime name: [:0]const u8,
    base: usize,
    size: usize,
    comptime io_type: Type
) ?usize
```

Reserves a specific range of I/O space for device.

- `name`: the name of the region, used for informational purposes only.
- `base`: the base address of the I/O space.
- `size`: the size of the space relative to the base address.
- `io_type`: the type of space, which can be `io_ports` or `mmio`.

The function returns `null` if the request fails, for example, if the space is already occupied by another device.

```zig
io.release(base: usize, comptime io_type: Type) void
```

Releases the reserved range of I/O space.

- `base`: the base address specified during `io.request`.
- `io_type`: the I/O type specified during `io.request`.

## I/O Ports

**Mechanism**

```zig
io.IoPortsMechanism(
    comptime name: [:0]const u8,
    comptime bus_width: io.BusWidth
) type
```

Returns a mechanism that implements I/O operations through I/O ports.
Implements `init` for automatic `io.request` invocation.

- `name`: the name used during mechanism initialization in the `io.request` call.
- `bus_width`: defines the data size for read/write operations.

  Supported options are: `.byte`: u8, `.word`: u16, `.dword`: u32; `.qword` is not supported.

**API Functions**

Reading:
```zig
io.inb(address); // read one byte.
io.inw(address); // read one word.
io.inl(address); // read a double word.
```

Writing:
```zig
io.outb(address, data); // write one byte.
io.outw(address, data); // write one word.
io.outl(address, data); // write a double word.
```

## MMIO Space

**Mechanism**

```zig
io.MmioMechanism(
    comptime name: [:0]const u8,
    comptime bus_width: io.BusWidth
) type
```

Returns a mechanism that implements I/O operations through MMIO space.
Implements `init` for automatic `io.request` invocation and maps the provided MMIO base address to virtual address space. `init` returns a **virtual address**.

- `name`: the name used during mechanism initialization in the `io.request` call.
- `bus_width`: defines the data size for read/write operations.

  Supported options are: `.byte`: u8, `.word`: u16, `.dword`: u32, `.qword`: u64.

**API Functions**

Reading:
```zig
io.readb(address); // read one byte.
io.readw(address); // read one word.
io.readl(address); // read a double word.
io.readq(address); // read a quad word.
```

Writing:
```zig
io.writeb(address, data); // write one byte.
io.writew(address, data); // write one word.
io.writel(address, data); // write a double word.
io.writeq(address, data); // write a quad word.
```