# SpotCast 🎯

> Scoperta automatica di attività locali — trova nuovi contatti su Google Maps ogni mattina, direttamente nella tua casella di posta.

---

## Cos'è SpotCast?


SpotCast è uno strumento leggero e open-source che si avvia ogni mattina sul tuo computer e automaticamente:

1. Cerca su Google Maps le attività locali corrispondenti alle categorie e alle città che hai configurato
2. Filtra le aziende che hai già visto nelle sessioni precedenti
3. Esporta i risultati in un file Excel formattato
4. Invia il file alla tua casella di posta — pronto all'uso

Nessun abbonamento cloud. Nessuna dashboard a cui accedere. Solo un file nella tua posta ogni mattina.

---

## Requisiti

- **Node.js** v20 o superiore — [scarica qui](https://nodejs.org/)
- Una **chiave API Google Maps** (con Places API abilitata) — [vedi la guida dedicata](https://claude.ai/chat/docs/TUTORIAL_google_api_key.md)
- Un **account email SMTP** per l'invio (Gmail, Outlook o qualsiasi provider)
- macOS, Windows o Linux

---

## Installazione

> ⚠️ **M10+:** le versioni future di SpotCast includeranno un installer grafico con una procedura guidata che automatizza tutti questi passaggi. Per ora, segui le istruzioni seguenti — richiede meno di 5 minuti.

**1. Scarica SpotCast**

Vai alla pagina [Releases](https://github.com/Belmani/SpotCast/releases), scarica l'archivio dell'ultima versione ed estrailo in una cartella a tua scelta.

**2. Installa Node.js** (se non lo hai già)

Scaricalo da [nodejs.org](https://nodejs.org/) e installalo normalmente. Al termine, verifica l'installazione aprendo un terminale e digitando:

```bash
node --version
```

Dovresti vedere un numero di versione (es. `v20.11.0`). Se ottieni un errore, riavvia il computer e riprova.

**3. Installa pnpm** (se non lo hai già)

```bash
npm install -g pnpm@10.30.2
```

**4. Installa le dipendenze di SpotCast**

Apri un terminale, naviga nella cartella di SpotCast e digita:

```bash
pnpm install
```

> 💡 **Come aprire un terminale:**
> 
> - **Windows:** clic destro sulla cartella SpotCast → "Apri nel terminale"
> - **Mac:** clic destro sulla cartella SpotCast → "Nuovo terminale nella cartella"
> - **Linux:** clic destro sulla cartella → "Apri terminale"

**5. Copia il file di configurazione**

```bash
cp config.example.json config.json
```

Su Windows:

```
copy config.example.json config.json
```

**6. Configura SpotCast**

Apri `config.json` con qualsiasi editor di testo e inserisci i tuoi dati. Consulta la sezione [Configurazione](https://claude.ai/chat/ed3860c9-99a9-46f8-964e-95b402a6c65b#configurazione) qui sotto.

---

## Configurazione

```json
{
  "language": "en",
  "google_api_key": "LA_TUA_CHIAVE_GOOGLE_PLACES_API",
  "categories": ["Dentist", "Gym", "Lawyer", "Accountant", "Real Estate Agent"],
  "cities": ["Milan", "Rome", "Turin"],
  "countries": ["Italy"],
  "results_per_run": 10,
  "schedule": "0 8 * * *",
  "output_dir": "results",
  "smtp": {
    "host": "smtp.gmail.com",
    "port": 587,
    "user": "tua@email.com",
    "pass": "tua_password_app"
  },
  "email_to": ["destinatario@email.com"],
  "email_template": "templates/email.html"
}
```

### Campi principali

|Campo|Descrizione|Esempio|
|---|---|---|
|`language`|Lingua dell'output|`"en"`, `"it"`, `"de"`|
|`google_api_key`|La tua chiave API Google Maps|`"AIzaSy..."`|
|`categories`|Tipi di attività da cercare|`["Dentist", "Gym"]`|
|`cities`|Città in cui cercare|`["Milan", "Rome"]`|
|`countries`|Filtro per paese|`["Italy"]`|
|`results_per_run`|Nuove attività da trovare al giorno|`10`|
|`schedule`|Pianificazione cron|`"0 8 * * *"` = ogni giorno alle 8:00|
|`output_dir`|Cartella output Excel|`"results"`|
|`email_to`|Destinatari del report|`["tu@email.com"]`|

> 📧 **Nota Gmail:** usa una Password per le app, non la password del tuo account. Generane una su [myaccount.google.com/security](https://myaccount.google.com/security) alla voce "Password per le app".

---

## Avviare SpotCast

### Esecuzione manuale — una sola volta

```bash
node SpotCast.js
```

### Esecuzione automatica giornaliera

Scegli il metodo per il tuo sistema operativo:

#### 🍎 macOS — LaunchAgent

Crea `~/Library/LaunchAgents/com.spotcast.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>com.spotcast</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/local/bin/node</string>
    <string>/percorso/a/spotcast/SpotCast.js</string>
    <string>--daemon</string>
  </array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>StandardOutPath</key><string>/percorso/a/spotcast/tracker.log</string>
  <key>StandardErrorPath</key><string>/percorso/a/spotcast/tracker.log</string>
</dict>
</plist>
```

```bash
launchctl load ~/Library/LaunchAgents/com.spotcast.plist
```

#### 🪟 Windows — Utilità di pianificazione

1. Apri **Utilità di pianificazione** → **Crea attività di base**
2. Trigger: **All'avvio del computer**
3. Programma: `C:\Program Files\nodejs\node.exe`
4. Argomenti: `C:\percorso\a\spotcast\SpotCast.js --daemon`

#### 🐧 Linux — systemd

Crea `/etc/systemd/system/spotcast.service`:

```ini
[Unit]
Description=SpotCast Lead Tracker
After=network.target

[Service]
Type=simple
User=IL_TUO_USERNAME
WorkingDirectory=/percorso/a/spotcast
ExecStart=/usr/bin/node SpotCast.js --daemon
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable spotcast && sudo systemctl start spotcast
```

---

## Output

Ogni esecuzione produce un file Excel nella cartella `output_dir` con tre fogli:

- **Businesses** — Nome, categoria, città, indirizzo, telefono, sito web, valutazione, numero di recensioni
- **Email Templates** — Email di contatto pre-compilate per ogni attività
- **Run Summary** — Data, totale trovati, duplicati saltati, fonte dei dati

Il file viene inviato automaticamente a tutti gli indirizzi in `email_to`.

---

## Deduplicazione

SpotCast tiene traccia di ogni attività che ha mai trovato. Le attività già viste non vengono mai reinviate. Per azzerare:

```bash
node SpotCast.js --reset
```

---

## Lingue Supportate

| Codice | Lingua                |
| ------ | --------------------- |
| `it`   | Italiano              |
| `en`   | Inglese               |
| `de`   | Tedesco               |
| `fr`   | Francese              |
| `es`   | Spagnolo              |
| `pt`   | Portoghese            |
| `zh`   | Cinese (semplificato) |
| `ja`   | Giapponese            |
| `ar`   | Arabo                 |
| `tr`   | Turco                 |

---

## REST API

SpotCast espone una REST API locale su `http://localhost:3847`:

|Endpoint|Metodo|Descrizione|
|---|---|---|
|`/status`|GET|Stato attuale e info sull'ultima esecuzione|
|`/run`|POST|Avvia un'esecuzione manuale immediata|
|`/results`|GET|Risultati dell'ultima esecuzione in formato JSON|
|`/config`|GET/PUT|Leggi o aggiorna la configurazione|

---

## Risoluzione dei Problemi

**SpotCast si avvia ma non arriva nessuna email** → Controlla le credenziali SMTP in `config.json`. Gmail richiede una Password per le app.

**"google_api_key is required"** → Aggiungi la tua chiave API Google in `config.json`. Consulta la [guida di configurazione](https://claude.ai/chat/docs/TUTORIAL_google_api_key.md).

**"node: command not found"** → Reinstalla Node.js da [nodejs.org](https://nodejs.org/) e riavvia il computer.

---

## Licenza

MIT — libero di usare, modificare e distribuire.

## Crediti

Sviluppato con [Google Places API](https://developers.google.com/maps/documentation/places/web-service), [ExcelJS](https://github.com/exceljs/exceljs), [Nodemailer](https://nodemailer.com/) e [node-cron](https://github.com/node-cron/node-cron).