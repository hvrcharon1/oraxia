# Oracle APEX Real-Time GPS Tracking — Supabase + Leaflet — AI Skill Definition

> Credit: the architecture pattern in this skill — pairing Oracle APEX with Supabase's Realtime WebSocket channel and Leaflet.js to add live GPS tracking without a custom Node.js server — is based on **Md.Kamrul Fardaus**'s article ["Real-Time GPS Tracking in Oracle APEX Using Supabase and Leaflet"](https://medium.com/@kamrulfardaus/real-time-gps-tracking-in-oracle-apex-using-supabase-and-leaflet-106bcf24c6aa) on Medium (Jul 4, 2026). See the [Credits](#credits) section at the end for details.

## Purpose
This skill teaches AI assistants how to add live, multi-client GPS/location tracking to an Oracle APEX application. Oracle APEX pages are excellent for CRUD-heavy enterprise apps but have no native way to push an event to a connected browser the moment new data arrives — every region either polls or waits for a manual refresh. This skill combines three purpose-built pieces instead of reaching for a bespoke Node.js/WebSocket server: Oracle stays the system of record for reference data and images, a Supabase Postgres table + Realtime channel carries only the ephemeral live-position stream, and Leaflet.js renders the map in the browser. Reach for this skill whenever someone wants a live map, a delivery/asset tracker, or any "watch it move in real time" feature bolted onto an existing APEX app.

## When To Use This Pattern
- A live map that should update the instant a tracked entity's position changes, without the user refreshing or a Dynamic Action polling on a timer
- Multiple people watching the same map need to see the same moving markers at the same time
- The existing application is already Oracle APEX and rewriting it around a Node.js/socket.io backend isn't worth the migration cost
- The routing need is genuine road-network distance/ETA, not just a straight line between two points

## Architecture At A Glance

```
Oracle APEX application (browser)
 ├── Leaflet.js          ────────► OpenStreetMap tiles (base map)
 ├── Supabase JS client  ────────► Supabase Realtime (WebSocket: live positions)
 ├── ORDS REST endpoint  ────────► Oracle BLOB columns (marker images, reference data)
 └── OSRM public API     ────────► road-network routing (shortest driving path)
```

Two databases, two jobs: Oracle keeps doing what it already does well — durable storage, PL/SQL, ORDS-exposed reference data — and Supabase carries only the one thing Oracle wasn't built to do, pushing events to a browser over a WebSocket. Leaflet is the glue in between. No message broker, no separate application server to deploy or patch.

## The Marker Registry Pattern (Core Technique)

The single most common bug in a "live map" build: every incoming position update creates a *new* marker while the previous one for that same entity is left behind, so the map fills up with duplicate ghost markers instead of one marker that moves. Fix it by keeping an in-memory registry keyed by entity ID, and always remove the previous marker layer before drawing the new one.

```javascript
// One entry per tracked entity: { <id>: { marker, lat, lng } }
const trackedEntities = {};

function updateEntityPosition(entityId, lat, lng) {
    const existing = trackedEntities[entityId];

    if (!existing) {
        const marker = L.marker([lat, lng], { icon: vehicleIcon() }).addTo(map);
        trackedEntities[entityId] = { marker, lat, lng };
        return;
    }

    // Remove the previous layer before adding the new one — this is the fix
    map.removeLayer(existing.marker);

    const marker = L.marker([lat, lng], { icon: vehicleIcon() }).addTo(map);
    trackedEntities[entityId] = { marker, lat, lng };
}
```

If a smoother look is wanted, swap the plain `L.marker` creation for an animated-move plugin between the old and new coordinates — the registry-and-remove logic underneath stays the same regardless of which visual layer draws the movement.

## Wiring Supabase Realtime To The Registry

Once the registry exists, subscribing to live position changes is a thin layer on top of it:

```javascript
supabase
    .channel('vehicle-location-changes')
    .on(
        'postgres_changes',
        { event: 'UPDATE', schema: 'public', table: 'vehicle_location' },
        (payload) => updateEntityPosition(payload.new.id, payload.new.lat, payload.new.lng)
    )
    .subscribe();
```

**Gotcha — silent write failures:** Supabase enables Row Level Security by default, which blocks anonymous inserts/updates unless an explicit `anon` policy grants them. A tracker page that "looks like it's sending updates" but never moves the map is almost always this — the write is being rejected server-side with no error surfaced to the page. Check the Network tab or Supabase's own logs the first time a tracker page is tested, before assuming the bug is in the JavaScript.

## Capturing Live Position (Tracker Page)

The write side is a browser geolocation watch that upserts into the Supabase table on every fix:

```javascript
navigator.geolocation.watchPosition(
    async (position) => {
        await supabase
            .from('vehicle_location')
            .update({
                lat: position.coords.latitude,
                lng: position.coords.longitude,
                updated_at: new Date().toISOString(),
            })
            .eq('user_id', currentUserId);
    },
    (error) => console.error('Geolocation error:', error),
    { enableHighAccuracy: true }
);
```

- APEX has had native Progressive Web App (PWA) support since 21.2 — enable it under **Application Properties** so the tracker page can be installed on a phone home screen like a native app.
- Call `navigator.wakeLock.request('screen')` when tracking starts so the screen doesn't sleep mid-session and silently stop the position stream.
- **Honest limitation:** even an installed PWA cannot reliably keep sending positions once the app is fully backgrounded — mobile browsers throttle background JavaScript execution regardless of PWA install state. If continuous background tracking is a hard requirement (not just "usually works"), the production-grade path is wrapping the APEX URL in a native shell (e.g. Capacitor) with a dedicated background-geolocation plugin. Say this plainly to whoever is asking for "always-on tracking" rather than letting the PWA version quietly under-deliver.

## Road-Network Routing With OSRM

For an actual driving route (not a straight line), the public OSRM API needs no API key:

```javascript
async function getDrivingRoute(origin, destination) {
    const coordString = `${origin.lng},${origin.lat};${destination.lng},${destination.lat}`;
    const url = `https://router.project-osrm.org/route/v1/driving/${coordString}?overview=full&geometries=geojson`;

    const { routes } = await (await fetch(url)).json();
    return routes[0]; // routes[0].geometry.coordinates, .distance, .duration
}
```

**Gotcha — coordinate order:** OSRM returns coordinates as `[lng, lat]`; Leaflet expects `[lat, lng]`. Flip every pair before handing them to a Leaflet polyline, or the route will draw in the wrong place (or throw if the values are out of range):

```javascript
const leafletCoords = route.geometry.coordinates.map(([lng, lat]) => [lat, lng]);
L.polyline(leafletCoords).addTo(map);
```

**UX tip:** when showing both a straight-line distance and the actual route distance, put them in a small floating info panel rather than as stacked tooltips on the map itself — two overlapping tooltip labels on a moving marker are hard to read at a glance.

## Anti-Patterns To Avoid

| Anti-pattern | Why it's wrong | Do this instead |
|---|---|---|
| Creating a new `L.marker()` on every position update | Leaves the previous marker on the map — duplicates accumulate instead of one marker moving | Keep an ID-keyed registry; call `map.removeLayer()` on the old marker before adding the new one |
| Assuming a Supabase `insert`/`update` from the browser succeeded because no exception was thrown | Row Level Security blocks anonymous writes by default and fails silently from the client's perspective | Add explicit `anon` policies for the exact operations needed, and check the Network tab / Supabase logs on first test |
| Feeding OSRM's `[lng, lat]` output straight into a Leaflet polyline | Leaflet expects `[lat, lng]` — the route draws in the wrong place | Map over the coordinate array and swap each pair before drawing |
| Presenting an installed APEX PWA as fully background-capable location tracking | Mobile browsers throttle backgrounded JS regardless of PWA install state | Say so explicitly; use a Capacitor (or similar) native shell with a background-geolocation plugin when true background tracking is required |
| Pushing high-frequency live position writes into Oracle directly | Oracle wasn't built to fan out realtime browser push the way a WebSocket-native service is; you end up rebuilding polling to fake "realtime" | Keep Oracle as the system of record for durable/reference data; use a WebSocket-native store (e.g. Supabase Realtime) only for the ephemeral live-position stream |

## Quick Reference

| Need | Pattern |
|---|---|
| Move one marker instead of stacking duplicates | ID-keyed registry + `map.removeLayer()` before redraw |
| Get notified the instant a row changes | `supabase.channel(...).on('postgres_changes', { event: 'UPDATE', ... })` |
| Push a browser's GPS fix to the backend | `navigator.geolocation.watchPosition(...)` → Supabase `update`/`upsert` |
| Keep the tracker screen awake | `navigator.wakeLock.request('screen')` |
| Free road-network routing, no API key | `https://router.project-osrm.org/route/v1/driving/{lng1},{lat1};{lng2},{lat2}?overview=full&geometries=geojson` |
| Fix OSRM → Leaflet coordinate mismatch | `.map(([lng, lat]) => [lat, lng])` |

---

## Credits

The overall architecture — Oracle APEX as the enterprise frontend and system of record, Supabase Realtime carrying only the live-position WebSocket stream, and Leaflet.js rendering the map — along with the marker-registry fix, the Supabase Row Level Security write gotcha, the APEX PWA + `wakeLock` tracker pattern (and its honest background-tracking limitation), and the OSRM routing/coordinate-order pattern, are all based on **Md.Kamrul Fardaus**'s article ["Real-Time GPS Tracking in Oracle APEX Using Supabase and Leaflet"](https://medium.com/@kamrulfardaus/real-time-gps-tracking-in-oracle-apex-using-supabase-and-leaflet-106bcf24c6aa), published on Medium, July 4, 2026.

All code examples above were independently written for this skill and are not reproduced verbatim from the original article. Readers wanting the original walkthrough, embedded demo video, and author commentary should consult the source article directly.
