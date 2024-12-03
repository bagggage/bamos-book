# Device Management Subsystem

**Subsystem: `dev`**

One of the primary functions of any operating system is to simplify interactions with the hardware on which it operates.

- For the end user, this is almost a seamless part of the OS, performing tons of critically important work.

- For software/driver developers, this part of the system should provide a convenient, high-level interface for interacting with and managing devices, as well as functionality for working with them within the kernel itself.

Device management is divided into two levels:

- Low-level
- High-level

The low-level involves system interaction with drivers, providing drivers with basic information about devices.
The high-level allows device drivers to offer a high-level interface for the system to interact with the device. This level is described in the [**Classes**](./devices/classes.md) section.

The low-level is based on three main components.

### 1. Buses `dev.Bus`

Conceptually, buses allow devices and their drivers to be grouped, enabling more efficient matching and combining of necessary elements.

The bus stores a list of devices associated with it, as well as the drivers intended to work with devices on this bus. When new devices are registered on the bus, for example, through a bus driver, the bus's drivers are checked for compatibility with the device. If they match based on specific parameters, the bus driver calls the `probe` function of the driver for further compatibility checks by the driver itself, and if successful, the device is assigned to the driver.

When a device is removed, the reverse process occurs, first detaching the device from the driver and then from the bus.

If no suitable driver is found for a device, it remains *unbound* and will be checked again if a new driver is registered for that bus.

When a driver is added, the bus iterates over all *unbound* devices and performs the same compatibility checks with the added driver.

---

To register a new bus in the system, the driver must call `dev.registerBus(...)` and pass previously initialized `dev.BusNode` structure.

> [!NOTE]
> `dev.BusNode` must be a staticaly allocated.

```zig
var bus = dev.Bus.init(
    // Name
    "my-bus-name", 

    // Operations
    .{
        .match = match,
        .remove = remove
    }
);
```

- `name`: bus name, must be known at compile time to allow device drivers to dynamically access the bus object. The driver developer can find the name of a specific bus in the documentation and then call `dev.getBus`.

- `ops`: bus operations.

  - `match` to check device compatibility with the driver.
  - `remove` to free resources when the device is removed.

### 2. Devices `dev.Device`

Devices represent real or virtual components within the system, managed by the bus. They can be attached to a specific driver, which will implement certain functionalities for the device.

Essentially, they are just structures that can store various specific data required by the device driver or bus to interact with them.

Devices can be registered either by device drivers or by the bus driver, for example, by enumerating all devices on the bus or through hot-plug interrupt detection.

---

Device registration is performed by calling `dev.registerDevice(...)`.

> [!TIP]
> If it is the bus driver, you can use `dev.Bus.addDevice(...)` method to register devices.

```zig
fn registerDevice(
    comptime bus_name: []const u8,
    name: Name,
    driver: ?*const Driver,
    data: ?*anyopaque
) !*Device
```

- `bus_name`: name of the bus this is device belongs to. Example: `"pci"`.

- `name`: name of the device.

  Since a device's name may change during operation (e.g., from the moment of registration to when the driver takes control), it is dynamic and may require memory allocation. To simplify this process, the subsystem provides APIs:

  - `dev.nameOf(str)`: sets the name based on a constant string.
  - `dev.nameFmt(fmt, args)`: sets the name through string formatting, useful for dynamic naming, for example: `pci:0000:00:00.00`, `pci:00fc:aa:20.01`

- `driver`: the driver managing the device.

  This is optional and can be set to `null`, in which case the bus is responsible for finding a suitable driver for the device.

  If the device driver registers a device that is already compatible with it, it can pass the driver object obtained during registration.

- `data`: any specific bus/driver/device data that may be used by device driver.

### 3. **Drivers** `dev.Driver`

A driver is the functional element in the device system, adding new capabilities for each device and implementing specific functionality.

To work within the subsystem, the driver developer must implement the following functions:

```zig
fn (device: *dev.Device) dev.Driver.Operations.ProbeResult
```
The function can use various methods to check the driver’s compatibility with a specific device.
 
If successful, it initializes the device, performs the necessary setup for its further use, and returns `success`.
Otherwise, it returns `missmatched`, or any other value defined in `dev.Driver.Operations.ProbeResult` enum.

```zig
fn (device: *dev.Device) void
```
Called when a device is removed, to detach it from the driver, allowing the driver to free up all resources allocated for this device.

---

Driver registration is performed by calling `dev.registerDriver(...)` and passing previously initialized `dev.DriverNode` structure.

> [!NOTE]
> `dev.DriverNode` must be a staticaly allocated.

```zig
var driver = dev.Driver.init(
    // Name
    "my-driver-name",

    // Operations
    .{
        .probe = .{ .universal = probe },
        .remove = remove
    }
);
```

- `name`: the driver name, must also be known at compile time, as it is purely informational and not intended to change at runtime.
- `ops`: operations implemented by the driver, such as `probe`, `remove`, etc.

Then, to register driver within the system:

```zig
try dev.registerDriver("bus-name", &driver);
```

## Internal Subsystems

Device management requires the use of various mechanisms. Therefore, the `dev` subsystem also includes its internal systems:

> More detailed information is available via the links.

- [**Objects and classes**](./devices/classes.md) `dev.obj`/`dev.classes`

  Enables drivers to provide a high-level interface for working with devices.

- [**Input/Output**](./devices/io.md) `dev.io`

  This subsystem provides a convenient API for performing platform-independent read/write operations and working with device registers.

- [**Interrupts**](./Interrupts) `dev.intr`

  Manages resources involved in interrupts and provides a convenient API for working with them.

- [**Registers API**](./devices/registers.md) `dev.regs`

  Useful common API for work with devices registers.

## Internal Implementations

The subsystem also includes various drivers and components for the most common hardware and standards.

### Platform Bus

The first and special bus in the device subsystem is the platform bus: `platform`. It is provided for platform-specific devices that cannot be discovered without specific code, such as RTC or HPET.

The key feature here is that platform drivers themselves are responsible for adding devices to this bus, not the bus driver.

Thus, a special functionality is used for platform drivers.

When registering the driver, the platform bus must be passed as the target for the driver, and the `probe` function should be specified as the `.platform` member. The function is called when the driver is registered, so the driver object is passed to it upon invocation, as the driver module itself does not have prior access to the registered driver object.

The `probe` function should look as follows:

```zig
fn probe(self: *dev.Driver) dev.Driver.Operations.ProbeResult
```

As seen in the prototype, the platform driver object is passed to the function, not the device. The driver should check for the presence of devices it targets, and if found, register them in the system by calling the member function `dev.Driver.addDevice(...)`, which automatically associates the added device with the bus and the driver.

### Standards

- **PCI** `dev.pci`

  PCI bus driver.

- **ACPI** `dev.acpi`

  Interface for working with *Advanced Configuration and Power Interface* and various tables.

### Built-in Drivers

Drivers that are part of the kernel are located at `/dev/drivers`.

Built-in drivers do not require dynamic linking and are initialized in the order specified in the `dev.AutoInit` structure.

Some bus drivers, such as `dev.pci`, are also considered built-in.