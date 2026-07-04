# Routing24 route optimizer — API reference

> Generated from Routing24's own types (skill version 0.1.0~beta). The
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

### `routing24_optimize` — `OptimizeInput` → `{ started: true; planUuid: string }`
Creates a fresh plan, fetches the O/D matrix, and starts the solve. Returns at once.
```ts
type OptimizeInput = {
    depot: Place;  // Single depot, used as both start and end for every vehicle.
    stops: OptimizeStop[];  // min 1
    vehicles: OptimizeVehicle[];  // min 1
    options?: { time_limit_s?: integer };
};
```
```ts
// A geographic point the caller supplies either as coordinates or as an address
// string to geocode. `lat`/`lng` are the solver's own location fields.
type Place = {
    lat?: number;
    lng?: number;
    address?: string;
};
```
```ts
// A delivery/pickup stop: a place plus solver site constraints.
type OptimizeStop = {
    lat?: number;
    lng?: number;
    address?: string;
    pickup?: number;
    delivery?: number;
    service_duration_s?: number;
    tw_early_s?: number;
    tw_late_s?: number;
    prize?: number;
    required?: boolean;
    id?: string;
};
```
```ts
// A vehicle (type): solver vehicle constraints; `available_count` clones this type.
type OptimizeVehicle = {
    tw_early_s?: number;
    tw_late_s?: number;
    capacity?: number;
    available_count?: number;
    cost?: { fixed?: number; distance?: number; duration?: number; stop?: number };
    id?: string;
};
```
- Each depot/stop needs **either** `lat`+`lng` **or** an `address` (geocoded
  automatically; if any fail, the call rejects listing them).
- Times are **seconds since midnight**. `delivery`/`pickup` are single-dimension
  loads. `available_count` = identical vehicles of that type. The single depot is
  both start and end.

### `routing24_status` → `OptimizeStatus`
No input. Snapshot of the current optimization — poll (~every 3s) while solving.
`phase` walks `idle → geocoding → matrix → solving → done` (or `error`).
```ts
type OptimizeStatus = {
    phase: OptimizePhase;
    running: boolean;
    progress?: number;  // 0..1 while solving, when available.
    feasible?: boolean;
    routes?: number;
    stops?: number;
    unservedCount?: number;
    distance?: number;
    distanceUnit?: "km" | "mi";
    durationHours?: number;
    error?: string;
    planUuid?: string;
};
```

### `routing24_solution` → `PlanSolution`
No input. The **full** solution for the currently-loaded plan — whether you just
optimized it or opened it via its URL — with no parsing needed. Same rollups as
`routing24_status`, plus every used route's ordered depot→stops→depot sequence
with each stop resolved to its `id`/`address`/`lat`/`lng`, arrival & departure
times (seconds since midnight), per-leg distance/duration and loads. Returns
`{ available: false }` when the plan has no solution yet — call it once
`routing24_status` reports `phase:"done"`.
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
    unservedCount?: number;  // Number of site stops that could not be served.
    distance?: number;  // Total travel distance across all routes, in `distanceUnit`.
    durationHours?: number;  // Total duration across all routes, hours.
    unserved?: string[];  // Ids of the sites left unserved, if any.
    routes?: PlanSolutionRoute[];  // One entry per used route, in on-screen order.
};
```
```ts
// One vehicle's optimized route: an ordered depot → stops → depot sequence.
type PlanSolutionRoute = {
    index: number;  // 1-based route number, matching the on-screen order.
    vehicleId?: string;  // Id of the vehicle serving this route.
    siteCount: number;  // Count of site stops served (the depot start/end are excluded).
    distance: number;  // Total travel distance of the route, in `distanceUnit`.
    durationHours: number;  // Total route duration (travel + service + wait), hours.
    startTimeS?: number;  // When the vehicle leaves the depot, seconds since midnight.
    endTimeS?: number;  // When the vehicle returns to the depot, seconds since midnight.
    stops: PlanSolutionStop[];  // Depot → stops → depot, in visit order.
};
```
```ts
// One stop on an optimized route, in visit order. Every field is resolved from
// the loaded plan — the caller never parses raw solver output.
type PlanSolutionStop = {
    seq: number;  // 0-based position within the route (0 = the starting depot).
    type: "depot" | "site";  // Whether this node is the depot or a delivery/pickup site.
    id?: string;  // The site/depot id (the id you passed to optimize, or an auto-assigned one).
    address?: string;  // Matched street address, when known.
    lat?: number;
    lng?: number;
    arrivalTimeS: number;  // Arrival time, seconds since midnight.
    departureTimeS: number;  // Departure time (arrival + service), seconds since midnight.
    serviceDurationS: number;  // Time spent servicing this stop, seconds.
    legDistance: number;  // Distance of the travel leg arriving at this stop, in `distanceUnit`.
    legDurationS: number;  // Duration of the travel leg arriving at this stop, seconds.
    carriedLoad: number;  // Vehicle load carried on arrival.
    deliveredLoad: number;  // Load delivered at this stop.
    pickedUpLoad: number;  // Load picked up at this stop.
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
