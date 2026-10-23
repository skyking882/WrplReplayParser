# Modifications in this fork (branch `wt-overlay`)

This branch is LivingTheDagor/WrplReplayParser at `221be5cac8e772ab41eefb0e7dac35ce6fb5c39b` (upstream `develop`) plus
the commits listed below. It is the replay backend of WT_overlay (`wt_overlay/wrpl/`), which freezes the Python module
into a standalone `wrpl_export` executable.

Copyright and licensing are unchanged: the project stays under the upstream BSD 3-Clause license in `LICENSE`, code
derived from Gaijin Entertainment's Dagor Engine stays under the Dagor Engine BSD 3-Clause license, and every bundled
third-party library keeps its own license file. None of those files was edited. Our changes are offered under the same
BSD 3-Clause terms as the upstream project.

## Changes

Commits 1-12 are produced by the patch scripts that WT_overlay previously ran on a clean checkout before building, in
the same order, one commit per script; the commit message is the script's docstring. Applying the 12 scripts to
`221be5c` gives exactly the tree of commit 12.

| # | Change | Files | Why |
|---|---|---|---|
| 1 | Normalize the letter case of `#include` paths to the real file names | Dagor `delta/`, `netutils/`, `replay/`, `python/`, `src/BitStream` | Upstream spells some includes with Windows-only case; Linux and case-sensitive file systems fail. No logic change |
| 2 | Add `ecs.clearPythonCallbacks()` | `python/ecs/ECSQueryES.cpp` | Registered Python callbacks were released by a C++ static destructor after CPython had shut down (SIGSEGV at exit). Only the callable references are cleared, while Python is alive |
| 3 | Add `mountLocalPackage()` and CDK include resolution | `python/state/State.cpp` | Read-only mount of the game's `content/pkg_local`, needed for replays with custom (CDK) units |
| 4 | Use the count-and-value `insert` overload when expanding text BLK includes | `3rdParty/Dagor_Engine/ioSys/datablock/blk_parser.cpp` | `insert(pos, n)` on `std::vector<char>` inserted one byte, then `n` bytes were written: heap overflow |
| 5 | Default `hasWeaponSlots` to "the merged BLK has a `WeaponSlots` block" | `replay/ecs/Unit.cpp` | Custom units are absent from the stock `wpcost` index, so their slotted loadouts were never built. An explicit `hasWeaponSlots` value is still used as before |
| 6 | Weapon field probe: `enableWeaponFieldProbe()` / `getWeaponFieldProbe()` | `replay/mpi/PositionSync.cpp`, `python/state/State.cpp` | Keep the WeaponSync fields the parser reads and discards. Decoding unchanged |
| 7 | Keep projectile creation fields and the raw bits of each successful WeaponSync | `replay/mpi/PositionSync.cpp`, `python/mpi/unit.cpp`, `python/state/State.cpp` | Raw data for the missile seeker block analysis (`getWeaponFieldProbeHex()`, `Rocket.creation_raw_fields`). Decoding unchanged |
| 8 | Register typed converters for primitive ECS scalars | `python/ecs/ECSTypes.cpp` | Primitive component values could not be read from Python |
| 9 | FM sync probe: `enableFmProbe()`, `getFmProbe()`, `getFmVelocity()` | `replay/mpi/PositionSync.cpp`, `python/state/State.cpp` | Keep the aircraft sync fields (sensors, countermeasures, target designations, packed velocity) that FMSync decodes and drops. Decoding unchanged |
| 10 | Unit MPI probe: `enableUnitMpiProbe()`, `getUnitMpiProbe()` | `replay/mpi/mpi.cpp`, `python/state/State.cpp` | List the per-unit MPI messages the parser skips (id, size, first 24 bytes). Nothing new is decoded |
| 11 | Bind the remaining `KillMessage` fields | `python/mpi/battle_messages.cpp` | Death type, weapon type, weapon flags and the unnamed extra fields were decoded but not exposed |
| 12 | macOS build: compile enet `unix.c` on Apple; drop `-static-libgcc` / `-static-libstdc++` | `3rdParty/enet/CMakeLists.txt`, `python/main/CMakeLists.txt` | Apple clang rejects the static flags and enet was missing its Unix sources. Build only |
| 13 | Delete `3rdParty/regExp` | `3rdParty/regExp/` | Digital Mars code marked "All Rights Reserved" with an empty `NO_LICENSE` file, so it cannot be redistributed. It was not built (`# add_subdirectory(regExp)` in `3rdParty/CMakeLists.txt`) and nothing includes it. `LICENSE` still lists it; that file is left as upstream wrote it |
| 14 | This file | `MODIFICATIONS.md` | |
| 15 | arm64 build: define `_TARGET_SIMD_NEON` instead of `_TARGET_SIMD_SSE` on arm64 / aarch64, compile with `-ffp-contract=off`, and turn FP contraction off after `float_control(precise, on)` in `netUtils.cpp` on clang/aarch64 | `cmake/functions.cmake`, `3rdParty/Dagor_Engine/netutils/netUtils.cpp` | Native Apple Silicon build through Dagor's NEON vecmath. Without FMA contraction the arm64 exports are bit-identical to the x86_64 ones (checked on four 2026-10 replays); with it, positions differed by 1-2 float ULP. x86_64 and MSVC builds are unchanged |

The probes (6, 7, 9, 10) are off unless the caller enables them. Changes 4 and 5 are bug fixes that can change parse
results; all other changes either add read-only accessors or touch the build only.

Further platform fixes (e.g. MSVC) go into separate commits on this branch and must not change parsing logic.
