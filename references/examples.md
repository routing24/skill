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
        { id: "S1", address: "STOP 1 ADDRESS", delivery: 1, service_duration_s: 300 },
        { id: "S2", address: "STOP 2 ADDRESS", delivery: 1, service_duration_s: 300 },
    ],
    vehicles: [
        { available_count: 2, capacity: 20, tw_early_s: 8 * 3600, tw_late_s: 18 * 3600 },
    ],
    // options: { time_limit_s: 30 },
});

// 5) Poll (~every 3s) until phase is "done" or "error". Relay progress meanwhile.
await __r24call('routing24_status');

// 6) Show routes on the map, then persist and get the plan link.
await __r24call('routing24_render');
await __r24call('routing24_save'); // -> { saved, planUrl }

// 7) Full solution: per-route ordered stops (resolved id/address/lat/lng,
//    ETAs, loads) + unserved list. Works now and on a plan reopened by URL.
await __r24call('routing24_solution');

// 8) Current plan URL (also returned by save).
await __r24call('routing24_plan_url');

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
