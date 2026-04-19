# Safety Core — Analisi Tecnica Dettagliata

> Ultimo aggiornamento: 2026-04-13
> Aggiornare questo file dopo ogni modifica al progetto.

---

## 1. STRUTTURA FILE DEL PROGETTO

```
safety-core/
├── safety_core.yaml                      # Core: stato macchina, script reset, input_select/boolean/text
├── safety_core_sensori.yaml              # Definizione 19 wrapper sensori fisici + 5 aggregati per categoria
├── safety_core_automazioni.yaml          # Automazioni: triggered, log, snapshot
├── safety_core_log.yaml                  # Sistema logging: script, template sensor timeline
├── safety_core_batterie.yaml             # Monitoraggio batterie wireless (percorso singolo)
├── safety_core_batterie_automazioni.yaml # Automazioni alert batterie
├── safety_core_anomalie_sensori.yaml     # binary_sensor aggregato anomalie fisiche
├── safety_core_anomalie_automazioni.yaml # Automazioni anomalie sensori
├── plancia_controllo.yaml                # Dashboard Lovelace unica
└── var/
    └── safety_core.yaml                  # Definizione variabili persistenti (var component)
```

**Principio chiave:** Sistema completamente indipendente da `allarme_core`. Gestisce esclusivamente pericoli ambientali (fumo, gas, monossido, acqua). Stessa architettura base di allarme_core, ma molto più semplice.

---

## 2. MACCHINA A STATI

```
ok ──[binary_sensor.safety_core_qualsiasi_attivo = on, for 3s]──► triggered
 ▲                                                                      │
 └──────────────[script.safety_core_reset (SOLO MANUALE)]──────────────┘
```

**Stati `input_select.safety_core_stato`:** `ok` | `triggered`

**Differenza fondamentale vs allarme_core:**
- Nessun arming/disarming — il sistema è sempre attivo
- Nessun profilo/zone — tutti i sensori abilitati rispondono sempre
- Reset **solo manuale** tramite `script.safety_core_reset` — NON esiste auto-reset
- Finestra di grazia post-reset: 10 secondi (blocca ri-trigger immediato)

**Trigger con debounce:** `for: 00:00:03` sull'automazione triggered per evitare falsi allarmi da glitch.

---

## 3. CATEGORIE E MASTER SWITCH

Il sistema è organizzato in 4 categorie, ognuna con un master switch che disabilita tutti i sensori della categoria:

| Categoria | Master Switch                                    | Device Class   |
|-----------|--------------------------------------------------|----------------|
| fumo      | `input_boolean.safety_core_categoria_fumo_abilitato`     | smoke          |
| gas       | `input_boolean.safety_core_categoria_gas_abilitato`      | gas            |
| carbonio  | `input_boolean.safety_core_categoria_carbonio_abilitato` | carbon_monoxide|
| acqua     | `input_boolean.safety_core_categoria_acqua_abilitato`    | moisture       |

**Logica di attivazione wrapper (tripla condizione AND):**
```
stato wrapper = ON se:
  1. input_boolean.safety_core_categoria_<cat>_abilitato = on
  2. input_boolean.safety_core_<nome>_abilitato = on
  3. sensore fisico sottostante = on
```

Se il sensore fisico è in stato unknown/unavailable → wrapper = `off` (non propaga anomalie come allarmi).

---

## 4. SENSORI WRAPPER — INVENTARIO COMPLETO

Ogni sensore è un `binary_sensor` template in `safety_core_sensori.yaml`.
Convenzione entity_id: `binary_sensor.safety_core_<categoria>_<luogo>`

### Attributi standard di ogni wrapper

| Attributo             | Tipo          | Fonte                                         |
|-----------------------|---------------|-----------------------------------------------|
| `sensore_origine`     | entity_id     | sensore fisico singolo                        |
| `tipo`                | str           | 'fumo'\|'gas'\|'carbonio'\|'acqua'            |
| `priorita`            | str           | 'alta' (tutti)                                |
| `camera`              | lista str     | input_text.safety_core_<nome>_camera (split ',') |
| `abilitato`           | bool          | input_boolean.safety_core_<nome>_abilitato    |
| `categoria_abilitata` | bool          | input_boolean.safety_core_categoria_<cat>_abilitato |
| `tipo_connessione`    | 'wireless'\|'filare' | hardcoded nel template               |
| `batteria_origine`    | entity_id\|"" | entità batteria (solo wireless, altrimenti "") |
| `disponibilita_origine` | entity_id\|"" | entità disponibilità (solo wireless, altrimenti "") |

**Nota:** Non c'è attributo `zone` — safety_core non usa il sistema zone/profili di allarme_core.

### Sensori per categoria (19 totali)

#### Fumo (6) — device_class: smoke

| Entity                                      | Sensore Fisico                           | Connessione |
|---------------------------------------------|------------------------------------------|-------------|
| `binary_sensor.safety_core_fumo_studio`     | binary_sensor.sensore_fumo_studio        | wireless    |
| `binary_sensor.safety_core_fumo_ufficio`    | binary_sensor.sensore_fumo_ufficio       | wireless    |
| `binary_sensor.safety_core_fumo_tecnologico_p1` | binary_sensor.sensore_fumo_tecnologico_p1 | wireless |
| `binary_sensor.safety_core_fumo_cucina`     | binary_sensor.sensore_fumo_cucina        | wireless    |
| `binary_sensor.safety_core_fumo_solaio`     | binary_sensor.sensore_fumo_solaio        | **filare**  |
| `binary_sensor.safety_core_fumo_tecnologico_pt` | binary_sensor.sensore_fumo_tecnologico_pt | **filare** |

#### Gas (3) — device_class: gas

| Entity                                      | Sensore Fisico                           | Connessione |
|---------------------------------------------|------------------------------------------|-------------|
| `binary_sensor.safety_core_gas_cucina`      | binary_sensor.gas_cucina_vd4             | **filare**  |
| `binary_sensor.safety_core_gas_caldaia`     | binary_sensor.gas_caldaia_vd4            | **filare**  |
| `binary_sensor.safety_core_gas_studio`      | binary_sensor.sensore_gas_studio         | **filare**  |

#### Monossido di Carbonio (1) — device_class: carbon_monoxide

| Entity                                      | Sensore Fisico                           | Connessione |
|---------------------------------------------|------------------------------------------|-------------|
| `binary_sensor.safety_core_carbonio_taverna`| binary_sensor.sensore_carbonio_taverna   | **filare**  |

#### Acqua (9) — device_class: moisture

| Entity                                           | Sensore Fisico                              | Connessione |
|--------------------------------------------------|---------------------------------------------|-------------|
| `binary_sensor.safety_core_acqua_vasca`          | binary_sensor.sensore_acqua_vasca           | wireless    |
| `binary_sensor.safety_core_acqua_bagno`          | binary_sensor.sensore_acqua_bagno           | wireless    |
| `binary_sensor.safety_core_acqua_cucina_principale` | binary_sensor.sensore_acqua_cucina_principale | wireless |
| `binary_sensor.safety_core_acqua_doccia`         | binary_sensor.sensore_acqua_doccia          | wireless    |
| `binary_sensor.safety_core_acqua_lavanderia`     | binary_sensor.sensore_acqua_lavanderia      | wireless    |
| `binary_sensor.safety_core_acqua_lavello_cucina` | binary_sensor.sensore_acqua_lavello_cucina  | wireless    |
| `binary_sensor.safety_core_acqua_studio`         | binary_sensor.sensore_acqua_studio          | wireless    |
| `binary_sensor.safety_core_acqua_stufa`          | binary_sensor.sensore_acqua_stufa           | wireless    |
| `binary_sensor.safety_core_acqua_tecnologico_pt` | binary_sensor.sensore_acqua_tecnologico_pt  | wireless    |

---

## 5. SENSORI AGGREGATI PER CATEGORIA

Definiti in `safety_core_sensori.yaml`, iterano dinamicamente su `binary_sensor.safety_core_<cat>_*`:

| Entity                                         | Logica                                    |
|------------------------------------------------|-------------------------------------------|
| `binary_sensor.safety_core_fumo_attivo`        | OR su tutti binary_sensor.safety_core_fumo_* (escluso _attivo) |
| `binary_sensor.safety_core_gas_attivo`         | OR su tutti binary_sensor.safety_core_gas_* |
| `binary_sensor.safety_core_carbonio_attivo`    | OR su tutti binary_sensor.safety_core_carbonio_* |
| `binary_sensor.safety_core_acqua_attivo`       | OR su tutti binary_sensor.safety_core_acqua_* |
| `binary_sensor.safety_core_qualsiasi_attivo`   | OR su tutti e 4 i _attivo sopra — **trigger principale** |

Ogni aggregato ha attributo `sensori_attivi` con lista entity_id dei sensori on.

---

## 6. ENTITÀ DI STATO E CONFIGURAZIONE

### Da `safety_core.yaml`

| Entity                                      | Tipo          | Scopo                                                     |
|---------------------------------------------|---------------|-----------------------------------------------------------|
| `input_select.safety_core_stato`            | input_select  | Stato sistema: ok\|triggered                              |
| `input_select.safety_core_ultima_categoria` | input_select  | Categoria che ha scatenato il triggered (nessuna\|fumo\|gas\|carbonio\|acqua) |
| `input_boolean.safety_core_reset_in_corso`  | input_boolean | Finestra di grazia post-reset (10 sec) — blocca ri-trigger |
| `input_text.safety_core_ultimo_evento`      | input_text    | Testo ultimo evento (mirror per coerenza)                 |

### Da `safety_core_log.yaml`

| Entity                                      | Tipo          | Scopo                                        |
|---------------------------------------------|---------------|----------------------------------------------|
| `input_text.safety_core_logbook_event`      | input_text    | Entity fittizia per filtrare Logbook card    |
| `input_text.safety_core_logbook_ultimo_evento` | input_text | Ultimo messaggio log (trigger timeline)      |
| `input_text.safety_core_nota_manuale`       | input_text    | Campo UI per note manuali                    |
| `input_text.safety_core_url_base_ha`        | input_text    | URL base HA (es. http://homeassistant.local:8123) — costruisce link cliccabili agli snapshot |
| `input_text.safety_core_url_base_frigate`   | input_text    | URL Frigate (es. http://192.168.2.92:5000) — usata per generare clip (sostituisce il valore hardcoded) |
| `input_text.safety_core_frigate_token`      | input_text    | Token JWT Frigate — se configurato aggiunge `?token=<valore>` alle URL clip |
| `input_number.safety_core_clip_secondi_prima` | input_number | Secondi prima del triggered nella clip Frigate (5-120, default 30) |
| `input_number.safety_core_clip_secondi_dopo`  | input_number | Secondi dopo il triggered nella clip Frigate (5-60, default 10) |

---

## 7. SCRIPT

| Script                                      | File                   | Scopo                                                       |
|---------------------------------------------|------------------------|-------------------------------------------------------------|
| `script.safety_core_reset`                  | safety_core.yaml       | Reset manuale: ok + categoria nessuna + delay 10s grazia   |
| `script.safety_core_logbook_emit`           | safety_core_log.yaml   | Scrive in logbook HA + aggiorna input_text ultimo evento    |
| `script.safety_core_logbook_nota`           | safety_core_log.yaml   | Nota manuale via campo msg                                  |
| `script.safety_core_logbook_nota_da_ui`     | safety_core_log.yaml   | Nota da campo UI + svuota campo dopo salvataggio            |
| `script.safety_core_logbook_cancella_tutto` | safety_core_log.yaml   | Reset tutte le var timeline                                 |
| `script.safety_core_snapshot_cameras`       | safety_core_log.yaml   | Salva snapshot in `/config/www/safety_core_snapshots/`     |

---

## 8. AUTOMAZIONI

### `safety_core_automazioni.yaml`

| Alias                                       | Trigger                                    | Condizioni                                        | Azione principale                              |
|---------------------------------------------|--------------------------------------------|---------------------------------------------------|------------------------------------------------|
| Triggered su sensore attivo                 | `safety_core_qualsiasi_attivo` → on, for 3s | Non già triggered; almeno 1 fisico realmente on; reset_in_corso = off | Imposta triggered + ultima categoria + salva var sensori |
| Log sensori tornati normali                 | `safety_core_qualsiasi_attivo` → off       | stato = triggered                                 | Log "tornati normali — reset rimane manuale"  |
| Logbook Cambio Stato                        | `safety_core_stato` cambia               | from ≠ to                                         | Log triggered (con categoria+sensori) o ok    |
| Snapshot Cameras su Triggered               | `safety_core_stato` → triggered           | cams_all.length > 0                               | Snapshot + var URL snapshot/clip              |
| UI Timeline Append                          | `safety_core_logbook_ultimo_evento` cambia | valore non vuoto e cambiato                      | Append a var timeline critici/operazioni/note  |

**Fix Bug 1 (automazione triggered):** Condizione che verifica che almeno 1 sensore fisico sia effettivamente in stato `on` (non solo il wrapper) — evita falsi allarmi post-riavvio da sensori in unknown/unavailable.

**Fix Bug 2 (automazione triggered):** Condizione `safety_core_reset_in_corso = off` — blocca ri-trigger nei 10 secondi dopo il reset.

**Fix Bug 3 (Timeline Append):** Usa `action variables` invece di `automation variables` per garantire che `ent` sia valorizzato prima del `var.set`.

### `safety_core_log.yaml` (automazioni implicite)

Le automazioni UI Timeline Append sono definite in `safety_core_automazioni.yaml` (non in safety_core_log.yaml come in allarme_core — il log file qui contiene solo script e template sensor).

---

## 9. VARIABILI PERSISTENTI (var component)

File: `var/safety_core.yaml`

| Var entity                                  | Contenuto                                                       |
|---------------------------------------------|-----------------------------------------------------------------|
| `var.safety_core_sensori_attivazione_var`   | Stringa comma-separated entity_id sensori che hanno causato triggered |
| `var.safety_core_timeline_md`               | Timeline completa (long_state)                                  |
| `var.safety_core_timeline_critici_md`       | Solo eventi critici 🚨 (long_state, max 25 righe)               |
| `var.safety_core_timeline_operazioni_md`    | Operazioni normali (long_state, max 25 righe)                   |
| `var.safety_core_timeline_note_md`          | Note manuali 📝 (long_state, max 25 righe)                      |
| `var.safety_core_last_snapshot_urls`        | URL snapshot (long_state, una per riga)                         |
| `var.safety_core_last_clip_urls`            | URL clip Frigate (long_state, una per riga)                     |

**Nota:** Non esiste `var.safety_core_sensori_esclusi_var` — safety_core non esclude mai sensori.

---

## 10. TEMPLATE SENSOR (safety_core_log.yaml)

| Entity                                      | Scopo                                                          |
|---------------------------------------------|----------------------------------------------------------------|
| `sensor.safety_core_sensori_attivazione_nomi` | Nomi leggibili dei sensori che hanno causato il triggered     |
| `sensor.safety_core_timeline_render`        | Timeline con emoji colore (🔴=critico, 🟢=ok, ⚫=nota, 🔵=altro) |
| `sensor.safety_core_count_critici`          | Conteggio eventi critici in var                                |
| `sensor.safety_core_count_operazioni`       | Conteggio operazioni in var                                    |
| `sensor.safety_core_count_note`             | Conteggio note in var                                          |

---

## 11. SISTEMA BATTERIE (`safety_core_batterie.yaml`)

**Versione semplificata rispetto ad allarme_core** — un solo percorso di risoluzione:
- **Percorso C unico** — `sensore_origine` (stringa singola): legge batteria da `batteria_origine` e disponibilità da `disponibilita_origine`

Tutti i wrapper hanno un solo sensore fisico (nessun multi-sensore in safety_core).

**Configurazione:**
- `input_number.safety_core_soglia_batteria_bassa` — soglia % (5-50, default 20)
- `input_number.safety_core_debounce_offline_minuti` — debounce offline (1-30 min, default 2)

**Output:**
- `sensor.safety_core_batteria_minima` — valore minimo globale con attributi dettaglio e sensori_bassi
- (entità per-wrapper, stessa struttura di allarme_core)

---

## 12. SISTEMA ANOMALIE (`safety_core_anomalie_sensori.yaml`)

`binary_sensor.safety_core_anomalia_rilevata` — identica logica ad allarme_core:
- Itera su `binary_sensor.safety_core_*` escludendo `_attivo` e `_anomalia`
- Legge `sensore_origine` e `sensori_origine` per trovare i fisici
- On se almeno un fisico in stato not in ['on','off']
- Attributi: `sensori_anomali` [{fisico, wrapper, stato}], `contatore`

---

## 13. INTEGRAZIONE FRIGATE

**Stesso pattern di allarme_core** ma cartella dedicata:
- Snapshot salvati in `/config/www/safety_core_snapshots/`
- URL base hardcoded in automazione: `http://192.168.2.92:5000`
- Clip generate con finestra -30s/+10s dal momento del triggered
- Var `safety_core_last_snapshot_urls` e `safety_core_last_clip_urls` per Node-RED/plancia

**Differenza:** safety_core **non ha** `binary_sensor.safety_core_dati_pronti` — non c'è il gate di sincronizzazione presente in allarme_core.

---

## 14. DIFFERENZE CHIAVE VS ALLARME_CORE

| Aspetto                    | safety_core              | allarme_core                          |
|----------------------------|--------------------------|---------------------------------------|
| Stati                      | ok / triggered           | disarmed / arming / armed / triggered |
| Reset                      | Solo manuale             | Manuale + auto-reset configurabile    |
| Profili/Zone               | **Nessuno**              | 5 profili, 5 zone                     |
| Esclusione sensori aperti  | **No**                   | Sì (all'arming)                       |
| Modalità test              | **No**                   | Sì (con auto-scadenza)                |
| Master switch              | Per categoria (4)        | Per singolo sensore                   |
| Sensori aggregati          | Sì (per categoria)       | Sì (per zona)                         |
| Finestra di grazia reset   | 10 sec (input_boolean)   | No                                    |
| Gate notifica (dati_pronti)| **No**                   | Sì                                    |
| Trigger debounce           | `for: 00:00:03`          | No (trigger immediato)                |
| Elettrovalvole             | Sì (gas + acqua)         | No                                    |
| File batterie              | Percorso C unico         | Percorsi A+B+C                        |

---

## 15. DASHBOARD — PLANCIA CONTROLLO (`plancia_controllo.yaml`)

**Tipo:** `type: sections` — path: `safety-core`, title: `Safety Core`

**Dipendenze custom card:** `card_mod`, `browser_mod`

**Sezione 1 — Controllo Safety:**

| Card | Entity/contenuto | Note |
|------|-----------------|------|
| tile | `input_select.safety_core_stato` | Border rosso=triggered, verde=ok |
| tile | `input_select.safety_core_ultima_categoria` | Categoria che ha triggerato |
| tile | `script.safety_core_reset` | Reset manuale, larghezza piena |
| tile x4 | binary_sensor fumo/gas/carbonio/acqua _attivo | Border rosso=on, verde=off |
| tile | `binary_sensor.safety_core_qualsiasi_attivo` | Aggregato globale |
| tile x4 | Master switch categorie fumo/gas/carbonio/acqua | — |
| tile x2 | `switch.eth_rly16_board1_relay_1` (GAS) + `switch.eth_rly16_board2_relay_3` (Acqua) | **Elettrovalvole fisiche** |

**Sezioni 2+ — Sensori per categoria:**

Una sezione per categoria (Gas, Monossido, Acqua) con una card tile per ogni sensore.

**Bottoni Configurazione & Debug (fine sezione 1):**

| Card | Contenuto popup | Note |
|------|----------------|------|
| button "Configurazione Avanzata" (9 cols) | vertical-stack: URL & Credenziali (url_base_ha, url_base_frigate) / Clip Frigate (clip_secondi_prima/dopo) / Monitoraggio Batterie (soglia, debounce) | card_mod: verde=entrambi URL configurati, arancio=parziale, rosso=nessuno + badge ⚠️ |
| button "Debug" (3 cols) | entities: var sensori attivazione, reset_in_corso, ultima_categoria, stato | opacità 0.5, bordo grigio |

**Sezione Log & Attività:**

| Card | Entity/contenuto | Note |
|------|-----------------|------|
| button | Registro (popup) | vertical-stack con 3 markdown scrollabili: Critici / Operazioni / Note da `var.safety_core_timeline_*_md` |
| tile | `script.safety_core_logbook_cancella_tutto` | Cancella Log, 4 cols, action: toggle |
| tile | `script.safety_core_logbook_nota_da_ui` | Aggiungi Nota, 4 cols, popup browser_mod |
| custom:timeline-card | `input_select.safety_core_stato` + `input_select.safety_core_ultima_categoria` | 480px, 24h, collapse_duplicates |
| tile | `input_text.safety_core_logbook_ultimo_evento` | Ultimo Evento, 12 cols |
| logbook | `input_text.safety_core_logbook_event` | 24h |

**Pattern card_mod per ogni sensore (differente da allarme_core):**

```
Border color:
  grigio  → sensore o categoria DISABILITATI (enabled==off OR cat==off)
  rosso   → sensore IN ALLARME (stato = on)
  arancio → fisico ANOMALO (sensore_origine not in ['on','off']) AND enabled
  verde   → tutto OK

Badge ::after:
  🔕 → disabilitato (sensore o categoria)
  ⚠️ → fisico anomalo
  (nessuno) → ok o in allarme
```

**tap_action su ogni sensore:** popup (browser_mod) con `input_boolean` abilitato + `input_text` camera.

**Elettrovalvole:**
- `switch.eth_rly16_board1_relay_1` — Elettrovalvola GAS
- `switch.eth_rly16_board2_relay_3` — Elettrovalvola Acqua

---

## 15b. INTEGRAZIONE CON RING KEYPAD

Safety-core è integrato con il Ring Keypad V2 tramite `ring-keypad/packages/ring_keypad_integration.yaml` (sezione 3).

**safety-core → tastiera:**

| Stato + Categoria | LED tastiera |
|-------------------|--------------|
| `triggered` + `fumo` / `gas` / `carbonio` | Alarming Smoke-Fire |
| `triggered` + `acqua` | Alarming Water Leak |
| `ok` (dopo reset manuale) | Ripristino LED allarme-core corrente |

**tastiera → safety-core (tasto Fire hold 3s):**

- Imposta `safety_core_stato = triggered` e `safety_core_ultima_categoria = fumo`
- Lo scatto è registrato nei log di safety-core come: `"🔥 Allarme fuoco manuale da tastiera Ring Keypad"`
- Distinguibile dallo scatto automatico dei sensori fisici solo tramite il log
- Anti-loop attivo: non invia se safety-core è già in triggered

---

## 16. BUG CORRETTI (storico)

| ID     | Descrizione                                                                   | File                        |
|--------|-------------------------------------------------------------------------------|-----------------------------|
| Bug 1  | Falsi allarmi post-riavvio da sensori in unknown/unavailable — aggiunta condizione controllo fisico realmente on | safety_core_automazioni.yaml |
| Bug 2  | Ri-trigger immediato dopo reset — aggiunta finestra di grazia 10s con input_boolean reset_in_corso | safety_core_automazioni.yaml + safety_core.yaml |
| Bug 3  | Timeline append: ent non valorizzato con automation variables → spostato in action variables | safety_core_automazioni.yaml |
| Bug 4  | `initial: 20` e `initial: 2` su input_number soglia batteria e debounce offline → reset al valore di default ad ogni riavvio HA, sovrascrivendo le impostazioni utente. Rimossi entrambi i campi `initial`. | safety_core_batterie.yaml |

---

## 17. REGOLE PER AGGIUNTA NUOVO SENSORE

### File da modificare (nell'ordine):

**1. `safety_core_sensori.yaml` — 3 inserimenti:**
- `input_boolean.safety_core_<cat>_<nome>_abilitato` nella sezione input_boolean della categoria
- `input_text.safety_core_<cat>_<nome>_camera` nella sezione input_text della categoria
- Template `binary_sensor` con: sensore_origine, tipo, priorita, camera, abilitato, categoria_abilitata, tipo_connessione, batteria_origine, disponibilita_origine

**2. `plancia_controllo.yaml` — 1 inserimento:**
- Tile card nella sezione della categoria con tap_action popup e card_mod completo (border grigio/rosso/arancio/verde + badge 🔕/⚠️)

### File che si aggiornano automaticamente:
- `safety_core_batterie.yaml` — itera su tutti i wrapper
- `safety_core_anomalie_sensori.yaml` — itera su tutti i wrapper
- I sensori aggregati `safety_core_<cat>_attivo` — iterano su tutti i binary_sensor della categoria

---

## 18. DIPENDENZE ESTERNE

- **var component** (HACS) — necessario per tutte le variabili persistenti
- **Frigate** — opzionale, per snapshot/clip su triggered
- **Node-RED** — per notifiche e azionamento elettrovalvole
- `/config/www/safety_core_snapshots/` — directory snapshot (deve esistere)
- **card_mod** (HACS) — stili CSS dinamici nella plancia
- **browser_mod** (HACS) — popup modal nella plancia
- **switch.eth_rly16_board1_relay_1** e **switch.eth_rly16_board2_relay_3** — elettrovalvole fisiche (ETH-RLY16)

---

*Aggiornare questo file dopo ogni modifica strutturale al progetto.*
