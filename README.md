# iNat Together 2026 Event Finder

A radius-based search tool for finding in-person iNat Together 2026 events (September 18–28, 2026). Visitors enter a location or use their device's GPS, and the tool shows approved events within an adjustable radius, sorted by distance.

**Live tool:** add your GitHub Pages URL here once published, e.g. `https://[username].github.io/inat-together-2026-finder/`

Embedded on: add the iNat wiki page URL here

---

## How it works

This is a single self-contained HTML file — no server, no database, no API key. It works entirely in the browser. All event data (title, host, date, project link, and location) is embedded directly in the file as a JavaScript array, generated at build time.

The only live network call the tool makes is geocoding a visitor's typed address via [Nominatim/OpenStreetMap](https://nominatim.openstreetmap.org/). It does **not** query Google Sheets or the iNaturalist API live — those are only used when the file is rebuilt.

This mirrors the architecture of the [City Nature Challenge 2026 project lookup tool](#), which uses the same build-once, embed-everything approach.

## Data sources (used to build the file, not at runtime)

- **Event list:** the iNat Together 2026 event submission Google Sheet, filtered to rows where **"Added to umbrella" = yes**
- **Event location:** each approved project's location requirement (its `place_id`, or occasionally a direct lat/lng), read from the iNaturalist API — the same data shown under **"Project Requirements"** on the project page
- **Place boundaries:** Alison's iNaturalist Places CSV export, matched by `place_id` to get a bounding box, which is centered to a single lat/lng per event
- **Note:** all iNaturalist Network node front-ends (inaturalist.ca, inaturalist.ala.org.au, panama.inaturalist.org, mexico.inaturalist.org, etc.) share the same underlying `api.inaturalist.org` API, differentiated by `site_id` — so project lookups don't need per-node API domains

## Updating the tool

There's no live pipeline — updating means regenerating the file and re-uploading it.

1. Share the current event submission sheet.
2. New approved projects get resolved (place lookup, then matched against a current Places CSV export) and added to the `EVENTS` array in the file.
3. The regenerated `index.html` replaces the old one in this repo (Add file → Upload files → same filename → Commit).
4. GitHub Pages picks up the change automatically within a minute or so — no need to touch the wiki embed.

The iNaturalist Places CSV export refreshes weekly (Fridays), so refresh timing is generally tied to that cadence. If a project references a Place created very recently (after the last export), it's resolved individually via a live API lookup instead of falling through.

## Known limitations

- **Place-level precision, not pin-level.** A project's location requirement is often a named iNaturalist Place (e.g. a park, or sometimes a larger region like a county or municipality), not an exact address. The tool uses the center point of that place's bounding box. For small, well-defined places (a specific park) this is accurate; for large or loosely-defined places, the point can be a rough regional center rather than the actual event site. Flagged instances are noted in the HTML file's comments as they come up.
- **Static snapshot.** The event list only reflects what was approved as of the last rebuild — it does not update automatically as new events are approved.
- **No visual map.** This is a radius/list search, not a map view.

## Files

- `index.html` — the tool itself
