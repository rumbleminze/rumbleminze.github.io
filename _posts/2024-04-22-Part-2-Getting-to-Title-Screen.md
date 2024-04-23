## Part 2: Getting to Title Screen

### Reset and NMI vectors

### Porting Hardware Register Calls

Now that we've got the SNES trying to execute the NES code, the next step is to start porting the NES specific logic to something the SNES will understand.

At a high level, this means translating what the game is trying to accomplish when it writes or reads from the NES hardware registers to something the SNES can do.  The registers don't match up 1:1, so we have to keep that in mind while we adjust the code.

Here's a look at the most common NES registers, how I've seen them be used, and how I adjust their functionality for the SNES.  I've Italicized the functionality that we'll be paying particular attention to.

**PPUCTRL ($2000)** - The PPU Control on the NES handles the following functions:
    1. _Enable/Disable NMI_ 
    2. PPU Master / Slave Select
    3. Sprite Size
    4. Background Pattern Table Address
    5. Sprite Pattern Table Address
    6. _Vram Increment per write_
    7. _Base Nametable address_ (which quadrant of BG is 0,0)
    
    
On the SNES these 3 things are handled by 3 individual registers.

The enable/disable NMI will be used at various times to make sure video work that we're doing isn't interrupted.  For PPUCTRL, and many of the other NES registers, the NES uses a pattern like this:

```
Load the state of the register from memory
Adjust the current value by toggling the appropriate bits
Store the value back in memory
Write the value to the register
```

This ensures that it can toggle various bits on the register without losing other settings.

Most NES games will use one byte in the zero page for this purpose.  Double Dragon uses `$FF`. 

Here's an example from Double Dragon, where it's enabling and storing the NMI:
```
LDA $FF
ORA #$80
STA $FF
STA PpuControl_2000
```

Or'ing the current state of PPUCTRL with `#$80` will ensure that the "enable nmi" is set to true.

For SNES, to enable/disable NMI we adjust the NMITIMEN register.  Just like the NES code, we'll want to keep track of what the value is while we flip bits in it.  Here's my `enabled_nmi_and_store` routine that updates both the PPU Control state varible, as well as enabling NMI:

```
enable_nmi_and_store:
    ; make sure any NMI flags are clear
    LDA RDNMI
    LDA NMITIMEN_STATE
    ORA #$80
    STA NMITIMEN_STATE
    STA NMITIMEN

    LDA PPU_CONTROL_STATE
    ORA #$80
    STA PPU_CONTROL_STATE
    
    RTL
```


### Bank Switching


