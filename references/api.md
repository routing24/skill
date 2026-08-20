# Routing24 route optimizer — API reference

> Generated from Routing24's own types (skill version 6.1.0). The
> always-current copy is served at https://routing24.com/llms.txt.

WebMCP tools registered on `document.modelContext` on every
`https://routing24.com/app/*` page (`navigator.modelContext` is a deprecated
alias). Discover them with `getTools()`; invoke with
`executeTool(tool, JSON.stringify(args))` — it resolves to a **JSON string of the
result object** (parse it) and **rejects** on validation/handler errors.
`routing24_reoptimize_plan` is **fire-and-poll**: it returns immediately after starting
the background WASM solve; observe it with `routing24_status`. Shapes reuse the
solver's own `Site`/`VehicleType`/`Location` fields and are validated at
runtime — the JSON Schema is authoritative. Zero-argument tools take `'{}'`.

### `routing24_get_auth_user` → `{ user: string }`
No input. Returns the signed-in user's email, or `"anonymous"` when nobody is
logged in. **Informational only** — every tool works anonymously, so never gate
the flow on this or ask the user to sign in. It only tells you *where* a saved
plan will live (see `routing24_save`).

### `routing24_new_plan` — `NewPlanInput` → `NewPlanResult`
Commits a fresh EMPTY plan and makes it the loaded one — the first step when
planning from scratch. The plan it replaces is not lost: unsaved changes are
auto-saved first, and while the loaded plan holds data the app asks the user
to confirm the replacement (the call rejects when they decline). Follow with
the `routing24_upsert_*` tools to create the depot/vehicles/stops, then solve
with `routing24_reoptimize_plan`.
```ts
// Input for `routing24_new_plan`: commit a fresh EMPTY plan, replacing the
// loaded one (the app asks the user to confirm when the loaded plan holds
// data, and unsaved changes are auto-saved first). Follow with the
// `routing24_upsert_*` tools to create the depot/vehicles/stops, then solve
// with `routing24_reoptimize_plan`.
type NewPlanInput = {
    name?: string;  // Plan display name; omitted = "Route plan <date> <time>".
};
```
```ts
// Result of `routing24_new_plan`: the fresh empty plan is now the loaded one.
type NewPlanResult = {
    created: true;
    planUuid: string;
};
```

### `routing24_status` → `OptimizeStatus`
No input. The optimization snapshot AND the solution overview for the
currently-loaded plan (whether you just optimized it or opened it via its URL):
the rollups, economic `cost`/`objective`, `driftFromOptimized`, and
`routeStats` — one COMPACT entry per route (vehicle, stop count, distance,
duration, feasibility, cost; **no** stop list). Poll by looping on routing24_status until phase is 'done' or 'error': while a solve runs each call holds its reply up to ~15s (returning early when the solve lands), so call it back-to-back — no sleep between calls is needed.
`phase` walks `idle → geocoding → matrix → solving → saving → done` (or `error`). Drill into a
single route's ordered stops with `routing24_route`, and read what's unserved
with `routing24_unassigned`.
```ts
type OptimizeStatus = {
    phase: "idle" | "geocoding" | "matrix" | "solving" | "saving" | "done" | "error";
    running: boolean;
    progress?: number;  // 0..1 while solving, when available.
    feasible?: boolean;
    routes?: number;
    stops?: number;
    unassignedCount?: number;
    unassignedSummary?: string;  // One-line WHY for a nonzero `unassignedCount` (from the last computed diagnostics), so every status poll keeps the reasons one hop away. The per-stop reasons, numeric targets and fixes come from `routing24_unassigned`.
    distance?: number;
    distanceUnit?: "km" | "mi";
    durationHours?: number;
    problemsCount?: number;  // Total constraint-problem markers across all routes (0 = feasible).
    driftFromOptimized?: OptimizedDrift;  // Drift vs the last full optimization (absent while a solve runs).
    error?: string;
    planUuid?: string;
    complete?: boolean;  // True when the solve ran to completion (not cancelled part-way).
    cost?: SolutionCost;  // Economic cost (money) of the whole plan. Coverage is `unassignedCount`, never a cost. Absent on pre-feature solutions.
    objective?: SolverObjective;  // Solver comparison scalar — see {@link SolverObjective}; never money.
    costModel?: CostModel;  // The per-unit pricing this solve used, fallbacks included — see {@link CostModel}. Absent mid-solve and on pre-feature solutions.
    routeStats?: RouteBrief[];  // One compact entry per route (empty routes included), in on-screen order: vehicle, stop count, distance, duration, feasibility and cost per route — NO stop list. Drill into one route's ordered stops with `routing24_route`.
    revision?: number;  // Session revision the routes reflect (present once an editing session is open); bumps on every committed edit.
    undoDepth?: number;  // Committed edits `routing24_undo` can walk back.
    redoDepth?: number;  // Undone edits `routing24_redo` can re-apply.
    dataSync?: DataSyncStatus;  // Present ONLY when the plan data (or the distance unit) changed after the last optimization — see {@link DataSyncStatus}.
};
```
```ts
// One route's compact stats in {@link OptimizeStatus}.`routeStats` — totals
// only, no stop list. Drill into the ordered stops with `routing24_route(route)`.
// The shared fields are {@link PlanSolutionRoute}'s, defined once there.
type RouteBrief = {
    cost?: RouteCost;  // Economic cost of this route (absent on pre-feature solutions).
    distance: number;  // Total travel distance of the route, in `distanceUnit`.
    route: number;  // 1-based route number, matching the on-screen order — the SAME number every route-addressed tool takes (`routing24_route`, `routing24_reoptimize_route`, the `routing24_edit_*` `to_route`/`route`/ `target`/`sources` fields). Numbers renumber after `routing24_edit_remove_routes`; re-read them from the returned state.
    vehicleId?: string;  // Id of the vehicle serving this route.
    stopCount: number;  // Count of site stops served (the depot start/end are excluded).
    durationHours: number;  // Total route duration (travel + service + wait), hours.
    feasible?: boolean;  // True when every constraint on this route is satisfied.
    problemsCount?: number;  // Constraint-problem markers on this route (0 = feasible); read the problems themselves from `routing24_route`.
};
```
```ts
// ECONOMIC cost of the solution, in the task's money units (the unit costs
// configured on the vehicles). Report money to the user from here. It does
// NOT include unassigned-order penalties — coverage is a count
// (`unassignedCount`), not a cost. Absent on pre-feature solutions.
type SolutionCost = {
    total: number;  // Full economic cost = Σ `components[].amount` (2-decimal rounding).
    components: CostComponent[];  // The solver's own decomposition of `total` — every charged line, zero lines omitted. Plan-level lines carry amounts and plan-wide quantities but NO rates (rates can differ per vehicle — read `OptimizeStatus.costModel`, or one route's cost from `routing24_route`, for rate-level verification).
};
```
```ts
// One line of a cost total. Self-describing: a future pricing element arrives
// as a new `kind`, never a new shape. `amount` is the solver's authoritative
// money value; when `rate` and `quantity` are present, `amount` ≈ `rate` ×
// `quantity` (2-decimal rounding; a force-placed stop on a restricted leg may
// legitimately price below that — `amount` wins).
type CostComponent = {
    kind: "fixed" | "distance" | "duration" | "overtime" | "stop" | "rideOvertime" | "loadDistance" | "other";  // `fixed` per-use vehicle cost; `distance` travel priced per mi/km; `duration` route time priced per hour, EXCLUDING the overtime premium — `overtime` (the surcharge for hours past the regular limit) is its own line, so lines always sum to `total`; `stop` per-visit fees; `rideOvertime` linked orders' ride time inside their overtime band; `loadDistance` carriage (load on board × leg length); `other` unattributed remainder (rare — solutions saved by older app versions).
    amount: number;  // Money — an addend of the parent `total`. Report this.
    rate?: number;  // Effective per-`unit` rate the solve used (on `routing24_route` costs only) — the authored rate at the engine's fixed-point step (1e-6 per wire unit), rounded to 2 decimals here, so it normally equals the vehicle's configured value. See `defaultRate` before presenting it as configured.
    quantity?: number;  // Billed quantity, in `unit`.
    unit?: "km" | "mi" | "stop" | "h" | "load-mi" | "load-km";  // Unit of `quantity` (and denominator of `rate`). Distance units are the SOLVE-TIME display unit; `load-mi`/`load-km` = load unit × mile/km.
    defaultRate?: true;  // The `rate` is an engine DEFAULT — the matching vehicle cost field is blank in the app, the engine supplied the value (see {@link VehicleEffectiveRates}.`defaultRates`). Call it a default; never present it as a cost the user configured.
};
```
```ts
// The pricing this solve ACTUALLY used — the vehicles' authored cost rates
// after the engine's fallbacks. `priced: false` means NO vehicle prices
// distance, duration or load-distance: the optimizer then minimizes distance
// and time together (every vehicle priced at the defaults of 1 per mile/km
// plus 1 per hour), so `cost.total` reads roughly as the plan's travel
// distance in miles/km plus its total route time in hours (driving + service
// + waiting) — quote `note` when explaining. Absent on pre-feature solutions.
type CostModel = {
    priced: boolean;
    note?: string;  // Ready-to-quote explanation, present when `priced` is false.
    distanceUnit: "km" | "mi";  // Solve-time display distance unit the rates are stated in.
    vehicles: VehicleEffectiveRates[];  // Effective per-unit rates per vehicle type.
};
```
```ts
// One vehicle type's effective rates, in `CostModel.distanceUnit` and hours:
// `fixed` per route driven, `distance` per mi/km, `duration` / `overtime` /
// `rideOvertime` per hour, `stop` per stop served, `loadDistance` per load
// unit per mi/km. An omitted rate = 0 (not charged). Rates are what the
// engine BILLS — the authored value at the engine's fixed-point step (1e-6
// per wire unit), rounded to 2 decimals here, so a rate normally equals the
// authored one; the authored values themselves are on the vehicle rows
// (`routing24_list_vehicles`).
type VehicleEffectiveRates = {
    vehicleId: string;
    fixed?: number;
    distance?: number;
    duration?: number;
    stop?: number;
    overtime?: number;
    rideOvertime?: number;
    loadDistance?: number;
    defaultRates?: ("distance" | "duration" | "overtime")[];  // Rates the ENGINE supplied — the vehicle's matching cost field is blank in the app: `distance` and `duration` when the fleet is unpriced (the defaults of 1 per mile/km and 1 per hour), `overtime` when overtime is allowed but unpriced (priced at the vehicle's hourly rate, else a nominal 1/hour). Call these defaults; never present one as a cost the user configured.
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
// How far the CURRENT plan has drifted from the last full optimization —
// covering every change since that solve, the user's manual edits included.
// Recompute-free: the baseline is frozen at solve completion; only a new
// `routing24_reoptimize_plan` (or the user's OPTIMIZE button) resets it. Absent when
// the plan predates the baseline feature or was never solved.
// 
// REPORTING OBLIGATION: when `severity` is `degraded` or `severe`, tell the
// user (quote `summary`) and stop there. An edit the user asked for is
// expected to cost more — never re-optimize (or propose it) to walk one back;
// re-optimizing is only ever the user's own request.
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
```ts
// What changed in the plan data AFTER the last optimization — the same
// condition that shows the user the NOT SYNC badge. The current solution and
// the route-editing tools still see the plan as of that solve; the changes
// counted here take effect on the next optimization (run it only when the
// user asks). A kind that is absent is unchanged.
type DataSyncStatus = {
    stops?: EntityKindDiff;
    vehicles?: EntityKindDiff;
    depots?: EntityKindDiff;
    addresses?: EntityKindDiff;
    distanceUnitChanged?: boolean;  // The display distance unit changed since the solve (miles <-> km), which alone re-prices the solution's distances.
};
```
```ts
// Counts of one entity kind's rows that differ from the last optimization's
// snapshot of the plan.
type EntityKindDiff = {
    added: integer;  // >= 0
    removed: integer;  // >= 0
    changed: integer;  // >= 0
};
```
- Two registers, never mixed: `cost` is MONEY (the task's unit costs) and
  coverage is a COUNT (`unassignedCount`; the ids/reasons are in
  `routing24_unassigned`). `objective` is the solver's comparison scalar in
  synthetic units — lower = better plan, and it already prices every unassigned
  order above any possible serving cost, so dropping stops NEVER improves it.
  Never quote `objective` as money. Hand rule: fewer unassigned wins; ties
  break on lower `cost.total`.
- To explain a cost, walk `cost.components`: the lines sum to `cost.total`
  and each names its `kind` (a route's cost from `routing24_route` adds
  `rate`/`quantity`/`unit` per line for arithmetic the user can verify).
  Cross-check with `rate × quantity`, but the solver's `amount` is the
  authoritative value — report it even when the product differs (a force-placed
  stop on a restricted leg can legitimately price below the product).
- `costModel` is the pricing the solve ACTUALLY used: each rate is the
  authored value at the engine's fixed-point step (1e-6 per wire unit — per
  yard/metre, per second), reported to 2 decimals, so it normally equals the
  authored rate; quote these effective rates when explaining amounts (the
  authored values are on the vehicle rows). When `priced` is false, no cost
  is configured anywhere and the defaults of 1 per mile/km plus 1 per hour
  apply, so `cost.total` reads roughly as travel distance in miles/km plus
  total route time in hours (driving + service + waiting) — quote `note`
  instead of presenting it as money. A rate listed in a vehicle's
  `defaultRates` (and a component's `defaultRate`) is an ENGINE DEFAULT:
  that cost field is blank in the app, so say "default rate", never "you set
  this cost".
- `driftFromOptimized` compares the CURRENT plan to the last full solve and
  covers every change since it — the user's manual edits included. On
  `severity: "degraded" | "severe"` you MUST tell the user (quote `summary`)
  and stop there — never re-optimize (or propose it) to walk back an edit the
  user asked for; re-optimizing is only ever the user's own request.
- `routeStats[].route` (1-based, the on-screen route number) is the ONE
  route address every tool takes — `routing24_route`,
  `routing24_reoptimize_route` and the editing tools alike.

### `routing24_route` — `RouteInput` → `PlanSolutionRoute`
One route's full detail by its 1-based `route` number (from
`routing24_status.routeStats` — the same number the user sees): the ordered
depot→stops→depot sequence, each stop resolved to its `id`/`address`,
arrival & departure times (seconds since midnight, physical timeline),
per-leg distance/duration, loads, waits and problems. Works right after
optimizing and on a plan reopened by URL. **Rejects** (no result envelope) when
there is no solution or the route number is out of range — read
`routing24_status` first.
```ts
// Input for `routing24_route`: one route's full detail by its 1-based `route`
// number.
type RouteInput = {
    route: integer;  // 1-based route number from `OptimizeStatus.routeStats[].route` — the same number shown to the user (Route 1 = `route: 1`). — >= 1
};
```
```ts
// One route: an ordered stop sequence, bounded by depot stops when the
// vehicle has a start/finish depot (open-ended ends start/finish at an order
// instead). Empty routes (a vehicle with no stops, e.g. just created by
// `routing24_edit_create_route`) are included with `stopCount: 0` so `route` stays a
// complete, editable address space.
type PlanSolutionRoute = {
    route: number;  // 1-based route number, matching the on-screen order — the SAME number every route-addressed tool takes (`routing24_route`, `routing24_reoptimize_route`, the `routing24_edit_*` `to_route`/`route`/ `target`/`sources` fields). Numbers renumber after `routing24_edit_remove_routes`; re-read them from the returned state.
    vehicleId?: string;  // Id of the vehicle serving this route.
    stopCount: number;  // Count of site stops served (the depot start/end are excluded).
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
// One route's ECONOMIC cost, in the task's money units. `total` =
// Σ `components[].amount`. In `routeStats` and session route summaries the
// lines carry amounts only; `routing24_route` adds the serving vehicle's
// effective `rate`/`quantity`/`unit` per line.
type RouteCost = {
    total: number;
    components: CostComponent[];
};
```
```ts
// One constraint problem (common.fbs `Problem`).
type PlanProblem = {
    type: "capacity" | "max_distance" | "vehicle_incompatible" | "unreachable" | "time_window" | "max_duration" | "depot_time_window" | "shift_window" | "precedence" | "driving_allowance" | "break_schedule" | "break_location" | "sequence" | "incompatible_load_class" | "shelf_life" | "field_not_applicable";
    amount?: number;  // How far over the constraint (see {@link PlanProblemType} for units).
    dimension?: number;  // Capacity dimension index (`capacity` problems only).
    classes?: string[];  // Conflicting load-class names (`incompatible_load_class` only).
};
```
- `cost.components` here carry the serving vehicle's effective `rate`,
  billed `quantity` and `unit` (solve-time units) — the full verifiable
  breakdown of this route's cost. `defaultRate: true` marks an engine-default
  rate (the vehicle's cost field is blank in the app; see
  `OptimizeStatus.costModel`).
- Per-stop `marginalCost`/`marginalDurationS` are the removal saving: what
  serving that stop adds to its route, in REAL money/seconds. An estimate for
  the CURRENT arrangement — never sum across stops, and re-read after any edit.
- `PlanProblem` codes worth decoding: `break_schedule` = the driver-break
  rules could not be met (typically a single leg longer than the driving
  trigger); `break_location` = a break had to be placed at a `no_break`
  stop; `shelf_life` = a ride ran past `max_time_in_vehicle_s` + band on an
  evaluated or edited route (a solve drops such pairs to `unassigned`
  instead); `field_not_applicable` = an intrinsic mismatch (e.g.
  `max_ride_overtime_s` on a plain stop), reads as an unassigned reason.
- `type:"break"` stops are solver-planned driver breaks, taken at the
  PREVIOUS stop's location: no `id`/`address`, `serviceDurationS` = pause
  length, travel-leg fields 0. They don't count toward `stopCount`, can't be
  addressed by the `routing24_edit_*` tools (no id), and re-place automatically
  after any edit.

### `routing24_unassigned` — `UnassignedInput` → `UnassignedReport`
Which sites of the loaded plan are unserved and WHY: the unassigned ids, the
group `alternativesNotChosen` (a sibling serves the group — NOT failures),
user-removed ids, and — once an editing session is open — the prose
`unassignedDiagnostics` (summary + per-site category/explanation/blockers/
levers) and ready-to-serve `insertionQuotes`. Pass `{ refresh_diagnostics:
true }` to recompute the diagnostics when `diagnosticsStale` is true. Returns
`{ available: false }` when the plan has no solution yet.
```ts
// Input for `routing24_unassigned`.
type UnassignedInput = {
    refresh_diagnostics?: boolean;  // Recompute the unassigned-site diagnostics before returning them (runs an engine pass; only needed when `diagnosticsStale` was true).
};
```
```ts
// Result of `routing24_unassigned`: which sites of the loaded plan are unserved
// and why. `available` is `false` when the plan has no solution yet.
type UnassignedReport = {
    available: boolean;  // `false` when the loaded plan carries no optimized solution yet.
    unassignedCount?: number;  // Number of site stops that could not be served.
    unassigned?: string[];  // Ids of the unassigned sites, if any.
    alternativesNotChosen?: string[];  // Group-alternative sites not chosen (a sibling serves the group) — these are NOT failures.
    userUnassigned?: string[];  // Unassigned site ids that were removed by the user/agent (not the solver).
    diagnosticsStale?: boolean;  // True when `unassignedDiagnostics` predates the latest edits — pass `refresh_diagnostics: true` to recompute before reading it.
    unassignedDiagnostics?: UnassignedDiagnosticsReport;  // Why sites are unassigned, in prose (capped; see `truncated`/`omitted`).
    insertionQuotes?: InsertionQuote[];  // Ready-to-serve quotes for the unassigned orders (the cheapest feasible placement for each quotable site; none when nothing fits). Refreshed with the diagnostics — pass `refresh_diagnostics: true` when `diagnosticsStale`.
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
// replayable with `routing24_edit_move_stops`: use `before`/`after` as the anchor (or
// `to_route: route` with `placement: "best"` when both are absent).
// `route` absent = a spare `vehicleId` vehicle serves it on a fresh
// single-visit route, its fixed cost included (an existing route's fixed
// cost is sunk and excluded). Estimates for the CURRENT arrangement only:
// quotes are NOT additive across orders and go stale with any edit or
// re-optimize (`diagnosticsStale`).
type InsertionQuote = {
    stop: string;
    route?: number;  // 1-based route number of the quoted insertion point.
    vehicleId?: string;
    cost: number;  // Route-cost increase, money.
    durationS: number;  // Route-duration increase, seconds.
    before?: string;
    after?: string;
    deliveryAfter?: string;  // Pair quote (stop = the pickup): where the DELIVERY step is quoted, after the pickup (precedence-aware). Absent for a single order.
};
```
- `insertionQuotes` are ready-to-serve estimates in REAL units (safe to
  quote as money/minutes): the cheapest feasible placement per unassigned
  order, replayable directly with `routing24_edit_move_stops` (anchor on
  `before`/`after`,
  or `to_route: route` + `placement:"best"`). For a pickup&delivery pair
  the quote keys on the pickup (`stop`); `deliveryAfter` anchors the delivery
  step (always after the pickup, precedence-aware). They hold for the CURRENT
  arrangement only — never sum them, and refresh via `refresh_diagnostics`
  when `diagnosticsStale`.

### `routing24_edit_move_stops` / `routing24_edit_unassign_stops` / `routing24_edit_set_route_vehicle` / `routing24_edit_create_route` / `routing24_edit_remove_routes` / `routing24_edit_split_route` / `routing24_edit_merge_routes` / `routing24_edit_mark_user_assigned` / `routing24_edit_clear_user_assigned` → `EditResult`
Manual editing of the loaded solution — **one tool per action**, each taking
one flat input (no op unions to compose). Stops are addressed by their `id`,
routes by their 1-based `route` number from `routing24_status.routeStats`.

| tool | input | does |
| --- | --- | --- |
| `routing24_edit_move_stops` | `EditMoveStopsInput` | move/re-plan stops, anchored or onto a route |
| `routing24_edit_unassign_stops` | `EditUnassignStopsInput` | take stops off their routes |
| `routing24_edit_set_route_vehicle` | `EditSetRouteVehicleInput` | rebind a route to another vehicle |
| `routing24_edit_create_route` | `EditCreateRouteInput` | append an empty route |
| `routing24_edit_remove_routes` | `EditRemoveRoutesInput` | delete routes (unassigns their stops first) |
| `routing24_edit_split_route` | `EditSplitRouteInput` | split one route in two after a stop |
| `routing24_edit_merge_routes` | `EditMergeRoutesInput` | move sources' stops onto a target |
| `routing24_edit_mark_user_assigned` | `EditMarkStopsInput` | badge + problem report on placed stops |
| `routing24_edit_clear_user_assigned` | `EditMarkStopsInput` | drop that badge ("Dismiss") |

Every call applies **atomically** and lands as ONE entry in the shared undo
history. **Constraint problems never reject an edit** — they are scored and
reported (`state`, `userAssignedReports`) exactly like the UI's manual drag &
drop; rejections are structural only (`rejection.code`: unknown ids, bad
anchors, `stale_revision` when the plan changed under you → re-read
`routing24_status` and retry). Editing locks the whole app behind a
full-screen "Agent controlled" overlay until the user clicks "Take control".
```ts
// Input for `routing24_edit_move_stops`: move/re-plan planned OR unassigned
// stops. Anchor with `before`/`after` (a planned stop id on the destination
// route), or target a route with `to_route` + `placement`.
type EditMoveStopsInput = {
    stops: string[];  // Stop business ids to move, kept together as one block. — min 1
    before?: string;  // Anchor stop id: insert the block immediately before it.
    after?: string;  // Anchor stop id: insert the block immediately after it.
    to_route?: integer;  // Destination route number, 1-based (see `OptimizeStatus.routeStats[].route`). — >= 1
    placement?: "append" | "best";  // With `to_route`: `append` to the end, or `best` = cheapest insertion.
    user_assigned?: boolean;  // Mark the stops user-assigned (badge + problem report). Default true.
};
```
```ts
// Input for `routing24_edit_unassign_stops`.
type EditUnassignStopsInput = {
    stops: string[];  // Stop business ids to remove from their routes. — min 1
    user_unassigned?: boolean;  // Stamp the "removed by you" marker (default true). Pass false to unassign without claiming the removal, keeping the stop's original reason.
};
```
```ts
// Input for `routing24_edit_set_route_vehicle`.
type EditSetRouteVehicleInput = {
    route: integer;  // Route number, 1-based (see `OptimizeStatus.routeStats[].route`). — >= 1
    vehicle: string;  // Vehicle id; rejects `vehicle_overused` when its count is exhausted.
};
```
```ts
// Input for `routing24_edit_create_route`.
type EditCreateRouteInput = {
    vehicle: string;  // Vehicle id for the new empty route (appended last).
};
```
```ts
// Input for `routing24_edit_remove_routes`. The stops on those routes are
// unassigned first, in the SAME atomic batch — the engine's own remove op
// accepts empty routes only.
type EditRemoveRoutesInput = {
    routes: integer[];  // Route numbers (1-based) to delete. The remaining routes renumber — re-read them after. — min 1
};
```
```ts
// Input for `routing24_edit_split_route`.
type EditSplitRouteInput = {
    route: integer;  // Route number (1-based) to split. — >= 1
    after: string;  // Stop id that becomes the last stop of the head route.
    vehicle?: string;  // Vehicle for the new tail route (defaults to the same vehicle type).
};
```
```ts
// Input for `routing24_edit_merge_routes`.
type EditMergeRoutesInput = {
    target: integer;  // Route number (1-based) that receives the stops. — >= 1
    sources: integer[];  // Route numbers (1-based) whose stops move onto `target`, left empty afterwards. — min 1
};
```
```ts
// Input for `routing24_edit_mark_user_assigned` / `..._clear_user_assigned`.
type EditMarkStopsInput = {
    stops: string[];  // Planned stop business ids. — min 1
};
```
```ts
// Shared result of every `routing24_edit_*` tool.
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
// `routing24_status` and rebuild the batch; `optimize_in_progress` = wait for
// the running optimization; `slot_out_of_range`/`overlapping_slots` mean your
// route numbers are outdated.
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
    routes?: RouteSummary[];  // Every route, fresh — supersedes route numbers from before the mutation.
    driftFromOptimized?: OptimizedDrift;  // Drift vs the last full optimization — report `degraded`/`severe`.
};
```
```ts
// One route's summary in a {@link SessionState} (post-edit snapshot).
type RouteSummary = {
    route: number;  // 1-based route number (fresh — valid for the next edit call).
    vehicleId?: string;
    stops: string[];  // Ordered site ids (depots excluded); empty = empty route.
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
    stop: string;
    route?: number;  // Route number (1-based) the stop sits on.
    intrinsic: PlanProblem[];
    introduced: PlanProblem[];
};
```
- Idioms: plan an unassigned stop with `routing24_edit_move_stops` +
  `to_route` + `placement:"best"`; delete a route with
  `routing24_edit_remove_routes` (its stops are unassigned in the same atomic
  edit — no need to empty it first); swap two group alternatives with
  `routing24_edit_unassign_stops` (`user_unassigned:false`) then
  `routing24_edit_move_stops`.
- `routing24_edit_remove_routes` RENUMBERS the remaining routes — take fresh
  route numbers from `state.routes`, never reuse pre-call numbers.

### `routing24_reoptimize_route` — `ReoptimizeRouteInput` → `ReoptimizeRouteResult`
Re-sequence ONE route's stops (single-route re-optimization). Synchronous and
fast (sub-second to a few seconds) — no polling; other routes are untouched.
Use it to tidy a route after manual moves. Undoable like any edit.
```ts
// Input for `routing24_reoptimize_route`: re-sequence ONE route's stops.
type ReoptimizeRouteInput = {
    route: integer;  // Route number to optimize — 1-based, as in `OptimizeStatus.routeStats[].route` and the UI; needs ≥2 stops. — >= 1
    time_limit_s?: integer;  // Wall budget in seconds; omit to auto-scale with route size (1.5–5s). — >= 1
};
```
```ts
// Result of `routing24_reoptimize_route`.
type ReoptimizeRouteResult = {
    started: boolean;  // False when the run could not start (see `errorCode`/`error`).
    errorCode?: "solve_in_progress" | "no_solution" | "slot_out_of_range" | "route_too_small" | "optimize_running" | "nothing_to_undo" | "nothing_to_redo" | "session_error";
    error?: string;
    state?: SessionState;  // Post-run state (present when `started`).
};
```

### `routing24_undo` / `routing24_redo` → `HistoryResult`
No input. Walk the edit history one committed batch at a time.
One interleaved history covering route-arrangement edits AND plan-data changes (upserts, deletes, SQL updates, merge-imports). It is SHARED with the user's own manual edits — an undo can revert something the user just did by hand, so never call it speculatively; prefer a compensating routing24_edit_* call.
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

### `routing24_list_stops` / `routing24_list_vehicles` / `routing24_list_depots` / `routing24_list_addresses` — `ListPageInput` → `ListStopsResult` / `ListVehiclesResult` / `ListDepotsResult` / `ListAddressesResult`
Read the loaded plan's data — **one tool per entity kind**, each returning ONE
row type (paged, default 30/page, with an optional case-insensitive substring
filter over id and address). Rows come back in the exact shape the matching
`routing24_upsert_*` accepts, so read-modify-write needs no field mapping. The
business ids in the rows are the ids every other tool uses; use
`routing24_list_stops` with a `query` to look ONE stop up.
```ts
// Input for the four `routing24_list_*` tools — one paging/filter shape, and
// each tool returns ONE row type. (A single list tool taking a `kind` would
// switch its own result shape, so a caller could not know what it would get
// back without re-reading the input it had just sent.)
type ListPageInput = {
    query?: string;  // Case-insensitive substring over the id and address fields.
    offset?: integer;  // 0-based row offset into the filtered list. — >= 0
    limit?: integer;  // Rows per page (default 30). — >= 1
};
```
```ts
// Result of `routing24_list_stops`. Rows use the SAME shape
// `routing24_upsert_stops` accepts, so read-modify-write needs no field
// mapping — including `area`.
type ListStopsResult = {
    total: number;  // Rows matching the filter, before paging.
    offset: number;
    returned: number;
    stops: StopRow[];
};
```
```ts
// Result of `routing24_list_vehicles` (rows accepted by `routing24_upsert_vehicles`).
type ListVehiclesResult = {
    total: number;  // Rows matching the filter, before paging.
    offset: number;
    returned: number;
    vehicles: VehicleRow[];
};
```
```ts
// Result of `routing24_list_depots` (rows accepted by `routing24_upsert_depots`).
type ListDepotsResult = {
    total: number;  // Rows matching the filter, before paging.
    offset: number;
    returned: number;
    depots: DepotRow[];
};
```
```ts
// Result of `routing24_list_addresses` (rows accepted by `routing24_upsert_addresses`).
type ListAddressesResult = {
    total: number;  // Rows matching the filter, before paging.
    offset: number;
    returned: number;
    addresses: AddressRow[];
};
```
```ts
// A vehicle row: the upsert shape plus the resolved depot business ids.
type VehicleRow = {
    tw_early_s?: number;  // Shift starts, seconds since midnight. Absent = no earliest start.
    tw_late_s?: number;  // Shift ends, seconds since midnight. Absent = open-ended: no shift-end limit at all, the strongest relaxation of the shift.
    capacity?: number;
    available_count?: number;
    cost?: { fixed?: number; distance?: number; duration?: number; stop?: number; overtime?: number; ride_overtime?: number; load_distance?: number };  // PRO: Load-distance cost (cost per load-km) (cost.load_distance), Max time in vehicle (shelf life) (cost.ride_overtime), Overtime cost per hour (cost.overtime)
    start_late_s?: number;  // Latest time the vehicle may start its shift (leave the depot), distinct from tw_late_s (latest shift end). Absent = defaults to tw_late_s solver-side.
    tags?: string[];  // PRO: Skills (vehicle & order tags)
    max_reloads?: number;
    max_duration_s?: number;  // PRO: Max duration — Max route duration, seconds. Absent = uncapped.
    max_overtime_s?: number;  // PRO: Max overtime
    max_distance?: number;  // PRO: Max distance — Max route distance in the plan's display unit. Absent = uncapped.
    break_rules?: { max_driving_s?: number; duration_s?: number; split_first_s?: number; split_second_s?: number; service_counts?: boolean }[];  // PRO: Driver breaks & driving limits
    fixed_breaks?: { tw_early_s?: number; tw_late_s?: number; duration_s?: number }[];  // PRO: Driver breaks & driving limits
    period_driving_limit_s?: number;  // PRO: Driver breaks & driving limits
    period_driven_s?: number;  // PRO: Driver breaks & driving limits
    no_mix_load_classes?: { classes: string[] }[];  // PRO: Product segregation (load classes)
    id?: string;
    force_allow_sites?: string[];  // PRO: Force allow / deny orders — Stop ids this vehicle type may serve even when tags forbid it — a PERMISSION override, not a reservation: other compatible vehicles can still take the stop. Precedence per vehicle: force_deny_sites beats force_allow_sites beats the tag rule. To make a stop exclusive to one vehicle, use tags (required_tags on the stop + that tag on only this vehicle) or force_deny_sites on every other vehicle.
    force_deny_sites?: string[];  // PRO: Force allow / deny orders — Stop ids this vehicle type must never serve (beats force_allow_sites and tags).
    reload_depots?: string[];  // PRO: Reload depots — Depot ids where mid-route reloads may happen (multi-trip). Omitted = single-trip unless `max_reloads` is set (then the home depot is used); pair with `max_reloads` to cap trips.
    start_depot_id?: string;  // Business id of the start/finish depot (see `routing24_upsert_vehicles`).
    end_depot_id?: string;
};
```
```ts
// An address row: the upsert shape plus how many entities reference it.
type AddressRow = {
    address: string;
    area?: string;  // Optional area/city qualifier displayed after the address.
    status?: "geocoded" | "ungeocoded";  // Whether the row resolved to a map location. Reported on `routing24_list_addresses` rows; accepted and ignored on upsert.
    usedBySites?: number;
    usedByDepots?: number;
};
```

### `routing24_upsert_stops` / `routing24_upsert_vehicles` / `routing24_upsert_depots` / `routing24_upsert_addresses` — `Upsert*Input` → `UpsertResult` (addresses: `UpsertAddressesResult`)
Create or update entities of the LOADED plan by business `id` (addresses by
their `address`+`area` pair) — one tool per kind, in the same row shapes the
`routing24_list_*` tools return. An existing id updates that entity in place
(cross-references stay intact); a new id creates one. The `address` string is
the ONLY location carrier: new addresses are geocoded internally, rows are
reported with `status` (`geocoded`/`ungeocoded`), and no tool takes or
returns coordinates — a caller holding exact coordinates sends a decimal
`"lat, lng"` literal AS the address, which resolves to that exact point and
stays the row's label. A geocode failure does NOT
reject the batch: the row is saved without a location and
`addressDiagnostics` reports the problems (counts and next steps; a plan
cannot optimize while a stop or depot uses an unlocated address). These never
create a plan and never delete anything; the changes affect the NEXT solve
(`routing24_reoptimize_plan`), not the current solution. Fields marked
`PRO`/`STARTER` below need a paid plan — see *Plans & paid features*;
`paidFeatures` on the solve result says what this account actually got.
```ts
// Input for `routing24_upsert_stops`.
type UpsertStopsInput = {
    stops: ({ pickup?: number; delivery?: number; service_duration_s?: number; priority?: number; required_tags?: string[]; forbidden_tags?: string[]; group?: string; transfer_type?: "pickup" | "delivery" | "depot"; transfer_id?: string; no_break?: boolean; load_class?: string; sequence_group?: string; sequence_rank?: number; address?: string; area?: string; max_time_in_vehicle_s?: integer; max_ride_overtime_s?: integer; id?: string; status?: "geocoded" | "ungeocoded"; tw_early_s?: null | number; tw_late_s?: null | number; release_time_s?: null | number })[];  // min 1
};
```
```ts
// Input for `routing24_upsert_vehicles`. `start_depot_id`/`end_depot_id` take
// depot business ids and may be omitted when the plan has exactly one depot.
type UpsertVehiclesInput = {
    vehicles: VehiclePatch[];  // min 1
};
```
```ts
// Input for `routing24_upsert_depots`.
type UpsertDepotsInput = {
    depots: ({ service_duration_s?: number; no_break?: boolean; address?: string; area?: string; id?: string; status?: "geocoded" | "ungeocoded"; tw_early_s?: null | number; tw_late_s?: null | number })[];  // min 1
};
```
```ts
// Input for `routing24_upsert_addresses` (identity is the `address`+`area` pair).
type UpsertAddressesInput = {
    addresses: AddressUpsert[];  // min 1
};
```
```ts
// A delivery/pickup stop: a place plus solver site constraints.
// 
// The intersection above is flattened in the generated schema, which drops the
// doc comments of the types it merges — and the schema is all a tool caller
// sees, so the conventions those types state are repeated here:
// the location is the `address` string, geocoded internally; `id` is the
// business id the caller chooses and every other tool refers to (there is no
// name field); loads go in `delivery` (loaded at the depot, dropped here) and
// `pickup` (taken on here, carried back to the depot); every `*_s` time field
// is in seconds — `tw_early_s`/`tw_late_s`/`release_time_s` counted from
// midnight, `service_duration_s` a duration.
type StopRow = {
    address?: string;  // The WHOLE place the user named, spelling fixed — every part they wrote (building, tower, unit, street). Order words riding in the same line ("pickup 2", a quantity) are NOT part of the address and never become one: they belong in the load fields.
    area?: string;  // Optional area/city qualifier shown after the address (the address book's `address`+`area` identity). The `routing24_list_*` tools return it on every stop/depot row, so accepting it here is what makes that row writable back through `routing24_upsert_stops`/`_depots` unchanged.
    pickup?: number;  // Load picked UP at this stop and carried back to the depot, in the same unit as vehicle capacity ("pickup 2" on an order means `pickup: 2` — a load field, never part of the id or address).
    delivery?: number;  // Load taken from the depot and DELIVERED at this stop, in the same unit as vehicle capacity.
    service_duration_s?: number;
    tw_early_s?: number;  // Window opens, seconds since midnight. Absent = opens with the day.
    tw_late_s?: number;  // Window closes, seconds since midnight. Absent = open-ended (no closing bound on this stop).
    release_time_s?: number;
    priority?: number;
    required_tags?: string[];  // PRO: Skills (vehicle & order tags)
    forbidden_tags?: string[];  // PRO: Skills (vehicle & order tags)
    group?: string;  // PRO: Alternative order groups, Alternative pickup/delivery locations
    transfer_type?: "pickup" | "delivery" | "depot";  // PRO: Pickup & delivery (transfers)
    transfer_id?: string;  // PRO: Pickup & delivery (transfers)
    no_break?: boolean;
    load_class?: string;  // PRO: Product segregation (load classes)
    sequence_group?: string;  // PRO: Order sequences
    sequence_rank?: number;  // PRO: Order sequences
    max_time_in_vehicle_s?: integer;  // PRO: Max time in vehicle (shelf life) — >= 0
    max_ride_overtime_s?: integer;  // PRO: Max time in vehicle (shelf life) — >= 0
    id?: string;  // Caller-chosen business id — the name every other tool refers to. ONE row per order: never a dumping ground for leftover words (a load like "pickup 1" is the `pickup` field, not an id and not a second stop).
    status?: "geocoded" | "ungeocoded";  // Whether the row's address resolved to a map location. Reported on `routing24_list_stops` rows; accepted and ignored on upsert (so a listed row writes back unchanged).
};
```
```ts
// A depot row: a place plus depot constraints (open window, handling).
// Same conventions as {@link StopRow}: an `address` geocoded internally, a
// caller-chosen `id`, `tw_early_s`/`tw_late_s` in seconds since midnight and
// `service_duration_s` in seconds. `status` is reported on
// `routing24_list_depots` rows; accepted and ignored on upsert.
type DepotRow = {
    address?: string;  // The WHOLE place the user named, spelling fixed — every part they wrote (building, tower, unit, street). Order words riding in the same line ("pickup 2", a quantity) are NOT part of the address and never become one: they belong in the load fields.
    area?: string;  // Optional area/city qualifier shown after the address (the address book's `address`+`area` identity). The `routing24_list_*` tools return it on every stop/depot row, so accepting it here is what makes that row writable back through `routing24_upsert_stops`/`_depots` unchanged.
    service_duration_s?: number;  // Handling time at the depot, seconds. Absent = none.
    tw_early_s?: number;  // Opens at, seconds since midnight. Absent = open from any time.
    tw_late_s?: number;  // Closes at, seconds since midnight. Absent = open-ended: the depot never closes and the return-by-close constraint is fully lifted.
    no_break?: boolean;
    id?: string;
    status?: "geocoded" | "ungeocoded";
};
```
```ts
// One address-book row for `routing24_upsert_addresses` (and the row shape
// `routing24_list_addresses` returns). The row's identity
// is the `address` (+ `area`) string pair — upserting the same pair updates
// the existing row.
type AddressUpsert = {
    address: string;
    area?: string;  // Optional area/city qualifier displayed after the address.
    status?: "geocoded" | "ungeocoded";  // Whether the row resolved to a map location. Reported on `routing24_list_addresses` rows; accepted and ignored on upsert.
};
```
```ts
// Result of every `routing24_upsert_*` tool.
type UpsertResult = {
    applied: boolean;
    error?: string;
    added: number;
    updated: number;
    skipped: number;  // Rows dropped (e.g. malformed beyond repair).
    unresolvedRefs?: string[];  // Unresolved cross-references, as human-readable messages.
    geocoded?: number;  // How many entities were geocoded from their address on the way in (rows whose address could not be geocoded are still saved — see `addressDiagnostics`).
    addressDiagnostics?: AddressDiagnostics;  // Present only when the batch left address problems behind.
    fleetDiagnostics?: { summary: string; problems: { category: "vehicle_incompatible"; count: number; explanation: string; lever: string }[] };  // Present only when the edit left stops no vehicle can serve.
};
```
```ts
// Result of `routing24_upsert_addresses`: the shared upsert counts plus, for
// SMALL batches only (at most 5 rows — the confirm-with-the-user use), one
// result per input row. `matched` is the canonical address the geocoder
// resolved for rows geocoded on this call — state it when confirming a
// doubtful address with the user; it is transient and never stored. Rows that
// reused an already-located address book entry come back `status: "geocoded"`
// with no `matched`. Larger batches return counts and `addressDiagnostics`
// only.
type UpsertAddressesResult = {
    applied: boolean;
    error?: string;
    added: number;
    updated: number;
    skipped: number;  // Rows dropped (e.g. malformed beyond repair).
    unresolvedRefs?: string[];  // Unresolved cross-references, as human-readable messages.
    geocoded?: number;  // How many entities were geocoded from their address on the way in (rows whose address could not be geocoded are still saved — see `addressDiagnostics`).
    addressDiagnostics?: AddressDiagnostics;  // Present only when the batch left address problems behind.
    fleetDiagnostics?: { summary: string; problems: { category: "vehicle_incompatible"; count: number; explanation: string; lever: string }[] };  // Present only when the edit left stops no vehicle can serve.
    rows?: ({ address: string; area?: string; status: "geocoded" | "ungeocoded"; matched?: string })[];
};
```
```ts
// Address problems the upsert path detected, host-computed (the geocoder and
// the geo analysis are black boxes — this block is their only agent-visible
// output). Carries counts and next calls, never the rows themselves: the user
// sees the affected addresses in the app and in the `fix_addresses` review
// dialog, so never enumerate them in chat or in an ask_user question.
type AddressDiagnostics = {
    summary: string;  // One host-composed sentence with the counts.
    problems: ({ category: "ungeocoded" | "far_from_plan"; count: number; explanation: string; lever: string })[];
};
```
- `id` is required on every stop/depot/vehicle here (it is the upsert key).
- `routing24_upsert_addresses` with **at most 5 rows** returns `rows` — one
  per input with `status` and the canonical `matched` text the geocoder
  resolved (present only for rows geocoded by this call) — the way to confirm
  a doubtful address with the user before creating stops on it. Larger
  batches (hundreds of rows are fine) return counts and
  `addressDiagnostics` only.
- A row whose `address`+`area` pair the plan already resolved reuses that
  stored location — re-sending known addresses never re-geocodes or moves
  them; only NEW or changed address text is geocoded.
- A vehicle's `start_depot_id`/`end_depot_id` take depot business ids; omit
  them only when the plan has exactly one depot. Vehicles whose depot reference
  resolves to nothing are skipped and reported in `unresolvedRefs` — never
  silently mis-assigned.
- `area` is accepted on stops and depots (and returned by the list tools), so a
  row read out can be written straight back.
- Times are **seconds since midnight**. `delivery`/`pickup` are
  single-dimension loads. `available_count` = identical vehicles of that
  type.
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
- **Product segregation**: give stops a `load_class` and vehicles
  `no_mix_load_classes` — groups of class names; two classes of one group
  never ride together on that vehicle's route (reload trips included). Groups
  are independent; a class no stop carries is ignored; at most 64 distinct classes per task.
  Both ends of a transfer carry the same class.
- **Shelf life**: on a plain stop `max_time_in_vehicle_s` bounds the time
  from its `release_time_s` to the start of service — the clock is PINNED
  at `release_time_s`, there is no departure-relative form, so a stop
  without one is measured from the start of the planning horizon (0), which
  is stricter, not looser: **always send `release_time_s` with it**. A stop
  whose `tw_early_s` is later than `release_time_s` + the bound cannot be
  served at all and rejects the solve. A plain stop carrying a
  `pickup` load is dropped on its own instead (the bound covers
  `delivery` goods, on board from the depot; a pickup load boards at
  service and rides on unbounded): the solve runs and that stop comes back
  in `unassigned`, reported as `field_not_applicable`.
  On a linked (pickup & delivery)
  stop it instead bounds the ride time from pickup to delivery (waiting
  counts; a reload does not reset it) — set identically on every end of the
  transfer: ends that disagree do NOT reject the solve, they are dropped from
  the solve together, so the plan comes back without them and they appear in
  no route and in no `unassigned` list.
  `max_ride_overtime_s` (linked stops only, requires
  `max_time_in_vehicle_s`, priced at the vehicle's `cost.ride_overtime`)
  allows a priced band of ride time past the bound; beyond it the bound is
  hard. On a plain (depot) stop the field does not apply: that stop is left
  unassigned with a reason, rather than rejecting the call. A bounded
  LINKED stop combines with driver breaks: break placement avoids the
  pickup-to-delivery span where it can, and a break placed inside it
  counts as ride time, priced against the bound and its band. On a solve,
  a pair whose ride cannot fit bound + band around the mandated breaks is
  simply not served: both ends come back in `unassigned`. `shelf_life`
  problem rows appear only on evaluated or manually edited routes.
- Vehicle `force_allow_sites`/`force_deny_sites`/`reload_depots` reference the
  plan's stop/depot business ids.
  `max_distance` is in the plan's display unit (km/mi); `cost.duration` and
  `cost.overtime` are per hour (`max_overtime_s` caps paid overtime past
  `max_duration_s`).
- **Vehicle costs**: `cost.distance` is per km/mi (the plan's display unit),
  `cost.fixed` per vehicle used, `cost.load_distance` per unit of load
  carried per km/mi (every leg charges the load on board times the leg
  length — load-dependent fuel/refrigeration burn; it counts as pricing the
  fleet like `cost.distance`). An omitted cost field means 0. To minimize
  distance and time, **omit `cost` entirely — do not send zeros**: a fleet
  whose every distance/duration rate is 0 has nothing to optimize, so the
  zeros are ignored, the defaults of 1 per mile/km plus 1 per hour apply, and
  the result carries a `warnings` entry. An explicit 0 next to priced
  vehicles is honored (a bike with free mileage beside a van at 2/km steers
  mileage onto the bike).
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
  Stops/depot take `no_break: true` to forbid hosting a break there; if no
  allowed host exists the break is placed there anyway and the route
  reports a `break_location` problem. Breaks never interrupt a travel
  leg: a single leg longer than `max_driving_s` cannot be repaired and
  reports `break_schedule`.
  Planned breaks come back as `type:"break"` stops in `routing24_route`.
  Breaks combine with ride-bounded transfers: see **Shelf life** above.

### `routing24_delete_stops` / `routing24_delete_vehicles` / `routing24_delete_depots` / `routing24_delete_addresses` — `DeleteByIdsInput` → `DeleteResult`
Delete entities of the loaded plan by business id — one tool per kind. Deletes
CASCADE, and the result's `cascade` says what went with them: deleting a depot
removes the vehicles referencing it; deleting an address removes the stops and
depots at it (and then their dependents); deleting a stop drops its address when
nothing else uses it.
Undoable via routing24_undo (cascaded removals included) — still confirm with the user before deleting anything they did not explicitly list. Ids that match nothing come back in notFound.
```ts
// Input for the four `routing24_delete_*` tools: business ids as returned by
// the matching `routing24_list_*` (for addresses, the `address` — or
// `address, area` — string).
type DeleteByIdsInput = {
    ids: string[];  // min 1
};
```
```ts
// Result of every `routing24_delete_*` tool.
type DeleteResult = {
    deleted: number;
    notFound?: string[];  // Ids that matched nothing (nothing was deleted for them).
    cascade?: string;  // What was removed alongside (deletes cascade: a depot delete removes vehicles referencing it; an address delete removes the depots/stops at it; a stop delete drops its address when nothing else uses it).
    fleetDiagnostics?: { summary: string; problems: { category: "vehicle_incompatible"; count: number; explanation: string; lever: string }[] };  // Present only when the delete left stops no vehicle can serve.
};
```

### `routing24_reoptimize_plan` — `ReoptimizePlanInput` → `SolveStarted`
Optimize the CURRENTLY LOADED plan in place — the same solve the Optimize
button runs, on the plan data as it stands (after imports, upserts, deletes).
Returns at once (fire-and-poll; poll `routing24_status`) and navigates to the
Optimize page so progress is visible. Replaces the plan's previous solution;
the plan data itself is untouched. After `routing24_new_plan` + upserts, this
is how a from-scratch plan gets solved.
```ts
// Input for `routing24_reoptimize_plan`.
type ReoptimizePlanInput = {
    options?: { time_limit_s?: integer };
};
```
```ts
// Result of `routing24_reoptimize_plan`: the solve is running, poll for progress.
type SolveStarted = {
    started: true;
    planUuid: string;
    solveRun: number;  // 1-based counter of solve launches in this session. Each launch is a NEW run even when the request repeats — two identical re-optimizes are two distinct solves, and this field is what makes their results distinct.
    paidFeatures?: PaidFeatureReport;  // Present only when the plan uses paid features outside this account.
    warnings?: string[];  // Non-blocking notes about how the request was interpreted (e.g. an all-zero cost model replaced by the default rates of 1 per mile/km plus 1 per hour). Omitted when there is nothing to say.
    timeLimitS?: number;  // The solver's time budget for this run, in seconds (`options.time_limit_s` or the size-based default). Geocoding and matrix time come on top, so the run lands shortly after this many seconds of solving.
};
```
```ts
// What the solve did with the paid features the plan uses. Absent when the
// plan uses none, or when every one of them is included in the user's
// subscription — the common case, and nothing to report.
type PaidFeatureReport = {
    locked: string[];  // Paid capabilities the plan uses that this account's plan does not include, as `Feature` member names (e.g. `"Skills"`, `"DriverBreaks"`).
    stripped: string[];  // The subset DROPPED from this solve because the free allowance for paid features is spent. Empty while the allowance lasts. A non-empty list means the routes ignore those constraints — say so when reporting the plan, and name the upgrade as the way to get them back.
    freeRunsLeft: number;  // Free solves left that may use locked features (0 once spent).
};
```

### `routing24_export_excel` → `ExportExcelResult`
No input. Downloads the loaded plan as an Excel workbook (browser download):
with an optimized solution, the routes workbook (Route stops, Route stats,
Orders, Depots, Vehicles, Documentation); without one, the plan-data workbook.
Modifies nothing.
```ts
// Result of `routing24_export_excel`.
type ExportExcelResult = {
    exported: boolean;
    fileName?: string;  // The browser download's file name.
    kind?: "solution" | "plan";  // `solution` = optimized routes workbook; `plan` = plan data workbook.
    error?: string;
};
```

### `routing24_open_page` — `OpenPageInput` → `{ ok: true }`
Navigate the app to one of its pages — same as the user clicking the left
menu. `stops` is the Orders page; `plans` is My Plans.
```ts
// Input for `routing24_open_page`.
type OpenPageInput = {
    page: "home" | "plans" | "addresses" | "stops" | "depots" | "vehicles" | "optimize";  // The app page to open ("stops" is the Orders page).
};
```

### `routing24_show_on_map` — `ShowOnMapInput` → `ShowOnMapResult`
Focus one entity on the map: resolves a stop, depot or address-book business
id to its location, navigates to the entity's page when the current page does
not show it, and flies the map to it. Modifies nothing.
```ts
// Input for `routing24_show_on_map`.
type ShowOnMapInput = {
    id: string;  // Business id of a stop, depot, or address-book row.
};
```
```ts
// Result of `routing24_show_on_map`.
type ShowOnMapResult = {
    shown: boolean;
    kind?: "address" | "depot" | "stop";  // What the id resolved to.
    error?: string;
};
```

### `routing24_list_plans` → `ListPlansResult`
No input. Refreshes and lists the account's saved plans: `plan_id`, name,
created/updated timestamps, entity counts, and which plan is currently loaded
(`current: true`). First page of the plans index only — `truncated: true`
signals more exist. Modifies nothing.
```ts
// Result of `routing24_list_plans`.
type ListPlansResult = {
    plans: PlanListEntry[];
    truncated?: boolean;  // More plans exist beyond this first page of the index.
};
```
```ts
// One saved plan in the `routing24_list_plans` result.
type PlanListEntry = {
    plan_id: string;
    name: string;
    createdAt: number;  // Unix epoch milliseconds.
    updatedAt: number;  // Unix epoch milliseconds.
    entitiesCount?: { addresses: number; vehicles: number; depots: number; sites: number };
    current?: boolean;  // Set on the plan currently loaded in the app.
};
```

### `routing24_load_plan` — `LoadPlanInput` → `LoadPlanResult`
Loads a saved plan by `plan_id` (from `routing24_list_plans`), making it the
loaded plan. Unsaved changes of the plan it replaces are auto-saved first —
nothing is lost. Refuses while an optimization is running: cancel with
`routing24_cancel` or wait for it to finish.
```ts
// Input for `routing24_load_plan`.
type LoadPlanInput = {
    plan_id: string;  // The plan's id, as returned by `routing24_list_plans`.
};
```
```ts
// Result of `routing24_load_plan`.
type LoadPlanResult = {
    loaded: boolean;
    plan_id: string;
    name?: string;
    alreadyLoaded?: boolean;  // The plan was already the loaded plan; nothing changed.
    error?: string;
};
```

### `routing24_sql_query` / `routing24_sql_update` — `SqlQueryInput` → `SqlQueryResult` / `SqlUpdateInput` → `SqlUpdateResult`
SQLite over the loaded plan's live data: `sites`, `vehicles`, `depots`,
`addresses`, `site_tags`, `solution_routes`, `solution_stops`, `unassigned`.
`routing24_sql_query` reads (up to 200 rows; statements that write are
rejected); `routing24_sql_update` collects UPDATEs on the writable columns of
sites/vehicles/depots, validates them, and applies atomically — validation
failures change nothing, and every applied change is one `routing24_undo`
step. INSERT/DELETE are rejected: create with `routing24_upsert_*`, delete
with `routing24_delete_*` or by criteria with `routing24_run_script`. The
runtime tool description carries the full commented per-column schema.
```ts
// Input for `routing24_sql_query`.
type SqlQueryInput = {
    query: string;  // One or more SQLite statements. Statements that write plan data are rejected — use `routing24_sql_update`.
};
```
```ts
// Result of `routing24_sql_query`.
type SqlQueryResult = {
    columns: string[];
    rows: ((null | string | number)[])[];
    rowCount: number;  // Rows the statement produced (before the wire cap).
    matchedRows?: number;
    noop?: string;  // Set when an UPDATE matched rows but changed nothing.
    truncated?: string;
};
```
```ts
// Input for `routing24_sql_update`.
type SqlUpdateInput = {
    query: string;  // One or more SQLite statements; UPDATEs on the writable columns of sites/vehicles/depots journal cell changes that apply atomically.
};
```
```ts
// Result of `routing24_sql_update`.
type SqlUpdateResult = {
    applied: boolean;
    changedCells?: number;
    updatedEntities?: number;
    matchedRows?: number;
    noop?: string;  // Set when the statement matched rows but changed nothing.
    errors?: { kind: string; id: string; message: string }[];  // Validation rejections — nothing was applied.
    warnings?: string[];
    fleetDiagnostics?: { summary: string; problems: { category: "vehicle_incompatible"; count: number; explanation: string; lever: string }[] };
};
```

### `routing24_run_script` — `RunScriptInput` → `RunScriptResult`
Sandboxed JavaScript/TypeScript against a draft of the loaded plan: mutate
entity fields in `data`, delete via `deleteSites`/`deleteVehicles`/
`deleteDepots(predicate)`, assign a JSON-serializable summary to `__output`.
Journaled writes validate and apply atomically (one `routing24_undo` step); a
run with no writes returns `__output` only — usable for pure analysis. 20 s
budget, one run at a time. The runtime tool description carries the sandbox
globals reference.
```ts
// Input for `routing24_run_script`.
type RunScriptInput = {
    code: string;  // JavaScript/TypeScript for the plan sandbox. Mutate `data` entity fields; delete via deleteSites/deleteVehicles/deleteDepots(predicate); assign a short JSON-serializable summary to `__output`.
};
```
```ts
// Result of `routing24_run_script`.
type RunScriptResult = {
    applied: boolean;
    output?: string;  // The script's `__output`, JSON-serialized (capped).
    stdout?: string[];
    journaledWrites: number;
    deletionsMarked?: number;
    updatedEntities?: number;
    deletedEntities?: number;
    cascades?: string[];
    errors?: { kind: string; id: string; message: string }[];  // Validation rejections — nothing was applied.
    warnings?: string[];
    fleetDiagnostics?: { summary: string; problems: { category: "vehicle_incompatible"; count: number; explanation: string; lever: string }[] };
};
```

## Machine-readable JSON Schema

The full JSON Schema is in [schema.json](schema.json) (same directory).
