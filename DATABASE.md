# DoqFlow — DATABASE

> Bozza v0.1 — schema relazionale a livello concettuale (senza SQL) e regole di gestione dei dati.
> Complementare a `ARCHITECTURE.md`, `AGENTS.md` e `WORKFLOWS.md`.

---

## 1. Decisioni

| # | Decisione |
|---|---|
| DB1 | **PostgreSQL** è l'unico database applicativo; JSONB per configurazioni e output degli agenti |
| DB2 | Il contenuto dei blocchi vive **nel database, una riga per ogni versione**; il Markdown si genera in fase di export |
| DB3 | Ogni entità ha un **ID interno univoco e immutabile** più un **codice leggibile per progetto** (es. `BLK-0042`, `SRC-001`, `REQ-003`, `REV-023`) |
| DB4 | Fine vita di un progetto: **archiviazione senza cancellazione** (sola lettura). Eliminazione definitiva solo dall'Administrator, registrata. Opzione di **esportazione in sola lettura** per lo storico |
| DB5 | Isolamento dei progetti: **un solo database, ogni riga con l'ID del progetto, controllo di sicurezza per riga in PostgreSQL** |
| DB6 | Schema per progetto separato: **non previsto**; da studiare solo se richiesto da un cliente |
| DB7 | Gli agenti **non accedono mai al database**: passano dai moduli Document Manager e Database |
| DB8 | Il filesystem conserva i file; il database conserva riferimenti e hash |

---

## 2. Principi

- **Non distruttivo:** nessuna modifica sovrascrive; ogni cambiamento crea una nuova versione collegata alla precedente.
- **Tracciabile:** ogni dato tecnico rimanda a fonte, revisione e posizione.
- **Ripristinabile:** ogni versione precedente resta disponibile.
- **Verificabile:** hash del contenuto, autore, data e motivazione di ogni modifica.
- **Ogni riga ha il progetto:** tutte le tabelle di dominio riportano l'ID del progetto.

---

## 3. Identificativi

| Elemento | Codice leggibile | Note |
|---|---|---|
| Progetto | `PROJECT-001` | |
| Fonte | `SRC-001` | |
| Requisito | `REQ-001` | |
| Blocco | `BLK-0042` | Il codice del blocco non cambia anche se il documento viene riorganizzato |
| Issue | `REV-023` | |
| Documento | `DOC-001` | |

L'ID interno è univoco e immutabile; il codice leggibile è univoco **dentro il progetto**.

---

## 4. Tabelle principali

### 4.1 Progetti e utenti

| Tabella | Contenuto | Note |
|---|---|---|
| `projects` | Progetti | Stato: attivo / archiviato; impostazioni di progetto (es. prompt differenziato) |
| `users` | Utenti | Ruoli: Administrator, Project Manager, Technical Reviewer, Writer, Viewer |
| `project_members` | Utenti per progetto e ruolo | Base dell'isolamento |

### 4.2 Fonti e requisiti

| Tabella | Contenuto |
|---|---|
| `sources` | Fonti originali: ID, versione/revisione, data, provenienza, hash, percorso |
| `source_extracts` | Testo, tabelle e riferimenti estratti, con posizione nella fonte |
| `requirements` | Requisiti, con stato di verifica (verificato / da verificare) |
| `requirement_sources` | Collegamento requisito ↔ fonte |

### 4.3 Documenti e blocchi

| Tabella | Contenuto |
|---|---|
| `documents` | Documenti del progetto (tipo, lingua, stato) |
| `document_versions` | Revisioni del documento (collega le versioni dei blocchi in una "istantanea") |
| `document_blocks` | Identità del blocco: codice, posizione, tipo, marcatura "sicurezza" |
| `block_versions` | **Una riga per ogni versione del blocco:** contenuto Markdown, hash, autore, motivazione, versione precedente, stato |
| `traceability` | Collegamenti blocco (versione) ↔ fonte/requisito, con revisione, posizione e stato di verifica |

### 4.4 Workflow, agenti e qualità

| Tabella | Contenuto |
|---|---|
| `workflows` | Stato del workflow del progetto (fase, gate, checkpoint) |
| `workflow_events` | Registro eventi: passi, esiti, tempi |
| `agents` | Configurazioni degli agenti (versionate) |
| `agent_runs` | Esecuzioni: agente, versione configurazione, modello, input, output JSON, esito, consumi |
| `reviews` | Report di revisione (anche "non indipendente" o "prompt differenziato") |
| `issues` | Anomalie: gravità, blocco, fonte, agente assegnato, stato, collegamento al ciclo |
| `approvals` | Approvazioni umane dei gate e dei blocchi di sicurezza: persona, data, esito, commenti, stato (valida / decaduta) |

### 4.5 Terminologia e traduzione (predisposte, non nell'MVP)

| Tabella | Contenuto |
|---|---|
| `glossaries` | Glossari |
| `terminology` | Termini (non traducibili, vietati, sconsigliati) |
| `translation_memory` | Segmenti tradotti |
| `translations` | Traduzioni collegate alla **revisione master** specifica |

### 4.6 Sistema

| Tabella | Contenuto |
|---|---|
| `audit_logs` | Registro delle operazioni (utenti, agenti, amministrazione); solo aggiunta |
| `model_usage` | Consumi per progetto, agente e modello; budget |
| `notifications` | Promemoria in-app (gate in attesa, issue assegnate) |
| `config_versions` | Versioni di prompt, permessi e configurazioni (modificabili solo dall'Administrator) |

---

## 5. Versionamento dei blocchi

- Il **blocco** ha un'identità stabile (`BLK-0042`).
- Ogni modifica crea una **nuova versione** del blocco, con riferimento alla precedente.
- Una **versione del documento** è un'istantanea: l'elenco delle versioni dei blocchi in vigore in quel momento.
- Il ripristino crea una nuova versione che riprende il contenuto precedente (la storia non si riscrive).
- Le traduzioni puntano alla versione master del blocco da cui derivano; se il master cambia, solo i segmenti interessati sono da aggiornare.

---

## 6. Tracciabilità e analisi d'impatto

- Ogni versione di blocco registra le sue evidenze: fonte, revisione, posizione, stato di verifica.
- Se una fonte cambia, una ricerca sulla tabella `traceability` restituisce i blocchi coinvolti, che passano a "da riverificare" (vedi `WORKFLOWS.md` §7).
- La stessa ricerca alimenterà in futuro la *Change Impact Analysis* (anche per le traduzioni).

---

## 7. Isolamento e sicurezza dei dati

- Le tabelle di dominio hanno l'ID del progetto.
- PostgreSQL applica **regole di sicurezza per riga**: ogni sessione vede solo i progetti di cui l'utente fa parte.
- Gli agenti non hanno accesso al database; operano tramite i moduli, che applicano permessi e registrano l'audit.
- `audit_logs` accetta solo inserimenti: nessuna modifica o cancellazione dall'applicazione.
- Credenziali dei provider cloud cifrate e mai riportate negli output degli agenti.

---

## 8. Ciclo di vita del progetto

| Stato | Descrizione |
|---|---|
| Attivo | Lavorazione normale |
| Archiviato | **Sola lettura**; nulla viene cancellato; audit, tracciabilità e approvazioni restano verificabili |
| Eliminato | Solo su azione dell'Administrator, registrata nell'audit; irreversibile |

**Esportazione per lo storico (opzionale):** un pacchetto completo e consultabile in sola lettura (documenti, versioni, approvazioni, report QA, registro delle modifiche), utile per conservazione e verifiche anche fuori dal sistema.

---

## 9. Migrazioni e backup

- Le modifiche allo schema avvengono solo tramite migrazioni versionate, applicate in ordine.
- Backup periodici del database e dell'area file; il ripristino deve essere verificato (test periodico).
- I file e il database vanno salvati insieme: i riferimenti e gli hash li legano.

---

## 10. Punti aperti

| ID | Domanda |
|---|---|
| D1 | Quanto conservare i log degli eventi e gli output grezzi degli agenti (spazio vs audit) |
| D2 | Formato preciso del pacchetto di esportazione per lo storico |
| D3 | Ricerca semantica con pgvector: serve già nell'MVP o solo dopo? |
| D4 | Politica di backup predefinita (frequenza, conservazione) |
| D5 | Schema per progetto separato: da valutare solo su richiesta di un cliente |
