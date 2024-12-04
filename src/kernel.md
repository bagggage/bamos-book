# Kernel

## Architecture

The BamOS kernel is a **monolithic** kernel, aimed at providing a complete set of functions necessary for working with various types of devices, interacting with the file system, fast and simple memory management, process and thread handling, and other features.

The main goal of its design is to **strike a balance between simple code and functionality, while ensuring the best possible performance**. These factors are essential to consider when developing new components of the system and expanding existing ones.

Therefore, despite the kernel’s monolithic nature, the code is clearly divided according to its purpose. All available tools and techniques are used to organize the code:
- Separation into files and folders.
- Implementation of distinct structures to create a clear hierarchy.
- Grouping of specific functions/constants/variables/types within code files.

This helps maintain code readability and scalability.

## Rationale

Monolithic kernels are often more tightly coupled compared to microkernels, which can lead to increased code dependency and is often considered bad practice for large-scale software. However, despite potential drawbacks, the primary reason for choosing this approach is to avoid the dynamic code issues seen in microkernels: system components do not know the complete system configuration in advance or the availability of subsystems. This necessitates dynamic identification of individual modules, their dynamic loading, and other aspects that are primarily **service-related** and do not directly provide specific functionality. This complicates the system without adding significant new capabilities for device management.

Moreover, an OS is always aimed at performing common tasks: working with hardware, file systems, running and executing programs, managing memory, and more. This is especially relevant for general-purpose operating systems, which is what BamOS aspires to be.

Thus, it is logical to implement pre-required known components in one place, gaining all the benefits of compile-time optimization and more.

## Structure

The kernel can be represented as a tree-like structure, starting with the main kernel subsystems and branching out with more details.

Thanks to the clear project organization, the subsystem hierarchy mostly mirrors the directory and file hierarchy of the source code.

> [!NOTE]
> The structure reflects the kernel's current [development stage](./current-progress.md).

### Subsystems:

> The most significant ones are **highlighted** in bold. Some have documentation available via links.

- **Architecture**
  - Template
  - **x86-64**
    - Paging
    - I/O
    - Registers
    - Interrupt system implementation
      - APIC (LAPIC, I/O APIC), PIC
      - IDT / Exceptions
- [**Devices**](./kernel/devices.md)
  - [**Classes**](./kernel/devices/classes.md)
  - **Standards**
    - ACPI
    - PCI
  - Built-in drivers
  - [**Interrupt System**](./kernel/devices/interrupts.md)
    - High-level interrupt handling
    - Resource management
  - [**Input/Output System**](./kernel/devices/io.md)
    - [Registers](./kernel/devices/registers.md)
    - Access mechanisms
        - MMIO, PIO
- [**Virtual Memory**](./kernel/memory.md)
  - [Page table management](./kernel/memory.md#working-with-page-tables)
  - [Physical page allocator](./kernel/memory.md#page-allocator-vmpageallocator)
  - Various memory allocators
- **Virtual File System**
- **Logging System**
- **Boot**
- **Video** (*may be removed in the future*)
  - Framebuffers
  - Text output
- **Utilities**
  - Data structures

### Main Components:

The kernel also includes specific components that are quite self-contained and do not belong to any particular subsystem.

- **Main**/**Start**

  The kernel's entry point.

- **Panic**/**Trace**

  Kernel panic handling and call stack tracing based on debugging information.

- **Symmetric Multiprocessing** (SMP)

  Functionality for working with individual processor cores.

## Additional Information

For more information, feel free to explore the kernel’s [source code](https://github.com/bagggage/bamos/tree/main/src/kernel), which also contains additional [documentation](https://bagggage.github.io/bamos/) in the form of comments.