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

### Payload Type 1 – Multicolor Attribute Screens (53c/127c)

This payload type is designed for multicolor modes where the bitmap pattern is constant across all cells, and only attributes change per frame. The visual illusion of additional colors is achieved through dithering patterns.

#### Delay Table

Contains `N` bytes, where `N` is the number of frames.
Each byte represents the delay for the corresponding frame in units of vertical interrupts (1/50 second).

#### Fill Pattern

8 bytes defining the bitmap pattern for each character cell.
This pattern is repeated for all 768 cells on screen (32×24).

Common patterns:
- **53c (checkerboard):** `AA 55 AA 55 AA 55 AA 55` — 50% pixels set, creates ~53 perceived colors
- **127c:** `DD 77 DD 77 DD 77 DD 77` — unequal pixel distribution for ~127 perceived colors

Here's a visual representation of the 8x8 cell patterns:

  53c Checkerboard (AA 55 alternating):

  AA = 10101010  ▓░▓░▓░▓░

  55 = 01010101  ░▓░▓░▓░▓
  
  AA = 10101010  ▓░▓░▓░▓░
  
  55 = 01010101  ░▓░▓░▓░▓
  
  AA = 10101010  ▓░▓░▓░▓░
  
  55 = 01010101  ░▓░▓░▓░▓
  
  AA = 10101010  ▓░▓░▓░▓░
  
  55 = 01010101  ░▓░▓░▓░▓
  
  
  50% ink, 50% paper → visually blends colors equally

  127c Pattern (DD 77 alternating):

  DD = 11011101  ▓▓░▓▓▓░▓
  
  77 = 01110111  ░▓▓▓░▓▓▓
  
  DD = 11011101  ▓▓░▓▓▓░▓
  
  77 = 01110111  ░▓▓▓░▓▓▓
  
  DD = 11011101  ▓▓░▓▓▓░▓
  
  77 = 01110111  ░▓▓▓░▓▓▓
  
  DD = 11011101  ▓▓░▓▓▓░▓
  
  77 = 01110111  ░▓▓▓░▓▓▓
  
  75% ink, 25% paper → color leans toward ink

  With 127c you'd use both DD/77 (75% ink) and its inverse 22/88 (25% ink) patterns across different cells to get more color gradations. The 8-byte fill field allows any custom pattern.


#### Frames

Each frame is 768 bytes of ZX Spectrum attribute data only.
Frames are stored sequentially with no padding.

---

### Payload Type 2 – Packed Frames

This payload type supports compressed frame data with per-frame border color and compressor selection. Each frame is fully self-contained — no lookup tables are needed. Different frames may use different compressors or no compression at all.

**Note:** The region field in the frame content byte defines the actual screen area used for decompression. It overrides the frame width/height values in the global file header (offsets 04–07).

#### Payload Header

| Offset | Size   | Description |
|--------|--------|-------------|
| +0     | 1 byte | Frame content byte `[FFFF RRRR]` |

**Frame content byte** encodes two values in nibbles:

```
  High nibble (bits 7–4): Frame Content Type (FCT)
    0 = mono only (bitmap)
    1 = mono + attributes (bitmap + attrs, two separate blocks)
    2 = attributes only (multicolor 53c/127c)
    3–F = reserved

  Low nibble (bits 3–0): Region
    0 = top         (rows 0–7,   8 char rows)
    1 = middle      (rows 8–15,  8 char rows)
    2 = bottom      (rows 16–23, 8 char rows)
    3 = top+middle  (rows 0–15,  16 char rows)
    4 = mid+bottom  (rows 8–23,  16 char rows)
    5 = full        (rows 0–23,  24 char rows)
    6–F = reserved
```

#### Fill Pattern (only if FCT = 2)

8 bytes defining the bitmap pattern for each character cell, placed immediately after the payload header. This pattern is applied to the screen bitmap once before animation starts. Only present when FCT = 2 (attributes only / multicolor mode).

See Payload Type 1 documentation for common patterns (53c, 127c).

#### Frame Data

Frames are stored sequentially with no padding. Each frame has the following structure:

```
┌──────────────────────────────────────────────────────────┐
│ [CCCCC BBB]  frame header byte                           │
│ [block 1]    bitmap data (if FCT = 0 or FCT = 1)        │
│ [block 2]    attribute data (if FCT = 1 or FCT = 2)     │
│ [delay]      wait time in HALTs (1/50 s)                 │
└──────────────────────────────────────────────────────────┘
```

**Frame header byte** `CCCCC BBB`:

```
  Bits 7–3 (CCCCC): Compressor type
    0 = uncompressed (raw data)
    1 = ZX0
    2 = Laser Compact
    3 = RLE
    4 = Chunks 4×4
    5 = Chunks 4×2
    6–31 = reserved

  Bits 2–0 (BBB): Border color (0–7)
```

**Blocks** are either raw data (when compressor = 0) or compressed data using the compressor specified in the frame header byte. The decompressor knows the output size from the FCT + region combination.

For FCT = 1, bitmap and attributes are always stored as two separate blocks, allowing independent compression of each.

For compressor types 4 and 5 (Chunks), see the **Chunks Compression** section below — the block format differs from stream compressors.

**Delay byte** follows the last block and specifies how many vertical interrupts (HALTs) to wait before processing the next frame.

#### Z80 Player Loop (pseudo-code)

```asm
frame_loop:
    ld a,(hl)        ; frame header byte
    inc hl
    ld c,a
    and 7
    out ($fe),a      ; set border color
    ld a,c
    rrca
    rrca
    rrca
    and %00011111    ; a = compressor type
    ; select decompression routine based on a
    ; decompress block(s) from (hl) to screen memory
    ; hl advances past compressed data
    ld a,(hl)        ; delay byte
    inc hl
    ld b,a
.wait:
    halt
    djnz .wait
    ; loop until frame count exhausted
    jp frame_loop
```

---

### Compressor Types 4, 5 – Chunks Compression

Chunks is a lossy monochrome bitmap compression that divides each 8×8 character cell into sub-chunks and encodes each sub-chunk as a 2-bit index into a fixed 4-pattern dictionary (codebook). The codebook is static and defined by the format — it is **not** stored in the SCA file. The decoder must know the codebook in advance.

Two modes are defined:

| Compressor | Mode | Chunk size | Chunks/cell | Bytes/cell | Encoded size (full screen) |
|------------|------|------------|-------------|------------|----------------------------|
| 4          | 4×4  | 4×4 pixels | 4           | 1          | 768 bytes                  |
| 5          | 4×2  | 4×2 pixels | 8           | 2          | 1536 bytes                 |

#### Constraints

- FCT **must** be 0 (bitmap only). Chunks encodes monochrome bitmap data; attributes are not stored.
- All regions (0–5) are supported. The encoded data size is determined by the number of character rows in the region (8, 16, or 24) × 32 columns.
- All frames must use the same compressor type (all 4×4 or all 4×2). Mixing modes within a single file is not supported.

#### Standard Codebook

The codebook contains 4 patterns indexed 0–3. Each pattern is a pixel bitmap of the chunk, stored MSB-first (leftmost pixel = highest bit).

**4×4 mode** — 16-bit patterns (4 rows × 4 pixels, MSB = top-left):

| Index | Hex      | Visual     | Description  |
|-------|----------|------------|--------------|
| 0     | `0x0000` | `....` `....` `....` `....` | Empty   |
| 1     | `0xFFFF` | `####` `####` `####` `####` | Solid   |
| 2     | `0x5F5F` | `.#.#` `####` `.#.#` `####` | Dither  |
| 3     | `0x080A` | `....` `#...` `....` `#.#.` | Sparse  |

Bit layout: bits 15–12 = row 0, bits 11–8 = row 1, bits 7–4 = row 2, bits 3–0 = row 3. Within each nibble, bit 3 = leftmost pixel, bit 0 = rightmost pixel.

**4×2 mode** — 8-bit patterns (2 rows × 4 pixels):

| Index | Hex    | Visual     | Description  |
|-------|--------|------------|--------------|
| 0     | `0x00` | `....` `....` | Empty   |
| 1     | `0xFF` | `####` `####` | Solid   |
| 2     | `0x5F` | `.#.#` `####` | Dither  |
| 3     | `0x0A` | `....` `#.#.` | Sparse  |

Bit layout: bits 7–4 = row 0, bits 3–0 = row 1.

#### Encoded Byte Format

Each encoded byte packs multiple 2-bit chunk indices, MSB-first:

**4×4 mode** — 1 byte per cell, 4 chunks packed as `[TL TL | TR TR | BL BL | BR BR]`:

```
  Bits 7–6: top-left chunk index     (rows 0–3, cols 0–3)
  Bits 5–4: top-right chunk index    (rows 0–3, cols 4–7)
  Bits 3–2: bottom-left chunk index  (rows 4–7, cols 0–3)
  Bits 1–0: bottom-right chunk index (rows 4–7, cols 4–7)
```

**4×2 mode** — 2 bytes per cell, 8 chunks packed into byte 0 and byte 1:

```
  Byte 0:
    Bits 7–6: chunk at rows 0–1, cols 0–3
    Bits 5–4: chunk at rows 0–1, cols 4–7
    Bits 3–2: chunk at rows 2–3, cols 0–3
    Bits 1–0: chunk at rows 2–3, cols 4–7
  Byte 1:
    Bits 7–6: chunk at rows 4–5, cols 0–3
    Bits 5–4: chunk at rows 4–5, cols 4–7
    Bits 3–2: chunk at rows 6–7, cols 0–3
    Bits 1–0: chunk at rows 6–7, cols 4–7
```

#### Cell Traversal Order

Encoded bytes are stored in screen order: 3 thirds (top, middle, bottom), each consisting of 8 character rows × 32 columns = 256 cells. Within each third, cells are stored left-to-right, top-to-bottom.

For full screen: 3 × 256 = 768 cells → 768 bytes (4×4) or 1536 bytes (4×2). For partial regions, the cell count scales with the number of thirds: 256 cells per third (8 rows × 32 cols).

#### Frame Layout in SCA

Each chunks frame within the type 2 payload:

```
┌─────────────────────────────────────────────────────┐
│ [CCCCC BBB]  frame header byte                       │
│   CCCCC = 4 (chunks 4×4) or 5 (chunks 4×2)         │
│   BBB = border color                                 │
│ [encoded data]  size depends on region and mode      │
│ [delay]  wait time in HALTs (1/50 s)                 │
└─────────────────────────────────────────────────────┘
```

No codebook, no lookup table, no length prefix — just raw encoded bytes. The data size is fixed and known from the compressor type and region:

| Region | Thirds | 4×4 encoded size | 4×2 encoded size |
|--------|--------|-------------------|-------------------|
| 0, 1, 2 | 1    | 256 bytes          | 512 bytes          |
| 3, 4   | 2      | 512 bytes          | 1024 bytes         |
| 5      | 3      | 768 bytes          | 1536 bytes         |

#### Lookup Table (optional optimization)

A depacker can use a precomputed lookup table (LUT) to convert a pair of 2-bit indices into a full 8-pixel screen byte in one step, avoiding per-chunk codebook lookups at runtime. The LUT is derived deterministically from the codebook and is not stored in the SCA file.

**4×4 LUT** — 64 bytes (16 index pairs × 4 rows):

For each pair of adjacent chunks (left index L, right index R), the LUT stores 4 bytes at offset `(L×4 + R) × 4`:

```
  byte 0: (codebook[L] row 0 nibble << 4) | (codebook[R] row 0 nibble)
  byte 1: (codebook[L] row 1 nibble << 4) | (codebook[R] row 1 nibble)
  byte 2: (codebook[L] row 2 nibble << 4) | (codebook[R] row 2 nibble)
  byte 3: (codebook[L] row 3 nibble << 4) | (codebook[R] row 3 nibble)
```

Each LUT byte is a complete 8-pixel screen byte: left chunk's nibble in bits 7–4, right chunk's nibble in bits 3–0.

**4×2 LUT** — 32 bytes (16 index pairs × 2 rows):

Same structure but 2 bytes per entry at offset `(L×4 + R) × 2`.

---
