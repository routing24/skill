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

// 3) Start a fresh plan FIRST — it commits an EMPTY plan (replacing the
//    loaded one); everything below, saved addresses included, lands in it.
await __r24call('routing24_new_plan', {});

// 3b) (Optional) Confirm doubtful addresses — a routing24_upsert_addresses
//    batch of at most 5 rows returns `rows`, one per input, with status
//    ('geocoded' | 'ungeocoded') and the canonical `matched` text. Show the
//    user ungeocoded rows and sanity-check `matched` before optimizing; stop
//    rows sent later with the SAME address strings reuse these locations.
await __r24call('routing24_upsert_addresses', {
    addresses: [{ address: "DEPOT ADDRESS" }, { address: "STOP 1 ADDRESS" }],
});

// 4) Build the plan piece by piece, then solve. Every row needs an `id` (the
//    business id every other tool refers to). The address string is the only
//    location carrier — a decimal "lat, lng" literal works as an address for
//    callers holding exact coordinates. Geocode failures come back in
//    addressDiagnostics; resolve them before solving.
await __r24call('routing24_upsert_depots', {
    depots: [{ id: "D1", address: "DEPOT ADDRESS" }],
});
await __r24call('routing24_upsert_vehicles', {
    vehicles: [
        // tags satisfy stops' required_tags. max_reloads caps mid-route reloads
        // (multi-trip); vehicles reload at the depot by default when needed.
        // break_rules: EU driver-break preset shown (45 min before exceeding
        // 4.5 h driving, splittable 15+30);
        // US = { max_driving_s: 28800, duration_s: 1800, service_counts: true }. Omit for no breaks.
        // cost: only when the user gives real rates (per km/mi, per hour,
        // fixed per use) — omit it entirely (as here) to minimize
        // distance and time at the defaults of 1 per mile(km) + 1 per
        // hour; never send zeros for that. Explicit 0 is for mixed
        // fleets: { cost: { distance: 0, duration: 6 } } bike vs
        // { cost: { distance: 2, duration: 18, fixed: 40 } } van.
        {
            id: "V1",
            available_count: 2, capacity: 20, tw_early_s: 8 * 3600, tw_late_s: 18 * 3600, tags: ["reefer"], max_reloads: 2,
            break_rules: [{ max_driving_s: 16200, duration_s: 2700, split_first_s: 900, split_second_s: 1800 }],
        },
    ],
});
await __r24call('routing24_upsert_stops', {
    stops: [
        // priority (1..1000, default 1): higher-priority stops are kept when not
        // everything fits. required_tags: only a vehicle carrying these tags may serve it.
        { id: "S1", address: "STOP 1 ADDRESS", delivery: 1, service_duration_s: 300, priority: 10 },
        { id: "S2", address: "STOP 2 ADDRESS", delivery: 1, service_duration_s: 300, required_tags: ["reefer"] },
    ],
});
await __r24call('routing24_reoptimize_plan', {}); // options: { time_limit_s: 30 }

// 5) Poll by looping on routing24_status until phase is 'done' or 'error': while a solve runs each call holds its reply up to ~15s (returning early when the solve lands), so call it back-to-back — no sleep between calls is needed. Relay progress meanwhile.
await __r24call('routing24_status');

// 6) Show routes on the map, then persist and get the plan link.
await __r24call('routing24_render');
await __r24call('routing24_save'); // -> { saved, planUrl }

// 7) Solution overview: rollups + economic cost/objective + per-route
//    `routeStats` (vehicle, stop count, distance, duration, feasibility, cost;
//    each with its 1-based `route` number). No stop dump. Works now and on a
//    plan reopened by URL.
await __r24call('routing24_status');
// 7b) One route's ordered stops (resolved id/address, ETAs, loads,
//     waits, problems) by its 1-based `route` number from routeStats.
await __r24call('routing24_route', { route: 1 });

// 8) Current plan URL (also returned by save).
await __r24call('routing24_plan_url');

// 9) Manual editing — one tool per action, each atomic on its own. Problems
//    don't reject: judge state.feasible / userAssignedReports after each.
await __r24call('routing24_edit_move_stops', { stops: ['S12', 'S13'], after: 'S07' });
await __r24call('routing24_edit_unassign_stops', { stops: ['S99'] });
await __r24call('routing24_edit_set_route_vehicle', { route: 2, vehicle: 'VAN-3' });

// 10) Plan an unassigned stop onto Route 1 at the engine-chosen cheapest
//     position, then tidy the sequence (sub-second, synchronous).
await __r24call('routing24_edit_move_stops', {
    stops: ['S99'],
    to_route: 1,
    placement: 'best',
});
await __r24call('routing24_reoptimize_route', { route: 1 });

// 11) Delete a route. Any stops still on it are unassigned in the SAME atomic
//     edit; the remaining routes renumber — re-read them from the returned state.
await __r24call('routing24_edit_remove_routes', { routes: [3] });

// 12) Undo the last edit (SHARED history with the user's manual edits — never
//     undo speculatively). Redo mirrors it.
await __r24call('routing24_undo');

// 13) Why are stops unassigned? Prose diagnostics (summary + per-site
//     explanation/blockers/levers), recomputed on demand.
(await __r24call('routing24_unassigned', { refresh_diagnostics: true })).unassignedDiagnostics;

// Optional: cancel a long-running solve (keeps the best solution so far).
await __r24call('routing24_cancel');

// Optional single blocking wait (prefer separate status calls to stream progress).
// Poll by looping on routing24_status until phase is 'done' or 'error': while a solve runs each call holds its reply up to ~15s (returning early when the solve lands), so call it back-to-back — no sleep between calls is needed.
await (async () => {
    for (let i = 0; i < 400; i++) {
        const s = await __r24call('routing24_status');
        if (s.phase === "done" || s.phase === "error") return s;
    }
    return __r24call('routing24_status');
})();
```
