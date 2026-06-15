# belmani-apex 📚

> Knowledge base operativa di Gabriela Belmani — Product Owner.
> Documentazione interna di progetto in italiano, ad uso del team di sviluppo.

---

## Cos'è questo vault

Belmani-apex raccoglie tutta la documentazione prodotta durante lo sviluppo dei progetti di cui Gabriela è Product Owner. Non è documentazione pubblica — quella vive nei repository di progetto. Questo è il posto dove il team si allinea, ragiona e costruisce memoria condivisa.

---

## Struttura trasversale

Ogni progetto ha la sua cartella con la seguente organizzazione standard:

```
belmani-apex/
└── <NomeProgetto>/
    ├── README.md          ← descrizione del progetto e indice dei documenti
    ├── docs/              ← DTR, README utente e developer (versione italiana)
    ├── grooming/          ← grooming delle milestone
    ├── milestones/        ← riepilogo di ogni milestone completata
    └── tutorials/         ← tutorial tecnici specifici per il progetto
```

---

## Convenzioni di naming

| Prefisso | Tipo documento          | Esempio                      |
| -------- | ----------------------- | ---------------------------- |
| `TTR_`   | Tutorial tecnico        | `TTR_SpotCast_M2_Vitest.md`  |
| `GRM_`   | Grooming milestone      | `GRM_SpotCast_M1.md`         |
| `MLS_`   | Riepilogo milestone     | `MLS_SpotCast_M2.md`         |
| `DTR`    | Decisioni tecniche      | `DTR.md`                     |
| `README` | Documentazione generale | `README.md`, `README_DEV.md` |

Il pattern dei file è sempre: `<SIGLA>_<Progetto>_<Milestone>_<Topic>.md`

---

## Progetti attivi

| Progetto                         | Descrizione                               | Stato                        |
| -------------------------------- | ----------------------------------------- | ---------------------------- |
| [SpotCast](SpotCast_README.md) | Lead generation automatica da Google Maps | 🟢 In sviluppo — v0.1.0      |
| NomadSync                        | Sincronizzazione cartelle via GitHub      | 🟡 Backend 95% — UI in avvio |
| ForgeUI                          | Design system JavaFX per Maven Central    | 🟡 M2 in corso               |

---

## Note operative

- La lingua di questo vault è **italiano** — lingua comune del team
- La documentazione pubblica di progetto (README, DTR, CONTRIBUTING) vive nei repository GitHub in inglese
- I tutorial qui presenti sono versioni estese e commentate — la versione sintetica per i contributor esterni sta in `docs/` nel repo

---

*Mantenuto da Gabriela Belmani — aggiornato ad ogni milestone completata.*
