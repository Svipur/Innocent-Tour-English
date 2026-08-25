# Info.

An English translation of the PC-98 and IBM PC DOS versions of the game **Innocent Tour** by Longshot.

![Screenie](Manual/images/screenie.png?raw=true)

# How to install.

The required downloads can be found on the [Releases](https://github.com/Svipur/Innocent-Tour-English/releases) page. The installation process is covered thereon as well.

# Game data.

The game data is packed in pairs of FSP + IDF files.

Each FSP file is just a concatenation of the resource files for the respective floppy disk.

The same-named IDF file is a table of file names and starting offsets in that FSP file. Each IDF entry is: 8 bytes for the file name, 3 bytes for the extension, 4 bytes for the offset (little-endian). The file ends on 11 more 0x00s and then 4 bytes indicating the total length of the FSP file.

# ASH format specifications.

The game uses a custom format for its graphics. It's a combination of LZ-based compression with Move-to-Front (MTF)
coding.

Images are decoded in vertical strips of 16 pixels wide (left to right, the last strip usually being 8 pixels wide), with each strip processed top to bottom. Within each strip, pixel data is split into 4-pixel-wide half-columns, with each half-column storing 4 nibbles (one per bitplane).

The structure of an ASH image file is this:

|Offset|Length|Description|
|---|---|---|
|0x00|8|Header|
|0x08|24|Palette|
|0x20|4|Screen position|
|0x24|4|Image dimensions|
|0x28|1|LZ table entry count (N)|
|0x29|Nx3|LZ table entries (3 bytes each)|
|Varies||Compressed pixel data

## Header.

|Offset|Length|Description|
|---|---|---|
|0x00|4|Magic bytes: `00 04 01 00`|
|0x04|4|Flags/color count: `10 00 00 00` (16 colors)|

## Palette.

The palette starts at offset 0x08 and ends at offset 0x1F, thus encoding 16 colors in 24 bytes. Each byte is split into two nibbles, and the 48 resulting nibbles are grouped into 16 triplets of 3 nibbles each, in **Blue, Red, Green** channel order.

Each nibble represents a 4-bit color intensity. To convert to 8-bit color, multiply by 17 (or just duplicate the nibble).

## Screen position and size.

Little-endian.

|Offset|Length|Description|
|---|---|---|
|0x20|2|X position on screen|
|0x22|2|Y position on screen|
|0x24|2|Width in pixels (must be a multiple of 8)|
|0x26|2|Height in pixels|

The screen position is often set to `(0, 0)` instead of being hardcoded into the image, and the game uses other means to position the image on the screen (often hardcoding it into the script).

## Distance table.

|Offset|Length|Description|
|---|---|---|
|0x28|1|Entry count N (number of 3-byte groups)|
|0x29|Nx3|Packed 12-bit distance descriptor pairs|

Each 3-byte group encodes **two** distance table entries. E.g. given three consecutive bytes `B0 B1 B2`, the first entry is the upper 12 bits of `B0:B1`, and the second entry is the lower 12 bits of `B1:B2`.

### Parsing entries.

```
si = 0x29
for _ in range(count):
	# First entry.
	word = (data[si] << 8)|data[si+1]
	word >>= 4
	row = word & 0xFF # Sign-extended if >= 128.
	col = (word >> 8) & 0xFF
	table.append(col * 1600 - row * 4)
	si += 1

	# Second entry.
	word = (data[si] << 8)|data[si+1]
	col = (word >> 8) & 0x0F
	row = word & 0xFF # Sign-extended if >= 128.
	table.append(col * 1600 - row * 4)
	si += 2
```

So each entry encodes a byte offset that is subtracted from the current write position to find the source for an LZ copy operation. It's encoded as: `col * 1600 - row * 4`, where row and col are extracted from nibble pairs. The constant 1600 (0x640) is the vertical stride in the circular buffer (basically, it's `4px * 400`, the fixed size of a region, to make it easier to go back by a specific number of regions).

Typical entries encode offsets such as:

- `col=1, row=0` > 1600 (previous region, same row).
- `row=-1, col=0` > 4 (previous row, same region).
- `row=0, col=3` > 4800 (three regions back).

## Compressed pixel data.

The pixel data is a bitstream encoding 4-pixel groups in planar format. Bits are read from each byte starting with the most significant bit.

The image is processed in vertical strips of 16 pixels wide from left to right. The last strip can be 8 pixels wide if the image width is not a multiple of 16.

Within each strip, decoding is done in a series of **regions**. Each 16-pixel-wide strip contains 4 regions, with each region being 4 pixels wide * height pixels tall.

Regions are decoded into a 12800-byte (0x3200) circular buffer. The buffer base alternates between 0x0000 and 0x1900 for each strip (giving us two 'banks'). Within each region, the write pointer starts at the bank base plus the region's offset and advances by 4 bytes per row.

Region Buffer Offsets (relative to base):
= Region 0: +0x0000
- Region 1: +0x0640
- Region 2: +0x0C80
- Region 3: +0x12C0

The circular buffer wraps at 0x3200.

### Region encoding.

Each row within a region produces 4 bytes in the circular buffer. This yields one byte per bitplane, with each byte holding a 4-bit nibble representing 4 pixels for that plane.

For each row, the bitstream begins with a control bit. If the control bit is 0, the 4 pixels are decoded individually via MTF (literal row). If the control bit is 1, it's an LZ back-reference (copy from earlier in the circular buffer).

### Literal encoding.

In literal rows, the four consecutive pixels are encoded using the MTF transform. Each pixel produces a 4-bit color index (0–15).

Pixels are packed into 4 plane bytes using bit interleaving:

- plane0 = (px0.bit0 << 3)|(px1.bit0 << 2)|(px2.bit0 << 1)|px3.bit0
- plane1 = (px0.bit1 << 3)|(px1.bit1 << 2)|(px2.bit1 << 1)|px3.bit1
- plane2 = (px0.bit2 << 3)|(px1.bit2 << 2)|(px2.bit2 << 1)|px3.bit2
- plane3 = (px0.bit3 << 3)|(px1.bit3 << 2)|(px2.bit3 << 1)|px3.bit3

The MTF encoder uses 17 tables of 16 entries each. Table selection is based on the previously decoded literal pixel value (0-15), with table 16 used at the start of each region. LZ back-references do not update the previously decoded pixel value state.

Each table N is just numbers 0-15 rotated so that N comes first.

The position within the table is encoded with a variable-length code:

|Bitstream Pattern|Position|
|---|---|
|`0`|0 (repeat last color)|
|`10`|1|
|`110`|2|
|`1110`|3|
|`11110` + `0`|4|
|`11110` + `1`|5|
|`111110` + `0`|6|
|`111110` + `1`|7|
|`1111110` + `00`|8|
|`1111110` + `01`|9|
|`1111110` + `10`|10|
|`1111110` + `11`|11|
|`1111111` + `00`|12|
|`1111111` + `01`|13|
|`1111111` + `10`|14|
|`1111111` + `11`|15|

After decoding, the value is moved to position 0 via sequential swaps.

E.g. suppose our last pixel was colour 1 and the bitstream for the next pixel is `1110`.

We take the table 1: `[1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,0]` and use the colour at position 3 (i.e. colour 4), bringing it to the front. The resulting table is `[4,1,2,3,5,6,7,8,9,10,11,12,13,14,15,0]`, and the next pixel will use table 4.

All 17 tables accumulate MTF swaps across the whole image.

### LZ reference encoding.

When the control bit is 1, a distance index and copy length are decoded, and data is copied from a previous position in the circular buffer. Each unit of length corresponds to 4 bytes (one row of one region, 4 planes).

The distance index selects an entry from the distance table:

|Prefix|Extra Bits|Index Range|
|---|---|---|
|`0`|3|0–7|
|`10`|4|8–23|
|`11`|6|24–87|

The distance index (0–87) is a lookup key into the distance table, not the offset itself. If the decoded distance index is greater than or equal to the number of entries in the distance table, the decoder defaults to a byte offset of 4 (effectively copying the previous column).

Given the distance at a specific distance index, the source position is calculated as: `(current_position - distance) mod 0x3200`.

The copy is performed one row (4 bytes) at a time, advancing both the source and destination pointers together after each row. Because they advance together, bytes written earlier in the same copy operation can become the source for later rows within it.

The copy length specifies how many 4-byte groups (rows) to copy:

|Prefix|Extra Bits|Length Range|
|---|---|---|
|`0`|1|2–3|
|`10`|2|4–7|
|`110`|3|8–15|
|`1110`|4|16–31|
|`11110`|6|32–94|
|`11110` + `111111` (escape)|8 + 1|95–606|

Note on the last entry. When the 6-bit value in the `11110` tier equals 63 (i.e. the preliminary length is 95 or greater), 9 additional bits are read (8 bits then 1 bit, combined as `(byte << 1) | bit`), and the final length is 95 + 9-bit value.

If a decoded length would advance past the region's row limit (the image height), the remaining rows requested by the code are discarded and decoding proceeds to the next region.

## Output reconstruction.

After all regions for a strip are decoded into the circular buffer, the pixel colors are reconstructed by recombining the bitplane data.

Regions 0 and 1 make the left 8 pixels of the strip. Regions 2 and 3 make the right 8 pixels. The first region in each pair supplies the first 4 pixels (the high nibble), and the second region supplies the other 4 pixels (low nibble). The 4 bytes for a row are written to the circular buffer in order: plane 0, plane 1, plane 2, plane 3.

Suppose we've decoded the first two regions for a row as `1010` and `0110`. Thus, the combined byte for plane 0 contains for this row: pixel0=1, pixel1=0, pixel2=1, pixel3=0, pixel4=0, pixel5=1, pixel6=1, pixel7=0.

We do this for the other planes. And get e.g.:
plane0 = `1010 0110`
plane1 = `0011 0101`
plane2 = `1100 1001`
plane3 = `0001 1110`

Reading from the bottom, we get the index for each pixel's colour: `0101` for p0 (index 5), `0100` for p1 (index 4), and so on.