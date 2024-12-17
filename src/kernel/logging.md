# Logging Subsystem

The logging system involves collecting and outputting various messages provided by all parts of the kernel. To improve convenience and standardization, the system’s primary feature is its use of the existing logging interface in Zig's standard library: `std.log`.

Currently, logging is limited to distributing logs via the serial port and displaying messages graphically on the screen. After the file system implementation, it is planned to redirect log messages to the appropriate file(s) in line with the concept of Unix-like systems.

## Implementation

The main component of the subsystem is the `logger`.

`logger` implements the `defaultLog` function in accordance with the `std.log.defaultLog` interface from the standard library. This replaces the default implementation, allowing the logger to be used directly via `std.log`.

### Thread Safety

The logger considers multithreaded systems and has built-in locks to ensure thread-safe logging. The subsystem also handles complex scenarios, such as logging during system failures, and avoids deadlocks using specialized functions:

```zig
fn capture() void
```

This is used for safely capturing or locking the output stream. It helps avoid situations where the stream was locked by the current thread and, due to an exception or interrupt, attempts to capture it again.

```zig
fn release() void
```

Safely unlocks the log output stream.

This functionality is not recommended for direct use without specific reasons. It is primarily intended for the `panic` module, which requires capturing the stream during calls to `@panic(...)` or when an unhandled hardware exception occurs, using `panic.exception(...)` to log useful information for debugging.

### Output Stream

Since kernel logging must always be available, including during various stages of system boot, not all required components may be accessible, such as a display for graphical output or a file system for saving logs. For this reason, the logging system uses the `std.io.AnyWriter` abstract interface from the standard library for log writing.

At different stages, the system uses different `writer` implementations:

1. **`EarlyWriter`**

   Used during the very early stages of system boot.  
   It provides a temporary buffer to store all logs recorded during these stages.  

   Additionally, it uses the serial port to output logs directly.

2. **`KernelWriter`**

   The kernel writer handles log output until the file system is mounted and user space is launched. It is used after the initialization of the main kernel subsystems and thus outputs logs not only via the serial port but also graphically on the display.

## Usage

To perform logging, you simply need to use `std.log`.

However, it is recommended to specify a logging scope, allowing you and other developers to conveniently search for and review logs.

Example:

```zig
const std = @import("std");

const log = std.log.scoped(.my_scope);

fn someTemporaryFunction() {
    ...
    log.info("I'm here!", .{});

    ...

    if (someFail) {
        log.err("Bad! Really Bad!", .{});
    }
}
```

This would result in output similar to:

```log
[INFO] my_scope: I'm here!
[ERROR] my_scope: Bad! Really Bad!
```