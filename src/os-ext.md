# Generic OS Extensions

## OS ABI

Lilium supports `OSABI_SYSV` (0) binaries and `OSABI_LILIUM` (TODO) binaries. 

## Program Loading

The default loader is `/lib/ld-lilium-<arch>.so`. 

The Lilium Kernel and Default USI loader impose the following constraints on objects loaded:
* Any Dynamically linked executable (executable a `PT_INTERP` segment) must be Position Independent (either PIE or full PIC). This is enforced by the loader.
* Any `PT_LOAD` or `PT_GNU_STACK` segment must not set both `PF_W` and `PF_X`. 

## OS Specific Program Header Types

In addition to standard Segment Types, and Arch-Specific ones, the following segment types are supported, with the specified values.

| Name                    | Value        | Description                       |
|:-----------------------:|--------------|-----------------------------------|
| `PT_LOOS`               | `0x60000000` | Lower bound for OS Specific Segment types |
| `PT_LILIUM_SIGN_CERT`   | `0x60010000` | Certificate used for signing (Reserved). |
| `PT_LILIUM_SIGNATURE`   | `0x60010001` | ELF File Signature (Reserved). |
| `PT_LILIUM_SIGN_TIME`   | `0x60010002` | 8-byte timestamp (seconds) the file was signed (Reserved). |
| `PT_GNU_EH_FRAME`       | `0x6474e550` | Recognized for compatibility with GNU and LLVM toolchains. |
| `PT_GNU_STACK`          | `0x6474e551` | Recognized for compatibility with GNU and LLVM toolchains. Ignored but validated like `PT_LOAD` (must not be `PF_W + PF_X`) |
| `PT_GNU_RELRO`          | `0x6474e552` | Recognized for compatibility with GNU and LLVM toolchains. |
| `PT_LILIUM_LOKERNEL`    | `0x6FE00000` | Lowest segment type for use by the Lilium Kernel|
| `PT_LILIUM_KMODULE`     | `0x6FE00000` | Segment that maps to the kernel module info |
| `PT_LILIUM_LOKSPECIFIC` | `0x6FEFFF00` | Lowest segment type defined by the specific kernel |
| `PT_LILIUM_HIKERNEL`    | `0x6FEFFFFF` | Highest segment type for use by the Lilium Kernel |
| `PT_LILIUM_USICOMPAT`   | `0x6FF00000` | USI Compatibility support |
| `PT_LILIUM_LOUSI`       | `0x6FF00001` | Lowest segment type reserved for the USI impl[^1] |
| `PT_LILIUM_HIUSI`       | `0x6FFFFFFF` | Highest segment type reserved for the USI impl[^1] |
| `PT_HIOS`               | `0x6FFFFFFF` | Upper bound for OS Specific Segment types |

### Kernel Range

The program types between `PT_LILIUM_LOKERNEL` and `PT_LILIUM_HIKERNEL` are for use in kernel modules (and the Lilium kernel itself). They are ignored or rejected when loading a userspace program or shared object. Program types between `PT_LILIUM_LOKSPECIFIC` and `PT_LILIUM_HIKERNEL` are not defined by the document and have kernel-specific meaning.

[^1]: See [USI Compatibility][segment-usi-compat] for details on how to use this segment portably (between USI impls)

### USI Compatibility {#segment-usi-compat}

[segment-usi-compat]: #segment-usi-compat

When using USI-specific segment type the following compatibility protocol should be used to ensure portable interpretation of these segments:
* Each group of USI-specific segment types shall be defined adjacent in memory. They do not need to be contiguous but must not have any intervening segments between them. 
* A `PT_LILIUM_USICOMPAT` segment must be defined, which covers the following structure, and each USI-specific segment type that conforms to the specified USI:

```rust
#[repr(C,align(16))]
struct LiliumUsiCompat{
    pub usi_id: Uuid,
    pub flags: u32,
    __reserved: [u32; 3],
}
```

`usi_id` is set to the `Uuid` of the USI impl. If it is set to the Full (All 1s) UUID, it matches all USI types not covered by another `PT_LILIUM_USICOMPAT`. 
This wildcard segment must appear last or the result is unspecified. 
In the case of the full UUID, the segment must not cover any USI-Specific types. 

Each word in `__reserved` must be set to `0` or the segment is invalid.
`flags` shall be set to the following:
* Bit 1 shall be set to 1 if the loader should reject loading for this USI impl Id. This can be used with a wildcard to specify that only specific USI implementations are supported,
* Bits 24-31 have USI-specific semantics. They must be ignored by a USI that skips the header. They must not be used in a valid segment for a wildcard `usi_id`.
* Other bits are reserved and must not be set in a valid segment.

`PT_LILIUM_USICOMPAT` and USI-specific segments are not interpreted by the Kernel Loader. The Elf Interpreter for the USI is responsible for correctly interpreting this segment.
The behaviour of an interpreter that reads an invalid segment, even one that is ignored, is not specified. Such interpreters should refuse to load the object.

Support for this protocol is not required to support USI-Specific segment types. However if a `PT_LILIUM_USICOMPAT` segment exists and the protocol is not implemented, all USI-Specific segments covered by any `PT_LILIUM_USICOMPAT` shall be ignored by the loader. 

A specified USI impl id may have multiple associated `PT_LILIUM_USICOMPAT` segments. This can be useful, for example, if only some USI segments are covered by a `PT_LOAD` segment.

## OS Specific Section Types

In addition to standard section types, and arch-specific ones, the following section types are supported, with the specified values

| Name                           | Value        | Description                         |
|:------------------------------:|--------------|-------------------------------------|
| `SHT_LOOS`                     | `0x60000000` | Lower bound for OS Specific section types |
| `SHT_LILIUM_REQUIRE_SUBSYSTEMS`| `0x60000000` | Subsystems required for loading the module |
| `SHT_LILIUM_LOKERNEL`          | `0x6FE00000` | Lowest Section type for use by the Lilium Kernel|
| `SHT_LILIUM_KMODULE`           | `0x6FE00000` | Section that maps to the kernel module info |
| `SHT_LILIUM_LOKSPECIFIC`       | `0x6FEFFF00` | Lowest section type defined by the specific kernel |
| `SHT_LILIUM_HIKERNEL`          | `0x6FEFFFFF` | Highest section type for use by the Lilium Kernel |
| `SHT_HIOS`                     | `0x6FFFFFFF` | Upper bound for OS Specific section types |

### `.lilium.require-subsystems`

The special section `.lilium.require-subsystems` (of type `SHT_LILIUM_REQUIRE_SUBSYSTEMS`) may provide a mechanism for communicating to the dynamic loader that a specified subsystem is required to be loaded. The Dynamic Loader will then make appropriate system calls when loading the module (this occurs after calling `DT_PREINIT_ARRAY` entries in an executable, but prior to calling `DT_INIT_ARRAY` entries). See [`DT_LILIUM_REQUIRE_SUBSYSTEMS`] for the behavior of dynamic linkers that process this section. Both `SHF_OS_NONCONFORMING` and `SHF_ALLOC` must be set.

The `sh_link` entry is a section index that refers to a `SHT_STRTAB` section that is `SHF_ALLOC`. `sh_entsize` is the length of the entries. For `ELFCLASS32`, only a `sh_entsize` of 4 is supported. For `ELFCLASS64`, `sh_entsize` may be 4 or 8. The section contains an array of offsets (byte offsets) into the string table mentioned by `sh_link`. If the entry is not `0` (an empty string), then the string is a name of a subsystem to pass to `OpenSubsystem`. 

When a `SHT_LILIUM_REQUIRE_SUBSYSTEMS` section is processed by a link editor, the entries must be adjusted so that they have the correct offsets after string tables are concatenated. When linking `.lilium.require-subsystems` into an executable or shared objects, this is the dynamic string table that will be pointed to by `DT_STRTAB`. 

How other sections of type `SHT_LILIUM_REQUIRE_SUBYSTEMS` are handled during linking is not specified. The link editor may include them in `DT_LILIUM_REQUIRES_SUBSYSTEMS`. 

## OS Specific Dynamic Tags

Dynamic tags between `DT_LOOS` and `DT_LILIUM_LOKERNEL` obey the rule set by `DT_ENCODING`. That is, even numbered tags will use `d_ptr` and odd numbered tags will use `d_val`.

| Name                           | Value        | `d_un`  | Executable | Shared Object | Description                         |
|:------------------------------:|--------------|---------|------------|---------------|-------------------------------------|
| `DT_LOOS`                      | `0x6000000D` | N/A     | N/A        | N/A           | Lower bound for OS Specific dynamic tags |
| `DT_LILIUM_HASH`[^3]           | `0x6000000E` | `d_ptr` | Optional   | Optional[^2]  | Provides an alternative to `DT_HASH` |
|`DT_LILIUM_REQUIRE_SUBSYSTEMSSZ`| `0x6000000F` | `d_val` | Optional   | Optional      | The total size (in bytes) of `DT_LILIUM_REQUIRE_SUBSYSTEMS` | 
|`DT_LILIUM_REQUIRE_SUBSYSTEMS`  | `0x60000010` | `d_ptr` | Optional   | Optional      | Pointer to the `SHT_LILIUM_REQUIRE_SUBSYSTEMS` section |
|`DT_LILIUM_REQUIRE_SUBSYSTEMSENT`| `0x60000011`| `d_val` | Optional   | Optional      | Entry size for `DT_LILIUM_REQUIRE_SUBSYSTEMS` |
| `DT_LILIUM_LOKERNEL`           | `0x6FE00000` | N/A     | Disallowed | N/A[^4]       | Lowest dynamic tag for use by the Lilium Kernel |
| `DT_LILIUM_KMODULES`           | `0x6FE00000` | `d_ptr` | Disallowed | Optional      | Points to the `PT_LILIUM_KMODULES` segment    |
| `DT_LILIUM_KMODULESZ`          | `0x6FE00001` | `d_val` | Disallowed | Optional      | Contains the size (in bytes) of the `DT_LILIUM_KMODULES` segment |
| `DT_LILIUM_HIKERNEL`           | `0x6FEFFFFF` | N/A     | Disallowed | N/A[^4]       | Highest dynamic tag  for use by the Lilium Kernel |
| `DT_HIOS`                      | `0x6FFFF000` | N/A     | N/A        | N/A           | Upper Bound for OS Specific dynamic tags |
| `DT_GNU_HASH`[^3]              | `0x6FFFFEF5` | `d_ptr` | Optional   | Optional[^2]  | Defined for compatibility with GNU/LLVM Toolchains. Provides an alternative to `DT_HASH`|

[^2]: A shared object must have at least one of `DT_HASH`, `DT_GNU_HASH`, or `DT_LILIUM_HASH`. Despite the Generic Specification, a shared object may omit a `DT_HASH` tag if (and only if) it defines `DT_GNU_HASH` or `DT_LILIUM_HASH`. However, it is not recommended to omit `DT_HASH` when using `DT_LILIUM_HASH`.

[^3]: Due to inconsistencies in the constraints between `DT_LILIUM_HASH` and `DT_GNU_HASH`, these tags cannot both appear in a shared object

[^4]: Dynamic tags in the range `DT_LILIUM_LOKERNEL` and `DT_LILIUM_HIKERNEL` must be ignored or treated as invalid by a userspace dynamic linkers. These are specifically used in the kernel. As kernel modules are exclusive shared objects, they cannot validly appear in an executable.

### `DT_LILIUM_REQUIRE_SUBSYSTEMS`

The `DT_LILIUM_REQUIRE_SUBSYSTEMS` tag is used to point to the `SHT_LILIUM_REQUIRE_SUBSYSTEMS` section. It may appear in both a dynamically linked executable and a shared object.
The `DT_LILIUM_REQUIRE_SUBYSTEMS` tag is interpreted the same as a `SHT_LILIUM_REQUIRE_SUBSYSTEMS` section, except the offsets are considered offsets into the `DT_STRTAB`. 

If `DT_LILIUM_REQUIRE_SUBSYSTEMS` is present, then `DT_LILIUM_REQUIRE_SUBSYSTEMSSZ` and `DT_LILIUM_REQUIRE_SUBSYSTEMSENT` must also be present.

The `DT_LILIUM_REQUIRE_SUBSYSTEMS` tag is processed only by the runtime linker (not by the kernel), and each named subsection is attempted to be loaded before any code in the module is executed (Except pointers in the `DT_PREINITARRAY` for an executable). This includes both `DT_INIT` and pointers in the `DT_INITARRAY`.
If loading fails, the behaviour is as follows:
* If the error occurs while loading the executable itself or a `DT_NEEDED` entry, the process is terminated with an exception,
* If the error occurs while loading a module via `rtld_open_object` (defined in `libusi-rtld.so`), either `rtld_open_object` returns the error code from `OpenSubsystem` (typically `MODULE_LOAD_FAILED`) or the process is terminated with an exception,
* If the error occurs while loading a module in any other manner, the behaviour is not specified.

Note that the `core` subsystems (`base`, `thread`, `io`, `process`, `debug`, and `kmgmt` currently) may be ignored by runtime loader if it knows to be running a kernel that supports those subsystems. Core subsystems supported by a given version kernel are always loaded.

[`DT_LILIUM_REQUIRE_SUBSYSTEMS`]: #dt_lilium_require_subsystems

