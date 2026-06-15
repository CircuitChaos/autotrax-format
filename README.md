# AutoTrax PCB and LIB File Format

This file describes the PCB and LIB file formats, used by Protel AutoTrax 1.61 ND. Both have been reverse-engineered by me (Circuit Chaos). They're described together, as they use roughly the same data types, just in different formats.

## Primitives

I introduce a concept of "primitives". This is what I call the basic shapes that AutoTrax uses. They're the building blocks of both PCB and LIB files. List of primitives:

* Arc
* Fill
* Pad
* String
* Track
* Via

## Attributes

Some primitives can have various numeric attributes. These are things like layers, rotations, arc quadrants, etc. They're specified as numbers, present in a decimal form in a PCB file, and in a binary form in a LIB file.

This is the list of attributes.

### Rotation

Used in strings.

* 0: 0°
* 1: 90°
* 2: 180°
* 3: 270°

Add 16 to flip on the X axis. Flipping on Y axis is done by flipping on X axis and rotating 180°.

Rotation is counter-clockwise.

### Quadrants

Used in arcs. It's a sum (bit mask) of the following values.

* 1: Third quadrant (upper right)
* 2: Second quadrant (upper left)
* 4: First quadrant (lower left)
* 8: Fourth quadrant (lower right)

0 means no quadrants shown (all are hidden).

Note that math says that "Quadrant I" is the top-right one, etc., but AutoTrax uses its own naming.

### Layer

Used in arcs, fills, pads, strings and tracks.

* 1: Top Layer
* 2: 1 Mid Layer
* 3: 2 Mid Layer
* 4: 3 Mid Layer
* 5: 4 Mid Layer
* 6: Bottom Layer
* 7: Top Overlay
* 8: Bottom Overlay
* 9: Ground Plane
* 10: Power Plane
* 11: Board Layer
* 12: Keep Out Layer
* 13: Multi Layer

### Shape

Used in pads.

* 1: Circular
* 2: Rectangular
* 3: Octagonal
* 4: Rounded Rectangle
* 5: Cross Hair Target
* 6: Moire Target

### Plane

Used in pads.

* 1: No plane connection
* 2: Relief to Ground Plane
* 3: Direct to Ground Plane
* 4: Relief to Power Plane
* 5: Direct to Power Plane
* 6: Tagged to Ground Plane
* 7: Tagged to Power Plane

### User-routed

Used in tracks.

* 0: Not user-routed
* 1: User-routed

A track in a LIB file doesn't have this field.

## PCB file overall format

* This is a text file, with CRLF line endings
* First line is always `PCB FILE 4`
* Last line is always `ENDPCB`
* Primitives and components can occur between first and last line
* Coordinates are in mils, even if the metric system is used by the program (they're just translated)

## LIB file overall format

* This is a binary file
* It is split into three parts (sections):
  * Header
  * Index
  * Data
* Multi-byte values are stored MSB-first (for example, bytes: 0x32 0x00 mean decimal value 50)
* Coordinates are stored as signed 16-bit integers, so they can be negative

### LIB file header

Header is 16 bytes, always 0x00 0x00 0x0d and string: `PCB 4 LIBRARY`.

### LIB file index

Index is a table of 200 structures, each occupying 32 bytes. Each structure represents one element. 
It means the library can't be larger than 200 elements.

Index structure format:

| Offset | Size | Meaning |
| ------ | ---- | ------- |
| 0x00   | 0x02 | Unknown, always 0x00 |
| 0x02   | 0x01 | Component name length in bytes, from 0x01 to 0x0c |
| 0x03   | 0x0c | Component name, right-padded with 0x00 |
| 0x0f   | 0x01 | Unknown, always 0x00 |
| 0x10   | 0x02 | Unknown, always 0x00 |
| 0x12   | 0x02 | Number of primitives in a component |
| 0x14   | 0x02 | Offset in file of the first primitive, divided by 16; for example, 0x91 0x01 translates to offset 0x1910 |
| 0x16   | 0x0a | Unknown, always 0x00 |

In an empty table entry:

* Number of primitives is 0xff 0xff
* Offset in file is 0xff 0xff

In a deleted table entry:

* Offset in file is 0xff 0xff
* Everything else is as normal

If an entry is deleted, and a new entry is added, this new entry will take place of the deleted entry.

Compacting the library does not erase anything from the index table.

### LIB file data

Data immediately follows the index. It contains one or more primitives. Each primitive is either 
16 or 32 byte long.

In case of a deleted entry, data stays in this section and new data is appended to the end of the file. 
Compacting the library fixes this and deletes old data (data belonging to deleted components).

Primitives correspond to primitives used in a PCB file, they're just stored in a binary form.

Coordinates are stored as signed little-endian 16-bit integers, in mils, relative to the reference point. 
Simple example of how to calculate: 0xfb 0xff -> 0xfffb -> 65531; 65536 - 65531 = -5. This is -5.

## Formats of primitives

AutoTrax supports different primitives – tracks, vias, etc. They're described below, and each spans either:

* Two lines in a PCB file and 16 bytes in a LIB file
* Three lines in a PCB file and 32 bytes in a LIB file

In a PCB file, first line is the primitive type. Always two letters, where first is `F` (or `C` in case of primitives being parts of components, see below).

Second line contains numeric parameters, separated by spaces.

Third line exists only for pads and strings, and contains text (pad name or string text).

### Arc

Size: 2 lines (PCB), 16 bytes (LIB)

Format in a PCB file:

* Type: `FA`
* Parameters:
  * X coordinate
  * Y coordinate
  * Radius
  * Quadrants
  * Line width
  * Layer

Format in a LIB file:

| Offset | Size | Meaning |
| ------ | ---- | ------- |
| 0x00   | 0x01 | Layer |
| 0x01   | 0x01 | Primitive type, always 0x01 |
| 0x02   | 0x02 | X position |
| 0x04   | 0x02 | Y position |
| 0x06   | 0x02 | Radius  |
| 0x08   | 0x01 | Quadrants |
| 0x09   | 0x01 | Line width |
| 0x0a   | 0x06 | Zeroes |

Example in a PCB file:

```
FA
325 1075 25 11 10 6
```

This is:

* Arc (FA)
* Placed at X=325 and Y=1075 mils
* Radius is 25 mils
* Quadrants are all visible except the first (lower left, value 4) one: 8 + 2 + 1 = 11
* Width is 10 mils
* Placed on Bottom Layer (6)

### Fill

Size: 2 lines (PCB), 16 bytes (LIB)

Format in a PCB file:

* Type: `FF`
* Parameters:
  * Starting X coordinate
  * Starting Y coordinate
  * Ending X coordinate
  * Ending Y coordinate
  * Layer

Format in a LIB file:

| Offset | Size | Meaning |
| ------ | ---- | ------- |
| 0x00   | 0x01 | Layer |
| 0x01   | 0x01 | Primitive type, always 0x02 |
| 0x02   | 0x02 | Starting X |
| 0x04   | 0x02 | Starting Y |
| 0x06   | 0x02 | Ending X |
| 0x08   | 0x02 | Ending Y |
| 0x0a   | 0x06 | Zeroes |

Example in a PCB file:

```
FF
200 1200 250 1275 1
```

This is:

* Fill (FF)
* Starting at X=200 and Y=1200 mils
* Ending at X=250 and Y=1275 mils
* Placed on Top Layer (1)

### Pad

Size: 3 lines (PCB), 32 bytes (LIB)

Format in a PCB file:

* Type: `FP`
* Parameters:
  * X coordinate
  * Y coordinate
  * X size
  * Y size
  * Shape
  * Hole size
  * Plane
  * Layer
* Name

Format in a LIB file:

| Offset | Size | Meaning |
| ------ | ---- | ------- |
| 0x00   | 0x01 | Layer (only 0x01, 0x06, 0x0d) |
| 0x01   | 0x01 | Primitive type, always 0x03 |
| 0x02   | 0x02 | X position |
| 0x04   | 0x02 | Y position |
| 0x06   | 0x02 | X size |
| 0x08   | 0x02 | Y size |
| 0x0a   | 0x02 | Hole size |
| 0x0c   | 0x01 | Shape |
| 0x0d   | 0x01 | Plane connection |
| 0x0e   | 0x02 | Zeroes |
| 0x10   | 0x01 | Zero |
| 0x11   | 0x01 | Primitive continuation type, always 0x04 |
| 0x12   | 0x01 | Designator size, 0...4 |
| 0x13   | 0x04 | Designator, zero-padded to the right |
| 0x17   | 0x09 | Zeroes |

Example in a PCB file:

```
FP
400 1400 50 70 2 30 1 13
TEST
```

This is:

* Pad (FP)
* Placed at X=400 and Y=1400 mils
* X size is 50 mils
* Y size is 70 mils
* Shape is rectangular (2)
* Hole size is 30 mils
* There's no plane connection (1)
* The pad is multi-layer (13)

### String

Size: 3 lines (PCB), 32 bytes (LIB)

Format in a PCB file:

* Type: `FS`
* Parameters:
  * X coordinate
  * Y coordinate
  * Height
  * Rotation
  * Line width
  * Layer
* Text

Format in a LIB file:

| Offset | Size | Meaning |
| ------ | ---- | ------- |
| 0x00   | 0x01 | Layer |
| 0x01   | 0x01 | Primitive type, always 0x06 |
| 0x02   | 0x02 | X position |
| 0x04   | 0x02 | Y position |
| 0x06   | 0x02 | Height |
| 0x08   | 0x01 | Line width |
| 0x09   | 0x01 | Rotation |
| 0x0a   | 0x06 | Zeroes |
| 0x10   | 0x01 | Zero |
| 0x11   | 0x01 | Primitive continuation type, always 0x07 |
| 0x12   | 0x01 | String length, max 0x0c |
| 0x13   | 0x0c | String, right-padded with zeroes, max 0x0c bytes |
| 0x1f   | 0x01 | Zero |

Interesting finding – a string in a PCB file can be longer than 0x0c, but when saving to a library, it's truncated.

Example in a PCB file:

```
FS
400 1175 48 17 12 1
TEST
```

This is:

* String (FS)
* Placed at X=400 and Y=1175 mils
* Height is 48 mils
* It's rotated 90° CCW and flipped: 1 + 16 = 17
* Line width is 12 mils
* String is placed on Top Layer (1)

### Track

Size: 2 lines (PCB), 16 bytes (LIB)

Format in a PCB file:

* Type: `FT`
* Parameters:
  * Starting X coordinate
  * Starting Y coordinate
  * Ending X coordinate
  * Ending Y coordinate
  * Width
  * Layer
  * User-routed

Format in a LIB file:

| Offset | Size | Meaning |
| ------ | ---- | ------- |
| 0x00   | 0x01 | Layer |
| 0x01   | 0x01 | Primitive type, always 0x05 |
| 0x02   | 0x02 | Starting X |
| 0x04   | 0x02 | Starting Y |
| 0x06   | 0x02 | Ending X |
| 0x08   | 0x02 | Ending Y |
| 0x0a   | 0x01 | Width |
| 0x0b   | 0x05 | Zeroes |

Example in a PCB file:

```
FT
400 800 500 800 25 5 1
```

This is:

* Track (FT)
* Starting at X=400 and Y=800 mils
* Ending at X=500 and Y=800 mils
* Width is 25 mils
* Layer is 4 Mid Layer (5)
* It's user-routed (1)

### Via

Size: 2 lines (PCB), 16 bytes (LIB)

Format in a PCB file:

* Type: `FV`
* Parameters:
  * X coordinate
  * Y coordinate
  * Diameter
  * Hole size

Format in a LIB file:

| Offset | Size | Meaning |
| ------ | ---- | ------- |
| 0x00   | 0x01 | Zero |
| 0x01   | 0x01 | Primitive type, always 0x08 |
| 0x02   | 0x02 | X position |
| 0x04   | 0x02 | Y position |
| 0x06   | 0x01 | Size |
| 0x07   | 0x01 | Hole size |
| 0x08   | 0x01 | Unknown, always 0x01 |
| 0x09   | 0x07 | Zeroes |

Example in a PCB file:

```
FV
600 800 62 28
```

This is:

* Via (FV)
* Placed at X=600 and Y=800 mils
* Via size is 62 mils
* Hole size is 28 mils

## Components

Component (it can exist only in a PCB file) is a set of primitives. Component structure:

* First line: `COMP`
* Designator (text)
* Pattern (text)
* Comment (text)
* Comment string data (as in a string), starting with a space
* Designator string data (as in a string), starting with a space
* Line with parameters:
  * X coordinate
  * Y coordinate
  * Designator status
  * Comment status
  * Placement status
* Zero or more primitives
* Last line: `ENDCOMP`

Designator or comment status values:

* 1: Shown
* 2: Hidden

Placement status values:

* 0: Free to move
* 1: Locked in place

Primitives in a component have first letter changed from `F` to `C`, so track (normally `FT`) becomes `CT`, and so on.

Example:

```
COMP
C1
RB.1/.2
470uF
 940 930 60 0 12 7
 940 930 60 0 12 7
1000 800 2 2 2
CA
1050 800 100 15 10 7
CP
1000 800 62 62 2 28 1 13
+
CP
1100 800 62 62 1 28 1 13
-
ENDCOMP
```

This is:

* Component (COMP)
* Pattern is RB.1/.2
* Comment is 470uF
* Comment string is placed at X=940 and Y=930 mils, text height is 60 mils, not rotated, line width is 12 mils, placed on Top Overlay layer
* Designator string is placed at X=940 and Y=930 mils, text height is 60 mils, not rotated, line width is 12 mils, placed on Top Overlay layer
* Component is placed at X=1000 and Y=800 mils
* Designator is hidden (2)
* Comment is hidden (2)
* Component is locked in place (2)
* Component consists of one arc (`CA`) primitive and two pads (`CP`), one marked `+` and one marked `-`.

## TODO

* Reverse-engineer all values marked `unknown` in a LIB file – can they have different values? When and why?

## Contact

If you found an error or want to report something, please use the GitHub reporting system.
