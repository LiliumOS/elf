# Elf Signature Format

An ELF File on Lilium can be signed by adding three segment headers to the file:
* `PT_LILIUM_SIGN_CERT`,
* `PT_LILIUM_SIGN_TIME`, and
* `PT_LILIUM_SIGNATURE`.

There may be at most one of each of those segments in a file, and if any of those segments are present, all must be present.

