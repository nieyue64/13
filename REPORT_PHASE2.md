# Phase 2: Render Pipeline & API Isolation - Report

## Completion Status: DONE

## Changes Made

### 1. PostFX Pipeline Infrastructure (Line ~21148)
- **Problem**: `Renderer.prototype.applyPostFX` was wrapped by 4 nested closures (weather tint, weather optimized, underwater fog, cloud biome sky), creating the "onion model":
  - Layer 4 (outermost): `__underwaterFogInstalled` -> calls prev
  - Layer 3: `__weatherPostTintOptimized` -> calls prev
  - Layer 2: `__weatherPostTintInstalled` -> calls prev
  - Layer 1 (innermost): Native applyPostFX (bloom, fog, vignette, grain)
- **Fix**: Added `Renderer._postFxPipeline` array and `registerPostFxStage(name, order, fn)` method
- **Behavior**: After the native applyPostFX completes its base effects, it iterates `_postFxPipeline` and calls each stage linearly with error boundaries
- **API**:
  ```javascript
  renderer.registerPostFxStage('weatherTint', 10, (renderer, ctx, time, depth01, reducedMotion) => { ... });
  renderer.registerPostFxStage('underwaterFog', 20, (renderer, ctx, time, depth01, reducedMotion) => { ... });
  ```
- **Impact**: New effects can register as pipeline stages instead of wrapping. Existing wrapping still works (backward compatible) but new code should use the pipeline.

### 2. Canvas getContext Hijack Replacement (from Phase 0)
- Already replaced the global `HTMLCanvasElement.prototype.getContext` override with scoped `TU_CanvasOptimizer.optimize()` in Phase 0.
- The Renderer now calls `TU_CanvasOptimizer.optimize(this.ctx)` directly on its main canvas context only.

### 3. Centralized Weather State (Line ~23554)
- **Problem**: Weather state was scattered across 5+ locations:
  - `game.weather` (partial object, lazily initialized in patches)
  - `window.AppServices.get('weatherFx')` (separate object with postFX params)
  - CSS classes on document body
  - Inline variables in various patch closures
- **Fix**: `game.weather` is now initialized as a complete canonical object in the Game constructor with ALL required fields:
  - Core: `type`, `intensity`, `targetIntensity`, `nextType`, `nextIntensity`, `lightning`, `acid`
  - PostFX derived: `postMode`, `postR/G/B`, `postA`, `shadowAlpha`, `shadowColor`
- **Registered**: As `window.AppServices.register('weather', this.weather)` and also as `weatherFx` for backward compatibility
- **Impact**: All subsystems can now read `window.AppServices.get('weather')` instead of maintaining their own weather copies

## Architecture Notes

### Existing Onion Layers (Still Active - Backward Compatible)
The existing monkey patches still function because they wrap `Renderer.prototype.applyPostFX` before the pipeline runs. The execution order is:

1. Outermost wrapper calls -> next wrapper calls -> ... -> native applyPostFX
2. Native applyPostFX runs base effects (bloom, fog, vignette, grain)
3. Native applyPostFX runs `_postFxPipeline` stages (currently empty)
4. Native applyPostFX emits `renderer:postFX:complete`
5. Control returns to wrapper, which adds its own layer

### Migration Path for Future Work
To fully linearize: move weather tint and underwater fog logic into `registerPostFxStage()` calls and remove the `Renderer.prototype.applyPostFX` wrapping. This would eliminate the closure chain entirely.

## Files
- `part2_pipeline_isolation.html` - After Phase 2 changes

## Verification Checklist
- [x] PostFX pipeline mechanism added to Renderer
- [x] registerPostFxStage() API available
- [x] Each stage has error boundary (try/catch)
- [x] Pipeline stages sorted by order
- [x] Duplicate stage names prevented
- [x] Centralized weather state in Game constructor
- [x] Weather registered with AppServices
- [x] Backward compatible with existing monkey patches
