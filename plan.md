# Fork plan: turning this repo into a commercial-grade granular synth

This plan comes from a full read of the repo: project config, HISE module tree, UI data, all scripts, scriptnode networks, the SNEX and native C++ granular engines, build scripts, CI workflows and the original author's dev notes in `oi grandad/Documentation/`.

Nothing here has been deleted. Each item is a task for you to do. Items are ordered by priority inside each section. Section 0 is the order I'd work in.

> **Placeholder name:** branding was swapped for **`MyGranular` / `MyCompany`**. Search the repo for those strings and replace them with your real product and company names.

---

## 0. Recommended order of work

1. **Legal check (Section 1).** It decides whether you can sell this at all, and on what terms.
2. **Finish the rebrand in HISE, then rename the repo on disk (Section 2).**
3. **Repo cleanup (Section 3).** The `.git` folder is 488 MB, mostly generated build output.
4. **Get a build you control end to end (Section 4).** That means your own HISE pin, your own exports, and CI that doesn't depend on someone else's machine paths.
5. **Replace content (Section 9).** You have no samples. The presets point at the original author's sample library.
6. **Engine quality and performance (Section 5).** This is the part you'll actually be compared on.
7. **Architecture and modulation cleanup (Sections 6 and 7).**
8. **UI and graphics (Section 8).** You said you'd do the graphics yourself; Section 8 lists every branded asset.
9. **Close the feature gaps against competitors (Section 10).**
10. **QA, installers, signing, release (Sections 11 and 12).**

**Security and code-quality pipelines (Section 13)** run alongside all of this. Do 13.1 straight away: it's cheap, and it protects your signing keys before you add your own.

---

## 1. Legal and licensing (read first)

- [ ] **The project is GPLv3** (`LICENSE.txt`). You *can* sell a GPLv3 plugin, but:
  - everyone you ship binaries to must also get the **complete corresponding source** under GPLv3;
  - you can't add terms that restrict GPL rights, so any EULA can't forbid redistribution or modification of the code;
  - you must **keep the licence and the original copyright/attribution notices**. Removing the *brand* (name, logo, store links) is fine and expected. Removing the *credit* is not. Add a line like "Based on *Oi, Grandad!* by Rick Taylor, GPLv3" to `Credits.txt` and your About box.
  - You also have to mark your changes (GPLv3 §5a): say in your docs or release notes that this is a modified version, and when.
- [ ] **Going closed-source isn't an option unless the original author agrees in writing.** A commercial HISE licence covers HISE, not Rick Taylor's code in this repo. Your choices are: (a) ship it as GPL, (b) get a relicensing agreement from the author, or (c) clean-room rewrite every part he wrote: scripts, networks, the C++ granular engine, UI layout and presets.
- [ ] **HISE and JUCE** are GPLv3 or commercial. Under GPL you're covered. If you ever go closed-source you need commercial licences for both.
- [ ] **VST3 SDK:** check the licence of the copy vendored in your HISE checkout. Recent SDK versions are MIT, older ones use Steinberg's dual licence.
- [ ] **AAX** needs an Avid developer agreement plus PACE/iLok signing tools before you can ship to Pro Tools.
- [ ] **Loris** is compiled into the frontend (`IncludeLorisInFrontend=1` in `project_info.xml`). It's GPL, and nothing in `Scripts/` uses it. Turn it off to shrink the binary, or confirm its licence if you keep it.
- [ ] **Asset licences.** Check each one before you sell:
  - `Images/Montserrat-Medium.ttf`: SIL OFL. Ship the OFL text with the product.
  - Knob filmstrips credited in `Credits.txt` to "Paule Amca" from the WebKnobMan gallery: gallery terms vary per knob. Replace them or get permission.
  - `Images/lock-open-svgrepo-com.png`: check the SVGRepo licence for that icon.
- [ ] **The name "Oi, Grandad!"** and the author's store links, expansions ("Ultra Pack Pro", the plugin pack, Xolotl) and sample packs aren't yours. Don't use them, link to them or ship them.

---

## 2. De-branding

### 2.1 Already done

| File | Change |
|---|---|
| `README.md` | emptied |
| `.github/FUNDING.yml` | deleted (the author's PayPal and Gumroad links) |
| `oi grandad/project_info.xml` | `Name` → `MyGranular`, `BundleIdentifier` → `com.mycompany.mygranular`, `PluginCode` `Msog` → `Mygr` |
| `oi grandad/user_info.xml` | `Company` → `MyCompany`, `CompanyCode` `Msms` → `Myco`, `CompanyURL` and `CompanyCopyright` cleared |
| `oi grandad/Scripts/ScriptProcessors/oigrandad/ui.js` | the three `Engine.openWebsite(...)` store links (UPP, PlgInPk and XolLink panels) now open `""` and carry a TODO comment |
| UI label `Label21` (`text="Oi, Grandad!"`) | → `MyGranular` in the live Desktop XML and in the three stale copies |
| `AudioUnit.pkgproj`, `VST3.pkgproj` | installer identifier → `com.mycompany.pkg.MyGranular` |
| `.github/workflows/linux.yml`, `windows.yml` | release zip names `oi-grandad-*` → `mygranular-*` |

**Why the plugin code and company code mattered:** with the original `Msog`/`Msms` codes, DAWs would treat your plugin as *the same plugin* as Oi, Grandad! and mix up presets, state and scans. Use your own unique 4-character codes (at least one uppercase letter). Never change them after release.

### 2.2 Still to do

These are generated files or internal IDs. Hand-editing them is pointless or risky, because HISE overwrites them.

- [ ] **In HISE**, rename the root container `oi grandad` to your product name, then save the XML (File → Save as XML). That changes:
  - `XmlPresetBackups/oi grandad.xml`, line 3: `ID="oi grandad"`
  - the `UIData Source="oi grandadUIData"` reference
  - `processorId="oi grandad"` on the master gain slider in the Desktop XML
  - `Scripts/MultiChannel.js:1`: `Synth.getChildSynth("oi grandad")`. Update this by hand.
  - the script folder `Scripts/ScriptProcessors/oigrandad/`. Check where HISE saves the external scripts after the rename, and move them if needed.
- [ ] **Re-export everything** so the generated code picks up the new name and codes. The current generated files still hard-code the old identity:
  - `oi grandad/Binaries/Source/Plugin.cpp:25-28`: project name, company, website, copyright
  - `oi grandad/Binaries/JuceLibraryCode/JuceHeader.h:69-70`
  - all `AutogeneratedProject.jucer` files (136 references)
  - all 781 files under `oi grandad/CIExports/`. **CI builds from these committed exports, so CI will keep producing "oi grandad" binaries with the old plugin code until you re-export.**
  - `DspNetworks/Binaries/dll/oigrandad.dylib`: recompile the DSP network DLL. HISE names it after the project, so it becomes `MyGranular.dylib`.
- [ ] **`.gitmodules`** points HISE at the original author's personal fork, `https://github.com/RickTaylor79/HISE.git`. See Section 4.
- [ ] The **dev notes in `oi grandad/Documentation/*.md`** contain `/Users/rick/...` paths and the old name. Keep them as engineering notes (they're useful) or rewrite them.
- [ ] **Graphics are yours to do.** Section 8 lists every branded image.

### 2.3 One command to rename the project folder and fix CI paths

Run this **after** you've renamed the root container in HISE and saved it as `XmlPresetBackups/MyGranular.xml`, and with your HISE source checked out at `~/Documents/GitHub/HISE`. That's the same layout the original author used, so the CI path rewrites line up once `rick` is swapped for your username. The command:

1. removes stale duplicates and generated trees;
2. renames the project folder;
3. rewrites names and paths in the CI, installer and build scripts;
4. regenerates the CI exports;
5. commits.

```bash
export NEW="MyGranular" && export HISEAPP="/Applications/HISE.app" && cd /path/to/kornnat && git rm -r -q --cached .xcode-build && rm -rf .xcode-build && git rm -r -q DspNetworks XmlPresetBackups "oi grandad.xml" && git rm -r -q "oi grandad/CIExports" "oi grandad/Binaries" && git rm -r -q --cached "oi grandad/DspNetworks/Binaries/Builds" "oi grandad/DspNetworks/Binaries/dll" && git mv "oi grandad" "$NEW" && perl -pi -e 's#/Users/rick/#/Users/$ENV{USER}/#g; s#\\Users\\rick\\#\\Users\\$ENV{USER}\\#g; s#Oi-Grandad#$ENV{NEW}#g; s#oi grandad#$ENV{NEW}#g; s#oi-grandad#\L$ENV{NEW}\E#g' .github/workflows/linux.yml .github/workflows/macos-signed.yml .github/workflows/windows.yml $(ls .github/workflows/security.yml 2>/dev/null) .gitignore AudioUnit.pkgproj VST3.pkgproj "$NEW/BuildScripts/prepare_ci_exports.sh" "$NEW/BuildScripts/test_build_macos.sh.sh" "$NEW/BuildScripts/BUILD.md" && chmod +x "$NEW/BuildScripts/prepare_ci_exports.sh" && "$NEW/BuildScripts/prepare_ci_exports.sh" --hise-bin "$HISEAPP" --hise-source "$HOME/Documents/GitHub/HISE" && git add -A && git status --short | head -50 && git commit -m "Rename project to $NEW, drop stale duplicates and regenerate CI exports"
```

Before you trust it, check that the new `CIExports/*/Builds/**` files contain your own HISE path and that the `perl` rewrites in the three workflows match it. If your HISE checkout lives somewhere else, edit the `s#/Users/rick/...#` patterns in all three workflow files to match.

---

## 3. Repo cleanup

- [ ] **Generated output is committed, even though `.gitignore` lists it:**
  - `oi grandad/CIExports/`: 133 MB, 781 files
  - `oi grandad/Binaries/`: 39 MB. `Binaries/Source/PresetData.cpp` alone is **38 MB**: the embedded pool of the original author's audio, images, scripts and presets.
  - `oi grandad/DspNetworks/Binaries/Builds/**`, including Xcode `build/` and `XCBuildData`
  - the compiled `DspNetworks/Binaries/dll/oigrandad.dylib`
  - `.xcode-build/` at the repo root
  - **Result:** `.git` is 488 MB. Once you've rebranded, start a fresh repo or run `git filter-repo`. That also drops the old author's binaries from your history.
- [ ] **Stale duplicates at the repo root that HISE never reads:** `/DspNetworks/`, `/XmlPresetBackups/` (including `AutoSave1*` and `MatrixTest*`) and `/oi grandad.xml`. The real project lives in `oi grandad/`.
- [ ] **Dead or experimental files to prune inside the project:**
  - `DspNetworks/CodeLibrary/snex_node copy/`, `snex_node copy 2/`, `snex_node copy 3/` (including the `*.before_restore_20260304_rollback` and `*.before_sync_*` files), and `snex_osc/.h`
  - `DspNetworks/OLD/` (14 networks) and `DspNetworks/TEMP REMOVE/`
  - unused networks in `DspNetworks/Networks/`. Keep only what `factory.cpp` registers and the patch uses: `MatrixTest2`, `sn_fin`, `DspNetwork`, `OutMods`, `ENV`, `MacroMod`, `res2`, plus the `*_networkdata` ones.
  - `Scripts/rec.js` and `AdditionalSourceCode/nodes/RNBO.cpp`, which are both empty
  - the `KnobLAF1-5.js` variants. Merge them into one LAF file.
  - `Presets/Autosave_*.hip`, `MatrixTest*.hip`, `SpecTest.hip`, `Preset.hip`, and the stale `Presets/oi grandad.xml`
  - `BuildScripts/test_build_macos.sh.sh`: fix the double extension.
  - `BuildScripts/build_dsp_networks.sh`, which is still a TODO stub
- [ ] **Rename "test" names that are actually production.** The engine network every layer uses is `MatrixTest2`; call it `GranularVoice`, for example. `DspNetwork` (the LFO network) and `sn_fin` (the master FX) need clearer names too.
- [ ] **Rewrite `.gitignore`** from scratch. It has duplicate lines, author-specific `.afdesign` entries, and ignores folders that are tracked anyway.
- [ ] **Add an `.editorconfig` and a `CONTRIBUTING.md`** with the build steps. GPL users need to be able to rebuild what you ship.

---

## 4. Build system and CI

- [ ] **Your own HISE pin.** The submodule points at `RickTaylor79/HISE` at commit `429866d`, the original author's personal fork. Options:
  - pin **official HISE** (`christophhart/HISE`) at a known-good commit, or
  - fork HISE under your own account and commit any patches you need there.
- [ ] **Stop patching HISE and JUCE inside CI.** All three workflows regex-patch HISE, JUCE and the VST3 SDK at build time:
  - `stricmp` rewrites
  - VST3 speaker-enum removals (`macos-signed.yml:220-233`)
  - the `NO`/`YES` macro rename in `ComplexGroupManager.cpp`
  - stubbed SDK `.cpp` files
  - generated FAUST `export.h` headers
  - IPP define stripping

  This is fragile: any HISE update will silently break it. Move the fixes into your HISE fork, or pick a HISE commit that builds cleanly with current Xcode and MSVC.
- [ ] **Hard-coded machine paths.** CI string-replaces `/Users/rick/Documents/GitHub/HISE` in the generated projects. Replace this with a variable, or export with relative paths.
- [ ] **Per-OS preprocessor definitions are inconsistent** in `project_info.xml`:
  - `NUM_HARDCODED_FX_MODS` is defined twice on every OS.
  - FX mods are 16 on Windows and Linux but 8 on macOS.
  - **Linux has a typo: `NUM_HARDCODED_SYNTH_MODS=166`.**
  - Only Linux sets `HISE_USE_SYSTEM_APP_DATA_FOLDER=0` and `JUCE_DISABLE_NATIVE_FILECHOOSERS=1`.

  Make the definitions identical across OSes unless a difference is intentional.
- [ ] **Longer term, let CI export the project itself.** Build the HISE CLI on the runner and call `export_ci`, so you don't have to commit `CIExports/` at all.
- [ ] **Add a release workflow:** tag → build all three OSes → sign → notarize → build installers → GitHub Release.
- [ ] **Code-sign Windows.** Only macOS is signed today. Add an Authenticode certificate or Azure Trusted Signing, and add a Windows installer (Inno Setup or WiX). The Packages `.pkgproj` files only cover macOS VST3 and AU.
- [ ] **Add AAX** (it needs the Avid SDK and PACE wrapping) and **AU validation** (`auval`) to the macOS job.
- [ ] Decide whether to keep 10 output channels (`HISE_NUM_PLUGIN_CHANNELS=10`). It's great for multi-out, but it confuses some hosts. Expose a stereo version, or label the buses.
- [ ] Decide on **ARM Linux and Windows ARM64** support.

---

## 5. Granular engine (DSP core)

The production engine is the native C++ node `DspNetworks/ThirdParty/granular_player_stepquant_density_hybrid_native.h` (2,122 lines). The network loads it as `project.granular_player_stepquant_density_hybrid_native`. The older SNEX version in `DspNetworks/CodeLibrary/snex_node/granular_player_stepquant_density_hybrid.h` (2,716 lines) is legacy.

### 5.1 Sound quality (what reviewers will hear first)

- [ ] **The resampler is linear-interpolation only** (`readStereoPairAt`, native `.h:1136`; same in SNEX `.h:696`). That aliases badly when pitched up and loses high end when pitched down. Commercial granulars use at least 4-point Hermite or cubic interpolation, ideally a windowed-sinc or polyphase resampler, plus anti-alias filtering when `delta > 1`. Offer a quality switch (Draft / Normal / HQ).
- [ ] **No oversampling** on the grain sum or on the resonator and drive stages.
- [ ] **Open click bugs, per the author's own notes** (`Documentation/GranularPostCommitTaskList.md`, "Audit playback modes"):
  - playback mode 4 (`Hold`) gets scratchy when pitch changes;
  - modes 1/3/5, mode 2 and mode 4 each have a **different pitch baseline**, so switching modes shifts pitch;
  - some modes still click at grain-wrap and read-wrap points.
- [ ] **When the audio-data lock fails, the voice resets** (native `.h:255-260` and `:284-289`). Every grain restarts while a sample is loading or being swapped, which causes dropouts and clicks. Fade out instead of hard-resetting, and double-buffer sample swaps.
- [ ] **Every grain restarts in sync on note-on** (`handleHiseEvent`, native `.h:301`), which locks grains into an audible rhythm. The notes describe a "cloud bloom" fix; check it's finished and tunable (`BloomDuration`, parameter 16).
- [ ] **`Cloud` mode isn't implemented.** `CloudAmount` (parameter 10) does nothing, and playback slot 6 falls back to the default loop (`HomogeneousCloudModePlan.md`). Finish it or remove it from the UI.
- [ ] **Window shapes:** Hann, Tukey and Blackman are computed with `std::cos` per grain per sample. Add a lookup table and more shapes (Gaussian, trapezoid, expodec/rexpodec, user-drawn).
- [ ] **Grain gain normalisation:** check perceived loudness as `MaxGrains` and `Density` sweep. Add an automatic gain compensation option.
- [ ] **Denormals:** check that `ScopedNoDenormals` (or the HISE equivalent) is active in feedback paths: the diffusion allpass, `res2` resonators and the delay.

### 5.2 Performance (CPU is a deal-breaker for granular synths)

- [ ] **`Engine.setMaximumBlockSize(32)`** in `ui.js:4` forces 32-sample processing for the whole plugin. This is a big CPU overhead. Raise it, or remove it and fix whatever needed it (probably modulation smoothness; Section 7 covers the matrix-mod path).
- [ ] **Everything is rendered per sample frame.** The node uses `processFrame`/`renderFrame`, and the network wraps it in `container.frame2_block`. Rewrite the core to render block-wise (process each grain for N samples, then sum), with SIMD where possible.
- [ ] **`refreshSourceState()` runs on every process call and every frame** (native `.h:262` and `:291`). Run it only when the data changes.
- [ ] **`setExternalData` calls `sp->setNumSliders(8)`** on the slider pack (native `.h:318-323`) and catches any slider pack regardless of index. Don't resize data from an audio-thread callback. Configure the slider pack when you build the network.
- [ ] **`std::exp` smoothing coefficients are recalculated per sample** (for example native `.h:705`, `:961`, `:1355`, `:1401`). Compute them once in `prepare()` or when the parameter changes.
- [ ] **CPU budget:** 4 layers × 2 voices × up to 32 grains × per-frame processing. Profile it with `MaxGrains=32` on all layers, then set targets (for example, under 10% of one core at 48 kHz / 256 samples on an M1 or a recent i5).
- [ ] **Each layer's voice limit is set to 2** (`VoiceLimit="2.0"` on `Granular1-4`), probably to protect the CPU. That makes the instrument effectively mono or duophonic per layer (see Section 10).

### 5.3 Code health

- [ ] **The SNEX version has 32 grain slots hand-unrolled** into `schedPhase1..32` fields with `if (i == 16)...` getters and setters (SNEX `.h:60-430`). The native port already uses `std::array<GrainState, 32>`. Delete the SNEX copy once you're sure the native one is the only one loaded.
- [ ] **Parameter layouts differ** between SNEX and native. In SNEX, parameter 6 is `PitchMode`, 9 is `ScrubMode` and 10 is `ScrubBlend`. In native, they're `Respawn`, `Diffusion` and `CloudAmount`. Check every connection in `MatrixTest2.xml` and every script `setAttribute` index against the native layout.
- [ ] **Add a parameter `enum`** to the granular node, the way `master_bus_compressor_native.h` already does, instead of the magic `P == n` numbers.
- [ ] **The author's own outstanding engine tasks**, kept here so you don't lose them (`Documentation/GranularPostCommitTaskList.md`, "Next Batch"):
  1. click audit;
  2. remove glide from `PitchMode`;
  3. give pitch sync its own controls;
  4. `PhaseScatter` Random/Spread switch;
  5. grain-spread switch, possibly tempo-synced inside the C++ node.
- [ ] **Performance ideas from `GranularEnginePerformanceTweaks.md` that aren't done yet:**
  - cache pitch-mode multipliers per grain;
  - a window lookup table;
  - tighter read-phase wrapping.
- [ ] **Unit-test the engine offline.** Build a small JUCE console harness that renders fixed parameter sets to WAV and compares them against reference files, so refactors don't silently change the sound.

---

## 6. Architecture and scripts

- [ ] **The four voices are copy-pasted.** `Scripts/Voice1.js` to `Voice4.js` are around 460 lines each and differ only in indices (a 230-line diff once you strip the digits). `macros.js` (1,233 lines) has the same pitch and volume handlers repeated four times. Refactor to one voice module parameterised by index, with arrays of components and processors.
- [ ] **`ModControls.js` (1,056 lines) and `macros.js`:** split them into modules (matrix registration, LAF, preset handling, voice UI).
- [ ] **`ui.js` (755 lines)** contains the website-link handlers plus a playback menu. Remove the three promo panels (UPP, PlgInPk, XolLink) and their callbacks once your graphics are in.
- [ ] **Remove dead `global_cable` routing.** `ui.js` creates 20 global cables (`Trk1-4` and `ModPosA1` to `ModPosD4`) for the legacy modulation system. Delete the old cable, branch and peak relay chains after the matrix migration (Section 7).
- [ ] **Hidden controls:** 335 of the 1,606 UI components are `visible="0"`. 128 of them are orphans that no script references (the full list is in `Documentation/HiddenUiAudit.md`). They still get registered as matrix targets and saved in presets. Delete them.
- [ ] **Unused modules:** the patch has 220 `ModulatorChain`s, 24 hardcoded LFO, macro and track modulators that are always running, and 7 `ScriptFX` (`Xf1-4`, `RecBuffer`, `RecordingScript`, `END`). Audit which ones are needed. Script FX are expensive. Move them into scriptnode or C++ where you can.
- [ ] **Unused MIDI processors:** the four `LegatoWithRetrigger` processors are all bypassed. Remove them, or turn them into a real mono/legato/poly voice-mode switch.
- [ ] **State and versioning:** add a preset format version, and a migration routine in the user preset handler, so future parameter changes don't break saved songs.

---

## 7. Modulation system

The patch is halfway through migrating from the old `global_cable`/branch/peak/pma selector chains to HISE's modulation matrix (`Engine.createModulationMatrix("Global Modulator Container1")` in `macros.js:627`, plus 32 `MatrixModulator`s). Finish that migration, using the author's contract in `Documentation/MatrixMigrationContract.md` and the remaining items in `Documentation/MatrixWorkList.md`:

- [ ] **Auto-enable modulators:** enable a modulator only while it has at least one route. 12 LFOs plus 8 macros plus 4 trackers run all the time right now.
- [ ] **Destination hygiene:** exclude helper and mirror controls, and make sure every target is audible.
- [ ] **Consistent target names**, for example `voice1_pitch` instead of a mix of `Voice_1_PositionA` and `Voice_2_Position_A`.
- [ ] **Move sensitive targets into DSP** via `core.matrix_mod`: pitch, position and grain size. That removes the need for 32-sample blocks.
- [ ] **Delete the legacy selector banks** once every consumer has moved over.
- [ ] Rename `s1..s12` to `LFO1..12`, rename `MacroMod1..8` to `MACRO1..8`, and give `seq2..12` the same layout as `seq1`.
- [ ] Add a **small-control modulation indicator**, and finish styling the matrix panel.
- [ ] **Commercial must-haves** on top of that:
  - drag-and-drop modulation;
  - per-route curve and bipolar settings;
  - modulation visualised on every knob;
  - MPE (per-note pressure, slide and pitch bend) as sources;
  - velocity, aftertouch and mod wheel as matrix sources.

---

## 8. UI and graphics (you're doing these; here's the full list)

### 8.1 Branded or promo assets to replace

- [ ] `oi grandad/oigrandad.png` (README screenshot) and `oi grandad/ognew.png` (the "Ultra pack pro" README advert)
- [ ] **Backgrounds:** `Images/OG_ BG.png`, `OG_ BG.aftemplate`, `ogbackground.png`, `ogbackground-notext.png`, `ogbackground20nov.png`, `ogbackground.afdesign`, `ogsketch2.afdesign` ("OG" means Oi, Grandad)
- [ ] **Store and promo panels:**
  - `Images/pack.png`, `pack2.png` (UPP panel)
  - `pluginpack.png` (PlgInPk)
  - `xol.png`, `xol2.png`, `xol.pdf` (XolLink, which advertises the author's Xolotl synth)
  - `ULPAD22.png`, `ulp.jpg`, `ad.png`, `ad2.png`, `ad3.png`, `bigyellowwithfire6.png`
- [ ] **Everything in `Other Images/`:** 20 MB of `.afdesign` source files, including `ogbackground` and `ogsketch2`.
- [ ] **`wiki-images/`**: 22 screenshots of the original UI, used in the author's wiki.
- [ ] `Documentation/images/OgFileManagement.png`
- [ ] **Missing file:** `bf6.png` is referenced by the UI (`Image12`) but isn't in `Images/`.
- [ ] **Knob filmstrips** (`k1`-`k9`, `K6`-`K9`, `JK.png`, `knbg.png` and similar): licence check (Section 1), or redraw them.

### 8.2 UI work for a commercial product

- [ ] **Resizable and HiDPI UI.** The interface is fixed at 960×720 (`Content.makeFrontInterface(960, 720)`). Add a zoom setting (75-200%) and 2× assets, or vector LAF drawing.
- [ ] **Tooltips and a help overlay** on every control.
- [ ] **Value pop-ups** with units, double-click to reset to default, and Shift-drag fine adjustment.
- [ ] **Accessible colour contrast.** Consider light and dark themes.
- [ ] **Grain visualiser.** Show grain positions over the waveform. `PosIndicators` already exists; make it smooth and cheap.
- [ ] **Clean, label-consistent text.** Rename the internal-looking labels, and fix spelling and capitalisation.

---

## 9. Content: samples, presets, library

- [ ] **No audio is in the repo.** `*.wav`, `*.aif` and `*.aiff` are gitignored, and `AudioFiles/` doesn't exist. The project expects sample sets you don't have, all owned by the original author:
  - `AudioFiles/Factory/*`
  - `/Library/Application Support/modularsamples/oi grandad/AudioFiles/...`: `Waveterm/`, `Chains/`, `Stems/`, `Loops/`
  - the chord and drone menu items in `Panel3` (`AIR`, `BANGER`, `Bad Guy m7_9` and so on), which are the author's sample names
- [ ] **The embedded pool belongs to the original author.** `EmbedAudioFiles=1` and `EmbedImageFiles=1` mean audio and images are compiled into the binary. The committed `Binaries/Source/PresetData.cpp` (38 MB) contains *his* pool. Rebuild it with your own content.
- [ ] **Build your own factory library.** Commercial granulars ship roughly 300-1,000 MB of curated sources: textures, vocals, field recordings, instruments, synth one-shots.
- [ ] **Package it as a HISE expansion or an HLAC monolith with an installer and sample-folder relocation**, instead of embedding it. That keeps the plugin binary small and makes updates cheap.
- [ ] **Rewrite all presets.** The 7 factory presets in `UserPresets/factory/Bank 1/` (`HELLO`, `Pluck`, `a`, `geewhiz` and so on) and the combo-box `items` lists in the Desktop XML contain absolute paths to `/Users/rick/...` and `/Library/Application Support/modularsamples/...`. Plan for 150-300 categorised, tagged presets made by sound designers.
- [ ] **Store sample references as `{PROJECT_FOLDER}`, `{EXP::...}` or user-folder-relative paths**, never absolute ones. Add a "missing sample" relocation dialog.
- [ ] **`DspNetworks/Networks/MatrixTest2.xml:771`** has an embedded absolute path to `.../oi grandad copy/AudioFiles/Loops/RNGGIT-85.2-02.wav`. Clear it.

---

## 10. Feature gaps against commercial granular synths

The bar is set by products such as Arturia Pigments' granular engine, Output Portal, Audio Damage Quanta, Emergence, Ableton Granulator III and Spektro Mangle. Compared with those, you need:

- [ ] **Real polyphony.** You need 8-16 voices per layer, with voice modes (poly, mono, legato, unison) and glide. The current limit is 2 voices per layer, and the CPU work in Section 5.2 is what makes more possible.
- [ ] **Drag and drop your own audio**, from the OS and from the DAW, onto each layer. Add a built-in sample browser, favourites, recent files, and auto-normalise and trim on import.
- [ ] **Live input granulation and an FX version.** `ProjectType` is `Instrument`, though `RecBuffer` and `RecordingScript` show recording was being explored. A granular **effect** build (sidechain or live input, freeze, tempo-synced buffer) is a big market of its own.
- [ ] **Host automation that covers the whole synth.** `HISE_MACROS_ARE_PLUGIN_PARAMETERS=1` with `HISE_NUM_MACROS=8` means the **DAW sees 8 parameters**. Expose every important control as a named plugin parameter (`isPluginParameter` and `pluginParameterName`), with stable IDs, and test sample-accurate automation.
- [ ] **A proper preset browser:** categories, tags, search, favourites, author field, A/B compare, randomise with locks (some random-lock UI already exists), and **undo/redo**.
- [ ] **MIDI learn** with save/restore, and **MPE**.
- [ ] **Better built-in effects.** The master chain (`sn_fin`) uses a basic `fx.reverb`, a cubic delay, SVF filters and a resonator bank (`res2`). Competitors ship lush algorithmic reverbs, chorus, saturation, an EQ and a limiter. The repo also has a `master_bus_compressor_native.h`; check that it's wired in.
- [ ] **Tempo sync everywhere:** grain rate, LFOs, delay, sequencer, plus host transport-locked scrubbing.
- [ ] **Formats:** VST3, AU, AAX, standalone, and possibly CLAP if your HISE version supports it.
- [ ] **Built-in docs:** a manual (PDF or web), tutorial videos and first-run tips.

---

## 11. QA and testing

- [ ] **[pluginval](https://github.com/Tracktion/pluginval)** at strictness 10 in CI for VST3 and AU, plus `auval -v aumu Mygr Myco` on macOS.
- [ ] **DAW test matrix:** Ableton Live, Logic Pro, FL Studio, Bitwig, Reaper, Cubase, Studio One, Pro Tools (AAX).
- [ ] **Audio settings:** sample rates 44.1, 48, 88.2, 96 and 192 kHz; buffer sizes 16 to 2048; offline and faster-than-realtime rendering must match realtime.
- [ ] **State and recall:** save and reload projects, preset recall, project recall after moving sample folders, and automation playback.
- [ ] **Stress tests:** all layers at 32 grains with fast modulation; rapid preset switching; sample swaps during playback (see the lock-reset issue in Section 5.1).
- [ ] **Leak, denormal and NaN checks.** The native node already guards `GrainMs` with `std::isfinite`; do the same for every parameter.
- [ ] **Beta programme:** 20-50 users across operating systems before launch.

---

## 12. Release, distribution and business

- [ ] **Installers:** macOS `.pkg`, signed and notarized (update `VST3.pkgproj` and `AudioUnit.pkgproj` with your names and paths, and add Standalone, AAX and content packages). Windows: a signed Inno Setup or WiX installer. Linux: a `.tar.gz` or `.deb`.
- [ ] **A GPL-compliant offer.** Link to the exact source tag for every binary release, and keep that tag buildable.
- [ ] **Licensing and copy protection.** `Binaries/Source/CopyProtection.cpp` is empty and the `Unlocker` RSA key is blank. Under GPL, copy protection is weak because anyone can rebuild from source. Commercial GPL products usually sell **content, convenience (signed installers), support and updates**. If you go closed-source after a relicence (Section 1), add HISE's `Unlocker` or another licensing system.
- [ ] **Versioning.** `Version` is still `2.5.0` (the original's). Restart at `1.0.0` under your name, and keep a CHANGELOG.
- [ ] **Brand basics:** a name check (trademark search), a domain, a website, a support email, an EULA that doesn't contradict GPL, a privacy statement if you add update checks, and pricing.
- [ ] **Update checker and crash reporting**, both opt-in.

---

## 13. Security and code-quality pipelines

### What the repo has today

Almost nothing. The only integrity check is that each workflow verifies the checked-out HISE commit matches the submodule pin.

| Area | Current state |
|---|---|
| PR builds | **None.** All three workflows run only on `push` to `main` and `workflow_dispatch`. Pull requests are never built or checked. |
| Workflow permissions | No `permissions:` block, so jobs get the repo's default `GITHUB_TOKEN` scope. |
| Action pinning | Actions are pinned to tags (`actions/checkout@v4`, `upload-artifact@v4`, `setup-msbuild@v2`), not commit SHAs. |
| Signing secrets | The Apple Developer ID certificate and notarization password are loaded on every push to `main`. They aren't protected by a GitHub Environment with required reviewers. |
| Static analysis | No CodeQL, clang-tidy, cppcheck or compiler warnings-as-errors. |
| Runtime checks | No AddressSanitizer or UndefinedBehaviorSanitizer builds, no fuzzing, no unit tests. |
| Dependencies | No Dependabot. HISE, JUCE and the VST3 SDK come from a personal HISE fork, and CI regex-patches their source at build time (Section 4). |
| Secret scanning | No gitleaks or trufflehog job. Turn on GitHub secret scanning and push protection as well. |
| Policy and provenance | No `SECURITY.md`, no SBOM, no release checksums, no build attestations. Windows binaries are unsigned. |

### Where the C++ attack surface is

Audio plugins load files that users download from strangers, so these are the code paths worth hardening:

- **Preset and state loading:** user presets (`.preset` XML), DAW state chunks (`setStateInformation`), sample maps and expansions.
- **Audio file loading** into the granular node: WAV and AIFF parsing (JUCE), then `setExternalData` and `refreshSourceState` in `granular_player_stepquant_density_hybrid_native.h`.
- **Sample reads in the engine.** `readStereoPairAt` (native `.h:1136`) only guards the upper end of the buffer (`size() <= i + 1`). Nothing checks `i < 0`, so if a reverse or ping-pong mode ever produces a negative position, that's an out-of-bounds read. Confirm it with an ASan build (13.2) instead of assuming it can't happen.
- **Parameter values:** non-finite or out-of-range values from hosts and automation. Only `GrainMs` is guarded with `std::isfinite` today.

### 13.1 Do now (cheap, low risk)

- [ ] **Build pull requests.** Add a `pull_request` trigger to the Linux and Windows workflows. Keep signing and notarization out of PR runs.
- [ ] **Least-privilege tokens.** Add `permissions: contents: read` at the top of every workflow, and grant more only to the job that needs it.
- [ ] **Pin every third-party action to a full commit SHA**, with the tag as a comment. Let Dependabot bump them.
- [ ] **Protect signing secrets.**
  - Move the Apple and Windows signing secrets into a GitHub **Environment** (for example `release`) with a required reviewer.
  - Run signing only on version tags, not every push to `main`.
  - Protect `main`: require PRs and passing checks.
- [ ] **Dependabot** for `github-actions` and `gitsubmodule`.
- [ ] **Secret scanning:** enable GitHub secret scanning and push protection in repo settings, and add a gitleaks job. The gitleaks action needs a free licence key for organisation-owned repos; personal repos don't.
- [ ] **`SECURITY.md`** with a private reporting route. Enable GitHub private vulnerability reporting.
- [ ] **cppcheck on first-party C++** (`DspNetworks/ThirdParty/*.h`). It currently passes clean on both native headers, so it can block from day one.

One command creates the Dependabot config, a `security.yml` workflow (cppcheck plus gitleaks, run on PRs, pushes to `main` and weekly) and `SECURITY.md`, then commits them. Run it from the repo root before the Section 2.3 rename; the rename command also rewrites the path inside `security.yml`.

```bash
cd /path/to/kornnat && mkdir -p .github/workflows && tee .github/dependabot.yml >/dev/null <<'EOF' && tee .github/workflows/security.yml >/dev/null <<'EOF2' && tee SECURITY.md >/dev/null <<'EOF3' && git add .github/dependabot.yml .github/workflows/security.yml SECURITY.md && git commit -m "Add security CI, Dependabot and security policy" && git push
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
  - package-ecosystem: "gitsubmodule"
    directory: "/"
    schedule:
      interval: "weekly"
EOF
name: Security

on:
  pull_request:
  push:
    branches: [ main ]
  schedule:
    - cron: '17 3 * * 1'
  workflow_dispatch:

permissions:
  contents: read

jobs:
  cppcheck:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - name: Install cppcheck
        run: sudo apt-get update && sudo apt-get install -y cppcheck
      - name: Static analysis of first-party C++
        shell: bash
        run: |
          cppcheck --enable=warning,performance,portability --inline-suppr \
            --suppress=missingIncludeSystem --suppress=missingInclude \
            --language=c++ --std=c++17 --error-exitcode=1 \
            "oi grandad/DspNetworks/ThirdParty/"*.h

  secrets-scan:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
EOF2
# Security policy

Please report vulnerabilities privately through GitHub's "Report a vulnerability" button on the Security tab. Do not open a public issue.

We aim to acknowledge reports within 5 working days.
EOF3
```

cppcheck checks headers on their own, without JUCE and HISE, so the analysis is shallow. 13.2 covers the deeper checks.

### 13.2 Do once you have the offline test harness (Section 5.3)

- [ ] **CodeQL for C++** (`github/codeql-action`, `security-extended` queries). Run it on a build of the Linux target, and exclude `HISE/`, `JUCE/` and generated `JuceLibraryCode/` so it reports on your code.
- [ ] **clang-tidy** with `bugprone-*`, `cert-*`, `cppcoreguidelines-*`, `performance-*` and `concurrency-*`, using `compile_commands.json` from the Linux Makefile build (generate it with `bear`). Start in report-only mode, then block on new findings.
- [ ] **Warnings as errors on your own code:** `-Wall -Wextra -Wshadow -Wconversion` on clang and gcc, `/W4` on MSVC.
- [ ] **ASan and UBSan job:** build the offline render harness with `-fsanitize=address,undefined` and run every playback, direction and scrub mode, pitch extremes, `MaxGrains` 1 to 32, tiny and empty samples, mono and 32-channel files, and NaN or Inf parameters.
- [ ] **ThreadSanitizer** on a harness that loads samples and changes parameters from a second thread while audio renders. The `DataTryReadLock` and reset path in Section 5.1 is the main target.
- [ ] **Fuzzing (libFuzzer)** on three targets: user preset and state XML loading, audio file loading into the granular node, and the engine's parameter and render path. Seed it with real presets and audio files, and run it nightly.
- [ ] **pluginval** at strictness 10 on every PR build (Section 11). It catches thread-safety and state-handling crashes that hosts trigger.

### 13.3 Do before the first public release

- [ ] **Provenance for every release binary:** an SBOM (for example `anchore/sbom-action` / syft), SHA-256 checksums, and `actions/attest-build-provenance` attestations.
- [ ] **Signed Windows binaries** (Section 4), plus the existing macOS signing and notarization, run only from protected tag builds.
- [ ] **Your own HISE fork with the CI patches committed** (Section 4), so builds don't depend on regex edits to third-party source.
- [ ] **Track security fixes in HISE and JUCE** (watch their releases) and rebase your HISE pin on a schedule.

---

## Appendix A: what the repo contains

| Path | What it is |
|---|---|
| `oi grandad/` | **The real HISE project:** `project_info.xml`, scripts, networks, images and presets |
| `oi grandad/XmlPresetBackups/oi grandad.xml` | Main module tree: 4 × `HardcodedSynth` (`Granular1-4`, network `MatrixTest2`), 4 × `HardcodedMasterFX` (`Final1-4`, network `sn_fin`), 24 hardcoded modulators (LFO1-12, MACRO1-8, Track1-4), 4 × ENV, 32 matrix modulators |
| `oi grandad/XmlPresetBackups/oi grandadUIData/oi grandadDesktop.xml` | Live UI layout (1,606 components, 335 hidden) |
| `oi grandad/Scripts/` | HISEScript: `ui.js` (main), `macros.js` (matrix and macros), `ModControls.js`, `Voice1-4.js`, LAF files |
| `oi grandad/DspNetworks/ThirdParty/*_native.h` | **C++ granular engine** and master bus compressor |
| `oi grandad/DspNetworks/Networks/` | Scriptnode graphs (voice, FX, LFOs, envelopes, modulation) |
| `oi grandad/AdditionalSourceCode/nodes/` | Exported C++ for the compiled networks and `factory.cpp` |
| `oi grandad/CIExports/{vst3,au,standalone}` | Committed HISE exports that CI compiles |
| `oi grandad/Binaries/` | Live HISE export, including the 38 MB `PresetData.cpp` |
| `oi grandad/BuildScripts/prepare_ci_exports.sh` | Regenerates `CIExports/` through the HISE CLI |
| `oi grandad/Documentation/` | The original author's engineering notes and TODO lists |
| `.github/workflows/` | Linux (unsigned), macOS (signed and notarized, universal), Windows (unsigned) |
| `HISE/` | Submodule pinned to the author's HISE fork (not initialised in this clone) |
| `/DspNetworks`, `/XmlPresetBackups`, `/oi grandad.xml`, `/.xcode-build` | Stale copies and build junk at the repo root |
