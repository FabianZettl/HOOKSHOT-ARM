# Hookshot on ARM64 Linux (experimental)

Status: **boots and draws, not playable yet.** Tested once, on an AYN Thor (Snapdragon 8 Gen 2, Adreno 740, 11 GB RAM,
ArmadaOS, Mesa 26.2.6 Turnip), Hookshot v2.1.0, game 2.0, minimum preset at 1280x720: the title screen runs at 54-60 fps,
the saves of a PC load, and the game world is complete but low in resolution and detail. In the world it runs at about
5 fps (GPU time about 190 ms a frame, 141 ms of the replay in the memory sync, RAM full and swapping). No other ARM64
device has been tried.

## Build (on the device, in a Fedora aarch64 container)

Install clang, lld, cmake, ninja, python 3.14 with `requirements.txt`, SDL2/SDL3, Vulkan and SPIR-V headers, VMA and
ffmpeg-free development packages, then, from the clone:

    python -I nx2/nup.py <game.nup> --extract work/dump
    python -I nx2/dump.py --dump work/dump --set
    mkdir -p work/image
    python -I nx2/lift.py --standalone --write                       # about 5 min
    RX_MARCH=armv8.4-a python -I nx2/build.py --standalone --jobs 3   # about 55 min on 8 cores, 11 GB RAM is tight

`RX_MARCH` must be set: the CPU detection only knows x86. Run `work/sa/build/hookshot` from a folder holding
`hookshot_vk.so`, `image.bin` (`work/sa/image`), `romfs` and `user`, with `image.bin --romfs romfs --user user`.
`RX_VK_LOW_SPEC=1` and `HOS_OUTPUT_RES=1280x720` keep a first run small.

## What changed for aarch64

- `nx2/build.py`: no `-mcx16` on `armv*`.
- `host/posix/win32_posix.c`: the signal handler's context for aarch64 (pc, sp, the write bit from the ESR).
- `host/hos_setjmp.c`: `recomp_sj_setjmp` / `recomp_sj_longjmp` for aarch64.
- `host/hos_thread.c`, `host/posix/windows.h`, `host/posix/prof.c`: pause and pc.
- `gpu/vk/vk_memory.cpp`: **export mode.** Turnip has no `VK_EXT_external_memory_host` and many kernels no udmabuf, so
  the guest pools cannot be imported. Instead each pool piece (32 MB) is allocated as exportable device memory (a
  dma-buf), the guest bytes are copied into it and it is mapped over the guest window with `MAP_FIXED`. While a piece is
  copied its range is read-only and a thread that writes into it waits in a vectored exception handler until the new
  mapping is in. `RX_VK_NO_EXPORT=1` turns it off.

## Not checked

- Visual correctness beyond "the world is there" (no side by side with the x86 build).
- Whether the dma-buf mapping stays coherent under long play (one crash before the write-protect step, none after, over
  a few minutes).
- Other Adreno or Mali devices, other Mesa versions, Android.
