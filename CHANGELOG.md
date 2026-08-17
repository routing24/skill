# Routing24 route optimizer — changelog

Version history of the generated `routing24-optimizer` skill content (SKILL.md +
references/*) and `llms.txt`. Generated; do not edit by hand.

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
