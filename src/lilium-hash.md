# Lilium Symbol Lookup

`DT_LILIUM_HASH` provides an alternative to the Sys-V `DT_HASH` for symbol lookup in shared objects and dynamically linked executables. 
It uses a more efficient table layout and higher enthropy algorithms.to hash symbol names.

When `DT_LILIUM_HASH` is present, `DT_LILIUM_HASHENT` is also present and specifies the size (in bytes) of hash values. Note that the required size is determined by the precise algorithm in use.

## Header

The `DT_LILIUM_HASH` structure begins will the following structure, aligned on a 16-byte boundary.
Note that the sizes of the header fields are controlled by the `EI_CLASS` entirely, a

```rust
#[repr(C, align(16))]
pub struct LiliumHashHeader {
    pub lh_symoffset: Elf_Word,
    pub lh_algorithm: Elf_Word,
    pub lh_bucket_count: Elf_Word,
    pub lh_nkeys: Elf_Word,
}
```

`lh_symoffset` is the offset into the symbol table at which the first symbol in the hashtable is. This is at least `1` since entry `0` is never present.

Following the header is the key array, which are `lh_nkeys` elements of size given by `DT_LILIUM_HASHENT`, these keys initialize the hash (`lh_nkeys` may be `0`).

Following the hash is the bucket array, which is `lh_bucket_count` long. `lh_bucket_count` is always a power of 2 (this simplifies lookup). The bucket array contains 8 byte elements, given by this structure:
```rust
#[repr(C)]
pub struct LiliumHashBucket {
    pub sym_entry: u32,
    pub bucket_info: u32,
}
```

`bucket_info` is a bitfield, with the following format

| Field | Start Bit | Size (Bits) | Description |
|-------|-----------|-------------|-------------|
| `ents` | `0`      | `16`        | The number of entries in the bucket |
| `filtersz` | `16` | `21`        | The number of entries before the bucket which form the filter. This is a power of two |
| `gather`  | `26`  | `30`        | The number of most significant bits to extract from the hash and check against the filter |

(Unspecified bits must be set to `0` and are ignored)

## Symbol Lookup

The algorithm for looking up a symbol with name `N` is as follows:
1. Given hash algorithm `H`, selected by `lh_algorithm` and keyed by the keys array, let `hash = H(N)`,
2. Compute the bucket. The index into the bucket array is `hash % lh_bucket_count` (note that because `lh_bucket_count` is guaranteed to be a power of two, this can be implemented efficiently with a bitmask)
3. Read the bucket. The `sym_entry` is the index into the symbol table, and the index into the chain array is `sym_entry - lh_symoffset`. If `sym_entry` is `0`, the entry is not present. Other values less than `lh_symoffset` are not permitted and are an error.
4. Check the symbol against the filter. To do this, the most significant `gather` bits are extracted from `hash` as a check value. Then