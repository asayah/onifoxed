# Oni on Apple Silicon (M1 and later)

This page records what it takes to run the **retail** game on an Apple Silicon Mac,
what this repository does today, and what is still missing.

## Short answer

* **To play the retail game on an M1 today, use [OniARM64](https://github.com/andiyar/OniARM64).**
  It is a native arm64 port built on top of this OniFoxed code base. It ships signed `.dmg`
  releases, loads both the Windows retail and the 2001 Mac retail `GameDataFolder`,
  renders through OpenGL or Metal, and has chapters 1 to 9 verified end to end
  (later chapters load but are not fully play-tested). It needs macOS 15 or newer.
* **Alternative without a source port:** run the Windows retail build (ideally with the
  community [Anniversary Edition](https://wiki.oni2.net) installer) through
  CrossOver, Wine or Parallels. That is how most M1 users played before OniARM64 existed.
* **This repository** now configures, compiles and links for macOS arm64 through the SDL
  platform layer, but a 64-bit executable **cannot load the game data yet** (see below).
  So the honest status is "compiles on Apple Silicon, does not run the game".

## Why a 64-bit build cannot load `level*_Final.dat` yet

Oni's level files are *instance files*: raw C structs written to disk by the original
32-bit tools. On load the engine memory-maps the file, byte-swaps it in place, and then
patches every template reference by writing a native pointer over the 4-byte slot the
file stores (`TMcSwapCode_TemplatePtr` and `TMcSwapCode_RawPtr` in
`BFW_TM_Game.c`). The C structs used to read that memory are the same structs the tools
used to write it, so their layout must match the file byte for byte.

With 8-byte pointers:

* every `tm_templateref` / `tm_raw(...)` field grows from 4 to 8 bytes, so every field
  after it in the struct is read from the wrong offset;
* the pointer patch writes 8 bytes into a 4-byte slot and clobbers the next field.

OniARM64 solved this with a translation layer (`TMrBridge_*` in its `BFW_TM_Game.c`):
each template gets a layout descriptor with on-disk and in-memory offsets, instances are
copied into a separately allocated "translated block" during load, and pointers are
resolved into the widened layout, with deferred fixups for cross-file references. That is
a multi-thousand-line change touching every template definition, and it is the real
work of an Apple Silicon port. Reimplementing it here is the next step if this fork wants
native Apple Silicon support of its own rather than pointing at OniARM64.

There is no way around 64-bit on Apple Silicon: macOS on arm64 cannot run 32-bit code at
all (Rosetta 2 only translates 64-bit Intel binaries).

## What this branch changed

* `BFW.h`: recognises modern macOS (`UUmPlatform_MacOSX`), x86_64 and ARM64
  (`UUmProcessor_ARM64`, 64-byte cache line hint, little endian), adds
  `UUmPlatform_Posix` for the code shared by Linux and macOS, and `UUmPointerSize`.
* All `UUmPlatform == UUmPlatform_Linux` checks that really mean "POSIX system" now use
  `UUmPlatform_Posix`, so macOS picks up the SDL, OpenAL and POSIX file manager paths.
* OpenGL and OpenAL headers resolve to `<OpenGL/...>` / `<OpenAL/...>` on macOS when
  the Homebrew `AL/al.h` is not on the include path; GL deprecation warnings are silenced.
* CMake: `Platform_SDL` defaults to ON on Apple, the SDL build on Apple reuses the Linux
  file manager instead of the classic Mac OS Carbon sources, Clang gets the same relaxed
  pointer warnings as GCC, and a 64-bit configure prints a warning pointing here.
* The instance file loader refuses with a clear message on a 64-bit build instead of
  corrupting memory.
* CI: new `sdl-linux-x86_64` and `sdl-macos-arm64` jobs so both 64-bit targets keep
  compiling.

## Building on an M1 (compile check only for now)

```sh
brew install cmake sdl2 openal-soft ffmpeg pkg-config
cmake -S . -B build -DPlatform_SDL=ON -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_PREFIX_PATH="$(brew --prefix openal-soft)"
cmake --build build --parallel
file build/bin/Oni   # Mach-O 64-bit executable arm64
```

## Retail vs demo data

This fork was aligned to the Windows *demo* data. OniARM64 validates the data by checksum
and accepts both Windows retail and Mac retail folders (Mac retail sound is Apple IMA4
encoded and is decoded on the fly). Any future 64-bit loader here should copy that
approach: detect the data flavour up front rather than assuming the demo layout.

## Sources

* OniARM64 repository and releases: <https://github.com/andiyar/OniARM64>
* OniFoxed upstream: <https://github.com/hogsy/OniFoxed>
* Oni community wiki (Anniversary Edition, Mac notes): <https://wiki.oni2.net>
