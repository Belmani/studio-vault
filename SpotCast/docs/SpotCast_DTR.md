# SpotCast — Decisioni Tecniche e Architetturali (DTR)

> Questo documento registra tutte le decisioni tecniche e architetturali significative prese durante il progetto, con motivazione e alternative valutate. Aggiornato man mano che il progetto evolve.

---

## DTR-001 — Linguaggio: TypeScript al posto di Python

**Data:** 2026-05-26
**Stato:** Accettata

**Decisione:** Riscrivere il `google_maps_tracker.py` originale in TypeScript/Node.js invece di mantenere il codebase Python.

**Motivazione:**
- Lo script Python originale è stato fornito dal cliente come riferimento, non come deliverable
- Lo stack primario del team di sviluppo è Node.js/TypeScript
- TypeScript offre tipizzazione forte, miglior supporto IDE e un ecosistema maturo per tutte le dipendenze necessarie
- Node.js gira nativamente su macOS (requisito del cliente confermato)

**Alternative valutate:**
- Mantenere Python: scartato — il team non ha competenze Python, onere di manutenzione inaccettabile
- Migrare a Java: scartato — eccessivo per uno strumento backend di questo scope

---

## DTR-002 — Singolo connettore dati: solo Google Places API

**Data:** 2026-05-27
**Stato:** Accettata

**Decisione:** Implementare esclusivamente il connettore Google Places API. Rimuovere il connettore Claude API presente nello script di riferimento. Nessuna interfaccia astratta `IFetcher`.

**Motivazione:**
- Lo script Python di riferimento usava Claude per generare dati aziendali fittizi come scorciatoia di sviluppo — non ha valore in produzione
- Lo scopo di SpotCast è la lead generation da dati reali
- Una Google Places API key è disponibile per lo sviluppo fin da subito; Onur creerà la sua alla consegna
- `IFetcher` è architettura speculativa: aggiunge complessità senza un secondo connettore concreto all'orizzonte

**Alternative valutate:**
- Mantenere connettore Claude come fallback dev: scartato — chiave Google disponibile dal giorno uno
- Interfaccia astratta `IFetcher`: rimandato a quando (e se) si renderà necessaria una seconda sorgente

**Nota futura:** se venisse aggiunta una seconda sorgente (es. Yelp, Foursquare), `IFetcher` sarà introdotto tramite refactor in quel momento.

---

## DTR-003 — REST API come bridge verso la GUI (Express su porta 3847)

**Data:** 2026-05-27
**Stato:** Accettata

**Decisione:** SpotCast espone una REST API locale su `http://localhost:3847` tramite Express, come ponte di comunicazione tra il backend Node.js e la futura GUI JavaFX (ForgeUI).

**Motivazione:**
- La GUI sarà costruita in JavaFX (Java), che non può invocare direttamente gli interni di Node.js
- Due opzioni architetturali valutate:
  - **Child Process:** Java lancia `node SpotCast.js` e legge stdout. Semplice ma unidirezionale, fragile, non interrogabile
  - **REST API:** Node espone endpoint HTTP; Java li chiama. Bidirezionale, JSON strutturato, stato interrogabile, robusto
- REST API scelta per robustezza e orientamento al futuro
- Porta `3847` per evitare conflitti con porte di sviluppo comuni
- Il server REST si avvia solo in modalità `--daemon`

**Alternative valutate:**
- WebSocket: scartato — overkill per interazione request/response
- gRPC: scartato — complessità non giustificata per uno strumento locale

---

## DTR-004 — Chiave di deduplicazione: Google Place ID

**Data:** 2026-05-27
**Stato:** Accettata

**Decisione:** Usare `place_id` come chiave di deduplicazione in `seen_firms.json`.

**Motivazione:**
- Nomi e indirizzi possono cambiare, causando false rilevazioni di "nuova" attività
- `place_id` è stabile, univoco e fornito dall'API Google Places su ogni risultato
- Garantisce uno storico affidabile nel lungo periodo

---

## DTR-005 — Configurazione: config.json + override da variabili d'ambiente

**Data:** 2026-05-27
**Stato:** Accettata

**Decisione:** Tutta la configurazione risiede in `config.json`. I campi sensibili (`google_api_key`, `smtp.user`, `smtp.pass`) vengono sovrascritti da variabili d'ambiente (`GOOGLE_API_KEY`, `SMTP_USER`, `SMTP_PASS`) quando presenti.

**Motivazione:**
- `config.json` offre un file di configurazione unico e leggibile per gli utenti finali
- Le variabili d'ambiente seguono la convenzione 12-factor app per i segreti
- `config.json` è gitignored; `config.example.json` (senza segreti) viene committato
- I segreti non toccano mai il repository — non possono essere condivisi accidentalmente

**Integrazione con M10+ Wizard:**
Il wizard grafico scriverà i valori non sensibili in `config.json` e imposterà le variabili d'ambiente sensibili direttamente nel profilo utente del sistema operativo (`~/.zshrc` su macOS, variabili di sistema su Windows, `~/.profile` su Linux). I segreti non toccano mai `config.json`.

---

## DTR-006 — Validazione configurazione: Zod

**Data:** 2026-05-27
**Stato:** Accettata

**Decisione:** Usare Zod v3 per la validazione di `config.json` e l'inferenza automatica del tipo `Config`.

**Motivazione:**
- `JSON.parse()` restituisce `any` — TypeScript non può garantire nulla sulla struttura di un file esterno
- Zod permette di dichiarare schema, validazione e tipo in un'unica dichiarazione
- Errori campo per campo leggibili dall'utente (es. `google_api_key is required`)
- Il tipo `Config` viene inferito automaticamente — nessuna duplicazione tra schema e interfaccia TypeScript

**Nota versione:** Zod 4 è disponibile ma non ancora stabile per produzione. Coerente con la filosofia di stabilità del progetto (vedi DTR-016).

**Alternative valutate:**
- Validazione manuale: scartato — verboso, ripetitivo, facile da dimenticare
- `io-ts`: scartato — API più complessa, curva di apprendimento più alta per benefici equivalenti

---

## DTR-007 — Logging: Winston

**Data:** 2026-05-27
**Stato:** Accettata

**Decisione:** Usare Winston come sistema di logging con transport file (`tracker.log`) e console (silenziata in produzione).

**Motivazione:**
- `fs.appendFileSync` è insufficiente per un tool distribuito: nessun livello, nessuna formattazione, nessuna rotazione
- Winston è il logger più maturo dell'ecosistema Node.js, production-ready
- Livelli `error/warn/info/debug` permettono di filtrare il rumore in produzione
- Formato log standardizzato: `[YYYY-MM-DD HH:mm:ss] LEVEL: message` — tutto in inglese
- Console silenziata in produzione (`NODE_ENV=production`) per non disturbare l'utente finale
- Nei moduli critici pre-boot (es. `ConfigLoader`) si usa `process.stderr.write` per evitare dipendenze circolari

---

## DTR-008 — Testing: Vitest

**Data:** 2026-05-27
**Stato:** Accettata

**Decisione:** Usare Vitest come framework di testing. TDD integrato in ogni milestone — i test non sono opzionali.

**Motivazione:**
- Vitest è ESM nativo, performance superiore a Jest, API identica (drop-in replacement per chi conosce Jest)
- Mocking API pulita e moderna — `vi.mock()`, `vi.fn()`, `vi.spyOn()`
- Integrazione nativa con TypeScript senza configurazione aggiuntiva
- Segnala consapevolezza dell'ecosistema moderno — scelta che comunica seniority
- Coverage provider `v8` tramite `@vitest/coverage-v8`

**Coverage target:** 80% minimo su tutti i moduli.

**Nota mocking:** i mock di classi instanziate con `new` devono usare `function` e non arrow function — le arrow function non possono essere costruttori in JavaScript. Lezione appresa in M2.

**Nota async nei test:** le callback di `it()` che contengono `await` (es. import dinamici) devono essere dichiarate `async`. Lezione appresa in M3.

**Alternative valutate:**
- Jest: scartato — architettura CommonJS-first, performance inferiore, configurazione TypeScript più verbosa
- Bun Test: scartato — ecosistema ancora giovane, prematuro per un progetto distribuito a terzi

---

## DTR-009 — Modello dati: composizione tramite Metadata

**Data:** 2026-05-27
**Stato:** Accettata

**Decisione:** Il modello base `Business` viene esteso nelle milestone successive tramite composizione (`EnrichedBusiness extends Business` con array `metadata: Metadata[]`), non tramite ereditarietà multipla o modifica del modello base.

**Motivazione:**
- Il modello base `Business` corrisponde a ciò che Google Places API fornisce oggi — deve rimanere stabile
- Le informazioni aggiuntive variano per sorgente e milestone — l'ereditarietà creerebbe una gerarchia rigida
- La composizione con `Metadata[]` è aperta per definizione: ogni nuova informazione è un `Metadata` in più, zero modifiche al modello base
- Schema analogo ai campi custom in un catalogo e-commerce: chiave/valore estensibile all'infinito
- `MetadataKey` enum estensibile milestone per milestone senza breaking changes

**Struttura:**
```typescript
interface Metadata {
  key: string;           // MetadataKey enum o stringa custom
  value: string;
  source?: string;       // 'google' | 'manual' | 'enrichment_api'
  collected_at?: string; // ISO timestamp
}

interface EnrichedBusiness extends Business {
  metadata: Metadata[];
}
```

---

## DTR-010 — Storico discovery: SQLite con better-sqlite3 (M11+)

**Data:** 2026-05-27
**Stato:** Pianificata (M11+)

**Decisione:** Il file `seen_firms.json` verrà sostituito da un database SQLite gestito tramite `better-sqlite3` per supportare lo storico discovery visualizzabile dalla GUI.

**Motivazione:**
- `seen_firms.json` è sufficiente per la deduplicazione ma non per query storiche
- SQLite è un singolo file `.db` nella cartella del progetto — zero server, zero configurazione, zero installazione aggiuntiva
- `better-sqlite3` è sincrono, performante e tipizzato
- È il database più distribuito al mondo — affidabilità comprovata

**Alternative valutate:**
- MongoDB/CouchDB: scartato — richiede server separato
- LowDB: scartato — non performante per query storiche
- PostgreSQL: scartato — overkill totale

---

## DTR-011 — Internazionalizzazione: 10 lingue di lancio

**Data:** 2026-05-27
**Stato:** Accettata

**Decisione:** SpotCast è distribuito con 10 lingue al lancio, selezionate per massimizzare la copertura del bacino di utenza mondiale.

**Lingue di lancio:** Italiano, Inglese, Tedesco, Francese, Spagnolo, Portoghese, Cinese semplificato, Giapponese, Arabo, Turco.

**Motivazione:**
- Le 10 lingue coprono oltre l'80% del traffico web mondiale
- Turco aggiunto per la rilevanza del mercato di riferimento (cliente iniziale Onur) e la dimensione del bacino (85M+ parlanti)
- Arabo richiede attenzione specifica per il layout RTL nell'export Excel
- Lingue aggiuntive arriveranno nelle milestone successive su richiesta della community

---

## DTR-012 — GUI: JavaFX tramite ForgeUI (M10+)

**Data:** 2026-05-27
**Stato:** Pianificata (M10+)

**Decisione:** La GUI sarà costruita in JavaFX usando il design system ForgeUI, sviluppato in parallelo con NomadSync. SpotCast funge da progetto pilota di ForgeUI.

**Motivazione:**
- JavaFX è nativo, cross-platform e non richiede server
- ForgeUI garantisce coerenza visiva tra SpotCast e NomadSync
- La REST API (DTR-003) fornisce il ponte tra GUI Java e backend Node.js
- L'alternativa web (Next.js + Tailwind) richiederebbe un server — scartata

---

## DTR-013 — Installer grafico con wizard (M10+)

**Data:** 2026-05-27
**Stato:** Pianificata (M10+)

**Decisione:** SpotCast includerà un installer grafico che automatizza l'intera procedura di setup.

**Motivazione:**
- L'installazione manuale è una barriera per gli utenti non tecnici
- La distribuzione su Softonic richiede un'esperienza bubez-proof
- Il wizard è il punto di ingresso naturale per la GUI JavaFX/ForgeUI

---

## DTR-014 — Modello di pricing: Freemium con sblocco Pro dal mese 2

**Data:** 2026-05-27
**Stato:** Accettata (decisione di business)

**Decisione:** SpotCast è gratuito per il primo mese. Dal mese 2, le funzionalità avanzate richiedono un abbonamento Pro a **€3,99/mese** (o €39/anno).

**Piano gratuito:** pipeline base, fino a 5 categorie, fino a 5 città, schedule giornaliero, 10 lingue.

**Piano Pro:** categorie e città illimitate, storico aggregato multi-settimana, template email personalizzabili da GUI, supporto prioritario.

---

## DTR-015 — Distribuzione: MIT su GitHub + Softonic

**Data:** 2026-05-27
**Stato:** Accettata (decisione di business)

**Decisione:** SpotCast è distribuito come open-source (MIT) su GitHub e pubblicato su Softonic e piattaforme freeware equivalenti.

---

## DTR-016 — Politica di versioning delle dipendenze

**Data:** 2026-06-04
**Stato:** Accettata

**Decisione:** Le dipendenze vengono fissate alla versione major stabile più recente e aggiornate deliberatamente, non inseguendo ogni novità. Priorità alla compatibilità con il parco installato reale degli utenti finali.

**Motivazione:**
- L'utente finale di SpotCast non è un developer — il wizard installerà Node per lui
- La stessa filosofia governa Java 21 su NomadSync e ForgeUI: stabilità a lungo termine > feature recenti
- Node.js `>=20.0.0`, pnpm `>=9.0.0` — soglie che coprono il 90%+ delle installazioni attuali
- Express 4.x (non 5 RC), Zod 3.x (non 4 beta): major version in produzione stabile confermata

**Regola operativa:** prima di aggiornare una major version, verificare compatibilità con tutte le dipendenze del progetto e documentare la decisione in un nuovo DTR.

---

## DTR-017 — Package manager: pnpm

**Data:** 2026-06-04
**Stato:** Accettata

**Decisione:** Usare pnpm come package manager al posto di npm.

**Motivazione:**
- npm si era rivelato problematico in progetti precedenti del team (ToDoList) in combinazione con StoryBook
- pnpm è più veloce, deterministico e Docker-friendly
- Store globale con hard link: installazioni più veloci, `node_modules` più leggeri
- Workspaces nativi per eventuale evoluzione monorepo
- `pnpm-lock.yaml` garantisce riproducibilità cross-macchina

**Versione adottata:** pnpm 10.30.2 — ultima versione compatibile con Node.js 18+ (pnpm 11 richiede Node 22).

**Alternative valutate:**
- npm: scartato — storico problematico nel team
- yarn v1: scartato — legacy, non più sviluppato attivamente
- yarn Berry: scartato — plug'n'play crea incompatibilità con alcuni tool
- bun: scartato — ecosistema non sufficientemente maturo per produzione

---

## DTR-018 — ESLint: flat config con eslint.config.mjs

**Data:** 2026-06-04
**Stato:** Accettata

**Decisione:** Usare ESLint 9 con flat config in formato `.mjs` (ES Module) invece del formato legacy `.eslintrc.json`.

**Motivazione:**
- ESLint 9 non supporta più `.eslintrc.*` — migrazione obbligatoria
- `.mjs` permette di usare `import/export` nel file di configurazione indipendentemente dal `"type"` in `package.json`
- Evita di dover aggiungere `"type": "module"` al `package.json`, che richiederebbe estensioni `.js` esplicite in tutti gli import TypeScript — comportamento considerato inaccettabile dal team

**Regola generale:** CommonJS (`require`/`module.exports`) è vietato nel codebase. Se un tool specifico lo richiede obbligatoriamente, si usa l'estensione `.cjs` per quel singolo file (convenzione già adottata in ToDoList per la config di StoryBook).

---

## DTR-019 — Statistiche aggregate discovery: rinviate a M11+

**Data:** 2026-06-08
**Stato:** Rinviata (M11+)

**Decisione:** Nessuna struttura di statistiche aggregate (per categoria, città, totali storici) viene implementata in `seen_firms.json` o altrove prima di M11+.

**Motivazione:**
- Le statistiche per categoria e città richiedono aggregazione dinamica — non modellabile in modo pulito in un JSON flat
- Qualsiasi struttura inventata ora in JSON sarebbe un workaround da buttare alla migrazione SQLite
- Il principio è: zero è meglio di sbagliato

**Implementazione futura (M11+):**
- SQLite con `better-sqlite3` come sorgente primaria
- Statistiche tramite query aggregate sul database
- `seen_firms.json` rimane come fallback offline, senza campi statistici

---

## DTR-020 — `first_seen` nel modello Business

**Data:** 2026-06-08
**Stato:** Accettata — implementata in M3

**Decisione:** Il campo `first_seen` (ISO timestamp) viene aggiunto al modello `Business` come campo opzionale. Viene popolato da `DedupService.markSeen()` al momento della prima rilevazione. Il `GoogleFetcher` non lo conosce e lo lascia `undefined`.

**Motivazione:**
- Le informazioni sull'azienda devono stare nel modello `Business`, non duplicate in `seen_firms.json`
- `first_seen` è un attributo dell'azienda, non del sistema di deduplica
- Disponibile nativamente per M11+ quando i dati migreranno su SQLite

---

## DTR-021 — `seen_firms.json`: responsabilità unica

**Data:** 2026-06-08
**Stato:** Accettata

**Decisione:** `seen_firms.json` ha una sola responsabilità: tenere l'elenco dei `place_id` già inviati. Nessun altro dato vive in questo file — niente `first_seen`, niente categorie, niente statistiche, niente contatori.

**Struttura finale:**
```json
{
  "seen": ["ChIJ...", "ChIJ..."],
  "last_reset": "2026-06-04T08:00:00Z"
}
```

**Motivazione:**
- Separazione netta delle responsabilità: deduplica nel file, dati aziendali nel modello, statistiche in M11+
- Nessuna duplicazione di informazioni già presenti in `Business`
- File minimo, veloce, senza rischio di inconsistenza

---

## DTR-022 — `last_seen` nel modello Business

**Data:** 2026-06-10
**Stato:** Accettata — implementata in M3

**Decisione:** Il campo `last_seen` (ISO timestamp) viene aggiunto al modello `Business` come campo opzionale, in parallelo a `first_seen` (DTR-020). Viene aggiornato da `DedupService.markSeen()` ad ogni esecuzione in cui il business è presente nel batch inviato.

**Motivazione:**
- `first_seen` traccia la prima rilevazione — invariante nel tempo
- `last_seen` traccia l'ultima rilevazione — aggiornato ad ogni run
- La distinzione è necessaria per analisi di frequenza e rilevanza dei lead (es. "questo studio dentistico appare ogni settimana")
- Entrambi i campi saranno colonne native nella tabella SQLite di M11+, senza necessità di migrazione dati

**Comportamento in `markSeen()`:**
- `first_seen`: scritto una volta, mai sovrascritto — se già presente sull'oggetto, viene preservato
- `last_seen`: sempre aggiornato al timestamp corrente dell'esecuzione

**Alternativa valutata e scartata:**
- `last_updated` (aggiornamento dei metadati del business): scartato — non ha ancora un produttore nel sistema. Territory di M10+, aggiunta ora sarebbe architettura speculativa (stesso principio di DTR-002 su `IFetcher`).

---

## DTR-023 — `DedupService`: path iniettabile nel costruttore

**Data:** 2026-06-10
**Stato:** Accettata — implementata in M3

**Decisione:** `DedupService` accetta un path opzionale nel costruttore per il file `seen_firms.json`. Il default è `path.resolve(process.cwd(), 'seen_firms.json')`.

```typescript
constructor(filePath = path.resolve(process.cwd(), 'seen_firms.json'))
```

**Motivazione:**
- Stesso pattern adottato da `loadConfig(configPath?)` in M1 — coerenza nell'approccio alla testability
- Permette ai test di usare directory temporanee isolate (`os.tmpdir()`) senza mock del filesystem
- Mock di `fs` nasconde bug reali di serializzazione JSON — file reali sono più affidabili
- Zero impatto sull'utilizzo in produzione: il default copre il caso nominale

---

## DTR-024 — `DedupService`: lazy loading e scrittura sincrona

**Data:** 2026-06-10
**Stato:** Accettata — implementata in M3

**Decisione:** `DedupService` carica `seen_firms.json` in modo lazy (al primo utilizzo, non nel costruttore) e scrive su disco in modo sincrono (`fs.writeFileSync`).

**Motivazione lazy loading:**
- Il file potrebbe non esistere alla costruzione dell'istanza (prima esecuzione)
- Il boot del processo non deve fallire per un file mancante
- La struttura vuota viene creata automaticamente al primo `markSeen()` o `reset()`

**Motivazione scrittura sincrona:**
- Il file è piccolo (< 1 MB anche dopo anni di utilizzo — solo array di stringhe)
- La sincronia garantisce che i `place_id` siano su disco anche se il processo viene terminato immediatamente dopo la scrittura
- La pipeline chiama `markSeen()` come ultima operazione dopo l'invio email — la latenza aggiuntiva è trascurabile

**Recovery da file corrotto:**
- JSON malformato o struttura non valida (`seen` non è un array) → warning su logger, riparte da struttura vuota
- I dati successivi vengono scritti correttamente — nessuna perdita di stato futuro

---

## DTR-025 — `excel.json`: configurazione colonne e sheet

**Data:** 2026-06-10
**Stato:** Accettata — da implementare in M4

**Decisione:** Il comportamento dell'export Excel è governato da un file di configurazione dedicato `excel.json`, separato da `config.json`. Definisce quali colonne sono visibili e quali sheet opzionali sono abilitati.

**Struttura:**
```json
{
  "columns": {
    "place_id":     false,
    "name":         true,
    "category":     true,
    "city":         true,
    "country":      true,
    "address":      true,
    "phone":        true,
    "website":      true,
    "rating":       true,
    "review_count": true,
    "maps_url":     false,
    "first_seen":   false
  },
  "include_email_templates": true
}
```

**Campi obbligatori (non disabilitabili):** `name`, `category`, `city`, `address`. Se un campo obbligatorio viene impostato a `false`, `ExcelExporter` lancia un errore descrittivo invece di generare un file privo di significato.

**Ordine colonne:** l'ordine delle chiavi in `columns` determina l'ordine delle colonne nel foglio — nessun hardcode nell'exporter.

**Motivazione:**
- Onur deve poter nascondere colonne tecniche (`place_id`, `maps_url`) senza modificare il codice
- La configurabilità via file evita di dover rilasciare una nuova versione per cambiare il layout
- La separazione da `config.json` mantiene la configurazione operativa distinta da quella di presentazione
- Coerente con il principio di responsabilità unica già applicato a `seen_firms.json` (DTR-021)

**Alternative valutate:**
- Parametri nel costruttore di `ExcelExporter`: scartato — non persistibile, non modificabile dall'utente finale
- Colonne sempre tutte presenti: scartato — `place_id` e `maps_url` sono rumore per un utente non tecnico

---

## DTR-026 — Sheet 2 (Email Templates): feature configurabile e seme del pannello email M12

**Data:** 2026-06-10
**Stato:** Accettata — da implementare in M4

**Decisione:** Lo Sheet 2 "Email Templates" è una feature opzionale controllata da `excel.json → include_email_templates`. Genera una riga per ogni azienda con oggetto e corpo email precompilati tramite Handlebars, pronti da copiare nel client email dell'utente.

**Struttura colonne Sheet 2:**
```
name | category | city | email_subject | email_body
```
`name`, `category`, `city` sono colonne di contesto — identificano l'azienda senza dover saltare tra fogli. Non si replicano tutti i campi dello Sheet 1.

**Template via Handlebars — chiavi i18n:**
```json
"email_subject_template": "Partnership opportunity — {{name}}",
"email_body_template":    "Dear {{name}},\n\nI noticed your business in {{city}} and would love to connect.\n\nBest regards"
```
Variabili disponibili: tutti i campi di `Business` — `{{name}}`, `{{city}}`, `{{category}}`, `{{address}}`, `{{website}}`.

**Testo multiriga:** `wrapText: true` sull'alignment della colonna `email_body` — il `\n` nel template diventa a-capo visibile in Excel.

**Connessione con la roadmap:**
Questo sheet è il seme del pannello di gestione email di M12. I template già strutturati in M4 eviteranno un refactor quando in M12 si aggiungerà l'invio diretto con tracciamento stato (inviata / risposto / ignorata).

**Motivazione:**
- Strumento di produttività immediata per Onur senza alcuna automazione — zero rischio
- Configurabile: chi non lo vuole lo disabilita in `excel.json`
- Anticipa la struttura dati di M12 a costo zero

---

## DTR-027 — `translate.ts`: utility i18n condivisa con fallback a cascata

**Data:** 2026-06-10
**Stato:** Accettata — da implementare in M4

**Decisione:** La funzione di traduzione viene estratta in un modulo condiviso `src/i18n/translate.ts`, usato da tutti i moduli che necessitano di stringhe localizzate (`ExcelExporter`, `MailService`, `Scheduler`).

**Implementazione:**
```typescript
export function t(
  key: string,
  i18n: Record<string, string>,
  fallback: Record<string, string>
): string {
  return i18n[key] ?? fallback[key] ?? key;
}
```

**Tre livelli di fallback:**
1. Lingua attiva (`i18n`)
2. `en.json` come fallback universale (`fallback`)
3. La chiave stessa come ultima istanza — almeno si vede cosa manca invece di stringa vuota

**Motivazione:**
- Evita la duplicazione della logica di fallback in ogni modulo
- Il terzo livello (chiave come fallback) rende immediatamente visibili le chiavi mancanti in fase di sviluppo e QA
- Coerente con il principio DRY — una sola implementazione, testata una volta sola
- Prerequisito per M8 (i18n completa) dove verrà aggiunto un test automatico che verifica la presenza di ogni chiave in ogni file lingua

**Caricamento i18n:** ogni modulo riceve il dizionario già caricato come parametro — `translate.ts` non carica file, solo traduce. Responsabilità di carico separata, testabilità massima.

---

## DTR-028 — Roadmap post v1.0.0 approvata

**Data:** 2026-06-10
**Stato:** Accettata (decisione di business e architettura)

**Decisione:** La roadmap delle milestone successive alla prima consegna pubblica (v1.0.0) è approvata nella seguente struttura:

**v1.0.0 — Consegna a Onur + pubblicazione Softonic**

| Milestone | Contenuto | Stato |
|---|---|---|
| M1 | Scaffold e configurazione | ✅ Completata |
| M2 | Google Places Fetcher | ✅ Completata |
| M3 | DedupService | ✅ Completata |
| M4 | Excel Export | 🔧 In sviluppo |
| M5 | MailService | ⏳ |
| M6 | Scheduler + daemon | ⏳ |

**v1.1.0 — Primo aggiornamento post-lancio**

| Milestone | Contenuto |
|---|---|
| M7 | REST API — bridge Node.js ↔ GUI JavaFX (Express porta 3847) |
| M8 | i18n completa — verifica tutte le chiavi in tutte le lingue, test automatici |
| M9 | Packaging e documentazione — README, CHANGELOG, release Softonic |

**v1.2.0**

| Milestone | Contenuto |
|---|---|
| M10 | GUI JavaFX + ForgeUI — dashboard, wizard installazione, form config |

**v1.3.0**

| Milestone | Contenuto |
|---|---|
| M11 | SQLite + storico discovery — migrazione da `seen_firms.json`, statistiche, vista storica GUI |

**v1.4.0**

| Milestone | Contenuto |
|---|---|
| M12 | Pannello gestione email — selezione aziende, invio diretto, tracciamento stato (prerequisito: M10 + M11) |

**v2.0.0**

| Milestone | Contenuto |
|---|---|
| M13 | Arricchimento dati — popolamento `Metadata[]` con email, LinkedIn, ad_budget da sorgenti esterne |
| M14 | Multi-utente e licenza Pro — gestione account, attivazione freemium €3,99/mese dal mese 2 |

**Motivazione:**
- La roadmap sequenzia correttamente le dipendenze architetturali: M7 (REST API) prerequisito di M10 (GUI), M10 + M11 prerequisiti di M12 (pannello email)
- Ogni versione è autonomamente rilasciabile e consegnabile
- Il Sheet 2 di M4 (DTR-026) è il seme strutturale di M12 — anticipazione a costo zero
- I `Metadata[]` di M2 (DTR-009) sono il seme strutturale di M13 — stessa filosofia

---

## DTR-029 — Policy: file di configurazione separati per dominio

**Data:** 2026-06-10
**Stato:** Accettata — policy trasversale a tutte le milestone

**Decisione:** Ogni dominio funzionale che richiede configurazione persistente ha il proprio file dedicato. Nessun file di configurazione aggrega responsabilità di domini diversi.

**File di configurazione del progetto:**

| File | Dominio | Gitignored |
|---|---|---|
| `config.json` | Configurazione operativa (API key, SMTP, schedule, lingue) | ✅ |
| `config.example.json` | Template committato senza segreti | ❌ |
| `excel.json` | Configurazione presentazione export Excel (colonne, sheet) | ❌ |
| `seen_firms.json` | Storico deduplicazione (place_id già inviati) | ✅ |

**Regola operativa:** quando emerge la necessità di configurare un nuovo dominio funzionale, si crea un file dedicato — non si estende `config.json`. Il nuovo file segue il pattern: file example committato, file reale gitignored se contiene dati sensibili o di stato.

**Motivazione:**
- Separazione netta delle responsabilità — ogni file ha una sola ragione per cambiare
- `config.json` rimane leggibile e non cresce indefinitamente
- File separati possono essere versionati, committati o gitignored in modo indipendente in base alla loro natura
- Coerente con le scelte già prese: `seen_firms.json` separato da `config.json` (DTR-021), `excel.json` separato da `config.json` (DTR-025)
- Facilita il wizard M10+: ogni pannello di configurazione della GUI mappa su un file specifico

---

## DTR-030 — Separazione codice/asset: `assets/i18n/` per le localizzazioni

**Data:** 2026-06-10
**Stato:** Accettata — implementata in M4

**Decisione:** I file di localizzazione JSON (`en.json`, `it.json`, …) risiedono in `assets/i18n/`. Il codice di traduzione (`translate.ts`) rimane in `src/i18n/`. La cartella `assets/` raccoglie tutto ciò che non è codice TypeScript né configurazione di root.

**Struttura:**
```
assets/
└── i18n/
    ├── en.json   ← lingua di riferimento e fallback universale
    ├── it.json
    ├── tr.json
    └── ...       ← tutti i file lingua

src/
└── i18n/
    └── translate.ts   ← codice — utility t() con fallback a cascata
```

**Motivazione:**
- Principio trasversale consolidato nel web development: localizzazioni, immagini, CSS e tutto ciò che non è codice o configurazione appartiene ad `assets/`
- Separazione netta tra artefatti compilabili (`src/`) e risorse statiche (`assets/`)
- Facilita la sostituzione o l'aggiornamento di un file lingua senza toccare il codice
- Coerente con la convenzione già adottata per `templates/email.html`

**Regola operativa:** qualsiasi risorsa statica non-codice introdotta nelle milestone successive segue lo stesso principio e va posizionata sotto `assets/`.

---

## DTR-031 — Import dinamici JSON: uso esplicito di `.default`

**Data:** 2026-06-10
**Stato:** Accettata — lezione appresa in M4

**Decisione:** Gli import dinamici di file JSON (`await import('…/file.json')`) devono sempre accedere alla proprietà `.default` per estrarre il contenuto tipizzato correttamente:

```typescript
// Corretto
const en = (await import('../../assets/i18n/en.json')).default;

// Errato — genera ts(2352): proprietà 'default' incompatibile con Record<string, string>
const en = await import('../../assets/i18n/en.json');
```

**Motivazione:**
- TypeScript con `esModuleInterop: true` e `resolveJsonModule: true` wrappa il contenuto JSON in un oggetto con chiave `default`
- Senza `.default` il tipo risultante include la proprietà `default` extra, incompatibile con `Record<string, string>` e simili indici generici
- Il compilatore segnala l'errore come `ts(2352)` — rilevato in M4 sui test di `translate.test.ts`

**Aggiornamento DTR-008 (Testing):** i test che importano dinamicamente file JSON devono usare `.default`. Lezione aggiunta alla nota sulle pratiche Vitest.

---

## DTR-032 — ExcelExporter: colonne configurabili con ordine da `excel.json`

**Data:** 2026-06-10
**Stato:** Accettata — implementata in M4

**Decisione:** Le colonne del Sheet 1 sono determinate esclusivamente da `excel.json`. L'ordine delle chiavi nel JSON determina l'ordine delle colonne nel foglio. Nessuna lista di colonne è hardcodata in `ExcelExporter.ts`.

**Implementazione:**
```typescript
const activeCols = Object.entries(this.excelConfig.columns)
  .filter(([, enabled]) => enabled)
  .map(([key]) => key);
```

**Motivazione:**
- Onur può riordinare le colonne modificando `excel.json` senza toccare il codice
- `Object.entries()` preserva l'ordine di inserimento delle chiavi — comportamento garantito in ES2015+
- La mappatura `colKey → fieldName` e `colKey → i18nKey` è dichiarativa e centralizzata nelle costanti `COL_TO_FIELD` e `COL_TO_I18N_KEY`

---

## DTR-033 — ExcelExporter: auto-width manuale post-inserimento

**Data:** 2026-06-10
**Stato:** Accettata — implementata in M4

**Decisione:** ExcelJS non dispone di un auto-fit nativo. Le larghezze delle colonne vengono calcolate manualmente dopo l'inserimento di tutti i dati, con una passata finale su ogni colonna:

```typescript
col.width = Math.max(MIN_COL_WIDTH, headerLen, maxDataLen) + COL_PADDING;
```

**Parametri:**
- `MIN_COL_WIDTH = 10` — larghezza minima garantita per colonne con dati brevi o vuoti
- `COL_PADDING = 2` — margine visivo aggiuntivo
- Il calcolo considera header e tutti i valori dati — prende il massimo

**Motivazione:**
- Il calcolo deve avvenire dopo l'inserimento dei dati — non è possibile farlo in anticipo
- La larghezza minima evita colonne troppo strette su dati opzionali (es. `phone`, `website`)
- Approccio consolidato nell'ecosistema ExcelJS — nessuna libreria di terze parti necessaria

---

## DTR-034 — ExcelExporter: Handlebars compile una volta per template

**Data:** 2026-06-10
**Stato:** Accettata — implementata in M4

**Decisione:** I template Handlebars per Sheet 2 vengono compilati una sola volta prima del loop sui business, non ad ogni iterazione:

```typescript
// Corretto — compilazione fuori dal loop
const subjectTemplate = Handlebars.compile(tr('email_subject_template'));
const bodyTemplate    = Handlebars.compile(tr('email_body_template'));

businesses.forEach(business => {
  const subject = subjectTemplate(business);
  const body    = bodyTemplate(business);
  // ...
});

// Errato — compilazione dentro il loop, O(n) compilazioni inutili
businesses.forEach(business => {
  const subject = Handlebars.compile(tr('email_subject_template'))(business);
});
```

**Motivazione:**
- `Handlebars.compile()` esegue il parsing e la compilazione del template — operazione non banale
- Con N aziende nel batch, compilare dentro il loop moltiplica il costo per N senza alcun beneficio
- Il template non cambia tra un business e l'altro — la funzione compilata è riutilizzabile

---

## DTR-035 — MailService: primo destinatario in `to`, tutti gli altri in `bcc`

**Data:** 2026-06-11
**Stato:** Accettata — da implementare in M5

**Decisione:** `MailService` invia un'unica email con il primo indirizzo di `config.email_to` in `to` e tutti gli altri in `bcc`. I destinatari non si vedono tra loro.

**Implementazione:**
```typescript
to:  recipients[0],
bcc: recipients.slice(1).join(', '),
```
Se `email_to` ha un solo destinatario, `bcc` è stringa vuota — Nodemailer lo gestisce correttamente senza errori.

**Motivazione:**
- Protegge la privacy dei destinatari — default più sicuro per uno strumento distribuito a terzi
- SpotCast può essere rivenduto o usato in team — esporre gli indirizzi tra loro è inaccettabile
- Zero complessità aggiuntiva rispetto all'alternativa `to` multiplo
- Comportamento da documentare nel README utente — non ovvio per chi configura `email_to`

**Alternative valutate:**
- Tutti in `to`: scartato — espone gli indirizzi tra i destinatari
- Email separata per destinatario: scartato — overkill, N chiamate SMTP invece di una

---

## DTR-036 — MailService: wrapping errori SMTP asincroni

**Data:** 2026-06-11
**Stato:** Accettata — da implementare in M5

**Decisione:** `MailService.send()` wrappa `transporter.sendMail()` in `try/catch`, logga l'errore con Winston, e **rilancia** — non decide autonomamente se il processo deve terminare.

```typescript
try {
  await transporter.sendMail(mailOptions);
  logger.info(`Email sent to ${config.email_to.length} recipient(s)`);
} catch (err) {
  logger.error(`SMTP error: ${(err as Error).message}`);
  throw err; // rilancia — la pipeline in M6 decide
}
```

**Motivazione:**
- `sendMail()` restituisce una Promise — l'errore è asincrono. Senza `try/catch` diventa `UnhandledPromiseRejection` che in Node.js 20+ termina il processo senza log utile
- `MailService` è stupido per design — sa solo mandare email. Non ha il contesto per decidere se un errore SMTP è fatale per il run
- La pipeline in M6 cattura il `throw` e decide: logga il fallimento del run, ma il processo daemon rimane attivo per il run successivo

**Contratto:** logga e rilancia. Non inghiotte mai.

---

## DTR-037 — i18n: utility `format.ts` per date e numeri localizzati

**Data:** 2026-06-11
**Stato:** Accettata — da implementare in M5 (prerequisito bloccante)

**Decisione:** La formattazione di date e numeri secondo la lingua attiva è estratta in `src/i18n/format.ts`, con tre funzioni separate per tre casi d'uso distinti:

```typescript
formatDate(date: Date, i18n: Record<string, unknown>): string
// Legge i18n.formats.date — es. "DD/MM/YYYY" per IT, "YYYY年MM月DD日" per ZH

formatInteger(n: number, i18n: Record<string, unknown>): string
// Usa thousands_separator — per count, totali, contatori
// Output: "1.234" (IT/DE) | "1,234" (EN) | "1 234" (FR)
// Nessun decimale — non ha senso dire "trovate 32.00 nuove attività"

formatDecimal(n: number, i18n: Record<string, unknown>): string
// Usa decimal_separator e thousands_separator — per rating, valori float
// Output: "4,5" (IT) | "4.5" (EN)
// Rimuove zero trailing — "4.0" → "4"
```

**Fallback:** se `i18n.formats` è assente o incompleto, comportamento EN di default (`YYYY-MM-DD`, `.` decimale, `,` migliaia).

**Motivazione:**
- Senza formattazione localizzata l'email di Onur mostra "10/06/2026" in italiano e "2026-06-10" in inglese — incoerente
- La separazione `formatInteger` / `formatDecimal` evita output assurdi come "trovate 32.00 nuove attività"
- `format.ts` è la naturale estensione di `translate.ts` — stesso modulo `src/i18n/`, stessa filosofia (riceve dizionario già caricato, non tocca il filesystem)

**Nota `formats` come oggetto:** `i18n.formats` è un oggetto annidato, non una stringa — `translate.ts` non lo gestisce. `format.ts` lo legge direttamente tramite accesso tipizzato.

---

## DTR-038 — MailService: template HTML con Handlebars, `\n` → `<br>` sul body i18n

**Data:** 2026-06-11
**Stato:** Accettata — da implementare in M5

**Decisione:** Il corpo email è composto da due layer separati:

1. **Body testuale** (da i18n): stringa `email_body` compilata con Handlebars (`{{date}}`, `{{count}}`, `{{categories}}`, `{{cities}}`), poi `\n` sostituiti con `<br>` — produce HTML inline leggibile
2. **Wrapper HTML** (`templates/email.html`): file statico con struttura visiva (font, colori, spaziatura), riceve `{{{body}}}` come variabile Handlebars (triple-stache per non escappare l'HTML già prodotto)

**Variabili disponibili nel wrapper:**
```
{{date}}        — data run formattata con formatDate()
{{count}}       — numero aziende formattato con formatInteger()
{{categories}}  — join ", " delle categorie dalla config
{{cities}}      — join ", " delle città dalla config
{{{body}}}      — corpo i18n con <br>, iniettato senza escaping
```

**Motivazione:**
- Separazione contenuto/presentazione: il testo cambia per lingua, il wrapper HTML è invariante
- `\n` → `<br>` applicato al body i18n, non nel template — mantiene i file JSON leggibili come testo normale
- Triple-stache `{{{body}}}` necessario perché il body contiene già tag `<br>` — il double-stache `{{body}}` li escaperebbe in `&lt;br&gt;`
- Handlebars è già nel progetto (M4 ExcelExporter) — zero dipendenze nuove

**Alternativa scartata — MJML:** overkill per un'email transazionale semplice con tre variabili. Risolve problemi di compatibilità cross-client che SpotCast non ha (Onur usa probabilmente sempre lo stesso client email).

**Alternativa scartata — React Email:** introduce React come dipendenza in un backend Node.js puro. Il costo è alto, il beneficio è zero per questo caso d'uso.

---

## DTR-039 — i18n: tipo `Record<string, unknown>` per dizionari con oggetti annidati

**Data:** 2026-06-11
**Stato:** Accettata — lezione appresa in M5

**Decisione:** I dizionari i18n devono essere tipizzati come `Record<string, unknown>` in tutta la codebase — non `Record<string, string>`. Dove il valore è certamente una stringa, si usa il cast esplicito `as string`.

```typescript
// Corretto
export function t(
  key: string,
  i18n: Record<string, unknown>,
  fallback: Record<string, unknown>
): string {
  return (i18n[key] as string) || (fallback[key] as string) || key;
}

// Errato — incompatibile con oggetti annidati come formats
export function t(key: string, i18n: Record<string, string>, ...): string
```

**Motivazione:**
- L'aggiunta di `formats` come oggetto annidato in M4 ha reso `Record<string, string>` incompatibile con la struttura reale dei file i18n
- TypeScript segnala `ts(2352)` quando si tenta di castare un tipo con proprietà non-stringa a `Record<string, string>`
- `Record<string, unknown>` è il tipo corretto per qualsiasi dizionario JSON con struttura eterogenea
- Il cast `as string` nei punti di utilizzo è esplicito e controllato — non nasconde bug

**Impatto:** aggiornati `translate.ts`, `translate.test.ts` e tutti i punti di utilizzo.

---

## DTR-040 — Testing: variabili fixture che collidono con funzioni Vitest

**Data:** 2026-06-11
**Stato:** Accettata — lezione appresa in M5

**Decisione:** Le variabili fixture nei test non devono avere lo stesso nome delle funzioni importate da Vitest (`it`, `describe`, `expect`, `vi`, `beforeEach`, `afterEach`). Convenzione adottata: suffisso `_` per le fixture che collidono.

```typescript
// Errato — 'it' è una funzione Vitest, non una variabile
const it = { formats: { date: 'DD/MM/YYYY' } };

// Corretto
const it_ = { formats: { date: 'DD/MM/YYYY' } };
```

**Motivazione:**
- Vitest importa `it` come funzione globale quando `globals: true` è attivo in `vitest.config.ts`
- Una variabile locale con lo stesso nome maschera la funzione globale — i test falliscono con `TypeError: it is not a function`
- Il suffisso `_` è una convenzione consolidata in TypeScript per evitare conflitti con keyword e identifier riservati

**Aggiornamento DTR-008 (Testing):** aggiunta lezione su naming delle fixture.

---

## DTR-041 — jsonTransport Nodemailer: struttura payload per le asserzioni di test

**Data:** 2026-06-11
**Stato:** Accettata — lezione appresa in M5

**Decisione:** `jsonTransport` di Nodemailer serializza gli indirizzi come oggetti `{address, name}` e gli allegati come `content` (buffer), non come `path`. I test devono assertire sulla struttura reale del payload, non su rappresentazioni stringa.

**Struttura payload jsonTransport:**
```typescript
// Indirizzi — NON stringhe flat
msg.to   // → Array<{ address: string; name: string }>
msg.bcc  // → Array<{ address: string; name: string }>
msg.from // → { address: string; name: string }

// Allegati — content risolto, path non presente
msg.attachments // → Array<{ filename: string; content: Buffer }>

// Accesso corretto
const to = msg.to as Array<{ address: string }>;
expect(to[0].address).toBe('primary@test.com');

// Accesso errato — String() su un oggetto produce '[object Object]'
expect(String(msg.to)).toContain('primary@test.com'); // FALLISCE
```

**Motivazione:**
- Il formato è documentato nel codice sorgente di Nodemailer ma non esplicitamente nella documentazione pubblica
- Scoperto empiricamente in M5 — 4 test falliti al primo run per questo motivo
- La conoscenza è ora tracciata per evitare lo stesso problema in future suite che usano `jsonTransport`
 ---
## DTR-042 — Sostituzione GoogleFetcher con HereFetcher (HERE Browse API)

**Data:** 2026-06-13
**Stato:** Accettata — implementata in M7

**Decisione:** `GoogleFetcher` viene deprecato e sostituito da `HereFetcher` basato su HERE Browse API. `GoogleFetcher.ts` rimane nel repository come riferimento storico ma non viene più istanziato dalla pipeline.

**Motivazione:**
- Google Places Text Search ha un limite strutturale di 20 risultati per query, 60 con paginazione — non aggirabile lato client
- In test end-to-end su Tolmezzo/Socchieve con `results_per_run: 10` globale, la prima categoria esauriva il budget nascondendo le altre
- HERE Browse API supporta paginazione reale con `offset`, nessun limite artificiale per query
- HERE ha copertura capillare Europa con dati stabili garantiti da contratti enterprise
- Approccio geografico (lat/lng + raggio + codice categoria) più preciso della query testuale
- Architettura compatibile con Overture Maps in M14+ — stesso modello dati

**Impatto:** solo `SpotCast.ts` cambia la riga di import. Il resto della pipeline è invariato.

**Alternative valutate:**
- Paginazione Google Places con `next_page_token`: cap di 60 risultati totali rimane, delay obbligatorio tra le pagine, costo API triplicato — scartato
- Selenium/Playwright su Google Search: fattibilità stimata 40%, manutenzione continua contro anti-bot Google, guerra di logoramento — scartato

---

## DTR-043 — `categories` in config.json: label testuali inglesi → codici HERE tramite HereCategoryMap

**Data:** 2026-06-13
**Stato:** Accettata — implementata in M7

**Decisione:** Le categorie in `config.json` rimangono stringhe testuali in inglese (`"Bar"`, `"Gym"`, `"Lawyer"`). La conversione verso i codici HERE avviene in `ConfigLoader.ts` tramite `HereCategoryMap.ts` prima che i dati raggiungano `HereFetcher`. `HereFetcher` riceve solo codici già validati — non conosce le label.

**Struttura `HereCategoryMap.ts`:**
```typescript
export const HERE_CATEGORY_MAP: Record<string, string> = {
  "Bar":         "100-1000-0000",
  "Restaurant":  "100-1100-0000",
  "Gym":         "400-4100-0141",
  "Lawyer":      "700-7400-0246",
  "Plumber":     "700-7400-0249",
  "Locksmith":   "700-7400-0116",
  // estensibile — aggiungere una categoria = aggiungere una riga
};
```

**Comportamento su categoria non riconosciuta:** WARN nel log + skip della categoria. Se tutte le categorie sono invalide → errore fatale con `fatal()` e exit code 1.

**Motivazione:**
- L'utente finale non deve conoscere codici numerici HERE
- Responsabilità separata: `ConfigLoader` valida e traduce, `HereFetcher` esegue
- Estensibile senza modificare `HereFetcher`

**Roadmap i18n (post-M7):** le label in config resteranno in inglese come lingua comune. A tendere, la pipeline tradurrà le label i18n attive in inglese prima della conversione — nessuna modifica alla mappa necessaria.

---

## DTR-044 — `results_per_run` rimosso, paginazione illimitata con HERE

**Data:** 2026-06-13
**Stato:** Accettata — breaking change in M7

**Decisione:** Il campo `results_per_run` viene rimosso da `config.json` e da `ConfigLoader`. `HereFetcher` pagina automaticamente fino ad esaurimento dei risultati HERE per ogni combinazione categoria×città.

**Motivazione:**
- `results_per_run` come cap globale nascondeva categorie: la prima categoria esauriva il budget prima che le altre venissero interrogate — problema riscontrato in test end-to-end M6
- HERE Browse API non ha limiti arbitrari — la paginazione è reale e deterministica
- L'utente finale vuole tutte le attività disponibili, non un sottoinsieme arbitrario

**Breaking change:** `config.json` esistenti con `results_per_run` producono un WARN al caricamento. Il campo viene ignorato silenziosamente dopo il warning.

---

## DTR-045 — Geocoding con cache su file `geocache.json`

**Data:** 2026-06-13
**Stato:** Accettata — implementata in M7

**Decisione:** Le coordinate geografiche delle città vengono cachate in `geocache.json` nella root del progetto. La cache viene consultata prima di ogni chiamata HERE Geocoding API. I risultati nuovi vengono scritti nella cache immediatamente dopo il fetch.

**Struttura:**
```json
{
  "Berlin, Germany":        { "lat": 52.5200, "lng": 13.4050 },
  "Munich, Germany":        { "lat": 48.1351, "lng": 11.5820 },
  "Niedernhausen, Germany": { "lat": 50.1731, "lng":  8.3197 }
}
```

**`geocache.json` viene committato nel repository.** Non contiene segreti. Pre-caricato con le principali città europee — migliora l'esperienza out-of-the-box.

**Motivazione:**
- Le coordinate geografiche non cambiano nel tempo
- Evita chiamate API ridondanti ad ogni run per le stesse città
- Il file cresce organicamente con le città usate dagli utenti

---

## DTR-046 — Roadmap M13: Handelsregister tedesco come sorgente lead B2B

**Data:** 2026-06-13
**Stato:** Pianificata (M13)

**Decisione:** In M13 verrà implementato un connettore per l'Handelsregister tedesco (registro imprese pubblico) come sorgente primaria di lead B2B per il mercato tedesco.

**Motivazione:**
- Ogni nuova GmbH registrata ha depositato almeno 25.000€ di capitale sociale — lead qualificati per definizione
- I dati sono pubblici, gratuiti e stabili — zero costi API, nessun rischio di ban
- SpotCast intercetta l'azienda nel momento esatto della nascita legale, settimane prima che abbia una scheda Google Maps o HERE
- Risponde esplicitamente al requisito di Benjamin: lead ad alta capitalizzazione appena emersi

**Architettura prevista:**
```
Feed giornaliero Handelsregister
  → parser iscrizioni nuove GmbH/UG
  → HERE Geocoding per coordinate
  → modello Business esistente
  → pipeline invariata (dedup → excel → email)
```

**Alternativa scartata — Selenium/Playwright su Google Search:** fattibilità 40%, manutenzione continua contro sistemi anti-bot Google, proxy rotativi ~20-50€/mese, insostenibile a lungo termine.

---
## DTR-047 — cities.json: struttura per paese con nomi geografici ufficiali

**Data:** 2026-06-13
**Stato:** Accettata — implementata in M7

**Decisione:** Le città non sono più definite in `config.json`. Un file separato `assets/cities/cities.json` contiene la lista delle città raggruppate per paese. Il campo `cities_file` in `config.json` indica il percorso del file.

**Struttura `cities.json`:**
```json
[
  {
    "country": "Germany",
    "cities": ["Berlin", "München", "Hamburg"]
  },
  {
    "country": "Italy",
    "cities": ["Roma", "Milano", "Tolmezzo"]
  }
]
```

**Regola sui nomi delle città:**
Il nome della città deve corrispondere al nome ufficiale sulla cartografia nazionale del paese. In paesi multilingua (es. Svizzera, Belgio), si usa il nome nella lingua parlata nella zona geografica dove la città si trova — es. "Genève" per Ginevra francofona, "Zürich" per Zurigo germanofona, "Lugano" per il Ticino italofono.

**Motivazione:**
- HERE Geocoding API restituisce risultati più precisi con il nome ufficiale locale
- Evita ambiguità su città omonime in paesi diversi
- Coerente con il comportamento atteso da un utente locale

**Breaking change in `config.json`:**
- Rimossi: `cities`, `countries`
- Aggiunto: `"cities_file": "assets/cities/cities.json"`

**`cities.json` è gitignored** — si committa `cities.example.json` con struttura vuota. L'utente copia, rinomina e popola con le proprie città.

---

## DTR-048 — CitiesLoader: flatten a stringa "city, country" per GeoCache

**Data:** 2026-06-13
**Stato:** Accettata — implementata in M7

**Decisione:** `CitiesLoader.ts` carica `cities.json`, valida con Zod e restituisce un array piatto di stringhe `"${city}, ${country}"` — la stessa chiave già usata da GeoCache. Nessuna interfaccia `CityEntry` intermedia necessaria.

```typescript
// Output di CitiesLoader.load():
[
  "Berlin, Germany",
  "München, Germany",
  "Roma, Italy",
  "Milano, Italy",
]
```

**Motivazione:**
- GeoCache usa già `"${city}, ${country}"` come chiave — zero modifiche
- Il flatten è un `flatMap` di tre righe — nessuna complessità aggiuntiva
- Evita un'interfaccia intermedia che non aggiunge valore

---

## DTR-049 — coverage_mode: selezione sorgente città (pianificata M11+)

**Data:** 2026-06-13
**Stato:** Pianificata (M11+)

**Decisione:** In M11+ verrà introdotto il campo `coverage_mode` in `config.json` per pilotare la sorgente delle città:

```json
"coverage_mode": "file"   // default — usa cities.json
"coverage_mode": "full"   // tutti i comuni del paese da SQLite
```

**Motivazione:**
- Benjamin richiede copertura nazionale (~11.000 comuni in Germania) — non gestibile con un file manuale
- SQLite (M11+) sarà la sorgente per la copertura completa
- `cities.json` rimane per utenti che vogliono controllo granulare
- `"full"` userà la tabella `cities` del database, filtrata per paese

**`cities.json` rimane valido** in tutti i casi come override esplicito — chi vuole un sottoinsieme specifico può sempre usarlo indipendentemente dal `coverage_mode`.
