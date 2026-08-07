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
  own browser tab via WebMCP tools (routing24_geocode, routing24_optimize,
  routing24_status, routing24_edit, routing24_save, ...) registered on
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
  version: "1.1.1"
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
handler errors). The optimizer runs client-side in WASM, so `routing24_optimize`
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
  (registered or anonymous). **No sign-in is required** — the full flow (geocode,
  optimize, render, save, share) works anonymously. Do not ask the user to log in.
  Signing in changes **where** the plan is stored (see the plan-link note under
  *Notes & pitfalls*) and which **paid constraints** a solve may keep (see
  *Plans & paid features*) — neither is a reason to stop or to ask for a login.

## Plans & paid features

Most of `OptimizeInput` is free. The fields below need a paid Routing24 plan;
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
**5 optimizations**. After that `routing24_optimize`
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
   Expect the `routing24_*` names (`routing24_optimize` etc.). If `null` or
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
   saved plan lives and what you tell the user about the plan link (see step 10).

5. **Parse the request** into the `routing24_optimize` input shape (see the
   **API contract** for the full definition). Times are seconds-since-midnight;
   all fields except the addresses are optional. The schema requires **≥1 stop**
   and **≥1 vehicle**; with a single stop there is nothing to sequence, so
   expect real requests to carry ≥2 stops. Bad input makes the call reject with
   a message naming the offending fields — relay it to the user.

6. **Geocode + confirm.** Run
   `await __r24call('routing24_geocode', { addresses: [<depot + every stop address>] })`.
   - Show the user any row with `ok:false` (not found) and ask them to fix the
     address. Also surface `matched` values that look wrong and confirm.
   - Once confirmed, use the returned `lat`/`lng` for each entity (pass them in
     the `routing24_optimize` input so you don't re-geocode). `routing24_optimize`
     will geocode any address you leave without coordinates, but doing it here lets
     the user confirm ambiguous matches first.

7. **Start optimization.**
   `await __r24call('routing24_optimize', { depot, stops, vehicles })`. It returns
   quickly with `{ started: true, planUuid }`.
   - If the result carries `paidFeatures`, the plan uses constraints outside the
     user's subscription. `stripped` is empty while the free allowance lasts;
     once it is non-empty the solve **ignored** those constraints — carry that
     into step 10 (see *Plans & paid features*).

8. **Poll progress.** Every ~3 seconds run `await __r24call('routing24_status')`
   until `phase === 'done'` or `phase === 'error'`.
   - Relay `progress` (0–1) and, once available, `routes` / `distance` / `feasible`.
   - Small jobs finish in seconds; large ones can take minutes (keep the tab
     active). If `phase === 'error'`, report `error` and stop.
   - To abort on the user's request: `await __r24call('routing24_cancel')`.

9. **Show + save.** Run `await __r24call('routing24_render')` (brings the routes
   onto the map), then `await __r24call('routing24_save')` to persist and get
   `{ saved, planUrl }`.
   - To *show the user the map*, take a screenshot of the tab after render —
     the tools return JSON, not images.
   - To read back the **full solution** (per-route ordered stops with resolved
     addresses/coords, ETAs and loads, plus the unassigned list), call
     `await __r24call('routing24_solution')`. It works both now and later on a
     plan reopened by URL, so you never have to reconstruct routes from
     `routing24_status` rollups.

10. **Report** to the user:
   - Number of routes, total distance (with unit), total duration, and any
     `unassignedCount` (stops that couldn't be served — mention them).
   - The optimized **stop sequence per route** (pull it from
     `routing24_solution` when the user wants the itinerary, not just totals).
   - Whether the solution is `feasible`.
   - **Any `paidFeatures.stripped` from step 7** — name the constraints the
     solve ignored and that upgrading restores them. A plan that quietly drops
     the user's tags, breaks or transfers is worse than no plan.
   - The **plan link** (`planUrl`) they can open, and note the map on screen shows
     the routes. If the user is **anonymous** (from step 4), add that this link
     opens the plan **only on this computer** and may be deleted later — it is not a
     durable share link.

## Editing routes manually

Once a plan has a solution (fresh solve or a plan opened by URL), you can edit
it in place — the same manual-editing engine the UI's drag & drop uses:

1. **Read the current arrangement**: `await __r24call('routing24_solution')`.
   Note each route's `slot` (0-based, empty routes included) and each stop's
   `id` — edit ops address routes by slot and stops by id.
2. **Compose ONE batch** of ops in `routing24_edit` and apply it atomically:
   move/re-plan stops (`move_sites` — anchor with `before`/`after` a planned
   stop id, or `to_route` + `placement: 'append' | 'best'`), `unassign_sites`,
   `set_route_vehicle`, `create_route`, `remove_routes` (empty routes only),
   `split_route`, `merge_routes`, `mark_user_assigned` / `clear_user_assigned`.
   Later ops see earlier ops' effects; the first rejection rolls back the whole
   batch (`rejection.code` says why — `stale_revision` means the plan changed
   under you: re-read `routing24_solution` and rebuild the batch).
3. **Judge the result, don't guess**: edits are never rejected for breaking
   constraints — problems are REPORTED instead. Check `state.feasible`,
   `state.problemsCount`, per-route `problems`, and `userAssignedReports`
   (per manually-placed stop: `intrinsic` = would violate anywhere,
   `introduced` = caused by this placement). If a placement looks bad, fix it
   with another batch or `routing24_undo`.
   **Always check `state.driftFromOptimized`** — how far the plan has moved
   from the last full solve (the user's own manual edits count too). When
   `severity` is `degraded` or `severe` you MUST tell the user, quoting
   `summary` — it leads with the COST change, the number the user pays
   (e.g. "cost +2348 (+12.3%), distance +12.4%, 1 new problem vs the last
   optimization") — and offer a fix: `routing24_optimize_route` for a single
   rough route, or a fresh `routing24_optimize`. Report `improved` drift
   too — it reinforces that the edits helped.
4. **Tidy up**: `routing24_optimize_route` re-sequences one rough route
   (sub-second, synchronous); `remove_routes` deletes emptied routes —
   slots COMPACT afterwards, so always take fresh slots from the returned
   `state.routes`.
5. **Explain unassigned stops**: `routing24_solution` with
   `{ refresh_diagnostics: true }` returns `unassignedDiagnostics` — a prose
   summary + per-site category/explanation/blockers/levers you can relay.
6. **Save** with `routing24_save` when the user is happy.

While you edit, the app locks behind a full-screen **"Agent controlled"**
overlay (blue dashed frame + a floating pane at the bottom) so the user can't
race you; they resume via its **Take control** button. `routing24_undo` /
`routing24_redo` walk the SAME history as the user's own edits — never undo
speculatively; prefer a compensating `routing24_edit` batch.

## Reference files

Load these only as the task calls for them (progressive disclosure):

- **API contract** (per-tool signatures + field definitions): [references/api.md](references/api.md)
- **Machine-readable JSON Schema** (OpenAPI 3.1 / JSON Schema 2020-12): [references/schema.json](references/schema.json)
- **Ready-to-eval WebMCP call snippets** (`javascript_tool`-ready): [references/examples.md](references/examples.md)
- **Always-current contract** — fetch `https://routing24.com/llms.txt` if a validation error
  suggests the bundled reference is behind the deployed API.

## Version & keeping current

- This skill is **version 1.1.1**. Its bundled reference
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
- `routing24_optimize` starts a **new plan** each time. When the user is
  **anonymous**, the plan is stored **only in this browser on this computer** and
  may be deleted later, so the plan link opens only here — it is not a durable
  share link. Say this when you hand over the link. (Signing in before saving
  persists the plan to the user's account so the link also opens on their other
  devices — but sign-in is never required to plan, save, or share.)
- If `routing24_status` never leaves `matrix`/`solving`, the network (matrix
  service) or the solve may be slow — keep polling; only treat it as failed on
  `phase:'error'`.
- **Editing needs a solved plan** (`routing24_solution` reports
  `available:true`) and refuses while a full solve runs
  (`errorCode:"solve_in_progress"`). Route `slot`s are 0-based and COMPACT
  after `remove_routes` — always re-read them from the returned
  `state.routes` or a fresh `routing24_solution`.
- **Two failure channels on the editing tools**: `rejection.code` = the
  ENGINE refused the batch (structural: unknown ids, bad anchors,
  `stale_revision`, …); `errorCode` = the tool refused or failed around the
  engine (`solve_in_progress`, `no_solution`, `slot_out_of_range`, `route_too_small`, `optimize_running`, `nothing_to_undo`, `nothing_to_redo`, `session_error`).
  `session_error` means the editing
  session itself failed and the app resynced — re-read `routing24_solution`
  and retry ONCE.
- **Edits are never rejected for violating constraints** — time windows,
  capacity etc. are scored and reported (`state.problemsCount`, per-route
  `problems`, `userAssignedReports`), exactly like the UI's manual drag &
  drop. Rejections are structural only (`rejection.code`: unknown ids, bad
  anchors, non-empty route removal, `stale_revision`, …).
- While you edit, the app locks behind a full-screen **"Agent controlled"**
  overlay until the user clicks its "Take control" button — tell the user to
  take control from there when you hand the plan back. The user may also edit
  concurrently before your first edit: a `stale_revision` rejection means
  re-read `routing24_solution` and rebuild your batch.
- `routing24_undo`/`routing24_redo` walk the same history as the user's own
  manual edits — never undo blindly.
- **`driftFromOptimized` is your metrics conscience.** It compares the CURRENT
  plan to the last full solve (baseline frozen at solve completion; only a new
  solve resets it) and covers ALL changes since — including edits the user made
  by hand while you were away. On every `routing24_solution`/`routing24_status`
  read and in every edit result: `severity` `degraded`/`severe` MUST be
  relayed to the user with the ready-made `summary`, plus an offer to
  re-optimize. Absent drift = the plan predates the baseline feature or has no
  completed solve — nothing to compare against.
- **Money, coverage and the objective are three separate registers — never
  mix them.** `cost` is money (report it to the user); `unassignedCount` is
  coverage (a count, never a cost); `objective` is the solver's comparison
  scalar in synthetic units where every unassigned order is priced above any
  possible serving cost. Judge "is the plan better?" by: fewer unassigned
  wins, ties break on lower `cost.total` (equivalently: lower
  `objective.total`). Unassigning a stop always WORSENS the plan even when
  `cost` falls — never present dropping stops as savings, and never quote
  `objective` numbers as money. (Distinct from the vehicle cost INPUTS on
  `routing24_optimize`, where `cost.duration`/`cost.overtime` are per-hour
  rates.)
- **Marginal costs are estimates, never sums.** `insertionQuotes` (cheapest
  way to serve an unassigned order) and per-stop `marginalCost` (removal
  saving) are real money/seconds and safe to quote — but they hold for the
  CURRENT arrangement only: never add them up across orders, and re-read
  after any edit or re-optimize (`refresh_diagnostics` when
  `diagnosticsStale`). To act on a quote, replay it as `move_sites` with
  its `before`/`after` anchor.
- **Units**: `max_distance` (vehicle constraint) and every returned distance
  use the plan's display unit (`distanceUnit`: km or mi); `cost.duration` and
  `cost.overtime` are per hour; `cost.load_distance` is per load unit per
  km/mi; all times are seconds-since-midnight.
- **Vehicle cost inputs: omit, don't zero.** An omitted cost field is 0. When
  the user gives no rates at all, send no `cost` objects — the optimizer then
  minimizes plain travel distance. Never write `cost: { distance: 0,
  duration: 0 }` to mean "no preference": an all-zero fleet has nothing to
  optimize, so the zeros are ignored (distance is minimized) and the optimize
  result returns a `warnings` entry you must relay. Explicit zeros are for
  mixed fleets only, e.g. `{ id: "Bike", cost: { distance: 0, duration: 6 } }`
  next to `{ id: "Van", cost: { distance: 2, duration: 18, fixed: 40 } }`
  makes mileage free on the bike and priced on the van.
