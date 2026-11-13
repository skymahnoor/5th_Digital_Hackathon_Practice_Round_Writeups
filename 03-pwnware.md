# Pwnware — Writeup (50 pts)

## Challenge Info
- **Category:** Binary Exploitation (pwn), AVR firmware
- **Given:** `firmware.bin` + `disasm.txt`, and a remote server to connect to via `nc`
- **Hint:** "How good is your AI with some foreign arch" — a hint that the architecture isn't the usual x86/ARM

## Step 1 — Identify the Architecture
Looking closely at the disassembly, several clues point to **AVR** (an 8-bit microcontroller architecture, like the one in Arduino boards):
- The interrupt vector table uses `0c 94 XX XX`, which is the AVR 2-word `JMP` instruction.
- The startup code clears registers and sets up the stack pointer in a pattern unique to AVR.
- There's a flash-to-RAM copy routine and a zeroed `.bss` section — standard AVR startup behavior.
- UART (serial) registers are mapped at known AVR addresses.

Note: the given `disasm.txt` had some incorrect decodes. Re-disassembling `firmware.bin` directly gave a cleaner result.

## Step 2 — Understand the Firmware
It's a simple serial console with a banner and a `$` prompt. It supports these commands:
- `echo` — reads a line of input and prints it back. **This has no length limit, so it can overflow its own stack buffer.**
- `peek` — reads a 16-bit address and prints 2 bytes of memory at that address. **No bounds checking** — you can read any memory address.
- `stat` — prints a fixed status message.
- `help` — lists the commands.
- `quit` — exits.

There's also an anti-tamper check: `echo` compares some saved global values against current ones, and freezes the program if they don't match (this acts like a basic stack canary).

## Step 3 — Find the Hidden "Win" Function
There's a function in the firmware that is never called by anything — dead code. It reads up to 64 bytes from the device's **EEPROM** into RAM. This is exactly what we need: the flag is stored in the remote device's EEPROM (not inside `firmware.bin` itself), and this unused function can read it out for us — if we can make the program jump to it.

## Step 4 — The Bug
The `echo` command reads input with no length limit. Since its stack buffer is only 34 bytes, sending a longer line overflows it — overwriting:
- The anti-tamper "canary" values (which can be read in advance using `peek`, since on a fresh instance they're just zero)
- The saved return address — which controls where the program jumps next after `echo` finishes

## Step 5 — Build the Exploit Chain
By carefully crafting the overflow payload, we can control what the program does after `echo` returns:
1. First, jump to the hidden EEPROM-reading function (loads the flag into RAM).
2. Then, jump back into the normal program loop so it doesn't crash.

After that, we simply use the `peek` command repeatedly to read out the flag bytes from RAM, two bytes at a time, and convert the hex values back into text.

## Tools Used
- A Python exploit script to send the overflow payload and automate reading the flag via `peek`
- A local AVR emulator to test and confirm the exploit chain works correctly before running it against the real remote server

## Status
The exploit chain was fully verified in a local emulator (confirmed match). The actual remote instance was temporarily down at the time, so the plan was to re-run the exploit as soon as a fresh instance became available.

## Lesson Learned
Any input read without a length limit is a potential stack buffer overflow. Even on small embedded systems like AVR microcontrollers, overflowing a buffer can let an attacker control the return address and hijack program flow — including jumping to "dead code" that was never meant to be reachable.
