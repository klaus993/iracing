# iRacing Graphics Configuration — Research & Tuning Notes

Reference notes for tuning `rendererDX11Monitor.ini`. Compiled 2026-08-13.

Every claim is tagged: **[DOC]** = iRacing-authored (ini comment, KB article, release note) ·
**[DEV]** = iRacing staff statement · **[BENCH]** = third-party measurement ·
**[COMMUNITY]** = guide/forum consensus, unmeasured · **[INFER]** = reasoning from
fundamentals, not sourced.

---

## 1. Hardware context

Current rig (as of 2026-08-13):

| Component | Spec |
|---|---|
| CPU | **Ryzen 5 8500G** — 6C/12T: 2× Zen 4 @ 5.0 GHz + 4× Zen 4c @ 3.7 GHz, 16 MB L3 (unified CCX) |
| GPU | **RTX 4080 16 GB** |
| RAM | 64 GB |
| Displays | Triple 1920×1080 (5760×1080) on the 4080 + 3440×1440 ultrawide |

Previous rig: ASUS X870E Hero + **Ryzen 9 9950X3D**, 64 GB, fast NVMe. The GPU did not
change — this was a CPU-only downgrade, and the config in this repo was tuned for the
old CPU.

### Why the 8500G changes the tuning problem

Two penalties stack, and they are independent:

**1. Draw-call throughput collapsed.** iRacing's DX11 renderer submits draw calls from
essentially one thread **[DEV]**, and triple screens triple that work **[DOC]**. The
8500G has 16 MB L3 versus 128 MB on the 9950X3D (96 MB on CCD0 alone), and only two
cores reach 5.0 GHz — the other four are Zen 4c capped at 3.7 GHz, a **26 % clock
deficit** measured by Phoronix **[BENCH]**. Zen 4c has identical IPC; it is a density
optimization, not a slower architecture. Note 16 MB L3 is below even a plain 7600/9600X
(32 MB), so this is not merely "an X3D chip without the cache."

**2. The GPU is on four PCIe lanes.** The 8500G (Phoenix 2) exposes 14 PCIe Gen 4 lanes
total, of which **only 4 reach the graphics slot** — regardless of motherboard **[BENCH,
TechPowerUp]**. No Ryzen 8000G part gives x16; the 8700G/8600G cap at x8.

| Link | Bandwidth | Relative |
|---|---|---|
| **PCIe 4.0 x4 (8500G)** | ~7.9 GB/s | 1.0× |
| PCIe 4.0 x16 | ~31.5 GB/s | 4× |
| PCIe 5.0 x16 (9950X3D) | ~63.0 GB/s | 8× |

The RTX 4080 is natively PCIe 4.0 x16, so it runs at **one quarter** of its designed host
bandwidth.

### The critical asymmetry: averages vs. 1 % lows

A narrow link costs ~6–11 % average FPS **[BENCH]**, but frame-time consistency degrades
far more. TechSpot's RX 6500 XT testing at x4:

| Game | Avg FPS drop | **1 % low drop** |
|---|---|---|
| Shadow of the Tomb Raider | −28 % | **−50 %** |
| Far Cry 6 | −17 % | −23 % |
| Assassin's Creed Valhalla | −9 % | −14 % |

**This asymmetry is the signature symptom**: fine when running alone, violent dips when
the scene churns.

The penalty is also **worst at low resolution** (−11 % at 1080p vs −6 % at 4K on an RTX
5090 **[BENCH]**), because per-frame host traffic is a larger share of the frame budget.
Triple 1080p at high refresh is close to the worst case.

**The 16 GB VRAM buffer is doing heavy lifting.** TechSpot measured the two regimes: with
VRAM headroom, a x4 link costs ~8 %; once VRAM overflows, **−49 % average and −56 % on
1 % lows**, with Doom Eternal degrading 171 % **[BENCH]**. On a x4 link, VRAM overflow
turns from annoying into unplayable. Never let the working set exceed 16 GB.

Estimated CPU delta vs the old rig: **~45–60 % of 9950X3D frame rate** in CPU-limited,
cache-heavy sim workloads, with the 1 % lows gap wider than the average gap. **[INFER]**
— no reviewer has benchmarked these two CPUs against each other; this is an evidence
chain, not a measurement.

---

## 2. Engine currency (verified 2026-08-13)

Checked against official release notes through **2026 S3 Patch 3 Hotfix 2 (2025-07-17)**.

- **iRacing is still DirectX 11 only.** Not "DX11 is still supported" — it is the only
  renderer. `rendererDX11Monitor.ini` is current and correct. **[DOC]**
- **No DLSS, no XeSS, no frame generation, no Reflex 2 / frame warp.** Zero mentions in
  any 2024–2026 release note. Upscaling is **AMD FSR only** (spatial, FSR1-class) per the
  official "Understanding Resolution Scaling" article, updated 2025-09-09. **[DOC]**
- **The replacement engine is "Spark."** DX12 prototype loading assets Aug 2024; DLSS
  prototype "performing very well" Aug 2025; Vertical Slice complete Feb 2026; May 2026
  dev update describes planning "a global cutover in a future release." **Vulkan is never
  mentioned in any source 2024–2026.** The Spark config filename has not been announced.
  **[DEV]**
- `NvReflexMode` is unchanged Reflex v1. `=2` (on+boost) is still the maximum.

### Recent changes that invalidate older guides

Most guides online date from 2016–2023 and are wrong on these points:

- **2025 S1 (2024-12-09) restructured MSAA.** Filtering modes added, with the note
  *"Filtered MSAAx2 should in most cases match the old MSAAx4."* **Sample counts are not
  comparable to any pre-2025 guide.** **[DOC]**
- **2025 S1** removed anisotropic filtering options — now always ×16. Dynamic Track
  Rendering is now always on. **[DOC]**
- **2024 S3 (2024-06-03)**: GPU particles mandatory, *"may not be disabled via the
  renderer.ini options"*; the option to disable Soft Particles was removed. **[DOC]**
- **2024 S4**: `AAQuality` removed from the ini. **[DOC]**
- **2026 S3 (2026-06-09)**: far terrains may no longer be disabled via ini. **[DOC]**
- **2022 S2 trap, still live**: "Low Quality Trees" was renamed "High Quality Trees" with
  the checkbox meaning **inverted**, while the ini key name stayed the same. So
  `LowQualityTrees=1` still means *low quality trees on* — the fast setting. No change
  found 2024–2026. **[DOC]**

---

## 3. CPU cost vs GPU cost — what is actually true

Most guides label nearly everything "GPU." iRacing's own Meter Box article is the
authoritative split **[DOC]**:

| Meter | Measures | Official prescribed fix |
|---|---|---|
| **R** | Renderer (CPU) frame time | Lower shadows, cubemaps, HDR, **object detail** |
| **G** | GPU draw time | Lower HDR, antialiasing, mirrors, sharpening, heat haze, distortion |
| **C** | Physics thread | Lower CPU settings **or reduce Max Cars** |
| **T** | Total frame time | Reduce R and G |

Press **F** in-sim to cycle the meters. **If R > G, the CPU is the limit and no GPU
setting will help.**

Settings iRacing attributes to CPU **in words**, verbatim from in-sim tooltips:

- **Shadow volumes** — *"very expensive as it requires CPU power to compute silhouette
  edges and CPU→GPU bus communications to extrude the volumes."* (Shadow *volumes* ≠
  shadow *maps*; maps are mostly GPU.)
- **SMP/MVP** — *"avoiding the CPU overhead associated with rendering each screen
  separately."*
- **3-projection triple-screen mode** — *"especially if used without SMP/MVP … the scene
  must be rendered two additional times every frame."*

Draw-call/scene-traversal settings (CPU by mechanism, per tooltip wording):
Event detail (*"more objects are active"* in races), Grandstands (*"very high detail and
complexity"*), Object detail, Car detail (*"particularly during race starts when driving
within a large pack"*), Pit objects, Cockpit mirrors, Draw Cars.

Genuinely GPU: MSAA, SSR, SSAO, sharpening, HDR, heat haze, distortion, shadow
*resolution*, resolution scaling/FSR. Official confirmation for the last one: *"This
feature will not reduce the CPU load of the sim"* **[DOC, 2025-09-09]**.

### Known errors in circulating guides

- **`AutoNoDynEmptyLOD` advice is inverted.** BoxThisLap's "hidden settings" article
  advises setting it to 1 to "aggressively eliminate peripheral objects." The real key is
  `AutoAddNoDynOnEmptyLOD`, its official comment is *"1 = Never use the FPS based LOD
  adjustment when popping to the 'empty' level"* — it **prevents** objects vanishing — and
  it is already 1 by default. Wrong key name, inverted effect.
- **"Max Cars is a graphics setting."** It is a *network request* that drives the physics
  thread; iRacing files it under the **C** meter.
- **Community CPU/GPU labels are unstable.** The most-cited author (Ehrling) reverses his
  own "Crowds" label between guide versions.
- **iRacing's own docs are inconsistent on mirrors** — listed as a GPU fix in the Meter
  Box article despite each mirror being an extra render pass.

---

## 4. Max Cars vs Draw Cars — two different settings

Frequently conflated. Only one makes cars invisible-but-collidable.

**`Max Cars` (text box) — network request, NOT graphics.** Official: *"controls how many
cars you are requesting the server send to your client … All of those cars have to be
moved through the world, they make sounds, they can be collided with, and they have to be
drawn on screen."* **[DOC]**

Documented downsides of lowering it: cars beyond the cap are **missing from your replay**;
the transmitted set **churns as you move through the field**; and **overlay/telemetry apps
break** (RaceLabs et al.). This is why the community standard is to leave it at **63**.

**`MaxCarsToDraw` / `MaxCarsToDrawInMirrors` — purely rendering.** Cars over this cap are
still transmitted, simulated, audible, collidable and recorded — just not drawn. Official:
*"by not drawing the least important cars relative to the camera."* Ranges: 10–64 and
4–64. **[DOC]**

**So yes — `MaxCarsToDraw` can produce a car you cannot see but can hit.** Mitigation is
built in: dropped cars are the distant ones. The real risk sits in **mirrors**, not the
main view.

**Correct approach: keep `Max Cars` at 63, do all FPS cutting with `Draw Cars`.**

In-game presets, format `Draw <main>(<mirrors>)`: Draw All · 64(30) · 64(16) · 64(8) ·
40(20) · 40(12) · 40(6) · 30(12) · 30(8) · 30(4) · 20(12) · 20(8) · 20(4) · Draw Min.

---

## 5. Dynamic LOD

Official tooltip: *"Dynamically adjusts the level-of-detail of cars, pitobjects, and world
to help maintain a minimum acceptable frame rate … The droplists of presets control how
much the LODs are allowed to change from normal."* **[DOC]**

Key mapping (from the sim's UI bindings):

- **World** dropdown → `LODPctMin`/`LODPctMax` + `LODPctMirrorsMin`/`LODPctMirrorsMax`
- **Cars** dropdown → `LODPctDynoMin`/`LODPctDynoMax` + `LODPctDynoMirrorsMin`/`Max`
- **FPS** → `LODMinFPSTarget` (shared)

Semantics (range 25–500 on all): `LODPctMin` is the *upper detail* bound — 100 means
detail can never exceed normal. `LODPctMax` is the *lower detail* bound — above 100
allows detail to drop.

**All eight keys at 100 means the system is inert**, regardless of `LODMinFPSTarget`.

**Critical scope limit [DEV]:** *"This feature doesn't adjust anything other than the
distance at which objects drop or gain polygon detail. It doesn't change shader, shadows,
or anything else."* So it reduces *vertex* load, **not draw-call count**. On a
draw-call-bound CPU its leverage is real but bounded. **[INFER]**

**Frame-time argument for decrease-only:** a controller that both raises and lowers detail
around a target will hunt near that target, and oscillation is precisely what damages
1 % lows. Keeping `Min=100` (never raise above normal) while allowing `Max>100` gives the
decrease-only behaviour that frame-time-focused guides converge on. **[INFER]**

Conflicting advice on `LODMinFPSTarget`: iRacing's 2020 S2 notes say set it to *"the bare
minimum you want to maintain"*; community VR guides say set it at refresh rate. For
frame-time consistency the official reading (a floor, not a ceiling) is safer.

---

## 6. Triple-screen: SMP

`EnableSMPSurround=1` + `RenderViewPerMonitor=1` is **the single largest CPU saving
available in the file.** SMP emits multiple projections from one geometry pass, so the CPU
submits the scene once instead of three times.

Still officially recommended as of the "Setting Up Three Monitors" article, **updated
2025-11-19**: *"If you have an NVIDIA graphics card, turn on 'Nvidia Simultaneous
Multi-Projection.' This will provide a performance enhancement."* No deprecation notice in
any 2024–2026 source. **[DOC]**

**They are a package deal** — `RenderViewPerMonitor=1` *without* SMP costs roughly 3×
scene submission.

### Documented forced downgrades under SMP — [DOC], widely unknown

*"When SMP is enabled, some other systems must be adjusted for compatibility: Particle
Detail is set to Low, Dynamic Night Shadows are Disabled (Shadow Volumes still work), and
Depth-of-Field effects are Disabled."*

**Consequence: the entire `DNSM*` block and `ParticleDetail` in this config are inert.**
Do not tune them expecting an effect.

### Rain caveat — [DOC, 2024 S3 Patch 2, 2024-06-28]

*"We have added a mechanism to disable SMP/MVP/SPS only for the particle effects system
code (PopcornFX)"* — added after GPU crashes. **So in the wet, the SMP speedup does not
cover spray particles.** The opt-in keys `PopcornFXSMPMVPAllowed` / `PopcornFXSPSAllowed`
default to off; re-enabling reintroduces the crash risk. Leave them absent.

Documented SMP limits: side screen angle clamped to 45°, FOV clamped to 160°.

`VRMode` in the *monitor* ini is almost certainly a no-op — Single Pass Stereo amplifies
geometry into a second *eye* view, and monitor rendering is monoscopic. The key exists
because iRacing writes one shared ini schema across all four display modes. **[INFER,
unverified]**

---

## 7. Applied configuration (2026-08-13)

Changes made to `rendererDX11Monitor.ini` for the 8500G. `[Replay Graphics]` deliberately
left untouched — replay quality does not affect driving performance.

### Tier A — no visual cost

| Setting | Old | New | Rationale |
|---|---|---|---|
| `CacheSwap3HighResCars` | 1 | **0** | Stops constant hi-res car texture swapping across the 7.9 GB/s x4 link as running order changes. **Prime suspect for traffic stutter.** |
| `LoadTexturesWhenDriving` | 1 | **0** | Moves texture load spikes out of green-flag running |
| `DesiredFPSLimit` | 140 | **100** | Was capping *above* the 120 Hz refresh with VSync off — outside the VRR window. A cap that is sustainable in a pack beats a high cap you drop out of |
| `maxParticleThreads` | 6 | **4** | 6 workers oversubscribes a 6-core CPU |
| `LODPctMax` | 100 | **200** | Dynamic LOD was fully inert (all 8 keys = 100) |
| `LODPctDynoMax` | 100 | **200** | " |
| `LODPctMirrorsMax` | 100 | **300** | " |
| `LODPctDynoMirrorsMax` | 100 | **300** | " |
| `LODMinFPSTarget` | 60 | **90** | Engage before the dip is felt, given a 100 cap |

All `LODPct*Min` left at 100 → decrease-only, no hunting.

### Tier B — minor visual cost, targets race starts

| Setting | Old | New | Rationale |
|---|---|---|---|
| `ObjectDetail` | 2 | **1** | The only maxed CPU-side setting; named explicitly in iRacing's R-meter fix list |
| `CrowdDetail` | 1 | **0** | iRacing's own dedicated FPS-tip article. Note crowds are suppressed in practice/qualifying and step up at green — a reason race FPS < practice FPS |
| `GrandstandDetail` | 1 | **0** | *"very high detail and complexity"* **[DOC]** |
| `WeekendDetail` | 1 | **0** | *"more objects are active"* in race sessions **[DOC]** |
| `MaxCarsToDrawInMirrors` | 8 | **4** | Virtual mirror still uses the mirror camera path |
| `HeadlightsInMirrors` | 1 | **0** | Extra mirror-pass lighting work |

`MaxCarsToDraw` deliberately kept at **20** — lowering it further is the one change with a
competitive edge (invisible-but-collidable cars).

### Held in reserve (Tier C)

`MaxCarsToDraw` 20→16 · `MaxPitObjsToDraw` 20→10 · `PitObjectDetail` 1→0 ·
`SSRLevel` 2→1 (halves wet-session SSR cost; `SSRRainOnly=1` already means zero cost when
dry).

### Already correct — do not "fix" these

`EnableSMPSurround=1`, `RenderViewPerMonitor=1`, `VisibilityFrameDelay=5` (max delay =
cheapest CPU; **lowering it costs CPU**), `TwoPassTrees=0`, `LowQualityTrees=1` (inverted
UI label — this is the fast setting), `SkyRefreshRate=0`, `OcclusionCull=1`,
`ParallelSorting=1`, `MaxCockpitMirrors=0`, `MirrorDetail=0`, all shadows/cubemaps/SSAO
off, `FoliageDetail=0`, `CompressTextures*=1`, `CompressedVertices=1`, `ZBuffer32Bits=1`,
`WorldNearPlaneDistance=10`.

Memory caps are already at the documented recommendation of *"20 % below your maximum"*
**[DOC]**: `SysMemToUseMB=32768` (the documented max, ~51 % of 64 GB) and
`VidMemToUseMB=12840` (~80 % of 16 GB). Setting these too high causes sessions to *"fail
to launch at 90 % of the session load"*; too low causes blurry car paints and wheel
displays.

---

## 8. Outstanding items outside the ini

1. **The ultrawide appears to be on the iGPU.** The Radeon 740M reports 3440×1440 while
   the RTX 4080 reports 1920×1080. If the primary desktop is on the 740M, every frame for
   it crosses the same 7.9 GB/s link the game is competing for. Microsoft documents the
   cross-adapter copy path costing ~16 % FPS and 27 % display latency **[BENCH,
   DirectX team]**; a Tom's Hardware desktop report showed 60→30 FPS with dips to 15.
   **Move all displays to the 4080.** G-Sync also cannot work on an iGPU-driven display.
2. **Verify actual refresh rate.** The ini says `RefreshRate=120.000` but Windows reports
   179 Hz on one display. If the triples are 165/180 Hz, the ini value is stale.
3. **BIOS**: enable Above 4G Decoding + Resizable BAR (requires UEFI/GPT boot and **CSM
   disabled**, or ReBAR silently won't engage). Expect little — TechSpot measured SAM
   giving +5 % at Gen4 x4 and **0 % at Gen3 x4**, i.e. no relief precisely when bandwidth
   is the constraint **[BENCH]**. Enable and A/B test.
4. **Check M.2 lane sharing** in the board manual — on many B650 boards a third M.2 slot
   shares bandwidth with the x16 slot. At x4 there is zero headroom to give up.
5. **`fullScreenWidth=7560`** is wrong for 3×1920 = 5760. Currently inert because
   `fullScreen=0` and `windowedWidth=5760` is correct, but it would misconfigure a switch
   to exclusive fullscreen.
6. **Known 2025 fix worth having**: 2025 S4 Patch 4 (2025-11-18) fixed *"framerate
   significantly drops after racing for some length of time."* Stay patched.

---

## 9. Validation method

1. Press **F** in-sim to show the meter box. Compare **R** vs **G**.
2. If **R > G** → CPU-limited; only object-count settings will help.
3. If **G > R** → GPU-limited; MSAA/SSR/resolution scaling become the levers.
4. **Test in a race start, not in practice.** Crowds are suppressed outside races and
   event objects step up at green — practice FPS is not predictive of race FPS.
5. Watch **1 % lows / frame-time graph**, not average FPS. On this hardware the average is
   not where the problem lives.

---

## 10. Unverified / open questions

- Whether a car the server never transmitted can still collide with you. Sources only
  confirm that *transmitted* cars can. No source addresses the non-transmitted case.
- Whether the virtual mirror is governed by `MaxCarsToDrawInMirrors`. The comment says
  "per mirror camera," implying yes; unconfirmed.
- `ReduceCockpitFlicker` — no authoritative explanation found beyond `0=off 1=enabled`.
- Exact numeric values each Dynamic LOD preset writes. Not published anywhere and not
  present as a table in the binary. **Settle it by selecting each preset in-game and
  diffing the ini.**
- Thread migration between Zen 4 and Zen 4c cores hurting sim frame times — mechanistically
  plausible, no benchmark isolates it.
- ReBAR on/off at Gen 4 x4 with a 16 GB card — no dedicated test exists.
- Magnitude of the iGPU-second-display penalty on desktop — mechanism confirmed by
  Microsoft, magnitude only from forum reports.
- `MaxCarsToDraw` / Max Cars: **no change found in any 2024–2026 release note** (stated
  explicitly rather than inferred).

---

## Sources

**iRacing official**
[Meter Box (F key)](https://support.iracing.com/support/solutions/articles/31000133494-meter-box-f-key-in-game-) ·
[Connection Type & Max Cars](https://support.iracing.com/support/solutions/articles/31000149355-connection-type-max-cars) ·
[Dealing with Freezing/Stuttering](https://support.iracing.com/support/solutions/articles/31000141916-dealing-with-freezing-and-or-stuttering-issues) ·
[Graphics performance tip](https://support.iracing.com/support/solutions/articles/31000133465-graphics-performance-tip-for-increased-fps) ·
[Setting Up Three Monitors](https://support.iracing.com/support/solutions/articles/31000171395-setting-up-three-monitors) ·
[Understanding Resolution Scaling](https://support.iracing.com/support/solutions/articles/31000167510-understanding-resolution-scaling) ·
[RAM/VRAM allocation](https://support.iracing.com/support/solutions/articles/31000172032-manually-setting-ram-and-vram-allocated-to-the-sim)

**Release notes**
[2026 S3](https://support.iracing.com/support/solutions/articles/31000179016-2026-season-3-initial-release-notes-2026-06-09-01-) ·
[2026 S2](https://support.iracing.com/support/solutions/articles/31000178217-2026-season-2-initial-release-notes-2026-03-09-03-) ·
[2026 S1](https://support.iracing.com/support/solutions/articles/31000177717-2026-season-1-initial-release-notes-2025-12-08-03-) ·
[2025 S4](https://support.iracing.com/support/solutions/articles/31000177148-2025-season-4-release-notes-2025-09-08-02-) ·
[2025 S1](https://support.iracing.com/support/solutions/articles/31000174324-2025-season-1-release-notes-2024-12-09-03-) ·
[2024 S4](https://support.iracing.com/support/solutions/articles/31000173869-2024-season-4-release-notes-2024-09-03-02-) ·
[2024 S3](https://support.iracing.com/support/solutions/articles/31000173378-2024-season-3-release-notes-2024-06-03-01-) ·
[2024 S3 P2](https://support.iracing.com/support/solutions/articles/31000173510-2024-season-3-patch-2-release-notes-2024-06-28-01-) ·
[2023 S2 P2 (ZBuffer32Bits)](https://support.iracing.com/support/solutions/articles/31000169554-2023-season-2-patch-2-release-notes-2023-03-20-02-) ·
[2022 S2 (trees rename)](https://support.iracing.com/support/solutions/articles/31000164646-2022-season-2-release-notes-2022-03-08-01-) ·
[2016 S4 (SMP)](https://www.iracing.com/2016-season-4-release-notes/)

**Dev updates**
[Feb 2026](https://www.iracing.com/iracing-development-update-february-2026/) ·
[May 2026](https://www.iracing.com/iracing-development-update-may-2026/) ·
[Aug 2025](https://www.iracing.com/iracing-development-update-august-2025/) ·
[Aug 2024](https://www.iracing.com/iracing-development-update-august-2024/)

**Hardware benchmarks**
[TechPowerUp — Ryzen 5 8500G](https://www.techpowerup.com/review/amd-ryzen-5-8500g/28.html) ·
[TechPowerUp — RTX 5090 PCIe scaling](https://www.techpowerup.com/review/nvidia-geforce-rtx-5090-pci-express-scaling/33.html) ·
[TechPowerUp — RTX 4090 PCIe scaling](https://www.techpowerup.com/review/nvidia-geforce-rtx-4090-pci-express-performance-scaling-with-core-i9-13900k/) ·
[TechPowerUp — ReBAR 22-game test](https://www.techpowerup.com/review/nvidia-pci-express-resizable-bar-performance-test/) ·
[TechSpot — PCIe bandwidth & VRAM](https://www.techspot.com/review/2396-pcie-bandwidth-test/) ·
[TechSpot — RX 6500 XT](https://www.techspot.com/review/2398-amd-radeon-6500-xt/) ·
[GamersNexus — RTX 5090 PCIe gen scaling](https://gamersnexus.net/gpus/nvidia-rtx-5090-pcie-50-vs-40-vs-30-x16-scaling-benchmarks) ·
[Phoronix — Zen 4 vs Zen 4c scaling](https://www.phoronix.com/review/amd-zen4-zen4c-scaling) ·
[PCGamesN — Ryzen 8000 PCIe lanes](https://www.pcgamesn.com/amd/ryzen-8000-pcie-lanes) ·
[Microsoft DirectX — CASO](https://devblogs.microsoft.com/directx/optimizing-hybrid-laptop-performance-with-cross-adapter-scan-out-caso/)

**Community**
[SimRacingCockpit CPU/GPU deep dive (May 2026)](https://simracingcockpit.gg/iracing-cpu-gpu-technical-deep-dive/) ·
[hone.gg settings guide (Aug 2025)](https://hone.gg/blog/iracing-graphics-settings/) ·
[Ehrling VR optimization guide](https://www.atlanticmotorsport.com/tutorials/iracing/iracing-vr-optimization-guide/) ·
[chen.do — stuttering with overlays/G-Sync/triples](https://chen.do/fixing-stuttering-in-iracing-with-overlays-g-sync-and-triple-monitors/) ·
[OverTake — X3D CPUs for sim racing](https://www.overtake.gg/news/comparing-amd-x3d-cpus-which-works-best-for-sim-racing.4622/)

*Note: forums.iracing.com requires login and Reddit blocks automated access; community
consensus above is drawn from guides that themselves cite forum threads and staff posts.*
