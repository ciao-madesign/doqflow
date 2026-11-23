# DoqFlow — WORKFLOWS

> Bozza v0.1 — fasi, stati, gate e cicli di correzione dell'MVP.
> Complementare a `ARCHITECTURE.md` e `AGENTS.md`.

---

## 1. Decisioni

| # | Decisione |
|---|---|
| W1 | Gate umano in attesa: il workflow **resta fermo in modo sicuro**, lo stato è salvato, il sistema invia **promemoria al Project Manager**. Nessuna decisione automatica |
| W2 | Cambia una fonte durante il lavoro: il sistema **segnala e sospende solo i blocchi interessati** (marcati "da riverificare", con issue). Gli altri proseguono |
| W3 | Gate rifiutato: il workflow **torna alla fase precedente**; i commenti dell'umano diventano issue assegnate agli agenti. Le parti non contestate restano valide |
| W4 | Ordine dei controlli: **prima i validatori in Python, poi i reviewer AI** |
| W5 | Lo stato dei workflow è salvato su PostgreSQL e **sopravvive al riavvio** |
| W6 | Il Supervisor è logica fissa in Python (vedi `AGENTS.md` §8) |
| W8 | I promemoria sono **solo in-app** (dashboard ed elenco attività) nell'MVP |
| W7 | Tutti i gate di contenuto (G1, G2, G3) sono approvati da un **Technical Reviewer umano**; il Project Manager gestisce il processo e riceve i promemoria |

---

## 2. Flusso principale (MVP)

```
Fase 0  Intake            Caricamento fonti, ID, hash, versione
Fase 1  Analisi           Source Analyst → inventario fonti, matrice requisiti
        ── Gate G1 ──     Completezza fonti e informazioni critiche
Fase 2  Pianificazione    Struttura e indice del manuale
        ── Gate G2 ──     Approvazione umana della struttura
Fase 3  Redazione         Writer, per blocchi indipendenti
Fase 4  Controllo qualità Validatori → Reviewer AI → correzioni → rivalidazione
        ── Gate G3 ──     Approvazione umana del master inglese
Fase 5  Pubblicazione     Export Markdown e PDF
        ── Gate finale ── Approvazione umana della pubblicazione
```

Traduzione e DITA sono fuori dall'MVP; il flusso è predisposto per inserirli tra G3 e la pubblicazione.

---

## 3. Fasi

### Fase 0 — Intake
- L'utente crea il progetto e carica le fonti (PDF, DOCX, XML, Markdown, HTML, CSV/XLSX tecnici).
- Ogni file riceve un ID e viene registrato con versione, data, provenienza e hash.
- Si rilevano duplicati, fonti obsolete o incomplete.
- **Uscita:** inventario delle fonti.

### Fase 1 — Analisi
- Il Source Analyst estrae testo, tabelle, immagini e riferimenti.
- Si costruisce la **matrice dei requisiti** con stato di verifica (verificato / da verificare).
- Le informazioni contraddittorie generano **issue**, non decisioni dell'AI.
- **Uscita:** inventario, matrice requisiti, elenco contraddizioni.
- **Gate G1:** l'umano conferma completezza delle fonti e delle informazioni critiche.

### Fase 2 — Pianificazione
- Si propone struttura, indice, contenuti obbligatori, fonti associate, tabelle e immagini necessarie.
- Il Document Manager crea i blocchi con ID persistente.
- **Gate G2:** l'umano approva la struttura prima della redazione.

### Fase 3 — Redazione
- Il Writer lavora **per blocchi indipendenti**: si può correggere una procedura senza rigenerare il manuale.
- Ogni blocco ha: ID persistente, contenuto, fonti, agente, versione, stato, issue aperte.
- I blocchi con avvertenze sono marcati **"sicurezza"**.

### Fase 4 — Controllo qualità
Ordine dei controlli (W4):

1. **Validatori in Python:** numerazione, unità di misura, valori numerici, riferimenti incrociati, sezioni obbligatorie, struttura, terminologia, formattazione.
2. **Reviewer AI:** Technical Reviewer, Language Reviewer, Safety Reviewer (sui blocchi di sicurezza). Possono lavorare in parallelo tra loro.
3. **Revisione umana dedicata** per i blocchi di sicurezza (firma di un Technical Reviewer umano).

Un blocco che non supera i validatori non arriva ai reviewer AI.

**Gate G3:** l'umano approva la versione master inglese. Se la revisione è marcata "non indipendente", la conferma è esplicita.

### Fase 5 — Pubblicazione
- Il Release Manager genera Markdown e PDF.
- Controlli finali sui formati e pacchetto con documenti, metadati, versioni, report QA, approvazioni, registro delle modifiche.
- **Gate finale:** approvazione umana della pubblicazione.

---

## 4. Stati

### 4.1 Stato del progetto

`Bozza → In analisi → In attesa G1 → In pianificazione → In attesa G2 → In redazione → In QA → In attesa G3 → Master approvato → In pubblicazione → In attesa gate finale → Pubblicato`

Stati trasversali: **Sospeso** (limite tentativi superato o intervento umano), **Bloccato** (output non valido dopo i tentativi massimi).

### 4.2 Stato del blocco

`Pianificato → In redazione → Validato → In revisione → Da correggere → Approvato`

Stato aggiuntivo: **Da riverificare** (fonte cambiata, W2).

### 4.3 Stato dell'issue

`Aperta → Assegnata → Corretta → In nuova verifica → Chiusa / Riaperta`

---

## 5. Issue e ciclo di correzione

| Gravità | Comportamento |
|---|---|
| Critical | Blocco immediato |
| Major | Blocca la pubblicazione |
| Minor | Correzione richiesta |
| Editorial | Correzione facoltativa o accettazione motivata |

Ciclo:

`Revisione → Creazione issue → Assegnazione → Correzione → Nuova verifica indipendente → Validatori (rivalidazione) → Chiusa / Riaperta`

- Chi ha aperto l'issue (il reviewer originale) verifica la correzione.
- Nessun agente può dichiarare risolta una propria issue.
- Limite configurabile di cicli (default 3); superato, il processo si sospende e viene segnalato all'umano.
- Una correzione fa sempre ripartire i validatori sul blocco modificato e sui blocchi collegati (controllo di regressione).

---

## 6. Gate umani

| Gate | Chi approva | Cosa verifica |
|---|---|---|
| G1 | Technical Reviewer | Completezza fonti, contraddizioni risolte |
| G2 | Technical Reviewer | Struttura del manuale |
| G3 | Technical Reviewer | Versione master inglese |
| Finale | Persona autorizzata | Pacchetto di pubblicazione |

Regole:

- Ogni approvazione è riconducibile a una persona autorizzata e registrata nell'audit.
- **In attesa (W1):** il workflow resta fermo, nessuna decisione automatica; promemoria periodici al Project Manager.
- **Rifiuto (W3):** il workflow torna alla fase precedente; i commenti diventano issue con assegnazione automatica; le parti non contestate restano valide.
- Un Supervisor o un agente non può approvare un gate.

---

## 7. Cambio di una fonte (W2)

1. L'utente carica una nuova revisione di una fonte.
2. Il sistema trova i blocchi collegati tramite la tracciabilità.
3. I blocchi interessati passano a **"Da riverificare"**; si apre un'issue per ciascuno.
4. Gli altri blocchi proseguono senza interruzioni.
5. L'umano decide per ogni blocco se aggiornare, accettare la fonte o respingere la modifica.
6. I blocchi aggiornati ripassano da validatori e reviewer.

Se il blocco era già approvato in un gate, l'approvazione viene marcata **decaduta** e richiede nuova conferma.

---

## 8. Persistenza e riprese

- Stato e checkpoint su PostgreSQL: dopo un riavvio il workflow riprende dall'ultimo passo completato.
- Ogni passo è registrato come evento (agente, esito, tempi, consumi).
- Un passo non completato viene rieseguito dall'inizio del passo, senza duplicare i risultati.

---

## 9. Punti aperti

| ID | Domanda |
|---|---|
| C1 | Frequenza dei promemoria (il canale è solo in-app nell'MVP; l'email è un'estensione futura) |
| C2 | Quali blocchi "collegati" fanno ripartire i validatori dopo una correzione (soglia della regressione) |
| C4 | Cosa succede ai blocchi già approvati se si riapre una fase precedente |
| C5 | Diagrammi formali degli stati (da aggiungere quando il flusso è stabile) |
