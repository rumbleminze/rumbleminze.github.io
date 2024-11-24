## MMC3 CHROM Bank Swapping Approximation on the SNES

In this post, I'm going to describe some currently in progress work that I'm doing while I attempt to port the NES game Double Dragon II: The Revenge to the SNES.  

### MMC3 Bank Swap Feature

Double Dragon II presents some additional challenges from previous ports that I've done.  It uses a more advanced mapper than previous ports, dubbed the MMC3.  The MMC3 provides several additional enhancements for NES games.  The two biggest ones, that are going to affect our Double Dragon II port are:

1. Ability to Swap out memory in smaller "banks", both in Program area and the PPU CHROM data
2. The ability to generate an interrupt on a specific scanline.  We'll talk about this one later.

For the program bank swapping, rather than fixing the last 16 KB of the 32 KB total space, and swapping the other 16KB, the MMC3 breaks up the PRG ROM into four 8KB sections:

1. `$8000-$9FFF`
2. `$A000-$BFFF`
3. `$C000-$DFFF`
4. `$E000-$FFFF`

The 2nd bank is always swappable with any other 8KB PRG ROM bank, and the 4th bank is always fixed to the last 8 KB Bank.  Then you can configure the game to fix **either** the 1st or 3rd bank to the 2nd to last 8KB of PRG ROM.  Double Dragon uses this capability in a very convienent way, it fixes banks 3 & 4 to the last 16 KB of PRG ROM, and whenever it loads a bank to the 2nd bank, it **always** immediately loads the previous bank to the 1st bank.

Here's the code it's using to do so:

```
  LDA #$87
  STA $00EB
  STA $8000     ; Storing 87 here tells the NES we're selecting which 8KB to 
  				; put into A000-BFFF, the next write to $8001 will actually switch it
  TSX			
  LDA $0102,X   ; load the "bank" we want to switch to into X 
  ASL A         ; Double it.  Why we do this will become apparent soon
  ORA #$01      ; OR it with 1, effectively adding 1 to it
  STA $8001     ; Store it to $8001, swapping that 16KB into A000-BFFF
  LDA #$86
  STA $00EB
  STA $8000     ; Storing 86 to $8000 tells the NES we're going to select which 8KB
                ; to put into the $8000-9FFF range.  the next write to $8000 will switch it
  LDA $0102,X   ; Load the same "bank" value we loaded before
  ASL A         ; double it again, this time do NOT OR it with 1
  STA $8001     ; Store it to $8001, swapping that 16 KB into $8000-9FFF
```

So hopefully it's apparent that doing things in this way, the game is actually handling the PRG banks in the same exact way that the MMC1 does!  If we treat the value it loads from `$0102,X` as MMC1 style PRG bank swap values, then Double Dragon II just loads (2x) and (2x+1) every time.  

This is incredibly fortunate, as it means we can keep our usual PRG setup that we use for all our MMC1 games for Double Dragon II!

### How Double Dragon II uses CHROM Swapping

We are not so lucky when it comes to CHROM swapping, but we are at least a little bit lucky.

Like the PRG ROM swapping, the MMC3 allows for much more granular tile swapping than the MMC1.

If we think about what the NES can have loaded, we have 8 KB of Tile RAM, which is broken up into two halves, with 4 KB used for sprite tiles and 4 KB used for background tiles.

The MMC3 allows for one of these halves to be swapped in four 1 KB banks, and the other half to be swapped in two 2 KB banks.  For Double Dragon, the background tiles are the four 1KB banks, and the sprite tiles are the two 2 KB banks.  You can see how it leverages this here:



### Approximating it in the SNES



