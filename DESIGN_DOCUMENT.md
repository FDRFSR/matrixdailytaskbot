# Matrix Task Bot - Documento di Pianificazione

*Bot per gestione task giornaliere con reminder personalizzati*

---

## 1. Executive Summary

Bot Matrix multi-utente per la gestione di task personali con sistema di reminder flessibile. Ogni utente opera in una room privata con il bot, gestendo le proprie task attraverso comandi in linguaggio naturale. Il bot fornisce notifiche programmate e report giornalieri automatici.

---

## 2. Requisiti Funzionali

### 2.1 Gestione Utenti e Privacy

- Multi-utente: il bot supporta più utenti contemporaneamente
- Room private 1:1 tra bot e utente (no gruppi)
- Isolamento completo dei dati: ogni utente vede solo le proprie task
- Nessun sistema di permessi gerarchici: tutti gli utenti hanno accesso equivalente

### 2.2 Struttura Task

Ogni task contiene i seguenti campi:

- Titolo (obbligatorio)
- Descrizione (opzionale)
- Priorità: Alta / Media / Bassa
- Categoria (es. lavoro, personale, casa)
- Stato: Attiva / Completata / Archiviata
- Data creazione (automatica)
- Reminder configurati (vedi sezione 2.3)

### 2.3 Sistema Reminder

#### 2.3.1 Tipi di Reminder

- Reminder personalizzati con pattern cron-like o espressioni naturali
- Esempi supportati: una tantum, giornaliero, settimanale, mensile, personalizzato
- Orari specifici (non relativi)

#### 2.3.2 Comportamento Reminder

- Reminder multipli per task: continuano finché la task non viene completata
- Task senza reminder: nessuna notifica automatica
- Reminder anticipati (opzionale): notifica preventiva prima dell'evento principale
- Recupero offline: reminder persi durante downtime vengono notificati al riavvio

#### 2.3.3 Fuso Orario

- Tutti gli utenti operano nello stesso fuso orario
- Configurazione globale del timezone in variabile d'ambiente

### 2.4 Operazioni su Task

#### 2.4.1 Azioni Supportate

- Creare nuove task
- Visualizzare lista task (attive, completate, tutte)
- Modificare task esistenti (titolo, descrizione, priorità, categoria)
- Completare task
- Archiviare task completate
- Eliminare definitivamente task
- Aggiungere / modificare / rimuovere reminder

#### 2.4.2 Funzionalità NON Supportate

- Assegnazione task ad altri utenti
- Ricerca testuale nelle task
- Note o commenti aggiuntivi su task
- Subtask o checklist interne
- Deadline separate dai reminder

### 2.5 Interazione Utente

#### 2.5.1 Modalità di Input

- Linguaggio naturale preferito (es. "ricordami di chiamare il dottore domani alle 15")
- Parser NLP per interpretare intento, entità temporali, priorità
- Fallback su comandi strutturati (vedi sezione 4.4)

#### 2.5.2 Feedback Sistema

- Nessuna conferma esplicita dopo ogni azione (comportamento silenzioso)
- Solo notifiche in caso di errore o ambiguità
- Messaggi concisi, minimo uso emoji

#### 2.5.3 Report Automatici

- Report giornaliero inviato automaticamente alle 07:00
- Contenuto: task attive del giorno, reminder in arrivo, task in scadenza
- Opzione report settimanale (da valutare in fase implementativa)

#### 2.5.4 Sistema Help

- Comando help disponibile
- Mostra esempi di comandi in linguaggio naturale e comandi strutturati
- Descrive funzionalità principali

---

## 3. Architettura Tecnica

### 3.1 Infrastruttura

#### 3.1.1 Hosting

Il bot gira su un **container LXC su Proxmox** presente nella rete locale.

- OS consigliato: Debian 12 (Bookworm) minimal
- Risorse LXC consigliate: 1 vCPU, 512 MB RAM, 4 GB disco
- Tipo container: unprivileged
- Rete: bridge verso LAN, IP statico consigliato
- Accesso: SSH per manutenzione

#### 3.1.2 Database

- SQLite per semplicità e portabilità
- File database: `/data/taskbot.db`
- WAL mode abilitato per concorrenza
- Schema versioning con migrazioni (vedi sezione 4.9)

#### 3.1.3 Persistenza e Backup

- Backup giornaliero automatico del database SQLite
- Retention: ultimi 7 backup giornalieri
- Directory backup: `/data/backups/`
- Opzionale: sync su cloud storage (S3, Google Drive)
- Opzionale: snapshot LXC periodici da Proxmox

### 3.2 Stack Tecnologico

#### 3.2.1 Linguaggio e Runtime

- Python 3.11+
- Async/await per gestione concorrente multipli utenti

#### 3.2.2 Librerie Principali

- **matrix-nio**: client Matrix async
- **APScheduler**: scheduling reminder e report
- **aiosqlite**: database access async
- **dateparser**: parsing date/time in linguaggio naturale
- **spaCy** o simile (opzionale, Fase 2): NLP avanzato per intent recognition

#### 3.2.3 Gestione Processo

- systemd service per auto-start e restart automatico
- Logging con rotation (logrotate)
- File di log: `/var/log/taskbot/bot.log`

### 3.3 Schema Database

> Dettaglio completo in [sezione 4.2 – Modello Dati](#42-modello-dati-sqlite).

Tabelle principali:
- **users**: utenti Matrix registrati
- **tasks**: task con titolo, priorità, categoria, stato
- **reminders**: reminder associati a task con espressione cron e next_run

### 3.4 Flusso Esecuzione

1. Bot si connette al server Matrix (sync loop persistente)
2. Ascolta eventi in tutte le room dove è presente
3. Al ricevimento di un messaggio:
   - a. Identifica utente (`matrix_user_id`) e room
   - b. Parser NLP analizza il testo → produce `Intent` + `Entities`
   - c. Se confidence bassa o ambiguità → messaggio errore con richiesta chiarimento
   - d. Se intento chiaro → esegue operazione su database via Service Layer
   - e. Risponde solo se output richiesto (es. lista task) o errore
4. Scheduler esegue job periodici:
   - a. Controlla reminder in scadenza (ogni 60s)
   - b. Invia notifiche agli utenti
   - c. Invia report giornaliero (default 07:00)
   - d. Esegue backup database (default 03:00)
   - e. Esegue pulizia retention (default 04:00)

---

## 4. Specifiche Tecniche Dettagliate (MVP)

### 4.1 Struttura Repository e Layout Moduli

#### 4.1.1 Struttura repository

```
matrix-taskbot/
├── bot.py                          # Entry point: config, db init, matrix client, scheduler
├── config.py                       # Configurazione centralizzata da env + default
├── requirements.txt                # Dipendenze Python
├── README.md                       # Documentazione progetto
├── .env.example                    # Template variabili d'ambiente
├── migrations/
│   ├── 001_init.sql                # Schema iniziale
│   └── 002_example.sql             # Migrazione esempio (futuro)
├── taskbot/
│   ├── __init__.py
│   ├── handlers/
│   │   ├── __init__.py
│   │   ├── message_router.py       # Normalizza input, smista a parser/services
│   │   └── help_handler.py         # Genera messaggio help
│   ├── services/
│   │   ├── __init__.py
│   │   ├── task_service.py         # CRUD task (logica dominio)
│   │   └── reminder_service.py     # CRUD reminder (logica dominio)
│   ├── nlp/
│   │   ├── __init__.py
│   │   ├── parser.py               # Intent recognition + entity extraction
│   │   └── rules.py                # Pattern matching, keyword → intent/entity
│   ├── scheduler/
│   │   ├── __init__.py
│   │   ├── reminder_scheduler.py   # Job: controlla e invia reminder
│   │   └── daily_report.py         # Job: genera e invia report giornaliero
│   ├── db/
│   │   ├── __init__.py
│   │   ├── models.py               # Dataclass / modelli
│   │   ├── repository.py           # Query DB (repository pattern)
│   │   └── migrations.py           # Logica applicazione migrazioni
│   ├── utils/
│   │   ├── __init__.py
│   │   ├── time.py                 # Conversioni timezone, formattazione date
│   │   ├── logging.py              # Setup logging strutturato
│   │   └── validation.py           # Validazione input (priorità, categoria, etc.)
│   └── backups/
│       ├── __init__.py
│       └── backup_service.py       # Backup DB + retention cleanup
├── tests/
│   ├── __init__.py
│   ├── test_parser.py              # Unit test NLP parser
│   ├── test_task_service.py        # Unit test task CRUD
│   ├── test_reminder_service.py    # Unit test reminder CRUD
│   ├── test_scheduler.py           # Test scheduler con freezegun
│   └── test_migrations.py          # Test migrazioni DB
└── scripts/
    └── deploy.sh                   # Script deploy automatico (vedi sezione 6.4)
```

#### 4.1.2 Linee guida layout
- **Entry point** (`bot.py`): minimale, avvia config, db, matrix client e scheduler
- **Package core**: tutto il codice applicativo sotto `taskbot/`
- **Handlers**: routing comandi, validazione input, gestione errori
- **Services**: logica di dominio, senza dipendenza diretta da Matrix
- **NLP**: parser + regole; isolare librerie esterne per sostituibilità
- **Scheduler**: job separati per reminder e report
- **DB**: modelli + repository pattern per query
- **Utils**: funzioni condivise (timezone, logging, validazione)
- **Backups**: logica backup isolata e testabile
- **Tests**: separati per livello (parser/service/scheduler/migrations)

---

### 4.2 Modello Dati (SQLite)

#### 4.2.1 Tabelle

**users**

| Campo | Tipo | Note |
|-------|------|------|
| id | INTEGER | PK, autoincrement |
| matrix_user_id | TEXT | UNIQUE NOT NULL |
| room_id | TEXT | NOT NULL |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP |

**tasks**

| Campo | Tipo | Note |
|-------|------|------|
| id | INTEGER | PK, autoincrement |
| user_id | INTEGER | FK → users.id, NOT NULL |
| title | TEXT | NOT NULL |
| description | TEXT | NULL |
| priority | TEXT | CHECK (high\|medium\|low), DEFAULT 'medium' |
| category | TEXT | NULL |
| status | TEXT | CHECK (active\|completed\|archived), DEFAULT 'active' |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP |
| completed_at | TIMESTAMP | NULL |

**reminders**

| Campo | Tipo | Note |
|-------|------|------|
| id | INTEGER | PK, autoincrement |
| task_id | INTEGER | FK → tasks.id, NOT NULL |
| cron_expression | TEXT | NULL (se one-shot) |
| next_run | TIMESTAMP | NULL |
| is_active | BOOLEAN | DEFAULT 1 |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP |

#### 4.2.2 Indici
- `idx_tasks_user_status` → tasks(user_id, status)
- `idx_reminders_task_active` → reminders(task_id, is_active)
- `idx_reminders_next_run` → reminders(next_run) WHERE is_active = 1
- `idx_users_matrix_id` → users(matrix_user_id)

#### 4.2.3 Vincoli
- ON DELETE CASCADE da users → tasks → reminders
- WAL mode abilitato a livello di connessione

---

### 4.3 Specifiche NLP (MVP)

#### 4.3.1 Intenti supportati

| Intento | Trigger keywords (esempi) |
|---------|---------------------------|
| `create_task` | "ricordami", "aggiungi task", "crea", "nuova task" |
| `list_tasks` | "mostra", "lista", "task attive", "cosa devo fare" |
| `complete_task` | "fatto", "completato", "finito", "segna come fatto" |
| `update_task` | "modifica", "cambia", "aggiorna" |
| `delete_task` | "elimina", "cancella", "rimuovi" |
| `archive_task` | "archivia" |
| `add_reminder` | "reminder", "promemoria", "avvisami" |
| `remove_reminder` | "togli reminder", "rimuovi promemoria" |
| `help` | "help", "aiuto", "come funziona" |

#### 4.3.2 Entità

| Entità | Esempi | Libreria |
|--------|--------|----------|
| `title` | testo libero dopo keyword intento | regex/regole |
| `description` | "descrizione: ..." | regex |
| `priority` | "alta priorità", "urgente", "bassa" | keyword mapping |
| `category` | "categoria lavoro", "cat: casa" | regex/keyword |
| `datetime` | "domani alle 18", "15 marzo 14:30" | dateparser |
| `recurrence` | "ogni giorno", "ogni lunedì", "mensile" | keyword + cron mapping |
| `task_ref` | "la prima", "report Q4", ID numerico | fuzzy match su titolo |

#### 4.3.3 Mapping priorità

| Input utente | Valore DB |
|-------------|-----------|
| alta, urgente, importante, 🔴 | high |
| media, normale, 🟡 | medium |
| bassa, poco importante, 🟢 | low |

#### 4.3.4 Mapping ricorrenza → cron

| Input utente | Cron expression |
|-------------|----------------|
| ogni giorno alle 18 | `0 18 * * *` |
| ogni lunedì alle 9 | `0 9 * * 1` |
| ogni mese il 1 alle 10 | `0 10 1 * *` |
| una tantum (datetime) | cron_expression = NULL, next_run = datetime |

#### 4.3.5 Gestione ambiguità
- Confidence threshold: se nessun intento supera soglia → richiesta chiarimento
- Task reference ambigua (>1 match su titolo) → lista candidati e richiesta selezione
- Datetime non parsabile → "Non ho capito quando. Specifica data e ora."

---

### 4.4 Comandi Strutturati (Fallback)

Se il parser NLP non riesce a interpretare il linguaggio naturale, l'utente può usare comandi espliciti:

| Comando | Descrizione | Esempio |
|---------|-------------|---------|
| `!task add <titolo>` | Crea task | `!task add Comprare latte` |
| `!task add <titolo> -p high -c casa` | Crea con priorità e categoria | `!task add Bolletta -p high -c finanze` |
| `!task list` | Lista task attive | `!task list` |
| `!task list all` | Lista tutte le task | `!task list all` |
| `!task list completed` | Lista task completate | `!task list completed` |
| `!task done <id\|titolo>` | Completa task | `!task done 3` |
| `!task edit <id> -t <titolo>` | Modifica titolo | `!task edit 3 -t Nuovo titolo` |
| `!task edit <id> -p <priorità>` | Modifica priorità | `!task edit 3 -p low` |
| `!task edit <id> -c <categoria>` | Modifica categoria | `!task edit 3 -c lavoro` |
| `!task archive <id\|titolo>` | Archivia task | `!task archive 3` |
| `!task delete <id\|titolo>` | Elimina task | `!task delete 3` |
| `!reminder add <task_id> <datetime\|cron>` | Aggiungi reminder | `!reminder add 3 "ogni giorno alle 18"` |
| `!reminder remove <reminder_id>` | Rimuovi reminder | `!reminder remove 5` |
| `!help` | Mostra guida comandi | `!help` |

Flag opzionali per `!task add`:
- `-p <high|medium|low>`: priorità (default: medium)
- `-c <categoria>`: categoria
- `-d <descrizione>`: descrizione

---

### 4.5 API Interna (Service Layer)

#### 4.5.1 TaskService

```python
class TaskService:
    async def create_task(user_id: int, title: str,
                          description: str = None,
                          priority: str = "medium",
                          category: str = None) -> int  # returns task_id

    async def list_tasks(user_id: int,
                         status: str = "active") -> list[Task]
                         # status: "active" | "completed" | "archived" | "all"

    async def update_task(task_id: int, **fields) -> None
                          # fields: title, description, priority, category

    async def complete_task(task_id: int) -> None
                            # set status=completed, completed_at=now()
                            # disattiva tutti i reminder associati

    async def archive_task(task_id: int) -> None
                            # set status=archived

    async def delete_task(task_id: int) -> None
                           # DELETE task + CASCADE reminder
```

#### 4.5.2 ReminderService

```python
class ReminderService:
    async def add_reminder(task_id: int,
                           cron_expr: str = None,
                           run_at: datetime = None) -> int  # returns reminder_id

    async def remove_reminder(reminder_id: int) -> None

    async def get_due_reminders(now: datetime) -> list[Reminder]
                                # WHERE is_active = 1 AND next_run <= now

    async def mark_triggered(reminder_id: int) -> None
                              # se cron: calcola next_run
                              # se one-shot: set is_active = 0
```

---

### 4.6 Scheduler

#### 4.6.1 Reminder Job (ogni 60s)
- Query: reminders attivi con `next_run <= now`
- Per ogni reminder trovato:
  - Recupera task associata (titolo, categoria)
  - Invia messaggio Matrix nella room dell'utente
  - Chiama `mark_triggered()`:
    - Se ricorrente (cron_expression != NULL): calcola prossimo `next_run`
    - Se one-shot: disattiva reminder (`is_active = 0`)
- Idempotenza: controlla flag `is_active` prima di inviare

#### 4.6.2 Daily Report Job (default 07:00)
- Per ogni utente con task attive:
  - Recupera task attive ordinate per priorità
  - Recupera reminder nelle prossime 24h
  - Formatta e invia report nella room dell'utente

#### 4.6.3 Backup Job (default 03:00)
- Copia file SQLite in `/data/backups/taskbot_YYYYMMDD_HHMMSS.db`
- Elimina backup più vecchi di 7 giorni

#### 4.6.4 Retention Cleanup Job (default 04:00)
- Elimina task archiviate con `completed_at` > 180 giorni fa
- Elimina reminder disattivati con `created_at` > 7 giorni fa

---

### 4.7 Formati Output (MVP)

#### 4.7.1 Lista Task
```
📋 Task Attive (3):

1. 🔴 Completare report Q4 (lavoro)
2. 🟡 Annaffiare le piante (casa) — Reminder oggi 18:00
3. 🟢 Pagare bolletta internet (finanze)
```

#### 4.7.2 Report Giornaliero
```
📅 Report Giornaliero — Giovedì 15 Marzo 2026

Task Attive: 3
1. 🔴 Completare report Q4 (lavoro)
2. 🟡 Annaffiare le piante (casa) — Reminder oggi 18:00
3. 🟢 Pagare bolletta internet (finanze)

⏰ Reminder Oggi:
  14:30 — Appuntamento dottore
  18:00 — Annaffiare le piante
```

#### 4.7.3 Messaggi di Errore

| Situazione | Messaggio |
|-----------|-----------|
| Intento non riconosciuto | "Non ho capito. Scrivi `!help` per la lista comandi." |
| Datetime non parsabile | "Non ho capito quando. Specifica data e ora (es. 'domani alle 15')." |
| Task reference ambigua | "Ho trovato più task corrispondenti:\n1. Titolo A\n2. Titolo B\nQuale intendi?" |
| Task non trovata | "Nessuna task trovata con questo riferimento." |
| Reminder su task completata | "Questa task è già completata. Riattivala prima di aggiungere reminder." |

#### 4.7.4 Messaggio Help
```
🤖 Matrix Task Bot — Guida

Linguaggio naturale:
  "Ricordami di comprare il latte domani alle 10"
  "Mostrami le task attive"
  "Fatto comprare il latte"

Comandi strutturati:
  !task add <titolo> [-p high|medium|low] [-c categoria]
  !task list [all|completed]
  !task done <id|titolo>
  !task edit <id> [-t titolo] [-p priorità] [-c categoria]
  !task archive <id|titolo>
  !task delete <id|titolo>
  !reminder add <task_id> <quando>
  !reminder remove <id>
  !help
```

---

### 4.8 Configurazione (.env)

```env
# Matrix
MATRIX_HOMESERVER=https://matrix.example.com
MATRIX_USER_ID=@taskbot:matrix.example.com
MATRIX_PASSWORD=secret

# Database
DATABASE_PATH=/data/taskbot.db

# Backup
BACKUP_PATH=/data/backups
BACKUP_RETENTION_DAYS=7

# Timezone e scheduling
TIMEZONE=Europe/Rome
DAILY_REPORT_TIME=07:00
BACKUP_TIME=03:00
RETENTION_CLEANUP_TIME=04:00

# Retention policy
TASK_ARCHIVE_RETENTION_DAYS=180
REMINDER_INACTIVE_RETENTION_DAYS=7

# Logging
LOG_LEVEL=INFO
LOG_FILE=/var/log/taskbot/bot.log
```

---

### 4.9 Strategia Migrazioni DB

#### 4.9.1 Convenzioni
- File SQL in `migrations/`, numerati sequenzialmente: `001_init.sql`, `002_add_index.sql`, ...
- Ogni file contiene solo SQL forward (no rollback nell'MVP)
- Tabella interna `schema_version` traccia le migrazioni applicate

#### 4.9.2 Tabella schema_version

| Campo | Tipo |
|-------|------|
| version | INTEGER PK |
| filename | TEXT NOT NULL |
| applied_at | TIMESTAMP DEFAULT CURRENT_TIMESTAMP |

#### 4.9.3 Logica (db/migrations.py)
1. All'avvio del bot, legge `schema_version`
2. Confronta con file presenti in `migrations/`
3. Applica in ordine i file non ancora eseguiti
4. Registra ogni migrazione applicata in `schema_version`
5. Se una migrazione fallisce → log errore, bot non si avvia

---

### 4.10 Log e Monitoraggio

- Log strutturato (JSON opzionale)
- Livelli: DEBUG / INFO / WARN / ERROR
- Rotazione con logrotate (max 7 file, 10 MB ciascuno)
- Metriche minime loggate:
  - Task create / completate / eliminate
  - Reminder inviati / falliti
  - Errori parsing NLP
  - Tempo di risposta medio

---

### 4.11 Testing (MVP)

- Unit test per parser NLP e regole (`test_parser.py`)
- Unit test per TaskService e ReminderService (`test_task_service.py`, `test_reminder_service.py`)
- Integration test DB con sqlite in-memory (`test_migrations.py`)
- Scheduler test con clock fittizio — libreria `freezegun` (`test_scheduler.py`)
- Coverage target MVP: ≥ 80% su services e parser

---

### 4.12 Policy di Retention (MVP)

| Dato | Retention | Azione |
|------|-----------|--------|
| Task archiviate | 180 giorni da completed_at | DELETE automatico |
| Reminder disattivati | 7 giorni da disattivazione | DELETE automatico |
| Backup DB | 7 giorni | DELETE file più vecchi |
| Log | 7 file × 10 MB | Rotazione logrotate |

---

## 5. Esempi d'Uso

### 5.1 Casi d'Uso Tipici

**Esempio 1 — Creare task con reminder giornaliero:**

```
Utente: "Ricordami di annaffiare le piante ogni giorno alle 18"
Bot: [silenzioso, crea task + reminder cron 0 18 * * *]
```

**Esempio 2 — Task prioritaria senza reminder:**

```
Utente: "Aggiungi task: completare report Q4, alta priorità, categoria lavoro"
Bot: [silenzioso]
```

**Esempio 3 — Visualizzare task:**

```
Utente: "Mostrami le task attive"
Bot:
  📋 Task Attive (2):
  1. 🔴 Completare report Q4 (lavoro)
  2. 🟡 Annaffiare le piante (casa) — Reminder oggi 18:00
```

**Esempio 4 — Completare task:**

```
Utente: "Fatto report Q4"
Bot: [silenzioso, marca come completata, disattiva reminder]
```

**Esempio 5 — Reminder personalizzato:**

```
Utente: "Ricordami appuntamento dottore giovedì 15 marzo alle 14:30, con promemoria il giorno prima alle 20"
Bot: [silenzioso, crea task + 2 reminder one-shot]
```

**Esempio 6 — Fallback comando strutturato:**

```
Utente: "!task add Comprare latte -p low -c casa"
Bot: [silenzioso, crea task]
```

**Esempio 7 — Errore ambiguità:**

```
Utente: "Fatto comprare"
Bot: "Ho trovato più task corrispondenti:
      1. Comprare latte
      2. Comprare regalo
      Quale intendi?"
```

### 5.2 Report Giornaliero

**Messaggio automatico alle 07:00:**

```
📅 Report Giornaliero — Giovedì 15 Marzo 2026

Task Attive: 3
1. 🔴 Completare report Q4 (lavoro)
2. 🟡 Annaffiare le piante (casa) — Reminder oggi 18:00
3. 🟢 Pagare bolletta internet (finanze)

⏰ Reminder Oggi:
  14:30 — Appuntamento dottore
  18:00 — Annaffiare le piante
```

---

## 6. Configurazione e Deploy

### 6.1 Variabili d'Ambiente

Vedi [sezione 4.8](#48-configurazione-env) per il file `.env` completo.

### 6.2 Setup LXC su Proxmox

#### 6.2.1 Creazione container

1. Template: Debian 12 (Bookworm) standard
2. Risorse: 1 vCPU, 512 MB RAM, 4 GB disco
3. Tipo: unprivileged
4. Rete: bridge `vmbr0`, IP statico consigliato
5. Hostname: `taskbot`

#### 6.2.2 Setup iniziale nel container

```bash
# Aggiornamento sistema
apt update && apt upgrade -y

# Installazione dipendenze
apt install -y python3 python3-venv python3-pip git sqlite3

# Creazione utente di sistema
useradd -r -m -s /bin/bash taskbot

# Creazione directory
mkdir -p /opt/matrix-taskbot /data/backups /var/log/taskbot
chown -R taskbot:taskbot /opt/matrix-taskbot /data /var/log/taskbot
```

#### 6.2.3 Deploy applicazione

```bash
# Come utente taskbot
su - taskbot
cd /opt/matrix-taskbot

# Clone repository
git clone https://github.com/<user>/matrix-taskbot.git .

# Virtual environment
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Configurazione
cp .env.example .env
nano .env  # editare con valori reali
```

### 6.3 Setup Systemd Service

File: `/etc/systemd/system/matrix-taskbot.service`

```ini
[Unit]
Description=Matrix Task Bot
After=network.target

[Service]
Type=simple
User=taskbot
WorkingDirectory=/opt/matrix-taskbot
EnvironmentFile=/opt/matrix-taskbot/.env
ExecStart=/opt/matrix-taskbot/venv/bin/python bot.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable matrix-taskbot
systemctl start matrix-taskbot
```

### 6.4 Script di Deploy Automatico

File: `scripts/deploy.sh`

Questo script viene eseguito nel container LXC per aggiornare il bot dopo un push sul repository.

```bash
#!/bin/bash
set -euo pipefail

APP_DIR="/opt/matrix-taskbot"
SERVICE_NAME="matrix-taskbot"
LOG_FILE="/var/log/taskbot/deploy.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "=== Deploy avviato ==="

cd "$APP_DIR"

# Pull ultime modifiche
log "Git pull..."
git fetch origin
git reset --hard origin/main

# Aggiorna dipendenze (solo se requirements.txt è cambiato)
if git diff HEAD~1 --name-only | grep -q "requirements.txt"; then
    log "Aggiornamento dipendenze..."
    source venv/bin/activate
    pip install -r requirements.txt --quiet
fi

# Restart servizio
log "Restart servizio..."
sudo systemctl restart "$SERVICE_NAME"

# Verifica stato
sleep 3
if systemctl is-active --quiet "$SERVICE_NAME"; then
    log "✅ Deploy completato con successo"
else
    log "❌ ERRORE: servizio non attivo dopo restart"
    systemctl status "$SERVICE_NAME" >> "$LOG_FILE" 2>&1
    exit 1
fi
```

Permessi: `chmod +x scripts/deploy.sh`

#### 6.4.1 Opzioni di trigger

**Opzione A — Manuale (SSH):**
```bash
ssh taskbot@<ip-lxc> '/opt/matrix-taskbot/scripts/deploy.sh'
```

**Opzione B — Webhook (GitHub → deploy):**
- Micro HTTP server nel container (es. `webhook` di adnanh) in ascolto su porta interna
- GitHub webhook POST su push → trigger `deploy.sh`
- Alternativa: cron job che fa pull periodico (es. ogni 5 min)

**Opzione C — GitHub Actions + SSH:**
```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.LXC_HOST }}
          username: taskbot
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: /opt/matrix-taskbot/scripts/deploy.sh
```

> **Nota**: per Opzione B e C il container LXC deve essere raggiungibile dalla rete esterna (port forward su Proxmox o VPN).

### 6.5 Setup Logrotate

File: `/etc/logrotate.d/matrix-taskbot`

```
/var/log/taskbot/*.log {
    daily
    rotate 7
    maxsize 10M
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
}
```

---

## 7. Testing e Manutenzione

### 7.1 Testing Strategy

Vedi [sezione 4.11](#411-testing-mvp) per dettagli.

- Unit test per parser NLP e logica task
- Integration test per database operations
- End-to-end test con account Matrix di test
- Test scheduler con mock time

### 7.2 Monitoring

- Log strutturati con timestamp e livelli
- Metriche: numero task create, reminder inviati, errori
- Health check endpoint (opzionale)

### 7.3 Manutenzione Routine

- Verifica backup giornalieri
- Pulizia automatica da retention policy (vedi [sezione 4.12](#412-policy-di-retention-mvp))
- Aggiornamenti dipendenze Python
- Monitoraggio spazio disco per database e log
- Snapshot LXC periodici da Proxmox (consigliato: settimanale)

---

## 8. Roadmap e Estensioni Future

### 8.1 Fase 1 — MVP (Minimum Viable Product)

- CRUD task base
- Reminder singoli a orario fisso e ricorrenti semplici
- Report giornaliero
- Parser NLP base + comandi strutturati fallback
- Deploy su LXC Proxmox con systemd

### 8.2 Fase 2 — NLP e UX

- Parser linguaggio naturale avanzato (spaCy)
- Reminder ricorrenti complessi
- Gestione ambiguità con domande chiarificatrici
- Report settimanale

### 8.3 Fase 3 — Features Avanzate (Opzionali)

- Integrazione calendario (CalDAV)
- Export task in vari formati (JSON, CSV, iCal)
- Statistiche e analytics personali
- Snooze reminder
- Task ricorrenti template
- Web dashboard read-only

---

## 9. Rischi e Mitigazioni

### 9.1 Rischi Tecnici

**Rischio: Connessione Matrix instabile**
- Mitigazione: Reconnect automatico, gestione eventi offline, queue messaggi in uscita

**Rischio: Database corruption**
- Mitigazione: Backup automatici frequenti, WAL mode SQLite, validation su restore

**Rischio: Parsing ambiguo comandi NLP**
- Mitigazione: Confidence threshold, richiesta chiarimenti, fallback comandi strutturati (sezione 4.4)

**Rischio: Scheduler miss reminder durante crash**
- Mitigazione: Check reminder persi all'avvio (query `next_run < now AND is_active = 1`), idempotenza notifiche

### 9.2 Rischi Operativi

**Rischio: Crescita incontrollata database**
- Mitigazione: Retention policy automatica (sezione 4.12), monitoring disk usage

**Rischio: Downtime homeserver Matrix esterno**
- Mitigazione: Nessuna mitigazione diretta, considerare self-hosted homeserver in futuro

**Rischio: LXC non raggiungibile dall'esterno per deploy automatico**
- Mitigazione: VPN (es. WireGuard), oppure deploy manuale via SSH locale, oppure cron pull

---

## 10. Conclusione

Questo documento fornisce le linee guida complete per lo sviluppo del bot Matrix per la gestione di task giornaliere. L'architettura proposta privilegia semplicità e affidabilità, con focus sull'esperienza utente attraverso comandi in linguaggio naturale e notifiche intelligenti.

Il sistema è progettato per essere estensibile, permettendo l'aggiunta graduale di funzionalità avanzate senza compromettere la stabilità del core. L'approccio multi-fase consente di validare l'utilità del bot prima di investire in feature complesse.

**Prossimi passi:**

1. Creare repository e struttura directory (sezione 4.1)
2. Implementare schema DB + migrazioni (sezione 4.2, 4.9)
3. Sviluppare MVP Fase 1 (sezione 8.1)
4. Setup LXC Proxmox + deploy (sezione 6.2–6.4)
5. Testing con utenti reali
6. Iterazione basata su feedback
