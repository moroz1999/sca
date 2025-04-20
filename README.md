# Screen Animation Format (.sca)

## General Description

The `.sca` format is designed for storing frame-based animations on the ZX Spectrum. The base format (type 0) is a sequence of uncompressed full-screen frames, each 6912 bytes in size. Frame delays are specified in units of vertical interrupts (1/50 second). The file consists of a header, a delay table, and frame data.

**Byte order:** little-endian (least significant byte first).

---

## Header Structure

| Offset   | Size    | Description |
|----------|---------|-------------|
| 00–02    | 3 bytes | Marker: ASCII string `"SCA"` |
| 03       | 1 byte  | Version (format version) |
| 04–05    | 2 bytes | Frame width in pixels |
| 06–07    | 2 bytes | Frame height in pixels (max 192) |
| 08       | 1 byte  | Recommended border color |
| 09–10    | 2 bytes | Frame count |
| 11       | 1 byte  | Payload type |
| 12–13    | 2 bytes | Payload offset (from start of file) |

---

## Payload

The structure of the payload depends on the specified type.

### Payload Type 0 – Unpacked 6912-byte Screens

#### Delay Table

Contains `N` bytes, where `N` is the number of frames.  
Each byte represents the delay for the corresponding frame in units of vertical interrupts (1/50 second).

#### Frames

Each frame is 6912 bytes of ZX Spectrum screen memory in `SCREEN$` format (bitmap + attributes).  
Frames are stored sequentially with no padding.

---
