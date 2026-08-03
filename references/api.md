# Routing24 route optimizer — API reference

> Generated from Routing24's own types (skill version 1.0.0). The
> always-current copy is served at https://routing24.com/llms.txt.

WebMCP tools registered on `document.modelContext` on every
`https://routing24.com/app/*` page (`navigator.modelContext` is a deprecated
alias). Discover them with `getTools()`; invoke with
`executeTool(tool, JSON.stringify(args))` — it resolves to a **JSON string of the
result object** (parse it) and **rejects** on validation/handler errors.
`routing24_optimize` is **fire-and-poll**: it returns immediately after starting
the background WASM solve; observe it with `routing24_status`. Shapes reuse the
solver's own `Site`/`VehicleType`/`Location` fields and are validated at
runtime — the JSON Schema is authoritative. Zero-argument tools take `'{}'`.

### `routing24_get_auth_user` → `{ user: string }`
No input. Returns the signed-in user's email, or `"anonymous"` when nobody is
logged in. **Informational only** — every tool works anonymously, so never gate
the flow on this or ask the user to sign in. It only tells you *where* a saved
plan will live (see `routing24_save`).

### `routing24_geocode` — `GeocodeInput` → `{ results: GeocodeResult[] }`
Batch-geocodes address strings (order preserved; every input gets a row back).
`ok:false` (`quality:"failed"`) = not found → ask the user to correct it.
```ts
// Input for the `routing24_geocode` WebMCP tool.
type GeocodeInput = {
    addresses: string[];  // Address strings to geocode (free-form, as a user would type them).
};
```
```ts
type GeocodeResult = {
    input: string;  // The address string as passed in.
    ok: boolean;
    matched?: string;  // Canonical/expanded address the geocoder matched.
    lat?: number;
    lng?: number;
    quality: "rooftop" | "failed";
};
```

### `routing24_optimize` — `OptimizeInput` → `OptimizeStarted`
Creates a fresh plan, fetches the O/D matrix, and starts the solve. Returns at once.
Fields marked `PRO`/`STARTER` below need a paid plan — see *Plans & paid
features*; `paidFeatures` in the result says what this account actually got.
```ts
type OptimizeInput = {
    depot: OptimizeDepot;  // Single depot, used as both start and end for every vehicle.
    stops: OptimizeStop[];  // min 1
    vehicles: OptimizeVehicle[];  // min 1
    options?: { time_limit_s?: integer };
};
```
```ts
// The single depot: a place plus depot constraints (open window, handling).
type OptimizeDepot = {
    lat?: number;
    lng?: number;
    address?: string;
    tw_early_s?: number;
    tw_late_s?: number;
    service_duration_s?: number;
    no_break?: boolean;
    id?: string;
};
```
```ts
// A delivery/pickup stop: a place plus solver site constraints.
type OptimizeStop = {
    lat?: number;
    lng?: number;
    address?: string;
    tw_early_s?: number;
    tw_late_s?: number;
    service_duration_s?: number;
    no_break?: boolean;
    pickup?: number;
    delivery?: number;
    release_time_s?: number;
    priority?: number;
    required_tags?: string[];  // PRO: Skills (vehicle & order tags)
    forbidden_tags?: string[];  // PRO: Skills (vehicle & order tags)
    group?: string;  // PRO: Alternative order groups, Alternative pickup/delivery locations
    transfer_type?: "pickup" | "delivery" | "depot";  // PRO: Pickup & delivery (transfers)
    transfer_id?: string;  // PRO: Pickup & delivery (transfers)
    sequence_group?: string;  // PRO: Order sequences
    sequence_rank?: number;  // PRO: Order sequences
    id?: string;
};
```
```ts
// A vehicle (type): solver vehicle constraints; `available_count` clones this
// type. The three reference lists take **stop/depot ids** (as passed in this
// request) and mirror task.fbs `Vehicle.force_allow_sites` /
// `force_deny_sites` / `reload_depots`.
type OptimizeVehicle = {
    tw_early_s?: number;
    tw_late_s?: number;
    capacity?: number;
    available_count?: number;
    cost?: { fixed?: number; distance?: number; duration?: number; stop?: number; overtime?: number };  // PRO: Overtime cost per hour (cost.overtime)
    start_late_s?: number;
    tags?: string[];  // PRO: Skills (vehicle & order tags)
    max_reloads?: number;
    max_duration_s?: number;  // PRO: Max duration
    max_overtime_s?: number;  // PRO: Max overtime
    max_distance?: number;  // PRO: Max distance
    break_rules?: { max_driving_s?: number; duration_s?: number; split_first_s?: number; split_second_s?: number; service_counts?: boolean }[];  // PRO: Driver breaks & driving limits
    fixed_breaks?: { tw_early_s?: number; tw_late_s?: number; duration_s?: number }[];  // PRO: Driver breaks & driving limits
    period_driving_limit_s?: number;  // PRO: Driver breaks & driving limits
    period_driven_s?: number;  // PRO: Driver breaks & driving limits
    id?: string;
    force_allow_sites?: string[];  // PRO: Force allow / deny orders — Stop ids ONLY this-typed vehicles may serve (pinned compatibility).
    force_deny_sites?: string[];  // PRO: Force allow / deny orders — Stop ids this vehicle type must never serve.
    reload_depots?: string[];  // PRO: Reload depots — Depot ids where mid-route reloads may happen (multi-trip). Omitted = single-trip unless `max_reloads` is set (then the home depot is used); pair with `max_reloads` to cap trips.
};
```
```ts
// Result of `routing24_optimize`: the solve is running, poll for progress.
type OptimizeStarted = {
    started: true;
    planUuid: string;
    paidFeatures?: PaidFeatureReport;  // Present only when the plan uses paid features outside this account.
    warnings?: string[];  // Non-blocking notes about how the request was interpreted (e.g. an all-zero cost model replaced by the plain-distance default). Omitted when there is nothing to say.
};
```
```ts
// What `routing24_optimize` did with the paid features the plan uses. Absent
// when the plan uses none, or when every one of them is included in the user's
// subscription — the common case, and nothing to report.
type PaidFeatureReport = {
    locked: string[];  // Paid capabilities the plan uses that this account's plan does not include, as `Feature` member names (e.g. `"Skills"`, `"DriverBreaks"`).
    stripped: string[];  // The subset DROPPED from this solve because the free allowance for paid features is spent. Empty while the allowance lasts. A non-empty list means the routes ignore those constraints — say so when reporting the plan, and name the upgrade as the way to get them back.
    freeRunsLeft: number;  // Free solves left that may use locked features (0 once spent).
};
```
- Each depot/stop needs **either** `lat`+`lng` **or** an `address` (geocoded
  automatically; if any fail, the call rejects listing them).
- Times are **seconds since midnight**. `delivery`/`pickup` are single-dimension
  loads. `available_count` = identical vehicles of that type. The single depot is
  both start and end.
- Stops sharing a `group` are mutually exclusive alternatives: the solver
  serves at most one per group and reports the rest as `alternativesNotChosen`.
- **Alternative pickup/delivery locations**: several stops with the same
  `transfer_id` and role (all `pickup` or all `delivery`), sharing one
  `group` of their own (equal loads), are candidate locations for that end —
  the solver serves exactly one candidate per role and reports the rest as
  `alternativesNotChosen`. Max 8 candidates per role, 16 pickup x delivery combinations.
- **Sequences**: stops sharing a `sequence_group` are served all-or-none, by
  ONE vehicle, in non-decreasing `sequence_rank` order (other stops may come
  between them). Plain stops only (no transfers, no `group`), one service
  window each, 2..16 stops per group.
- Vehicle `force_allow_sites`/`force_deny_sites`/`reload_depots` reference the
  stop/depot ids **from this request**; unknown ids reject the call.
  `max_distance` is in the plan's display unit (km/mi); `cost.duration` and
  `cost.overtime` are per hour (`max_overtime_s` caps paid overtime past
  `max_duration_s`).
- **Vehicle costs**: `cost.distance` is per km/mi (the plan's display unit),
  `cost.fixed` per vehicle used. An omitted cost field means 0. To simply
  minimize travel distance, **omit `cost` entirely — do not send zeros**: a
  fleet whose every distance/duration rate is 0 has nothing to optimize, so
  the zeros are ignored, plain distance is minimized, and the result carries a
  `warnings` entry. An explicit 0 next to priced vehicles is honored (a bike
  with free mileage beside a van at 2/km steers mileage onto the bike).
- **Driver breaks** (per vehicle). `break_rules` (max 1): a driving-trigger
  rule — after `max_driving_s` of accumulated driving the driver needs a
  `duration_s` pause; optional `split_first_s`/`split_second_s` allow taking
  it as two ordered parts; `service_counts: true` lets any contiguous
  non-driving time (service, waiting) of the required length satisfy the rule.
  Legal presets: **EU** (561/2006) `{ max_driving_s: 16200, duration_s: 2700, split_first_s: 900, split_second_s: 1800 }`;
  **US** (FMCSA) `{ max_driving_s: 28800, duration_s: 1800, service_counts: true }`.
  `fixed_breaks` (max 2): compulsory clock-window pauses (e.g. lunch) that
  must START inside `[tw_early_s, tw_late_s]`. Carry-in allowance:
  `period_driving_limit_s` minus `period_driven_s` (externally tracked, per
  driver) caps the route's total driving time (EU week = 201600, US 7-day = 216000).
  Stops/depot take `no_break: true` to forbid hosting a break there.
  Planned breaks come back as `type:"break"` stops in `routing24_solution`.

### `routing24_status` → `OptimizeStatus`
No input. Snapshot of the current optimization — poll (~every 3s) while solving.
`phase` walks `idle → geocoding → matrix → solving → saving → done` (or `error`).
```ts
type OptimizeStatus = {
    phase: "idle" | "geocoding" | "matrix" | "solving" | "saving" | "done" | "error";
    running: boolean;
    progress?: number;  // 0..1 while solving, when available.
    feasible?: boolean;
    routes?: number;
    stops?: number;
    unassignedCount?: number;
    distance?: number;
    distanceUnit?: "km" | "mi";
    durationHours?: number;
    problemsCount?: number;  // Total constraint-problem markers across all routes (0 = feasible).
    driftFromOptimized?: OptimizedDrift;  // Drift vs the last full optimization (absent while a solve runs).
    error?: string;
    planUuid?: string;
};
```

### `routing24_solution` — `SolutionInput` → `PlanSolution`
The **full** solution for the currently-loaded plan — whether you just
optimized it or opened it via its URL — with no parsing needed. Same rollups as
`routing24_status`, plus one entry per route **slot** (empty routes included;
`slot` is the address the editing tools use) with the ordered
depot→stops→depot sequence, each stop resolved to its
`id`/`address`/`lat`/`lng`, arrival & departure times (seconds since
midnight, physical timeline), per-leg distance/duration, loads, waits and
problems — plus the unassigned list, user-assigned/user-unassigned marks,
`revision` and undo/redo depths. Pass `{ refresh_diagnostics: true }` to
recompute `unassignedDiagnostics` when `diagnosticsStale` is true. Returns
`{ available: false }` when the plan has no solution yet — call it once
`routing24_status` reports `phase:"done"`.
```ts
// Input for `routing24_solution`.
type SolutionInput = {
    refresh_diagnostics?: boolean;  // Recompute the unassigned-site diagnostics before returning them (runs an engine pass; only needed when `diagnosticsStale` was true).
};
```
```ts
// The full optimized solution for the currently-loaded plan — whether it was
// just optimized in this tab or opened via its URL. This is the structured data
// behind {@link OptimizeStatus}: the same rollups plus every route's ordered
// stops. `available` is `false` (and the detail fields are omitted) when the
// loaded plan has no solution yet.
type PlanSolution = {
    available: boolean;  // `false` when the loaded plan carries no optimized solution yet.
    planUuid?: string;
    planUrl?: string;  // Absolute URL of the plan's optimize page.
    feasible?: boolean;  // True when every constraint is satisfied.
    complete?: boolean;  // True when the solve ran to completion (not cancelled part-way).
    distanceUnit?: "km" | "mi";
    routeCount?: number;  // Number of routes used.
    stopCount?: number;  // Number of site stops served across all routes.
    unassignedCount?: number;  // Number of site stops that could not be served.
    distance?: number;  // Total travel distance across all routes, in `distanceUnit`.
    durationHours?: number;  // Total duration across all routes, hours.
    unassigned?: string[];  // Ids of the sites left unassigned, if any.
    cost?: SolutionCost;  // Economic cost (money). Coverage is `unassignedCount`, never a cost.
    objective?: SolverObjective;  // Solver comparison scalar — see {@link SolverObjective}; never money.
    problemsCount?: number;  // Total constraint-problem markers across all routes (0 = feasible).
    routes?: PlanSolutionRoute[];  // One entry per route slot (empty slots included), in slot order.
    revision?: number;  // Session revision the routes reflect; bumps on every committed edit.
    undoDepth?: number;  // Committed edits that `routing24_undo` can walk back.
    redoDepth?: number;  // Undone edits that `routing24_redo` can re-apply.
    userAssigned?: string[];  // Site ids currently marked user-assigned (manually placed).
    userUnassigned?: string[];  // Unassigned site ids that were removed by the user/agent (not the solver).
    alternativesNotChosen?: string[];  // Group-alternative sites not chosen (their sibling serves the group).
    editedAt?: number;  // Last manual-edit timestamp (ms since epoch); absent = never edited.
    diagnosticsStale?: boolean;  // True when `unassignedDiagnostics` predates the latest edits — pass `refresh_diagnostics: true` to recompute before reading it.
    unassignedDiagnostics?: UnassignedDiagnosticsReport;  // Why sites are unassigned, in prose (capped; see `truncated`/`omitted`).
    insertionQuotes?: InsertionQuote[];  // Ready-to-serve quotes for the unassigned orders (one per quotable site; none when nothing fits). Refreshed with the diagnostics — pass `refresh_diagnostics: true` when `diagnosticsStale`.
    driftFromOptimized?: OptimizedDrift;  // Drift vs the last full optimization (user's manual edits included) — when `severity` is `degraded`/`severe`, tell the user and offer a fix.
};
```
```ts
// One route slot: an ordered stop sequence, bounded by depot stops when the
// vehicle has a start/finish depot (open-ended ends start/finish at an order
// instead). Empty slots (a vehicle with no stops, e.g. just created by
// `create_route`) are included with `siteCount: 0` so `slot` stays a
// complete, editable address space.
type PlanSolutionRoute = {
    slot: number;  // 0-based route slot — the address every `routing24_edit` op (`to_route`, `route`, `target`, `sources`) and `routing24_optimize_route` use. Slots compact after `remove_routes`; re-read them from the returned state.
    index: number;  // 1-based route number, matching the on-screen order (slot + 1).
    vehicleId?: string;  // Id of the vehicle serving this route.
    siteCount: number;  // Count of site stops served (the depot start/end are excluded).
    distance: number;  // Total travel distance of the route, in `distanceUnit`.
    durationHours: number;  // Total route duration (travel + service + wait), hours.
    startTimeS?: number;  // When the route starts (depot departure, or arrival at the first order on an open-start route), seconds since midnight.
    endTimeS?: number;  // When the route ends (depot return, or service completion at the last order on an open-ended route), seconds since midnight.
    feasible?: boolean;  // True when every constraint on this route is satisfied.
    problems?: PlanProblem[];  // Route-level constraint problems (also pinned per-stop).
    cost?: RouteCost;  // Economic cost of this route (absent on pre-feature solutions).
    stops: PlanSolutionStop[];  // Stops in visit order; bounded by depot stops unless the vehicle's start/finish depot is empty (open-ended).
};
```
```ts
type PlanSolutionStop = {
    seq: number;  // 0-based position within the route (0 = the starting depot).
    type: "depot" | "site" | "break";  // The depot, a delivery/pickup site, or a scheduled driver break. A `break` stop is a solver-planned pause taken AT the previous stop's location: it has no `id`/`address`, `serviceDurationS` is the pause length, and its travel-leg fields are 0.
    id?: string;  // The site/depot id (the id you passed to optimize, or an auto-assigned one).
    address?: string;  // Matched street address, when known.
    lat?: number;
    lng?: number;
    arrivalTimeS: number;  // Arrival time, seconds since midnight (physical driver timeline).
    departureTimeS: number;  // End of service (arrival + wait + service), seconds since midnight.
    serviceDurationS: number;  // Time spent servicing this stop, seconds.
    waitDurationS?: number;  // Wait before the stop's time window opens, seconds (0/absent = none).
    legDistance: number;  // Distance of the travel leg arriving at this stop, in `distanceUnit`.
    legDurationS: number;  // Duration of the travel leg arriving at this stop, seconds.
    carriedLoad: number;  // Vehicle load carried on arrival.
    deliveredLoad: number;  // Load delivered at this stop.
    pickedUpLoad: number;  // Load picked up at this stop.
    problems?: PlanProblem[];  // Constraint problems materializing at this stop (absent = none).
    marginalCost?: number;  // Removal saving — how much cheaper (money) / shorter (seconds) this route gets without this stop, same vehicle kept. An estimate for the CURRENT stop order: savings are NOT additive across stops and change after any edit or re-optimize. Absent on depots and pre-feature plans.
    marginalDurationS?: number;
};
```
```ts
// ECONOMIC cost of the solution, in the task's money units (the unit costs
// configured on the vehicles). Report money to the user from here. It does
// NOT include unassigned-order penalties — coverage is a count
// (`unassignedCount`), not a cost. Absent on pre-feature solutions.
type SolutionCost = {
    total: number;  // Full economic cost: fixed + distance + duration + stop (incl. overtime).
    overtime: number;  // Overtime premium — a breakdown line already inside `total`.
    vehicle: number;  // `total` minus `overtime`.
    stop?: number;  // Per-stop cost addend inside `total` (absent when no vehicle prices stops).
};
```
```ts
// One route's ECONOMIC cost, in the task's money units. Same shape rules as
// {@link SolutionCost}.
type RouteCost = {
    total: number;  // Full economic cost of the route (fixed + distance + duration + stop incl. overtime).
    overtime: number;  // Overtime premium inside `total`.
    vehicle: number;  // `total` minus `overtime`.
    stop?: number;  // Per-stop cost addend inside `total` (absent when the vehicle has no stop cost).
};
```
```ts
// The scalar the solver minimizes — SOLVER UNITS, NOT MONEY, never quote it
// to the user. Each unassigned order carries a synthetic penalty priced
// above any possible serving cost, so fewer-unassigned always outranks
// cheaper-routes. Use it ONLY to compare two states of the same plan:
// lower `objective.total` = better plan. Equivalent hand rule: fewer
// unassigned wins; ties break on lower `cost.total`.
type SolverObjective = {
    total: number;  // cost.total + unassignedPenalty. Comparisons only.
    unassignedPenalty: number;  // Σ synthetic penalties of unassigned orders (incl. alternatives not chosen).
};
```
```ts
// One constraint problem (common.fbs `Problem`).
type PlanProblem = {
    type: "capacity" | "max_distance" | "vehicle_incompatible" | "unreachable" | "time_window" | "max_duration" | "depot_time_window" | "shift_window" | "precedence" | "driving_allowance" | "break_schedule" | "break_location" | "sequence";
    amount?: number;  // How far over the constraint (see {@link PlanProblemType} for units).
    dimension?: number;  // Capacity dimension index (`capacity` problems only).
};
```
```ts
// Prose explanation of why sites are unassigned (common.fbs
// `UnassignedDiagnosticsLlm` — the agent-facing projection, capped in size).
type UnassignedDiagnosticsReport = {
    summary: string;  // One-paragraph overview of the unassigned situation.
    truncated?: boolean;  // True when the per-site list was capped (see `omitted`).
    sites: { site: string; category: string; explanation: string; blockers?: string[]; levers?: string[] }[];
    omitted?: { sitesCount: number; dominantKind?: string; dominantConstraint?: string };  // Rollup of the sites the cap left out.
};
```
```ts
// The cheapest FEASIBLE way to serve one unassigned order in the current
// routes — REAL units (money / seconds), safe to quote to the user. Directly
// replayable as a `move_sites` edit: use `before`/`after` as the anchor (or
// `to_route: routeSlot` with `placement: "best"` when both are absent).
// `routeSlot` absent = a spare `vehicleId` vehicle serves it on a fresh
// single-visit route, its fixed cost included (an existing route's fixed
// cost is sunk and excluded). Estimates for the CURRENT arrangement only:
// quotes are NOT additive across orders and go stale with any edit or
// re-optimize (`diagnosticsStale`).
type InsertionQuote = {
    site: string;
    routeSlot?: number;
    vehicleId?: string;
    cost: number;  // Route-cost increase, money.
    durationS: number;  // Route-duration increase, seconds.
    before?: string;
    after?: string;
    deliveryAfter?: string;  // Pair quote (site = the pickup): where the DELIVERY step is quoted, after the pickup (precedence-aware). Absent for a single order.
};
```
```ts
// How far the CURRENT plan has drifted from the last full optimization —
// covering every change since that solve, the user's manual edits included.
// Recompute-free: the baseline is frozen at solve completion; only a new
// `routing24_optimize` (or the user's OPTIMIZE button) resets it. Absent when
// the plan predates the baseline feature or was never solved.
// 
// REPORTING OBLIGATION: when `severity` is `degraded` or `severe`, tell the
// user (quote `summary`) and offer to fix it — `routing24_optimize_route` for
// a single rough route, or a fresh solve.
type OptimizedDrift = {
    optimizedAt: number;  // When the baseline solve completed (ms since epoch).
    distancePct: number;  // Travel-distance change vs the baseline, percent (+ = longer).
    durationPct: number;  // Total-duration change vs the baseline, percent (+ = slower).
    distance: number;  // Distance change in `distanceUnit` (+ = longer).
    durationHours: number;  // Duration change in hours (+ = slower).
    cost?: number;  // ECONOMIC cost change, money (+ = more expensive). Coverage drift is `unassigned` (a count) — the two are never merged into one number. Absent when the baseline carried no cost.
    costPct?: number;  // Cost change vs the baseline, percent (+ = more expensive).
    problems: number;  // New problem markers vs the baseline (+ = worse).
    unassigned: number;  // Newly unassigned stops vs the baseline (+ = worse).
    severity: "improved" | "neutral" | "degraded" | "severe";  // Fixed-threshold verdict: any new problem/unassigned ⇒ at least `degraded`; cost, distance or duration > +5% ⇒ `degraded`, > +15% ⇒ `severe`; everything ≤ baseline with a ≥1% gain ⇒ `improved`; else `neutral`.
    summary: string;  // One ready-to-quote sentence, lexicographic: coverage first, then money.
};
```
- `insertionQuotes` are ready-to-serve estimates in REAL units (safe to
  quote as money/minutes): the cheapest feasible placement per unassigned
  order, replayable directly as `move_sites` (anchor on `before`/`after`,
  or `to_route: routeSlot` + `placement:"best"`). For a pickup&delivery pair
  the quote keys on the pickup (`site`); `deliveryAfter` anchors the delivery
  step (always after the pickup, precedence-aware). Per-stop
  `marginalCost`/`marginalDurationS` are the reverse: what serving that
  stop adds to its route. Both are estimates for the CURRENT arrangement —
  never sum them, and refresh via `refresh_diagnostics` when
  `diagnosticsStale`.
- Two registers, never mixed: `cost` is MONEY (the task's unit costs;
  `cost.overtime` is a line inside `cost.total`, not an addend) and
  coverage is a COUNT (`unassignedCount`/`unassigned`). `objective` is the
  solver's comparison scalar in synthetic units — lower = better plan, and
  it already prices every unassigned order above any possible serving cost,
  so dropping stops NEVER improves it. Never quote `objective` as money.
  Hand rule: fewer unassigned wins; ties break on lower `cost.total`.
- `driftFromOptimized` compares the CURRENT plan to the last full solve and
  covers every change since it — the user's manual edits included. On
  `severity: "degraded" | "severe"` you MUST tell the user (quote `summary`)
  and offer `routing24_optimize_route` or a fresh solve.
- `type:"break"` stops are solver-planned driver breaks, taken at the
  PREVIOUS stop's location: no `id`/`address`, `serviceDurationS` = pause
  length, travel-leg fields 0. They don't count toward `siteCount`, can't be
  addressed by `routing24_edit` (no id), and re-place themselves automatically
  after any edit.

### `routing24_edit` — `EditInput` → `EditResult`
Manually edit the loaded solution: an ordered batch of edit ops applied
**all-or-nothing** (the first rejection rolls the whole batch back; later ops
see earlier ops' effects, e.g. `create_route` then `move_sites` onto the new
slot). Stops are addressed by their `id`, routes by `slot` from
`routing24_solution`. **Constraint problems never reject an edit** — they
are scored and reported (`state`, `userAssignedReports`) exactly like the
UI's manual drag & drop; rejections are structural only (`rejection.code`:
unknown ids, bad anchors, non-empty route removal, `stale_revision` when the
plan changed under you → re-read `routing24_solution` and rebuild). Applying
an edit locks the whole app behind a full-screen "Agent controlled" overlay
until the user clicks "Take control". Undoable — every batch is one entry in
the shared undo history.
```ts
// Input for `routing24_edit` (edits.fbs `EditBatch`): an ordered batch applied
// ALL-OR-NOTHING — the first rejected op rolls the whole batch back. Ops see the
// effects of earlier ops in the same batch (e.g. `create_route` then
// `move_sites` onto the new slot). Revision guarding is handled internally.
type EditInput = {
    ops: (EditMoveSites | EditSetRouteVehicle | EditUnassignSites | EditMarkUserAssigned | EditClearUserAssigned | EditCreateRoute | EditRemoveRoutes | EditSplitRoute | EditMergeRoutes)[];  // min 1
};
```
```ts
// Move `sites` as ONE ordered block (edits.fbs `MoveSites`). Destination is
// exactly one of: `before`/`after` (anchor site id, must be planned and not in
// `sites`), or `to_route` (+ `placement`). Also the way to plan an unassigned
// site: move it from nowhere onto a route.
type EditMoveSites = {
    op: "move_sites";
    sites: string[];  // min 1
    before?: string;  // Anchor site id: insert the block immediately before it.
    after?: string;  // Anchor site id: insert the block immediately after it.
    to_route?: integer;  // Destination route slot (see `PlanSolutionRoute.slot`). — >= 0
    placement?: "append" | "best";  // With `to_route`: `append` to the end, or `best` = cheapest insertion.
    user_assigned?: boolean;  // Mark the sites user-assigned (badge + problem report). Default true.
};
```
```ts
// Rebind a route slot to another fleet vehicle (edits.fbs `SetRouteVehicle`).
type EditSetRouteVehicle = {
    op: "set_route_vehicle";
    route: integer;  // >= 0
    vehicle: string;  // Vehicle id; rejects `vehicle_overused` when its count is exhausted.
};
```
```ts
// Remove sites from their routes into the unassigned list (edits.fbs `UnassignSites`).
type EditUnassignSites = {
    op: "unassign_sites";
    sites: string[];  // min 1
    user_unassigned?: boolean;  // Stamp the "removed by you" marker (default true). Pass false to unassign without claiming the removal, keeping the site's original unassigned reason.
};
```
```ts
// Mark already-planned sites user-assigned (badge only; edits.fbs `MarkUserAssigned`).
type EditMarkUserAssigned = {
    op: "mark_user_assigned";
    sites: string[];  // min 1
};
```
```ts
// Drop the user-assigned mark + report ("Dismiss"; edits.fbs `ClearUserAssigned`).
type EditClearUserAssigned = {
    op: "clear_user_assigned";
    sites: string[];  // min 1
};
```
```ts
// Append an empty route slot for `vehicle` (edits.fbs `CreateRoute`).
type EditCreateRoute = {
    op: "create_route";
    vehicle: string;
};
```
```ts
// Delete EMPTY route slots (edits.fbs `RemoveRoutes`; `route_not_empty`
// otherwise — `unassign_sites` or move the stops first). Remaining slots compact:
// put this op last in the batch and re-read slots from the result.
type EditRemoveRoutes = {
    op: "remove_routes";
    routes: integer[];  // min 1
};
```
```ts
// Split a route in two after site `after` (edits.fbs `SplitRoute`). The head
// keeps the slot + vehicle; the tail becomes a new appended slot served by
// `vehicle` (defaults to the same vehicle type).
type EditSplitRoute = {
    op: "split_route";
    route: integer;  // >= 0
    after: string;  // Site id that becomes the last stop of the head route.
    vehicle?: string;
};
```
```ts
// Append `sources` routes' stops onto `target`, leaving the sources empty (edits.fbs `MergeRoutes`).
type EditMergeRoutes = {
    op: "merge_routes";
    target: integer;  // >= 0
    sources: integer[];  // min 1
};
```
```ts
// Result of `routing24_edit`.
type EditResult = {
    applied: boolean;  // True when the whole batch committed; false = nothing was applied.
    errorCode?: "solve_in_progress" | "no_solution" | "slot_out_of_range" | "route_too_small" | "optimize_running" | "nothing_to_undo" | "nothing_to_redo" | "session_error";
    error?: string;  // Human-readable failure detail accompanying `errorCode`.
    rejection?: EditOpStatus;  // Why the batch rolled back (all-or-nothing), when the engine rejected it.
    state?: SessionState;  // Current state — post-apply, or unchanged when the batch was rejected.
    userAssignedReports?: UserAssignedProblemReport[];  // Problem reports for all currently user-assigned sites (when `applied`).
};
```
```ts
// The rejected op of a failed batch (edits.fbs `EditStatus`).
type EditOpStatus = {
    edit: number;  // Index into the submitted `ops` array.
    code: EditRejectionCode;
    reason?: string;  // Human-readable rejection detail, when the engine supplies one.
};
```
```ts
// Why an edit batch was rejected (edits.fbs `EditRejection`, minus `none`).
// Notable: `stale_revision` = the plan changed under you — re-read
// `routing24_solution` and rebuild the batch; `optimize_in_progress` = wait for
// the running optimization; slot codes mean your `slot` numbers are outdated.
type EditRejectionCode = "slot_out_of_range" | "nothing_to_undo" | "nothing_to_redo" | "unknown_site" | "unknown_vehicle" | "bad_anchor" | "anchor_not_assigned" | "anchor_in_moved_sites" | "ambiguous_destination" | "missing_destination" | "destination_mismatch" | "route_not_empty" | "vehicle_overused" | "group_conflict" | "stale_revision" | "too_many_trips" | "overlapping_slots" | "empty_split" | "site_not_assigned" | "optimize_in_progress" | "not_sole_op";
```
```ts
// Post-mutation session snapshot shared by the editing tools' results.
type SessionState = {
    revision?: number;  // Session revision after the operation.
    undoDepth?: number;
    redoDepth?: number;
    feasible?: boolean;
    problemsCount?: number;  // Total constraint-problem markers across all routes (0 = feasible).
    distance?: number;
    distanceUnit?: "km" | "mi";
    durationHours?: number;
    unassignedCount?: number;
    unassigned?: string[];  // Ids of the sites currently unassigned.
    cost?: SolutionCost;  // Economic cost (money). Coverage is `unassignedCount`, never a cost.
    objective?: SolverObjective;  // Solver comparison scalar — see {@link SolverObjective}; never money.
    userAssigned?: string[];
    userUnassigned?: string[];
    routes?: RouteSummary[];  // Every route slot, fresh — supersedes slots from before the mutation.
    driftFromOptimized?: OptimizedDrift;  // Drift vs the last full optimization — report `degraded`/`severe`.
};
```
```ts
// One route slot's summary in a {@link SessionState} (post-edit snapshot).
type RouteSummary = {
    slot: number;  // 0-based route slot (fresh — valid for the next edit batch).
    vehicleId?: string;
    stops: string[];  // Ordered site ids (depots excluded); empty = empty slot.
    distance?: number;  // Total travel distance, in `SessionState.distanceUnit`.
    durationHours?: number;  // Total duration (travel + service + wait), hours.
    feasible?: boolean;
    problems?: PlanProblem[];
    cost?: RouteCost;  // Economic cost of this route (absent on pre-feature solutions).
};
```
```ts
// The problem report behind a user-assigned site's badge (edits.fbs
// `UserAssignedReport`): what placing it costs, split into problems the site
// would have anywhere (`intrinsic`) vs ones this placement introduced.
type UserAssignedProblemReport = {
    site: string;
    route?: number;  // Route slot the site sits on.
    intrinsic: PlanProblem[];
    introduced: PlanProblem[];
};
```
- Idioms: plan an unassigned stop with `move_sites` + `to_route` +
  `placement:"best"`; empty-then-remove a route with
  `[unassign_sites, remove_routes]` in one batch; swap two group alternatives
  with `unassign_sites` (`user_unassigned:false`) + `move_sites`.
- `remove_routes` compacts the remaining slots — take fresh slots from
  `state.routes`, never reuse pre-batch numbers.

### `routing24_optimize_route` — `OptimizeRouteInput` → `OptimizeRouteResult`
Re-sequence ONE route's stops (single-route re-optimization). Synchronous and
fast (sub-second to a few seconds) — no polling; other routes are untouched.
Use it to tidy a route after manual moves. Undoable like any edit.
```ts
// Input for `routing24_optimize_route`: re-sequence ONE route's stops.
type OptimizeRouteInput = {
    route: integer;  // Route slot to optimize (see `PlanSolutionRoute.slot`); needs ≥2 stops. — >= 0
    time_limit_s?: integer;  // Wall budget in seconds; omit to auto-scale with route size (1.5–5s). — >= 1
};
```
```ts
// Result of `routing24_optimize_route`.
type OptimizeRouteResult = {
    started: boolean;  // False when the run could not start (see `errorCode`/`error`).
    errorCode?: "solve_in_progress" | "no_solution" | "slot_out_of_range" | "route_too_small" | "optimize_running" | "nothing_to_undo" | "nothing_to_redo" | "session_error";
    error?: string;
    state?: SessionState;  // Post-run state (present when `started`).
};
```

### `routing24_undo` / `routing24_redo` → `HistoryResult`
No input. Walk the edit history one committed batch at a time.
The edit history is SHARED with the user's own manual edits — an undo can revert something the user just did by hand, so never call it speculatively; prefer a compensating routing24_edit batch.
Returns {applied:false, errorCode:"nothing_to_undo" | "nothing_to_redo"} on an empty history (error carries the human-readable detail).
```ts
// Result of `routing24_undo` / `routing24_redo`.
type HistoryResult = {
    applied: boolean;  // False when nothing was undone/redone (see `errorCode`).
    errorCode?: "solve_in_progress" | "no_solution" | "slot_out_of_range" | "route_too_small" | "optimize_running" | "nothing_to_undo" | "nothing_to_redo" | "session_error";
    error?: string;
    state?: SessionState;
};
```

### `routing24_render` → `{ ok: true }`
No input. Navigates to the plan's optimize page so the routes draw on the map
(screenshot the tab afterwards to show the user).

### `routing24_save` → `{ saved: boolean; planUrl: string }`
No input. Persists the plan. When **anonymous**, the plan is stored in this
browser on this computer only — the link opens just here and may be deleted
later, so it is not a durable share link. Tell the user this when you hand over
the link. (If the user is signed in, the plan persists to their account and the
link opens on their other devices too.)

### `routing24_plan_url` → `{ planUrl: string }`
No input. Absolute URL of the current plan's optimize page.

### `routing24_cancel` → `{ ok: true }`
No input. Aborts an in-flight solve (mirrors the UI's cancel button).

## Machine-readable JSON Schema

The full JSON Schema is in [schema.json](schema.json) (same directory).
