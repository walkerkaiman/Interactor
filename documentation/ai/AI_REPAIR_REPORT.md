# Interactor Happy‑Path Audit and Repair Plan (post‑refactor)

This document maps the primary happy paths end‑to‑end and pinpoints miswirings introduced in the refactor. It includes exact, minimal edits for a naive AI agent to implement, plus strict guard‑rail ideas to prevent regressions.

---

## TL;DR — What’s broken and why

- Time→Output routing no longer delivers messages:
  - Routes are remapped to the output module’s first input event (e.g., `audioPlay`) during registration, so they no longer match the source event (`timeTrigger`).
  - Output module instances are not created on register, so there’s nothing to receive routed messages.
  - Router wiring for outputs looks up instances from the `ModuleLoader` (which returns `undefined`) instead of the live instance registry held by the server.
- Frontend uses an endpoint (`POST /api/interactions/sync`) that doesn’t exist on the backend.
- Audio upload flow can create a second, detached Audio Output instance, drifting from the one on the graph.

---

## Architecture refresher (short)

- Backend: `InteractorServer` exposes REST + WS, tracks live instances, persists state via `StateManager`, loads manifests via `ModuleLoader`, routes messages via `MessageRouter`.
- Frontend: React node editor builds interactions and registers via REST. WebSocket is for runtime values and pulses.
- Manifests: Identify modules and their events. Input modules emit events (e.g., `timeTrigger`), outputs subscribe via router wiring.

---

## Happy paths and breakpoints

### 1) Load manifests → Sidebar
- Expected: `/api/modules` → manifests render in Sidebar → drag to canvas.
- Backend: `ModuleLoader` scans `backend/src/modules/**/manifest.json`.
- Status: OK.

Key code
```1:67:/workspace/backend/src/core/ModuleLoader.ts
export class ModuleLoader {
  // ... existing code ...
  private async loadModules(): Promise<void> {
    // Recursively search for modules
  }
  // ... existing code ...
}
```

### 2) Build interactions → Register
- Expected: Drag creates local interaction; clicking Register POSTs to `/api/interactions/register`; server persists interactions + creates live instances for inputs and outputs; router wired.
- Current:
  - Server creates live instances only for certain inputs (e.g., `Time Input`). Outputs are not instantiated.
  - Server remaps route.event for output targets to the output’s first input event name, breaking event equality matching.
  - Router wiring looks up output instances via `ModuleLoader.getInstance()` which returns `undefined` and never falls back, so no wiring is established.

Breakpoints
- Event remap during register:
```678:703:/workspace/backend/src/index.ts
interactionMap.forEach(interaction => {
  if (interaction.routes) {
    interaction.routes.forEach(route => {
      const targetModule = interaction.modules.find(m => m.id === route.target);
      if (targetModule) {
        const manifest = this.moduleLoader.getManifest(targetModule.moduleName);
        if (manifest?.type === 'output' && manifest.events) {
          const inputEvents = manifest.events.filter(e => e.type === 'input');
          if (inputEvents.length > 0 && inputEvents[0]?.name) {
            const mappedRoute = {
              ...route,
              event: inputEvents[0].name
            };
            messageRouter.addRoute(mappedRoute);
          }
        } else {
          messageRouter.addRoute(route);
        }
      } else {
        messageRouter.addRoute(route);
      }
    });
  }
});
```
- Output instance wiring uses `ModuleLoader.getInstance(id)` which always returns `undefined`:
```1253:1292:/workspace/backend/src/index.ts
private setupTriggerEventListeners(): void {
  const moduleInstances = this.stateManager.getModuleInstances();
  moduleInstances.forEach(moduleInstance => {
    const manifest = this.moduleLoader.getManifest(moduleInstance.moduleName);
    if (manifest?.type === 'output') {
      const actualModule = this.moduleLoader.getInstance(moduleInstance.id);
      if (actualModule) {
        // router.on(moduleId, ...) → deliver to actualModule
      }
    }
  });
}
```
- `ModuleLoader.getInstance` stub:
```176:182:/workspace/backend/src/core/ModuleLoader.ts
public getInstance(id: string): any {
  // Simplified implementation - return undefined since we don't track instances
  return undefined;
}
```

Expected flow
- Do not remap route events. Router should match the source event name (e.g., `timeTrigger`).
- On register, create live instances for both inputs and outputs.
- Wire router listeners against the server’s `moduleInstances` registry, not `ModuleLoader`.

### 3) Time Input runtime updates
- Expected: `TimeInputModule` emits runtime fields (`currentTime`, `countdown`) over WS without overwriting config.
- Status: OK. Uses `emitRuntimeStateUpdate` and server relays targeted updates.

Key code
```136:151:/workspace/backend/src/modules/input/time_input/index.ts
this.engine.on('pulse', () => {
  this.emitTrigger('timeTrigger', {
    mode: 'metronome',
    millisecondDelay: this.millisecondDelay,
    currentTime: new Date().toISOString(),
    timestamp: Date.now(),
  });
});
```

### 4) Time → Audio trigger delivery
- Expected: `timeTrigger` → `MessageRouter` → `AudioOutputModule.onTriggerEvent` → playback via `onSend(payload)`.
- Current: Fails because of (a) event remap, (b) missing output instance, (c) router wiring to `ModuleLoader`.

Key code (Audio Output accepts any trigger payload)
```198:205:/workspace/backend/src/modules/output/audio_output/index.ts
protected async handleTriggerEvent(event: TriggerEvent): Promise<void> {
  await this.onSend(event.payload);
}
```

### 5) Manual triggers
- Expected: POST `/api/trigger/:moduleId` with `{type: 'manualTrigger'}` triggers source and emits WS pulse.
- Status: Works for Time Input; for outputs depends on (2)/(4) wiring.

Key code
```850:879:/workspace/backend/src/index.ts
if ('onManualTrigger' in moduleInstance && typeof moduleInstance.onManualTrigger === 'function') {
  await moduleInstance.onManualTrigger();
  this.broadcastTriggerEvent(req.params.moduleId as string, 'manual');
  return res.json({ success: true });
}
```

### 6) Audio file upload/listing
- Expected: Query file list → upload via shared uploader on port 4000.
- Current: If the graph’s audio instance doesn’t exist yet, frontend creates a separate instance using `/api/modules/instances`. This can drift from the graph’s ID.

Key code
```137:171:/workspace/frontend/src/api/index.ts
async getAudioFiles(moduleId: string) {
  const instance = await this.getModuleInstance(moduleId);
  if (!instance) {
    const createdInstance = await this.createAudioOutputInstance();
    if (!createdInstance) throw new Error('Failed to create Audio Output module instance');
    return this.getAudioFiles(createdInstance.id);
  }
  const uploadPort = instance.config.uploadPort ?? 4000;
  const response = await fetch(`http://localhost:${uploadPort}/files/audio-output`);
  // ... existing code ...
}
```

### 7) Config updates
- Expected: Local edits accumulate as unregistered changes; Register persists.
- Status: OK for “apply on register” workflow. There is also a live-update endpoint (`PUT /api/modules/instances/:id`) available if needed later.

### 8) WebSocket state
- Expected: On connect, `state_update` with interactions + moduleInstances. Then targeted `module_runtime_update` and `trigger_event`.
- Status: OK.

---

## Minimal repair plan (actionable edits)

1) Remove event remap during registration
- Keep routes’ `event` exactly as authored by the source (e.g., `timeTrigger`).
- Edit `backend/src/index.ts` in `/api/interactions/register`:
  - Delete the branch that overrides `route.event` for output targets; always `addRoute(route)`.
  - Post‑edit behavior: `MessageRouter.findMatchingRoutes` will match on original `event`.

2) Always instantiate outputs on register
- In `/api/interactions/register`, after building `moduleInstances`, also create live instances for outputs (do not necessarily “start” them; just construct so they can receive routed messages). Reuse `createModuleInstance` for both input and output types.

3) Wire router listeners against live instance registry
- In `setupTriggerEventListeners()` use `this.moduleInstances.get(moduleInstance.id)` when the manifest is `type === 'output'`. If found, register:
  - `messageRouter.on(moduleInstance.id, (message) => { ... actualModule.onTriggerEvent(...) ... })`.
- Keep the special cases that translate generic events to output handlers, but do not filter by event name here—router matching already handled that.

4) Call `setupTriggerEventListeners()` after creating instances at registration time
- After creating both input and output instances in `/api/interactions/register`, call `this.setupTriggerEventListeners()` so newly created outputs are wired immediately.

5) Avoid creating detached audio instances from the frontend
- Prefer using the graph’s module ID. Since (2) instantiates outputs on register, the instance should exist once user has registered at least once.
- If a fallback is still needed, consider enhancing backend `/api/modules/instances` creation to preserve the requested `id` when provided, or reject duplicate “module type with different id” creations. For now, keep frontend code but document the drift risk.

6) Remove dead frontend call or add backend endpoint
- Frontend exposes `POST /api/interactions/sync` but backend has no such route. Either remove the method or implement a no‑op backend route that invokes `syncInteractionsWithModules()`.

---

## Precise code pointers for the naive agent

Apply these exact changes.

A) backend/src/index.ts — stop remapping route.event
- Remove the block that changes `route.event` to the target’s first input event.

Search for: `Mapped route for output module` and delete the entire if‑branch so we always `messageRouter.addRoute(route)`.

B) backend/src/index.ts — instantiate outputs after register
- In the loop that currently creates and starts inputs (search `// Create and start live module instances for input modules`), also:
  - For each module where `manifest.type === 'output'`, call `await this.createModuleInstance(moduleInstance);` but do not call `.start()` unless needed by that module.

C) backend/src/index.ts — fix setupTriggerEventListeners()
- Replace lookup:
  - From: `const actualModule = this.moduleLoader.getInstance(moduleInstance.id);`
  - To: `const actualModule = this.moduleInstances.get(moduleInstance.id);`
- Only if `actualModule` exists, register both trigger and stream handlers (existing code block is correct once instance is found).

D) backend/src/index.ts — rewire after register
- After creating instances in `/api/interactions/register`, invoke `this.setupTriggerEventListeners()` again to capture the newly created outputs.

E) Optional: backend — add `/api/interactions/sync` route (no‑op sync)
- Implement POST that calls `await this.syncInteractionsWithModules();` and returns success. Or remove frontend method.

---

## Verification checklist (manual)

- Start backend and frontend.
- Register a simple interaction: Time Input (metronome 1000ms) → Audio Output.
- Observe:
  - WS `state_update` includes both modules; Audio Output instance exists.
  - TimeInput pulses result in `trigger_event` and Audio Output `onSend(...)` call (logs show playback state changes even if actual audio is stubbed).
- Manual trigger: Trigger Time Input via UI; observe pulse and Audio Output reaction.
- Audio uploader: Open Audio Output node; list files, upload file, list updates. Confirm one Audio Output instance (graph id) is used.

---

## Strict guard‑rails for AI agents (to prevent regressions)

Implement the following safeguards so working features aren’t overwritten.

1) Route/event contract locks
- Add a shared constant type for event names and enforce equality matching only at the router. Disallow any code that mutates `route.event` during registration.
- Lightweight runtime assertion in `/api/interactions/register`: if code tries to change `route.event`, log error and reject the request in dev mode.

2) Instance wiring policy
- A single source of truth for live instances is `InteractorServer.moduleInstances`.
- Ban lookups from `ModuleLoader` for live instances via a lint rule (custom ESLint rule that flags `moduleLoader.getInstance(` usage outside the loader itself).

3) Protected regions annotations
- Annotate critical sections with sentinel comments, and add a pre‑commit script that fails if diff shows edits inside a protected region without updating a `RATIONALE.md`:
  - Backend: `/api/interactions/register`, `setupTriggerEventListeners`, `createModuleInstance`.
  - Core: `MessageRouter`, `StateManager` serialization semantics, `ModuleBase.emitRuntimeStateUpdate`.

4) Contract snapshots (no test harness required)
- Maintain “golden” JSON snapshots for:
  - `GET /api/modules` manifests (names/types/events)
  - `POST /api/interactions/register` request → stored state schema (no event remap)
  - WebSocket envelopes (`state_update`, `module_runtime_update`, `trigger_event`)
- A tiny CLI `scripts/contract-check.ts` that fetches endpoints from a running dev server and diffs against `documentation/contracts/*.json`. Fail CI if changed without updating the snapshots and `CHANGELOG.md`.

5) Schema checks at runtime
- Validate registered interactions against manifests:
  - Verify each route’s `event` exists on the source module’s manifest events (not the target).
  - Verify route `source`/`target` IDs exist in the submitted interaction.
  - Reject with a helpful error if invalid.

6) “AI Change Plan” gate
- Require an `AI_CHANGE_PLAN.md` entry for any PR that touches protected regions, including:
  - What behavior is changing and why
  - Affected flows
  - Rollback plan

7) Feature flags for risky changes
- Introduce a config flag `features.strictRouting=true` defaulting to true that disables any future attempt to remap events.
- Add `features.instantiateOutputsOnRegister=true` so an accidental removal is caught during boot checks.

8) Logging tripwires
- On boot, assert and log if:
  - Any output module instance exists in state without a matching live instance.
  - Router has routes targeting a module ID that has no live instance wired.
  - A route was added whose `event` doesn’t exist in the source module’s manifest.

9) Frontend API parity checks
- Add a small dev‑time check to hide buttons/features that call non‑existent endpoints (e.g., `/api/interactions/sync`), and log a console warning prompting to remove or implement the route.

10) Read‑only mode for manifests
- Treat `backend/src/modules/**/manifest.json` as read‑only in CI. Changes require a separate “manifest‑change” label and manual approval.

---

## Notes for the naive agent
- Do not change event names carried in routes. Router matches `{ source, event }` from the source module’s emissions.
- Always build output module live instances at registration time so router deliveries have a receiver.
- When wiring router → outputs, fetch instances from `InteractorServer.moduleInstances`.
- Keep WS envelopes runtime‑only for `module_runtime_update`. Do not push configuration over WS; REST owns config.
- Do not add or remove API endpoints used by the frontend without updating the other side (and this doc’s snapshots/notes).

---

## Appendix: Useful file map
- Backend server: `backend/src/index.ts`
- Loader: `backend/src/core/ModuleLoader.ts`
- Router: `backend/src/core/MessageRouter.ts`
- State: `backend/src/core/StateManager.ts`
- Module bases: `backend/src/modules/InputModuleBase.ts`, `backend/src/modules/OutputModuleBase.ts`, `backend/src/core/ModuleBase.ts`
- Time Input: `backend/src/modules/input/time_input/*`
- Audio Output: `backend/src/modules/output/audio_output/*`
- Frontend API: `frontend/src/api/index.ts`
- WebSocket client: `frontend/src/hooks/useWebSocket.ts`, `frontend/src/state/useBackendSync.ts`
- Node editor: `frontend/src/components/NodeEditor.tsx`, `frontend/src/components/*Node.tsx`