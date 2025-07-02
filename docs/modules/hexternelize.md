# Externalized Strings in Hashes (HEXTERNELIZE)

The `HEXTERNELIZE` functionality allows Valkey modules to store raw `char*` pointers and their lengths directly as values within hash fields. This is an advanced feature intended for modules that manage their own memory for string-like data and wish to avoid the overhead of SDS string allocations for values stored in Valkey hashes.

## Overview

Normally, hash field values are SDS strings managed by Valkey. When a module uses externalized strings:
- Valkey stores a `ValkeyModuleExternelizeString` struct (containing the `char* buf` and `size_t len` provided by the module) as the value of a hash field.
- A special bit (`FIELD_SDS_AUX_BIT_EXTERNELIZED`) is set in the SDS header of the *field* (not the value) to indicate that its corresponding value is externalized.
- The module is **entirely responsible** for the lifecycle and validity of the memory pointed to by `buf`. Valkey will not allocate, free, or modify this buffer.

## Module API

A module would typically expose its own command (e.g., `MYMODULE.HEXTSET key field buf_ptr_str len_str`) that internally uses `ValkeyModule_Call` to invoke a lower-level command (e.g., `DEBUG.HEXTERNELIZE` or a future dedicated module API function) to achieve this.

The core mechanism involves:
1.  Obtaining a `ValkeyModuleKey*` for the target hash key, opened for writing.
2.  Using an internal server function (like `hashTypeSetExternalized`) to set the field with the provided `char* buf` and `size_t len`. This function will:
    *   Create a `ValkeyModuleExternelizeString` structure.
    *   Store this structure as the value in the hash entry.
    *   Mark the field's SDS with the `FIELD_SDS_AUX_BIT_EXTERNELIZED` auxiliary bit.

## Retrieving Externalized Strings

When standard Valkey commands like `HGET` or `HGETALL` are used on a hash containing externalized fields:
- The value for an externalized field is returned as a special string: `[EXT:<pointer_address>:<length>]`.
    - `<pointer_address>` is the hexadecimal memory address of the `buf`.
    - `<length>` is the decimal length of the buffer.
- This string acts as a marker. The raw content of `buf` is **not** returned directly by these commands.
- Modules that need to access the raw `char* buf` and `len` would typically:
    1.  Open the hash key.
    2.  Iterate through fields or access a specific field.
    3.  For a given field, check its `FIELD_SDS_AUX_BIT_EXTERNELIZED` aux bit.
    4.  If the bit is set, retrieve the pointer to the `ValkeyModuleExternelizeString` struct and access its `buf` and `len` members.

## Persistence

**RDB (Snapshotting):**
- When an RDB snapshot is created, fields marked as externalized will have their special string representation (e.g., `[EXT:0x12345678:100]`) saved as the value.
- The raw `char* buf` content is **not** saved by Valkey's RDB mechanism.
- Upon RDB loading, the special string marker is restored.
- **Module Responsibility:** The module that created the externalized string is responsible for:
    - Persisting the actual content of `buf` through its own means if desired (e.g., module aux fields, separate files).
    - After RDB load, scanning its relevant keys, identifying the `[EXT:...]` markers, and re-linking them to the correctly loaded (or re-allocated) `buf` data.

**AOF (Append Only File):**
- The command used to set the externalized string (e.g., `DEBUG.HEXTERNELIZE key field <addr_str> <len_str>` or a module-specific command) is written to the AOF.
- Upon AOF replay:
    - The logged command is re-executed.
    - **Module Responsibility:** The module must ensure that the `buf_addr_str` and `len_str` arguments are meaningful in the context of AOF replay. This typically means the module must also handle the persistence and reloading of the underlying data that `buf` points to, such that the address and length are valid after replay. If the original memory is not available, the module's command should handle this gracefully (e.g., log an error, set a placeholder, or re-initialize the data).

## `ValkeyModule_ReplyWithExternelize`

Modules can use `ValkeyModule_ReplyWithExternelize(ValkeyModuleCtx *ctx, const char *buf, size_t len)` to send a reply that signifies an externalized string.
- Currently, this function replies with the special string marker `[EXT:<pointer_address>:<length>]`.
- This is primarily intended for debugging or module-to-module communication where the recipient module understands how to interpret this marker.

## Use Cases

- Storing large, read-only data blobs managed by the module, avoiding SDS copying.
- Integrating with external memory allocators or shared memory segments where the data lifetime is managed outside of Valkey's main memory allocation.

## Caveats

- **Memory Management:** The module is **solely responsible** for the memory pointed to by `buf`. Valkey will not free it. Dangling pointers will lead to crashes.
- **Pointer Validity:** Pointers are only valid within the process space of the Valkey server instance where they were set. They are meaningless if an RDB/AOF is loaded into a different process or machine unless the module has a robust mechanism for re-establishing these pointers.
- **Complexity:** This is an advanced feature. Incorrect usage can easily lead to instability or crashes.
- **Overwriting:** When an externalized field is overwritten (either by another externalized string or a normal value), Valkey core will free the `ValkeyModuleExternelizeString` struct it allocated to hold the `buf` and `len`. However, it will **not** free the `buf` itself. If the module needs to reclaim or otherwise manage the old `buf` when it's being replaced, it must do so *before* calling the command that overwrites the externalized field.
