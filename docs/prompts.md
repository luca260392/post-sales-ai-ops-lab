# Prompt Registry

## Introduzione

Questo documento è il registro centrale di tutti i prompt usati nei workflow n8n
del progetto **Post-Sales AI Ops Lab**. Ogni prompt è collegato a un nodo
`HTTP Request` (chiamata a `POST /v1/chat/completions` su OpenAI) all'interno del
workflow indicato.

**Versionamento:** ogni prompt ha un numero di versione (`1.0`, `1.1`, ...) e una
data di ultimo aggiornamento. Quando il testo di un prompt viene modificato:
1. Aggiornare la sezione corrispondente in questo file (system message, user
   message template, versione, data).
2. Aggiornare il nodo `HTTP Request` nel workflow n8n.
3. Ri-esportare il workflow aggiornato in `n8n/`.

**Variabili:** nei template `{{nome_variabile}}` rappresenta un campo dinamico
proveniente dal nodo precedente nel workflow (in n8n equivalgono a espressioni
del tipo `{{ $json.nome_variabile }}`).

---

## Prompt: Onboarding Email

- **Workflow:** `01_onboarding`
- **Modello consigliato:** `gpt-4o-mini`
- **Versione:** `1.0`
- **Data di creazione:** 15/06/2026

### System message

```
Sei un assistente per comunicazioni aziendali di un SaaS B2B.
Scrivi in italiano, tono professionale ma cordiale.
```

### User message template

```
Genera una email di benvenuto per un nuovo cliente con queste caratteristiche:
- Nome: {{name}}
- Piano acquistato: {{plan}}
- Data iscrizione: {{created_at}}
L'email deve: salutare il cliente, spiegare i prossimi passi, offrire supporto.
Lunghezza: 150-200 parole. Non usare markdown, solo testo piano.
```

### Esempio di output atteso

```
Ciao Mario,

grazie per aver scelto il piano Pro di [Prodotto]! Siamo felici di averti con noi.

Nei prossimi giorni il nostro team ti contatterà per guidarti nei primi passi di
configurazione. Nel frattempo, puoi accedere alla tua dashboard e iniziare a
esplorare le funzionalità incluse nel tuo piano.

Se hai domande o hai bisogno di assistenza, rispondi semplicemente a questa email:
il nostro supporto è a tua disposizione.

A presto,
Il team di [Prodotto]
```

(Testo in italiano, nessun markdown, 150-200 parole)

### Fallback

Se questo prompt fallisce (timeout OpenAI > 5000ms, definito in `AI_TIMEOUT_MS`,
o risposta non valida), viene usato il template statico in
`n8n/templates/welcome_static.txt`. Il log in `Automation Logs` registra
`status: partial` e `ai_skipped: true`.

---

## Prompt: Ticket Classification

- **Workflow:** `02_ticket_triage`
- **Modello consigliato:** `gpt-4o-mini`
- **Versione:** `1.0`
- **Data di creazione:** 15/06/2026

### System message

```
Sei un assistente di supporto tecnico. Leggi il messaggio di un cliente e
restituisci un JSON con la classificazione richiesta. Rispondi SOLO con il
JSON, senza testo aggiuntivo.
```

### User message template

```
Leggi il seguente messaggio di un cliente e restituisci un JSON con:
- "category": uno tra ["billing", "bug", "feature_request", "how_to", "other"]
- "priority": uno tra ["low", "medium", "high", "critical"]
- "confidence": numero da 0 a 1
- "summary": massimo 20 parole che riassumono il problema

Messaggio cliente:
{{message}}

Rispondi SOLO con il JSON, senza testo aggiuntivo.
```

### Esempio di output atteso

```json
{
  "category": "bug",
  "priority": "medium",
  "confidence": 0.78,
  "summary": "Cliente non riesce ad accedere all'account, errore password."
}
```

### Fallback

Se la risposta LLM non è JSON valido o la chiamata fallisce
(`Continue on Fail: true`), il ticket viene creato con `category: unknown`,
`priority: medium` e flaggato per revisione manuale. Il log in
`Automation Logs` registra `classification_error: true`.

---

## Prompt: Retention Reminder

- **Workflow:** `03_retention_reminder`
- **Modello consigliato:** `gpt-4o-mini`
- **Versione:** `1.0`
- **Data di creazione:** 15/06/2026

### System message

```
Sei un assistente per comunicazioni aziendali di un SaaS B2B.
Scrivi in italiano, tono professionale ma cordiale, con l'obiettivo di
ri-coinvolgere un cliente che non usa il prodotto da alcuni giorni.
```

### User message template

```
Genera un messaggio di re-engagement per un cliente con queste caratteristiche:
- Nome: {{name}}
- Piano acquistato: {{plan}}
- Data iscrizione: {{created_at}}
Il messaggio deve: far notare l'assenza di attività recente, ricordare il
valore del prodotto rispetto al piano sottoscritto, invitare il cliente a
tornare ad utilizzarlo e offrire supporto in caso di difficoltà.
Lunghezza: 100-150 parole. Non usare markdown, solo testo piano.
```

### Esempio di output atteso

```
Ciao Mario,

abbiamo notato che non accedi al tuo account da un po' di tempo. Il tuo piano
Pro include funzionalità che potrebbero aiutarti a ottenere ancora più valore
dal prodotto.

Se hai riscontrato difficoltà o hai domande su come utilizzarlo al meglio, il
nostro team è a disposizione per aiutarti.

Accedi quando vuoi: siamo qui per supportarti.

A presto,
Il team di [Prodotto]
```

(Testo in italiano, nessun markdown, 100-150 parole)

### Fallback

Se la chiamata OpenAI fallisce (`Continue on Fail: true`), viene inviato un
messaggio di retention statico da template
(`n8n/templates/retention_static.txt`). Il log in `Automation Logs` registra
`status: partial` e `ai_skipped: true`.
