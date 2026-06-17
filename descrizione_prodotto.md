# Post-Sales AI Ops Lab — Descrizione del Prodotto

## Panoramica

**Post-Sales AI Ops Lab** è un sistema di automazione post-vendita per micro-SaaS e piccole software house, costruito interamente su strumenti no-code/low-code orchestrati da n8n. Il progetto simula il ciclo di vita completo di un cliente dopo l'acquisto: dall'onboarding automatico, alla gestione dei ticket di supporto, fino ai reminder di retention — con un layer di monitoraggio che registra log, metriche e anomalie di ogni flusso.

L'obiettivo non è costruire un'intelligenza artificiale "magica", ma dimostrare la capacità di progettare automazioni leggibili, ben documentate, con fallback espliciti e metriche operative reali.

---

## Problema che risolve

Le aziende SaaS di piccole dimensioni perdono clienti e tempi operativi per mancanza di automazione strutturata nei processi post-vendita. Un nuovo cliente che compra un piano non riceve welcome email consistenti, i ticket aperti non vengono classificati prima di toccare un operatore, e nessuno tiene traccia di quanti flussi falliscono ogni settimana.

**Post-Sales AI Ops Lab** replica questa realtà in versione portfolio-ready: mostra come si costruisce, si monitora e si itera su un sistema di agenti AI operativi in contesti aziendali reali.

---

## Stack tecnologico

| Strumento | Ruolo nel progetto |
|---|---|
| **n8n** (self-hosted o cloud) | Orchestratore principale di tutti i workflow |
| **Airtable** | Database operativo (CRM clienti, log ticket, stato flussi) |
| **OpenAI API** (o Claude API) | Classificazione ticket, generazione testi email, analisi sentiment |
| **Webhook / API REST** | Trigger in ingresso da form, CRM o sistemi esterni simulati |
| **Notion** (opzionale) | Gestione knowledge base e playbook interni |
| **ElevenLabs** (stub opzionale) | Simulazione voice follow-up — non attivato in produzione portfolio |
| **n8n built-in nodes** | Email (SMTP), Slack/Discord, HTTP Request, Set, IF, Switch, Wait |

> **Nota sulle API key:** Il repo include un file `.env.example` con tutti i placeholder. Nessuna chiave reale è mai committata.

---

## Architettura generale

Il sistema è composto da **4 workflow principali** più un **layer di monitoring** trasversale. Ogni workflow è un file JSON esportato da n8n, importabile in qualsiasi istanza.

```
TRIGGER (webhook / schedule / form)
    │
    ▼
[WORKFLOW n8n]
    │
    ├── Logic nodes (IF / Switch / Set)
    │       │
    │       ├── AI node (OpenAI / Claude)
    │       │
    │       └── Fallback path (operatore manuale)
    │
    ├── Data layer (Airtable read/write)
    │
    └── Output (email / Slack / log entry / KPI update)
```

Ogni flusso scrive un record di log in Airtable alla fine della sua esecuzione, contenente: workflow ID, timestamp, esito (success / partial / failed), payload sintetico e link alla run di n8n. Questo alimenta la dashboard KPI.

---

## I 5 moduli del sistema

### Modulo 1 — Lead-to-Client Onboarding

**Trigger:** Webhook POST da form di registrazione o acquisto simulato.

**Cosa fa:**
- Valida il payload in ingresso (nome, email, piano acquistato)
- Crea o aggiorna il record cliente in Airtable (tabella `Clients`)
- Chiama OpenAI per personalizzare la welcome email in base al piano
- Invia l'email via SMTP con testo generato e signature statica
- Crea una task di onboarding in Airtable (tabella `Tasks`) con scadenza a 3 giorni
- Scrive un log in `Automation Logs` con esito e dati sintetici

**Fallback:** Se OpenAI non risponde entro 5 secondi, viene inviata una email di welcome con template statico predefinito. Il log registra `status: partial` e un flag `ai_skipped: true`.

**Dati Airtable coinvolti:**
- Tabella `Clients`: `id`, `name`, `email`, `plan`, `status`, `created_at`
- Tabella `Tasks`: `client_id`, `type: onboarding`, `due_date`, `completed`
- Tabella `Automation Logs`: `workflow`, `timestamp`, `status`, `payload_summary`

---

### Modulo 2 — Support Ticket Triage

**Trigger:** Webhook POST da form ticket o email inbound via IMAP (simulato con payload statico).

**Cosa fa:**
- Legge il testo del ticket e lo passa a OpenAI con un prompt di classificazione
- Assegna categoria (`billing`, `bug`, `feature request`, `how-to`) e priorità (`low`, `medium`, `high`, `critical`)
- Crea il record in Airtable (tabella `Tickets`) con tutti i metadati
- Se priorità è `critical`: notifica immediata su Slack/Discord con link al ticket
- Se priorità è `low` o `medium`: invia risposta automatica al cliente con stima di risposta
- Scrive log con categoria assegnata, priorità, modello usato e confidence score (estratto dalla risposta LLM)

**Fallback:** Se la classificazione LLM restituisce risposta non parsabile, il ticket viene assegnato a categoria `unknown`, priorità `medium` e flaggato per revisione manuale. Il log registra `classification_error: true`.

**Prompt documentato:**
```
Sei un assistente di supporto tecnico. Leggi il seguente messaggio di un cliente
e restituisci un JSON con:
- "category": uno tra ["billing", "bug", "feature_request", "how_to", "other"]
- "priority": uno tra ["low", "medium", "high", "critical"]
- "confidence": numero da 0 a 1
- "summary": massimo 20 parole che riassumono il problema

Messaggio cliente:
{{$json.message}}

Rispondi SOLO con il JSON, senza testo aggiuntivo.
```

---

### Modulo 3 — Retention Reminder

**Trigger:** Schedule node — eseguito ogni giorno alle 09:00.

**Cosa fa:**
- Legge da Airtable tutti i clienti con `last_activity` > 14 giorni fa e `status: active`
- Per ogni cliente, chiama OpenAI per generare un messaggio personalizzato basato sul piano e sulla data di iscrizione
- Invia l'email di re-engagement
- Aggiorna il campo `last_reminder_sent` sul record cliente
- Scrive log con numero di clienti processati, inviati, saltati e eventuali errori

**Fallback:** Se un cliente ha già ricevuto un reminder negli ultimi 7 giorni (campo `last_reminder_sent`), viene saltato. Se OpenAI fallisce, viene inviato un messaggio di retention statico da template.

**Logica anti-spam:** campo `reminder_count` in Airtable — dopo 3 reminder senza risposta, il cliente viene flaggato come `churn_risk` e tolto dalla coda automatica per revisione manuale.

---

### Modulo 4 — Voice Follow-Up Stub *(opzionale / simulato)*

**Status nel portfolio:** Questo modulo è presente come **architettura documentata e codice stub**, ma non attivato con API reali ElevenLabs per motivi di costo e semplicità demo.

**Cosa rappresenta:**
- Workflow n8n che, dopo un ticket `critical` chiuso o dopo 3 giorni senza risposta al reminder, prepara un testo di follow-up
- Il testo viene passato a un nodo HTTP Request che chiama l'endpoint ElevenLabs (con API key placeholder)
- L'audio generato verrebbe salvato e inviato via link nell'email successiva

**Perché è incluso:** Dimostra comprensione dell'integrazione vocale nei flussi post-vendita, come richiesto dall'annuncio, senza fingere un'implementazione funzionante non testabile in contesto portfolio.

---

### Modulo 5 — Monitoring Board

**Trigger:** Schedule node — ogni ora + webhook on-demand.

**Cosa fa:**
- Aggrega i record della tabella `Automation Logs` degli ultimi 7 giorni
- Calcola le KPI principali (vedi tabella sotto)
- Aggiorna una tabella `KPI Dashboard` in Airtable
- Se il tasso di errore > 20%, invia un alert su Slack

**Tabella KPI aggiornata da questo modulo:**

| KPI | Descrizione | Fonte |
|---|---|---|
| `total_runs` | Numero totale di esecuzioni workflow | `Automation Logs` |
| `success_rate` | % di run con `status: success` | `Automation Logs` |
| `partial_rate` | % di run con `status: partial` (AI skipped) | `Automation Logs` |
| `error_rate` | % di run con `status: failed` | `Automation Logs` |
| `onboarding_count` | Nuovi clienti onboardati negli ultimi 7 giorni | `Clients` |
| `tickets_triaged` | Ticket classificati automaticamente | `Tickets` |
| `ai_accuracy_proxy` | Media confidence score da classificazione LLM | `Tickets` |
| `reminders_sent` | Email retention inviate | `Automation Logs` |
| `churn_risk_flagged` | Clienti flaggati come churn risk | `Clients` |

---

## Schema Airtable

Il database operativo è composto da 5 tabelle collegate:

```
Clients
├── id (auto)
├── name
├── email
├── plan [free | starter | pro]
├── status [active | churned | churn_risk]
├── created_at
├── last_activity
├── last_reminder_sent
└── reminder_count

Tickets
├── id (auto)
├── client_id → Clients
├── message (testo originale)
├── category [billing | bug | feature_request | how_to | other | unknown]
├── priority [low | medium | high | critical]
├── confidence_score
├── status [open | in_progress | closed]
└── created_at

Tasks
├── id (auto)
├── client_id → Clients
├── type [onboarding | follow_up | manual_review]
├── due_date
└── completed

Automation Logs
├── id (auto)
├── workflow [onboarding | triage | retention | monitoring]
├── timestamp
├── status [success | partial | failed]
├── payload_summary
├── ai_skipped (boolean)
├── classification_error (boolean)
└── notes

KPI Dashboard
├── date
├── total_runs
├── success_rate
├── partial_rate
├── error_rate
├── onboarding_count
├── tickets_triaged
├── ai_accuracy_proxy
├── reminders_sent
└── churn_risk_flagged
```

---

## Struttura del repository

```
post-sales-ai-ops-lab/
├── README.md
├── .env.example
├── n8n/
│   ├── 01_onboarding.json
│   ├── 02_ticket_triage.json
│   ├── 03_retention_reminder.json
│   ├── 04_voice_followup_stub.json
│   └── 05_monitoring_board.json
├── docs/
│   ├── flowcharts/
│   │   ├── 01_onboarding_flow.png
│   │   ├── 02_triage_flow.png
│   │   ├── 03_retention_flow.png
│   │   └── 05_monitoring_flow.png
│   ├── airtable_schema.md
│   ├── prompts.md
│   └── playbook_troubleshooting.md
├── airtable/
│   └── base_schema.json         ← schema esportato o documentato
└── screenshots/
    ├── n8n_onboarding_workflow.png
    ├── airtable_clients_view.png
    └── kpi_dashboard_view.png
```

---

## Cosa dimostra questo progetto

- Progettazione di workflow multi-step con logica condizionale (IF / Switch / fallback path)
- Integrazione di LLM in flussi operativi con prompt versionati e output strutturati (JSON)
- Separazione netta tra trigger, logica e azione in ogni workflow
- Gestione dei casi limite: timeout API, output non parsabile, clienti già contattati
- Data flow tra piattaforme distinte (webhook → n8n → OpenAI → Airtable → email)
- Monitoraggio operativo: log strutturati, KPI aggregati, alert automatici
- Documentazione professionale: flowchart, schema dati, prompt, playbook di troubleshooting

---

## Limiti volontari (scope del portfolio)

Questo progetto è intenzionalmente circoscritto per restare credibile a livello junior-mid. Non include:

- Rete di agenti autonomi con memoria condivisa o comunicazione inter-agente
- Orchestrazione multi-modello in tempo reale
- Integrazione ElevenLabs funzionante (presente solo come stub documentato)
- CRM esterno reale (Salesforce, HubSpot) — sostituito da Airtable
- Multi-tenant o gestione di più aziende cliente
- Frontend web dedicato alla dashboard

Queste estensioni sono documentate nella sezione **Roadmap** del README come milestone future esplicite.
