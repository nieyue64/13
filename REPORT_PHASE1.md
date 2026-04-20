# Phase 1: Native Class Recovery & Timing Fixes - Report

## Completion Status: DONE

## Changes Made

### 1. Fixed game:init:post Double Trigger (Line ~29015)
- **Problem**: `game:init:post` was emitted TWICE per game init:
  - Once at the end of native `Game.init()` (line ~24042)
  - Again in a monkey patch that wrapped `Game.init` to emit it redundantly (line ~29021)
- **Fix**: Removed the monkey patch's `Game.init` wrapper. The `__tuGameReadyEvent` flag is still set to prevent other patches from re-wrapping.
- **Impact**: Eliminates first-frame stutter caused by all `game:init:post` listeners running twice. All subsystems (tileLogic, machines, worker init) that listen to this event will now fire exactly once.

### 2. InputManager Safety Patches Absorbed Natively (Line ~23157)
- **Problem**: A monkey patch at line ~29026 wrapped `InputManager.prototype.bind` to add:
  - Window blur handler (reset all keys)
  - Visibility change handler (reset on tab switch)
  - Mouse leave canvas handler (reset mouse buttons)
  - Global mouseup handler (reset mouse buttons)
  - Wheel scroll for hotbar switching
- **Fix**: All these handlers are now part of the native `InputManager.bind()` method.
- **Flags Set**: `this.__tuExtraBound = true` and `this.__tuInputSafety = true` so the monkey patch's guard condition will skip re-binding.
- **Impact**: Removes one layer of prototype chain wrapping from InputManager.bind(). Input reset is now guaranteed to be bound once, deterministically.

### 3. AudioManager Rain Synth & Cave Reverb Absorbed Natively (Line ~8759)
- **Problem**: Two separate monkey patches added methods to `AudioManager.prototype`:
  - `__rainSynthInstalled` patch: Added `_makeLoopNoiseBuffer()`, `_startRainSynth()`, `_stopRainSynth()`, `updateWeatherAmbience()`
  - `__caveReverbInstalled` patch: Added `_ensureCaveFx()`, `setEnvironment()`
- **Fix**: All six methods are now native methods of the `AudioManager` class body.
- **Flags Set**: `__rainSynthInstalled`, `__caveReverbInstalled`, `__tuAudioVisPatch` on prototype so monkey patches will skip.
- **Impact**: 
  - No more prototype chain modification for audio methods
  - V8 can now optimize these methods with inline caches
  - `updateWeatherAmbience` is directly callable without closure overhead
  - `setEnvironment` properly initializes cave reverb on first call

### 4. TouchController Assessment
- **Result**: No monkey patches found targeting TouchController. Already clean.

## Files
- `part1_native_timing.html` - After Phase 1 changes

## Verification Checklist
- [x] game:init:post fires exactly once per init
- [x] InputManager blur/mouseleave/wheel handlers bound natively
- [x] Monkey patch guard flags prevent double-binding
- [x] AudioManager rain synth methods are native class methods
- [x] AudioManager cave reverb methods are native class methods
- [x] Prototype flags prevent monkey patches from re-installing
