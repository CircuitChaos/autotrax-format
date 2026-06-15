# AutoTrax PCB File Format

This file describes the PCB file format used by Protel AutoTrax 1.61 ND („PCB FILE 4”). It's been reverse-engineered by me (Circuit Chaos).

## Overall format

* PCB file is a text file, with CRLF line endings
* First line is always `PCB FILE 4`
* Last line is always `ENDPCB`

## Attribute types

Some attributes, like layer, rotation, circle quadrants, etc. are specified as numbers. Here are the numbers.

All sizes and coordinates are in mils. If metric system is used, then it's just translated by the program – PCB file is still in mils.

### Rotation

Used in strings.

* 0: 0°
* 1: 90°
* 2: 180°
* 3: 270°

Add 16 to flip on the X axis. Flipping on Y axis is done by flipping on X axis and rotating 180°.

Rotation is counter-clockwise.

### Quadrants

Used in arcs. It's a sum of the following values.

* 1: Third quadrant (upper right)
* 2: Second quadrant (upper left)
* 4: First quadrant (lower left)
* 8: Fourth quadrant (lower right)

0 means no quadrants shown (all are hidden).

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

### User-routed

Used in tracks.

* 0: Not user-routed
* 1: User-routed

## Primitives

AutoTrax supports different primitives – tracks, vias, etc. They're described here. Each primitive spans either two or three lines.

First line is the primitive type. Always two letters, where first is `F` (or `C` in case of primitives being parts of components, see below).

Second line contains numeric parameters, separated by space.

Third line exists only for pads and strings, and contains text (pad name or string text).

### Arc

* Type: `FA`
* Parameters:
  * X coordinate
  * Y coordinate
  * Radius
  * Quadrants
  * Line width
  * Layer

Example:

```
FA
325 1075 25 11 10 6
```

This is:

* Arc (FA)
* Placed at X=325 and Y=1075 mils
* Radius is 25 mils
* Quadrants are all except first (lower left): 8 + 2 + 1 = 11
* Width is 10 mils
* Placed on Bottom Layer (6)

### Fill

* Type: `FF`
* Parameters:
  * Starting X coordinate
  * Starting Y coordinate
  * Ending X coordinate
  * Ending Y coordinate
  * Layer

Example:

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

Example:

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

* Type: `FS`
* Parameters:
  * X coordinate
  * Y coordinate
  * Height
  * Rotation
  * Line width
  * Layer
* Text

Example:

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

* Type: `FT`
* Parameters:
  * Starting X coordinate
  * Starting Y coordinate
  * Ending X coordinate
  * Ending Y coordinate
  * Width
  * Layer
  * User-routed

Example:

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

* Type: `FV`
* Parameters:
  * X coordinate
  * Y coordinate
  * Diameter
  * Hole size

Example:

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

Component is a set of primitives. Component structure:

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

* 1: shown
* 2: hidden

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

* Reverse-engineer the library (.LIB) format

## Contact

If you found an error or want to report something, please use the GitHub reporting system.
