# DGwine

Reproducible builds of the few Wine modules the DGPlayer WIN console (WinRunner)
replaces inside its container.

This is **not a fork of Wine**. It carries an (optionally empty) patch series plus
a CI workflow that checks out upstream Wine at a tag, applies the series, builds a
WoW64 tree, strips the result and publishes only the modules we actually swap in.

## Why it exists

`rootfs.tzst` ships Wine 10.10, whose `dlls/quartz/avidec.c` cannot handle a
dynamic format change. A game whose intro video renegotiates its format mid-stream
gets `Sample size is too small (99840 < 199680)` and a black screen — 창세기전:
서풍의광시곡 hit this 2,044 times in one run. Upstream fixed it in 10.11, so the
container runs a 10.11 `quartz.dll` dropped over the 10.10 install.

That file used to be taken out of a third-party prebuilt tarball. This repository
builds it from upstream source instead, so the binary in the APK asset has a
provenance we can point at and rebuild.

> Wine loads builtin PE modules from the install directory
> (`opt/wine/lib/wine/i386-windows/`), **not** from the prefix's `syswow64`.
> Replacing the copy inside the prefix has no effect. Because the rootfs is
> extracted on device, a module can be A/B tested over adb without rebuilding
> the APK.

## Output

A tree that maps 1:1 onto the container's Wine install:

```
lib/wine/i386-windows/quartz.dll
lib/wine/i386-windows/winegstreamer.dll
lib/wine/x86_64-windows/winegstreamer.dll
lib/wine/x86_64-unix/winegstreamer.so
```

Only `quartz.dll` is currently swapped in production; the `winegstreamer` modules
are built so the whole DirectShow stack can be moved to one version if it ever
becomes necessary.

## Build environment

Matched to the container: glibc 2.39 and GStreamer 1.25, so CI builds on
`ubuntu-24.04` (glibc 2.39, GStreamer 1.24 headers — ABI compatible). Gate 1
checks GStreamer was detected, Gate 2 checks the unix module really links it, and
Gate 3 checks the PE modules are PE32 images.

⚠ The GStreamer gate must match GStreamer's *own* diagnostics. Wine's configure
prints `<lib> ... won't be supported` for every optional dependency it cannot find
— Wayland, Vulkan, CUPS, SDL2 and a dozen more — so matching that phrase rejects
every build.

## Withdrawn: the "non-repositioning seek" patch

The first commit carried a patch to `GST_Seeking_SetPositions` that forced
`AM_SEEKING_NoFlush` when the caller passed `AM_SEEKING_NoPositioning`, on the
theory that Wine's MPEG splitter was skipping ~37.9 s of a looping MP3 on every
such seek. **It was wrong twice over and has been removed.**

1. **Wrong culprit.** Wine's MPEG-I splitter was never in that graph. The real
   chain was `Reader → LAV Splitter → AC3Filter → Wave Audio Renderer` — a
   third-party StarCodec install in the user's prefix, which autoplug preferred on
   merit. The `fixme:quartz:mpeg_splitter_sink_query_accept Unsupported subtype`
   lines that pointed me at Wine were autoplug *probing* Wine's splitter and being
   turned down. Renaming `LAVSplitter.ax` out of the way fixed the looping.

   Read `FilterGraph2_AddFilter graph …, name L"…"` to learn which filters a graph
   actually contains. Counting `GST_Seeking_SetPositions` calls is a good
   cross-check: 528 seeks reached the passthroughs, none reached Wine's parser.

2. **Harmful patch.** The `BeginFlush` it skipped is not only about discarding
   buffered data — it is what unblocks a streaming thread parked inside
   `IMemInputPin::Receive` on a full renderer. Without it that thread holds
   `pin.flushing_cs` forever and the seeking thread deadlocks
   (`err:sync:RtlpWaitForCriticalSection … quartz_parser.c: pin.flushing_cs`),
   which froze the game as soon as Wine's splitter was finally in the audio path.

Any patch that forces `AM_SEEKING_NoFlush` on that path reproduces the deadlock.

## Game speed: a second series, on a different base

`patches-winlator/` and the **Build Winlator ntdll (game speed)** workflow are a
separate track from everything above, because they cannot build on upstream Wine.

WinRunner can now fast-forward and slow-motion a game. Wine is not an emulator, so
there is no core loop to run more or fewer times; the only lever is the clock the
game reads. The Android side publishes a scale into a small mmap'd file and the
patch makes ntdll derive its clock from it — which covers
`QueryPerformanceCounter`, `timeGetTime` (winmm builds it on QPC), `Sleep` and
every wait timeout. `GetTickCount` is *not* covered: it reads
`user_shared_data->TickCount`, which wineserver writes.

### Why it cannot use upstream

The container sets `WINEESYNC=1` on every session, and **esync has never been in
any upstream Wine release** — not 10.10, not master (whose answer is the in-kernel
`ntsync` driver instead), and not in current wine-staging either. The shipped
`ntdll.so` carries `WINEESYNC`×4 and the shipped `wineserver` `esync`×45, so the
rootfs is definitively an esync build.

`quartz.dll` got away with a cross-version swap because it is a leaf module. ntdll
is the syscall boundary: an upstream-built one would drop esync and would not
speak the same protocol as the container's wineserver. The base is therefore
Winlator's own tree, [`brunodev85/wine-10.10-custom`](https://github.com/brunodev85/wine-10.10-custom),
which carries esync and matches the container's version. Gate 0 fails the build if
that tree ever stops having it.

Its four stock call sites are byte-identical to upstream 10.10; `esync.c` adds a
fifth hunk, and missing it would leave every esync wait at 1x while the rest of
the clock ran scaled.

### Output

```
lib/wine/i386-windows/ntdll.dll
lib/wine/x86_64-windows/ntdll.dll
lib/wine/x86_64-unix/ntdll.so
bin/wineserver
```

ntdll's two halves are one unit — the syscall numbers in the PE stubs have to match
the unix dispatcher — and `wineserver` ships with them because it shares the esync
protocol. Everything else in the container imports ntdll by name and is unaffected.

### The gate that matters

Not "does 9x work" but **"is 1x still identical"**. With no mapping file every
helper falls back to the same clock (`CLOCK_MONOTONIC_RAW` first) and the same
arithmetic as the unpatched code, so a 1x regression means the swap itself is
wrong — most likely the rootfs was built from a different revision of the base
tree than the one CI used.
