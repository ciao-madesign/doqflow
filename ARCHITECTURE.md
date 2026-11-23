# DoqFlow — ARCHITECTURE

> Bozza v0.1 — documento di riferimento per architettura e contratti tra moduli.
> Fonte: specifica "AI Documentation Harness V1.1", rivista alla luce delle decisioni prese (vedi §1).

---

## 1. Decisioni di partenza

| # | Decisione | Conseguenza |
|---|---|---|
| AR1 | DoqFlow è un **prodotto separato** da DoqTool | Repository, stack e ciclo di rilascio indipendenti |
| AR2 | Target: **PMI** (come DoqTool) + realtà più grandi | Installazione semplice di default, funzioni enterprise come moduli attivabili |
| AR3 | **Cloud AI opzionale**, mai obbligatorio | Il sistema deve funzionare end-to-end con il solo Ollama locale |
| AR4 | **Markdown** è il formato master interno | PDF, HTML e DITA sono output generati |
| AR5 | **Monolite modulare**, nessun microservizio | Un solo backend, moduli con confini chiari |
| AR6 | Collaborazione con DoqTool tramite **file di scambio**, non tramite codice condiviso | Nessuna dipendenza tecnica diretta tra i due prodotti |

---

## 2. Posizionamento rispetto a DoqTool

| | **DoqTool** | **DoqFlow** |
|---|---|---|
| Ruolo | Strumento di scrittura e consultazione quotidiano | Linea di produzione documentale con agenti AI |
| Utente tipico | Redattore tecnico (1–3 persone) | Team con ruoli (PM, writer, reviewer) |
| Installazione | Desktop, offline, un utente | Server, Docker Compose, più utenti |
| Stack | Electron, Next.js, SQLite | Python/FastAPI, LangGraph, PostgreSQL, React |
| Punto di forza | Semplicità, HMI/PLC, commesse, shop floor | Tracciabilità, peer review, audit, cicli di correzione |

**Relazione:** DoqFlow produce documenti master verificati; DoqTool li rifinisce, li aggancia a commesse/lotti e li porta in officina.
I due prodotti non condividono codice né database.

### 2.1 Confine di responsabilità

| Funzione | Dove vive |
|---|---|
| Redazione assistita dall'AI, editing manuale | DoqTool |
| Redazione automatica a blocchi, revisione multi-agente | **DoqFlow** |
| Tracciabilità requisito → fonte → blocco | **DoqFlow** |
| Issue, cicli di correzione, gate di approvazione | **DoqFlow** |
| HMI Content Editor, commesse/lotti, shop floor | DoqTool |
| Translation Memory e glossario | Entrambi (formato di scambio XLIFF/TBX) |
| Export PDF / HTML | Entrambi |
| Export XML/DITA | **DoqFlow** (priorità), DoqTool (XML proprio) |

### 2.2 Formato di scambio

| Direzione | Formato | Contenuto |
|---|---|---|
| DoqFlow → DoqTool | Markdown + metadati in intestazione | ID blocchi, fonti, revisione, stato di approvazione, lingua |
| DoqFlow ↔ DoqTool | XLIFF 2.0 | Traduzioni (già previsto in DoqTool MOD-01) |
| DoqFlow ↔ DoqTool | Glossario (CSV/TBX) | Terminologia condivisa |
| DoqTool → DoqFlow | Cartella di file | Fonti da importare |

Il pacchetto di esportazione include sempre un manifest con versioni, hash e approvazioni.
Da definire in dettaglio (punto aperto A1).

---

## 3. Vista generale

```
FRONTEND (React)
   Projects · Documents · Workflows · Agents · Reviews · Localization
          │  REST API
BACKEND (Python, FastAPI) — un'unica applicazione
   ├── api            Endpoint REST, autenticazione
   ├── core           Configurazione, logging, errori comuni
   ├── orchestrator   Workflow LangGraph, checkpoint, gate
   ├── agents         Ruoli per dipartimento, configurati in YAML
   ├── tools          Tool Registry (PDF, OCR, XML, validatori…)
   ├── models         Model Router + provider
   ├── documents      Blocchi, versioni, tracciabilità, export
   ├── database       Accesso PostgreSQL, migrazioni
   ├── security       Ruoli, permessi, audit, policy cloud
   └── integrations   Adapter esterni (post-MVP)
          │
DATI:  PostgreSQL (stato, metadati)  ·  Filesystem (file e documenti)
AI:    Ollama (locale, predefinito)  ·  API cloud (opzionale)
```

### 3.1 Regole di dipendenza tra moduli

- `api` chiama `orchestrator`, `documents`, `security`. Non chiama direttamente `agents` o `tools`.
- `orchestrator` coordina `agents`. Gli agenti non si chiamano tra loro.
- `agents` usano solo `tools` e `models`. **Non accedono mai al database.**
- Ogni lettura/scrittura di dati passa da `documents` o `database`, che applicano i permessi.
- `security` è trasversale: verifica permessi e registra l'audit.
- `integrations` è l'unico modulo che parla con sistemi esterni.

Ogni modulo espone un'interfaccia documentata ed è testabile da solo.

---

## 4. Moduli e contratti

| Modulo | Responsabilità | Riceve | Restituisce |
|---|---|---|---|
| **Project Manager** | Progetti, utenti, fonti caricate | Richieste utente | Progetto, inventario fonti |
| **Document Manager** | Blocchi, versioni, hash, tracciabilità | Contenuti prodotti | Blocchi versionati, grafo fonte→blocco |
| **Workflow Engine** | Fasi, cicli, gate, checkpoint su PostgreSQL | Stato progetto | Prossima azione, eventi |
| **Agent Manager** | Carica la configurazione YAML, applica i permessi, esegue gli agenti | Compito + blocchi assegnati | Output strutturato (JSON) |
| **Model Router** | Sceglie il modello (locale/cloud), budget, log consumi | Richiesta AI + policy | Risposta + registro d'uso |
| **Quality Manager** | Validatori deterministici, issue, cicli di correzione | Blocchi | Report QA, issue |
| **Translation Manager** | Glossario, TM, segmentazione, stato per segmento | Master congelato | Traduzioni collegate alla revisione master |
| **Security & Audit** | Ruoli, permessi, policy cloud, audit log | Ogni operazione | Autorizzato / negato + traccia |

---

## 5. Agenti

- Quattro categorie: **Worker**, **Reviewer**, **Validator**, **Supervisor**.
- Ruoli (non modelli distinti) configurati in YAML: input, strumenti, permessi, formato di output, limiti.
- Un Reviewer non modifica il documento. Un Validator non può ignorare una regola fallita. Un Supervisor non autorizza rilasci riservati all'umano.
- Il **reviewer indipendente** usa un modello diverso dal writer. Se è disponibile un solo modello, la revisione è marcata "non indipendente" e richiede conferma umana; su comando umano si può usare la modalità a prompt differenziato.
- Solo l'Administrator modifica prompt e permessi (con versione e audit). Prompt in inglese, output solo JSON validato, fino a 3 tentativi.

### 5.1 Ruoli dell'MVP

| Ruolo | Categoria |
|---|---|
| Source Analyst | Worker |
| Technical Writer | Worker |
| Technical Reviewer | Reviewer |
| Language Reviewer | Reviewer |
| Safety Reviewer | Reviewer |
| Validatori in Python (numerici, unità, struttura, riferimenti) | Validator |
| Orchestratore del ciclo (logica fissa in Python, non AI) | Supervisor |

I blocchi di sicurezza richiedono sempre la revisione umana dedicata.
Gli altri ruoli della specifica (Standards, Peer indipendente, Localization, Release) arrivano nelle milestone successive,
senza modificare l'architettura. Dettagli in `AGENTS.md`.

---

## 6. Workflow

Sei fasi e tre gate umani (come da specifica):

`Intake → Analisi → [G1] → Pianificazione → [G2] → Redazione → Review/QA → [G3] → Traduzione → Pubblicazione → [Gate finale]`

- Lo stato è salvato su PostgreSQL: i workflow **sopravvivono al riavvio**.
- Le contraddizioni tra fonti generano **issue**, non decisioni dell'AI.
- Ciclo di correzione: issue → assegnazione → correzione → **nuova verifica indipendente** → chiusura o riapertura.
- Limite tentativi configurabile (default 3); al superamento il processo si sospende.
- Gravità: Critical e Major bloccano la pubblicazione.

I dettagli (stati, transizioni, eventi) andranno in `WORKFLOWS.md`.

---

## 7. Dati

### 7.1 Database (PostgreSQL)

Unico database applicativo. Tabelle relazionali per stato e metadati, JSONB per configurazioni e output degli agenti.
pgvector solo se la ricerca semantica si dimostra utile.
Schema completo in `DATABASE.md`.

### 7.2 Filesystem

Un'area per progetto: `sources`, `working`, `master`, `translations`, `exports`, `reports`.
PostgreSQL conserva riferimenti e hash.
**Le fonti originali non vengono mai modificate.**

### 7.3 Versionamento

Applicativo, non distruttivo: versione di documento e di blocco, hash, autore, motivazione, link alla versione precedente.
Ogni versione resta ripristinabile. Git è un'integrazione futura.

### 7.4 Tracciabilità

Ogni informazione tecnica riporta le sue prove: fonte, revisione, posizione, stato di verifica.
Se cambia una fonte, il sistema individua i blocchi interessati (base della futura *Change Impact Analysis*).

---

## 8. Model Router

- Interfaccia comune per tutti i provider (generazione, consumi, informazioni sul modello).
- Provider iniziali: **Ollama** (obbligatorio per AR3). OpenAI, Anthropic, Gemini attivabili per configurazione.
- Regole di instradamento per tipo di attività: classificazione, estrazione e controllo linguistico in locale; analisi complessa, redazione e traduzione in locale o cloud secondo policy.
- Budget per progetto e per agente, limiti di token e iterazioni, registro dei consumi.
- Policy cloud: disattivato per default, abilitato per progetto dall'Administrator. Il cloud non accede mai a file o database. Prima di ogni invio: autorizzazione, avviso con conferma esplicita (se l'invio include più di un blocco; sono ammessi anche interi documenti) e registrazione. Dettagli in `SECURITY.md`.
- **Con cloud disabilitato nessuna funzione smette di funzionare** (può essere più lenta o meno accurata).

---

## 9. Sicurezza

- Autenticazione: utenti locali nell'MVP, progettata per passare a SSO/LDAP senza riscrivere ruoli e audit.
- Ruoli umani: Administrator, Project Manager, Technical Reviewer, Writer, Viewer.
- Agenti: leggono solo le fonti assegnate, scrivono solo nei blocchi autorizzati, nessun accesso al database, nessun comando arbitrario, nessuna pubblicazione.
- Protezione da prompt injection nei documenti caricati: il contenuto delle fonti è sempre trattato come dato, mai come istruzione.
- Isolamento tra progetti, audit log, approvazioni riconducibili a persone autorizzate.
- Dettagli e threat model in `SECURITY.md`.

---

## 10. Deployment

- **Predefinito:** Docker Compose con backend, frontend e PostgreSQL; Ollama locale.
- Tesseract e DITA-OT come motori esterni nel container.
- **Opzionale:** provider cloud, abilitati per progetto.
- Per realtà più grandi: stesso pacchetto con autenticazione centralizzata e integrazioni, aggiunte come moduli in fasi successive.

---

## 11. Perimetro MVP

Un ciclo reale su un **manuale di installazione industriale**:

1. Caricamento PDF e documenti tecnici
2. Estrazione informazioni e matrice requisiti
3. Struttura del manuale (gate G2)
4. Redazione con tracciabilità delle fonti
5. Revisione tecnica e linguistica indipendente
6. Almeno un'issue corretta e verificata
7. Approvazione umana del master inglese (gate G3)
8. Export Markdown e PDF

Fuori dall'MVP: traduzione, DITA, integrazioni CCMS/PLM/Git, agenti aggiuntivi.

---

## 12. Regole vincolanti

1. Nessuna nuova tecnologia senza necessità dimostrata.
2. Nessun microservizio nella V1.
3. Nessun database aggiuntivo.
4. LangGraph è l'unico framework di orchestrazione.
5. Gli agenti non accedono al database.
6. Le fonti originali non si modificano.
7. Nessuna approvazione automatica di contenuti critici.
8. Nessuna traduzione scollegata dalla versione master.
9. Nessun rilascio senza QA.
10. Nessuna dipendenza obbligatoria da un provider AI commerciale.
11. Il sistema funziona end-to-end con il solo AI locale.
12. I checkpoint dei workflow sono persistenti.

---

## 13. Punti aperti

| ID | Domanda |
|---|---|
| A1 | Formato esatto del pacchetto di scambio con DoqTool (campi dei metadati, manifest) |
| A2 | Quali modelli locali garantiscono qualità sufficiente per redazione e review? Serve un dataset di valutazione |
| A4 | Lingue della documentazione: solo inglese come master, o anche italiano? |
| A5 | Modello commerciale e licenza (come DoqTool: licenza locale perpetua?) |
| A6 | Nome definitivo dei moduli e della versione (V1 / MVP) |
