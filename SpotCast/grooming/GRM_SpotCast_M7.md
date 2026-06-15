# SpotCast — Grooming

# GRM-M7 — HereFetcher + CitiesLoader

## Obiettivo
Sostituire `GoogleFetcher` con `HereFetcher` basato su HERE Browse API, e introdurre `CitiesLoader` per gestire la configurazione delle città per paese in un file separato.

## User Story
*Come utente, voglio che SpotCast trovi tutte le attività disponibili nella mia area senza limiti arbitrari, e voglio poter configurare le mie città di interesse in modo semplice indicando il paese di appartenenza.*

---

## Contesto — perché si cambia fetcher

Google Places Text Search ha un limite strutturale di 20 risultati per query (60 con paginazione). In test end-to-end su Tolmezzo e Socchieve con categorie Bar/Tabacchino/Fabbro, il limite globale `results_per_run: 10` ha restituito esclusivamente tabacchini, nascondendo completamente le altre categorie. Anche rimuovendo il limite globale, il cap di Google rimane un problema per qualsiasi comune non triviale.

HERE Browse API non ha limiti arbitrari per query, usa coordinate geografiche invece di query testuali, e ha copertura capillare Europa. (DTR-042, DTR-044)

---

## Criteri di Accettazione

- [ ] `HereFetcher.ts` implementato ed esportato
- [ ] `HereCategoryMap.ts` con mappa label inglese → codice HERE
- [ ] `CitiesLoader.ts` carica `cities.json`, valida con Zod, restituisce `string[]` di coppie `"city, country"`
- [ ] `cities.example.json` committato in `assets/cities/`
- [ ] `cities.json` in `.gitignore`
- [ ] `geocache.json` nella root del progetto, committato con struttura vuota `{}`
- [ ] `ConfigLoader.ts` aggiornato: rimossi `cities`/`countries`, aggiunto `cities_file`
- [ ] `ConfigLoader.ts` emette WARN per campi legacy `google_api_key`, `results_per_run`, `cities`, `countries`
- [ ] `HereFetcher` pagina automaticamente con `offset` fino ad esaurimento risultati HERE
- [ ] `GoogleFetcher.ts` deprecato con commento — non rimosso
- [ ] `SpotCast.ts` aggiorna solo la riga di import e passa le città da `CitiesLoader`
- [ ] Test suite verde al 100%

---

## Struttura file

```
src/
  config/
    ConfigLoader.ts          ← aggiornato
    CitiesLoader.ts          ← nuovo
  fetcher/
    HereFetcher.ts           ← nuovo
    HereCategoryMap.ts       ← nuovo
    GoogleFetcher.ts         ← deprecato, non rimosso

assets/
  cities/
    cities.example.json      ← committato
    cities.json              ← gitignored

geocache.json                ← committato, struttura vuota {}

tests/
  config/
    CitiesLoader.test.ts     ← nuovo
  fetcher/
    HereFetcher.test.ts      ← nuovo
```

---

## Breaking changes in `config.json`

| Campo | Prima | Dopo |
|---|---|---|
| `google_api_key` | presente | rimosso — WARN se presente |
| `here_api_key` | assente | obbligatorio |
| `categories` | stringhe libere (`"Tabacchino"`) | label inglesi dalla mappa (`"Bar"`) |
| `cities` | presente | rimosso — WARN se presente |
| `countries` | presente | rimosso — WARN se presente |
| `cities_file` | assente | obbligatorio |
| `results_per_run` | presente | rimosso — WARN se presente |
| `search_radius_meters` | assente | opzionale, default `15000` |

---

## `cities.json` — struttura e naming

```json
[
  {
    "country": "Germany",
    "cities": ["Berlin", "München", "Hamburg"]
  },
  {
    "country": "Italy",
    "cities": ["Roma", "Milano", "Tolmezzo"]
  },
  {
    "country": "France",
    "cities": ["Paris", "Lyon", "Marseille"]
  }
]
```

**Regola nomi città:** nome ufficiale sulla cartografia nazionale. In paesi multilingua si usa il nome della lingua parlata nella zona geografica (DTR-047).

---

## `CitiesLoader.ts` — interfaccia pubblica

```typescript
export function loadCities(citiesFilePath: string): string[]
// Restituisce: ["Berlin, Germany", "München, Germany", "Roma, Italy", ...]
// Errore fatale se il file non esiste o è malformato
```

Validazione Zod:
```typescript
const CitiesSchema = z.array(z.object({
  country: z.string().min(1),
  cities:  z.array(z.string().min(1)).min(1),
})).min(1, 'At least one country with cities is required');
```

---

## `HereCategoryMap.ts` — validazione in `ConfigLoader`

Le categorie in `config.json` sono label inglesi — es. `"Bar"`, `"Gym"`. `ConfigLoader` le converte in codici HERE tramite `HereCategoryMap`. Categoria non riconosciuta → WARN + skip. Tutte invalide → errore fatale (DTR-043).

---

## `HereFetcher.ts` — architettura

### Geocoding con cache su file

```typescript
// geocache.json — committato, struttura vuota {}
{
  "Berlin, Germany": { "lat": 52.52, "lng": 13.40 },
  "Roma, Italy":     { "lat": 41.90, "lng": 12.50 }
}
```

1. Controlla `geocache.json` — se presente usa le coordinate
2. Se assente → chiama HERE Geocoding API → salva in `geocache.json`

### Paginazione HERE Browse

```typescript
do {
  const response = await browseRequest(coords, categoryCode, offset);
  results.push(...mapResults(response.items));
  if (response.items.length < 100) break;
  offset += 100;
} while (true);
```

Nessun delay obbligatorio — HERE non ha il problema del `next_page_token` di Google.

### Endpoint HERE

```
Browse:   GET https://browse.search.hereapi.com/v1/browse
Geocode:  GET https://geocode.search.hereapi.com/v1/geocode
```

---

## Test (Vitest)

### `CitiesLoader.test.ts`

| Caso | Descrizione |
|---|---|
| File valido | Restituisce array piatto di stringhe `"city, country"` |
| Più paesi | Tutte le città di tutti i paesi nel risultato |
| File mancante | Errore fatale con messaggio descrittivo |
| JSON malformato | Errore fatale |
| Array vuoto | Errore Zod |
| Paese senza città | Errore Zod |

### `HereFetcher.test.ts`

| Caso | Descrizione |
|---|---|
| Geocoding cache hit | Nessuna chiamata HERE Geocoding |
| Geocoding cache miss | Chiamata HERE Geocoding → risultato salvato in cache |
| Paginazione singola pagina | Un solo request browse |
| Paginazione multipagina | Loop fino a pagina < 100 |
| Combinazioni categoria × città | N categorie × M città = N×M chiamate browse |
| Mapping Business corretto | Campi HERE mappati su modello `Business` |
| Campi opzionali mancanti | Nessun crash, campi `undefined` |
| Errore browse su combinazione | Log + risultati parziali, nessun crash |
| Geocoding fallisce | Log + array vuoto, nessun crash |

---

## `config.example.json` aggiornato

```json
{
  "language":             "en",
  "here_api_key":         "YOUR_HERE_API_KEY",
  "categories":           ["Bar", "Gym", "Lawyer"],
  "cities_file":          "assets/cities/cities.json",
  "search_radius_meters": 15000,
  "schedule":             "0 8 * * *",
  "output_dir":           "results",
  "smtp": {
    "host": "smtp.gmail.com",
    "port": 587,
    "user": "your@email.com",
    "pass": "your_app_password"
  },
  "email_to":       ["recipient@email.com"],
  "email_template": "assets/templates/email.html"
}
```

---

## Stima di Effort
**4–5 ore** inclusi CitiesLoader, HereFetcher aggiornato, test suite completa, aggiornamenti ConfigLoader e SpotCast
