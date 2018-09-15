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

### Faster transfert techniques

The first idea when thinking about fast transfert is to use dedicated hardware. The gameboy has a Direct emory Access (*DMA*) which can copy 40x28 bits (140 bytes) in about 160 microseconds (640 cycles). That means 4.6 bytes per cycle, or just a bit more than 1 cycle per pixel ! Unfortunately this DMA is only able to access the OAM zone of the gameboy memory (the place that contains information about where to display sprites on screen and if you want them flipped horizontally or vertically)

The gameboy processor has instruction that allows to transger two bytes at a time. e.g., the load (`ld`) instruction can load a pair of registers if you give it not the address of some source memory, but the direct value you want to put in the register pair.
But as you we discussed in the first chapter, the instruction in our cartridge won't be hardcoded into the ROM, they will be dynamically generated by the microcontroller on the cartridge. Hence we can generate this instruction :
```
unrolled_loop:
  ld BC, <two following bytes of picture>   ; 12 cycles
  ld (HL), B                                ; 8 cycles
  inc DL                                    ; 8 cycles
  ld (HL), C                                ; 8 cycles
  inc DL                                    ; 8 cycles
...
```
We now can read two bytes in B and C in 12 cycles instead o 2x8 cycles. But we loose the powere of `ldi` which freely increased HL in previous code (but is only usable with A register, and we can't use the 16 bits transfer with A register) So the total is 44 cycles for two bytes, i.e. 22 cycles per byte, just a tad better than the previous 24.

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
