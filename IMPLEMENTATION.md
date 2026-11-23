# DoqFlow — IMPLEMENTATION

> Bozza v0.1 — task tecnici ordinati e criteri di accettazione dell'MVP.
> Fonte di verità per gli agenti di coding, insieme a `ARCHITECTURE.md`, `AGENTS.md`, `WORKFLOWS.md`, `DATABASE.md`, `SECURITY.md`.

---

## 1. Decisioni

| # | Decisione |
|---|---|
| I1 | Ordine di costruzione: **per strati, partendo dal nucleo** (backend, database, accesso, workflow con agente finto), poi funzioni, poi interfaccia completa |
| I2 | Lo sviluppo è affidato ad **agenti di coding guidati da questi documenti**, con compiti piccoli, criteri di accettazione e test da superare |
| I3 | Materiale di prova: **documenti pubblici** (un inverter o azionamento + una macchina o componente meccanico, per coprire dati elettrici e procedure di montaggio/manutenzione) + **documenti sintetici creati apposta** (con errori e contraddizioni note) |
| I4 | Prima di scrivere codice per un compito, l'agente di coding rilegge il documento di riferimento; se trova un conflitto o un vuoto, **si ferma e lo segnala**, senza decidere da solo |
| I5 | Nessuna nuova tecnologia oltre allo stack definito (vedi `ARCHITECTURE.md` §12) |

---

## 2. Regole per gli agenti di coding

1. Un compito alla volta, con il suo criterio di accettazione.
2. Ogni compito include i test; un compito non è finito finché i test non passano.
3. Nessuna modifica all'architettura, allo stack o ai contratti tra moduli senza decisione umana.
4. Le dipendenze tra moduli rispettano `ARCHITECTURE.md` §3.1 (gli agenti AI non toccano il database).
5. I punti aperti dei documenti non si risolvono "per intuito": si segnalano.
6. Commit piccoli e descrittivi; la documentazione si aggiorna insieme al codice.

---

## 3. Milestone dell'MVP

| ID | Milestone | Risultato |
|---|---|---|
| M0 | Fondamenta | Applicazione avviabile con Docker Compose, database, accesso utenti, audit |
| M1 | Workflow e agenti finti | Workflow persistente con checkpoint; agenti simulati; gate umani |
| M2 | Fonti | Importazione, estrazione, inventario, matrice requisiti |
| M3 | Documento | Struttura, blocchi versionati, tracciabilità |
| M4 | Redazione | Writer con modelli reali (Ollama), output JSON validato |
| M5 | Qualità | Validatori in Python, reviewer AI, issue e cicli di correzione |
| M6 | Pubblicazione | Export Markdown e PDF, pacchetto di rilascio |
| M7 | Interfaccia | Schermate complete per l'intero ciclo |
| M8 | Rifinitura | Sicurezza, valutazione AI sul dataset, documentazione utente |

Ordine per strati (I1): M0 e M1 costruiscono il nucleo e fanno emergere presto i rischi di orchestrazione; l'interfaccia completa (M7) arriva dopo le funzioni, con una dashboard minima già da M1 per seguire i test.

---

## 4. Task ordinati

### M0 — Fondamenta

| ID | Task | Criterio di accettazione |
|---|---|---|
| M0-1 | Struttura del repository e Docker Compose (backend, frontend, PostgreSQL) | `docker compose up` avvia tutti i servizi; controllo di salute del backend |
| M0-2 | Migrazioni del database e schema di base (progetti, utenti, ruoli) | Le migrazioni si applicano da zero e in sequenza |
| M0-3 | Accesso utenti locali (password con hash, sessioni a scadenza) | Login e logout; blocco dopo tentativi falliti; l'identità è separata dal metodo di accesso (S2) |
| M0-4 | Ruoli e permessi per progetto | Un utente non vede i progetti di cui non è membro |
| M0-5 | Isolamento per riga in PostgreSQL | Test che verifica l'impossibilità di leggere dati di un altro progetto |
| M0-6 | Audit log solo in aggiunta | Operazioni registrate; modifica/cancellazione dall'applicazione impossibile |
| M0-7 | Configurazione versionata (agenti, modelli) e permessi di modifica | Solo l'Administrator modifica; ogni modifica crea una nuova versione |

### M1 — Workflow e agenti finti

| ID | Task | Criterio di accettazione |
|---|---|---|
| M1-1 | Workflow su LangGraph con checkpoint su PostgreSQL | Dopo un riavvio il workflow riprende dall'ultimo passo completato |
| M1-2 | Fasi e stati (`WORKFLOWS.md` §2–4) | Transizioni valide e non valide testate |
| M1-3 | Gate umani G1, G2, G3 e finale | Il workflow resta fermo in attesa; approvazione/rifiuto registrati; rifiuto torna alla fase precedente con i commenti come issue |
| M1-4 | Promemoria in-app per i gate in attesa | Compaiono nella dashboard e nell'elenco attività |
| M1-5 | Agent Manager: caricamento YAML, permessi, validazione dell'output JSON | Output non valido: fino a 3 tentativi, poi "bloccato" |
| M1-6 | Agenti finti per tutti i ruoli dell'MVP | Il ciclo completo gira end-to-end con risposte simulate |
| M1-7 | Supervisor deterministico | Regole di `AGENTS.md` §8 coperte da test |
| M1-8 | Dashboard minima (stato workflow, gate, attività) | Visibile cosa è automatico, cosa attende l'umano, cosa è bloccato |

### M2 — Fonti

| ID | Task | Criterio di accettazione |
|---|---|---|
| M2-1 | Caricamento file con ID, versione, data, provenienza, hash | Ogni file è identificabile; duplicati rilevati |
| M2-2 | Estrazione da PDF (testo, tabelle, immagini) e OCR | Testo e tabelle leggibili e collegati alla posizione nella fonte |
| M2-3 | Estrazione da DOCX, XML, Markdown, HTML, CSV/XLSX | Come sopra per ogni formato |
| M2-4 | Source Analyst con modello reale | Inventario fonti e requisiti estratti con evidenza |
| M2-5 | Matrice dei requisiti con stato di verifica | Requisiti collegati alla fonte, stato "verificato / da verificare" |
| M2-6 | Rilevamento delle contraddizioni | Le contraddizioni generano issue e bloccano il gate G1 |
| M2-7 | Protezione da prompt injection nelle fonti | Test con fonti contenenti istruzioni ostili: nessun effetto |

### M3 — Documento

| ID | Task | Criterio di accettazione |
|---|---|---|
| M3-1 | Documenti, blocchi e versioni (una riga per versione) | Ogni modifica crea una nuova versione collegata alla precedente |
| M3-2 | Pianificazione: struttura e indice proposti | Struttura approvabile al gate G2 |
| M3-3 | Tracciabilità blocco ↔ fonte/requisito | Dato un blocco si risale alla fonte; data una fonte si trovano i blocchi |
| M3-4 | Ripristino di una versione precedente | Il ripristino crea una nuova versione; la storia resta intatta |
| M3-5 | Cambio fonte: blocchi interessati "da riverificare" | Solo i blocchi collegati si sospendono (`WORKFLOWS.md` §7) |

### M4 — Redazione

| ID | Task | Criterio di accettazione |
|---|---|---|
| M4-1 | Model Router con provider Ollama | Il ciclo funziona con il solo AI locale |
| M4-2 | Writer su blocchi indipendenti | Contenuto Markdown con fonti citate; output validato |
| M4-3 | Marcatura dei blocchi di sicurezza | I blocchi con avvertenze risultano marcati |
| M4-4 | Budget, limiti di token e registro dei consumi | I consumi per progetto e agente sono consultabili |
| M4-5 | Provider cloud opzionali con avviso e registro (S3–S6) | Cloud disattivato per default; invio multi-blocco con conferma; ogni invio registrato |

### M5 — Qualità

| ID | Task | Criterio di accettazione |
|---|---|---|
| M5-1 | Validatori: numerazione, unità, valori, riferimenti incrociati, sezioni obbligatorie, struttura, terminologia, formattazione | Ogni validatore ha test con casi corretti e sbagliati |
| M5-2 | Ordine dei controlli: validatori prima dei reviewer AI | Un blocco che fallisce i validatori non arriva ai reviewer |
| M5-3 | Technical, Language e Safety Reviewer | Output strutturato con evidenza obbligatoria |
| M5-4 | Gestione dell'indipendenza (stesso modello: avviso; prompt differenziato per progetto su comando umano) | La revisione "non indipendente" è marcata e richiede conferma al gate |
| M5-5 | Issue e ciclo di correzione | Nessun agente chiude un'issue propria; nuova verifica indipendente; limite di 3 cicli |
| M5-6 | Regressione dopo una correzione (`WORKFLOWS.md` W10) | Ricontrollo del blocco e dei blocchi collegati |
| M5-7 | Revisione umana dei blocchi di sicurezza | Nessun blocco di sicurezza si chiude automaticamente |

### M6 — Pubblicazione

| ID | Task | Criterio di accettazione |
|---|---|---|
| M6-1 | Export Markdown con metadati (formato di scambio con DoqTool) | Il file riporta ID blocchi, fonti, revisione, stato |
| M6-2 | Export PDF | PDF generato e verificato |
| M6-3 | Pacchetto di rilascio (documenti, metadati, versioni, report QA, approvazioni, registro modifiche) | Pacchetto completo e consistente |
| M6-4 | Gate finale e blocco senza QA | Nessun rilascio senza controlli superati |
| M6-5 | Archiviazione del progetto ed esportazione storica in sola lettura | Il progetto archiviato è in sola lettura; l'eliminazione definitiva solo per l'Administrator |

### M7 — Interfaccia

| ID | Task | Criterio di accettazione |
|---|---|---|
| M7-1 | Progetti e fonti | Creazione progetto, caricamento fonti, inventario |
| M7-2 | Documenti | Editor, struttura, versioni, confronto |
| M7-3 | Workflow | Stato, gate, promemoria, attività automatiche/umane/bloccate |
| M7-4 | Agenti | Esecuzioni e risultati |
| M7-5 | Revisioni | Issue, approvazioni, revisione dei blocchi di sicurezza |
| M7-6 | Amministrazione | Utenti, configurazioni degli agenti, abilitazione cloud, registro invii |

### M8 — Rifinitura

| ID | Task | Criterio di accettazione |
|---|---|---|
| M8-1 | Valutazione AI su dataset di riferimento | Metriche di qualità documentate (vedi `TEST_PLAN.md`) |
| M8-2 | Verifica dei requisiti di sicurezza | Tutti i punti di `SECURITY.md` coperti da test |
| M8-3 | Backup e ripristino | Ripristino testato |
| M8-4 | Guida utente e documentazione di installazione | Un nuovo utente completa il ciclo seguendo la guida |

---

## 5. Materiale di prova

| Materiale | Scopo |
|---|---|
| **Documenti pubblici** di un inverter/azionamento e di una macchina o componente meccanico (manuali e schede tecniche) | Prova realistica di importazione ed estrazione, con dati elettrici e procedure meccaniche |
| **Documenti sintetici** creati apposta | Contengono errori e contraddizioni noti: permettono di misurare cosa il sistema rileva e cosa no |

Il dataset sintetico deve includere almeno: un valore tecnico contraddittorio tra due fonti, un'unità di misura errata, un riferimento incrociato rotto, una fonte obsoleta e un'avvertenza di sicurezza.
Metriche e criteri in `TEST_PLAN.md`.

---

## 6. Criteri di completamento dell'MVP

L'MVP è completo quando, sul caso di prova (manuale di installazione), il sistema:

1. Importa PDF e documenti tecnici e li rende identificabili.
2. Genera la matrice dei requisiti e rileva le contraddizioni, bloccando il gate G1.
3. Propone una struttura approvabile al gate G2.
4. Redige i blocchi con tracciabilità delle fonti.
5. Esegue validatori e revisione tecnica, linguistica e di sicurezza.
6. Registra almeno un'issue e ne verifica la correzione con una nuova verifica indipendente.
7. Richiede la revisione umana dei blocchi di sicurezza.
8. Ottiene l'approvazione umana del master inglese (G3).
9. Genera Markdown e PDF e un pacchetto di rilascio.
10. Funziona con il solo AI locale e, in più, può usare il cloud rispettando le regole di `SECURITY.md`.

---

## 7. Fuori dall'MVP

Traduzione e localizzazione, DITA, agenti aggiuntivi (Standards, Peer indipendente, Document Architect, Procedure/Safety Writer, Visual Specialist), integrazioni CCMS/PLM/Git, SSO/LDAP, notifiche via email, schema per progetto separato.
L'architettura è predisposta per aggiungerli senza ristrutturazioni.

---

## 8. Punti aperti

| ID | Domanda |
|---|---|
| F1 | Scelta dei documenti pubblici precisi: categoria decisa (inverter/azionamento **e** macchina o componente meccanico come pompa o riduttore), restano da scegliere i prodotti e verificarne la licenza di riuso |
| F2 | Quali modelli Ollama usare di default (da decidere con prove sul dataset, `AGENTS.md` B2) |
| F3 | Schemi JSON esatti per ciascun ruolo (`AGENTS.md` B1), da fissare in M1-5 |
| F4 | Hardware minimo e raccomandato per l'installazione predefinita |
| F5 | Strumento di integrazione continua (esecuzione automatica dei test) |
