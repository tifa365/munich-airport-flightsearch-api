# Flughafen München – Flugsuche-API (inoffiziell)

Beschreibung der AJAX-Endpunkte hinter der Flugsuche auf <https://www.munich-airport.de/flugsuche-89523>
(Rails-Anwendung „pinuts“). Die Spezifikation liegt in [`openapi.yaml`](openapi.yaml) (OpenAPI 3.1, validiert mit
`openapi-spec-validator` und `redocly lint`). Stand der Live-Prüfung: **2026-10-25**.

> Inoffiziell. Keine Zusicherung über Stabilität; Nutzung nur im Rahmen der Nutzungsbedingungen des Betreibers.
> Antworten der Flugtafel sind **HTML-Fragmente**, kein JSON.

## Endpunkte

| Pfad | Format | Status |
|---|---|---|
| `GET /flightsearch/departures` | HTML-Fragment | live bestätigt |
| `GET /flightsearch/arrivals` | HTML-Fragment | live bestätigt |
| `GET /flightsearch/autocomplete_airport?search_term=` | JSON | live bestätigt |
| `GET /flightsearch/autocomplete_airline?search_term=` | JSON | live bestätigt |
| `GET /flightsearch/tracker?flight_ids[]=` | HTML-Fragment | live bestätigt |
| `GET /flightsearch/weather_info?airport_id=&time=` | HTML-Fragment | live bestätigt |
| `GET /flightsearch/flight_process_time/{id}?direction=` | JSON | nur 404 beobachtet, Erfolgsformat **inferiert** |
| `GET /flugdetailseite-75020?flight_id=` | HTML-Seite | live bestätigt (nur mit `Accept: text/html`) |
| `GET /_b/…/flugplan.pdf` | PDF | statisch, Hash wechselt je Ausgabe |

## Schnellstart

```bash
# Abflüge heute, 6 je Seite (Such-Modus)
curl 'https://www.munich-airport.de/flightsearch/departures?flight_search_presenter%5Bflight_date_from_muc%5D=2026-10-25&page=0&per_page=6'

# Abflugtafel ab jetzt, Blättern mit page=±n (Board-Modus)
curl 'https://www.munich-airport.de/flightsearch/departures?from=2026-10-25T10%3A25%3A47&allow_scroll_back=1&per_page=50&min_date=2026-10-24T00%3A00%3A00&max_date=2026-10-27T00%3A00%3A00&page=0'

# IDs für Filter
curl 'https://www.munich-airport.de/flightsearch/autocomplete_airport?search_term=FCO'   # → id "1117"
curl 'https://www.munich-airport.de/flightsearch/autocomplete_airline?search_term=LH'    # → id 52

# Gefiltert: Lufthansa nach Rom
curl 'https://www.munich-airport.de/flightsearch/departures?flight_search_presenter%5Bflight_date_from_muc%5D=2026-10-25&flight_search_presenter%5Bairport_id%5D=1117&flight_search_presenter%5Bairline_id%5D=52&per_page=6&page=0'
```

Keine Authentifizierung, kein CSRF-Token und kein `X-Requested-With` nötig (alles GET). `POST` → 403.

## Sprache

`flight_search_presenter[locale]` wird ignoriert. Die Sprache hängt am Session-Cookie `_fmg-a1`, das beim Aufruf einer
beliebigen Seite des jeweiligen Hosts gesetzt wird. Englisch:

```bash
curl -c jar.txt -o /dev/null https://www.munich-airport.com/all-departures-1450598
curl -b jar.txt 'https://www.munich-airport.com/flightsearch/departures?per_page=6&page=0'   # "Flights from Munich on …"
```

Ohne Cookie antworten beide Hosts auf Deutsch.

## Parsen der Flugtafel

- Jeder Flug steht **zweimal** im Fragment (`table.fp-flights-table-large` und `table.fp-flights-table-small`).
  Nach `tr.fp-flight-item[data-flight-id]` deduplizieren oder nur die große Tabelle parsen.
- `div.fp-flight-board-body[data-total-results]` = Gesamtzahl über alle Seiten.
- `td.fp-flight-time-muc` enthält `geplant&nbsp;|&nbsp;erwartet`; der zweite Teil ist leer, wenn es keine Abweichung gibt.
- `td.fp-flight-number` enthält `LH 1842 (A20N)`; weitere `span.nobr` in der Zelle sind Codeshare-Nummern.
- Beobachtete Statuswerte (de): `geplant`, `Check-In`, `boarding`, `gestartet`, `gelandet`. Bereiche: `T1A`–`T1D`, `T2`.
- Keine Treffer: `div.fp-flights-not-found`.
- Selektoren vollständig in `openapi.yaml` unter `components.x-htmlFragments`.

## Beobachtete Grenzen und Fehler (2026-10-25)

- `per_page`: 0/ungültig → 10. Keine feste Obergrenze: Such-Modus mit 1000 → alle 400 Tagesflüge; Board-Modus über die gesamte Zukunft mit 750 ok (3,3 MB), mit 1000 → **500**.
- Ungültiges `from`/`min_date`/`max_date` oder `flight_search_presenter=skalar` → **500**.
- Datumsabfragen ab 2026-10-27 lieferten durchgängig **500** (vermutlich Datenstand zum Winterflugplan-Wechsel);
  2027-03-28 → 200 mit 0 Treffern. Vor Nutzung erneut prüfen.
- Ungültiges Datum im Such-Modus → Rückfall auf heute, unbekannte `airport_id` → 0 Treffer (jeweils 200).
- `flight_search_presenter[airport]`/`[airline]` (Anzeigetexte) und `flight_from_muc`/`flight_to_muc` haben serverseitig keine Wirkung.
- Eine WAF blockiert auffällige URLs (Steuerzeichen, absurde Zahlen/Daten, leere Parameternamen) und `POST`/`TRACE` mit **403** „Request Rejected“; es gibt kein 405.
- `flight_process_time` antwortet bei 404 mit Klartext unter `Content-Type: application/json` (kein gültiges JSON).
- `flight_time_from_muc` akzeptiert beliebige `HH:MM-HH:MM`-Intervalle und `week`.

## Abdeckung

- **Bestätigt**: alle oben als „live bestätigt“ markierten Endpunkte inkl. Parameter-Semantik, Pagination, Locale-Mechanismus, Fehlerfälle.
- **Inferiert**: Erfolgsantwort von `flight_process_time` (nur aus Client-JS rekonstruiert, live nur 404);
  Wirkung von `allow_pagination` und `perform_search` (keine messbare Änderung).
- **Nicht abgedeckt**: die Site-Suche `/solr/aggregation_widget` (keine Flugdaten), Buchungs-Links, Mobility-as-a-Service-Formular.

## Validierung und Fuzzing

```bash
uvx --from openapi-spec-validator openapi-spec-validator openapi.yaml
npx @redocly/cli@1.34.3 lint openapi.yaml
scripts/fuzz.sh            # Schemathesis, gedrosselt (2 req/s, 10 Beispiele je Operation)
```

Der Fuzz-Lauf läuft gegen das Live-System eines Dritten und ist deshalb bewusst klein dimensioniert
(`MAX_EXAMPLES`, `RATE`, `BASE_URL` per Umgebungsvariable anpassbar). Ergebnisse unter `fuzz-reports/`,
Zusammenfassung in [`fuzz-reports/SUMMARY.md`](fuzz-reports/SUMMARY.md).
