# Project History

## 2026-10-09 - Initial Analysis & Planning Phase

### Actions Taken
- Analyzed workspace and inspected `Polyrhythm_Synth_v5.fmx` (size: 502,761 bytes).
- Reverse-engineered the `.fmx` Google Flow Music format:
  - Discovered top-level JSON container: `version`, `versionId`, `spaceId`, `spaceVersion`, `name`, `type`, `html` (gzipped base64), `wasm` (gzipped base64), `metadata` (JSON parameter specifications), and `signature`.
  - Extracted and decompressed the WebAssembly DSP module (`18,098` bytes).
  - Reverse-engineered WebAssembly ABI Version 1:
    - Imports: `env.abort`, `env.hostExposeBuffer`.
    - Exports: `init(sampleRate: f32)`, `getNumParams() -> i32`, `noteOn(note, velocity)`, `noteOff(note)`, `reset()`, `process(inL, inR, outL, outR, numFrames)`, `getAbiVersion()`, `memory`.
    - Discovered memory parameter table at fixed offset `0x10000` (`65536`) with 32-bit denormalized floats for each parameter ID (`65536 + id * 4`).
    - Successfully verified audio DSP synthesis: simulated `noteOn(60, 100)` and rendered 4 blocks of 512 samples with real-time polyrhythmic synthesis and telemetry buffer reporting.
  - Decompressed and inspected the Web UI single-page application (`1,977,316` bytes uncompressed HTML/JS/CSS).
  - Reverse-engineered the iPlug2/FlowMusic communication bridge:
    - Host-to-UI functions: `window.SPVFD(paramIdx, value)`, `window.__vstOnState(state)`, `window.__vstOnTransport(transport)`, `window.__vstOnProcessorBuffers(buffers)`, `window.__vstOnCustomState(data)`.
    - UI-to-Host messages via `window.IPlugSendMsg`: `SPVFUI` (param value changed), `BPCFUI` / `EPCFUI` (gesture begin/end), `vstGetCustomState`, `vstGetTransport`, `resize`.
- Assessed host environment:
  - Windows x86_64, GCC 15.2.0 (`g++`, `gcc`, `make`), Node.js v26.11.1, Scoop package manager.
  - Pre-installed Microsoft Edge WebView2 runtime confirmed at `C:\Program Files (x86)\Microsoft\EdgeWebView\Application`.
- Formulated complete architectural and implementation plan for the Windows VST Plugin Wrapper.

## 2026-10-09 - Plan Refinement (CLAP Support & FMX Embedding)

### Actions Taken
- Incorporated user specifications into the master architectural plan:
  1. **CLAP Plugin Format**: Added native CLAP target support alongside VST3 and Standalone executable using JUCE/CLAP extensions.
  2. **Full .FMX Project Embedding**: Designed robust project session chunk serialization with compressed `.fmx` embedding, ensuring portable session recall even if external files are renamed or moved, plus an "Export Embedded .fmx" UI feature.
- Updated master plan artifact (`plan_fmx_windows_vst_wrapper.md`).

## 2026-10-09 - Implementation of Phase 1 (Core DSP Engine) & Phase 2 (Package & History)

### Actions Taken
- Setup build toolchain: CMake 4.4.4, Ninja 1.13.2, GCC 15.2.0 (MinGW-w64).
- Added third-party dependencies:
  - `wasm3` (embedded C WebAssembly runtime) in `third_party/wasm3`
  - `miniz` (zlib/deflate/gzip decompressor) in `third_party/miniz`
  - `json.hpp` (nlohmann json) in `third_party/json`
  - `JUCE` 8 framework in `third_party/JUCE`
  - `clap-juce-extensions` with CLAP submodules in `third_party/clap-juce-extensions`
- Implemented `include/ParamMap.hpp`:
  - Parameter curves: Linear, Exponential, Squared, Cubed, Sqrt, Cbrt.
  - Normalization and denormalization algorithms matching `.fmx` runtime math.
- Implemented `include/WasmEngine.hpp` and `src/WasmEngine.cpp`:
  - Embedded `wasm3` engine integration.
  - Export bindings: `init`, `process`, `noteOn`, `noteOff`, `reset`, `getNumParams`, `getAbiVersion`.
  - Host import bindings: `env.abort` and `env.hostExposeBuffer`.
  - Memory parameter mapping at `0x10000` (65536) and audio buffer dispatching at `0x20000` (131072).
  - Telemetry visualizer buffer extraction.
- Implemented `include/FmxPackage.hpp` and `src/FmxPackage.cpp`:
  - JSON container parser, Base64 decoder, Gzip decompressor with header skipping.
  - Portable SHA-256 calculation.
  - Binary `FMXW` DAW chunk serialization with zlib compression for project state embedding.
  - Automatic fallback hierarchy: checks local file, then restores from embedded payload.
- Implemented `include/HistoryManager.hpp` and `src/HistoryManager.cpp`:
  - Persistent recent plugins list stored in `%APPDATA%\FlowMusicWrapper\recent_plugins.json`.
- Implemented and executed `tests/test_dsp.cpp`:
  - Verified package extraction on `Polyrhythm_Synth_v5.fmx`.
  - Verified real-time synthesis of 4 blocks of 512 samples with peak Left: 0.2289, Right: 0.2458.
  - Verified telemetry buffer reading (8 floats).
  - Verified DAW chunk embedding roundtrip (502 KB -> 380 KB compressed chunk -> restored cleanly).
  - All automated test assertions passed.

## 2026-10-10 - Implementation of Phase 3 (UI & Bridge), Phase 4 (CLAP & Standalone), and VST3 Wrapping

### Actions Taken
- Implemented `include/WebViewHost.hpp` and `src/WebViewHost.cpp`:
  - Native Win32 window hosting Microsoft Edge WebView2 (`ICoreWebView2Controller`, `ICoreWebView2`).
  - Custom glassmorphic top navigation bar with:
    - Flow Music brand indicator and active plugin status pill (showing loaded name & embedded status).
    - "Load .fmx..." file dialog button (`GetOpenFileNameA`).
    - "Recent v" popup menu integrated with `HistoryManager` and "Clear History" option.
    - "Options v" popup menu with "Export Embedded .fmx to File...", "Reload Current Plugin", and "About".
    - "Panic" button (All Notes Off / DSP engine reset).
  - HTML drag-and-drop landing card for `.fmx` files when no plugin is loaded.
  - High-performance two-way JS bridge:
    - Host-to-UI parameter updates (`window.SPVFD`), transport updates (`window.__vstOnTransport`), and visualizer telemetry buffer dispatch (`window.__vstOnProcessorBuffers`).
    - UI-to-Host messages (`SPVFUI`, `OPEN_FILE_DIALOG`, `LOAD_EXAMPLE`, `FILE_DROPPED`).
- Implemented `src/ClapPlugin.cpp`:
  - Native CLAP plugin implementing `clap_plugin_t`, `clap_plugin_entry_t`, and `clap_plugin_factory_t`.
  - Extensions implemented:
    - `CLAP_EXT_GUI` (Win32 HWND embedding, resize notifications).
    - `CLAP_EXT_PARAMS` (DAW parameter automation, curve mapping).
    - `CLAP_EXT_STATE` (Full `.fmx` container DAW session chunk embedding & recall).
    - `CLAP_EXT_AUDIO_PORTS` (Stereo input/output).
    - `CLAP_EXT_NOTE_PORTS` (MIDI noteOn, noteOff, and transport event handling).
- Implemented `src/StandaloneMain.cpp`:
  - Standalone desktop player (`FlowMusicWrapper.exe`).
  - Multi-threaded low-latency Win32 `waveOut` audio callback engine with circular buffering.
  - Win32 MIDI Input device support (`midiInOpen`) routing hardware MIDI controllers directly to WebAssembly DSP.
  - 30 Hz timer for visualizer telemetry buffer dispatch to WebView2.
- Resolved MinGW-w64 COM UUID linkage:
  - Replaced MSVC-specific `__uuidof` operator with `memcmp(&riid, &IID_..., sizeof(GUID)) == 0` using explicit `WebView2.h` GUID constants (`IID_ICoreWebView2CreateCoreWebView2EnvironmentCompletedHandler`, `IID_ICoreWebView2CreateCoreWebView2ControllerCompletedHandler`, `IID_ICoreWebView2WebMessageReceivedEventHandler`, `IID_IUnknown`).
  - Added `uuid` library to CMake link targets.
- Built all primary targets:
  - `build/FlowMusicWrapper.clap` (CLAP plugin)
  - `build/FlowMusicWrapper.exe` (Standalone Windows app)
  - `build/test_dsp.exe` (DSP verification suite)
- Integrated Steinberg VST3 support via Free Audio Consortium `clap-wrapper`:
  - Configured and built `FlowMusicWrapper.vst3` bundle (`Contents/x86_64-win/FlowMusicWrapper.vst3`) with Steinberg VST3 SDK 3.8.0.
  - Exported standard VST3 factory symbols (`GetPluginFactory`, `InitDll`, `ExitDll`).
  - Copied VST3 bundle, CLAP binary, and WebView2Loader runtime into `build/`.

## 2026-10-10 - Root Cause Resolution: Audio Thread Safety, WebView2 Navigation, Self-Contained VST3 & Loader

### Issues Diagnosed & Resolved
1. **CLAP Host Load Failure in DAWs**:
   - *Root Cause*: `my_clap_process` in `src/ClapPlugin.cpp` was directly calling `ICoreWebView2::ExecuteScript` and allocating heap JSON memory from inside the real-time audio thread. In Windows COM STA, calling interface methods from a background audio callback violates COM apartment threading (`RPC_E_WRONG_THREAD` / crash), causing DAWs to fail during scanning or initialization.
   - *Fix*: Completely decoupled the audio thread from COM. Parameter changes from host automation are flagged via atomic bitfields (`guiParamDirty`) and visualizer telemetry buffers are read on a 30 Hz timer on the Win32 UI message loop (`onTimerTick`). The audio processing loop is now 100% lock-free and real-time safe.
2. **VST3 Not Detected by DAWs**:
   - *Root Cause*: The default VST3 wrapper dynamically called `findPlugin()` to locate a `.clap` binary on disk. Because the user copied only the `.vst3` file, `GetPluginFactory()` failed to find the CLAP file and returned `nullptr`. Steinberg DAW scanners reject any VST3 returning `nullptr` from `GetPluginFactory()`. Furthermore, MinGW created a `libFlowMusicWrapper.vst3` filename by default.
   - *Fix*: Reconfigured `clap-wrapper` with `STATICALLY_LINKED_CLAP_ENTRY=1`, linking `ClapPlugin.cpp`, `WebViewHost.cpp`, and `fmx_core` directly into the `.vst3`. `GetPluginFactory()` now returns a valid, non-null factory pointer with zero external file dependencies. Removed `lib` prefix from CMake output. Created both `FlowMusicWrapper.vst3` (bundle) and `FlowMusicWrapper_single_file.vst3`.
3. **Standalone App "Did Nothing" When Loading `.fmx`**:
   - *Root Cause*: `ICoreWebView2::NavigateToString` in Edge WebView2 has a strict 2 MB size limit and assigns an `about:blank` origin, which broke Three.js/WebGL contexts and failed on `Polyrhythm_Synth_v5.fmx` (1.98 MB HTML payload). Additionally, `GetOpenFileName` changed the process current working directory.
   - *Fix*: Extracted HTML to `%LOCALAPPDATA%\FlowMusicWrapper\ui_cache\<pluginName>.html` and navigated using file URI `file:///...`, giving full WebGL context, local storage, and removing size limits. Added `OFN_NOCHANGEDIR` to file dialog.
4. **Missing `WebView2Loader.dll` Dependency**:
   - *Root Cause*: When users copy the `.clap` or `.vst3` to plugin folders, `WebView2Loader.dll` is not in the host's directory or System32.
   - *Fix*: Generated `src/EmbeddedLoader.cpp` with embedded `WebView2Loader.dll` binary payload. On first launch, the wrapper automatically extracts the loader to `%LOCALAPPDATA%\FlowMusicWrapper\WebView2Loader.dll` if not present, and loads it via absolute path.
5. **Interactive Keyboard Musical Typing Added**:
   - Added QWERTY musical typing (`A, W, S, E, D, F, T, G, Y, H, U, J, K` -> C4 to C5) in `src/StandaloneMain.cpp` and bridge script so standalone users can immediately play sounds without needing a connected MIDI controller.
6. **Diagnostic & Verification Tools Built**:
   - Created `tests/test_clap_host.cpp` and `tests/test_vst3_host.cpp`.
   - Verified 100% pass across all 4 suites: `test_dsp.exe`, `test_clap_host.exe`, `test_vst3_host.exe` (bundle & single-file).

## 2026-10-10 - Native CLAP Compliance & Sandboxed Host Compatibility Resolution

### Issues Diagnosed & Resolved
1. **Illegal Audio Ports on Instrument**:
   - *Root Cause*: `clap_audio_ports_count(..., is_input)` was returning 1 for input ports. In the CLAP specification and standard hosts (especially Bitwig Studio), instrument plugins (`CLAP_PLUGIN_FEATURE_INSTRUMENT`) must NOT declare audio input ports (`count = 0`). Declaring an audio input port caused host scanners to reject the plugin as malformed/contradictory.
   - *Fix*: Set `clap_audio_ports_count(plugin, true)` to return `0` and `clap_audio_ports_get` to return `false` for inputs, keeping `Main Output` (stereo, 2 channels) as the sole audio port at index 0.
2. **GUI Floating Mode Rejection in Sandboxed Scanners**:
   - *Root Cause*: `clap_gui_is_api_supported` was returning `false` for `is_floating = true`. DAW out-of-process scanners (such as Bitwig's sandbox scanner and Reaper's dedicated process hosting) check or require floating window support.
   - *Fix*: Added full floating window support (`is_floating = true`) in `clap_gui_is_api_supported`, `clap_gui_create` (creating top-level HWND with `nullptr` parent), `clap_gui_set_transient`, and `clap_gui_suggest_title`.
3. **Null Pointer Crash on `on_main_thread` Callback**:
   - *Root Cause*: `plugin->on_main_thread` in `clap_plugin_t` was set to `nullptr`. When CLAP host scanners test callback dispatch or call `plugin->on_main_thread(plugin)`, calling a null function pointer triggered an immediate Access Violation exception (0x00000000), crashing the scanner sandbox.
   - *Fix*: Implemented `clap_on_main_thread(const clap_plugin_t*)` and assigned it to `plugin->on_main_thread`.
4. **Missing Latency & Render Extensions**:
   - *Root Cause*: DAWs querying plugin delay compensation (PDC) and rendering modes failed when `CLAP_EXT_LATENCY` was absent.
   - *Fix*: Implemented `CLAP_EXT_LATENCY` (`clap_plugin_latency_t` returning 0 latency) and `CLAP_EXT_RENDER` (`clap_plugin_render_t`).
5. **Embedded Default .fmx Payload (`Polyrhythm_Synth_v5.fmx`)**:
   - *Root Cause*: When copied alone to `C:\Program Files\Common Files\CLAP\FlowMusicWrapper.clap`, no companion `.fmx` existed in the DAW's working directory or next to the file, leaving the wrapper completely empty with 0 parameters and uninitialized DSP.
   - *Fix*: Generated `src/EmbeddedFmx.cpp` with embedded `Polyrhythm_Synth_v5.fmx` (502,761 bytes) compiled into `fmx_core`. The CLAP plugin now self-initializes with 10 automatable parameters and instant sound synthesis even if copied with zero companion files.
6. **Comprehensive Automated Test Validation**:
    - Updated `tests/test_clap_host.cpp` to validate audio port counts (0 in / 1 out), note ports, latency, floating and embedded GUI creation/destruction, and `on_main_thread` callback dispatch.
    - Verified 100% pass on `build/FlowMusicWrapper.clap` and isolated folder `build/test_isolated/FlowMusicWrapper.clap`.
    - Rebuilt `FlowMusicWrapper.vst3` bundle and single-file, verifying factory export with `test_vst3_host.exe`.

## 2026-10-10 08:50 - Audio Effect Support (`vst_effect`), CLAP Stereo Input Routing, Dynamic UI Refresh & Recall

### Issues Diagnosed & Resolved
1. **Audio Effects Failed to Load (`Export 'noteOn' not found`)**:
   - *Root Cause*: `WasmEngine::findExports()` unconditionally required `noteOn` and `noteOff` exports (`m3_FindFunction(&m_fnNoteOn, ...); if (res) return false;`). Audio effects (`vst_effect`) such as `Spectrum_Analyzer_v11.fmx` only export `init`, `process`, `memory`, and `getAbiVersion`—they do not export MIDI note handlers. Consequently, `loadPlugin()` aborted at the WASM loading stage. This also prevented the recent plugin history from updating and prevented the UI from being loaded.
   - *Fix*: Made `noteOn`, `noteOff`, `reset`, `getNumParams`, and `getAbiVersion` optional exports in `src/WasmEngine.cpp`. Audio effects and instruments both load properly. Verified with `test_dsp.exe` on `Spectrum_Analyzer_v11.fmx`, successfully computing real-time FFT telemetry buffers (`spectrum` with 64 float bands, `meters` with 2 float peak channels).
2. **Audio Blocked in Bitwig & No Audio Fed to Effect Plugins**:
   - *Root Cause*: 
     - `clap_audio_ports_count(..., is_input = true)` returned 0, advertising no audio input to the host. In Bitwig Studio, placing a plugin without audio inputs on an audio track silences/blocks the track.
     - `my_clap_process` in `src/ClapPlugin.cpp` passed hardcoded `nullptr` pointers for audio inputs (`engine.process(nullptr, nullptr, outL, outR, ...)`), starving effect plugins of any track audio.
   - *Fix*: 
     - Declared 1 stereo input port (id 0, 2 channels, `CLAP_PORT_STEREO`) and 1 stereo output port (id 0, 2 channels, `CLAP_PORT_STEREO`) in `CLAP_EXT_AUDIO_PORTS`.
     - Added `CLAP_PLUGIN_FEATURE_AUDIO_EFFECT` to `CLAP_FEATURES` alongside `CLAP_PLUGIN_FEATURE_INSTRUMENT`.
     - In `my_clap_process`, bound `process->audio_inputs[0].data32[0]` and `[1]` to `inL` and `inR`, feeding audio directly to `engine.process(inL, inR, outL, outR, numFrames)`.
     - If DSP is uninitialized or bypassed, input audio transparently passes through to output buffers via `memcpy`.
3. **UI Did Not Refresh When Loading a New `.fmx`**:
   - *Root Cause*: WebView2 cached HTML assets or failed to navigate cleanly when the existing URL matched or if `loadPlugin()` failed early.
   - *Fix*: Added timestamp cache-busting query parameter (`?t=<GetTickCount64>`) to `WebViewHost::loadHtmlContent()` file URL navigation. Added immediate parameter broadcast (`guiParamDirty = true`) and CLAP parameter rescan request (`CLAP_PARAM_RESCAN_ALL | CLAP_PARAM_RESCAN_VALUES`) upon loading so new controls reflect live states instantly.
4. **Plugin Stuck Defaulting to `Polyrhythm_Synth_v5.fmx`**:
   - *Root Cause*: Because `loadPlugin()` aborted on `Spectrum_Analyzer_v11.fmx`, `HistoryManager::addRecent()` was never reached. As a result, `%APPDATA%\FlowMusicWrapper\recent_plugins.json` was never updated with the new path.
   - *Fix*: With the module loading bug resolved, `loadPlugin()` completes and persists `recent_plugins.json` successfully, restoring whichever plugin was last loaded.
5. **Build and Test Verification**:
   - Rebuilt `build/FlowMusicWrapper.clap`, `build/FlowMusicWrapper.exe`, `build/FlowMusicWrapper_single_file.vst3`, and `build/FlowMusicWrapper.vst3` bundle.
   - Tested with `test_clap_host.exe` and `test_vst3_host.exe`: 100% passed. Verified stereo audio in/out ports, MIDI note ports, GUI instantiation, and DSP synthesis.

## 2026-10-10 09:35 - VST3 DAW Scanner Crash Resolution & Module Manifest Addition

### Issues Diagnosed & Resolved
1. **VST3 Rejected by Reaper and Bitwig Host Scanners**:
   - *Root Cause Identified via Vectored Exception Diagnostics*:
     When DAWs scanned the VST3, they invoked `factory->createInstance(...)` and `component->initialize(...)`.
     Inside `ClapAsVst3::initialize`, `clap-wrapper` called `create_plugin(...)`. During `create_plugin`, `PluginContext` constructor called `loadPlugin()`, which immediately fired `host_params->rescan(host, CLAP_PARAM_RESCAN_ALL | CLAP_PARAM_RESCAN_VALUES)`.
     However, at that point in the instantiation lifecycle, `_plugin` (the `std::shared_ptr<Clap::Plugin>` handle in `ClapAsVst3`) had not yet been assigned by the factory. `ClapAsVst3::param_rescan` attempted to access `_plugin->_plugin`, dereferencing a null pointer (`0xC0000005` Access Violation at offset `+0x1572A`).
     DAW scanner subprocesses (`reaper_plugin_scanner.exe`, `BitwigPluginHost64.exe`) trapped the crash during plugin probing and blacklisted the VST3 as invalid/unsupported.
   - *Secondary Vulnerability*:
     `ClapAsVst3::initialize(context)` called `context->queryInterface(...)` without checking `if (context)`. Scanners passing `nullptr` as a quick probe crashed immediately.
2. **Fixes Applied**:
   - In `src/ClapPlugin.cpp`:
     - Added `bool isInitialized = false` to `PluginContext`.
     - In `loadPlugin()`, only invoke `host_params->rescan` if `isInitialized && host` is true. During initial plugin construction, the host queries parameter counts and metadata directly, so early rescan callbacks are suppressed in compliance with the CLAP lifecycle.
     - Set `isInitialized = true` in `clap_init(plugin)`.
   - In `third_party/clap-wrapper/src/wrapasvst3.cpp`:
     - Added `if (context)` null check before querying `IHostApplication` in `ClapAsVst3::initialize`.
     - Added `if (!_plugin) return;` guard at the top of `ClapAsVst3::param_rescan`.
3. **VST3 Bundle Manifest (`moduleinfo.json`) Added**:
   - Created `Contents/Resources/moduleinfo.json` within `FlowMusicWrapper.vst3` adhering to the VST 3.8.0 manifest specification with class CID `4B051F92A713DE5C9783196AD414C7E3`, enabling modern fast scanner detection in Reaper 6.7+ and Bitwig Studio 4+.
4. **Enhanced Test Validation**:
   - Upgraded `tests/test_vst3_host.cpp` to execute the entire DAW scanner lifecycle: `InitDll()`, `GetPluginFactory()`, `countClasses()`, `getClassInfo()`, `createInstance()`, `initialize(nullptr)`, `initialize(&host)`, `IAudioProcessor` query, `IEditController` query, `terminate()`, `ExitDll()`, and `FreeLibrary()`.
   - Ran diagnostic test suite: 100% passed with zero crashes on both `FlowMusicWrapper_single_file.vst3` and `FlowMusicWrapper.vst3` (bundle).
   - Confirmed regression tests for `test_clap_host.exe` and `test_dsp.exe` continue to pass 100%.
