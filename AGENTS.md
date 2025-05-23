# AGENTS.md – CodeX Autonomous Agents

*Project:* **Dora Biohazard Engine** – a 2‑D RPG framework inspired by *Nobita’s Resident Evil* (野比大雄の生化危機)
*Language & Tech:* Modern **C++23**, **Vulkan 1.3** renderer, hot‑reloadable **Lua 5.4** scripting

---

## 1 · Mission Statement

Create a lean, data‑oriented game engine that delivers 16‑bit JRPG charm with modern pipelines, maintaining **≤ 1 ms/frame CPU budget** on baseline desktop hardware and **≤ 8 ms** on Steam Deck–class devices.

---

## 2 · Design Tenets

1. **Deterministic & Cache‑Friendly:** Favor plain structs, SoA layouts, and fixed‑step logic.
2. **Zero‑Cost Abstractions:** Use templates, `constexpr`, and coroutines without heap penalties.
3. **Lua‑First:** Every gameplay layer (maps, events, AI, UI) must be scriptable and hot‑reloadable.
4. **Single Render Backend:** Vulkan is the one‑true path; platform shims live behind a thin HAL.
5. **Editor ≈ Game:** Tools run the same runtime DLL to catch regressions instantly.

---

## 3 · Agent Index

| ID  | Agent           | Primary Role                                    | Key Outputs                                        |
| --- | --------------- | ----------------------------------------------- | -------------------------------------------------- |
| A‑1 | **CodexBuild**  | Build system, CI, cross‑compile tool‑chains     | Ninja+Clang presets, Docker images, GitHub Actions |
| A‑2 | **CodexRender** | Vulkan renderer scaffolding                     | Swap‑chain, sprite batcher, post‑FX graph          |
| A‑3 | **CodexECS**    | Entity‑Component‑System core & gameplay systems | Header‑only registry, system scheduler             |
| A‑4 | **CodexLua**    | Lua VM, C++↔Lua bridge, script hot‑reload       | `lua_bindings.cpp`, auto‑generated docs            |
| A‑5 | **CodexAsset**  | Offline/online asset pipeline                   | CLI importers, `.cdbin` caches, live‑reload hooks  |
| A‑6 | **CodexAudio**  | Low‑latency audio playback                      | Ogg/Vorbis streamer, SFX mixer, BGM bus            |
| A‑7 | **CodexEditor** | ImGui/Qt hybrid editor                          | Map painter, dialogue graph, profiler panels       |
| A‑8 | **CodexTest**   | Unit, integration & perf testing                | Catch2 suites, RenderDoc captures, flamegraph diff |
| A‑9 | **CodexDoc**    | Living documentation generator                  | Doxygen site, MD guides, API JSON index            |

---

## 4 · Agent Templates

Each agent lives in `/agents/<name>/agent.toml` describing its contract.

```toml
[meta]
id          = "CodexRender"
description = "Owns all Vulkan objects, frame graph, and GPU diagnostics."

[requires]
modules = ["CodexBuild"]

[provides]
headers = ["include/codex/render/*.hpp"]
libs    = ["libcodex_render.a"]
```

> **Rule:** Agents may **only** communicate via *declared* outputs; no hidden headers or global singletons.

---

## 5 · Agent Responsibilities

### A‑1 CodexBuild

* Generates CMake + Ninja files for host and target triplets (Win64, Linux‑x86‑64, Steam Deck, macOS‑ARM64).
* Emits GitHub Actions YAML with cache‑friendly matrices.
* Owns `conanfile.py` to distribute 3rd‑party deps (Vulkan‑SDK, Lua, miniaudio).

### A‑2 CodexRender

* Sets up Vulkan 1.3 instance, device, descriptor pools.
* Provides *sprite batcher*, *tilemap renderer*, and *post‑processing graph*.
* Exposes `Render::submit(lua_State*, RenderCommand&)` so Lua can queue draw calls.
* Collects GPU timestamps and publishes per‑stage MS in ImGui overlay.

### A‑3 CodexECS

* Header‑only `Registry` with *struct‑of‑arrays* storage.
* Job‑system powered system scheduler leveraging `std::jthread` and work‑stealing.
* Provides reflection info for CodexLua to auto‑bind components.

### A‑4 CodexLua

* Embeds Lua 5.4 with all integer math (`lua_Integer` = 64‑bit).
* Binds C++ through *sol3*‑generated glue; hot‑reloads changed files via `inotify`/`ReadDirectoryChangesW`.
* Guarantees **≤ 0.1 ms** script dispatch for trivial events.

### A‑5 CodexAsset

* CLI `codex-import` converts PNG → texture atlas, TTF → distance‑field font, OGG → ADPCM stream.
* Produces `.cdbin` (flatbuffer‑like) with CRC checksum for quick mismatch detection.
* Live‑reload via `AssetWatcher` broadcasting to ECS.

### A‑6 CodexAudio

* Wraps **miniaudio** for cross‑platform stability.
* Two threads: *decode* (Vorbis → PCM) & *mix* (high‑priority RT).
* Lua API: `play_bgm(id, loop)`, `play_sfx(id, pan, pitch, vol)`.

### A‑7 CodexEditor

* Single‑executable editor that links CodexRender/ECS/Lua
* Panels: TileMap, Event Graph (node‑based), Asset Browser, Profiler.
* Saves user content into `data/` tree consumed directly at runtime.

### A‑8 CodexTest

* Uses **Catch2** for logic tests, **RenderDoc** for frame regression.
* Runs on every PR; fails > 5 % perf deviation.

### A‑9 CodexDoc

* Scrapes headers to build a searchable API site with dark‑mode.
* Pushes docs preview to GitHub Pages on branch.

---

## 6 · Agent Interaction Protocol (AIP)

1. **Request–Reply JSON:** Agents communicate via `/tmp/aip/<from>_<to>.json` files.
2. **Versioning:** Every message includes `schema_version` (`semver`).
3. **Timeouts:** Sender kills the request if no reply in 10 s (CI) or 30 s (local dev).
4. **Logging:** All agents log to UTF‑8 rotating files in `.logs/<agent>.log`.

---

## 7 · Performance SLA

| Metric             | Target                            | Measured by                 |
| ------------------ | --------------------------------- | --------------------------- |
| CPU frame budget   | ≤ 1 ms (PC) / ≤ 8 ms (Steam Deck) | `CodexTest` flamegraph diff |
| Lua event dispatch | ≤ 0.1 ms                          | `CodexLua` microbench       |
| VRAM usage         | ≤ 256 MB for 1080p                | GPU perf overlay            |
| Import pipeline    | ≤ 200 ms/MB                       | `CodexAsset` CI run         |

Failures gate merge until regressions are fixed or waiver issued.

---

## 8 · Contributing

1. Fork, create feature branch.
2. Run `./tools/setup.sh` to fetch deps.
3. Implement feature + unit test.
4. `./scripts/format.sh` (clang‑format, stylua).
5. Submit PR; agents post checks.

---

## 9 · Roadmap Snapshot (Q3 2025)

* [ ] Finish Vulkan frame graph (A‑2)
* [ ] Ship Lua‑driven cut‑scene system (A‑4)
* [ ] Add nav‑mesh & pathfinding (A‑3)
* [ ] Publish playable demo map in Editor (A‑7)

---

**End of AGENTS.md**
