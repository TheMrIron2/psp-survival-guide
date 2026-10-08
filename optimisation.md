# PSP Optimisation

I see a lot of developers streaming into the PSP with the advent of LLM-assisted and vibecoded programming. The enthusiasm is great, and I am happy to see that.
But there are a lot of platform specifics that neither a new developer nor an LLM will be appropriately equipped for.

From my experience, and from what I see of recent ports, LLMs make several core misunderstandings of how the PSP should be optimised for, either due to lack of knowledge or modern assumptions. This is a working document and I've written it myself; you may read it for your own benefit or just pass it to an LLM. Either way, the quality of your PSP software should improve. This document is not ordered and should be read in its entirety; its learnings can then be applied to the most relevant bottlenecks of your software.

## Hardware

As any quick search will tell you, the PSP has a 333MHz MIPS CPU called Allegrex, 32 / 64MB RAM and a GPU called the GE with 2MB RAM. Both GE and bus are clocked at exactly half the CPU speed, so at 333MHz, bus and GPU are 166MHz.

Or is that really it? The PSP has several other hardware features that LLMs are typically not aware of.
1. VRAM is actually increased to 4MB on PSP-2000 onwards, coinciding with the doubling of main RAM to 64MB.
2. The PSP has a second 333MHz CPU called the Media Engine. This is effectively the same chip as Allegrex, except without a VFPU. It has its own 2MB (PSP-1000) / 4MB (PSP-2000+) RAM.
3. Within the ME, there is a chip called the Virtual Mobile Engine (VME). This is taken directly from Sony's early 2000s Walkmans; it is a reprogrammable and highly efficient chip.
4. The PSP also has 16KB of scratchpad. However, this is probably useless. It is only situationally faster than main RAM and homebrew developers have never found a good use for it. It's not impossible that a use will be discovered for it, but at the moment, it's considered not useful.
5. The PSP also has encryption and cryptography hardware called Kirk and Spock, after Star Trek. This is useful if you are hashing, eg. SHA1, and is nicer than a CPU implementation in software.
6. The PSP's VFPU is essential to good 3D performance. It performs T&L operations up to 8x faster than the CPU alone, and is well documented with a wide range of features.

So there are in fact several important areas that a naive application will not be aware of. It is important to be aware of them; an additional 2MB VRAM and a second high performance CPU lie entirely in the dark to most developers.

## Media Engine & VME

This deserves its own section. The Media Engine really is just a slightly cut version of the main CPU, and many new developers are entirely unaware of it. In commercial games, they were exposed only through a high level abstraction; developers were not allowed to write custom code to it. That is not so anymore! Developers can use the original libraries to cheaply decode media such as ATRAC audio and h264 video - and save a lot of CPU time - but also write custom code.

Custom code to the ME is tricky because it is not coherent the way a second CPU core would be. Work sent to the ME has to be asynchronous, like audio, or else wait for Allegrex - in which case Allegrex is probably faster on its own. It also breaks PPSSPP compatibility because PPSSPP can only emulate the ME via HLE; unexpected usage crashes. (As a rule of thumb, that means until PPSSPP gets an accurate LLE ME - if your code works on PPSSPP, it's not maximising the PSP!)

Tasks that can be performed in parallel gain huge performance this way. GBAdhoc has a good example; instead of rendering the emulated GBA screen on Allegrex, it hands the graphics over to ME, so that it can render the previous frame while Allegrex is already preparing the next one. This costs 1 frame of input latency as it is "behind" by 1 frame, but rendering time in Advance Wars improved from 12.5ms to 5.9ms - over 100% faster performance. An LLM-generated document about this technique was published by the developer and is [available to read here](https://gbadhoc.rayquaza.ca/docs/media-engine-renderer/).

The lesson is to think in terms of pipelines, not functions. The ME is very useful when it can perform substantial work that is decoupled from Allegrex's immediate task. mcidclan's [Media Engine custom core](https://github.com/mcidclan/psp-media-engine-custom-core) is currently the most useful library for providing access to the ME.

A research note: Sony's own PS1 emulator, POPS, has an even further optimised system that is not well understood yet at the time of writing (8 OCT 2026). Rather than periodically sending audio mixing jobs to the ME, POPS runs the PS1's entire SPU emulator on the ME. The ME independently generates 44,100 stereo samples per second, emulating 24 ADPCM voices and audio processing. Allegrex maintains the emulated SPU's register interface and communicates changes through shared memory, leaving the ME to advance the audio subsystem.
The key is persistent ownership. Instead of repeatedly transferring responsibility between processors, one processor owns a subsystem and the other communicates with it through small, carefully synchronised state updates.

This architecture can be applicable beyond audio, particularly to emulators whose original hardware contains independent processors or peripherals. However, it demands careful management. Further disassembly of POPS is required to understand how Sony actually did it - and doing so could benefit the whole scene! - but the idea can be experimented with today. The only public documentation is on [PS Dev Wiki](https://www.psdevwiki.com/psp/POPS).

### VME

The VME is only recently becoming understood, but it's a very versatile chip. It's an efficient reprogrammable CGRA with a 24-bit bus and 4 PEs (processing elements). Its usage is potentially far more interesting than just media! As mcidclan has discovered, these PEs can be reprogrammed to perform virtually any repeated operation. In his experiments, he found ways of using the VMEs for:

- Texture processing
- Pico AI neural networks
- Matrix multiplication and batching

This is not an exhaustive set of uses for the VME and it is still very much experimental. The primary problem is locality; the VME is not going to perform these arbitrary operations faster than equivalents on the VFPU, even if it is faster than the ME in scalar. It will therefore accelerate ME operations well, but is an unintuitive solution for tasks that are on the Allegrex side. It may provide an excellent playground for a developer eager to try and accelerate ME code or find new hardware solutions. His [VME samples](https://github.com/mcidclan/psp-virtual-mobile-engine-ext) are available here.

Please additionally refer to mcidclan's [excellent technical investigation](https://github.com/mcidclan/psp-media-engine-cracking-the-unknown/blob/main/the-psp-virtual-mobile-engine/the-virtual-mobile-engine.md) of the chip's specifics, which is the basis for most of our current knowledge.

## VFPU

The PSP's VFPU is essential to good optimisation. Use of the VFPU should be considered non-negotiable for just about any application concerned with 3D performance.

It has 128 32-bit registers - arranged as eight 4x4 matrices - and supports arithmetic, dot products, matrix multiplication, transformation, trigonometry and various data operations. This allows the PSP to perform calculations in far fewer instructions than the CPU alone. It should be noted that the VFPU is a coprocessor of Allegrex, not a separate processor; further, the GE has its own T&L pipeline, so it may be useful to perform some tasks on the GPU directly. However, on work that is local to Allegrex, it is enormously faster.
Star Fox 64 PSP gained an 8.8x speedup on matrix multiplication by applying the VFPUs instead of performing the maths scalar, reducing what was a 112.5ms scene in early development to 46.6ms - a 2.5x boost in framerate. 4x4 matrix multiplication requires 112 instructions on Allegrex; the VFPU can perform it in 1.

Developers do not need to implement everything from scratch. PSPSDK contains VFPU-enabled routines in [pspgum_vfpu.c](https://github.com/pspdev/pspsdk/blob/master/src/gum/pspgum_vfpu.c), which provides useful examples of matrix manipulation, scaling and rotation. The [n64psp](https://github.com/TheMrIron2/n64psp/tree/main/src) library contains adapted VFPU routines based on DaedalusX64 which are tailored for N64 code, but are generally applicable.

Do **not** assume that the compiler will convert maths to the VFPU. If you do, lots of slow scalar code will sneak in. You'll need to do it yourself - it's worth it. 

### VFPU Memory

The VFPU is not limited to this set of mathematical applications. One example of particular note is DaedalusX64's "VFPU memcpy" routine. Normal MIPS code might copy 16 bytes using 8 instructions - four 32-bit loads and four 32-bit stores. The VFPU can do this in just two:

```
lv.q C000, 0(a1)    # Load 16 bytes from source
sv.q C000, 0(a0)    # Store 16 bytes to destination
```

Each register holds four 32-bit elements, so one instruction transfers 16 bytes. Both instructions require 16-byte-aligned memory addresses. Despite being a floating-point coprocessor, the VFPU doesn't actually perform any floating-point calculations here. The load/store instructions simply transfer the underlying bits.
This makes the technique useful for arbitrary data, not just vectors or matrices.

This technique typically improves performance but is workload dependent. DaedalusX64 itself notes that for operations that are too small or with no 32-bit alignment, it is not worth using.

You can read this [VFPU memcpy](https://github.com/DaedalusX64/daedalus/blob/master/Source/SysPSP/Utility/FastMemcpyPSP.cpp) code here.

## APIs

PSPGL is a good choice when porting software to the PSP. It is a quite compact implementation of OpenGL ES 1.1 that sends commands straight to GE. In frontend-heavy programs, expect a penalty compared to GU of ~10% or more, but this is not scientific and heavily variable. A good PSPGL backend is better than a naive GU one.
Start with PSPGL if your ported software already has an OpenGL renderer, then rework it to GU if you are not satisfied with performance. Use more direct GE calls only if the bottleneck specifically demands it - measure it first.

SDL2 and SDL3 are similarly useful for covering the bases of porting software to PSP. PSP's implementations of SDL are mature and regularly maintained. Don't shy away from them; they will be more than adequate for most use cases.

## CPU Caches

The PSP's CPU has just 16KB instruction cache and 16KB data cache. When dealing with complex software, cache locality is likely to have a huge impact on performance! The compiler can only do so much to put the hottest code in the right place. One example from sf64-psp; converting Peppy's limbs to native PSP seemed to save 0.7ms on the title screen, from 15.5ms to 14.8ms. Removing the selector not only erased the optimisation but made the whole frame worse, down to 16.0ms. You should expect cache nightmares with such little space.

This does make the lowest level of optimisation difficult. Changing, moving, expanding or shrinking unrelated work can and will affect the performance of critical code, often making it difficult to tell if you are optimising anything at all. Without manual cache management, this leaves performance completely up to arrangement luck.

There are few good resources on dealing with this problem at present. The most recent research and testing comes from [GBAdhoc](https://gbadhoc.rayquaza.ca/docs/layout-pinning/), which profiled the hottest functions and pinned them to cache, improving performance by 7-25%. 

Related to this cache sensitivity, don't assume that `-O3` or `-Os` are the fastest flags for compiling. 

## Textures

Choosing the right texture format can be a significant factor in performance. The PSP is not especially fast at moving data around; optimising the format can avoid a lot of stalls. CLUT4 and CLUT8 are useful for reducing the footprint of textures. Keep them small and lower bit depth where possible! Format switching for textures is basically free.

Swizzle your textures. **Always**. There's no reason not to - it significantly improves the speed at which the PSP can move and operate on them for free. The PSP can read textures from main memory fast, so don't try and stuff the already small VRAM with textures. Find a good sized texture cache - you'll probably need to experiment for the right number - and allow the code to decide. 

## Remove Work, Don't Bookkeep

The PSP does not like indexed work. Indexed geometry is slower on PSP than non-indexed geometry, even though traditionally this is a speedup. From Star Fox 64; a transformed water reuse experiment was performed where the game tried to avoid re-transforming water redundantly by using a large transformed representation. This made VFPU instructions fall by 39%, but performance regressed substantially (4.6ms). Subsequently, a rebatch that removed redundant loads won back 1.20ms per task. This is a textbook example of how trying to be too smart and bookkeeping / indexing your work typically runs worse than just crunching it.

## High Performance GE Design

Hrydgard documented a technique used by commercial developers who wanted to maximise the PSP's GPU capabilities that he calls the "double buffer command list" approach. The idea is simple; while the GE draws, the Allegrex is already preparing the next frame. An SDK example of this technique can be [found here](https://github.com/pspdev/pspsdk/blob/master/src/samples/gu/doublelist/doublelist.c).

You will lose 1 frame of latency this way, due to the structure of this approach, but it enables more concurrent operation between CPU and GPU and thus better performance in CPU-intensive games.

## Debugging

Use the full set of debugging tools available to you. [PSPDEV](https://pspdev.github.io/debugging.html) has resources on PSPLink, gprof and GDB.

PSPLink is the obvious pick for real-time testing; you can set `profmode t` for hardware counters too. `psp-gprof` is very useful for finding hot functions and attacking them. Ensure you use it right! Regular `gprof` can get symbols muddled.

# References

The following links are useful further reading for exploring optimisation for the PSP. I have adapted some of their learnings here, and credit is due to them.

- PPSSPP: [Optimizing code for the GE](https://www.ppsspp.org/docs/development/psp-internals/ge-performance/)
- PPSSPP: [Tips for developing for the PSP](https://www.ppsspp.org/docs/development/psp-internals/psp-development/)
- Rodrigo Copetti: [PSP Architecture](https://www.copetti.org/writings/consoles/playstation-portable/)
- TyRaNiD: [The Naked PSP](https://uofw.github.io/upspd/docs/software/naked_psp.pdf)
- Jeremy Fitzhardinge: [PSP Cache HOWTO](https://uofw.github.io/upspd/docs/hardware/PSP_cache-howto.html)
- PSPSDK: [Topics](https://pspdev.github.io/pspsdk/topics.html) 