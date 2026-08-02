# Routing24 route optimizer — WebMCP call snippets

Ready-to-eval expressions for `javascript_tool` — each resolves to a value that
comes back to you. Replace the ADDRESS/STOP/VEHICLE placeholders.

```js
// 0) Discover the tools (undefined/missing names => page still loading; retry).
const mc = document.modelContext ?? navigator.modelContext;
(await mc?.getTools())?.map(t => t.name) ?? null;

// 1) One-time call helper: executeTool takes the tool OBJECT + a JSON-string
//    argument and resolves to a JSON string (rejects on errors).
const mc2 = document.modelContext ?? navigator.modelContext;
const tools = await mc2.getTools();
window.__r24call = async (name, args = {}) => {
    const tool = tools.find(t => t.name === name);
    if (!tool) throw new Error('tool not found: ' + name);
    const raw = await mc2.executeTool(tool, JSON.stringify(args));
    return raw == null ? null : JSON.parse(raw);
};

// 2) (Optional) who's signed in — { user: email | "anonymous" }. Never gate on
//    this; the whole flow works anonymously. It only affects where the plan is
//    stored.
await __r24call('routing24_get_auth_user');

// 3) Geocode depot + all stop addresses. Inspect results for ok:false and
//    sanity-check the `matched` strings with the user before optimizing.
await __r24call('routing24_geocode', {
    addresses: ["DEPOT ADDRESS", "STOP 1 ADDRESS", "STOP 2 ADDRESS"],
});

// 4) Start optimization. Prefer passing lat/lng from step 3 so nothing is
//    re-geocoded (addresses alone also work; they'll be geocoded).
await __r24call('routing24_optimize', {
    depot: { address: "DEPOT ADDRESS" /*, lat, lng */ },
    stops: [
        // priority (1..1000, default 1): higher-priority stops are kept when not
        // everything fits. required_tags: only a vehicle carrying these tags may serve it.
        { id: "S1", address: "STOP 1 ADDRESS", delivery: 1, service_duration_s: 300, priority: 10 },
        { id: "S2", address: "STOP 2 ADDRESS", delivery: 1, service_duration_s: 300, required_tags: ["reefer"] },
    ],
    vehicles: [
        // tags satisfy stops' required_tags. max_reloads caps mid-route reloads
        // (multi-trip); vehicles reload at the depot by default when needed.
        // break_rules: EU driver-break preset shown (45 min before exceeding
        // 4.5 h driving, splittable 15+30); US = { max_driving_s: 28800,
        // duration_s: 1800, service_counts: true }. Omit for no breaks.
        // cost: only when the user gives real rates (per km/mi, per hour,
        // fixed per use) — omit it entirely (as here) to minimize plain
        // distance; never send zeros for that. Explicit 0 is for mixed
        // fleets: { cost: { distance: 0, duration: 6 } } bike vs
        // { cost: { distance: 2, duration: 18, fixed: 40 } } van.
        {
            available_count: 2, capacity: 20, tw_early_s: 8 * 3600, tw_late_s: 18 * 3600, tags: ["reefer"], max_reloads: 2,
            break_rules: [{ max_driving_s: 16200, duration_s: 2700, split_first_s: 900, split_second_s: 1800 }],
        },
    ],
    // options: { time_limit_s: 30 },
});

// 5) Poll (~every 3s) until phase is "done" or "error". Relay progress meanwhile.
await __r24call('routing24_status');

// 6) Show routes on the map, then persist and get the plan link.
await __r24call('routing24_render');
await __r24call('routing24_save'); // -> { saved, planUrl }

// 7) Full solution: per-route-slot ordered stops (resolved id/address/lat/lng,
//    ETAs, loads, waits, problems) + unassigned list, user-assigned marks and
//    undo/redo depths. Works now and on a plan reopened by URL. Route `slot`
//    is the address the editing tools use.
await __r24call('routing24_solution');

// 8) Current plan URL (also returned by save).
await __r24call('routing24_plan_url');

// 9) Manual editing — ONE atomic batch: move two stops after S07 on their
//    route, drop S99 to unassigned, and rebind route slot 2 to vehicle VAN-3.
//    Problems don't reject: judge state.feasible / userAssignedReports.
await __r24call('routing24_edit', {
    ops: [
        { op: 'move_sites', sites: ['S12', 'S13'], after: 'S07' },
        { op: 'unassign_sites', sites: ['S99'] },
        { op: 'set_route_vehicle', route: 2, vehicle: 'VAN-3' },
    ],
});

// 10) Plan an unassigned stop onto route slot 1 at the engine-chosen cheapest
//     position, then tidy the sequence (sub-second, synchronous).
await __r24call('routing24_edit', {
    ops: [{ op: 'move_sites', sites: ['S99'], to_route: 1, placement: 'best' }],
});
await __r24call('routing24_optimize_route', { route: 1 });

// 11) Empty a route and remove it (remove_routes takes EMPTY routes only, so
//     unassign first; slots compact — re-read them from the returned state).
await __r24call('routing24_edit', {
    ops: [
        { op: 'unassign_sites', sites: ['S3', 'S4'] },
        { op: 'remove_routes', routes: [3] },
    ],
});

// 12) Undo the last edit (SHARED history with the user's manual edits — never
//     undo speculatively). Redo mirrors it.
await __r24call('routing24_undo');

// 13) Why are stops unassigned? Prose diagnostics (summary + per-site
//     explanation/blockers/levers), recomputed on demand.
(await __r24call('routing24_solution', { refresh_diagnostics: true })).unassignedDiagnostics;

// Optional: cancel a long-running solve (keeps the best solution so far).
await __r24call('routing24_cancel');

// Optional single blocking wait (prefer separate status calls to stream progress):
await (async () => {
    for (let i = 0; i < 400; i++) {
        const s = await __r24call('routing24_status');
        if (s.phase === "done" || s.phase === "error") return s;
        await new Promise((r) => setTimeout(r, 3000));
    }
    return __r24call('routing24_status');
})();
```
