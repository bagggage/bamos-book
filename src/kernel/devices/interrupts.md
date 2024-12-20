# Interrupts Subsystem

**Subsystem: `dev.intr`** *(hereafter `intr`)*

The interrupt subsystem provides a high-level interface for working with interrupts within drivers and other kernel components.

The core concept is similar to the interrupt handling in GNU/Linux, but this is only a superficial resemblance, and this implementation does not guarantee any compatibility.

Interrupts are set up, configured, and balanced across CPUs automatically. The kernel or driver developer only needs to call specific API functions to gain access and request the necessary resources for handling interrupts.

The interrupt system supports and distinguishes between two main types of interrupts: [IRQ](#irq) and [MSI](#msi), allowing drivers to request the required ones.

## IRQ
```zig
intr.requestIrq(
    pin: u8,
    device: *dev.Device,
    handler: Handler.Fn,
    tigger_mode: TriggerMode,
    shared: bool
) Error!void
```

Allows requesting and setting an interrupt handler for a specific `pin`.

- `pin`: specifies the exact line number for which the handler is to be set.

  This line is referenced according to the device's documentation and, if necessary, automatically converted into the line number connected to the interrupt controller.
    
  For example: on the **x86-64** architecture, IRQ-8 corresponds to pin 8.

- `device`: the device object for which the interrupt is requested.
- `handler`: the interrupt handler, see [interrupt handler](#interrupt-handler) below.
- `trigger_mode`: the trigger mode: `.edge`, `.level_low` or `.level_high`.
- `shared`: whether other devices can use this line or if it must be dedicated to the current device.

  It is recommended to always set this parameter to `true` and only set it to `false` in exceptional cases where there are strong justifications. If `false` is used, no other device in the system will be able to use this line, and as a result, some devices may not have an interrupt handler at all.

```zig
intr.releaseIrq(
    pin: u8,
    device: *const dev.Device
) void
```
  
Removes the interrupt handler for the given device and frees the resources.

- `pin`: the interrupt line number specified when calling `intr.requestIrq`.
- `device`: the device for which the interrupt was requested.

## MSI
> [!NOTE]
> This is a platform-independent MSI implementation, which does not work directly with devices
> but only configures the necessary components related to the CPU and interrupt controller.
> Therefore, drivers using MSI must also properly configure the message generated by the device,
> including its address and data, which are provided through `intr.getMsiMessage`, see below.

> [!TIP]
> To work with MSI on PCI devices, use the functionality provided by the [PCI bus driver](./Pci.md).

```zig
intr.requestMsi(
    device: *dev.Device,
    handler: Handler.Fn,
    trigger_mode: TriggerMode
) Error!u8
```

Used to request and configure **MSI** interrupts.

- `device`: the device object for which the interrupt is requested.
- `handler`: the interrupt handler, see [interrupt handler](#interrupt-handler) below.
- `trigger_mode`: the trigger mode: `.edge`, `.level_low`, `.level_high`.

The function returns a **unique MSI interrupt number**, which is used for further handling.

```zig
intr.releaseMsi(idx: u8) void
```

Removes the MSI interrupt handler and frees the resources.

- `idx`: the unique MSI interrupt number obtained when calling `intr.requestMsi`.

```zig
intr.getMsiMessage(idx: u8) Msi.Message
```

Returns the `intr.Msi.Message` structure, allowing you to retrieve the necessary address and data for the message that the device will send when generating an interrupt.

- `idx`: the unique MSI interrupt number obtained when calling `intr.requestMsi`.

## Interrupt Handler

A high-level interrupt handler is a simple function like:

```zig
fn handler(device: *dev.Device) bool
```

It takes the device for which the interrupt was configured.
The function returns a value because **IRQ** interrupts can be shared and may be triggered by multiple devices. It’s necessary to determine whether this interrupt was generated by your specific device.

The handler should check if the interrupt relates to its device. If not, it should simply return `false`; otherwise, it should handle the interrupt and return `true`.

## Architecture

To achieve such simplicity in interaction, a specific architecture was introduced that separates platform-dependent code from the general implementation.

On the platform side, the architecture must provide functions to initialize the interrupt system and obtain an object representing the platform's interrupt controller.

This object is of type `intr.Chip`.

The `intr.Chip` structure must provide the following functions:

- `eoi`: sends the **End Of Interrupt** signal to the interrupt controller.
- `bindIrq`: sets up and binds the `intr.Irq` interrupt.
- `unbindIrq`: unbinds the `intr.Irq` interrupt.
- `maskIrq`: masks the `intr.Irq` interrupt.
- `unmaskIrq`: unmasks the `intr.Irq` interrupt.
- `configMsi`: configures and sets up the `intr.Msi` interrupt.

Currently, BamOS supports using only one interrupt controller at a time.

The platform code provides the `intr.Chip` structure via the `arch.intr.init` function during the interrupt system initialization, which is invoked by the high-level interrupt system `intr`.

Thus, all platform-dependent code is encapsulated within one structure. When writing platform code, it is strongly recommended to implement a unified interface for working with interrupt vector tables and other features common to all CPU architectures. Then, specific code can be written to work with the interrupt controller(s), dynamically selecting the most suitable/available controller during initialization.

This is the approach taken by the **x86-64** architecture implementation.

## Example Usage

```zig
fn probe(device: *dev.Device) dev.Driver.Operations.ProbeResult {
    ...

    // Request IRQ
    dev.intr.requestIrq(
        IRQ_NUM, device, intrHandler, .edge, true
    ) catch {
        ...

        return .failed;
    };

    ...
    return .success;
}

// IRQ Handler
fn intrHandler(device: *dev.Device) bool {
    ...

    return true;
}
```