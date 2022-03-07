# PMG.BAS

This example sets up a simple approach for initializing and rendering Player Missile graphics as well as handling keyboard and joystick input for vertical/horizontal movement

## Initialization

    100 REM SETUP PMG & MEMORY VARS
    110 DIM PM$(2048)
    120 PM$(1)=CHR$(0):PM$(2048)=CHR$(0):PM$(2)=PM$
    130 DIM CLEAR$(128):CLEAR$(1)=CHR$(0):CLEAR$(128)=CHR$(0):CLEAR$(2)=CLEAR$

PMG requires reading and writing directly to the memory addresses by way of `PEEK` and `POKE`.  The `PM$` byte string is setup to act as the entire PM memory space.  For single line graphics mode, this space is 2048 bytes which comprise the following:

**Single Line Resolution** memory diagram

    +-------------------+ PMBASE
    | Unused            |
    |                   |                 (768 bytes)
    |                   |
    +-------------------+ +768
    | M3 | M2 | M1 | M0 | Missiles        (256 bytes - 64 bytes per each)
    +-------------------+ +1024
    | Player 0          |                 (256 bytes)
    +-------------------+ +1280
    | Player 1          |                 (256 bytes)
    +-------------------+ +1536
    | Player 2          |                 (256 bytes)
    +-------------------+ +1792
    | Player 3          |                 (256 bytes)
    +-------------------+ +2048

To match the above memory layout, `PM$` is dimensioned to 2048 (2K) bytes and filled with zeros `CHR$(0)`. Lastly, a 128 byte clear string is created and filled with zeros.  This will later by used to replace dirty bytes left behind when moving the player vertically.

## Memory Locations

    200 REM MEMORY LOCATIONS
    210 PMGMEM=54279
    220 P0CMEM=704
    230 DMACTL=559
    240 GRACTL=53277
    250 P0XMEM=53248
    260 JOYSTICK=632

To add clarity, I've noted the memory locations involved in using PMG.

- `PMGMEM` - Player Missile Graphics memory area
- `P0CMEM` - Player 0 Color memory area
- `DMACTL` - Dynamic Memory Access Control memory area
- `GRACTL` - Graphics Control memory area
- `P0XMEM` - Player 0 Position X memory area
- `JOYSTICK` - Joystick memory area

## Graphics and PMG Initialization

    1000 REM INITIALIZE GRAPHICS AND PMG
    1010 GR.8
    1020 SETCOLOR 2,0,0
    1030 A=ADR(PM$)
    1040 PMBASE=INT(A/1024)*1024
    1050 IF PMBASE<A THEN PMBASE=PMBASE+1024
    1060 S=PMBASE-A
    1070 P0=S+1024
    1080 POKE PMGMEM,PMBASE/256
    1090 POKE P0CMEM,20:POKE DMACTL,62:POKE GRACTL,3

For this demo the graphics mode has been set to 8 `GR.8` and the background color has been set to black `SETCOLOR 2,0,0` (2 is playfield, 0 is color, last 0 is luminance).

    1030 A=ADR(PM$)

The approach here is to utilize the `PM$` as the actual PM graphics by string.  To do so, we must first retrieve the `PM$` address in memory, later we will poke this into memory so that the Atari will reference this string when reading values to apply to the PMG.

    1040 PMBASE=INT(A/1024)*1024
    1050 IF PMBASE<A THEN PMBASE=PMBASE+1024
    1060 S=PMBASE-A

PMG requires that the memory space be rounded to the nearest full 1K (1024) bytes space.  We take the address of the `PM$`, divide it by 1024 and drop the decimal precision, then multiply it by 1024 to see where we end up.  If the result `PMBASE` is less than the current address of the `PM$` (`A`) then we need to increase the `PMBASE` by an additional 1024 bytes.  Once we've found the correct `PMBASE` address space, we subtract the `PM$` address to determine the actual start of the `PMBASE` (`S`).

    1070 P0=S+1024

Referring back to the **Single Line Resolution** memory diagram, the Player 0 memory is 1K (1024) bytes from `PMBASE` address, we then store the resulting value in `P0`.  If we were using double line resolution, then we would need to change this line to: `1070 P0=S+512`, because the memory size is much smaller for double line resolution.  See the following diagram.  `P0` gives us our offset for the Player 0 memory location in the `PM$`.

**Double Line Resolution** memory diagram

    +-------------------+ PMBASE
    | Unused            |
    |                   |                 (384 bytes)
    |                   |
    +-------------------+ +384
    | M3 | M2 | M1 | M0 | Missiles        (128 bytes - 32 bytes per each)
    +-------------------+ +512
    | Player 0          |                 (128 bytes)
    +-------------------+ +640
    | Player 1          |                 (128 bytes)
    +-------------------+ +768
    | Player 2          |                 (128 bytes)
    +-------------------+ +896
    | Player 3          |                 (128 bytes)
    +-------------------+ +1024

Let's register our `PM$` as the byte array to use with PMG.

    1080 POKE PMGMEM,PMBASE/256

This allows us to use the `PM$` as we've set the `PMBASE` address high byte into the `PMGMEM` memory area.

    1100 POKE P0CMEM,20:POKE DMACTL,62:POKE GRACTL,3

With these series of commands, we have set the color of our Player 0, turned on DMA control with the following modes:

    +-------------------------------------------------+
    + Option |                                | Value |
    +-------------------------------------------------+
    | No playfield                            |   0   |
    | Playfield size:                         |       |
    |    Narrow                               |   1   |
    |    Standard                             |  *2*  |
    |    Wide                                 |   3   |
    | Enable:                                 |       |
    |    Missiles only                        |   4   |
    |    Players only                         |   8   |
    |    Missiles and players                 |  *12* |
    | Resolution:                             |       |
    |    Single line                          |  *16* |
    |    Double line                          |   0   |
    | Enable DMA                              |  *32* |
    +-------------------------------------------------+

    64 = (standard playfield size + missiles and players + single line resolution _+ enable DMA)
    64 = 2 + 12 + 16 + 32

And turned on graphics control so that our players are visible and triggers remain in the pressed state until reset.  See the following GRACTL bit table:

    +---------------------------------+
    | Bit | Function                  |
    +---------------------------------+
    | 7   | Unused                    |
    +---------------------------------+
    | 6   | Unused                    |
    +---------------------------------+
    | 5   | Unused                    |
    +---------------------------------+
    | 4   | Unused                    |
    +---------------------------------+
    | 3   | Unused                    |
    +---------------------------------+
    | 2   | Latch Triggers when =1    |
    +---------------------------------+
    | 1   | Turn on players when =1   |
    +---------------------------------+
    | 0   | turn on missiles when =1  |
    +---------------------------------+

One thing to note, if we were going to enable double line resolution, in addition to the above `DMTCTL` we would also need to set `POKE 53256,1`.  This sets the Player 0 size to be used when reading the bits out of the PMG memory space. By default for our purposes, you can think of this as being set to `POKE 53256,8` due to our Player 0 data being 8 bits per row (or wide).

## Player and Location Initialization

    2000 REM INITIALIZE PLAYER GRAPHICS AND LOCATION
    2010 DIM PLAYER0$(256):PLAYER0$(1)=CHR$(0):PLAYER0$(256)=CHR$(0):PLAYER0$(2)=PLAYER0$
    2020 PPX=120:POKE P0XMEM,PPX
    2030 PPY=50
    2040 FOR I=1 TO 8:READ A:PLAYER0$(I,I)=CHR$(A):NEXT I
    2050 PM$(P0+PPY)=PLAYER0$

We'll need a byte string to hold our Player 0 graphic `PLAYER0$`, this will be used to copy our Player 0 graphics into our `PM$` Player Missile graphics string.  It is initialized filled with zeros (`CHR$(0)`).

    2020 PPX=120:POKE P0XMEM,PPX

The Player 0 X position is then updated setting the initial X to 220 (half the horizontal screen).  This `PPX` value is then set into the `P0XMEM` (Player 0 Horizontal position) memory space.

    2030 PPY=50

A variable containing the Y or vertical position is initialized to 50 (near the top vertically of the screen space).

    2040 FOR I=1 TO 8:READ A:PLAYER0$(I,I)=CHR$(A):NEXT I

Graphic data is read from the `DATA` at line 5010 (Keep in mind BASIC does a first pass scan to load things such as this) and fed into the `PLAYER0$`.

    2050 PM$(P0+PPY)=PLAYER0$

The now populated `PLAYER0$` is rendered into the `PM$` which is referenced by the `PMBASE` (PMG memory).  The `P0` is the string offset or position of the Player 0 data in the `PM$`.  `PPY` is the vertical position.

## The Main Loop

    3000 REM MAIN LOOP
    3010 GOSUB 4010
    3030 GOTO 3010

This loop consists of a subroutine call to line 4010 to determine any updates from the user (keyboard/joystick), then returns to continually loop.

## Player Position Updates

    4000 REM UPDATE PLAYER 0 POSITION
    4010 KEY=PEEK(53769)
    4020 JOY=PEEK(JOYSTICK)
    4030 IF PEEK(753)=0 AND JOY=15 THEN RETURN
    4040 IF KEY=19 OR JOY=11 THEN DX=-2
    4050 IF KEY=20 OR JOY=7 THEN DX=2
    4060 IF KEY=3 OR JOY=14 THEN DY=-2
    4070 IF KEY=4 OR JOY=13 THEN DY=2
    4080 PPX=PPX+DX
    4090 IF DY>0 THEN PM$(P0+PPY,P0+PPY+1)=CLEAR$
    4100 PPY=PPY+DY
    4110 IF DY<>0 THEN PM$(P0+PPY)=PLAYER0$
    4120 DX=0:DY=0
    4130 POKE P0XMEM,PPX
    4140 RETURN

I wanted to enable both keyboard and joystick inputs.  Admittedly there's a better way to handle the keyboard input, but I wanted it to work simply for testing purposes.

    4010 KEY=PEEK(53769)

This reads in the key being pressed.

    JOY=PEEK(JOYSTICK)

This reads in the joystick actions.

    IF PEEK(753)=0 AND JOY=15 THEN RETURN

We want to make sure that if no input is being detected that we don't waste our time and instead just return.

    4040 IF KEY=19 OR JOY=11 THEN DX=-2
    4050 IF KEY=20 OR JOY=7 THEN DX=2
    4060 IF KEY=3 OR JOY=14 THEN DY=-2
    4070 IF KEY=4 OR JOY=13 THEN DY=2

Determine which key is being pressed or which joystick direction, then set the appropriate delta variable `DX` and `DY` respectively where `DX` is horizontal and `DY` is vertical.

    4080 PPX=PPX+DX

Update the `PPX` variable with any changes in horizontal direction

    IF DY>0 THEN PM$(P0+PPY,P0+PPY+1)=CLEAR$

This is a special optimization case where if the direction is down, then we need to update (paint) the current player position with a clear or VBLANK.  Because the vertical motion is 2 pixels per update `DY +or- 2`, we only need to clear the last 2 lines.  If this number were larger, we would need to clear more to possibly the entire rendered player graphic area.

    4100 PPY=PPY+DY

Update the `PPY` variable with any changes in vertical direction

    4110 IF DY<>0 THEN PM$(P0+PPY)=PLAYER0$

If there is any change in vertical direction, we need to repaint the Player 0 after the above described clear/VBLANK.

    4120 DX=0:DY=0

Reset the change in X and Y variables back to 0.

    4130 POKE P0XMEM,PPX

Update the Player 0 PMG memory position.

## Player Graphics

    5000 REM PLAYER 0 GRAPHIC DATA
    5010 DATA 60,126,247,251,255,255,126,60

The player graphics model is relatively simple.  Each rendered row consists of a single byte or 8 bits.  Each bit determines if a pixel is on or off.  In the case of the above `DATA` we are rendering 8 rows of 1 byte of data per row or, and 8x8 bit image.  See the following player graphic bit table:

    Value     Binary      Visual Result
    
      60      00111100        XXXX
     126      01111110       XXXXXX
     247      11110111      XXXX XXX
     251      11111011      XXXXX XX
     255      11111111      XXXXXXXX
     255      11111111      XXXXXXXX
     126      01111110       XXXXXX
      60      00111100        XXXX
