# Registers API

**Subsystem**: `dev.regs` (*hereafter `regs`*)

This API provides convenient abstractions for generalized work with device registers.

Interacting with registers is one of the primary tasks a driver developer faces. The API addresses common issues with choosing how to access and interpret register layouts in code.

Typically, developers define constants for register offsets relative to a base address, like:
```zig
const SOME_REG = 0x100;
const ANOTHER_REG = 0x1F0;
```
They then write specific `read` and `write` functions.

Since this is a common task, many aspects can be generalized and pre-implemented. To ensure uniform and clean code in drivers, this API is presented in the kernel.

## Register Group `regs.Group`

A register group is a structure that abstracts working with a specific group of registers.

With it, you can define the method or mechanism ([`dev.io.Mechanism`](./io.md#io-mechanism-iomechanism)) for accessing registers, as well as the names, locations, and sizes of the registers.

**`regs.Group` uses `comptime` calculations and does not add any runtime overhead.**

### API

```zig
fn regs.Group(
    comptime IoMechanism: type,
    comptime base: ?comptime_int,
    comptime size: ?comptime_int,
    comptime regs: []const Register
) type
```

Returns a register group type.

- `IoMechanism`: type of [mechanism](./io.md#io-mechanism-iomechanism) used for interacting with registers.

- `base`: `comptime` base address of the register group.

  This parameter is optional and can be `null`. It is used only if the base is always static and known at compile time, avoiding runtime address calculations.

- `size`: size of the register group.

  This is also optional and can be `null`. The size can be calculated automatically based on the location and size of all the registers in the group.

  It is used if the `IoMechanism` implements an `init` function to pass the `size` parameter.

- `regs`: a slice of registers contained in the group.

  Each register in the slice is of type [`regs.Register`](#register-regsregister), see below.

**Usage**

To use, you need to create an instance of the group structure.

- For a static group, where the base was specified in `regs.Group` and is known at compile time:

  ```zig
  const/var my_regs = try MyGroup.init();
  ```

  The `init` function returns a structure instance, and if the `IoMechanism` implements its own `init` function, it is also called to initialize the access mechanism.

- For a dynamic base group, where the base is known at runtime:

  ```zig
  const/var my_regs = try MyGroup.initBase(base);
  ```

    - `base`: the base address of the group.

  The `initBase` function returns a structure instance, and if the `IoMechanism` implements its own `init` function, it is also called to initialize the access mechanism. In this case, the base returned by the `IoMechanism.init` function is used.

For a dynamic base group, the structure stores only one field of type `IoMechanism.Address`. If the base address is known at compile time, the group structure stores no fields and takes up no memory.

When `io.Group` is called, an enumeration of register names is generated, which is then used when calling read/write functions.

**Interacting with registers**

- Reading:
  ```zig
  fn read(member) RegIntType
  ```

  Reads a register.

  - `member`: the register name from the enumeration, e.g., `.reg_name`.

  `RegIntType` is the register type defined in `regs.Register`.
    
  `IoMechanism` ensures that data is read at a properly aligned address of the size defined by `IoMechanism.DataType`. However, the register itself may be larger or smaller, so `RegIntType` represents the specific register type, and the reading process accounts for register offset and size.

  ```zig
  fn get(T: type, member) T
  ```

  Reads a register and converts the read value into type `T`.

  - `member`: the register name from the enumeration, e.g., `.reg_name`.
  - `T`: the return type.

    The read numeric value is cast into type `T` using `@bitCast`.

  This function is useful if you need to read a register and get its representation in a specific type, for example, to work with register bit fields.

- Writing:
  ```zig
  fn write(member, data: RegIntType) void
  ```

  Writes to a register.

  - `member`: the register name from the enumeration, e.g., `.reg_name`.
  - `data`: the value to write.

  Similar to reading, writing takes register offset and size into account, allowing writes to misaligned or non-multiple-of-size registers.

  ```zig
  fn set(member, data: anytype) void
  ```

  Writes to a register, casting the data to a numeric value.

    - `member`: the register name from the enumeration, e.g., `.reg_name`.

    - `data`: the value to write.

      The `data` value is cast into the target register type `RegIntType` using `@bitCast`.

    This function is useful when you need to write an entire user-defined structure to a register, for example, after initially obtaining the structure using `get`.

## Register `regs.Register`

A structure used to provide register information to `io.Group`. It allows you to specify some details.

```zig
fn regs.reg(
    comptime name: [:0]const u8,
    comptime offset: comptime_int,
    comptime Type: ?type, 
    comptime access: Access,
) Register
```

- `name`: the register name.
- `offset`: the offset relative to the base address of the group.
- `Type`: the register type, used as `RegIntType`.

  This is optional and can be `null`.

  > The `Type` must be an unsigned integer: `u<x>`, where `x` is the bit width.

- `access`: access mode for the register, can be: `.rw`, `.read`, `.write`.

  Used to enhance security, preventing unauthorized read or write operations for the register. A compilation error is generated when an invalid operation is called.

  - `rw`: read and write.
  - `read`: read-only.
  - `write`: write-only.

Additionally, register information can be automatically generated based on a user-defined structure as a layout. Use the `regs.from(...)` function for this.

```zig
fn regs.from(comptime Layout: type) []const Register
```

Generates registers information and returns a slice of `[]const regs.Register`.

- `Layout`: the name of the user-defined structure.

  The layout can be a `struct`, `packed struct`, `extern struct`, or `union`.

  Each field in this structure is interpreted as a separate register. The register name corresponds to the field name, the register offset is the field's offset within the structure, and the field's type is the register type.

  Fields whose names start with an underscore `_` are ignored.

  To specify access mode `access`, use special types for structure fields:

  - `regs.ReadOnly(u<x>)`: for read-only registers.
  - `regs.WriteOnly(u<x>)`: for write-only registers.

    > `u<x>`: the type, an unsigned integer where `x` is the bit width.

    > For `packed` or `extern` structs, use the postfix `P` or `E`: `regs.ReadOnlyP(u32)`, `regs.WriteOnlyE(u32)`, etc.

    By default, other registers are read-write.

If `Layout` is a `union`, then the `Layout` itself is interpreted as a group of `Layout`s. In this case, each `union` field allows you to define its own set of unique registers with their own names and offsets.
    
> [!NOTE]
> However, register names must not be duplicated.

Example:
```zig
const Layout = packed union {
    regs1: packed struct {
        address_reg: u64,
        control_reg: u16,
        id_reg: u16
    },
    regs2: packed struct {
        _rsrdv: u64, // Ignored

        data_reg: u32,
        status_reg: u32
    }
};
```

## Examples

At first glance, the abstraction may seem overly complicated. However, it is quite simple to use in practice.
To better understand, examples are provided below.

### Manually Defining Registers

```zig
const reg = dev.regs.reg;

const uart_base = 0x3f8;

const UartRegs = dev.regs.Group(
    dev.io.IoPortsMechanism(
        "uart 8250/16450/16550",
        .byte
    ),          // IoMechanism
    uart_base,  // Base
    null,       // Size

    // Registers slice
    &.{
        // DLAB == 0
        reg("data",         0x0, null, .rw),
        reg("intr_enable",  0x1, null, .rw),

        // DLAB == 1
        reg("div_lo",       0x0, null, .rw),
        reg("div_hi",       0x1, null, .rw),

        reg("intr_id",      0x2, null, .read),

        reg("fifo_ctrl",    0x2, null, .write),
        reg("line_ctrl",    0x3, null, .rw),
        reg("modem_ctrl",   0x4, null, .rw),

        reg("line_status

",  0x5, null, .read),
        reg("modem_status", 0x6, null, .read),

        reg("scratch",      0x7, null, .rw),
    }
);
```

### Automatic Registers Layout Based on a Structure

The code is similar to the manual register example.

```zig
const UartRegs = dev.regs.Group(
    dev.io.IoPortsMechanism(
        "uart 8250/16450/16550",
        .byte
    ),          // IoMechanism
    uart_base,  // Base
    null,       // Size

    // Registers slice
    dev.regs.from(UartRegsLayout)
);

// Layout `union`
const UartRegsLayout = packed union {
    dlab_0: packed struct {
        data: u8,
        intr_enable: u8,

        fifo_ctrl: dev.regs.WriteOnlyP(u8);

        line_ctrl: u8,
        modem_ctrl: u8,

        line_status: dev.regs.ReadOnlyP(u8),
        modem_status: dev.regs.ReadOnlyP(u8),

        scratch: u8
    },
    dlab_1: packed struct {
        div_lo: u8,
        div_hi: u8
    },

    intr: packed struct { // Just for `intr_id` register.
        _offset: u16,

        intr_id: dev.regs.ReadOnlyP(u8);
    },
};
```

### Usage

This example shows how a previously defined group of registers can be used.

```zig
const regs = UartRegs{};

pub fn init() !void {
    // Call `init` at runtime to trigger `dev.io.request`
    // which is used by `dev.io.IoPortsMechanism`.
    _ = try UartRegs.init();

    regs.write(.intr_enable, 0x00); // Disable all interrupts

    regs.write(.line_ctrl, 0x80);   // Enable DLAB (set baud rate divisor)
    regs.write(.div_lo,    0x03);   // Set divisor to 3 (low byte) 38400 baud
    regs.write(.div_hi,    0x00);   //                   (high byte)

    regs.write(.line_ctrl, 0x03);   // 8 bits, no parity, one stop bit
    regs.write(.fifo_ctrl, 0xC7);   // Enable FIFO, clear it, with 14-byte threshold

    regs.write(.modem_ctrl, 0x0B);  // IRQs enabled, RTS/DSR set
}

pub fn deinit() void {
    // Don't forget to release I/O space.
    dev.io.release(uart_base, .io_ports);
}
```