# Class-based High-Level Interface for Devices

**Subsystem: `dev.obj`** *(hereafter `obj`)*

This subsystem serves as an intermediary between device drivers and other system components.
With this interface, drivers can provide a high-level implementation for interacting with devices within the system.

The device management system `dev` initially provides only low-level functionality for working with devices, specifically through `dev.Bus` buses and `dev.Device` structures. This enables linking the appropriate drivers with the appropriate devices.

However, a driver must also provide the system with an interface to work with the device. Therefore, a class/object system was developed to offer high-level access to devices.

## Classes and Objects

To keep things simple, classes are straightforward: a class is just a standard type in Zig. The only requirement is that **the type must be a structure**.

Thus, any driver can implement its own class without issue, though this is only meaningful if another component can interact with that class, i.e., if other code also references the class structure used by the driver.

The kernel provides several classes that are already used by the system:

- Source code: `/dev/classes`.
- Import: `dev.classes`.

The main idea of class implementation is to define a virtual method table (`vtable`) so that the driver provides its own implementation of certain functions. At the same time, based on `vtable` functions, an API for convenient and straightforward device interaction can be implemented within the class itself.

This serves as a high-level interface that enables other system components to interact consistently with devices of specific classes.

### Implementing the Interface in a Driver

What needs to be implemented depends on the specific class, as the `obj` subsystem does not impose restrictions.

As described above, it’s assumed that a class provides certain fields and a virtual table (`vtable`) so the driver can fill in the relevant values. Thus, the driver needs to implement specific methods for the `vtable` and set field values according to the class structure.

However, the driver may also need to store additional data for each object. Therefore, the class structure can be extended via `obj.Inherit`:

```zig
obj.Inherit(Base: type, Derived: type) type
```

Returns a new type that includes both the base class and the derived class.

Essentially, it’s a structure with fields:
- `base: Base`;
- `derived: Derived`.

This function safely merges two structure types into one, ensuring that:
- `pointer_to_base == pointer_to_new_inherit_struct`
- `pointer_to_derived != pointer_to_new_inherit_struct`

This allows you to create an instance of the new structure and safely cast it to the base type `Base` (but not to the `Derived` type).

This approach is necessary due to the way memory allocation and [object registration](#object-registration) are handled in this subsystem.

## Object Registration

To register a high-level device object in the system, follow these steps:

1. Allocate an object of the specified class (type).
2. Initialize and configure the object’s fields (structure).
3. Add the object to the object subsystem to make it accessible.

**Object Allocation**:

```zig
obj.new(T: type) Error!*T
```

Allocates an object of class `T`. Returns an `NoMemory` error if it fails.

- `T`: the class type.

If necessary, you can also free the allocated object:

```zig
obj.free(T: type, object: *T)
```

- `T`: the class type.
- `object`: a pointer to the object being freed.

**Initialization**:

Initialization depends solely on the specific class, so make sure you complete all necessary steps for the particular class and fill in the required structure fields.

**Adding the Object**

> [!IMPORTANT]
> If you used `obj.Inherit` to extend the implemented class, you must specify the **base class** type when registering the object and pass a pointer to the base class structure as `object`.

```zig
obj.add(T: type, object: *T)
```

Registers an object of the specified class `T` in the object subsystem.

- `T`: the class type.
- `object`: a pointer to the object.

To remove an object from the system:

> [!NOTE]
> Resource deallocation occurs automatically when an object is removed. Calling `obj.free(...)` is redundant.

```zig
obj.remove(T: type, object: *T)
```

Removes a previously registered object and frees the allocated resources.

- `T`: the class type.
- `object`: a pointer to the object being removed.