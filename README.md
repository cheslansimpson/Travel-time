# Travel Time

A single-file Progressive Web App that builds weather-aware packing lists and pre-fills travel booking sites with the right parameters in one tap. Designed for a federal contractor traveling carry-on only out of PSC, with built-in opinions about clothing counts, government rate codes, and Delta/Marriott/Hilton loyalty.

---

## Quick start

This is a single HTML file. No build step, no dependencies to install, no package manager.

### On desktop

Open `index.html` in any modern browser. Everything works because desktop browsers are lenient about `file://` origin requests to CORS-enabled APIs.

### On mobile (required for the app to function)

Mobile browsers (Chrome on Android, Safari on iOS) block all network calls from `file://` URLs. The file must be served over HTTPS.

**GitHub Pages path** (this repo):

1. Push `index.html` to the repo root
2. Settings → Pages → Source: Deploy from a branch → `main` / `(root)` → Save
3. Wait about 60 seconds for the URL to appear
4. On phone: open URL in Chrome → three-dot menu → **Install app**

Alternatives that work the same way: Netlify, Cloudflare Pages, Vercel. All free.

---

## What it does

Per trip, you enter: destination city, start/end dates, trip type (business / personal / leisure, multi-select). The app then:

- Geocodes the destination via Open-Meteo
- Pulls weather forecast for the trip window (or climate proxy if more than 16 days out)
- Builds a categorized packing list scaled to trip length, type, and weather
- Estimates carry-on volume and warns if over ~38L
- Generates pre-filled deep links to Delta, Marriott (with gov rate code), Hilton (with gov rate code), Hertz, Uber, plus Google Flights and Kayak as comparison
- Persists trips, packing checkmarks, customizations, and confirmation numbers in `localStorage`

Packing list is fully editable per trip. Customizations can be saved as a template that auto-applies to every new trip.

---

## Profile baked in

Established during the initial interview. Hardcoded as the `PROFILE` constant near the top of the script block. Change them there when assumptions shift.

| Setting | Value | Why it matters |
|---|---|---|
| Home airport | `PSC` | Pre-fills origin in all Delta deep links |
| Bag preference | Carry-on only, always | Drives volume warnings and item count caps |
| Gov rate eligible | `true` | Adds `clusterCode=GOV` to Marriott and `specialRates=GVT` to Hilton when trip type is business |
| Outfits per day | 1 | No evening change; affects shirt and pants quantities |
| Workout gear | `false` | Skipped entirely from packing list |
| Underwear / socks buffer | `+2` over trip length | Always pack two extra |
| Laundry threshold | 5 days | For trips at or above this, clothing counts cap at 6 days and a mid-trip laundry warning fires |
| Tech kit | Laptop + noise-canceling headphones | Always included; no tablet or e-reader |
| Loyalty programs | Marriott Bonvoy, Hilton Honors, Delta SkyMiles | Affects which booking sites the app prioritizes |

---

## How weather drives packing

Thresholds live in `buildPackingList()`:

| Condition | Triggers |
|---|---|
| Avg low < 50°F | Light insulated jacket |
| Avg low < 32°F | Heavy winter coat, beanie, gloves, scarf |
| Min low < 20°F | Thermal long underwear |
| Any day ≥ 50% precipitation probability | Packable rain shell |
| Freezing + rain expected | Waterproof boots |
| Avg high > 80°F | Lightweight emphasis, no specific addition |
| Avg high > 90°F | Shorts replace jeans, sun hat added, sunscreen in toiletries |

For trips beyond 16 days out, the forecast API is replaced with same-dates-last-year from the Open-Meteo historical archive as a climate proxy. The UI flags this with a `CLIMATE EST` pill so estimates aren't mistaken for forecasts.

International destinations (anything not `United States`, `USA`, or `US`) automatically add a passport reminder and universal power adapter.

---

## Per-trip customizations

Every item in the packing list can be edited, deleted, or supplemented per trip without changing any code.

**Edit an existing item.** Tap the `EDIT` button at the right edge of any item. A panel opens with a quantity stepper, a note field, and three actions:

- `Delete` removes the item from this trip
- `Cancel` closes the panel without saving
- `Save` persists the change

Setting quantity to 0 and saving is equivalent to deleting.

**Add a new item.** Each category has a `+ Add item to [Category]` button at the bottom. Name, quantity, and note. Custom items display with a `· CUSTOM` tag so they're visually distinct from rule-generated ones.

**Reset.** A `Reset all customizations` link at the bottom of the packing list restores the rule-generated defaults for the current trip. Checkmarks survive the reset.

**Where it lives.** Per-trip customizations are stored on the trip object under `customizations`:

```javascript
trip.customizations = {
  overrides: {
    "Toiletries::Razor + travel shave cream": { deleted: true },
    "Tops::Polos / casual shirts": { qty: 4 }
  },
  custom: [
    {
      id: "c_xyz123",
      category: "Toiletries",
      name: "Electric shaver + charger",
      qty: 1,
      note: "keep in carry-on"
    }
  ]
}
```

Customizations on one trip never affect any other trip.

---

## Default template

If the same customizations recur trip after trip (the canonical example: swap razor for electric shaver), save them as a template so they auto-apply to every new trip.

**To save a template.** Customize a trip the way you want it, then scroll past the `Reset all customizations` link to the `Default template` section. Tap `Save as template`. A confirmation shows what's being saved (X overrides + Y custom items). Confirm.

**To clear a template.** Tap `Clear template` in the same section. Existing trips are not affected.

**Status indicator.** The template section shows whether a template is currently set. Active state shows count and save date.

**Important behaviors:**

- Templates apply only to **new** trips created after the template was saved. Existing trips are never modified retroactively.
- Editing a trip after saving a template does **not** update the template. To update, customize a trip the new way and re-save.
- Custom items get fresh internal IDs when the template applies to a new trip, so per-trip edits never bleed across trips.
- Templates live in `localStorage` and are therefore per-device. Installing the PWA on a second device requires setting the template there too.

**Storage shape** under localStorage key `travelTime_template_v1`:

```javascript
{
  overrides: { ... },           // same shape as trip.customizations.overrides
  custom: [                      // same shape as trip.customizations.custom, but no IDs
    { category, name, qty, note }
  ],
  savedAt: "2026-05-21T12:34:56.789Z"
}
```

---

## Tech stack

- **No framework.** Vanilla HTML, CSS, JavaScript.
- **Single file.** Markup, styles, logic, manifest, and SVG icon all live in `index.html`.
- **Persistence.** `localStorage` under two keys: `travelTime_v1` for trips, `travelTime_template_v1` for the default template.
- **APIs.** Open-Meteo geocoding (`geocoding-api.open-meteo.com`) and forecast (`api.open-meteo.com`). Both free, no API key required, both CORS-enabled.
- **Fonts.** Fraunces (display serif), Manrope (body sans), JetBrains Mono (code and data), loaded from Google Fonts.
- **PWA install.** Inline manifest via Blob URL created at runtime. Inline SVG icon via data URI for both `<link rel="icon">` and `<link rel="apple-touch-icon">`.
- **No service worker.** Could be added later for explicit offline-first caching.

---

## Deep link reference

Useful for future debugging when one of these vendors changes their URL schema.

| Vendor | URL pattern | Notes |
|---|---|---|
| Delta | `delta.com/flight-search/book-a-flight?tripType=ROUND_TRIP&originCity=PSC&destinationCity=...&departureDate=...&returnDate=...` | Time-of-day filter not in URL; sort on results page |
| Marriott | `marriott.com/search/findHotels.mi?destinationAddress.destination=...&fromDate=MM/DD/YYYY&toDate=MM/DD/YYYY&clusterCode=GOV` | `clusterCode=GOV` only added for business trips |
| Hilton | `hilton.com/en/search/?query=...&arrivalDate=YYYY-MM-DD&departureDate=YYYY-MM-DD&specialRates=GVT` | `specialRates=GVT` only added for business trips |
| Hertz | `hertz.com/rentacar/reservation/?puDate=YYYY-MM-DD&doDate=YYYY-MM-DD` | Delta's rental partner, earns SkyMiles |
| Uber | `m.uber.com/ul/?action=setPickup&pickup=my_location&dropoff[formatted_address]=...` | Pickup defaults to current location |
| Google Flights | Standard query URL | Always included as a comparison backup |
| Kayak | `kayak.com/cars/CITY/YYYY-MM-DD/YYYY-MM-DD` | Always included as a rental car comparison |

---

## Known limitations

Intentional, not bugs:

- **No live prices.** Flight, hotel, and car prices require paid airline/hotel APIs and a backend. This app is client-side only.
- **Delta URL schema is undocumented.** Delta could change parameters at any time. The Google Flights backup link insulates against this.
- **Time-of-day flight filter is manual.** Delta has no public URL param for "early morning only." Sort on results page.
- **Government rate at check-in.** Pre-fill of `GOV` / `GVT` rate codes still requires valid federal ID at the front desk. Per-diem caps should be cross-checked against GSA and DoD travel sites.
- **Weather beyond 16 days is an estimate.** Same dates from previous year, close enough for packing decisions but won't catch anomalies.
- **No cloud sync.** Trips and template both live in browser localStorage. Switching phones means starting fresh, including resetting the template, until export/import is added.
- **Carry-on volume estimate is heuristic.** Based on item count and average sizes. Real volume varies with packing technique and brand.
- **Mobile fetch from `file://` blocked.** Hosting required, see Quick Start above. The app shows a prominent banner if it detects this state.

---

## Modifying defaults in code

Per-trip edits and templates handle most adjustments without code changes. For changes to the rules themselves, four locations in `index.html` cover the rest:

| What to change | Where |
|---|---|
| Profile defaults (airport, loyalty, buffers, laundry threshold) | `PROFILE` constant near the top of the script block |
| Weather thresholds and conditional items | `buildPackingList()` function |
| Per-item volume estimates (if carry-on warning misfires) | `estimateVolume()` function |
| Colors, fonts, spacing | CSS variables at the top of `<style>` |

---

## Possible additions

Reasonable next steps, not implemented:

- **JSON export/import for trip and template backup.** Solves the "switching devices" problem without a backend.
- **Cloud sync.** Firebase, Supabase, or Cloudflare Workers + KV. Currently each device needs its own template and trip list.
- **Multiple named templates.** "Business defaults", "Beach vacation", "Cold weather" — selected per trip rather than one auto-applied default.
- **Per-diem lookup for federal travel.** GSA CONUS rates and DoD OCONUS.
- **Multi-destination trips.** Current model assumes a single destination.
- **Conference or event-specific dress code overrides.**
- **Service worker for explicit offline-first caching.**
- **Real flight pricing via Duffel or Amadeus.** Requires backend and paid API.
- **Airport code lookup** so destination resolves to nearest airport, not city.

---

## Repository structure

```
.
├── index.html      # The entire app
└── README.md       # This file
```

No `node_modules`, no build artifacts, no config files.

---

## License

Personal project. Not licensed for redistribution.
