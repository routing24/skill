# Routing24 route optimizer — call snippets

One worked call per step of the flow: the tool, and the arguments it takes.
Over the connector, call them as written; on the WebMCP surface, wrap each one
as shown under *Driving the page directly*. Replace the ADDRESS/STOP/VEHICLE
placeholders.

```js
// 1) (Optional) who's signed in — { user: email | "anonymous" }. The connector
//    always runs signed in; "anonymous" happens only on the WebMCP surface,
//    where it decides where the plan is stored.
routing24_get_auth_user();

// 2) Start a fresh plan FIRST — it commits an EMPTY plan (replacing the
//    loaded one); everything below lands in it.
routing24_new_plan({});

// 2b) (Optional) Check doubtful addresses — routing24_geocode_addresses takes
//     at most 10 rows and returns `rows`, one per input, with status
//     ('geocoded' | 'ungeocoded') and the canonical `matched` text. It creates
//     nothing. Show the user ungeocoded rows and sanity-check `matched`; stop
//     rows sent later with the SAME address strings are not geocoded again.
routing24_geocode_addresses({
    addresses: [{ address: "DEPOT ADDRESS" }, { address: "STOP 1 ADDRESS" }],
});

// 3) Build the plan piece by piece, then solve. Every row needs an `id` (the
//    business id every other tool refers to). The address is geocoded
//    internally; a row you hold exact coordinates for pins them instead, with
//    `coordinates` or a "lat, lng" address literal. Geocode failures come back in
//    addressDiagnostics; resolve them before solving.
routing24_upsert_depots({
    depots: [{ id: "D1", address: "DEPOT ADDRESS" }],
});
routing24_upsert_vehicles({
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
routing24_upsert_stops({
    stops: [
        // priority (1..1000, default 1): higher-priority stops are kept when not
        // everything fits. required_tags: only a vehicle carrying these tags may serve it.
        { id: "S1", address: "STOP 1 ADDRESS", delivery: 1, service_duration_s: 300, priority: 10 },
        { id: "S2", address: "STOP 2 ADDRESS", delivery: 1, service_duration_s: 300, required_tags: ["reefer"] },
        // A stop you hold exact coordinates for: pinned, never geocoded.
        { id: "S3", coordinates: { lat: 25.19882, lng: 55.27939 }, delivery: 1 },
    ],
});
routing24_reoptimize_plan({}); // options: { time_limit_s: 30 }

// 4) Poll by looping on routing24_status until phase is 'done' or 'error': while a solve runs each call holds its reply up to ~15s (returning early when the solve lands), so call it back-to-back — no sleep between calls is needed. Relay progress meanwhile.
routing24_status();

// 5) Show routes on the map, then persist and get the plan link.
routing24_render();
routing24_save(); // -> { saved, planUrl }

// 6) Solution overview: rollups + economic cost/objective + per-route
//    `routeStats` (vehicle, stop count, distance, duration, feasibility, cost;
//    each with its 1-based `route` number). No stop dump. Works now and on a
//    plan reopened by URL.
routing24_status();
// 6b) One route's ordered stops (resolved id/address, ETAs, loads,
//     waits, problems) by its 1-based `route` number from routeStats.
routing24_route({ route: 1 });

// 7) Current plan URL (also returned by save).
routing24_plan_url();

// 8) Manual editing — one tool per action, each atomic on its own. Problems
//    don't reject: judge state.feasible / userAssignedReports after each.
routing24_edit_move_stops({ stops: ['S12', 'S13'], after: 'S07' });
routing24_edit_unassign_stops({ stops: ['S99'] });
routing24_edit_set_route_vehicle({ route: 2, vehicle: 'VAN-3' });

// 9) Plan an unassigned stop onto Route 1 at the engine-chosen cheapest
//    position, then tidy the sequence (sub-second, synchronous).
routing24_edit_move_stops({
    stops: ['S99'],
    to_route: 1,
    placement: 'best',
});
routing24_reoptimize_route({ route: 1 });

// 10) Delete a route. Any stops still on it are unassigned in the SAME atomic
//     edit; the remaining routes renumber — re-read them from the returned state.
routing24_edit_remove_routes({ routes: [3] });

// 11) Undo the last edit (SHARED history with the user's manual edits — never
//     undo speculatively). Redo mirrors it.
routing24_undo();

// 12) Why are stops unassigned? The result's `unassignedDiagnostics` carries a
//     summary plus per-site explanation/blockers/levers, recomputed on demand.
routing24_unassigned({ refresh_diagnostics: true });

// Optional: cancel a long-running solve (keeps the best solution so far).
routing24_cancel();
```
