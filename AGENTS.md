# Pitch Dark: Display Artwork Troubleshooting & Fixes

This document records the series of hardware switching, memory, and WeeGUI interaction bugs encountered while developing the "Display Artwork" feature, along with their ultimate resolutions.

## 1. Mouse Failure Due to Overwritten Screen Holes
- **Issue**: Initially, the `BlankTextAndHGR` routine was implemented to fill the entire `$0400-$07FF` (text memory) range with spaces (`$A0`). This caused the mouse and WeeGUI to become completely unresponsive.
- **Root Cause**: In the Apple II's text display memory, specific ranges like `$x78-$x7F` and `$xF8-$xFF` are known as "Screen Holes". These memory segments do not correspond to any characters on the screen; instead, they are reserved by the system as scratchpad memory for peripherals (such as the mouse interface card). Forcibly clearing these areas destroys the internal state of the mouse driver.
- **Solution**: Modified `BlankTextAndHGR` to include address boundary checks within the clearing loop, actively skipping the Screen Holes.

## 2. Premature Exit Caused by `plx` Overwriting the Z Flag
- **Issue**: Users reported that upon entering graphics mode, "without pressing any buttons, the screen flashes the image and immediately returns to text mode."
- **Root Cause**: In the `AnyKeyOrClick` function located in `ui.common.a`, the following pattern was used:
  ```assembly
  cpx #$FF
  plx
  bne @pressed
  ```
  In 6502 (specifically 65c02) assembly, the `plx` instruction (which pulls the top value from the stack back into the X register) **modifies the Z (Zero) and N (Negative) flags**. Since the pulled value is typically non-zero (e.g., the `$FF` we just pushed), the Z flag is always cleared. Consequently, the subsequent `bne` instruction always branches, tricking the program into thinking the user has clicked, resulting in an immediate exit from the wait loop.
- **Solution**: Branch based on the comparison result *before* executing `plx`:
  ```assembly
  cpx #$FF
  beq @noClick
  plx
  sec
  rts
  ```

## 3. Freeze Caused by an Infinite Loop
- **Issue**: To solve the issue of lingering clicks, a `@drainMouse` loop was temporarily introduced to continuously call `WGPendingClick` to drain the mouse queue. However, when the user clicked the mouse to enter graphics mode, the system completely froze and could not return to text mode.
- **Root Cause**: The `WGPendingClick` API in WeeGUI is strictly a "peek" operation; it **does not** consume or clear the pending click record. This meant that if a residual click existed, the loop would spin infinitely without ever clearing the state.
- **Solution**: Removed the infinite loop and reverted to using the standard `ClearPendingInput` routine (which internally calls `WGClearPendingClick`), safely and reliably clearing the pending click queue exactly once.

## 4. Cursor Disappearance and Memory Leaks via Repeated `WGInit` Calls
- **Issue**: After transitioning in and out of graphics mode several times, the mouse cursor would permanently disappear.
- **Root Cause**: To restore the text interface when exiting graphics mode, the old code called `WGInit` and `WGEnableMouse` in `ui.artwork.a`. However, `WGInit` is designed strictly for "startup initialization". It registers the mouse interrupt handler in the Apple II's IRQ vector chain. Repeatedly calling `WGInit` without unregistering the old handler endlessly stacks interrupt routines, eventually exhausting the vector chain or causing a memory leak, which crashes the mouse driver and makes the cursor disappear.
- **Solution**: Removed all calls to `WGInit` when exiting graphics mode. Since WeeGUI is never actually shut down when switching to DHGR (only the hardware display modes are toggled), exiting simply requires jumping back to `MainScreen` to redraw the interface. Re-initialization is neither necessary nor safe.

## 5. Garbled 80-Column Text (Interleaved Spaces)
- **Issue**: After removing `WGInit`, exiting graphics mode resulted in garbled 80-column text, with spaces interleaved between every character (e.g., `Z R : T E G`).
- **Root Cause**: The Apple II's 80-column display is achieved by interleaving writes between Main Memory and Aux Memory. Before entering DHGR graphics mode, the program turned off `80STORE` (`sta $C000`). By failing to turn it back on upon exit, all subsequent text writes were forced entirely into Main Memory. This caused the characters in the even columns to be rendered as the blank spaces left over in Aux Memory.
- **Solution**: Manually restored the hardware switches during the `ui.artwork.a` exit sequence, perfectly restoring the 80-column text mode state:
  ```assembly
  sta $C001 ; 80STORE ON
  sta $C00D ; 80VID ON
  ```

## Verified Emulators
This final fix has been extensively tested and verified to pass across the following Apple II emulation software:
- **[izapple2](https://github.com/ivanizag/izapple2)**
- **[Apple2TS](https://github.com/ct6502/apple2ts)**
- **[AppleWin](https://github.com/AppleWin/AppleWin)**
- **[microM8](https://paleotronic.com/software/microm8/)**
