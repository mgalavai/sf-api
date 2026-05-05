# Filmstaden API notes

Observed 2026-05-05. The old `app.sfbio.se` mobile API documented in the
README no longer responded from local testing. Filmstaden's current web site
uses `https://services.cinema-api.com/` for catalog, show, and seat status data.

These endpoints are unauthenticated in the browser flow observed on
`filmstaden.se`. Be polite: cache responses and avoid high-frequency polling.

## Constants

```text
Base URL:        https://services.cinema-api.com
Country alias:   se
Language:        sv
Sales channel:   1
Tenant/site id:  1024
Remote system:   Sys99-SE
Channel query:   Channel=Web
```

Examples below use Gothenburg (`CityAlias=GB`). Other confirmed aliases:

```text
GB = Gothenburg
SE = Stockholm
VL = Vetlanda
VS = Vasteras
```

## Shows

List shows for a city:

```bash
curl -sS 'https://services.cinema-api.com/show/stripped/sv/1/1024/?CountryAlias=se&CityAlias=GB&Channel=Web'
```

List shows for all of Sweden:

```bash
curl -sS 'https://services.cinema-api.com/show/stripped/sv/1/1024/?CountryAlias=se&Channel=Web'
```

Important fields in `items[]`:

```json
{
  "reId": "show id",
  "mId": "movie id",
  "mvId": "movie version id",
  "cId": "cinema id",
  "ct": "cinema title",
  "sId": "screen id",
  "st": "screen title",
  "utc": "2026-05-05T09:30:00Z",
  "sa": ["show attributes"],
  "res": [],
  "seatT": [],
  "lzp": 229
}
```

`reId` is the identifier used for seat status and detailed show calls.

## Seat status

Seat status is a POST that accepts up to many show ids. Batching 500 ids per
request worked during testing.

```bash
curl -sS 'https://services.cinema-api.com/show/seatstatussummary/Sys99-SE/se' \
  -H 'Content-Type: application/json; charset=utf-8' \
  --data '{"showIds":["SHOW_ID_1","SHOW_ID_2"]}'
```

Example response:

```json
[
  {
    "isActive": true,
    "text": "K&ouml;p biljetter",
    "showId": "SHOW_ID_1",
    "status": {
      "Free": 158,
      "Sold": 1
    },
    "isReservable": false,
    "bookingQuota": 0,
    "freeSeatsCount": 158,
    "maxVisitors": 0
  }
]
```

For an empty-screenings project, treat a show as empty when:

```text
sum(status bucket counts) - freeSeatsCount == 0
```

For example, `{"Free": 159}` means the whole auditorium appears unsold. A
response like `{"Free": 158, "Sold": 1}` means one seat is taken and 158 are
free. The status buckets may include more than `Free` and `Sold`, so compute
"taken" from all buckets rather than checking only `Sold`.

## Movie catalog

Scheduled/current movie metadata:

```bash
curl -sS 'https://services.cinema-api.com/movie/scheduled/sv/1/1024/false'
```

Upcoming movie metadata:

```bash
curl -sS 'https://services.cinema-api.com/movie/upcoming/sv/1/1024/false'
```

Join `show/stripped` `mId` to movie `ncgId` to get title, slug, genres, images,
runtime, rating, and version metadata.

## Detailed show

Fetch richer details for a single show:

```bash
curl -sS 'https://services.cinema-api.com/show/Sys99-SE/SHOW_ID/sv'
```

This returns cinema details, screen data, pricing/product metadata, movie
version, serving type, and time fields. The summary endpoint is enough for
counting empty shows; use this detailed endpoint only when the UI needs richer
cinema or checkout-adjacent metadata.

## Show attributes

```bash
curl -sS 'https://services.cinema-api.com/show/attributes/sv/1/1024'
```

Useful for resolving attributes in `sa` and movie/show version metadata.

## Example: empty shows in Gothenburg today

This script lists empty shows for Gothenburg for the current local day in
Europe/Stockholm:

```bash
node <<'NODE'
const base = 'https://services.cinema-api.com';
const cityAlias = 'GB';
const shows = await fetch(`${base}/show/stripped/sv/1/1024/?CountryAlias=se&CityAlias=${cityAlias}&Channel=Web`).then(r => r.json());
const scheduled = await fetch(`${base}/movie/scheduled/sv/1/1024/false`).then(r => r.json());
const upcoming = await fetch(`${base}/movie/upcoming/sv/1/1024/false`).then(r => r.json());
const movieById = new Map([...(scheduled.items || []), ...(upcoming.items || [])].map(m => [m.ncgId, m]));

const fmtDate = new Intl.DateTimeFormat('sv-SE', {
  timeZone: 'Europe/Stockholm',
  year: 'numeric',
  month: '2-digit',
  day: '2-digit'
});
const fmtTime = new Intl.DateTimeFormat('sv-SE', {
  timeZone: 'Europe/Stockholm',
  hour: '2-digit',
  minute: '2-digit',
  hourCycle: 'h23'
});

const now = new Date();
const today = fmtDate.format(now);
const todaysShows = (shows.items || [])
  .filter(show => new Date(show.utc) >= now && fmtDate.format(new Date(show.utc)) === today);

const statusResponse = await fetch(`${base}/show/seatstatussummary/Sys99-SE/se`, {
  method: 'POST',
  headers: {'Content-Type': 'application/json; charset=utf-8'},
  body: JSON.stringify({showIds: todaysShows.map(show => show.reId)})
});
const statuses = await statusResponse.json();
const statusById = new Map(statuses.map(status => [status.showId, status]));

const rows = todaysShows.map(show => {
  const status = statusById.get(show.reId);
  const buckets = status?.status || {};
  const free = status?.freeSeatsCount ?? buckets.Free ?? 0;
  const total = Object.values(buckets).reduce((sum, value) => sum + (typeof value === 'number' ? value : 0), 0);
  const taken = Math.max(0, total - free);
  return {
    time: fmtTime.format(new Date(show.utc)),
    cinema: show.ct,
    screen: show.st,
    title: movieById.get(show.mId)?.title || show.mId,
    free,
    taken,
    total
  };
});

console.table(rows.filter(row => row.taken === 0).sort((a, b) => b.free - a.free));
NODE
```

## Snapshot stats

At 2026-05-05 09:37 Europe/Stockholm, the all-Sweden show endpoint returned:

```text
Future shows listed:       5,901
Distinct cinemas:          76
Distinct screens/salons:   418
Distinct movies with shows: 88
Shows today:               819
Shows in next 7 days:      4,099
```

For the next 7 days, `seatstatussummary` returned data for every listed show:

```text
Empty shows:                  2,318
Shows with <= 1 taken seat:   2,402
Shows with <= 5 taken seats:  3,322
Shows with <= 10 taken seats: 3,639
Known seats:                  441,229
Free seats:                   414,447
Sold/unavailable seats:       26,782
Approx occupancy:             6.07%
```

At 2026-05-05 09:39 Europe/Stockholm, Gothenburg (`GB`) for the rest of the
local day returned:

```text
Remaining shows today:        93
Distinct cinemas:             3
Distinct screens/salons:      28
Distinct movies:              36
Empty shows:                  38
Shows with <= 5 taken seats:  61
Known seats:                  11,801
Free seats:                   11,198
Sold/unavailable seats:       603
Approx occupancy:             5.11%
```
