# SpotCast — README per Sviluppatori

> Riferimento tecnico per contributori e manutentori.

---

## Stack Tecnologico

| Livello | Tecnologia | Versione |
|---|---|---|
| Runtime | Node.js | >=20.0.0 |
| Linguaggio | TypeScript | ^5.5.0 |
| Gestore pacchetti | pnpm | 10.30.2 |
| Server HTTP | Express | ^4.21.0 |
| Google Maps | `@googlemaps/google-maps-services-js` | ^3.4.2 |
| Generazione Excel | ExcelJS | ^4.4.0 |
| Invio email | Nodemailer | ^6.9.0 |
| Pianificatore | node-cron | ^3.0.3 |
| Validazione config | Zod | ^3.23.0 |
| Logging | Winston | ^3.13.0 |
| Testing | Vitest | ^2.1.0 |
| Linting | ESLint | ^9.0.0 |

---

## Struttura del Progetto

```
spotcast/
│
├── SpotCast.ts              ← punto di ingresso (modalità CLI e daemon)
│
├── config.json              ← configurazione operativa (in gitignore)
├── config.example.json      ← template incluso nel repo
├── excel.json               ← configurazione export Excel (in gitignore)
├── excel.example.json       ← template incluso nel repo
├── seen_firms.json          ← storico deduplicazione (in gitignore)
├── tracker.log              ← log di esecuzione (in gitignore)
├── eslint.config.mjs        ← configurazione flat ESLint 9
├── vitest.config.ts         ← configurazione Vitest
├── nodemon.json             ← config hot-reload nodemon
│
├── assets/
│   └── i18n/                ← localizzazioni — non codice
│       ├── en.json          ← lingua di riferimento (fallback universale)
│       ├── it.json
│       ├── de.json
│       ├── fr.json
│       ├── es.json
│       ├── pt.json
│       ├── zh.json
│       ├── ja.json
│       ├── ar.json
│       └── tr.json
│
├── src/
│   ├── config/
│   │   ├── ConfigLoader.ts       ← carica, valida e unisce config.json + env vars
│   │   └── ExcelConfigLoader.ts  ← carica e valida excel.json (M4)
│   ├── fetcher/
│   │   ├── Business.ts           ← modelli Business, EnrichedBusiness, Metadata
│   │   └── GoogleFetcher.ts      ← connettore Google Places API
│   ├── dedup/
│   │   └── DedupService.ts       ← gestione seen_firms.json
│   ├── excel/
│   │   └── ExcelExporter.ts      ← generazione .xlsx (M4)
│   ├── mailer/
│   │   └── MailService.ts        ← invio email (M5)
│   ├── scheduler/
│   │   └── Scheduler.ts          ← wrapper node-cron (M6)
│   ├── api/
│   │   └── RestServer.ts         ← REST API Express porta 3847 (M7)
│   ├── i18n/
│   │   └── translate.ts          ← utility t() con fallback a cascata
│   └── logger.ts                 ← logger Winston condiviso
│
├── templates/
│   └── email.html               ← template email Handlebars
│
├── results/                     ← output Excel (in gitignore)
└── tests/
    ├── config/
    │   ├── ConfigLoader.test.ts
    │   └── ExcelConfigLoader.test.ts
    ├── fetcher/
    │   └── GoogleFetcher.test.ts
    ├── dedup/
    │   └── DedupService.test.ts
    ├── excel/
    │   └── ExcelExporter.test.ts
    └── i18n/
        └── translate.test.ts
```

### Convenzione struttura

| Cartella | Contenuto |
|---|---|
| `src/` | Tutto il codice TypeScript — moduli, classi, utility |
| `assets/` | Tutto ciò che non è codice: localizzazioni, template statici |
| `tests/` | Suite Vitest — specchia la struttura di `src/` |
| root | File di configurazione del progetto (`*.json`, `*.ts`, `*.mjs`) |

---

## File di Configurazione

Ogni dominio funzionale ha il proprio file dedicato (DTR-029):

| File | Dominio | Committato |
|---|---|---|
| `config.json` | Configurazione operativa (API key, SMTP, schedule) | ❌ gitignored |
| `config.example.json` | Template senza segreti | ✅ |
| `excel.json` | Layout export Excel (colonne, sheet) | ✅ |
| `seen_firms.json` | Storico deduplicazione | ❌ gitignored |

---

## Configurazione dell'Ambiente di Sviluppo

```bash
# Clone
git clone https://github.com/Belmani/SpotCast.git
cd SpotCast

# Installa pnpm se necessario
npm install -g pnpm@10.30.2

# Installa le dipendenze
pnpm install

# Copia le configurazioni
cp config.example.json config.json
cp excel.example.json excel.json
# Inserisci google_api_key e credenziali SMTP in config.json

# Avvia in modalità sviluppo (hot-reload)
pnpm dev

# Build di produzione
pnpm build && pnpm start
```

---

## Script Disponibili

```bash
pnpm dev            # ts-node + hot-reload nodemon
pnpm build          # compila TypeScript → dist/
pnpm start          # esegue la build compilata
pnpm test           # Vitest in modalità watch
pnpm test:run       # esecuzione singola, senza watch
pnpm test:coverage  # report di copertura in /coverage
pnpm lint           # analisi ESLint
pnpm lint:fix       # correzione automatica problemi ESLint
pnpm typecheck      # controllo tipi senza build
pnpm clean          # elimina dist/
```

---

## Variabili d'Ambiente

I valori sensibili sovrascrivono i corrispondenti campi di `config.json` quando presenti:

| Variabile | Campo config |
|---|---|
| `GOOGLE_API_KEY` | `google_api_key` |
| `SMTP_USER` | `smtp.user` |
| `SMTP_PASS` | `smtp.pass` |

```bash
# macOS / Linux
export GOOGLE_API_KEY="AIzaSy..."

# Windows
setx GOOGLE_API_KEY "AIzaSy..."
```

---

## Moduli Principali

### ConfigLoader (`src/config/ConfigLoader.ts`)

Carica `config.json`, valida con lo schema Zod, applica le sovrascritture delle variabili d'ambiente. Gli errori fatali vengono scritti su `process.stderr` — nessuna dipendenza da Winston nella fase di avvio, per evitare importazioni circolari.

```typescript
export type Config = z.infer<typeof ConfigSchema>;
export function loadConfig(configPath = 'config.json'): Config
```

### ExcelConfigLoader (`src/config/ExcelConfigLoader.ts`)

Carica `excel.json`, valida con Zod. Protegge i campi obbligatori (`name`, `category`, `city`, `address`) — se disabilitati lancia errore descrittivo.

```typescript
export type ExcelConfig = z.infer<typeof ExcelConfigSchema>;
export function loadExcelConfig(configPath = 'excel.json'): ExcelConfig
```

### GoogleFetcher (`src/fetcher/GoogleFetcher.ts`)

Racchiude `@googlemaps/google-maps-services-js`. Per ogni combinazione `(categoria × città)` nella config, esegue `textSearch` e mappa i risultati nel modello interno `Business`. Non lancia mai eccezioni in caso di errore API — restituisce risultati parziali.

```typescript
fetchAll(): Promise<Business[]>
fetchOne(category, city, limit): Promise<Business[]>
```

### Modello Business (`src/fetcher/Business.ts`)

```typescript
interface Business {
  place_id: string;
  name: string;
  category: string;
  city: string;
  country: string;
  address: string;
  phone?: string;
  website?: string;
  rating?: number;
  review_count?: number;
  maps_url: string;
  first_seen?: string;  // ISO — valorizzato da DedupService.markSeen()
  last_seen?: string;   // ISO — aggiornato da DedupService.markSeen() ad ogni run
}
```

### DedupService (`src/dedup/DedupService.ts`)

Gestisce `seen_firms.json`. Path iniettabile nel costruttore per testabilità.

```typescript
constructor(filePath?: string)
filter(businesses: Business[]): Business[]
markSeen(businesses: Business[]): void   // chiamare DOPO l'invio email
reset(): void
```

### ExcelExporter (`src/excel/ExcelExporter.ts`)

Genera il file `.xlsx` con tre sheet. Riceve i dizionari i18n già caricati — non li carica lui.

```typescript
constructor(config: Config, excelConfig: ExcelConfig)
async export(
  businesses: Business[],
  duplicatesSkipped: number,
  i18n: Record<string, string>,
  fallback: Record<string, string>
): Promise<string>  // restituisce il path assoluto del file generato
```

### translate (`src/i18n/translate.ts`)

Utility condivisa con tre livelli di fallback (DTR-027).

```typescript
// Fallback chain: i18n[key] → fallback[key] → key
export function t(
  key: string,
  i18n: Record<string, string>,
  fallback: Record<string, string>
): string
```

### Logger (`src/logger.ts`)

Istanza Winston condivisa. Livelli: `error/warn/info/debug`. Il trasporto su file è sempre attivo; il trasporto su console è disattivato in produzione.

---

## Sistema i18n

Le localizzazioni vivono in `assets/i18n/` — separazione netta tra codice (`src/`) e risorse statiche (`assets/`). La logica di traduzione vive in `src/i18n/translate.ts`.

**10 lingue al lancio:** `it`, `en`, `de`, `fr`, `es`, `pt`, `zh`, `ja`, `ar`, `tr`

Il caricamento del file lingua attivo è responsabilità del chiamante — `translate.ts` riceve dizionari già caricati e non tocca il filesystem.

Per aggiungere una nuova lingua:
1. Copia `assets/i18n/en.json` → `assets/i18n/{lang}.json`
2. Traduci tutti i valori (mantieni le chiavi identiche)
3. Aggiungi il codice lingua all'enum Zod in `ConfigLoader.ts`
4. Apri una PR

---

## Strategia di Testing (Vitest)

**Obiettivo di copertura: minimo 80% su tutti i moduli.**

```bash
pnpm test:run                    # esegue tutti i test una volta
pnpm test:coverage               # genera report di copertura HTML
```

**Filosofia:** file reali in `os.tmpdir()` invece di mock del filesystem — nascondere le librerie con mock maschera bug reali di serializzazione. Applicato a `DedupService` (JSON) e `ExcelExporter` (xlsx).

**Nota mocking costruttori:** mock di classi con `new` usano `function`, non arrow function.

**Nota async:** callback `it()` con `await` devono essere `async`.

---

## Git Flow

```
main          ← solo release stabili
develop       ← branch di integrazione
feature/xxx   ← una feature per branch, parte da develop
release/xxx   ← preparazione release
hotfix/xxx    ← correzioni urgenti su main
```

---

## Politica di Versioning delle Dipendenze

Dipendenze fissate all'ultima versione **stabile** major. Nessun aggiornamento a RC o beta. Prima di aggiornare una major version, verificare compatibilità e documentare nel DTR.

Vincoli attuali:
- Node.js `>=20.0.0`
- Express `^4.x` (v5 ancora in RC)
- Zod `^3.x` (v4 non ancora stabile)

---

## Checklist di Release

- [ ] Tutti i test passano (`pnpm test:run`)
- [ ] Copertura ≥ 80% (`pnpm test:coverage`)
- [ ] Nessun errore ESLint (`pnpm lint`)
- [ ] Nessun errore TypeScript (`pnpm typecheck`)
- [ ] `config.example.json` aggiornato se aggiunti nuovi campi
- [ ] `excel.example.json` aggiornato se aggiunti nuovi campi
- [ ] `seen_firms.json` e `config.json` in `.gitignore`
- [ ] `CHANGELOG.md` aggiornato
- [ ] Versione aggiornata in `package.json`
- [ ] Riepilogo milestone aggiunto in `docs/`
