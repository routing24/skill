---
name: routing24-optimizer
description: >-
  Plan and optimize vehicle delivery routes: turn a list of stop addresses and
  vehicles into an efficient multi-stop route plan with stop assignments,
  sequence, distance and ETAs, plus a shareable plan link. Use whenever the
  user wants to plan routes, optimize delivery or pickup stops, build a
  delivery run or dispatch schedule, solve a vehicle routing problem (Rich
  VRP), sequence stops for one or more drivers, vans or trucks from a depot,
  do route planning or last-mile delivery optimization, edit routes manually
  (move stops between routes, unassign stops, split/merge routes, change
  vehicles, undo), or asks for Routing24 routing24.com. Driven from the user's
  own browser tab via WebMCP tools (routing24_new_plan,
  routing24_upsert_stops, routing24_reoptimize_plan, routing24_status,
  routing24_edit_move_stops, routing24_save, ...) registered on
  document.modelContext.
license: Proprietary. Visit https://routing24.com/terms for full terms and conditions.
compatibility: >-
  Requires a browser agent on an open routing24.com tab: a WebMCP-capable
  host, or a JS-eval integration such as Claude in Chrome / Cowork
  (javascript_tool and navigate) driving the routing24_* tools via
  document.modelContext. Route optimization runs client-side; geocoding,
  routing/distance matrices, and ML/LLM run on Routing24 servers under an
  opaque token issued for a registered or anonymous account. No public API or
  API key is needed, and no sign-in: the full flow works anonymously.
metadata:
  author: Routinghub LLC
  version: "6.1.0"
---

# Routing24 route optimizer

Turn a natural-language routing request ("optimize these 8 addresses with 2 vans
from this depot") into an optimized route plan on **Routing24**, shown on the map,
with a link the user can open.

Routing24 exposes **no public Optimization API** — you drive it through the page. The
route optimization runs **client-side in the browser** (WASM), while geocoding,
routing/distance matrices, and ML/LLM run on **Routing24's own servers**, reached
with an opaque auth token issued for the user's account (registered or anonymous).
The page registers **WebMCP tools** under the `routing24_*` prefix on
`document.modelContext` (the W3C WebMCP draft surface; the app bundles a polyfill
so the tools exist even without native browser support). Discover them with
`getTools()`, invoke them with `executeTool(tool, JSON.stringify(args))` — which
resolves to a **JSON string** you must parse (and **rejects** on validation or
handler errors). The optimizer runs client-side in WASM, so `routing24_reoptimize_plan`
is asynchronous: you start it, then **poll `routing24_status`**.

## Runtime requirements

- This skill drives Routing24 **in the user's own browser**. It needs either a
  **WebMCP-capable browser agent host**, or a JS-eval browser integration —
  **Claude in Chrome / Cowork** (Chrome or Edge) with the `navigate` and
  `javascript_tool` tools, which calls the same tools via
  `document.modelContext`. If neither is available, tell the user this
  automation requires a browser integration and stop.
- You drive Routing24 through the `routing24_*` WebMCP tools; there is **no
  public API and no API key**. Route optimization runs **client-side in the
  browser**, while geocoding, routing/distance matrices, and ML/LLM run on
  **Routing24's own servers** under an opaque token issued for the user's account
  (registered or anonymous). **No sign-in is required** — the full flow (create,
  optimize, render, save, share) works anonymously. Do not ask the user to log in.
  Signing in changes **where** the plan is stored (see the plan-link note under
  *Notes & pitfalls*) and which **paid constraints** a solve may keep (see
  *Plans & paid features*) — neither is a reason to stop or to ask for a login.

## Plans & paid features

Most plan-data fields are free. The fields below need a paid Routing24 plan;
everything not listed — addresses, loads, time windows (**including multi-day
`+1` offsets**), shifts, capacity, `max_reloads` and every `cost` field the
table does not name — works on every account.

| Capability | Plan | Fields |
| --- | --- | --- |
| Alternative order groups | PRO | `group` |
| Alternative pickup/delivery locations | PRO | `group` |
| Driver breaks & driving limits | PRO | `break_rules`, `fixed_breaks`, `period_driving_limit_s`, `period_driven_s` |
| Force allow / deny orders | PRO | `force_allow_sites`, `force_deny_sites` |
| Load-distance cost (cost per load-km) | PRO | `cost.load_distance` |
| Max distance | PRO | `max_distance` |
| Max duration | PRO | `max_duration_s` |
| Max overtime | PRO | `max_overtime_s` |
| Max time in vehicle (shelf life) | PRO | `max_time_in_vehicle_s`, `max_ride_overtime_s`, `cost.ride_overtime` |
| Order sequences | PRO | `sequence_group`, `sequence_rank` |
| Overtime cost per hour | PRO | `cost.overtime` |
| Pickup & delivery (transfers) | PRO | `transfer_type`, `transfer_id` |
| Product segregation (load classes) | PRO | `load_class`, `no_mix_load_classes` |
| Reload depots | PRO | `reload_depots` |
| Skills (vehicle & order tags) | PRO | `required_tags`, `forbidden_tags`, `tags` |

**What happens on a free account.** Nothing is rejected and nothing is hidden:
a plan using paid fields solves normally for the first
**5 optimizations**. After that `routing24_reoptimize_plan`
still runs, but it **drops those constraints from the solve** and says so in
its result — `paidFeatures.stripped` lists what it ignored and
`paidFeatures.freeRunsLeft` is `0`.

**When `stripped` is non-empty the routes do not honour those constraints.**
Tell the user plainly which ones were ignored and that upgrading restores them;
never present such a plan as if the request was fully satisfied. The allowance
is per browser, so it is not something you can reset or work around — do not
retry the call hoping for a different answer.

## Procedure

1. **Open the app.** `navigate` to `https://routing24.com/app/plan/new/optimize`.

2. **Verify the tools are present.** Via `javascript_tool`:
   ```js
   const mc = document.modelContext ?? navigator.modelContext;
   (await mc?.getTools())?.map(t => t.name) ?? null;
   ```
   Expect the `routing24_*` names (`routing24_reoptimize_plan` etc.). If `null` or
   missing, the page may still be loading — wait 2s and retry, then reload once;
   if still missing, tell the user the Routing24 agent tools aren't available on
   this page and stop.

3. **Set up a call helper** (used by every later step; `executeTool` takes the
   tool OBJECT from `getTools()` and a JSON-string argument, and resolves to a
   JSON string):
   ```js
   const mc = document.modelContext ?? navigator.modelContext;
   const tools = await mc.getTools();
   window.__r24call = async (name, args = {}) => {
       const tool = tools.find(t => t.name === name);
       if (!tool) throw new Error('tool not found: ' + name);
       const raw = await mc.executeTool(tool, JSON.stringify(args));
       return raw == null ? null : JSON.parse(raw);
   };
   ```

4. **(Optional) Note who's signed in.** `await __r24call('routing24_get_auth_user')`
   returns `{ user }` — the user's email, or `"anonymous"`. **Do not require
   sign-in** — the whole flow works anonymously. This only affects *where* the
   saved plan lives and what you tell the user about the plan link (see step 11).

5. **Parse the request** into entity batches — depot row(s), vehicle rows, stop
   rows (see the **API contract** for the row shapes). Times are
   seconds-since-midnight; every row needs a caller-chosen `id` (the business
   id every other tool refers to). The `address` string is the ONLY location
   carrier — no tool takes or returns coordinates. A caller holding exact
   coordinates sends them AS the address: a decimal `"lat, lng"` literal
   (e.g. `"25.19882, 55.27939"`) resolves to that exact point and stays the
   row's address label. A solve needs **≥1 stop**, **≥1 vehicle**
   and a depot; with a single stop there is nothing to sequence, so expect
   real requests to carry ≥2 stops. Bad input makes a call reject with a
   message naming the offending fields — relay it to the user.

6. **Start a fresh plan.** Everything from here on — confirmed addresses
   included — lands in this plan:
   ```js
   await __r24call('routing24_new_plan', {}); // fresh EMPTY plan (replaces the loaded one)
   ```

7. **Confirm doubtful addresses.** The `routing24_upsert_*` tools geocode every
   address internally, so pre-resolving is NOT needed. For the few addresses
   you are unsure you read correctly (a typo you fixed, a part you could not
   place), save them first with
   `await __r24call('routing24_upsert_addresses', { addresses: [<those rows>] })`
   — a batch of **at most 5 rows** returns `rows`, one per input, each with
   `status` (`'geocoded' | 'ungeocoded'`) and the canonical `matched` text
   the geocoder resolved (present only for rows geocoded by this call).
   - Show the user any row with `status: 'ungeocoded'` (not found) and ask
     them to fix the address. Also surface `matched` values that look wrong
     and confirm.
   - Then send the stop rows with the SAME address strings — they reuse the
     locations already resolved, nothing is re-geocoded.

8. **Create the entities and start optimization.** Build the plan piece by
   piece, then solve it:
   ```js
   await __r24call('routing24_upsert_depots',   { depots:   [ /* depot rows */ ] });
   await __r24call('routing24_upsert_vehicles', { vehicles: [ /* vehicle rows */ ] });
   await __r24call('routing24_upsert_stops',    { stops:    [ /* stop rows, batches fine */ ] });
   await __r24call('routing24_reoptimize_plan', {});
   ```
   Each upsert result carries `addressDiagnostics` when rows failed to geocode
   or landed far from the rest — resolve those before solving.
   `routing24_reoptimize_plan` returns quickly with
   `{ started: true, planUuid, timeLimitS }` — `timeLimitS` is the solver's
   time budget in seconds.
   - If the result carries `paidFeatures`, the plan uses constraints outside the
     user's subscription. `stripped` is empty while the free allowance lasts;
     once it is non-empty the solve **ignored** those constraints — carry that
     into step 11 (see *Plans & paid features*).

9. **Poll progress.** Poll by looping on routing24_status until phase is 'done' or 'error': while a solve runs each call holds its reply up to ~15s (returning early when the solve lands), so call it back-to-back — no sleep between calls is needed.
   - Relay `progress` (0–1) and, once available, `routes` / `distance` / `feasible`.
   - Small jobs finish in seconds; large ones can take minutes (keep the tab
     active). If `phase === 'error'`, report `error` and stop.
   - To abort on the user's request: `await __r24call('routing24_cancel')`.

10. **Show + save.** Run `await __r24call('routing24_render')` (brings the routes
   onto the map), then `await __r24call('routing24_save')` to persist and get
   `{ saved, planUrl }`.
   - To *show the user the map*, take a screenshot of the tab after render —
     the tools return JSON, not images.
   - To read the solution back, call `await __r24call('routing24_status')` for
     the overview (rollups + per-route `routeStats`), then
     `await __r24call('routing24_route', { route })` for one route's ordered
     stops (resolved addresses, ETAs, loads). Both work now and on a plan
     reopened by URL. Read what's unserved and why with
     `await __r24call('routing24_unassigned')`.

11. **Report** to the user:
   - Number of routes, total distance (with unit), total duration, and any
     `unassignedCount` (stops that couldn't be served — mention them).
   - The optimized **stop sequence per route** (pull one route's stops from
     `routing24_route` when the user wants the itinerary, not just totals).
   - Whether the solution is `feasible`.
   - **Any `paidFeatures.stripped` from step 8** — name the constraints the
     solve ignored and that upgrading restores them. A plan that quietly drops
     the user's tags, breaks or transfers is worse than no plan.
   - The **plan link** (`planUrl`) they can open, and note the map on screen shows
     the routes. If the user is **anonymous** (from step 4), add that this link
     opens the plan **only on this computer** and may be deleted later — it is not a
     durable share link.

## Editing routes manually

Once a plan has a solution (fresh solve or a plan opened by URL), you can edit
it in place — the same manual-editing engine the UI's drag & drop uses:

1. **Read the current arrangement**: `await __r24call('routing24_status')` for
   the per-route `routeStats` (each carries `route`, the 1-based on-screen
   route number, empty routes included) and
   `await __r24call('routing24_route', { route })` for a route's stop
   `id`s — every tool addresses routes by that same `route` number and
   stops by `id`.
2. **Call the tool for the action** — one tool per action, one flat input each,
   nothing to compose: `routing24_edit_move_stops` (anchor with
   `before`/`after` a planned stop id, or `to_route` +
   `placement: 'append' | 'best'`), `routing24_edit_unassign_stops`,
   `routing24_edit_set_route_vehicle`, `routing24_edit_create_route`,
   `routing24_edit_remove_routes` (unassigns any stops still on them, same
   atomic edit), `routing24_edit_split_route`, `routing24_edit_merge_routes`,
   `routing24_edit_mark_user_assigned` / `routing24_edit_clear_user_assigned`.
   Each call applies atomically and is one undo entry; a rejection says why in
   `rejection.code` (`stale_revision` = the plan changed under you: re-read
   `routing24_status` and retry).
3. **Judge the result, don't guess**: edits are never rejected for breaking
   constraints — problems are REPORTED instead. Check `state.feasible`,
   `state.problemsCount`, per-route `problems`, and `userAssignedReports`
   (per manually-placed stop: `intrinsic` = would violate anywhere,
   `introduced` = caused by this placement). If a placement looks bad, fix it
   with another call or `routing24_undo`.
   **Always check `state.driftFromOptimized`** — how far the plan has moved
   from the last full solve (the user's own manual edits count too). When
   `severity` is `degraded` or `severe` you MUST tell the user, quoting
   `summary` — it leads with the COST change, the number the user pays
   (e.g. "cost +2348 (+12.3%), distance +12.4%, 1 new problem vs the last
   optimization") — and stop there. An edit the user asked for is expected
   to cost more; never re-optimize (or propose it) to walk one back.
   Report `improved` drift too — it reinforces that the edits helped.
4. **Re-optimize only on the user's request**: `routing24_reoptimize_route`
   re-sequences one rough route (sub-second, synchronous; membership stays),
   `routing24_reoptimize_plan` recomputes the whole arrangement — both replace
   manual edits in their scope, so when `status.undoDepth > 0` confirm with
   the user first. `routing24_edit_remove_routes` deletes routes — the
   remaining routes RENUMBER afterwards, so always take fresh route numbers
   from the returned `state.routes`.
5. **Explain unassigned stops**: `routing24_unassigned` with
   `{ refresh_diagnostics: true }` returns `unassignedDiagnostics` — a prose
   summary + per-site category/explanation/blockers/levers you can relay.
6. **Save** with `routing24_save` when the user is happy.

While you edit, the app locks behind a full-screen **"Agent controlled"**
overlay (blue dashed frame + a floating pane at the bottom) so the user can't
race you; they resume via its **Take control** button. `routing24_undo` /
`routing24_redo` walk the SAME history as the user's own edits — never undo
speculatively; prefer a compensating `routing24_edit_*` call.

## Reference files

Load these only as the task calls for them (progressive disclosure):

- **API contract** (per-tool signatures + field definitions): [references/api.md](references/api.md)
- **Machine-readable JSON Schema** (OpenAPI 3.1 / JSON Schema 2020-12): [references/schema.json](references/schema.json)
- **Ready-to-eval WebMCP call snippets** (`javascript_tool`-ready): [references/examples.md](references/examples.md)
- **Always-current contract** — fetch `https://routing24.com/llms.txt` if a validation error
  suggests the bundled reference is behind the deployed API.

## Version & keeping current

- This skill is **version 6.1.0**. Its bundled reference
  (`references/api.md` + `references/schema.json`) is generated from Routing24's
  own types and is correct as of this version.
- The **always-current** copy of the full contract is served at
  `https://routing24.com/llms.txt` (regenerated from the deployed API on every release). If a
  call rejects with a validation error that looks like a field this reference
  doesn't describe, fetch that URL and use its schema — then consider
  re-downloading the latest skill from `https://routing24.com/routing24.skill`.
- To update the skill itself, re-download `https://routing24.com/routing24.skill` and re-install
  it; that is the update mechanism.

## Notes & pitfalls

- You drive everything through the user's own tab via the WebMCP tools; you never
  call a Routing24 server API directly. The tools do that for you (geocoding,
  routing/matrix, ML/LLM) under an opaque token, while the optimizer itself runs
  client-side in the tab.
- `executeTool` resolves to a **JSON string** (parse it; `null` means no
  value) and **rejects** on validation or handler errors — wrap calls in
  try/catch and relay the message. You cannot receive an image from the tools —
  to show the user the map, call `routing24_render` then screenshot the tab.
- Prefer `document.modelContext`; `navigator.modelContext` is a deprecated
  alias kept for older hosts.
- `routing24_new_plan` starts a **new plan** each time (replacing the loaded
  one — the app asks the user to confirm when the open plan holds data, and
  auto-saves unsaved changes first). When the user is
  **anonymous**, the plan is stored **only in this browser on this computer** and
  may be deleted later, so the plan link opens only here — it is not a durable
  share link. Say this when you hand over the link. (Signing in before saving
  persists the plan to the user's account so the link also opens on their other
  devices — but sign-in is never required to plan, save, or share.)
- If `routing24_status` never leaves `matrix`/`solving`, the network (matrix
  service) or the solve may be slow — keep polling; only treat it as failed on
  `phase:'error'`.
- **Editing needs a solved plan** (`routing24_status` reports `phase:"done"`
  with `routeStats`) and refuses while a full solve runs
  (`errorCode:"solve_in_progress"`). Route numbers are 1-based (the
  on-screen order) and RENUMBER after `routing24_edit_remove_routes` — always
  re-read them
  from the returned `state.routes` or a fresh `routing24_status`.
- **Two failure channels on the editing tools**: `rejection.code` = the
  ENGINE refused the batch (structural: unknown ids, bad anchors,
  `stale_revision`, …); `errorCode` = the tool refused or failed around the
  engine (`solve_in_progress`, `no_solution`, `slot_out_of_range`, `route_too_small`, `optimize_running`, `nothing_to_undo`, `nothing_to_redo`, `session_error`).
  `session_error` means the editing
  session itself failed and the app resynced — re-read `routing24_status`
  and retry ONCE.
- **Edits are never rejected for violating constraints** — time windows,
  capacity etc. are scored and reported (`state.problemsCount`, per-route
  `problems`, `userAssignedReports`), exactly like the UI's manual drag &
  drop. Rejections are structural only (`rejection.code`: unknown ids, bad
  anchors, `stale_revision`, …).
- While you edit, the app locks behind a full-screen **"Agent controlled"**
  overlay until the user clicks its "Take control" button — tell the user to
  take control from there when you hand the plan back. The user may also edit
  concurrently before your first edit: a `stale_revision` rejection means
  re-read `routing24_status` and rebuild your batch.
- `routing24_undo`/`routing24_redo` walk the same history as the user's own
  manual edits — never undo blindly.
- **`driftFromOptimized` is your metrics conscience.** It compares the CURRENT
  plan to the last full solve (baseline frozen at solve completion; only a new
  solve resets it) and covers ALL changes since — including edits the user made
  by hand while you were away. On every `routing24_status`
  read and in every edit result: `severity` `degraded`/`severe` MUST be
  relayed to the user with the ready-made `summary` — and nothing more: an
  edit the user asked for is expected to cost more, and re-optimizing (or
  proposing it) to walk one back is never your call. Absent drift = the plan
  predates the baseline feature or has no completed solve — nothing to
  compare against.
- **Money, coverage and the objective are three separate registers — never
  mix them.** `cost` is money (report it to the user); `unassignedCount` is
  coverage (a count, never a cost); `objective` is the solver's comparison
  scalar in synthetic units where every unassigned order is priced above any
  possible serving cost. Judge "is the plan better?" by: fewer unassigned
  wins, ties break on lower `cost.total` (equivalently: lower
  `objective.total`). Unassigning a stop always WORSENS the plan even when
  `cost` falls — never present dropping stops as savings, and never quote
  `objective` numbers as money. (Distinct from the vehicle cost INPUTS on
  `routing24_upsert_vehicles`, where `cost.duration`/`cost.overtime` are per-hour
  rates.)
- **Explaining a cost = walking `cost.components`.** The lines sum to
  `cost.total`, each named by `kind`; a route's cost from `routing24_route`
  adds the serving vehicle's effective `rate`/`quantity`/`unit` per line so
  the user can verify the arithmetic (`amount` stays authoritative when the
  product differs). `OptimizeStatus.costModel` is the pricing the solve
  actually used: when `priced` is false no cost is configured and
  `cost.total` reads roughly as travel distance in miles/km plus total route
  hours (driving + service + waiting) at the defaults of 1 per mile/km + 1
  per hour — quote its `note`.
  A rate in `defaultRates` (or a `defaultRate: true` line) is an ENGINE
  DEFAULT, blank in the app — call it a default, never a cost the user set.
- **Marginal costs are estimates, never sums.** `insertionQuotes` (from
  `routing24_unassigned` — the cheapest way to serve an unassigned order) and
  per-stop `marginalCost` (from `routing24_route` — the removal saving) are
  real money/seconds and safe to quote — but they hold for the CURRENT
  arrangement only: never add them up across orders, and re-read after any edit
  or re-optimize (`refresh_diagnostics` when `diagnosticsStale`). To act on a
  quote, replay it with `routing24_edit_move_stops` and its `before`/`after`
  anchor.
- **Units**: `max_distance` (vehicle constraint) and every returned distance
  use the plan's display unit (`distanceUnit`: km or mi); `cost.duration` and
  `cost.overtime` are per hour; `cost.load_distance` is per load unit per
  km/mi; all times are seconds-since-midnight.
- **Vehicle cost inputs: omit, don't zero.** An omitted cost field is 0. To
  minimize distance and time, omit `cost` entirely — do not send zeros: the
  optimizer then prices every vehicle at the defaults of 1 per mile/km plus 1
  per hour. Never write `cost: { distance: 0, duration: 0 }` to mean "no
  preference": an all-zero fleet has nothing to optimize, so the zeros are
  ignored (the same defaults apply) and the optimize result returns a
  `warnings` entry you must relay. Explicit zeros are for mixed fleets only,
  e.g. `{ id: "Bike", cost: { distance: 0, duration: 6 } }` next to
  `{ id: "Van", cost: { distance: 2, duration: 18, fixed: 40 } }` makes
  mileage free on the bike and priced on the van.
