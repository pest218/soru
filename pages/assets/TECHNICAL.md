# Technical Overview

This document explains how the hack is built, for anyone who wants to understand or extend it.

## Boot hook

The reset vector is redirected to custom code. That code first replays the original game's own boot-init bytes verbatim (CPU native-mode/stack/direct-page setup, the original PPU BG-mode/tilemap/CHR register writes, both of the game's own original init subroutine calls), then continues into the menu instead of jumping straight to the title screen. Because this runs before the title screen exists at all, there's no interaction with any other in-game screen — the menu owns the whole frame until "START GAME" hands control back to the untouched vanilla boot flow at the exact point it would normally have continued.

## Menu rendering

The menu is drawn on the BG3 background layer, which is unused by the game at this point in boot. A small custom font (one tile per character, reusing the same tile format as the game's own shop-menu font) is DMA'd into VRAM, along with several palettes: normal text, a highlighted cursor color, and colors sampled directly from the real title screen's own art (used for the animated "ULTIMATE" title lettering). The whole 32×32 BG3 tilemap is explicitly cleared at boot — the original boot path never touches BG3 at all, so any register or VRAM region it doesn't set up itself has to be initialized from scratch.

Menu state (which row the cursor is on, each cheat toggle, the chosen track/direction/laps, per-row highlight color) lives in a dedicated block of WRAM (bank `$7F`) that vanilla code never reads or writes, so there's no risk of colliding with anything the game itself uses.

## Persistent effects (NMI hook)

The same code also installs itself at both NMI vectors (native and emulation mode) for the lifetime of the ROM. Before "START GAME" is pressed, the NMI hook's only job is signalling the menu loop that a new frame has started. After "START GAME", every single frame it:
- Re-writes the player's money/nitro/stat values if their respective toggle is on, so spending or losing them in-game has no lasting effect — they're restored right back on the next frame.
- Applies the direction/track/lap configuration chosen in the menu.

## Race length (LAPS) and the finish condition

The vanilla game checks for a race finish with a hardcoded comparison against a fixed lap count, duplicated at two separate points in its own code. That comparison is patched to instead read a WRAM byte that the menu sets from whatever LAPS value was chosen, so any race length from 1 to 9 laps finishes correctly — not just the original hardcoded length.

There's one display-only quirk worth explaining: the on-screen lap counter has room for exactly one digit per player. Finishing an N-lap race requires the game's internal lap-position counter to actually reach N+1 (so a 9-lap race's true finish value is 10) — that's necessary for the win/placement logic to fire correctly and is left completely untouched. The problem is purely that "10" can't be shown in a single digit. A separate, narrowly-scoped patch clamps only what's drawn on screen (not the real counter used by game logic) so that a true value of 10 displays as "0" instead of a broken tile. The underlying race result is unaffected either way — this only changes what's shown for that one frame.

## Freezing the last lap

Turning on "LAST LAP" stops the lap-position counter from advancing past whatever LAPS is currently set to, so the race behaves as if the player is permanently on the final lap — this works correctly at any race length from 1 to 9, not just the default.

## Track and direction selection

The vanilla game picks each race's track and direction by indexing a fixed 64-entry table with a single WRAM race counter that increments once per race and wraps every 64 races. Selecting a track and direction in the menu overrides the exact point where the game consumes that counter to start a race, computing the correct table index for the chosen combination before handing off to the game's own unmodified race-start logic — so the race that plays out afterward is identical in every other respect to a normal one. A small number of tracks have no reverse layout in the original game; selecting REVERSE on one of those has no effect, matching how the game already treats them.

## Build process

- `src/build.py` assembles all of the above into 65816 machine code and produces the font/palette/track-data tables that get embedded alongside it.
- `src/patch_rom.py` applies that assembled code to a base ROM: it locates and rewrites the exact bytes described above (finish-line comparison, lap-digit render, race-start counter consumption), embeds the assembled code and data into unused ROM space, updates the interrupt vector table to point at the new boot/NMI code, and recomputes the SNES ROM header checksum.
- The same script then diffs the original ROM against the patched one to produce the `.ips` patch, and verifies the diff round-trips back to an identical ROM before writing anything out.

Two versions are produced from two base ROMs (standard and FastROM) because the free/unused ROM space available for the injected code differs slightly between them.
