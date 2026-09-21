# Furnished Finder — monthly rental search (furnishedfinder.com)

Next.js App Router site. Listing search results are **server-rendered into the RSC payload**
(`self.__next_f.push(...)` script tags). There is no XHR to intercept on first load, and the
GraphQL endpoint (`https://cfw-edge-auth.furnishedfinder.com/graphql`) rejects a plain
in-page `fetch()` (CORS/auth). Read the payload instead.

## URL patterns

- City page: `https://www.furnishedfinder.com/housing/us--ca--san-francisco` (region id
  `us--<state>--<city-slug>`). `/housing/San-Francisco-CA` redirects to the homepage — don't use it.
- Map-bounded search:
  `?map={"min":{"latitude":..,"longitude":..},"max":{"latitude":..,"longitude":..}}&searchType=REGIONAL&googlePlaceId=<place id>`
  (JSON url-encoded). The Google place id for a city is in the URL after you pick the
  city from the homepage autocomplete.
- Property page: `/property/<listingId>` where listingId looks like `1044041_1`.
- Other search params seen: `moveDate={"in":"2026-10-08"}`, `budget={}`,
  `filters={"headerFilters":[],"accessibility":[],"amenities":[],"propertyType":[]}`,
  `page={"currentPage":2}`.

## Getting structured listings (no API key needed)

Each results page embeds every card as a `SearchResultItem` JSON object. Pull them:

```python
raw = js("""Array.from(document.querySelectorAll('script'))
  .filter(s=>/self\\.__next_f\\.push/.test(s.textContent)).map(s=>s.textContent).join('\\n')""")
u = raw.replace('\\"', '"')
items = []
for m in re.finditer(r'\{"propertyType":"[^"]*","propertyTypeClass"[\s\S]*?"__typename":"SearchResultItem"\}', u):
    try: items.append(json.loads(m.group(0)))
    except ValueError: pass
# each listing appears twice in the payload → dedupe on listingId
```

Fields per item: `listingId name propertyType propertyTypeClass (entire_unit|room)
approxLocation{latitude,longitude} availableOnDate ("Available" = now, else "Available: Oct. 08, 2026")
isAvailableNow minimumStayInDays rentAmount.amount ("$$1,650/month" — note the doubled $)
bedroomCount bathroomCount totalSleeps laundryType amenities[] isGoodResponder
isNewListing (this is the "Recently added" badge) unitExternalId photos[]`.

The GraphQL operation behind it is `UnifiedListingSearch` with variables
`{searchType:"REGIONAL", regionId, searchSessionId, includeUnmatched:true, isFormat:true,
placeId, filters:[], pageRequest:{pageNumber,pageSize:72}, viewport:{min,max}}` and the
response has `pageInfo{totalResults hasNextPage ...}` plus `listings[]`.

## Pagination trap → use quadrants

The payload only ever contains **page 1 (72 items)**. `page={"currentPage":2}` in the URL
still renders page 1 server-side; the client fetches page 2 through Apollo (which holds its
own `fetch` reference, so patching `window.fetch` sees nothing). Easiest reliable approach:
split the map viewport into sub-boxes until each has `totalResults <= 72`, load each box's
URL, parse, and dedupe. ~160 listings in a 3 km box needed 4 boxes.

## Property page

Server-rendered text. Useful lines in `document.body.innerText`: `Property ID`, `Reply time`,
`Calendar updated <date>` (staleness signal — many SF landlords have 2025 dates),
`Tenure`, `<n> Sq. Ft.`, `Minimum stay: N months`, `Available: <date>`, `$X /month`,
`Utilities: included`, `Deposit (Refundable) $X`, `Cleaning Fee $X`, `Neighborhood overview`,
`Closest facilities` (hospital distances — a decent proxy for location when the title is vague).
The "similar rentals" strip at the bottom is `a[href*="/property/"]` links and can surface
listings outside your viewport.

## Traps

- `approxLocation` is fuzzed. A listing that says "Laurel Heights" plotted 0.5 km away.
  Use the description / closest-facility distances to confirm the neighborhood.
- Titles lie about bedroom count ("1BR + Office / Optional 2BR" is indexed as 2 bedrooms).
- Rooms (`propertyTypeClass:"room"`) are mixed into results; filter on `entire_unit`.
- Contact requires an account (Send Booking Inquiry). Everything above is anonymous.
