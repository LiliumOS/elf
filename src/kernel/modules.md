# Kernel Modules

## Module Headers

Each kernel Module contains a header, which is pointed to from the module in the following ways:
* a `PT_LILIUM_KMODULE` segment, and
* a `DT_LILIUM_KMODULE` dynamic tag.

Where both are present, they must point to the same array of module headers

### Module Headers Segment

The Module headers Segment is a segment that is defined to enable kernels to check 