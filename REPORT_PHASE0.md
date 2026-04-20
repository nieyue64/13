# Phase 0: Safety Net & Performance Baselines - Report

## Completion Status: DONE

## Changes Made

### 1. TU.Safe.run Enhancement (Line ~4981)
- **What**: Extracted error handling into shared `_handleError()` function
- **Added**: `runAsync(tag, fn, opts)` - async-safe error boundary that catches both sync throws AND Promise rejections
- **Improved**: All caught errors now emit `error:caught` event for centralized monitoring
- **Improved**: Error logging always goes to `console.error` with `[TU.Safe]` prefix - never silently swallowed
- **Impact**: Worker hangs, audio race conditions, and IDB write failures will now surface to console

### 2. IndexedDB Error Classification (Line ~8675)
- **What**: Added `_classifyError(e, op)` method to `IndexedDBStorage` class
- **Behavior**: Classifies errors as QuotaExceeded, AbortError, or DataCorruption
- **Critical**: `set()` now **re-throws** QuotaExceededError instead of returning `false` - save-critical path must not silently fail
- **Added**: `storage:error` event emission for monitoring
- **Added**: Automatic toast notification for quota errors
- **Impact**: "Zombie bad saves" (IDB write silently fails) will now surface as explicit errors

### 3. Canvas getContext Hijack Scoped (Line ~5401)
- **What**: Replaced global `HTMLCanvasElement.prototype.getContext` override with scoped `window.TU_CanvasOptimizer`
- **Before**: Every `getContext("2d")` call in the entire page was intercepted and modified
- **After**: `TU_CanvasOptimizer.optimize(ctx)` is called explicitly only on the Renderer's main canvas context
- **Impact**: Third-party libraries, UI canvases, OffscreenCanvas no longer have modified prototype behavior

### 4. GlobalErrorBoundary (Line ~5548)
- **What**: Added `window.TU_ErrorBoundary` / `TU.ErrorBoundary` with performance baseline tracking
- **Features**:
  - `recordFrameTime(ms)` / `recordTickTime(ms)` - rolling window of 120 samples
  - `getBaselines()` - returns p50/p95/p99 for frame time and tick time
  - `report()` - prints baselines to console
  - Fatal/warning counters fed by `error:caught` events
- **Wired**: Into `Renderer._updatePerfMonitor()` to automatically record frame times
- **Impact**: Can now measure performance regression/improvement across phases

### 5. Unhandled Promise Rejection Visibility (Line ~5539)
- **What**: Added explicit `console.error()` call in `unhandledrejection` handler
- **Before**: Unhandled promise rejections were only queued to ErrorReporter (might not be ready)
- **After**: Always logged to console with `[GlobalErrorBoundary]` prefix
- **Impact**: Worker Promise hangs and async failures now visible in devtools

## Files
- `part0_original_backup.html` - Original untouched file
- `part0_safety_net.html` - After Phase 0 changes

## Verification Checklist
- [x] TU.Safe.run still works for sync code (backward compatible)
- [x] TU.Safe.runAsync added for async code
- [x] IndexedDB quota errors surface to user
- [x] Canvas optimization still applied to game canvas
- [x] Global prototype no longer hijacked
- [x] Performance baselines can be collected
- [x] Error events emitted for monitoring
