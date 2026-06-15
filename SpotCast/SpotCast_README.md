# SpotCast 🎯

> Scoperta automatica di attività locali — trova nuovi lead su Google Maps ogni mattina, direttamente nella tua casella email.

**Repository:** [github.com/Belmani/SpotCast](https://github.com/Belmani/SpotCast)
**Product Owner:** Gabriela Belmani
**Tech Lead:** Alessandro De Prato
**Versione corrente:** 0.1.0
**Stato:** 🟢 In sviluppo attivo — M3 in pianificazione

---

## Descrizione

SpotCast è uno strumento di lead generation locale che ogni mattina cerca su Google Maps le attività corrispondenti alle categorie e città configurate, filtra i duplicati, esporta i risultati in Excel e li invia via email. È il primo prodotto commerciale del team, distribuito come freeware con piano Pro opzionale dal secondo mese.

**Cliente iniziale:** Onur — ha fornito lo script Python di riferimento (`google_maps_tracker.py`) da cui è partito il progetto.

**Target di distribuzione:** Softonic e piattaforme freeware equivalenti, con passaparola come canale primario di crescita.

---

## Stack tecnologico

| Layer | Tecnologia |
|---|---|
| Linguaggio | TypeScript 5 |
| Runtime | Node.js ≥20 |
| Package manager | pnpm 10.30.2 |
| Dati | Google Places API |
| Excel | ExcelJS |
| Email | Nodemailer |
| Scheduler | node-cron |
| Validazione config | Zod v3 |
| Logging | Winston |
| Testing | Vitest (copertura ≥80%) |
| Linting | ESLint 9 |
| GUI (futuro) | JavaFX + ForgeUI (M10+) |

---

## Milestone

| # | Nome | Stato | Note |
|---|---|---|---|
| M1 | Scaffold e Configurazione | ✅ Completa | Inclusa in v0.1.0 |
| M2 | Google Places Fetcher | ✅ Completa | Inclusa in v0.1.0 |
| M3 | Servizio di Deduplicazione | ⏳ Prossima | `DedupService` + `seen_firms.json` |
| M4 | Esportatore Excel | ⏳ | ExcelJS, 3 fogli, i18n |
| M5 | Servizio Email | ⏳ | Nodemailer + Handlebars |
| M6 | Scheduler | ⏳ | node-cron + modalità daemon |
| M7 | REST API | ⏳ | Bridge per GUI JavaFX |
| M8 | Internazionalizzazione | ⏳ | 10 lingue di lancio |
| M9 | Packaging e Documentazione | ⏳ | Preparazione release pubblica |
| M10+ | GUI JavaFX + Wizard | 🔮 Futuro | ForgeUI design system |
| M11+ | Database SQLite | 🔮 Futuro | Storico discovery |

**Deadline prima consegna (backend M1→M6):** venerdì

---

## Indice documenti

### Documentazione tecnica
- [DTR.md](SpotCast_DTR.md) — tutte le decisioni architetturali con motivazione
- [README.md](SpotCast_README_USER.md) — guida utente
- [README_DEV.md](SpotCast_README_DEV.md) — guida per sviluppatori

### Grooming
- [GRM_SpotCast_M1_M6.md](./grooming/GRM_SpotCast_M1_M6.md) — grooming completo M1→M6

### Milestone completate
- [MLS_SpotCast_M1.md](./milestones/MLS_SpotCast_M1.md) — Scaffold e Configurazione
- [MLS_SpotCast_M2.md](./milestones/MLS_SpotCast_M2.md) — Google Places Fetcher

### Tutorial
- [TTR_SpotCast_M1_GitHub.md](./tutorials/TTR_SpotCast_M1_GitHub.md) — configurazione repo GitHub
- [TTR_SpotCast_M1_Glossario_GIT.md](TTR_SpotCast_M1_Glossario_git.md) — glossario Git e GitHub
- [TTR_SpotCast_M1_Stack_Tecnico.md](./tutorials/TTR_SpotCast_M1_Stack_Tecnico.md) — Zod, Winston, ts-node, Vitest, ExcelJS, Handlebars, node-cron
- [TTR_SpotCast_M1_Google_API_Key.md](TTR_SpotCast_M1_Google_api_key.md) — setup Google Places API Key
- [TTR_SpotCast_M2_Vitest.md](./tutorials/TTR_SpotCast_M2_Vitest.md) — TDD con Vitest
- [TTR_SpotCast_M2_Pnpm.md](./tutorials/TTR_SpotCast_M2_Pnpm.md) — migrazione a pnpm

---

## Decisioni chiave

Le decisioni complete sono nel [DTR.md](SpotCast_DTR.md). Le più rilevanti per il P.O.:

- **TypeScript** invece di Python — stack del team, nessuna competenza Python (DTR-001)
- **Solo Google Places API** — i dati fittizi dello script originale non hanno valore in produzione (DTR-002)
- **Pricing freemium** — gratuito il primo mese, €4,99/mese dal secondo (DTR-014)
- **Distribuzione MIT** — open source su GitHub + Softonic (DTR-015)
- **10 lingue al lancio** — IT, EN, DE, FR, ES, PT, ZH, JA, AR, HI (DTR-011)
- **GUI JavaFX** con ForgeUI come progetto pilota del design system (DTR-012)
