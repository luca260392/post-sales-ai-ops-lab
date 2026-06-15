# Post-Sales AI Ops Lab

Laboratorio per l'automazione delle operazioni post-vendita tramite n8n, Airtable e AI.

## Struttura del progetto

- `n8n/` - workflow ed export di n8n
- `airtable/` - schemi e dati di esempio Airtable
- `docs/` - documentazione del progetto
  - `airtable_schema.md` - schema delle tabelle Airtable
  - `prompts.md` - prompt utilizzati nei flussi AI
  - `playbook_troubleshooting.md` - guida alla risoluzione dei problemi
  - `flowcharts/` - diagrammi di flusso dei processi
- `screenshots/` - screenshot di configurazioni e workflow

## Setup

1. Copia `.env.example` in `.env` e compila le variabili richieste
2. Avvia n8n tramite docker-compose
3. Configura le credenziali Airtable e AI nei workflow n8n

## Requisiti

- Docker e Docker Compose
- Account Airtable
- API key del provider AI utilizzato
