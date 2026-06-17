# CLAUDE.md — Istruzioni per Claude Code

Questo file viene letto automaticamente da Claude Code ad ogni sessione.
Contiene tutto il contesto necessario per lavorare su questo progetto.

## Cos'è questo progetto

Post-Sales AI Ops Lab è un sistema di automazione post-vendita per micro-SaaS,
costruito su n8n, Airtable e OpenAI. È un progetto portfolio che dimostra
capacità di orchestrazione workflow, integrazione LLM e monitoraggio operativo.

Livello: junior-mid. Obiettivo: codice ordinato, leggibile, documentato.
Non aggiungere complessità non richiesta.

## File di riferimento obbligatori

Prima di qualsiasi operazione, leggi questi due file nella root:
- `descrizione_prodotto.md` — architettura, moduli, schema Airtable, stack
- `sviluppo_prodotto.md` — setup, istruzioni nodo per nodo, milestone, prompt

## Regole generali

- Non inventare strutture non descritte nei file di riferimento
- Rispetta i nomi di tabelle, campi e workflow esattamente come definiti
- Ogni file che crei deve avere un commento iniziale che spiega cosa fa
- Usa sempre il file `.env.example` come riferimento per le variabili
- Non committare mai .env, chiavi API o dati sensibili

## Stack tecnologico

n8n · Airtable · OpenAI API · Webhook · SMTP (Mailtrap in dev) · Slack

## Variabili ambiente

Tutte le variabili sono definite in `.env.example`.
In sviluppo locale usa `.env` (mai committato).

## Convenzioni nomi file

- Workflow n8n: `01_onboarding.json`, `02_ticket_triage.json`, ecc.
- Documentazione: snake_case, es. `airtable_schema.md`
- Screenshot: `n8n_[nome_workflow].png`, `airtable_[nome_vista].png`