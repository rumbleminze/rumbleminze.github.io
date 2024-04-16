---
layout: post
title: Porting NES Games to SNES Part 0
---

This blog is going to go over the process I use to port NES games to the Super Nintendo.  Something to keep in mind while we go through this process:  I'm by no means an expert on all these things.  But I do know what's worked for me.  If you have ideas on ways to improve it, I'd love to hear them!  Please reach out to me on Twitter.

## The Software Tools I use
* [cc65](https://www.cc65.org/) - the 65c816 compiler and linker
* [go](https://go.dev/) - For various tools that I write for processing the roms
* [Mesen2](https://github.com/SourMesen/Mesen2) - A fantastic emulator for development on a bunch of platforms.  It provides a great debugger that allows for things like "step-backwards"
* [bsnes-plus](https://github.com/devinacker/bsnes-plus) - Another emulator, touted for it's accuracy, used for final testing of the build
* [HxD](https://mh-nexus.de/en/hxd/) - A Hex Editor
* [Visual Studio Code](https://code.visualstudio.com/) - Used as the development environment
* Windows Calculator - useful for quick hex/binary math/conversions
* Git/GitBash - general source control.

## Why though?

The most frequent question I get is "why?".  There are a lot of great reasons for porting games from the NES to the SNES.  Here's a few:

1. eliminate slowdown
2. eliminate sprite flicker
3. allow for MSU-1 enhancements like CD quality music and videos
4. Take advantage of FastROM cartridge chips
5. Up-rez-ing the sprites to 16-bit
6. better video output on SNES instead of NES
7. Much, much more room for romhacks/enhancements
8. Most importantly, it's fun!!  Finding ways to port these games is, to me, like solving a big puzzle.


## My Goals when I port

When I started porting NES games, there were a few goals that I had for how I would do the ports.  My goals for these ports were:

1. Keep the game as true to the original as possible
2. Have an actual *build* of the game, one that I could use to add features to easily.  One of my motiviations was to be able to have more room to add features to the Kid Icarus Randomizer, which is squeezed into the NES Rom.  So I wanted, effectively, source code for as much of the game as possible.
3. Make it easy for other modders / romhackers to update things like the graphics / sound


## The Overall Process

To accomplish my ports, I follow this general approach:

1. Take the original NES ROM, and break it up into it's requisite memory banks by creating `.asm` files for each bank.
2. Use a buildable SNES game skeleton, with the NES games memory banks mapped to various convienent banks in the SNES game.
3. Hook up the SNES skeleton to jump to the NES initialization/nmi at the proper times
4. Slowly replace routines of the NES byte code with decompiled assembly, aquired from running the NES game in Mesen2.
5. Find and modify the NES specific logic and tile graphics to work on the nes.  Especially things like:
	a. All writes to Video Memory
    b. Deal with differences in screen resolution / BG size / Mirroring
    c. Update audio to use Membler's 2A03 emulator
    d. Properly handle bank switching

## (Current) Limitations on my Approach

My current approach only works on games with either no mapper, or the simpler mappers like UNROM and MMC1.  There's also limitations for dealing with games that use a lot of CHR ROM bank swapping, but so far even games that use that like Chip n' Dale and Super Dodge Ball have been doable.

## Future Blog Articles

In the next series of blog articles, I'll be going through the process for porting the game Double Dragon to the SNES.  Hopefully it goes smoothly, but even if it doesn't, my approach should give you a starting point for trying to do this yourself.  My rough draft of the articles will be something like this:

1. Setting up a new NES to SNES conversion project
2. Loading Title Screen Tiles into Video Memory
3. Drawing the title screen
4. Setting up Sprite conversion
5. Getting to Level 1
6. Scrolling
7. NSF Sound via Membler's Emulator
8. Tile Attributes (this will probably be more than 1 post)
9. Adding MSU-1 Support

This will probably change as we go, and it's possible that some of these will be multiple posts.  For instance I might just do 1 post on dealing with bank switching, or I might include it in the 2nd post, since it's generally one of the first things you need to solve.
