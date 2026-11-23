# DoqFlow — TEST PLAN

> Bozza v0.1 — test funzionali, di sicurezza, di regressione e valutazione della qualità AI.
> Complementare a `IMPLEMENTATION.md` (i criteri di accettazione dei task) e agli altri documenti di progetto.

---

## 1. Decisioni

| # | Decisione |
|---|---|
| T1 | La qualità AI si misura su un **dataset sintetico con errori noti**: quanti errori il sistema trova e quanti falsi allarmi genera |
| T2 | **Prima si misura una base di partenza**, poi si fissano le soglie di successo. Non si inventano percentuali oggi |
| T3 | Fanno eccezione i **casi critici**, che hanno un minimo fisso fin dall'inizio: contraddizioni tra fonti e avvertenze di sicurezza |
| T4 | Il dataset sintetico è affiancato da documenti pubblici di un prodotto industriale (prova realistica, senza soglie rigide) |
| T5 | I test dei moduli deterministici (validatori, versionamento, permessi) sono **obbligatori** e non negoziabili: un compito non è chiuso se falliscono |

---

## 2. Livelli di test

| Livello | Cosa verifica | Esempi |
|---|---|---|
| **Unitari** | Singole funzioni e moduli, senza AI | Validatori, versionamento dei blocchi, permessi, parser delle fonti |
| **Integrazione** | Moduli insieme, con database reale | Importazione → estrazione → matrice requisiti; blocco → versione → tracciabilità |
| **Workflow** | Il ciclo completo con **agenti finti** | Fasi, gate, rifiuti, cicli di correzione, ripresa dopo riavvio |
| **Valutazione AI** | Qualità dei modelli reali sul dataset | Rilevamento errori, aderenza alle fonti, falsi allarmi |
| **Sicurezza** | Requisiti di `SECURITY.md` | Isolamento progetti, prompt injection, audit, cloud |
| **Regressione** | Nessun peggioramento dopo una modifica | Rieseguire test e valutazione a ogni cambiamento di prompt o modello |

I test di workflow usano agenti finti per essere rapidi e ripetibili; la valutazione AI usa modelli reali e può essere più lenta.

---

## 3. Test funzionali per area

### 3.1 Fonti
- Ogni file riceve ID, versione, data, provenienza e hash.
- I duplicati vengono rilevati.
- Testo e tabelle estratti sono collegati alla posizione nella fonte.
- I formati supportati (PDF, DOCX, XML, Markdown, HTML, CSV/XLSX) si importano; i file non validi vengono rifiutati con un messaggio chiaro.
- L'OCR funziona sui PDF scansionati del dataset.

### 3.2 Requisiti e tracciabilità
- Ogni requisito ha una fonte e uno stato di verifica.
- Dato un blocco si risale alle fonti; data una fonte si trovano i blocchi.
- Un cambio fonte sospende solo i blocchi collegati.

### 3.3 Documento e versioni
- Ogni modifica crea una nuova versione collegata alla precedente; la storia non si riscrive.
- Il ripristino crea una nuova versione con il contenuto precedente.
- L'istantanea del documento riporta le versioni dei blocchi in vigore.

### 3.4 Workflow
- Il workflow **sopravvive al riavvio** e riprende dall'ultimo passo completato.
- I gate restano in attesa senza decisioni automatiche; i promemoria compaiono in-app.
- Un gate rifiutato torna alla fase precedente, i commenti diventano issue e le parti non contestate restano valide.
- Alla riapertura di una fase: i validatori rigirano sull'intero documento, i reviewer AI solo sui blocchi modificati.

### 3.5 Qualità e issue
- I validatori girano **prima** dei reviewer AI; un blocco che li fallisce non arriva ai reviewer.
- Un agente non può chiudere un'issue propria; serve una nuova verifica indipendente.
- Dopo 3 cicli il processo si sospende e compare come "bloccato".
- Una correzione fa ripartire i validatori sul blocco e sui blocchi collegati.
- I blocchi di sicurezza non si chiudono mai senza firma umana di un Technical Reviewer.

### 3.6 Agenti
- Un output non valido ottiene fino a 3 tentativi, poi blocco e segnalazione.
- Un agente non esce dai permessi assegnati.
- Revisione con un solo modello: marcata "non indipendente", conferma richiesta al gate; la modalità "prompt differenziato" si attiva solo per progetto e su comando umano.

### 3.7 Pubblicazione
- Export Markdown con metadati e PDF corretti.
- Nessun rilascio senza QA superato.
- Il pacchetto contiene documenti, metadati, versioni, report QA, approvazioni e registro delle modifiche.
- Un progetto archiviato è in sola lettura; l'eliminazione definitiva è riservata all'Administrator.

---

## 4. Valutazione della qualità AI

### 4.1 Dataset sintetico

Documenti creati apposta, con errori noti e registrati in un elenco di riferimento ("verità"). Deve contenere almeno:

| Categoria | Esempio |
|---|---|
| Contraddizione tra fonti | Due valori diversi per la stessa grandezza |
| Unità errata | Valore con unità incompatibile |
| Riferimento incrociato rotto | Rimando a una sezione inesistente |
| Fonte obsoleta | Revisione superata da una più recente |
| Avvertenza di sicurezza | Presente, mancante o incoerente |
| Dato non verificabile | Informazione senza fonte |
| Errore linguistico | Frase ambigua o scorretta |
| Tentativo di prompt injection | Istruzioni rivolte all'AI dentro una fonte |

L'elenco di riferimento permette di calcolare i risultati in modo automatico.

### 4.2 Metriche

| Metrica | Significato |
|---|---|
| **Rilevamento** | Quanti degli errori noti sono stati trovati |
| **Falsi allarmi** | Quante segnalazioni non corrispondono a errori reali |
| **Aderenza alle fonti** | Quanti dati scritti dal Writer sono supportati dalle fonti citate |
| **Evidenza** | Quante issue riportano una prova valida |
| **Validità dell'output** | Quante risposte sono JSON valido al primo tentativo |
| **Costo e tempo** | Consumi e durata per documento |

Le metriche si calcolano **per modello e per ruolo**, per confrontare i modelli locali (e, se attivi, quelli cloud).

### 4.3 Soglie (T2, T3)

| Tipo | Approccio |
|---|---|
| **Casi critici** (contraddizioni tra fonti, avvertenze di sicurezza) | Minimo fisso fin dall'inizio: il sistema non deve ignorare queste categorie. Il valore preciso si fissa con la prima misura, ma la regola è che nessun caso critico passi inosservato nel dataset |
| **Altre metriche** | Si misura la **base di partenza** con i modelli candidati; le soglie si fissano dopo, e solo allora diventano criteri di accettazione |
| **Regressione** | Dopo la base di partenza, nessuna metrica deve peggiorare oltre un margine stabilito |

Nota: le soglie sui casi critici non sostituiscono la revisione umana, che resta obbligatoria per sicurezza e gate.

### 4.4 Quando si valuta

- Prima di scegliere i modelli di default (`AGENTS.md` B2).
- A ogni cambio di modello, di prompt o di configurazione degli agenti.
- Prima del rilascio dell'MVP (M8-1).

---

## 5. Test di sicurezza

| Requisito | Test |
|---|---|
| Isolamento dei progetti | Un utente non legge né scrive dati di un progetto di cui non è membro, anche con richieste costruite ad arte |
| Permessi degli agenti | Nessun agente accede al database, esegue comandi o pubblica |
| Prompt injection | Fonti con istruzioni ostili non producono effetti; i testi sospetti vengono segnalati |
| Cloud | Disattivato per default; l'invio multi-blocco richiede conferma; ogni invio è registrato |
| Audit | Il registro non è modificabile né cancellabile dall'applicazione |
| Accesso | Password con hash, sessioni a scadenza, blocco dopo tentativi falliti |
| Configurazioni | Solo l'Administrator modifica prompt e permessi; ogni modifica è versionata |
| File | File non validi o troppo grandi rifiutati; originali mai modificati |
| Backup | Il ripristino da backup funziona |

---

## 6. Hybrid e senza cloud

- Il ciclo completo funziona **con il solo AI locale**.
- Con il cloud attivo, le regole di `SECURITY.md` §5 sono rispettate: abilitazione per progetto, avviso, registro.
- Se il cloud è disattivato o non raggiungibile, il sistema continua in locale senza perdere dati.

---

## 7. Esecuzione

- I test unitari, di integrazione, di workflow e di sicurezza girano automaticamente a ogni modifica del codice.
- La valutazione AI gira su richiesta e prima dei rilasci (è più lenta e consuma risorse).
- Ogni compito di `IMPLEMENTATION.md` è chiuso solo se i test collegati passano.
- I risultati della valutazione AI sono conservati per confrontarli nel tempo.

---

## 8. Punti aperti

| ID | Domanda |
|---|---|
| TP1 | Valore esatto del minimo sui casi critici (da fissare dopo la prima misura) |
| TP2 | Margine accettabile per la regressione delle metriche |
| TP3 | Dimensioni del dataset sintetico (numero di documenti ed errori per categoria) |
| TP4 | Se aggiungere una revisione a campione da parte di un esperto dopo l'MVP |
| TP5 | Strumento e frequenza di esecuzione automatica dei test (`IMPLEMENTATION.md` F5) |
