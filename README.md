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
not_loop:
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

BTW, with 24 cycles for one byte (6 cycles per pixel), we now can transfert a bit less than (70224 cycles / 6 cycles per pixel) = 11700 pixels per frame, enough to cover 50% of the screen at 60frp, or full screen at 30 fps.
