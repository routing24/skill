# Routing24 route optimizer — changelog

Version history of the generated `routing24-optimizer` skill content (SKILL.md +
references/*) and `llms.txt`. Generated; do not edit by hand.

## 8.3.0

`routing24_map_image` now also returns `image_url`: the same frame at full resolution behind a temporary, unauthenticated https link, valid until `image_url_expires_at` (about 30 minutes). Fetch it with plain HTTP and no credentials. That is the only way to put the map into a file you produce (a PDF, a report, a slide) — the MCP image block reaches the conversation and is never written to a code sandbox — and it is the link to hand the user when they want to see the map bigger. The image block itself is now a downscaled preview so the whole result stays inside the host inline-result limit; `mimeType`/`width`/`height`/`bytes` describe the full-resolution frame at the URL. If a fetch is refused with `host_not_allowed`, the sandbox has an egress allowlist that needs `routing24.ai` added. `image_url` is absent when the upload failed; the preview is still there.

## 8.2.0

Multi-unit loads. A plan can declare up to 4 named load units (independent capacity axes: pallets AND kg AND seats). `delivery`/`pickup`/`capacity` accept an array of per-unit entries over the declared unit names — {"delivery":[{"unit":"pallets","value":2},{"unit":"kg","value":300}]} — next to the existing bare number, which stays valid while the plan declares at most one unit. The list tools echo the same shape, the `units` SQL table maps unit names to the per-unit SQL columns (pickup_kg, capacity_seats), and `UpsertResult.unitsDeclared` reports units a batch declared. Unit names may be introduced only while the plan has none; after that an unknown name rejects the row.

## 8.1.0

The connector guidance now names the page that has to be open: https://routing24.com/app. The MCP endpoint is served from a different host than the app, so an assistant holding only the connector URL had nothing to go on and sent users to the endpoint host, which serves the server and never the app — a tab that can never connect. No tool contract changed.

## 8.0.0

BREAKING — the four `routing24_upsert_*` tools apply PARTIALLY. A row the contract rejects no longer discards the batch: the good rows are written and each bad one comes back in the new `UpsertResult.rejected[]` as { row, id?, error }, where `row` is its 0-based index in the array you sent — the only handle that works when the missing field IS the `id`. Re-send just those rows. `added + updated + skipped` now always equals the number of rows sent; `rejected` is capped at 20 entries with `rejectedOmitted` counting the rest, and `applied: false` means nothing at all was written. Everything that previously threw for one row is covered: a missing `id`, a constraint out of range, a row with no address, a stop whose tag is both required and forbidden, a vehicle naming a depot the plan does not have, and a shape error confined to one row. `unresolvedRefs[]` is GONE — a dropped row is now in `rejected`, while a value dropped from a row that DID land is in the new `warnings[]`. Also breaking: `id` is now REQUIRED in the schema for stop, depot and vehicle rows, which it always was in practice. `routing24_upsert_addresses` still returns per-row `rows` for batches of at most 5, now covering the accepted rows only and carrying each row's input index.

## 7.1.0

`routing24_edit_split_route` no longer refuses a split just because the source route's vehicle type is fully committed. Omit `vehicle` and the tail still reuses that type while it has an instance left; when it does not, an UNUSED vehicle is taken instead and named in the new `EditResult.autoPickedVehicle` (`route`, `vehicle`, `insteadOf`) — the tail then runs a different capacity, shift and cost, so report which vehicle it got. The editing-tool `SessionState` gained `unusedVehicles[]` ({ id, count }): the vehicle types with an instance free for a new route, previously derivable only by counting `routing24_list_vehicles` against `routeStats[].vehicleId`. `vehicle_overused` rejections now name those unused vehicles, and when there are none they correct the engine's "add a vehicle to the plan" advice, which cannot help mid-session.

## 7.0.0

BREAKING — the hosted MCP server is now the primary way in. Add https://routing24.ai/mcp as a custom connector (OAuth 2.1) and the `routing24_*` tools arrive natively, with no page scripting. Two preconditions replace the old browser-agent requirement: the user stays signed in, and a https://routing24.com/app tab stays open, because every call executes inside it. The procedure is now stated as tool calls rather than `javascript_tool` expressions, and the connector guidance (sessions, plan scoping, the drift rejection) is spliced from the same source the server returns from `initialize`, so the two cannot disagree. Driving the page over WebMCP still works and moved to its own section, "Driving the page directly", which now owns the getTools/executeTool wrapper. Corrected: earlier versions claimed there is no server API, no API key, and that every tool works anonymously; all three hold on the WebMCP surface only. Why: the skill taught the one path most assistants cannot take — adding a connector is a setting, scripting a page is a capability.

## 6.2.0

Driver breaks are first-class in every read surface. `OptimizeStatus.routeStats[]`, `routing24_route` and the editing-tool `SessionState.routes[]` now carry `breakCount` / `breakDurationS` per route (absent = no breaks — legitimate when total driving stays under the rule trigger), so "which routes have breaks" is answerable from `routing24_status` alone. The SQL surface gained `solution_routes.break_count` / `break_duration_s` and `solution_stops.service_duration_s` (for break rows, the pause length). Docs now spell out the rest semantics: without `service_counts` the driving clock resets ONLY at scheduled break stops — waiting, service and depot reloads never count as a break, however long.

## 6.1.0

Unpriced fleets get a real default objective: when no vehicle prices distance, duration or load-distance, the optimizer now bills 1 per mile (or km, per the plan's display unit) PLUS 1 per hour — previously 1 per yard/metre and nothing for time — so `cost.total` reads roughly as miles(km) + hours instead of raw distance in matrix units. `VehicleEffectiveRates.defaultRates` may now contain `duration` (alongside `distance`/`overtime`), and the unpriced `costModel.note` names the new defaults. Rate quantization is finer too: the engine's fixed-point step is now 1e-6 per wire unit, eliminating the old visible 1.08/hour quantization error — an effective rate now matches the authored one at the 2 decimals these surfaces report. Routes for unpriced plans can change — time now steers the optimization, as the product copy always said.

## 6.0.0

BREAKING — cost objects are self-describing: `SolutionCost` and `RouteCost` are now `{ total, components[] }` (the named `overtime`/`vehicle`/`stop`/`rideOvertime`/`loadDistance` fields are gone). Each `CostComponent` is one addend of `total` — `kind` (`fixed`/`distance`/`duration`/`overtime`/`stop`/`rideOvertime`/`loadDistance`/`other`) + `amount`; `overtime` is its own line now (no longer folded into duration), so lines always sum to `total`. On `routing24_route` every line adds the serving vehicle's effective `rate`, billed `quantity` and `unit` for user-verifiable arithmetic. New `OptimizeStatus.costModel` states the pricing the solve ACTUALLY used per vehicle, engine fallbacks marked in `defaultRates` (with `defaultRate: true` on the affected lines): in particular `priced: false` means no vehicle has any cost rate and `cost.total` is plain travel distance in yards/metres — the ready-to-quote `note` explains it. Why: "what does 43000 mean?" was unanswerable from `{ total, overtime, vehicle }`, and a default rate presented as a configured cost misled users.

## 5.0.0

BREAKING — the surface is coordinate-free: `routing24_geocode` is REMOVED and `lat`/`lng` are gone from every input and output (stop/depot/address rows, list rows, `routing24_route` stops). The `address` string is the only location carrier — geocoding happens internally, rows report `status` (`geocoded`/`ungeocoded`), and a caller holding exact coordinates sends a decimal `"lat, lng"` literal AS the address (it resolves to that exact point and stays the label). Confirm doubtful addresses with `routing24_upsert_addresses`: a batch of at most 5 rows returns per-row `status` + the canonical `matched` text (`UpsertAddressesResult.rows`); known `address`+`area` pairs reuse their stored location and are never re-geocoded. SQL: the coordinate columns and the `viewport` table are gone — use the `status` and `in_viewport` columns, `distance_m(uuid_a, uuid_b)` and `in_polygon(uuid, geojson)` (replacing `haversine_m`). Why: a model-invented coordinate written into an upsert placed a stop 1.2 km off; with no coordinate fields, invented numbers are a schema violation by construction.

## 4.0.0

BREAKING — `routing24_optimize` is REMOVED. Plans are built incrementally: `routing24_new_plan` commits a fresh empty plan (replacing the loaded one; unsaved changes are auto-saved and the app asks the user to confirm when the open plan holds data), the `routing24_upsert_*` tools create the depot/vehicles/stops (geocoding internally, problems reported in `addressDiagnostics`; every row needs a caller-chosen `id`), and `routing24_reoptimize_plan` runs the solve (same fire-and-poll result, `paidFeatures` included). A single opaque 50-stop payload cannot be built in iterations or reviewed; the incremental path can.

## 3.1.0

Stale-session transparency — `routing24_status` gains an optional `dataSync` block reporting per-entity added/removed/changed counts when the plan data changed after the last optimization (the editing tools see the solve-time plan; such changes take effect on the next solve). Vehicle/stop edit rejections against a since-changed plan now say so and name the way out instead of the engine's bare "add a vehicle to the plan".

## 3.0.1

Docs correction — the structural-rejection examples no longer cite non-empty route removal; `routing24_edit_remove_routes` unassigns remaining stops in the same atomic edit, so callers never see `route_not_empty`.

## 3.0.0

BREAKING rename — `routing24_optimize_current` → `routing24_reoptimize_plan`, `routing24_optimize_route` → `routing24_reoptimize_route` (scope in the name); re-optimizing is user-initiated only and confirms when manual edits exist.

## 2.2.0

Status long-poll — while a solve runs, `routing24_status` holds its reply until the solve lands or ~15s pass, so callers loop on it without sleeping between calls. `routing24_optimize` / `routing24_reoptimize_plan` results gain `timeLimitS` (the solver's time budget in seconds).

## 2.1.0

One tool per action, part 2 — `routing24_list_entities` / `routing24_upsert_entities` / `routing24_delete_entities` are REPLACED by per-kind tools (`routing24_list_stops`, `routing24_upsert_vehicles`, `routing24_delete_depots`, …). Each list tool returns ONE row type instead of a `kind`-switched union, and stop/depot rows now carry `area`, which the upsert shape accepts — so a row read out can be written straight back.

## 2.0.0

One tool per action. `routing24_edit` (an ordered batch of a nine-way op union) is REPLACED by nine flat `routing24_edit_*` tools — move_stops, unassign_stops, set_route_vehicle, create_route, remove_routes, split_route, merge_routes, mark_user_assigned, clear_user_assigned — each taking one shape and applying atomically. `remove_routes` unassigns the stops still on those routes in the same atomic edit, so the old empty-then-remove batch is gone. Breaking: callers composing `ops` arrays must move to the named tools.

## 1.3.0

Granular read surface — `routing24_solution` is REMOVED. Read the overview (rollups + economic cost/objective + per-route `routeStats`) from `routing24_status`, one route's ordered stops from `routing24_route` (by 1-based index), and the unassigned reasons (ids, prose diagnostics, insertion quotes) from `routing24_unassigned`. No tool dumps the whole solution.

## 1.2.1

Failure-semantics completions — the solve path drops an unservable bounded pair to `unassigned` (`shelf_life` rows are evaluate/edit-only), a `no_break` stop hosts a break anyway when nothing else can (reported `break_location`), breaks never interrupt a travel leg (over-long leg reports `break_schedule`), and `routing24_solution` documents the break/shelf problem codes.

## 1.2.0

Breaks combine with ride-bounded transfers — the 1.1.0 rejection is gone; break placement avoids the pickup-to-delivery span where it can, and a break placed inside it counts as ride time, priced against the bound and its overtime band.

## 1.1.1

Cold-chain corrections — a plain stop that carries a `pickup` load alongside `max_time_in_vehicle_s` is DROPPED (unassigned, `field_not_applicable`), it does not reject the call; the cost `total` breakdown names its ride-overtime and load-distance addends; the free-tier sentence no longer calls every `cost` field free.

## 1.1.0

Cold chain — product segregation (stop `load_class` + vehicle `no_mix_load_classes`) and shelf life (`max_time_in_vehicle_s`, `max_ride_overtime_s`, vehicle `cost.ride_overtime`), both PRO, with the plain-stop clock pinned at `release_time_s`, the linked-stop ride bound, and its rejection when combined with driver breaks (removed in 1.2.0).

## 1.0.0

First stable release. Docs corrections — undo/redo empty-history shape is `errorCode` (not `error`), the status phase walk includes `saving`, the tool error-code list is complete (derived from the schema), and product limits/break presets are interpolated from the app's own constants.

## 0.6.0

Plans & paid features — the generated tier table, per-field PRO/STARTER markers on OptimizeStop/OptimizeVehicle, and the `paidFeatures` report on `routing24_optimize` (the agent path now spends the same free allowance as the Optimize button and strips locked constraints once it is gone).

## 0.5.0

Driver breaks — per-vehicle break_rules (EU/US presets or custom, splittable, service_counts) + fixed clock-window breaks, carry-in driving allowance (period_driving_limit_s/period_driven_s), stop/depot no_break, and type:"break" stops in `routing24_solution`.

## 0.3.0

driftFromOptimized — every read/edit surface reports how far the plan drifted from the last full solve (user edits included), with a fixed-threshold severity the agent must relay to the user.

## 0.2.0

Manual route editing (`routing24_edit` batches, `routing24_optimize_route`, `routing24_undo`/`redo`), full solver-constraint parity in `routing24_optimize` (site groups, depot windows, force lists, reloads, overtime, max distance), and the problems/waits/session read surface on `routing24_solution`/`status`.

## 0.1.0

The `window.R24Agent` façade was replaced by `routing24_*` WebMCP tools on `document.modelContext` — a breaking contract change.
