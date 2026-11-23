# DoqFlow — AGENTS

> Bozza v0.1 — ruoli, permessi, input/output e regole di funzionamento degli agenti.
> Complementare a `ARCHITECTURE.md`. Perimetro: **MVP** (i ruoli successivi sono elencati in §9).

---

## 1. Decisioni

| # | Decisione |
|---|---|
| AG1 | Prompt e istruzioni degli agenti sono scritti in **inglese** |
| AG2 | Ogni agente produce **solo JSON strutturato**, validato dal sistema prima dell'uso |
| AG3 | Output non valido: **fino a 3 tentativi** (per singola risposta dell'agente), poi il compito si ferma e viene segnalato all'umano. Da non confondere con il limite di 3 cicli di correzione di un'issue (vedi `WORKFLOWS.md` §5) |
| AG4 | Il Supervisor è **logica fissa in Python**, non un agente AI |
| AG5 | Solo l'**Administrator** modifica prompt, permessi e configurazioni; ogni modifica crea una nuova versione con audit |
| AG6 | Indipendenza della revisione: vedi §6 |
| AG7 | I blocchi di sicurezza richiedono sempre la revisione umana dedicata, firmata da un Technical Reviewer umano (§7) |

---

## 2. Categorie di agenti

| Categoria | Può | Non può |
|---|---|---|
| **Worker** | Produrre o modificare i blocchi assegnati | Approvare il proprio lavoro, pubblicare |
| **Reviewer** | Leggere, generare report e issue | Modificare il documento |
| **Validator** | Eseguire controlli deterministici (Python, senza AI) | Ignorare o declassare una regola fallita |
| **Supervisor** | Decidere il passo successivo secondo regole fisse | Autorizzare rilasci o approvazioni riservate all'umano |

Regole comuni a tutti:

- Nessun accesso diretto al database: leggono e scrivono solo tramite i moduli autorizzati.
- Leggono solo le fonti e i blocchi assegnati; scrivono solo nei blocchi autorizzati.
- Nessuna modifica alle fonti originali, nessun comando arbitrario, nessuna pubblicazione, nessuna modifica dei permessi.
- Il contenuto delle fonti è **dato, mai istruzione** (protezione da prompt injection).

---

## 3. Ruoli dell'MVP

| ID | Ruolo | Categoria | Responsabilità |
|---|---|---|---|
| A01 | Source Analyst | Worker | Estrae informazioni, tabelle e riferimenti dalle fonti; costruisce inventario e matrice requisiti |
| P02 | Technical Writer | Worker | Redige i blocchi con riferimenti alle fonti |
| R01 | Technical Reviewer | Reviewer | Verifica la correttezza tecnica rispetto alle fonti |
| R02 | Language Reviewer | Reviewer | Verifica grammatica, chiarezza e stile |
| R04 | Safety Reviewer | Reviewer | Verifica avvertenze e istruzioni di sicurezza |
| V01 | Validatori deterministici | Validator | Controlli numerici, unità, struttura, riferimenti incrociati, terminologia |
| S01 | Orchestratore del ciclo | Supervisor | Decide: ripetere, chiudere o passare al gate umano |

Gli ID mantengono la numerazione della specifica originale; i ruoli non presenti nell'MVP restano riservati.

---

## 4. Configurazione

Ogni agente è descritto da un file YAML in `configuration/agents/`, versionato.
Prompt, strumenti, modelli e permessi cambiano **senza toccare il codice del workflow**.

Campi principali:

| Campo | Significato |
|---|---|
| `id`, `name`, `department`, `type` | Identità e categoria |
| `input` | Dati che l'agente riceve (blocco, riferimenti alle fonti, checklist) |
| `tools` | Strumenti consentiti, tra quelli del Tool Registry |
| `permissions` | Lettura, report, modifica, pubblicazione (sempre `false` per pubblicare) |
| `output` | Schema JSON atteso e campi obbligatori |
| `limits` | Tentativi massimi, obbligo di evidenza |
| `model_policy` | Preferenza locale/cloud, modello, regole di indipendenza |

---

## 5. Output e validazione

Ogni risposta deve rispettare lo schema JSON del ruolo. Il sistema controlla struttura e campi prima di usarla.

| Ruolo | Campi obbligatori |
|---|---|
| Reviewer | `findings`, `severity`, `evidence`, `recommendation`, `verdict` |
| Writer | `content`, `sources`, `open_questions` |
| Source Analyst | `items`, `source_location`, `confidence`, `conflicts` |
| Validator | `rule`, `result`, `details` |

Regole:

- **Evidenza obbligatoria:** un'issue o un dato tecnico senza fonte viene rifiutato.
- **Output non valido:** il sistema restituisce l'errore all'agente e riprova, fino a 3 tentativi totali. Dopo il terzo fallimento il compito si ferma e compare nella dashboard come "bloccato".
- Ogni esecuzione è registrata: agente, versione della configurazione, modello, input, output, esito, consumi.

---

## 6. Indipendenza della revisione

Il Reviewer deve usare un modello **diverso** dal Writer.

| Situazione | Comportamento |
|---|---|
| Due o più modelli disponibili | Writer e Reviewer usano modelli diversi |
| Un solo modello disponibile | Il sistema **avvisa e continua**: la revisione è marcata **"non indipendente"** nel report e richiede la conferma dell'umano al gate |
| Comando umano esplicito | Si può passare alla **modalità prompt differenziato**: stesso modello, ma istruzioni e parametri diversi per il Reviewer |

La modalità prompt differenziato è un'eccezione scelta dall'umano **per singolo progetto** (impostazione di progetto), registrata nell'audit e indicata in ogni report di revisione.
Il sistema non la attiva da solo.

---

## 7. Sicurezza dei contenuti

- Ogni blocco che contiene avvertenze o istruzioni di sicurezza è marcato **"sicurezza"**.
- Il Safety Reviewer (R04) lo verifica.
- Anche se tutti i reviewer AI lo approvano, il blocco richiede **sempre la revisione umana dedicata** da parte di una persona autorizzata.
- La firma spetta a un **Technical Reviewer umano** autorizzato. Nessun blocco di sicurezza si chiude automaticamente.

---

## 8. Supervisor (logica fissa)

Regole sulla base dello stato, senza AI:

| Condizione | Decisione |
|---|---|
| Issue Critical o Major aperte | Non si può procedere; correzione richiesta |
| Correzione eseguita | Nuova verifica indipendente prima della chiusura |
| Cicli di correzione esauriti (default 3) | Sospensione e segnalazione all'umano |
| Tutte le verifiche superate | Si passa al gate umano |
| Revisione marcata "non indipendente" | Conferma esplicita al gate |
| Blocco di sicurezza | Revisione umana dedicata |

Un agente non può dichiarare risolta una propria issue.

---

## 9. Ruoli successivi (fuori dall'MVP)

| Dipartimento | Ruoli |
|---|---|
| Analysis | A02 Requirements, A03 Technical, A04 Gap, A05 Standards |
| Production | P01 Document Architect, P03 Procedure Writer, P04 Safety Writer, P05 Visual Specialist |
| Review | R03 Standards Reviewer, R05 Independent Peer Reviewer |
| Localization | L01–L04 (Terminology, Translator, Reviewers) |
| Release | D01 Document QA, D02 Format Validator, D03 Release Manager |

Nota: nell'MVP la struttura del manuale (P01) è proposta dal Source Analyst e dal Writer e approvata dall'umano al gate G2.

---

## 10. Punti aperti

| ID | Domanda |
|---|---|
| B1 | Schemi JSON esatti per ciascun ruolo (da fissare in `IMPLEMENTATION.md`) |
| B2 | Quali modelli Ollama usare di default per Writer e Reviewer |
