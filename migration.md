# Objective
Backport `voxy` from Minecraft `1.21.11` to `1.21.1` on branch `codex/backport-1.21.1`, targeting first-milestone compatibility for core rendering and major integrations (Sodium, Iris, Nvidium).

# Scope Decisions
- Scope: `Core + Major Compat`
- Dependency policy: `Latest 1.21.1 Versions`
- Old backport (`9dbb8174`) is a reference, not a direct cherry-pick target.

# Baseline State
- Repo: `/Users/saejin/Projects/personal/voxy`
- Branch at start: `dev`
- Backport branch: `codex/backport-1.21.1`
- Baseline commit on branch creation: `13230c27`
- Baseline compile (before backport edits): `./gradlew --no-daemon --no-build-cache clean compileJava --rerun-tasks` => `BUILD SUCCESSFUL`
- Observed timing note: warm/local cached runs are often short (~10s), but first cold setup/build can be much slower (user-observed ~2 minutes).

# Reference Backport
- Reference repo: `/Users/saejin/Projects/personal/voxy-other-port`
- Reference commit: `9dbb8174` (`backport to 1.21.1`)
- File stats versus current `dev` shape:
  - touched by old backport: `41`
  - exact pre-backport matches in current repo: `10`
  - diverged files: `28`
  - missing files: `3`

# Dependency Matrix
| Component | Before (1.21.11 branch) | Target (1.21.1 backport) | Status |
|---|---|---|---|
| Minecraft | 1.21.11 | 1.21.1 | done |
| Fabric Loader | 0.18.2 | 0.18.4 | done |
| Loom | 1.14-SNAPSHOT | 1.14-SNAPSHOT (unless blocked) | done |
| Fabric API | 0.140.2+1.21.11 | 0.116.8+1.21.1 | done |
| Sodium | mc1.21.11-0.8.4-fabric | mc1.21.1-0.6.13-fabric | done |
| Lithium | mc1.21.11-0.21.0-fabric | mc1.21.1-0.15.2-fabric | done |
| Iris | 1.10.5+1.21.11-fabric | 1.8.8+1.21.1-fabric | done |
| ModMenu | 17.0.0-alpha.1 | 11.0.3 | done |
| Chunky | 1.4.54-fabric | 1.4.23-fabric | done |
| Sodium Extra | not pinned for 1.21.11 | mc1.21.1-0.6.0+fabric | done |
| Nvidium | 0.4.1-beta4:1.21.6 | 0.4.1-beta9:1.21-1.21.1 | done |

# API Delta Matrix
| Area | 1.21.11-side usage | 1.21.1 target delta | Status |
|---|---|---|---|
| Sodium fog pipeline | `FogParameters` path in renderer/mixins | remove/adapt for Sodium 0.6.x signatures | done |
| Sodium renderer signatures | `DefaultChunkRenderer.render(..., FogParameters, ..., terrainSampler)` | 0.6.x reduced signatures | done |
| Nvidium hook | `renderFrame(... FogParameters ...)` signature | 1.21.1 signature without fog param | done |
| Render layer types | `ChunkSectionLayer` callsites | use 1.21.1-compatible `RenderType` flow | done |
| FogRenderer mixin target | `net.minecraft.client.renderer.fog.FogRenderer` style | 1.21.1 `net.minecraft.client.renderer.FogRenderer` style | done |
| Config integration | Sodium config API (`net.caffeinemc.mods.sodium.api.config`) | fallback to 0.6.x compatible config UI path | done |
| Access widener entries | 1.21.11 descriptors | update to 1.21.1 descriptors | done |

# Workstreams
- [x] W1 `done` Dependency/version pivot (`gradle.properties`, `build.gradle`, `fabric.mod.json`)
- [x] W2 `done` Metadata + entrypoint compatibility (`fabric.mod.json`, config entrypoints)
- [x] W3 `done` High-confidence replay from old backport files
- [x] W4 `done` Render/fog/signature workstream
- [x] W5 `done` Model/render-layer workstream
- [x] W6 `done` World/import API workstream
- [x] W7 `done` Debug/config integration workstream
- [x] W8 `done` Mixin/AW/resource workstream
- [x] W9 `done` Compile stabilization loop
- [ ] W10 `blocked` Runtime smoke validation (client boot verified; full in-world Voxy validation blocked on local GPU capability)

# Risk Register
- Sodium 0.6.x API differences are extensive and can cascade across mixins.
- Debug entry APIs differ; F3/debug integration may require temporary disable.
- Mod dependency availability on 1.21.1 may differ from 1.21.11 stack.
- Nvidium jar coordinate/repository reliability may vary by environment.

# Verification Log
- Baseline: `./gradlew --no-daemon --no-build-cache clean compileJava --rerun-tasks` => `BUILD SUCCESSFUL` (pre-backport).
- Pivot compile check: `./gradlew --no-daemon --no-build-cache clean compileJava --rerun-tasks` => `BUILD FAILED` with broad 1.21.1 API drift (`Identifier`, Sodium API, FogParameters/debug classes).
- Replay/adaptation compile checks: iterative `./gradlew --no-daemon compileJava` loops used to reduce error set from 100+ to 0 compile errors.
- Final acceptance compile: `./gradlew --no-daemon --no-build-cache clean compileJava --rerun-tasks` => `BUILD SUCCESSFUL`.
- Final changed-file inventory captured via `git status --short` on branch `codex/backport-1.21.1` (includes tracked updates, deletions, and new compatibility files).
- Runtime boot check: `./gradlew --no-daemon runClient` launched Minecraft `1.21.1` successfully (Fabric loader/mod init, render thread, resource reload, audio init, atlas creation) with no fatal mixin apply failures observed in `run/logs/latest.log`.
- Runtime caveats from same launch:
  - `Voxy is unsupported on your system` on Apple M1 Pro/OpenGL `4.1 Metal - 90.5` (prevents validating active Voxy rendering path on this host).
  - `Nvidium` self-disabled due unmet requirements on this hardware/driver stack.
  - Optional Flashback mixin targets missing (`ClassNotFoundException`), non-fatal for base startup.
- Capability diagnostics run (2026-02-08): unsupported gate fails due missing `compute-dispatch-indirect` and `arb_indirect_parameters`; snapshot on Apple M1 Pro reported missing function pointers for `glDispatchCompute`, `glDispatchComputeIndirect`, `glMultiDrawElementsIndirectCountARB`, and core DSA calls (`glCreateBuffers`, `glNamedBufferStorage`, `glBindTextureUnit`, `glCreateFramebuffers`).
- Windows offload prep (2026-02-08): `./gradlew --no-daemon build` => `BUILD SUCCESSFUL`; network probe to `winbe` confirmed SMB reachable (`445/139`) while SSH/WinRM unavailable; helper scripts were used locally for transfer/log collection during validation.
- Windows runtime crash analysis (2026-02-08): crash on world join traced to Voxy vertex shader compile failure (`undefined variable "modelHasMipmaps"` in `lod/gl46/quads2.vert`). Fixed by adding compatibility alias `modelHasMipmaps(...)` in `assets/voxy/shaders/lod/block_model.glsl`; local `./gradlew --no-daemon build` passes after patch.
- Windows re-test (log bundle `artifacts/winbe-logs/20260207-202312`): world join succeeds with Voxy renderer active (`NormalRenderPipeline` + `MDICSectionRenderer`), Nvidium enabled, and no crash report generated.
- Iris shaderpack popup diagnosis (same run): enabling BSL shaderpack triggered Voxy log `The following uniforms could not be found: [endFlashIntensity]` (non-fatal compatibility mismatch). Adjusted `IrisVoxyRenderPipelineData` to log this as warning instead of error to avoid intrusive popup while still recording the mismatch.
- Added reproducible Chunky pregen procedure for validation runs: configure Chunky with `continueOnRestart=true` and execute a 64-chunk circular region task (`/chunky spawn`, `/chunky shape circle`, `/chunky radius 64`, `/chunky pattern concentric`, `/chunky start`).
- Shaderpack compatibility research (2026-02-08):
  - Photon `v1.2a` (Modrinth `rz2vlXVm`, 2025-06-29) does not contain `shaders/program/voxy.json` or `voxy_*.glsl`, so release builds on that tag cannot activate Voxy-specific Iris patching.
  - Photon `main` branch zip (`https://github.com/sixthsurge/photon/archive/refs/heads/main.zip`) includes full Voxy integration files: `shaders/program/voxy.json`, `shaders/program/voxy_opaque.glsl`, `shaders/program/voxy_translucent.glsl`, world wrappers, and `shaders/include/misc/lod_mod_support.glsl` with `VOXY` paths (`vxDepthTex*`, `vxProj*`, `vxRenderDistance`).
  - Photon upstream issue `#512` (`Render Error With Voxy`) includes maintainer guidance dated 2026-02-06 to use `main` instead of `1.2a` for Voxy compatibility.
  - BSL `v10.1.1` (Modrinth `NDEQ77pU`, 2026-02-07) already ships full Voxy patch files (`voxy.json`, `voxy_opaque.glsl`, `voxy_translucent.glsl`) in `program/` plus `world-1/world0/world1` includes.
  - BSL `v10.1` and `v10.1.1` share identical `voxy.json`; only `voxy_opaque.glsl` and `voxy_translucent.glsl` changed (alpha-guard logic adjustment), indicating ongoing shader-side Voxy fixes independent of Voxy mod code.
- External launcher crash analysis (2026-02-08): third-party run failed on `MixinMinecraft` with `InvalidInjectionException` targeting `disconnect(...;ZZ)V`. Root cause: stale `1.21.11` signature in `src/main/java/me/cortex/voxy/client/mixin/minecraft/MixinMinecraft.java`; `1.21.1` exposes `disconnect(Screen, boolean)` / `disconnect(Screen)` instead. Patched mixin to target both `1.21.1` signatures with `require = 0`, rebuilt via `./gradlew --no-daemon remapJar`, and verified remapped output now targets intermediary `method_18096(Lnet/minecraft/class_437;Z)V` and `method_56134(Lnet/minecraft/class_437;)V` without remap warnings.
- External launcher crash follow-up (2026-02-07 21:05:28 on `winbe`): crash report `/Users/saejin/crash-2026-02-07_21.05.28-client.txt` indicates `NullPointerException` in `team.creative.ambientsounds.engine.AmbientEngine.fastTick` (`soundEngine` null) during Fabric end-tick event. No `me.cortex.voxy` frames appear in the exception chain; this failure is attributable to AmbientSounds/CreativeCore in the tested pack, not Voxy.
- Border flicker mitigation pass (2026-02-08): added a short chunk-bound removal grace window (`REMOVE_GRACE_FRAMES=4`) in `ChunkBoundRenderer` to prevent one-frame Voxy/vanilla boundary flapping during transient Sodium section rebuild transitions; also added a small conservative depth epsilon (`+0.00002`) in `assets/voxy/shaders/lod/gl46/quads.frag` depth-bound discard test to reduce precision jitter at transition edges. Rebuilt with `./gradlew --no-daemon remapJar` (`BUILD SUCCESSFUL`).
- Early occlusion culling mitigation pass (2026-02-08): made hierarchical Hi-Z culling in `assets/voxy/shaders/lod/hierarchical/screenspace.glsl` more conservative by adding mip-scaled depth bias (`hizDepthBias = 0.00025 + ml * 0.00015`) and guarding invalid samples (`pointSample < 0 => visible`) before occlusion compare. Rebuilt with `./gradlew --no-daemon remapJar` (`BUILD SUCCESSFUL`).

# Decision Log
- 2026-02-07: Milestone locked to `Core + Major Compat`.
- 2026-02-07: Dependency strategy locked to `Latest 1.21.1 Versions`.
- 2026-02-07: Track migration state in `migration.md` as source of truth.
- 2026-02-07: Reused old backport (`9dbb8174`) as a compatibility base for Java/resource files, then reconciled with current branch constructor/signature differences.
- 2026-02-07: Removed 1.21.11-only Sodium config API classes (`SodiumConfigBuilder`, `VoxyConfigMenu`) and switched ModMenu config integration to Sodium 0.6.x UI path.
- 2026-02-08: Photon shaderpack support baseline is `main` branch Voxy files (not Modrinth `v1.2a`); treat `main` Voxy assets as reference for any local patching or downstream support docs.
- 2026-02-08: Keep missing Iris shader uniforms non-fatal for shaderpack interoperability; maintain warn-level logging for unresolved optional uniforms (for example `endFlashIntensity` in tested BSL run).
- 2026-02-08: `MixinMinecraft` world-close hook must be version-tolerant on `1.21.1`; avoid `1.21.11`-specific disconnect descriptors.

# Open Issues
- Full in-world Voxy rendering validation is blocked on this machine due to OpenGL capability (`4.1 Metal`) and Voxy’s runtime support gate.
- Nvidium runtime path cannot be validated on this machine because Nvidium disables itself when requirements are unmet.
- Apple Silicon/macOS support would require a new compatibility backend (non-compute/non-MDIC and limited DSA assumptions) or running on an environment exposing modern OpenGL/Vulkan features.
- `winbe` copy automation depends on SMB authentication/share details (or pre-mounted Finder share) before first sync can run.
- Pending verification: rerun Windows test after Iris missing-uniform log-level change and confirm no intrusive popup when shaderpack lacks optional uniforms.
- Backport intentionally disables/removes newer debug entry integration paths that rely on unavailable 1.21.11 debug APIs.
- Photon Modrinth release cadence may lag Voxy-support branch state; users on Photon `v1.2a` may report incompatibility until a newer release containing `voxy.*` files is published.
- External modpack instability (AmbientSounds + CreativeCore) can cause world-load crashes independent of Voxy; keep shader/Voxy validation runs on a minimized dependency stack when triaging Voxy-specific issues.
