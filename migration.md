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

# Decision Log
- 2026-02-07: Milestone locked to `Core + Major Compat`.
- 2026-02-07: Dependency strategy locked to `Latest 1.21.1 Versions`.
- 2026-02-07: Track migration state in `migration.md` as source of truth.
- 2026-02-07: Reused old backport (`9dbb8174`) as a compatibility base for Java/resource files, then reconciled with current branch constructor/signature differences.
- 2026-02-07: Removed 1.21.11-only Sodium config API classes (`SodiumConfigBuilder`, `VoxyConfigMenu`) and switched ModMenu config integration to Sodium 0.6.x UI path.

# Open Issues
- Full in-world Voxy rendering validation is blocked on this machine due to OpenGL capability (`4.1 Metal`) and Voxy’s runtime support gate.
- Nvidium runtime path cannot be validated on this machine because Nvidium disables itself when requirements are unmet.
- Apple Silicon/macOS support would require a new compatibility backend (non-compute/non-MDIC and limited DSA assumptions) or running on an environment exposing modern OpenGL/Vulkan features.
- `winbe` copy automation depends on SMB authentication/share details (or pre-mounted Finder share) before first sync can run.
- Pending verification: rerun Windows test after Iris missing-uniform log-level change and confirm no intrusive popup when shaderpack lacks optional uniforms.
- Backport intentionally disables/removes newer debug entry integration paths that rely on unavailable 1.21.11 debug APIs.
