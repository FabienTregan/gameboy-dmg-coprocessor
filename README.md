# gameboy-dmg-coprocessor
Can we build a Gameboy DMG catridge with a coprocessor ?

## The idea

I wanted to use a gameboy as an usb joypad for PC. Not wanting to modify nor destroy the gameboy, I thought of this solution : Make a gameboy cartrdige that would just send to the gameboy minimal code that would read the cross, select, start, A and B button, and send them back to the cartridge, which would read the value and send them via USB to the computer.

Though this could be donne with a real gameboy cartridge (ROM + RAM) and a microcontroler that would read the ram while the gameboy is not accessing it, it could be done for way cheaper if the mCU emulates the ram and rom.

But if you emulate ram and rom with an mcu, can you emulate a rom containing frame (image on screen) data that would be calculated in real time by a powerfull mcu, and deliver to the gameboy the code that would copy the mcu calculated picture to the gameboy video ram ? Let's see.

## Challenge of transfering the picture from cartridge to video ram at 60 frames per second

The gameboy is able to display 60 fps. It displays 144 rows of 160 pixels. We want to send it 1382400 pixels per second. Each pixel has four possibles colors, hence is coded on two bits, we want to transfert 2764800 bits, or 345600 bytes per second.

The CPU runs at 4.19MHz (it's clock ticks 4.190.000 times per second), and it usually needs 4 or 8 ticks (we all them *cycles*) to do most things. The LCD screen is refresh every 70224 cycles. We want to transfer a new frame in less than 70224 cycles to have maximum fluidity.

### Simple copy from ROM to VRAM

The default way of transfering an array of bytes from rom to ram would be :
1. Loading address of first byte into a 16 bits registers pair (e.g. BC, using register B and C to form a 16 bits register)
1. Loading destination address into `HL` register pair
1. Loading a 16 bits register with the counter (length of array)
1. Loading the content of this address into `A` register
1. Loading content of `A` register at destination adress (this automatically increases HL for next byte)
1. Increasing the first 16 bits register (BC)
1. decreasing the 16 bits counter
1. Repeating to step 3 if counter did not read 0.

This would look like this :
```
  ld  BC, source_address   ; 12 cycles
  ld  HL, vram_start       ; 12 cycles
  ld  DE, length           ; 12 cycles, length is (160*144 pixels) * (2 bits per pixel) / (8 bits per byte) = 5760
loop:
  ld  A, (BC)              ; 8 cycles
  ldi (HL), A              ; 8 cycles, increments HL
  inc BC                   ; 8 cycles
  dec DE                   ; 8 cycles
  jz  nz, loop             ; 12 cycles
```
  The loop takes 44 cycles and copies one byte (8 bits, 4 pixels), i.e. 11 cycles per pixel. At this rate, copying a full frame of 160x144 pixels would take 253440 cycles. The screen is refreshed every 70224 cycles. This is currently 3.6 times to slow (we can refresh only 28% o the screen on each frame).
  
  In reality, this is even less because :
1. While the gameboy is reading the video ram to send data to the LCD screen, the cpu can not access it. We need to be faster. (more on this later)
1. Only addresses from $0000 to $7FFF (for the ROM), $A000 to $BFFF (for the external ram, the one on the cartridge). That means we can only loop over 32768 bytes (32KB) of data, then we have to send the cartridge some command so it switch the current *bank* (chunk of 16k of data which is currently exposed to the processor through the address range $4000-$7FFFF)  And we need to transfert more than 44KB of picture per frame, and the code for this is also in the ROM and needs to be addressable, then at some point we will have to switch the bank.

### Unrolling the loop

In the previous code, 20 cycles out of 44 are taken by decreasing the counter and jumping back to the start of the loop if zero is not reached. There is a common technic known as "unrolling the loop", which basically just mean you copy-past the code by hand instead of having the processor repeat it. The previous code would then look like this :
```
  ld  BC, source_address   ; 12 cycles
  ld  HL, vram_start       ; 12 cycles
  ld  DE, length           ; 12 cycles, length is (160*144 pixels) * (2 bits per pixel) / (8 bits per byte) = 5760
unrolled_loop:
  ld  A, (BC)              ; 8 cycles
  ldi (HL), A              ; 8 cycles, increments HL
  inc BC                   ; 8 cycles
  ld  A, (BC)              ; 8 cycles
  ldi (HL), A              ; 8 cycles, increments HL
  inc BC                   ; 8 cycles
  ld  A, (BC)              ; 8 cycles
  ldi (HL), A              ; 8 cycles, increments HL
  inc BC                   ; 8 cycles
  ld  A, (BC)              ; 8 cycles
  ldi (HL), A              ; 8 cycles, increments HL
  inc BC                   ; 8 cycles
  ld  A, (BC)              ; 8 cycles
  ldi (HL), A              ; 8 cycles, increments HL
  inc BC                   ; 8 cycles
...
```
We now have code that takes only 24 cycles per byte (from 44). It also takes a lot of ROM space. In fact, it takes 3 byte of code to copy 1 byte of data. we do not want to use 3x44KB of code to copy the full frame, but we can unroll it e.g. 64 times, so we would pay the 20 cycles for looping only once every 64 bytes, so the cost would stay very close to 24 cycles per bytes while staying shorter than 256 bytes.

BTW, with 24 cycles for one byte (6 cycles per pixel), we now can transfert a bit less than (70224 cycles / 6 cycles per pixel) = 11700 pixels per frame, enough to cover 50% of the screen at 60fps, or full screen at 30 fps.

### Using the DMA ?

The first idea when thinking about fast transfert is to use dedicated hardware. The gameboy has a Direct emory Access (*DMA*) which can copy 40x28 bits (140 bytes) in about 160 microseconds (640 cycles). That means 4.6 bytes per cycle, or just a bit more than 1 cycle per pixel ! Unfortunately this DMA is only able to access the OAM zone of the gameboy memory (the place that contains information about where to display sprites on screen and if you want them flipped horizontally or vertically)

### Two bytes at a time

The gameboy processor has instruction that allows to transger two bytes at a time. e.g., the load (`ld`) instruction can load a pair of registers if you give it not the address of some source memory, but the direct value you want to put in the register pair.

As we discussed in the first chapter, instructions in our cartridge won't be hardcoded into the ROM, they will be dynamically generated by the microcontroller on the cartridge. Hence we can generate this instructions :
```
unrolled_loop:
  ld BC, <two following bytes of picture>   ; 12 cycles
  ld (HL), B                                ; 8 cycles
  inc HL                                    ; 8 cycles
  ld (HL), C                                ; 8 cycles
  inc HL                                    ; 8 cycles
...
```
We now can read two bytes in B and C in 12 cycles instead o 2x8 cycles. But we loose the power of `ldi` which freely increased HL in previous code (but is only usable with A register, and we can't use the 16 bits transfer with A register) So the total is 44 cycles for two bytes, i.e. 22 cycles per byte, just a tad better than the previous 24.

But there is another 16 bits transfer instruction : `push`. TThis instruction is intended for putting thing on the stack, and it does two things :
* move data at the address of the current stack pointer (`SP`)
* decrease `SP` by two so it points to the next position in the stack

We could then generate with the microcontroller the a `push` instruction that will make the CPU a pair of registers to the end of the video ram and automatically point to the address of the previous pair of bytes we want to transfert. This would look like this :
```
  ld  SP, <adresse of second byte before end of where we need to copy the picture>
unrolled_loop:
  ld BC, <two last bytes of picture>        ; 12 cycles
  push BC                                   ; 16 cycles
  ld BC, <two previous bytes of picture>    ; 12 cycles
  push BC                                   ; 16 cycles
...
```
We now go down to 12+16=28 cycles per byte pair, 14 cycles per byte from 22 ! With 4 pixels per byte, that mean 3.5 cycles per pixel (we started at 6). At 70224 cycles per frame, we can move 20064 pixels per frame, or 87% of the screen.

### More realistic calculation

87% o the screen is nearly eanough : we could just leave 6% of blank lines at the bottom and top of the screen hand be happy with that. But if you remember earlyer we told about time when the VRAM could not be accessed by the CPU because it was used to sget data to be send to the LCD driver. The process goes like this :
* We start in mode 0 (also, called H-BLANK), during which the LCD drivers display a line and already has the data is needs. This last for 201 to 207.
* We follow with mode 2, when the drivers needs to acces OAM ram (the one which contains data about the sprite and that the DMA can write to), during this time (77 to 83 cycles) we still can access VRAM
* Then follows mode 3, when the drivers loads data to draw next line, and we can't acces neither VRAM nor OAM for 169 to 175 cycles
This repeats for the 144 lines,  then inally some free time (called mode 2 or V-BLANK) lasts before the next frame is sent to the LCD for  4560 cycles. We cann access bot VRAM and OAM during V-BLANK

So, with 28 cycles for 2 bytes, during each line we can be sure to have at least 201 (mode 0) plus 77 (mode 2) cycles to access the VRAM. This means (201+77)/28=9 pairs of bytes (very close to 10). We can tranfer 18 bytes per line. With 144 lines that means 2592 bytes or 10368 pixels.

Then during V-BLANK, we have 4560 cycles to transfer 162 pairs of bytes, that's 1296 more pixels.
Total is 10368+1296=11664, that's only 50.6% of the screen. Maybe we can manage to move 19 instead of 18 bytes per lines, adding 4x144 pixels, for 53.1% of the screen.

Of course, with a gameboy color, things would be way better :
-CPU can be run at double speed (106% of the screen)
-The DMA can access the VRAM, and it can transfert nearly 1 pixel per cycle (with twice the number of cycles).
But the GBC is less iconic than the DMG GB. And -I found while looking for technical info for this project- Anders Granlund already made a [coprocessor for the GBC](http://www.happydaze.se/wolf/)

So, can we gain something more ?

### Mixing tiles and sprites

When I first missread the specification of the DMA, I thought it would be able to transfert some image for the sprites. I then thought that using gameboy's sprites unstead of tiles could allow for faster DMA enabled transfer.

The tiles are 8x8 pixels blocks which are positionned on a grid (they all are perfectly aligned on screen, even though the grid itsellf can scroll). Sprites get their images from the same memory array as the tiles, but can be positonned at any abitrary position on the screen, and have transparent pixels that let you see the tiles behind them.

The gameboy can only handle 40 (8x8 pixels) sprites (or 20 of size 8x16), moreover it can only display 10 sprites per lines, covering only 10x80 pixel out of 160 on each line at max. I then thought I would have to mix sprites and tiles and use the DMA transfert for only half of the frame, but that would have been nice anyway.

### Scrolling the tiles

Another thing I tryed to use is the possibility to scroll the tiles. You may remember that data to display the next line is only read by the LCD driver during the *mode 2* (for sprites OAM infos) and *mode 3* (for the actual pictures of tiles and sprites).

This mean you can, e.g., scroll the grid after the driver read data for LCD first line, but before it read data for LCD second line. Imagine the first LCD line started with first line of tile 1. The driver should then start LCD second line with second line of tile 1. Bu if you scrolled one line up, it would then use Tile 1 third line instead. If you scrolled down it would use Tile 1 first line again ! And you can do more : you can change the tile number, so second LCD line can start with any line of any tile existing tile - but you it would need to be the same line number in the tile for a gien LCD line, because you can not scroll the grid between rendering of two successive tiles.

So, why is this of interest to us ? A tile is 8 pixels wide. By changing just the tile number (1 byte) you can change 8x8 pixels. But there is a (huge) drawback : in one byte you only have 256 different possible values. And a tile have 64 pixels with 4 possible colors each, so a total of 4^64= a very big lot of possibilitied. 
