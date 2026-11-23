# DoqFlow — SECURITY

> Bozza v0.1 — requisiti di sicurezza e threat model di alto livello.
> Complementare a `ARCHITECTURE.md`, `AGENTS.md`, `WORKFLOWS.md` e `DATABASE.md`.

---

## 1. Decisioni

| # | Decisione |
|---|---|
| S1 | Autenticazione MVP: **utenti locali con password**, creati dall'Administrator, password cifrate, sessioni a scadenza |
| S2 | Il sistema è progettato per passare in futuro a **SSO/LDAP** senza riscrivere ruoli, permessi e audit: l'identità dell'utente è separata dal metodo di accesso |
| S3 | Cloud AI **disattivato per default**; attivabile **per progetto** solo dall'Administrator |
| S4 | Con il cloud attivo possono essere inviati **anche interi documenti**, ma con **avviso prima dell'invio** |
| S5 | Avviso: **conferma esplicita a ogni invio che include più di un blocco**, con elenco di cosa esce e verso quale provider. I singoli blocchi usano l'autorizzazione di progetto |
| S6 | Ogni invio al cloud è **registrato** (cosa, a chi, perché, chi ha confermato) |
| S7 | Gli agenti non accedono al database, non eseguono comandi, non pubblicano e non modificano permessi |
| S8 | Isolamento dei progetti con controllo per riga in PostgreSQL (vedi `DATABASE.md` §7) |
| S9 | Password robuste (lunghezza minima, controllo contro password banali); **2FA opzionale per utente**, che l'Administrator può rendere obbligatoria per ruoli sensibili (es. Technical Reviewer) |
| S10 | Cifratura a riposo: **cifratura del disco/volume a livello di installazione**, raccomandata e documentata nella guida di installazione, non gestita da DoqFlow nell'MVP. Cifratura applicativa o con chiave del cliente da valutare solo su richiesta |

---

## 2. Ruoli umani

| Ruolo | Autorizzazioni |
|---|---|
| Administrator | Configurazione del sistema, utenti, prompt e permessi degli agenti, abilitazione cloud, eliminazione definitiva dei progetti |
| Project Manager | Gestione di progetti e workflow, promemoria dei gate |
| Technical Reviewer | Approvazione tecnica: gate G1, G2, G3 e blocchi di sicurezza |
| Writer | Redazione e correzione |
| Viewer | Sola consultazione |

I permessi si applicano **per progetto**: un utente può avere ruoli diversi in progetti diversi.

---

## 3. Autenticazione e sessioni

- Account creati dall'Administrator; password conservate solo come hash robusto con sale.
- Sessioni con scadenza e possibilità di revoca.
- Blocco temporaneo dopo tentativi falliti ripetuti.
- Password robuste: lunghezza minima e controllo contro password banali.
- **2FA opzionale** per ogni utente; l'Administrator può renderla obbligatoria per ruoli sensibili (per esempio i Technical Reviewer che firmano gate e blocchi di sicurezza).
- **Predisposizione a SSO/LDAP:** l'autenticazione è un modulo sostituibile; ruoli, permessi e audit dipendono dall'identità dell'utente, non dal modo in cui ha effettuato l'accesso.

---

## 4. Permessi degli agenti

- Leggono solo le fonti e i blocchi assegnati; scrivono solo nei blocchi autorizzati.
- Nessun accesso diretto al database, nessuna modifica alle fonti originali.
- Nessuna esecuzione arbitraria di comandi, nessuna pubblicazione autonoma.
- Nessuna modifica delle autorizzazioni o delle configurazioni (solo l'Administrator).
- Un Reviewer non modifica il documento; un Validator non ignora una regola fallita.

---

## 5. Cloud AI

### 5.1 Regole

- Disattivato per default; l'Administrator lo abilita **per singolo progetto**.
- Il cloud non ha accesso a file o database: riceve solo ciò che il sistema invia.
- Ogni invio passa dal Model Router, che applica autorizzazione, avviso e registrazione.

### 5.2 Cosa si può inviare

- Anche interi documenti del progetto, **con avviso** (S4).
- Il principio di partenza resta l'invio del minimo necessario; l'invio esteso è una scelta consapevole dell'utente.

### 5.3 Avviso prima dell'invio

- Conferma esplicita per ogni invio con più di un blocco.
- L'avviso mostra: quali documenti/blocchi, dimensione, provider di destinazione, motivo.
- Il rifiuto annulla l'invio; il compito può proseguire in locale o fermarsi, secondo la configurazione.

### 5.4 Registrazione

Per ogni invio: data, progetto, utente che ha confermato, provider, elenco di ciò che è stato inviato, esito e consumi. Il registro è consultabile dall'Administrator.

### 5.5 Funzionamento senza cloud

Il sistema funziona sempre end-to-end con il solo Ollama locale. Il cloud non è mai un requisito.

---

## 6. Prompt injection e contenuti non fidati

Le fonti caricate possono contenere testo che tenta di dare istruzioni al modello.

- Il contenuto di fonti e documenti è sempre **dato, mai istruzione**: viene delimitato e segnalato come tale nei prompt.
- Le risposte degli agenti sono accettate solo nel formato JSON previsto; qualsiasi altro contenuto viene scartato.
- Gli agenti non hanno strumenti pericolosi (niente comandi, niente rete, niente scritture fuori dai blocchi assegnati), quindi un'istruzione ostile non può causare azioni esterne.
- Testi sospetti nelle fonti (istruzioni rivolte all'AI) vengono segnalati come issue.

---

## 7. Dati e file

- Validazione di tipo, dimensione e struttura dei file caricati.
- I file originali non si modificano mai; l'hash ne garantisce l'integrità.
- Isolamento dei file per progetto nel filesystem.
- Strumenti esterni (Tesseract, DITA-OT, Pandoc) eseguiti con permessi minimi e senza rete.
- Cifratura a riposo affidata al disco/volume (S10); la guida di installazione la raccomanda.
- Credenziali dei provider cloud cifrate e mai riportate negli output degli agenti o nei log.

---

## 8. Audit

- Registro delle operazioni **solo in aggiunta**: nessuna modifica o cancellazione dall'applicazione.
- Registra: accessi, azioni utente, esecuzioni degli agenti, invii al cloud, approvazioni, modifiche alle configurazioni, eliminazioni di progetti.
- Le approvazioni tecniche e di sicurezza sono riconducibili a persone autorizzate.
- Le modifiche ai prompt e ai permessi degli agenti creano nuove versioni della configurazione, tutte tracciate.

---

## 9. Backup e ripristino

- Backup periodici di database e file, salvati insieme.
- Test periodico del ripristino.
- Politica predefinita da definire (vedi `DATABASE.md`, punto D4).

---

## 10. Minacce principali

| Minaccia | Mitigazione |
|---|---|
| Accesso a dati di un altro progetto | Controllo per riga in PostgreSQL, ruoli per progetto |
| Fuga di dati verso il cloud | Cloud disattivato per default, abilitazione per progetto, avviso, registro |
| Prompt injection nelle fonti | Contenuti trattati come dati, output solo JSON, nessuno strumento pericoloso per gli agenti |
| Un agente che si approva da solo | Reviewer indipendente, Supervisor deterministico, approvazioni solo umane |
| Alterazione del registro | Audit solo in aggiunta |
| Fonti manipolate | Hash di integrità, fonti originali immutabili |
| Furto di credenziali | Password con hash, sessioni a scadenza, credenziali cloud cifrate |
| Esecuzione di codice da file caricati | Validazione dei file, strumenti esterni con permessi minimi |

---

## 11. Punti aperti

| ID | Domanda |
|---|---|
| E3 | Conservazione dei registri di audit (durata) |
| E4 | Esposizione in rete: solo rete locale, o anche accesso remoto protetto |
| E5 | Se l'invio al cloud debba bloccare per default alcune fonti marcate "riservate" (da valutare dopo l'MVP) |
