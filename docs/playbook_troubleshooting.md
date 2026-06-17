# Playbook di Troubleshooting

## Introduzione

Questo documento è la guida operativa per diagnosticare e risolvere i problemi
più comuni nei workflow di **Post-Sales AI Ops Lab**. Usalo quando:

- un workflow segna `status: failed` o `status: partial` in `Automation Logs`;
- un cliente non riceve un'email attesa (onboarding, ticket, retention);
- la dashboard KPI mostra un `error_rate` anomalo o un alert Slack.

Per ogni scenario sono indicati: sintomo visibile, causa probabile, passi di
risoluzione e come prevenirlo in futuro.

---

## Come leggere i log

La tabella `Automation Logs` (vedi `docs/airtable_schema.md`) è il punto di
partenza per ogni indagine.

1. Apri la tabella `Automation Logs` in Airtable.
2. Filtra per `status = failed` (o `status = partial` per i casi di fallback AI).
3. Ordina per `timestamp` decrescente per vedere prima le run più recenti.
4. Usa il campo `workflow` per capire quale dei 4 workflow attivi
   (`onboarding`, `triage`, `retention`, `monitoring`) ha generato il log.
5. Leggi `payload_summary` per capire quali dati sono stati elaborati nella run.
6. Controlla i flag booleani:
   - `ai_skipped: true` → il fallback statico è stato usato al posto dell'AI.
   - `classification_error: true` → la risposta LLM non era JSON valido
     (solo Workflow 02).
7. Leggi `notes` per eventuali messaggi aggiuntivi (es. `"No clients to process"`).

---

## Scenari di errore

### 1. OpenAI non risponde / timeout

- **Sintomo:** log con `status: partial` e `ai_skipped: true`; il cliente
  riceve un'email con testo statico invece che personalizzato dall'AI.
- **Causa probabile:** rate limit OpenAI, timeout di rete (oltre i
  `AI_TIMEOUT_MS` configurati, default `5000`), o chiave API scaduta/non valida.
- **Passi di risoluzione:**
  1. Verifica lo stato e i limiti d'uso sulla dashboard OpenAI.
  2. Controlla che `OPENAI_API_KEY` in `.env` sia corretta e valida (vedi test
     con `curl https://api.openai.com/v1/models`).
  3. Ripeti manualmente il trigger del workflow (Webhook o "Execute Workflow"
     in n8n) per verificare se l'errore è temporaneo.
- **Come prevenire:** mantenere `Continue on Fail: true` sul nodo
  `HTTP Request` verso OpenAI (già previsto nei workflow 01/02/03), così il
  fallback statico entra in azione senza bloccare l'intero flusso.

---

### 2. Classificazione ticket restituisce JSON non valido

- **Sintomo:** log con `classification_error: true`; il ticket viene creato in
  `Tickets` con `category: unknown` e `priority: medium`.
- **Causa probabile:** la risposta del modello contiene testo extra attorno al
  JSON (es. spiegazioni o markdown), oppure il prompt non è stato seguito.
- **Passi di risoluzione:**
  1. Apri `docs/prompts.md`, sezione "Ticket Classification", e verifica che il
     prompt nel nodo n8n corrisponda a quello documentato (in particolare la
     riga `Rispondi SOLO con il JSON, senza testo aggiuntivo`).
  2. Testa il prompt manualmente con il payload di esempio (vedi sotto) e
     ispeziona l'output grezzo del nodo `HTTP Request`.
  3. Se necessario, rafforza il prompt e aggiorna `docs/prompts.md` (nuova
     versione, es. `1.1`).
- **Come prevenire:** mantenere il nodo `Code` che fa `JSON.parse()` dentro un
  blocco che gestisce l'errore e porta al ramo di fallback (`category:
  unknown`), come da step 4 del Workflow 02.

---

### 3. Email non inviata (onboarding o retention)

- **Sintomo:** log con `status: failed`; il cliente non riceve l'email di
  benvenuto o di reminder.
- **Causa probabile:** credenziali SMTP errate o scadute, oppure il provider
  blocca il relay (comune con provider gratuiti).
- **Passi di risoluzione:**
  1. Verifica le credenziali SMTP in n8n: **Settings → Credentials**.
  2. In sviluppo, usa Mailtrap (vedi `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`,
     `SMTP_PASS` in `.env.example`) per isolare il problema dal provider reale.
  3. Esegui manualmente il nodo `Send Email` in n8n e leggi l'errore restituito.
- **Come prevenire:** testare sempre le credenziali SMTP con un invio di prova
  in Mailtrap prima di passare a un provider di produzione.

---

### 4. Airtable rate limit

- **Sintomo:** log con `status: failed` e `notes` contenente un errore HTTP
  `429`, oppure il nodo Airtable nel workflow termina con errore durante
  l'esecuzione.
- **Causa probabile:** troppe richieste verso la stessa base Airtable in un
  breve intervallo (il piano gratuito ha un limite di richieste al secondo per
  base), tipicamente durante il loop `SplitInBatches` del Workflow 03.
- **Passi di risoluzione:**
  1. Controlla in `Automation Logs` se gli errori sono concentrati in run del
     Workflow 03 (retention, che itera su più clienti).
  2. Attendi qualche secondo e ripeti manualmente l'esecuzione.
  3. Se il problema è ricorrente, verifica il piano Airtable in uso.
- **Come prevenire:** inserire un nodo `Wait` (es. 1 secondo) tra le iterazioni
  del `SplitInBatches` nel Workflow 03, ed evitare chiamate Airtable ridondanti
  all'interno dello stesso ramo.

---

### 5. Reminder inviato a cliente già contattato

- **Sintomo:** un cliente con `reminder_count` alto riceve email di retention
  duplicate a distanza di pochi giorni.
- **Causa probabile:** il campo `last_reminder_sent` non viene aggiornato
  correttamente dal nodo Airtable Update nel Workflow 03 (step 10).
- **Passi di risoluzione:**
  1. Apri il record del cliente in `Clients` e controlla il valore di
     `last_reminder_sent`.
  2. Nel Workflow 03, verifica che il nodo Airtable Update (step 10) scriva
     correttamente sia `last_reminder_sent` che `reminder_count` e che il
     `field ID` puntato corrisponda al campo corretto.
  3. Controlla in `Automation Logs` se per quel cliente sono presenti più
     run del Workflow 03 nello stesso giorno.
- **Come prevenire:** verificare che la condizione del nodo `If` allo step 5
  (`last_reminder_sent < oggi - 7 giorni`) sia valutata correttamente con il
  formato data usato da Airtable.

---

### 6. churn_risk non aggiornato

- **Sintomo:** un cliente con `reminder_count >= 3` continua ad avere
  `status: active` e a ricevere reminder, invece di passare a `churn_risk` e
  uscire dalla coda automatica.
- **Causa probabile:** il nodo Airtable Update del ramo `reminder_count >= 3`
  (step 6 del Workflow 03) non viene eseguito, oppure aggiorna un campo/field ID
  errato.
- **Passi di risoluzione:**
  1. In `Automation Logs`, individua le run del Workflow 03 relative al
     cliente in questione e verifica che siano arrivate allo step 6.
  2. Controlla nel record `Clients` i valori di `reminder_count` e `status`.
  3. Esegui manualmente il Workflow 03 con `Execute Workflow` su un record di
     test con `reminder_count = 3` per verificare che il branch aggiorni
     `status: churn_risk`.
- **Come prevenire:** includere sempre, tra i record di test (vedi
  `docs/airtable_schema.md`), un cliente con `reminder_count = 3` per validare
  questo branch prima di ogni modifica al Workflow 03.

---

## Come testare manualmente ogni workflow

### Workflow 01 — Onboarding

Trigger: `Webhook POST /webhook/onboarding`

```json
{
  "name": "Mario Rossi",
  "email": "mario.rossi@test.com",
  "plan": "pro"
}
```

Dopo l'esecuzione, verifica:
- record creato/aggiornato in `Clients` (upsert su `email`);
- nuova task in `Tasks` con `type: onboarding` e `due_date` a +3 giorni;
- email ricevuta (Mailtrap in sviluppo);
- nuovo record in `Automation Logs` con `workflow: onboarding`.

### Workflow 02 — Ticket Triage

Trigger: `Webhook POST /webhook/ticket`

```json
{
  "client_email": "mario.rossi@test.com",
  "message": "Non riesco ad accedere al mio account, mi dice password errata ma sono sicuro sia corretta.",
  "source": "form"
}
```

Dopo l'esecuzione, verifica:
- nuovo record in `Tickets` con `category`, `priority`, `confidence_score`
  popolati;
- se `priority: critical`, alert ricevuto su Slack;
- altrimenti, email di risposta automatica al cliente;
- nuovo record in `Automation Logs` con `workflow: triage`.

### Workflow 03 — Retention Reminder

Trigger: `Schedule` (ogni giorno alle 09:00). Per test manuale:

1. In Airtable, prepara un record `Clients` di test (vedi sezione successiva
   "Reset ambiente di test").
2. In n8n, apri il workflow `03_retention_reminder` e usa il pulsante
   **Execute Workflow**.
3. Verifica il log creato in `Automation Logs` (`workflow: retention`) e,
   se applicabile, l'email ricevuta in Mailtrap.

### Workflow 04 — Voice Follow-Up (stub)

Trigger: `Manual Trigger` (workflow lasciato `Inactive`).

1. In n8n, apri il workflow `04_voice_followup_stub`.
2. Usa il pulsante **Execute Workflow**.
3. Verifica che l'esecuzione termini sul nodo `NoOp` finale con l'output
   simulato `{ "audio_url": "https://example.com/stub-audio.mp3" }`, senza
   chiamare realmente l'API ElevenLabs (nodo con `Continue on Fail: true`).

### Workflow 05 — Monitoring Board

Trigger: `Schedule` (ogni ora) + webhook on-demand. Per test manuale:

1. In n8n, apri il workflow `05_monitoring_board` e usa **Execute Workflow**.
2. Verifica che venga creato/aggiornato un record in `KPI Dashboard` con la
   data corrente.
3. Se `error_rate > 20`, verifica che venga inviato l'alert su Slack.

---

## Reset ambiente di test

Per riportare i dati Airtable a uno stato pulito prima di una nuova sessione di
test:

1. **Clients:** mantieni un solo record di test (es. Mario Rossi, vedi
   `docs/airtable_schema.md` → "Record di test"). Per testare il Workflow 03:
   - imposta `last_activity` a 20 giorni fa;
   - imposta `last_reminder_sent` a 10 giorni fa;
   - imposta `reminder_count` a `0`, `1` o `3` in base allo scenario da
     verificare (rispettivamente: reminder normale, reminder già recente da
     saltare, passaggio a `churn_risk`).
2. **Tickets:** elimina o imposta `status: closed` sui ticket creati durante i
   test del Workflow 02.
3. **Tasks:** elimina o marca `completed: true` le task di onboarding create
   durante i test del Workflow 01.
4. **Automation Logs:** non eliminare lo storico; se necessario per non
   alterare i KPI, filtra/ignora i log con `timestamp` relativi alla sessione
   di test.
5. **KPI Dashboard:** dopo il reset, esegui manualmente il Workflow 05
   (`Execute Workflow`) per ricalcolare un record KPI pulito per la data
   corrente.
