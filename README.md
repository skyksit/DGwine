# DGwine

Patched Wine modules for the DGPlayer WIN console (WinRunner container).

This repository does **not** fork Wine. It carries a small patch series plus a CI
workflow that checks out upstream Wine at a tag, applies the patches, builds a
WoW64 tree and publishes only the modules we actually replace in the container.

## Why

The WIN console runs games under Wine inside the WinRunner container. Wine's
DirectShow MP3 path has a defect that makes looping background music restart
every couple of seconds and eventually deadlocks the DirectShow filter graph.

`IMediaSeeking::SetPositions` is documented to leave the current position alone
when `AM_SEEKING_NoPositioning` is passed for it. Games use this to re-assert a
loop end point while playback continues — 창세기전: 서풍의광시곡 does it about
three times a second.

Wine treats that call as a full flushing seek anyway:

* `dlls/quartz/filtergraph.c` pauses the running graph, forwards the seek and
  re-runs it.
* `dlls/winegstreamer/quartz_parser.c` `GST_Seeking_SetPositions` calls
  `IPin_BeginFlush` on every source pin and `IAsyncReader_BeginFlush`, because
  the caller did not pass `AM_SEEKING_NoFlush`.
* `dlls/winegstreamer/wg_parser.c` `wg_parser_stream_seek` sends a GStreamer
  seek with `GST_SEEK_TYPE_NONE` **and** `GST_SEEK_FLAG_FLUSH`.

A flushing seek with `GST_SEEK_TYPE_NONE` discards everything queued downstream
and resumes from the demuxer's read-ahead point instead of from what has been
rendered. Measured on device with a 117.3 s 320 kbps MP3: each call advanced the
stream by **37.93 s** while only 323 ms of wall time passed.

```
NewSegment start   0.0          (after the game's rewind)
NewSegment start  37.7469387    (+1 call)
NewSegment start  75.6767346    (+1 call)
NewSegment start 113.6065306    (+1 call)
NewSegment start 117.3942857    clamped to EOF -> EOS -> EC_COMPLETE
```

Four calls, ~1.3 s, and the track "ends". The game rewinds to 0 and the cycle
repeats forever. The constant `Stop`/`Run` churn also reaches a
`strmbase_filter.stream_cs` deadlock, which is what froze the app.

Verified present in Wine 10.10, 10.11 and current master; not reported upstream.

## Patch

`patches/0001-winegstreamer-do-not-flush-on-a-non-repositioning-seek.patch`

Forces `AM_SEEKING_NoFlush` when the current position is not being moved, which
covers both the DirectShow-side flush and the GStreamer-side one, and
additionally drops `GST_SEEK_FLAG_FLUSH` in `wg_parser_stream_seek` whenever
`start_type` ends up as `GST_SEEK_TYPE_NONE`, so the invariant holds for any
other caller that reaches it.

## Output

The workflow publishes a tree that maps 1:1 onto the container's Wine install:

```
lib/wine/i386-windows/winegstreamer.dll
lib/wine/x86_64-windows/winegstreamer.dll
lib/wine/x86_64-unix/winegstreamer.so
lib/wine/i386-windows/quartz.dll
```

Drop these over `opt/wine/lib/wine/...` inside `rootfs.tzst`.

`quartz.dll` is included because the container ships Wine 10.10, whose
`dlls/quartz/avidec.c` cannot handle a dynamic format change and leaves the
game's intro video black (`Sample size is too small`); 10.11 fixed that.

> Wine loads builtin PE modules from the install directory
> (`opt/wine/lib/wine/i386-windows/`), **not** from the prefix's `syswow64`.
> Replacing the copy in the prefix has no effect.

## Build environment

The container is glibc 2.39 with GStreamer 1.25, so the workflow builds on
`ubuntu-24.04` (glibc 2.39, GStreamer 1.24 headers — ABI compatible).
