# iNat Together 2026 Event Finder

A radius-based search tool for finding in-person iNat Together 2026 events (September 18–28, 2026). Visitors enter a location or use their device's GPS, choose a search radius, and see approved events within that radius, sorted by distance.

**Live tool:** add your GitHub Pages URL here, e.g. `https://[username].github.io/inat-together-2026-finder/`

Embedded on: add the iNat wiki page URL here

Current as of this README: **258 approved events** (last built September 6, 2026).

---

## How it works

This is a single self-contained HTML file — no server, no database, no API key. It works entirely in the browser. All event data (title, host, date, project link, and location) is embedded directly in the file as a JavaScript array (`EVENTS`), generated at build time.

The only live network calls the tool makes when someone uses it are:
- **Geocoding** a visitor's typed address, via [Nominatim/OpenStreetMap](https://nominatim.openstreetmap.org/)
- **Browser geolocation**, if they click "Use my location" (requires the embedding `<iframe>` to include `allow="geolocation"` — see [Embedding](#embedding) below)

It does **not** query Google Sheets or the iNaturalist API live — those are only used when the file is rebuilt.

This mirrors the architecture of the City Nature Challenge 2026 project lookup tool: build once, embed everything, rebuild when the data changes.

## Data sources (used to build the file, not at runtime)

- **Event list:** the iNat Together 2026 event submission Google Sheet, filtered to rows where **"Added to umbrella" = yes**. One project (Janet Wright's Graveline Bayou Coastal Preserve BioBlitz) is intentionally excluded despite being approved — the host asked for it not to be public; see the comment above the `EVENTS` array for details.
- **Event location:** each approved project's location requirement — its `place_id` (or, occasionally, a direct lat/lng circle), read from the iNaturalist API's `search_parameters`. This is the same data shown under **"Project Requirements"** on the project's page. For projects with a multi-place requirement, **all** places in the array are used (see Multi-location events below) — the compact top-level `place_id` field only reflects one of several and will silently miss the rest.
- **Place boundaries:** an iNaturalist Places CSV export (Alison's), matched by `place_id` to get a bounding box, generally reduced to a single center-point lat/lng per location. Places too new for the current export snapshot are resolved live via the iNaturalist API instead.
- **Network-wide API:** all iNaturalist Network node front-ends (inaturalist.ca, inaturalist.ala.org.au, panama.inaturalist.org, mexico.inaturalist.org, uk.inaturalist.org, etc.) share the same underlying `api.inaturalist.org` API, differentiated by `site_id` — so project lookups never need per-node API domains, regardless of which branded domain a project's URL uses.

## Special-case location handling

A single center-point isn't always a good stand-in for "where the project's location actually is." Three situations get different treatment:

### Multi-location events
Some projects require observations across several places (e.g. a citywide bioblitz spanning multiple parks, or a project with sub-projects like Goshen250's East/West split). These store an array of points (`locs: [[lat,lng], [lat,lng], ...]`), and a visitor matches if they're within range of the **closest** one.

### Country-scale events
A few projects (currently: Egypt, Russia, Chile) use an entire country as their location requirement. A single bounding-box center point badly under-serves these — a country's geometric center often falls somewhere almost nobody lives (empty desert, deep ocean, Siberia), so a normal radius search could miss nearly everyone actually searching from within the country. These events instead carry a `bboxes` field and match by **bounding-box containment**: if the visitor's location falls anywhere inside the country, it's a match regardless of the radius slider, and the result shows "Open to the whole country" instead of a distance. (Russia's bounding box also crosses the antimeridian — 180°/-180° longitude — which the containment check handles as a special case.)

If another country-wide project gets approved, give it the same treatment: check the resolved place's `place_type` (12 = country in iNaturalist's system) and bbox size, add a `bboxes` array, and it'll be picked up by the existing `matchEvent()` logic automatically.

### Irregular/concave place shapes
Some places aren't country-sized, but their *shape* still breaks simple centroid math — a thin coastline strip, a river corridor, anything that curves back on itself. For these, neither the bounding-box midpoint nor the true polygon-area centroid necessarily falls inside the actual shape (a known issue with concave/crescent-like geometry). One instance was found and fixed by hand: **Monterey Peninsula Intertidal Bioblitz**, whose place is a shoreline strip — its stored point was manually corrected to the closest real point *on* the shoreline instead of a centroid that landed outside it.

This isn't checked proactively for every project (verifying real polygon shape requires a heavier per-project API call, and isn't worth the cost across 250+ events by default). Per Alison, the standing approach is: fix these as they're spotted or reported, using the same "closest point on the actual shape" method, rather than auditing everything up front.

## Known imprecision (flagged, not fixed)

A handful of approved projects resolve to a place broader than where the event is actually happening — e.g. an entire county, city, or state, rather than the specific park or site described in the project. These are usually a project-setup issue on the host's end (their location requirement is set to a bigger place than intended) rather than a bug in this tool. Current flagged instances are listed in the comment block above the `EVENTS` array in the HTML file, along with the date each was noticed.

## The "widen your radius" guidance

Because event locations are single points rather than real boundaries, a nearby event can sometimes fall just outside a visitor's chosen radius even though the actual event is closer than the map suggests. When a search returns no results, the tool now shows a note explaining this and suggesting a wider radius, plus a callout that country-scale events (see above) won't be affected by radius changes since they already match regardless of distance.

## Keeping locations current: the `obs` field

Every event stores an `obs` field — iNaturalist's `observation_requirements_updated_at` timestamp for that project, captured at the time its location was last resolved. On a refresh, this timestamp can be checked in bulk against the live API to identify which projects have had their location requirement edited since the last build — only those need a full place re-resolution, which is far cheaper than resolving all 250+ events every time.

**Caveat:** this field does not reliably update on every location edit. At least one case (No Bones Bioblitz in Monterey Bay) had its place list changed with no corresponding change to `observation_requirements_updated_at`, even on a fresh live fetch. Treat the timestamp check as a way to catch *most* changes cheaply, not a guarantee — take host- or Alison-reported location updates seriously even when the timestamp check shows nothing changed.

## Updating the tool

There's no live pipeline — updating means regenerating the file and re-uploading it.

1. Share the current event submission sheet (a CSV export is more reliable than a live Google Sheet link once the sheet gets large — the "read the sheet" tool has been known to silently truncate partway through on this sheet's size).
2. New approved projects get resolved (place lookup via the API, then matched against the current Places CSV export, or live API for places too new for that export) and added to the `EVENTS` array.
3. Existing events get their `obs` timestamps checked for changes (see above); any that changed get their location re-resolved.
4. The regenerated `index.html` replaces the old one in this repo (Add file → Upload files → same filename → Commit).
5. GitHub Pages picks up the change automatically within a minute or so — no need to touch the wiki embed.

The iNaturalist Places CSV export refreshes weekly (Fridays); refresh timing generally aligns with that cadence, with a final refresh planned before the wiki page is promoted to the broader iNat community.

## Embedding

Embed via `<iframe>` on the wiki page. To make the "Use my location" button work inside the iframe, the embed must include the `allow="geolocation"` attribute:

```html
<iframe src="https://[username].github.io/inat-together-2026-finder/" width="100%" height="900" style="border:none;" allow="geolocation"></iframe>
```

Some wiki/CMS editors strip custom iframe attributes on save — if geolocation stops working after an edit to the embed code, check that this attribute survived.

## Accessibility

- The location input has an `aria-label`.
- The results container uses `aria-live="polite"` so screen readers announce new results as they load.
- Example location chips are real `<button>` elements (keyboard-focusable, activate on Enter/Space), not styled `<span>`s.
- Error states (unsupported browser, geolocation denied, geocoding failure) render into the results area rather than using `alert()`.

## Known limitations

- **Point precision, not exact addresses.** A project's location requirement is often a named iNaturalist Place (a park, sometimes a larger region like a county or country), not a street address. See "Special-case location handling" and "Known imprecision" above.
- **Static snapshot.** The event list only reflects what was approved as of the last rebuild — it does not update automatically as new events are approved.
- **No visual map.** This is a radius/list search, not a map view.

## Files

- `index.html` — the tool itself (all event data, styling, and logic in one file)
