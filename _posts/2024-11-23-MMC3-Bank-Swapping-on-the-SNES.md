---
layout: post
---
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

![mmc3spriteswapexample.gif]({{site.baseurl}}/_posts/mmc3spriteswapexample.gif)

You can see as the last enemey is defeated, the NES swaps out the 2nd half of the sprite tiles with the tiles needed to show the "go this way" sprites, then as the next enemy is appearing, it loads that enemy's sprite tiles.

This is actually how Double Dragon II uses this feature for the entire game.  The player sprite tiles are _always_ in the first half of the sprite tile memory, and whatever enemy you're fighting is in the 2nd half (or other miscellaneous sprites like the hand or spikes).

Additionally, when the game loads BG Tiles into the four slots it has allocated, it does so in much the same way as the PRG Rom loading that we saw above:

```
  STX $049C
  INX
  STX $049D
  INX
  STX $049E
  INX
  STX $049F
  JSR $FEB4 ; load the 4 banks for BG tiles from 049C - 049F
```

So, joyfully, we do not have to worry about handling the 1KB bank switches, as we're always loading the full 4 KB.

### Approximating Sprite Bank Swapping in the SNES

Now that we know how the NES game does it, how are we going to approximate this behavior on the SNES?  We don't have the ability to instantly swap out VRAM and CHROM.

First, let's take a look at how the SNES OAM utilizes Video RAM.  The SNES has 32 KB of VRAM.  Let's just break these up into 8 4KB "pages":

```
0 - 0000 - 1FFF
1 - 2000 - 3FFF
2 - 4000 - 5FFF
3 - 6000 - 7FFF
4 - 8000 - 9FFF
5 - A000 - BFFF
6 - C000 - DFFF
7 - EFFF - FFFF
```

We need _at least_ one of the pages for BG tiles.  It seems like that will be enough, as the background tiles don't change in the middle of the level.  We'll use slot 1 for that.  We also need 1 KB for two screens worth of Background.  We'll use a full 2 KB just to have some spare (and because pretty much everything has to be aligned with one of these 7 boundries anyway).

That leaves us with:
```
0 - 0000 - 1FFF
1 - 2000 - 3FFF - BG Tiles
2 - 4000 - 5FFF - BG1
3 - 6000 - 7FFF
4 - 8000 - 9FFF
5 - A000 - BFFF
6 - C000 - DFFF
7 - EFFF - FFFF
```

Sprite (OAM) tiles can only start on any 16 KB boundry, meaning 0, 2, 4, or 6 in our table.  Additionally, Sprite Tiles are split up into two "name tables".  Each sprite has an attribute that defines if it should load from the first 8KB table or the second 8KB table.  Additionally, we can define the offset of the 2nd table as being `2000`, `4000` `6000`, or `8000` byte offset.  Note that this will wrap around from 7 to 0 also.  So, we can starte our sprite tils at slot 4, and go from there:

```
0 - 0000 - 1FFF - Sprite Nametbale 1 offset 11
1 - 2000 - 3FFF - BG Tiles
2 - 4000 - 5FFF - BG1
3 - 6000 - 7FFF
4 - 8000 - 9FFF - Sprite Nametable 0
5 - A000 - BFFF - Sprite Nametbale 1 offset 00
6 - C000 - DFFF - Sprite Nametbale 1 offset 01
7 - EFFF - FFFF - Sprite Nametbale 1 offset 10
```

Because each of the Double Dragon CHROM banks that was being swapped was 2 KB of 2bpp NES tiles (which is 4 KB of 4bpp SNES tiles), we can fit 2 of them in each slot.  Since the two that we'll need the most are the player sprites and the between fight sprites, we'll load those two into slot 4.  We'll then fill the rest with some of the enemy sprites for now.  Later on we'll optimize this, but the idea would be to load the sprites needed for a level inbetween level loads, when the screen is off anyway.  Once we've loaded them, our VRAM will look like this:

```
0 - 0000 - 1FFF - Abore / Barnov sprites
1 - 2000 - 3FFF - BG Tiles
2 - 4000 - 5FFF - BG1
3 - 6000 - 7FFF
4 - 8000 - 9FFF - Player / between fight sprites
5 - A000 - BFFF - Roper / Linda sprites
6 - C000 - DFFF - William / Right Hand Sprites
7 - EFFF - FFFF - Chin / Abobo Sprites
```

We would initialize our OAM OBJ with:

```
lda %00000010
sta OBSEL
```

This sets the start of sprite tiles to 8000 (4000 WORD address), with the 2nd page at A000.  

This would result in this OAM Layout:

![oamtilelayout1.png]({{site.baseurl}}/_posts/oamtilelayout1.png)

Then, let's say the game progressed, and needed to load William's sprites.

we'd switch the offset of the 2nd page to `01` so that `C000-DFFF` were available:

```
lda %00001010
sta OBSEL
```

now our OAM2 is updated:
![oamtilelayout2.png]({{site.baseurl}}/_posts/oamtilelayout2.png)

And our sprite tiles for William are available!

### Still to do

This doesn't solve _everything_.  We'll still need to add some logic when we convert our NES Object Data into our SNES format.  We already need to do some conversions there, but essentially, what we'll need to do is some logic like this:

```
; is this a player sprite?  if so do not adjust it, it is always in the same spot as the NES game

; non-player sprite handling
   ; what is the active enemy?
   ; is that enemy sprite sheet 
   		; the 2nd page of OAM1 ? use value unmodified
        ; the 1st half of OAM2 ? subtract $80 from the tile address, use the 2nd nametable
        ; the 2nd half of OAM2 ? do not modify the tile address, use the 2nd nametable
```

We'll keep track of what the active enemy sprite sheet is as we load different enemy sprite tables, making this logic as fast as possible (since it'll need to happen _every frame_)


## Conclusion

While this isn't a complete covering of everything I need to do for Double Dragon II, it does solve a lot of the big concerns with the port being achievable.  It means that I can keep 10 different "banks" of sprite tiles available for any one section.  But it's possible there's a section of the game that uses more than 10.  If so I'll need to swap them out at an inoptimal time, possibly causing a pause or flickering.  

Additionaly, we haven't touched on the usage of IRQ yet to draw the HUD, which is a different approach than the first Double Dragon, which used Sprite 0 detection.
