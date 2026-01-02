# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a complete disassembly of the NES Tetris ROM (U) [!].nes into 6502 assembly language. The project can rebuild a byte-perfect copy of the original game ROM (`md5: ec58574d96bee8c8927884ae6e7a2508`).

## Build Commands

Build the ROM:
```bash
make
```

Verify the build matches the original ROM:
```bash
make compare
```

Clean build artifacts:
```bash
make clean
```

Build only the C tools (scan_includes):
```bash
make tools
```

## Build System Architecture

The build uses the cc65 toolchain (CA65 assembler and LD65 linker) with a custom build configuration:

1. **Dependencies**: The build automatically compiles custom C tools in `tools/cTools/` before assembling. The `scan_includes` tool analyzes .asm files to determine dependencies.

2. **Assembly Process**: Two main object files are created:
   - `tetris-ram.o` - RAM/zero page variable definitions
   - `tetris.o` - Main game code and data

3. **Graphics**: PNG files in `gfx/` are converted to .chr (CHR ROM) format using Python tools from `tools/nes-util/`

4. **Linker**: The linker uses `tetris.nes.cfg` to organize memory segments and produces debug files (`.dbg`, `.lbl`)

## Code Organization

### Entry Points and Segments

The build is controlled by segment definitions in `tetris.nes.cfg`:

- **HEADER**: iNES ROM header (defined in `tetris.asm`)
- **PRG_chunk1**: Main program code starting at $8000
- **unreferenced_data1**: Unused data section
- **PRG_chunk2**: Additional program code at $DD00
- **unreferenced_data4**: More unused data
- **PRG_chunk3**: Final program code at $FF00
- **VECTORS**: NMI/Reset/IRQ vectors at $FFFA
- **CHR**: Graphics data (tilesets)

### Key Files

- **tetris.asm**: Top-level file with iNES header and includes `main.asm`
- **main.asm**: Contains all game logic (5700+ lines). Organized by segments with includes for music and data
- **tetris-ram.asm**: Zero page and RAM variable definitions using `.res` directives
- **constants.asm**: Hardware registers (PPU, APU, MMC1) and button constants

### Memory Layout

The game uses a sophisticated memory layout with conditional compilation for NWC (Nintendo World Championships) variant:

- **$0000-$003F**: Zero page temporaries and core variables
- **$0040-$005F**: Active player data (copied from player-specific areas for processing)
- **$0060-$007F**: Player 1 data
- **$0080-$009F**: Player 2 data
- **$00A0-$00FF**: Game state, rendering, audio staging
- **$0200-$02FF**: OAM staging (sprite data)
- **$0400-$04FF**: Player 1 playfield
- **$0500-$05FF**: Player 2 playfield
- **$0680-$06FF**: Audio system state

The active player's data is copied to $0040-$005F for processing, then copied back. This allows the same code to operate on either player.

### Graphics Pipeline

Graphics are PNG files that get converted to NES CHR format:

1. Source PNGs in `gfx/`: `*_tileset.png` for character/tile graphics
2. Converted to `.chr` files via `nes_chr_encode.py`
3. Included in final ROM via `.incbin` directives in `tetris.asm`
4. Nametables (screen layouts) stored as binary `.bin` files in `gfx/nametables/`

### Audio System

Music data is in `audio/music/` and included into `main.asm`. The audio engine uses:

- Music staging variables ($0680-$068D) for APU register values
- Music data structures with note/duration tables and channel pointers
- Bytecode-like instruction system with loop counters

### NWC Variant

The codebase supports building a Nintendo World Championships variant using conditional assembly (`.if NWC = 1`). This affects:

- Memory layout (variables at different addresses)
- Graphics (different CHR data)
- Unreferenced data sections

## Development Notes

### Character Encoding

The game uses a custom character map defined in `charmap.asm` for high score name entry. Most text is embedded directly in nametable graphics data.

### MMC1 Mapper

The game uses MMC1 mapper (iNES mapper 1) with:
- Horizontal mirroring
- 16KB PRG ROM chunks (2 chunks)
- 8KB CHR ROM chunks (2 chunks)

### Verification

After making changes, always run `make compare` to verify the ROM still matches the original SHA1 hash in `tetris.sha1`. Byte-perfect accuracy is a core goal of this disassembly.
