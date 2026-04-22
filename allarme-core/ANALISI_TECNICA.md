# Allarme Core — Analisi Tecnica

> Ultimo aggiornamento: 2026-04-22
> Aggiornare dopo ogni modifica strutturale.

---

## 0. PRINCIPIO ARCHITETTURALE — REGOLA OBBLIGATORIA

**Ogni file dichiara le entità che usa.**

| File | Entità di... |
|------|--------------|
| `allarme_core.yaml` | Stato macchina, profili, zone, scripts arm/disarm/panico, sensori aggregati globali |
| `allarme_core_sensori.yaml` | input_boolean/input_text per ogni sensore wrapper |
| `allarme_core_log.yaml` | input_text logbook, input_number clip video, script log, template timeline |
| `allarme_core_test.yaml` | input_boolean/input_number/input_datetime modalità test + auto-reset triggered |
| `allarme_core_batterie.yaml` | input_number soglia/debounce, template sensor batterie |
| `allarme_core_automazioni.yaml` | Solo automazioni — nessuna entità propria |
| `allarme_core_supporto.yaml` | binary_sensor zone_valida |
| `allarme_core_anomalie_sensori.yaml` | binary_sensor anomalia_rilevata |

**Esempi corretti:**
- `input_number.allarme_core_clip_secondi_prima/dopo` → `allarme_core_log.yaml`
- `input_number.allarme_core_test_timeout_minuti` → `allarme_core_test.yaml`
- `input_number.allarme_core_soglia_batteria_bassa` → `allarme_core_batterie.yaml`

---

## 1. STRUTTURA FILE

```
allarme-core/
├── allarme_core.yaml                   # Core: stato macchina, scripts, sensori aggregati
├── allarme_core_sensori.yaml           # 46 wrapper sensori fisici
├── allarme_core_supporto.yaml          # binary_sensor *_zone_valida
├── allarme_core_automazioni.yaml       # Automazioni principali
├── allarme_core_log.yaml               # Log: script, automazioni timeline, sensori render
├── allarme_core_test.yaml              # Modalità test con auto-scadenza
├── allarme_core_batterie.yaml          # Monitoraggio batterie/connettività wireless
├── allarme_core_batterie_automazioni.yaml
├── allarme_core_anomalie_sensori.yaml  # binary_sensor aggregato anomalie fisiche
├── allarme_core_anomalie_automazioni.yaml
├── plancia_operativa.yaml
├── plancia_sensori.yaml
├── var/allarme_core.yaml               # Variabili persistenti (var component)
└── nodered/mosaico_immagini.json
```

---

## 2. MACCHINA A STATI

```
disarmed ──[script.arm_allarme_core]──► arming ──[delay]──► armed
   ▲                                                             │
   └──────────────[script.disarm_allarme_core]──────────────────┘
                                                                 │
armed ──[binary_sensor.allarme_core_sensori_aperti_filtrati=on]──► triggered
   ▲                                                             │
   └──────────────[script.disarm_allarme_core / auto-reset]──────┘
```

**Stati `input_select.allarme_core_stato`:** `disarmed` | `arming` | `armed` | `triggered`

**`alarm_control_panel.allarme_core`** (platform: template): mappa `armed` → `armed_away`; arm_away → `script.arm_allarme_core`; disarm → `script.disarm_allarme_core`

**Entità core (tutte in `allarme_core.yaml`):**

| Entità | Tipo | Scopo |
|--------|------|-------|
| `input_select.allarme_core_stato` | input_select | Stato macchina principale |
| `input_select.allarme_core_profilo` | input_select | Profilo attivo |
| `input_number.allarme_core_arming_delay` | input_number | Exit delay (5-120s, step 5); sync ring-keypad |
| `input_number.allarme_core_notifica_timeout_secondi` | input_number | Timeout attesa notifica post-trigger (10-120s) |
| `input_datetime.allarme_core_arming_started` | input_datetime | Timestamp inizio arming (fix AC-BUG-05) |

---

## 3. PROFILI E ZONE

**Profili:** `nessuno` | `sera` | `giorno` | `notte` | `tutti`

**Mappatura profilo → zone attive** (`sensor.allarme_core_zone_attive`):
```
nessuno → []
sera    → ['sera']
giorno  → ['giorno']
notte   → ['notte']
tutti   → ['perimetrali', 'volumetrici']
```

**Zone disponibili:** `perimetrali` | `volumetrici` | `giorno` | `sera` | `notte`

---

## 4. SENSORI WRAPPER — INVENTARIO (46 totali)

Ogni wrapper = `binary_sensor` template in `allarme_core_sensori.yaml`.
Entity_id convention: `binary_sensor.allarme_core_<nome>`

**Attributi standard:**

| Attributo | Tipo | Fonte |
|-----------|------|-------|
| `abilitato` | bool | input_boolean.allarme_core_<nome>_abilitato |
| `zone` | lista str | input_text.allarme_core_<nome>_zone (split ',') |
| `camera` | lista str | input_text.allarme_core_<nome>_camera (split ',') |
| `tipo_connessione` | 'wireless'\|'filare' | hardcoded |
| `tipo` | str | 'volumetrico'\|'perimetrale'\|ecc. |
| `device_class` | str | 'motion'\|'door'\|'vibration' |
| `sensore_origine` | entity_id | sensore fisico singolo |
| `batteria_origine` | entity_id | sensor. o binary_sensor. |
| `disponibilita_origine` | entity_id | disponibilità wireless |
| `sensori_config` | JSON lista | wrapper multi-fisico (percorso A) |
| `sensori_origine` | lista | multi-fisico legacy (percorso B) |

**Inventario per region:**

| Region | Sensori | Tot |
|--------|---------|-----|
| **taverna** | porta_ingresso, movimento, movimento_corridoio, barriera_tapparella, finestre, vibrazione_ingresso | 6 |
| **cucina** | movimento, finestra, vibrazione | 3 |
| **bagno** | movimento, finestra, vibrazione | 3 |
| **lavanderia** | movimento, finestra, vibrazione | 3 |
| **ufficio** | movimento, finestra | 2 |
| **camera** | movimento, finestra | 2 |
| **doccia** | movimento, anticamera_doccia, finestra | 3 |
| **studio** | movimento, finestra | 2 |
| **sala** | porta_ingresso, movimento, finestra | 3 |
| **tecnologico p1** | porta_ingresso, movimento, finestra, armadio | 4 |
| **tecnologico pt** | porta_ingresso, movimento, finestra, armadio | 4 |
| **solaio** | movimento | 1 |
| **esterno** | movimento_box, movimento_giardino, movimento_dietro, movimento_davanti, pir_bidoni_esterni | 5 |
| **locale attrezzi** | porta_ingresso, movimento | 2 |
| **locali tecnici esterni** | porta_caldaia_esterna, porta_sottoscala, porta_gas | 3 |
| **TOTALE** | | **46** |

---

## 5. SENSORI DI SISTEMA (aggregati)

### In `allarme_core.yaml`

| Entity | Tipo | Scopo |
|--------|------|-------|
| `sensor.allarme_core_zone_attive` | template sensor | Zone attive per profilo |
| `sensor.allarme_core_count_zona_sera/giorno/perimetrali/volumetrici/notte` | template sensor | Conteggio sensori attivi per zona |
| `sensor.allarme_core_sensori_esclusi_memoria` | template sensor | Wrapper su input_text esclusi |
| `sensor.allarme_core_sensori_attivi` | template sensor | Nomi sensori attivi |
| `sensor.allarme_core_sensori_aperti_filtrati` | template sensor | Sensori aperti attivi filtrando esclusi e zona — usato da Node-RED |
| `sensor.allarme_core_arming_secondi_rimanenti` | trigger sensor | Countdown arming (ogni secondo) |
| `binary_sensor.allarme_core_sensori_aperti_su_profilo_attivo` | template | On se sensori attivi sulle zone del profilo corrente |
| `binary_sensor.allarme_core_dati_pronti` | template | Gate sincronizzazione: on quando Frigate + media pronti (o timeout) |
| `binary_sensor.allarme_core_zona_sera/giorno/perimetrali/volumetrici/notte` | template | Stato aggregato per zona |

### In `allarme_core_log.yaml`

| Entity | Scopo |
|--------|-------|
| `sensor.allarme_core_timeline_render` | Timeline con emoji da var.allarme_core_timeline_md |
| `sensor.allarme_core_count_critici/operazioni/note` | Conteggio eventi per categoria |
| `sensor.allarme_core_ultimo_trigger` | Ultimo sensore che ha triggerato |

### In `allarme_core_test.yaml`

| Entity | Scopo |
|--------|-------|
| `sensor.allarme_core_test_minuti_rimanenti` | Countdown timeout modalità test |

### In `allarme_core_batterie.yaml`

| Entity | Scopo |
|--------|-------|
| `sensor.allarme_core_batteria_minima` | Batteria minima globale (con dettaglio/sensori_bassi) |
| (sensori per-wrapper) | Batteria e connettività per ogni wrapper wireless |

### In `allarme_core_anomalie_sensori.yaml`

| Entity | Scopo |
|--------|-------|
| `binary_sensor.allarme_core_anomalia_rilevata` | On se almeno un fisico in stato unknown/unavailable |

---

## 6. VARIABILI PERSISTENTI (var component)

File: `var/allarme_core.yaml`

| Var | Contenuto |
|-----|-----------|
| `var.allarme_core_sensori_esclusi_var` | Lista entity_id sensori aperti all'arming (esclusi da triggered) |
| `var.allarme_core_sensori_attivazione_allarme_var` | Comma-separated entity_id causa allarme + attr profilo |
| `var.allarme_core_timeline_md` | Timeline completa (long_state) |
| `var.allarme_core_timeline_critici_md` | Solo eventi critici 🚨 (max 25 righe) |
| `var.allarme_core_timeline_operazioni_md` | Operazioni (max 25 righe) |
| `var.allarme_core_timeline_note_md` | Note manuali 📝 (max 25 righe) |
| `var.allarme_core_ultimo_allarme_per_sensore` | JSON {entity_id: timestamp_str} |
| `var.allarme_core_last_snapshot_urls` | URL snapshot Frigate (long_state, una per riga) |
| `var.allarme_core_last_clip_urls` | URL clip Frigate (long_state, una per riga) |

---

## 7. SCRIPTS

| Script | File | Scopo |
|--------|------|-------|
| `script.arm_allarme_core` | allarme_core.yaml | arming → delay → armed (mode: restart) |
| `script.disarm_allarme_core` | allarme_core.yaml | Stoppa arm → disarmed |
| `script.allarme_core_attiva_panico` | allarme_core.yaml | Forza triggered + logga PANICO |
| `script.allarme_core_logbook_emit` | allarme_core_log.yaml | Scrive logbook HA + update input_text |
| `script.allarme_core_logbook_nota` | allarme_core_log.yaml | Nota manuale via campo msg |
| `script.allarme_core_logbook_nota_da_ui` | allarme_core_log.yaml | Nota da input_text.allarme_core_nota_manuale |
| `script.allarme_core_logbook_cancella_tutto` | allarme_core_log.yaml | Reset tutte le var timeline |
| `script.allarme_core_snapshot_cameras` | allarme_core_log.yaml | Snapshot da lista camera entity_id |
| `script.allarme_core_attiva_test_mode` | allarme_core_test.yaml | Attiva flag + timestamp |
| `script.allarme_core_disattiva_test_mode` | allarme_core_test.yaml | Disattiva + logga |

---

## 8. AUTOMAZIONI

### `allarme_core_automazioni.yaml`

| Alias | Trigger | Azione |
|-------|---------|--------|
| Registra timestamp inizio arming | stato → arming | Set input_datetime.allarme_core_arming_started |
| Registra timestamp inizio triggered | stato → triggered | Set input_datetime.allarme_core_triggered_started |
| Reset sensori esclusi al disarm | stato → disarmed | Clear input_text esclusi |
| Memorizza sensori esclusi all'arming | stato → armed | Snapshot sensori aperti → input_text |
| Memorizza sensori esclusi con Var | stato → armed | Snapshot → var |
| Reset sensori esclusi al disarm con Var | stato → disarmed | Clear var.allarme_core_sensori_esclusi_var |
| Memorizza sensori che provocano allarme | stato → triggered | FIX-C: lettura diretta binary_sensor → var_attivazione |
| Reset panico al disarm | stato → disarmed | Turn off input_boolean.allarme_core_panico_attivo |
| Reset guardia avviso auto-reset | stato → triggered | Turn off avviso_loggato flag |
| Log avviso pre auto-reset triggered | time_pattern /5s | Log avviso 30sec prima del reset |
| Auto-reset da triggered | time_pattern /5s | Disarma dopo N minuti se auto_reset abilitato |
| Auto-disattiva modalità test al timeout | template + HA start | Disattiva test mode (resiliente al riavvio) |

### `allarme_core_log.yaml`

| Alias | Trigger | Azione |
|-------|---------|--------|
| Logbook Sensori Esclusi | state var.sensori_esclusi_var | Log aggiornamento lista esclusi |
| Logbook Cambio Stato | state allarme_core_stato | Log stato (escluso triggered) |
| Logbook Cambio Profilo | state allarme_core_profilo | Log cambio profilo |
| Logbook Allarme Scattato | stato → triggered | Log con sensori aperti e esclusi |
| Snapshot e clips su Triggered | stato → triggered | Aspetta var, calcola camere, snapshot + URL |
| UI Timeline Append | state input_text.ultimo_evento | Append riga a var.timeline_md |
| UI Timeline Sezioni (Livello 4) | state input_text.ultimo_evento | Append a critici/operazioni/note per emoji |

---

## 9. INTEGRAZIONE FRIGATE

- `input_text.allarme_core_url_base_ha` — URL esterno HA
- `input_text.allarme_core_url_base_frigate` — URL Frigate
- `input_text.allarme_core_frigate_token` — Token auth
- `input_number.allarme_core_clip_secondi_prima` — clip pre-trigger (5-120, def 30) — `allarme_core_log.yaml`
- `input_number.allarme_core_clip_secondi_dopo` — clip post-trigger (0-60, def 10) — `allarme_core_log.yaml`

**Flusso su triggered:**
1. Aspetta `var.allarme_core_sensori_attivazione_allarme_var` (wait 5s)
2. Estrae entity_id sensori causa allarme
3. Legge attributo `camera` per ogni sensore
4. Chiama `script.allarme_core_snapshot_cameras` → `/config/www/allarme_core_snapshots/`
5. Salva URL in `var.allarme_core_last_snapshot_urls` e `var.allarme_core_last_clip_urls`
6. `binary_sensor.allarme_core_dati_pronti` → on quando tutto pronto (o timeout)
7. Node-RED triggera su off→on di `allarme_core_dati_pronti` per notifiche

---

## 10. SISTEMA BATTERIE

**3 percorsi di risoluzione:**
- **A** — `sensori_config` (JSON lista): itera ogni sotto-sensore
- **B** — `sensori_origine` (lista flat, legacy): calcola minimo gruppo
- **C** — `sensore_origine` (stringa singola): singolo sensore

**Config:**
- `input_number.allarme_core_soglia_batteria_bassa` — soglia % (5-50)
- `input_number.allarme_core_debounce_offline_minuti` — debounce offline (1-30 min)

**Output:** `sensor.allarme_core_batteria_minima` — valore minimo globale + dettaglio + sensori_bassi

---

## 11. SISTEMA ANOMALIE

`binary_sensor.allarme_core_anomalia_rilevata` itera su tutti `binary_sensor.allarme_core_*` escludendo aggregati (`_attivo`, `_filtrati`, `_zone_valida`, `_aperto`, `_anomalia`).

Per ogni wrapper legge `sensore_origine` / `sensori_origine` → verifica stato not in ['on','off'].

Attributi: `sensori_anomali` [{fisico, wrapper, stato}], `contatore`

---

## 12. MODALITÀ TEST (tutte in `allarme_core_test.yaml`)

- `input_boolean.allarme_core_modalita_test` — flag letto da Node-RED
- `input_number.allarme_core_test_timeout_minuti` — timeout auto-disattivazione (5-120 min)
- `input_datetime.allarme_core_test_attivato_alle` — timestamp attivazione
- Automazione: `platform: template` + `platform: homeassistant event: start` (resiliente al riavvio)

**Auto-reset da triggered:**
- `input_boolean.allarme_core_auto_reset_triggered` — abilita disarmo automatico dopo N min in triggered
- `input_number.allarme_core_auto_reset_minuti` — minuti attesa
- `input_boolean.allarme_core_auto_reset_avviso_loggato` — flag interno anti-doppio-log (reset ad ogni triggered)

---

## 13. SISTEMA PANICO

- `input_boolean.allarme_core_panico_attivo` — flag canale separato letto da Node-RED
- `script.allarme_core_attiva_panico` — attiva flag, registra triggered, logga PANICO
- Auto-reset flag al disarm tramite automazione dedicata

---

## 14. VALIDAZIONE ZONE (`allarme_core_supporto.yaml`)

Per ogni sensore: `binary_sensor.allarme_core_<nome>_zone_valida`
- Stringa in input_text non vuota
- Ogni zona (split virgola) esiste in `input_select.allarme_core_zone_disponibili`

---

## 15. DASHBOARD — PLANCIA OPERATIVA (`plancia_operativa.yaml`)

`type: sections`, path: `configurazione-allarme`, max 4 colonne.

**Dipendenze:** `card_mod`, `custom:timeline-card`, `browser_mod`

**Header (sempre visibile):**
- Alert warning se modalità test attiva (+ minuti rimanenti)
- Alert error se `binary_sensor.allarme_core_batteria_critica` on (soglia 5%)
- Barra progress arming: visibile se alarm_control_panel in arming → secondi + barra ASCII emoji (🟢/🟡/🔴)

**Sezione 1 — Controllo Allarme:**

| Card | Entity/contenuto |
|------|-----------------|
| tile | `sensor.allarme_core_zone_attive` |
| tile | `input_select.allarme_core_profilo` + feature select-options |
| tile | `alarm_control_panel.allarme_core` + feature alarm-modes |
| tile | `input_select.allarme_core_stato` |
| markdown | sensori esclusi da `sensor.allarme_core_sensori_esclusi_memoria`.sensori |
| button | **PANICO** (rosso #c0392b, confirmation, → `script.allarme_core_attiva_panico`) |
| tile x5 | count zona sera/giorno/perimetrali/volumetrici/notte |
| tile | `sensor.allarme_core_sensori_attivi` |
| tile | `sensor.allarme_core_ultimo_trigger` + tap popup dettagli |
| tile | `input_boolean.allarme_core_modalita_test` |
| tile | `sensor.allarme_core_test_minuti_rimanenti` |
| tile | `input_number.allarme_core_test_timeout_minuti` |
| tile x2 | script attiva/disattiva test |
| tile | `input_boolean.allarme_core_auto_reset_triggered` |
| tile | `input_number.allarme_core_auto_reset_minuti` |
| button | Impostazioni Variabili (popup URL/Frigate/token/delay/timeout/clip) |
| button | Debug Variabili (popup var esclusi, attivazione, dati_pronti, triggered_started) |

**Sezione 2 — Log:**

| Card | Contenuto |
|------|-----------|
| tile | `script.allarme_core_logbook_cancella_tutto` |
| tile | `script.allarme_core_logbook_nota_da_ui` → popup `input_text.allarme_core_nota_manuale` |
| `custom:timeline-card` | `input_select.allarme_core_stato` ultime 24h |
| markdown | `sensor.allarme_core_timeline_render`.long_state |
| markdown | Registro unificato: Critici 🚨 / Operazioni ⚙️ / Note 📝 da var |
| logbook | Filtra su `input_text.allarme_core_logbook_event` |

**Sezione 3 — Immagini:**

| Card | Contenuto |
|------|-----------|
| markdown | Snapshot: da `var.allarme_core_last_snapshot_urls` max 4 come `[![](url)](url)` |
| markdown | Clip: da `var.allarme_core_last_clip_urls` max 4 link |

**Sezione 4 — Batterie & Connettività:**

| Card | Entity/contenuto |
|------|-----------------|
| gauge | `sensor.allarme_core_batteria_minima` (0-100, green 30/yellow 20/red 0) |
| entity | `sensor.allarme_core_batterie_basse` |
| entity | `sensor.allarme_core_sensori_offline` |
| markdown | Tabella batterie basse da `.lista` |
| markdown | Tabella sensori offline da `.lista_offline` |
| markdown | Tabella anomalie da `binary_sensor.allarme_core_anomalia_rilevata`.sensori_anomali |
| entities | Soglia batteria + debounce offline |

---

## 16. DASHBOARD — PLANCIA SENSORI (`plancia_sensori.yaml`)

`type: sections`, path: `allarme-mappa-sensori`.

**Pattern card per ogni sensore:**
```yaml
type: tile
entity: binary_sensor.allarme_core_<nome>
tap_action:
  action: call-service
  service: browser_mod.popup   # popup con input_boolean/zone/camera
hold_action:
  action: more-info
card_mod:
  style: |
    # Border: rosso=disabilitato | arancio=fisico anomalo | verde=ok
    # Badge ::after: ❌=zona invalida | 🔌=fisico anomalo | ⚠️=zona/camera vuota
```

**Struttura grouping:**
- Primo Piano: Sala, Cucina, Bagno, Lavanderia, Ufficio, Camera, Doccia, Studio, Tecp1
- Piano Terra: Taverna, Sala/ingresso PT, Tecnologico PT
- Esterno/Accessori: Esterno, Locale Attrezzi, Locali Tecnici, Solaio

**Dipendenze:** `card_mod`, `browser_mod`

---

## 17. ENTITÀ BATTERIE EXTRA (in `allarme_core_batterie.yaml`)

| Entity | Tipo | Scopo |
|--------|------|-------|
| `sensor.allarme_core_batterie_basse` | template sensor | Count sotto soglia — attr `lista` [{nome, batteria}] |
| `sensor.allarme_core_sensori_offline` | template sensor | Count offline — attr `lista_offline` [{nome, fisico}] |
| `binary_sensor.allarme_core_batteria_critica` | template | On se batteria < 5% (soglia fissa, diversa da configurabile) |

---

## 18. BUG CORRETTI

| ID | Descrizione | File |
|----|-------------|------|
| AC-BUG-01 | entity_id in `data:` deprecato → spostato in `target:` | allarme_core.yaml, automazioni |
| AC-BUG-04 | Template indentati a 4 spazi invece di 6 → non caricati | allarme_core_log.yaml |
| AC-BUG-05 | input_datetime.allarme_core_arming_started non dichiarato → sensori unavailable | allarme_core.yaml |
| AC-BUG-06 | script arm continuava background dopo disarm → si armava da solo | allarme_core.yaml |
| FIX-B | Sensori aperti filtrati ignorava stato "triggered" | allarme_core.yaml |
| FIX-C | Race condition lettura causa allarme (template invece di lettura diretta binary_sensor) | allarme_core_automazioni.yaml |
| FIX-D | `esclusi` usava stringa grezza invece di lista parsed — `not in` faceva substring match. Corretto con `replace("'",'"') + from_json` | allarme_core.yaml |
| FIX-E | Automazione "Memorizza sensori causa allarme" scriveva var="" anche con lista vuota. Aggiunta `condition: template aperti|length > 0` | allarme_core_automazioni.yaml |

---

## 19. PROCEDURA AGGIUNTA NUOVO SENSORE

File da modificare (ordine):

1. **`allarme_core_sensori.yaml`** — 4 inserimenti nella `#region` corretta:
   - `input_boolean.allarme_core_<nome>_abilitato`
   - `input_text.allarme_core_<nome>_zone`
   - `input_text.allarme_core_<nome>_camera`
   - template `binary_sensor.allarme_core_<nome>`

2. **`allarme_core_supporto.yaml`** — 1 inserimento `binary_sensor.*_zone_valida`

3. **`plancia_sensori.yaml`** — 1 inserimento (heading subtitle + tile card con card_mod)

**File che si aggiornano automaticamente:** `allarme_core_batterie.yaml`, `allarme_core_anomalie_sensori.yaml`

Dettaglio completo in `.claude/CLAUDE.md` sezione "Procedura: Aggiunta Nuovo Sensore".

---

## 20. DIPENDENZE ESTERNE

- **var component** (HACS) — tutte le variabili persistenti
- **Frigate** — opzionale, snapshot/clip su triggered
- **Node-RED** — notifiche e azioni fisiche (sirena, relay)
- `/config/www/allarme_core_snapshots/` — deve esistere
- **card_mod** (HACS)
- **browser_mod** (HACS)
- **custom:timeline-card** (HACS)
