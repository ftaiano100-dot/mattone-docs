# Usai — Domos Sardinia S.R.L.S · audit Conduit e piano di intervento

**Workspace Conduit** MCP `conduit-students` — id nel repo privato
**Cliente** Giuseppe Usai · sender `ota@gusaipropertyconsulting.it`
**Portafoglio** 50+ listing, tutti sincronizzati KrossBooking — Stintino, Castelsardo, Alghero, Sassari, Lu Bagnu, Valledoria
**Audit** 30/08/2026, sola lettura. Cliente ATTIVO: nessun intervento senza ok di Francesco e di Usai.

> 🔒 **Versione ridotta.** `mattone-docs` è un repo GitHub **pubblico**: `.mintignore` tiene
> questa cartella fuori dal sito docs.mattone.co, **non fuori da GitHub**.
> Perciò qui **codici di accesso e identificativi tecnici sono oscurati**.
> La versione completa, con codici e id, sta nel repo privato
> `mattone-integrations-docs`, in `clienti/usai-domos-sardinia.md`.

---

## Cosa funziona già bene

Vale dirlo, perché condiziona le priorità: la parte "intelligente" è fatta bene, quella rotta è l'impianto.

- **48 guardrail sull'agente ospite**, scritti bene: de-escalation, allineamento alla lingua, niente sconti, rilascio codici alle 15:50, policy fatture e IVA, selezione della proprietà corretta. È il pezzo più solido del workspace.
- **4 agenti chat + 1 vocale**, divisi per inbox in modo coerente. Ospiti, Compliance e Proprietari sono in **copilot** (l'umano approva): postura corretta visto tutto il resto.
- **I workflow legati alle prenotazioni girano oggi**: check-in reminder, final warning, conferma portale, sentiment review.
- **3 template WhatsApp approvati** da Meta.
- Runtime sano: 4 run falliti in assoluto, tutti il 03/06/2026, tutti errori infrastrutturali Conduit.

---

## 🔴 P1 — I codici delle cassette di tutte le proprietà sono raggiungibili da una chat ospite

**Nodo** "Codici Accesso – Tutte le Proprietà".
`entity_ids: []`, `allowed_agents: null`, `effective_allowed_agents: null`.
Contiene il codice di default **«codice di default»** per ~50 proprietà, le eccezioni (due eccezioni) e un codice cancello.

**Evidenza.** Query `semantic_knowledge_search` limitata al listing 1 (Villa Panorama), domanda
*"come entro in casa? qual è il codice della cassetta delle chiavi?"* → **primo risultato,
rilevanza 0,82, con la tabella dei codici dentro il chunk restituito**.

**Perché succede.** Il nodo sta sotto la directory "Internal – Team" (`6a130568…`), che ha
`allowed_agents: [qn8b6arrs2zw1553yt8ds6s29n87bcph]`. **La restrizione non scende ai figli**:
la directory mostra `effective_allowed_agents` popolato, i documenti dentro mostrano `null`.
L'ACL protegge la cartella, non i documenti.

**Impatto.** L'unica barriera fra un ospite e tutti i codici del portafoglio è il testo di un
guardrail ("NEVER provide key box codes"), cioè l'obbedienza del modello, non un permesso.
Vale anche per l'agente vocale. E poiché i codici sono quasi tutti identici, una fuga sola
compromette tutte e cinquanta le proprietà insieme.

**Da verificare prima di allarmare il cliente.** La query è stata fatta col token API, non
impersonando l'agente ospite. È certo che il nodo non ha filtri applicabili e che esce primo;
la prova definitiva è una conversazione di test su un listing reale, oppure il pannello
accessi in dashboard.

**Come si risolve.**
1. Creare un nodo per listing con `entity_ids: [<listing_id>]` **e** `allowed_agents` valorizzato
   **sul nodo**, non solo sulla cartella.
2. Cancellare il nodo globale.
3. ⚠️ `entity_ids` **non è modificabile in update**: serve il pattern ricrea-e-cancella del
   playbook (crea con entity_ids → verifica il body → cancella il vecchio → rinomina togliendo " (1)").
4. Aprire un ticket a Conduit: la mancata cascata della ACL cartella→figli è un comportamento
   da chiarire. Se dovrebbe cascare, è un bug loro.

---

## 🔴 P2 — L'escalation non funziona, su due livelli che si sommano

**Livello 1 — mancano i tool.** L'agente ospite ha **14 tool su 244** abilitati. Spenti:
`assign_escalation`, `list_escalations`, `resolve_escalation`, `hopr_create_task`, `leave_note`,
`create_ticket_note`, `get_reservation`, `list_contact_reservations`, `search-reservations`.
L'agente Check-in Compliance ha gli stessi 14. Ma i loro guardrail dicono *"Always create a
follow-up task ... and escalate the conversation"* e *"After Each Message Create an internal
note"*. Sono istruzioni che l'agente non ha modo di eseguire.

**Livello 2 — i workflow sono morti.** Tutti e quattro gli `AI_TRIGGER`:

| Workflow | Stato | Ultima esecuzione |
|---|---|---|
| `Escalations` | ACTIVE | **28/05/2026** — tre mesi fa |
| `Schedule Maintenance Intervention (Imported)` | ACTIVE | ultime due **FAILED / GENERIC_ERROR**, 28/05 |
| `Early and Late Escalation (Imported)` | ACTIVE | **mai eseguito** |
| `Gate Remotely Opening`, `Cleaning Issue Escalation`, `Additional Supplies Escalation` | DRAFT | — |

**Impatto.** Il formato di escalation che gli agenti sono istruiti a produrre
(`@antonio @giuseppe / @kasia / @beatrice`) genera testo che non arriva a nessuno. Segnalazioni
di manutenzione, conferme di early/late check-in e blocchi di conformità si fermano alla bozza.

**Come si risolve.** Riabilitare i tool via `set_agent_tool_access` sugli agenti Ospiti e
Compliance; poi `get_workflow` sul grafo dei tre workflow rotti per capire dove si spezzano,
prima di riattivarli.

---

## 🟠 P3 — Bleed fra listing: l'anti-pattern AP1 del playbook, riprodotto

Tutti i nodi property-specific manuali hanno `entity_ids: []`.

| Nodo | Problema |
|---|---|
| `Casa Sofia — no WiFi` `<node-id>` | body **in spagnolo** in un workspace italiano; esce chiedendo il WiFi del listing **88 (Angedras)** — proprietà sbagliata |
| `Angedras – Accessi e Operativo` `<node-id>` | scheda operativa interna con keybox «codice», nome della cleaner, link Drive, prezzo pulizia €120; esce a **0,75** sulla query di Villa Panorama |
| `Stintino (1)`, `Angedras – Informazioni Proprietà`, `La Pelosa FAQ`, `beaches`, `Stintino – How to Arrive` | stesso difetto |

Solo tre nodi sono scopati correttamente: `Castelsardo` e `STINTINO` con entity id di gruppo,
`L'OTRE DI EOLO` con `entity_ids: ["4"]`.

---

## 🟠 P4 — 183 nodi da scraping web inquinano la ricerca, e sponsorizzano i concorrenti

`knowledge_search(source: "link")` → **183 nodi**.

- **Pubblicità ai concorrenti.** Lo scrape Airbnb di `Kian House` (`<node-id>`)
  include il blocco *"Altri alloggi nelle vicinanze"*: **8 strutture rivali di Stintino** con
  prezzi e link Airbnb cliccabili (In Barca €1.116, Casa Bella Vista €481, la Casa del
  Pescatore €305…). Contraddice direttamente il loro guardrail *"Never mention availability of
  listings... We don't want to sponsor competitors."*
- **Spazzatura pura.** `<node-id>` "Page Sitemap.Xml" è la sitemap di
  domossardinia.it, migliaia di URL `.png` ripetuti. Occupa **3 dei primi 4 risultati** sulla
  query del WiFi.
- **Policy in conflitto.** Domanda *"a che ora è il check-in?"* sul listing **14 (IL TERRAZZO,
  Castelsardo)** → ai posti 2 e 3 escono le pagine Airbnb di **Casa Arborea** e **Kian House**,
  con finestre di check-in di altre proprietà, una delle quali contraddice la policy ufficiale
  16:00–22:00 e la late fee di €50 dopo le 22.

---

## 🟠 P5 — Le credenziali WiFi non esistono da nessuna parte

La query sul WiFi torna `status: "weak_match"` (percentile 0,05). Nessun nodo WiFi per listing.
L'unica scheda operativa che ne parla lo dice esplicitamente: *"WiFi: rete e password da
completare in Notion"*. Un ospite che chiede il WiFi riceve XML di sitemap o il "no WiFi" di
un'altra casa.

---

## 🟡 P6 — Tutta la sincronizzazione della knowledge base è spenta

Ogni sorgente ha `sync_cadence: "off"`.

| Sorgente | Ultimo sync | Ritardo |
|---|---|---|
| **PMS KrossBooking** `<id>` | 23/07/2026 | 38 giorni |
| **Notion** (House Manuals IT/EN, Strutture, Numeri Utili) | 29/05/2026 | 3 mesi |

Listing nuovi, cambi di prezzo, amenity e orari di check-in avvenuti dopo quelle date non sono
nella KB.

---

## 🟡 P7 — Agente vocale mal configurato

`<id>` "Domos Sardinia Assistant":

- Numero **+1 415 449 9931** — un numero **americano** per un operatore sardo
- `escalation_enabled: false`, `fallback_phone_number: null` → non può passare una chiamata a un umano
- Orario Lun-Sab `{"start": "06:00 PM", "end": "09:00 AM"}` — **inizio dopo la fine**. Domenica
  è l'unica finestra ben formata (07:00–23:00). Come Conduit risolva un intervallo invertito
  non è deducibile dall'API
- `Send Summary to Slack When AI Answers a Call` fermo dal **06/08** → 24 giorni senza chiamate
- `Missed Calls (Imported)` è DRAFT → le chiamate perse non vengono raccolte

---

## 🟡 P8 — Circa il 7% dei check-in reminder fallisce in silenzio

`Check in reminder` `<workflow-id>`: sulle 60 esecuzioni più recenti
(15–30 agosto), **4 FAILED con `API_REQUEST_FAILED`** — 22, 23, 28 e 30 agosto.
Anche `Guest porta-confimation` è fallito oggi alle 08:21 prima di riuscire alle 08:37.

Ogni fallimento è **un ospite che non ha ricevuto il link del portale** — e per policy, senza
portale niente codice all'arrivo.

---

## 🟡 P9 — Duplicazione di skill e nodi che degrada la ricerca semantica

- **46 skill**, con cluster sovrapposti evidenti: 6 sulla verifica del check-in online, 3 sulle
  recensioni Google, 3 sull'invio del link, 3 sull'early check-in
- **Skill assegnate agli agenti sbagliati**: "Assistente Team" (interno, autopilot) porta 20
  skill rivolte all'ospite fra cui `car-rental-cross-selling` e
  `request-google-review-after-positive-stays`. Stesso per "email automatiche". "Assistente
  Proprietari" ne ha 29
- **Nodi duplicati**: `Online Check-in & Guest Verification Policy` vs `…Requirements`; due
  `House Manual`; quattro varianti su Stintino
- **Residui**: 5 nodi intitolati `Untitled`, i suffissi `(1)` mai rinominati dopo un
  ricrea-e-cancella, un titolo malformato `Quick Facts & "Gotchas`, un workflow
  `Guest porta-confimation`
- **7 import di workflow duplicati/abbandonati**

---

## 🟡 P10 — Automazione cleaner: template approvato, nessuno lo manda

`pulizie_reminder` è approvato da Meta, ma `Cleaners reminder before arrival` è **INACTIVE** e
`Notifica Cleaner - Last Minute (Imported)` è **DRAFT**. Anche `Automated Email Routing` è
INACTIVE. Minore: `checkinreminder` e `pulizie_reminder` hanno corpo italiano registrato con
`language: "en"`.

---

## ⚪ Custom tool: non ce ne sono

`list_custom_tools` → `{"data": []}`, e nessuna skill ha `allowed_tools` popolato.
Quindi il difetto noto dei "tool silenziosi e sbagliati" (che affermano successo su errori
transitori) **non può essere presente**: non c'è codice custom in questo workspace.

**Ma è anche il motivo per cui manca tutto il resto.** Senza custom tool l'agente non può
aprire un cancello, generare un link di pagamento, verificare una disponibilità o creare un
task. È la differenza fra un assistente che risponde e uno che fa.

---

## Cosa non è verificabile via API

- Se l'agente ospite recuperi davvero il nodo dei codici **a runtime** — serve una
  conversazione sandbox o il pannello accessi in dashboard
- L'effetto reale dell'orario vocale invertito
- **Lo stato di salute dei canali.** `list_workspace_senders` mostra la configurazione — email
  `ota@gusaipropertyconsulting.it`, SMS `+39 070 705 4279`, SMS `+1 415 449 9931`, WhatsApp
  `whatsapp:+390707054279` — ma **non espone connesso/verificato/errore**. Se il numero WhatsApp
  Business sia vivo e se il fisso italiano possa davvero consegnare SMS richiede la dashboard
- La causa di `API_REQUEST_FAILED`: serve `get_workflow` sul grafo più il dettaglio esecuzione
- `knowledge_type` per nodo non è restituito da nessun endpoint in questa versione dell'API

---

## PIANO DI INTERVENTO

### Fase 1 — sicurezza, subito (nostra, via API)
- [ ] Spezzare il nodo codici in nodi per listing, con `entity_ids` **e** `allowed_agents` sul nodo
- [ ] Cancellare il nodo globale
- [ ] Riscopare gli orfani: `Casa Sofia — no WiFi` → 57 (e riscriverlo in italiano),
      `Angedras – Accessi e Operativo` e `– Informazioni Proprietà` → 88
- [ ] Cancellare il nodo sitemap e potare i 183 nodi `source: "link"`, a partire dagli scrape
      Airbnb che contengono i concorrenti

### Fase 2 — far funzionare l'impianto (nostra, via API)
- [ ] Riabilitare i tool di escalation su Ospiti e Compliance
- [ ] Diagnosticare e riparare `Escalations`, `Early and Late Escalation`, `Schedule Maintenance`
- [ ] Riattivare `Cleaners reminder before arrival`
- [ ] Riaccendere il sync: PMS giornaliero, Notion settimanale
- [ ] Valutare se spegnere `web_search` / `web_answer` sull'agente ospite: contraddicono il
      guardrail "solo quello che c'è nelle informazioni del listing"

### Fase 3 — numeri e custom tool (serve materiale da noi + operatore Conduit)
- [ ] **Numeri sardi**: sostituire il vocale `+1 415 449 9931` con un numero italiano, impostare
      `escalation_enabled` e un `fallback_phone_number` reale, correggere l'orario Lun-Sab
- [ ] Verificare in dashboard che `whatsapp:+390707054279` sia vivo e verificato
- [ ] Attivare `Missed Calls`
- [ ] **Registrare i custom tool** — è lavoro da operatore Conduit, non API: i tool si creano in
      dashboard e solo dopo si agganciano alle skill via `allowed_tools`

### Fase 4 — knowledge base (serve Usai)
- [ ] Verificare se Usai ha il **Notion pronto**; se sì, collegarlo alla **mattone platform** e
      da lì alimentare la KB
- [ ] Completare le **credenziali WiFi per listing** — oggi non esistono
- [ ] Sistemare **KrossBooking** e riportare il sync a giornaliero
- [ ] **Testare la propagazione con Booking collegato e con Airbnb collegato**, separatamente:
      va verificato che i dati arrivino in entrambe le configurazioni e non solo in una
- [ ] Unire le skill duplicate, togliere le skill guest-facing dagli agenti interni, rinominare
      i residui `(1)` e gli `Untitled`, cancellare i 7 import duplicati
