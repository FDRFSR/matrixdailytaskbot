# Matrix Task Bot

Bot Matrix multi-utente per la gestione delle task personali con
reminder intelligenti e report automatici.

Supporta input in linguaggio naturale e comandi strutturati, con
isolamento completo dei dati per ogni utente.

------------------------------------------------------------------------

## Funzionalità

-   Gestione task personali
-   Reminder singoli e ricorrenti
-   Report giornaliero automatico
-   Input in linguaggio naturale
-   Privacy per utente
-   Backup automatici
-   Deploy su LXC Proxmox

------------------------------------------------------------------------

## Caratteristiche

### Gestione Task

Ogni task include:

-   Titolo
-   Descrizione (opzionale)
-   Priorità (Alta / Media / Bassa)
-   Categoria
-   Stato (Attiva / Completata / Archiviata)
-   Reminder

### Reminder

-   One-shot e ricorrenti
-   Sintassi naturale
-   Supporto cron
-   Recupero notifiche dopo downtime

### Report

-   Invio automatico alle 07:00
-   Task attive
-   Reminder imminenti

------------------------------------------------------------------------

## Architettura

-   Linguaggio: Python 3.11+
-   Client Matrix: matrix-nio
-   Database: SQLite (WAL)
-   Scheduler: APScheduler
-   NLP: dateparser (spaCy opzionale)

### Infrastruttura

-   LXC su Proxmox
-   Debian 12
-   systemd
-   Backup automatici

------------------------------------------------------------------------

## Struttura Progetto

    matrix-taskbot/
    ├── bot.py
    ├── config.py
    ├── requirements.txt
    ├── taskbot/
    ├── migrations/
    ├── tests/
    └── scripts/

------------------------------------------------------------------------

## Installazione

### Prerequisiti

-   Debian 12
-   Python 3.11+
-   Account Matrix
-   Git

``` bash
apt install -y python3 python3-venv python3-pip git sqlite3
```

------------------------------------------------------------------------

### Clonazione

``` bash
git clone https://github.com/<user>/matrix-taskbot.git
cd matrix-taskbot
```

------------------------------------------------------------------------

### Ambiente Virtuale

``` bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

------------------------------------------------------------------------

### Configurazione

``` bash
cp .env.example .env
nano .env
```

Esempio:

``` env
MATRIX_HOMESERVER=https://matrix.example.com
MATRIX_USER_ID=@taskbot:matrix.example.com
MATRIX_PASSWORD=secret

DATABASE_PATH=/data/taskbot.db

TIMEZONE=Europe/Rome
DAILY_REPORT_TIME=07:00
```

------------------------------------------------------------------------

## Avvio Manuale

``` bash
source venv/bin/activate
python bot.py
```

------------------------------------------------------------------------

## Setup systemd

File `/etc/systemd/system/matrix-taskbot.service`:

``` ini
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

[Install]
WantedBy=multi-user.target
```

------------------------------------------------------------------------

## Utilizzo

### Linguaggio Naturale

Esempi:

    Ricordami di chiamare il dottore domani alle 15
    Mostrami le task attive
    Fatto report Q4

------------------------------------------------------------------------

### Comandi

  Comando         Descrizione
  --------------- -------------
  !task add       Crea task
  !task list      Lista
  !task done      Completa
  !task edit      Modifica
  !task delete    Elimina
  !reminder add   Reminder
  !help           Guida

------------------------------------------------------------------------

## Testing

``` bash
pytest tests/
```

Coverage target: ≥ 80%

------------------------------------------------------------------------

## Backup

-   Backup giornalieri SQLite
-   Retention: 7 giorni
-   Pulizia automatica
-   Log rotation

------------------------------------------------------------------------

## Roadmap

### MVP

-   CRUD task
-   Reminder
-   Report
-   NLP base

### Fase 2

-   NLP avanzato
-   Report settimanale
-   Reminder complessi

### Fase 3

-   Dashboard
-   Export dati
-   Statistiche
-   Integrazione calendario

------------------------------------------------------------------------

## Licenza

MIT (da confermare)

------------------------------------------------------------------------

## Stato

Progetto in sviluppo (MVP)
