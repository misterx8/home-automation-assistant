# Allarme Core — Analisi Tecnica Dettagliata

> Ultimo aggiornamento: 2026-04-09
> Aggiornare questo file dopo ogni modifica al progetto.

---

## 0. PRINCIPIO ARCHITETTURALE — REGOLA OBBLIGATORIA

**Ogni file dichiara le entità che usa.**

Le entità HA (`input_boolean`, `input_number`, `input_text`, `input_datetime`, `input_select`, `template`, `script`, ecc.)
devono essere definite nel file che le utilizza, non centralizzate in `allarme_core.yaml` salvo che appartengano
concettualmente al core del sistema (stato macchina, profilo, script arm/disarm).

| File | Dichiara le entità di... |
|------|--------------------------|
| `allarme_core.yaml` | Stato macchina, profili, zone, scripts arm/disarm/panico, sensori aggregati globali |
| `allarme_core_sensori.yaml` | input_boolean/input_text per ogni sensore wrapper |
| `allarme_core_log.yaml` | input_text logbook, input_number clip video, script di log, template timeline |
| `allarme_core_test.yaml` | input_boolean/input_number/input_datetime modalità test |
| `allarme_core_batterie.yaml` | input_number soglia/debounce, template sensor batterie |
| `allarme_core_automazioni.yaml` | Solo automazioni — nessuna entità propria |
| `allarme_core_supporto.yaml` | binary_sensor zone_valida |
| `allarme_core_anomalie_sensori.yaml` | binary_sensor anomalia_rilevata |

**Esempi corretti:**
- `input_number.allarme_core_clip_secondi_prima/dopo` → in `allarme_core_log.yaml` (usati dall'automazione snapshot lì definita)
- `input_number.allarme_core_test_timeout_minuti` → in `allarme_core_test.yaml` (usato solo dalla modalità test)
- `input_number.allarme_core_soglia_batteria_bassa` → in `allarme_core_batterie.yaml` (usato solo dai template batterie)

**Prima di aggiungere una nuova entità, chiedersi: quale file la usa? Quella è dove va dichiarata.**

---

## 1. STRUTTURA FILE DEL PROGETTO

```
allarme-core/
├── allarme_core.yaml                   # Core: stato macchina, scripts, sensori aggregati
├── allarme_core_sensori.yaml           # Definizione ~46 wrapper sensori fisici
├── allarme_core_supporto.yaml          # binary_sensor *_zone_valida (validazione zone)
├── allarme_core_automazioni.yaml       # Automazioni principali (arming, triggered, esclusioni)
├── allarme_core_log.yaml               # Sistema logging: script, automazioni timeline, sensori render
├── allarme_core_test.yaml              # Modalità test con auto-scadenza
├── allarme_core_batterie.yaml          # Monitoraggio batterie/connettività wireless
├── allarme_core_batterie_automazioni.yaml  # Automazioni batterie (alert, notifiche)
├── allarme_core_anomalie_sensori.yaml  # binary_sensor aggregato anomalie fisiche
├── allarme_core_anomalie_automazioni.yaml  # Automazioni anomalie sensori
├── plancia_operativa.yaml              # Dashboard Lovelace operativa
├── plancia_sensori.yaml                # Dashboard Lovelace gestione sensori
├── var/
│   └── allarme_core.yaml               # Definizione variabili persistenti (var component)
└── nodered/
    └── mosaico_immagini.json           # Flow Node-RED (composizione immagini)
```

---

## 2. MACCHINA A STATI

```
disarmed ──[script.arm_allarme_core]──► arming ──[delay configurabile]──► armed
   ▲                                                                          │
   └──────────────[script.disarm_allarme_core]──────────────────────────────┘
                                                                              │
armed ──[binary_sensor.allarme_core_sensori_aperti_filtrati = on]──► triggered
   ▲                                                                          │
   └──────────────[script.disarm_allarme_core / auto-reset]──────────────────┘
```

**Stati input_select.allarme_core_stato:** `disarmed` | `arming` | `armed` | `triggered`

**alarm_control_panel.allarme_core** (platform: template):
- Mappa `armed` → `armed_away` per compatibilità HA
- arm_away → `script.arm_allarme_core`
- disarm → `script.disarm_allarme_core`

---

## 3. PROFILI E ZONE

**Profili (input_select.allarme_core_profilo):** `nessuno` | `sera` | `giorno` | `notte` | `tutti`

**Mappatura profilo → zone attive** (sensor.allarme_core_zone_attive):
```
nessuno → []
sera    → ['sera']
giorno  → ['giorno']
notte   → ['notte']
tutti   → ['perimetrali', 'volumetrici']
```

**Zone disponibili (input_select.allarme_core_zone_disponibili):**
`perimetrali` | `volumetrici` | `giorno` | `sera` | `notte`

---

## 4. SENSORI WRAPPER — INVENTARIO COMPLETO

Ogni sensore è un `binary_sensor` template in `allarme_core_sensori.yaml`.
Convenzione entity_id: `binary_sensor.allarme_core_<nome>`

### Attributi standard di ogni wrapper

| Attributo           | Tipo          | Fonte                                    |
|---------------------|---------------|------------------------------------------|
| `abilitato`         | bool          | input_boolean.allarme_core_<nome>_abilitato |
| `zone`              | lista str     | input_text.allarme_core_<nome>_zone (split ',') |
| `camera`            | lista str     | input_text.allarme_core_<nome>_camera (split ',') |
| `tipo_connessione`  | 'wireless'\|'filare' | hardcoded nel template             |
| `tipo`              | str           | 'volumetrico'\|'perimetrale'\|ecc.      |
| `device_class`      | str           | 'motion'\|'door'\|'vibration'           |
| `sensore_origine`   | entity_id     | sensore fisico singolo                  |
| `batteria_origine`  | entity_id     | entità batteria (sensor. o binary_sensor.) |
| `disponibilita_origine` | entity_id | entità disponibilità wireless         |
| `sensori_config`    | JSON lista    | per wrapper multi-fisico (percorso A)   |
| `sensori_origine`   | lista         | per wrapper multi-fisico legacy (percorso B) |

### Sensori per region (46 totali)

| Region                  | Sensori                                                                    | Tot |
|-------------------------|---------------------------------------------------------------------------|-----|
| **taverna**             | porta_ingresso, movimento, movimento_corridoio, barriera_tapparella, finestre, vibrazione_ingresso | 6 |
| **cucina**              | movimento, finestra, vibrazione                                            | 3   |
| **bagno**               | movimento, finestra, vibrazione                                            | 3   |
| **lavanderia**          | movimento, finestra, vibrazione                                            | 3   |
| **ufficio**             | movimento, finestra                                                        | 2   |
| **camera**              | movimento, finestra                                                        | 2   |
| **doccia**              | movimento, anticamera_doccia, finestra                                     | 3   |
| **studio**              | movimento, finestra                                                        | 2   |
| **sala**                | porta_ingresso, movimento, finestra                                        | 3   |
| **tecnologico p1**      | porta_ingresso, movimento, finestra, armadio                              | 4   |
| **tecnologico pt**      | porta_ingresso, movimento, finestra, armadio                              | 4   |
| **solaio**              | movimento                                                                  | 1   |
| **esterno**             | movimento_box, movimento_giardino, movimento_dietro, movimento_davanti, pir_bidoni_esterni | 5 |
| **locale attrezzi**     | porta_ingresso, movimento                                                  | 2   |
| **locali tecnici esterni** | porta_caldaia_esterna, porta_sottoscala, porta_gas                    | 3   |
| **TOTALE**              |                                                                            | **46** |

---

## 5. SENSORI DI SISTEMA (aggregati, non wrapper fisici)

### In `allarme_core.yaml`

| Entity                                        | Tipo          | Scopo                                                 |
|-----------------------------------------------|---------------|-------------------------------------------------------|
| `sensor.allarme_core_zone_attive`             | template sensor | Lista zone attive in base al profilo              |
| `sensor.allarme_core_count_zona_sera/giorno/perimetrali/volumetrici/notte` | template sensor | Conteggio sensori attivi per zona |
| `sensor.allarme_core_sensori_esclusi_memoria` | template sensor | Wrapper su input_text esclusi (con attributo lista) |
| `sensor.allarme_core_sensori_attivi`          | template sensor | Nomi sensori attivi (attributo elenco)            |
| `sensor.allarme_core_sensori_aperti_filtrati` | template sensor | Sensori aperti attivi filtrando esclusi e zona — usato da Node-RED |
| `sensor.allarme_core_arming_secondi_rimanenti`| trigger sensor  | Countdown arming (aggiornato ogni secondo)        |
| `binary_sensor.allarme_core_sensori_aperti_su_profilo_attivo` | template | Aperto se ci sono sensori attivi sulle zone del profilo corrente |
| `binary_sensor.allarme_core_dati_pronti`      | template        | Gate sincronizzazione notifica: on quando sensori causa allarme + media Frigate pronti (o timeout) |
| `binary_sensor.allarme_core_zona_sera/giorno/perimetrali/volumetrici/notte` | template | Stato aggregato per zona |

### In `allarme_core_log.yaml`

| Entity                                        | Tipo          | Scopo                                                 |
|-----------------------------------------------|---------------|-------------------------------------------------------|
| `sensor.allarme_core_timeline_render`         | template sensor | Timeline con emoji colore da var.allarme_core_timeline_md |
| `sensor.allarme_core_count_critici/operazioni/note` | template | Conteggio eventi per categoria                   |
| `sensor.allarme_core_ultimo_trigger`          | template sensor | Ultimo sensore che ha triggerato l'allarme        |

### In `allarme_core_test.yaml`

| Entity                                        | Tipo          | Scopo                                                 |
|-----------------------------------------------|---------------|-------------------------------------------------------|
| `sensor.allarme_core_test_minuti_rimanenti`   | template sensor | Countdown timeout modalità test                   |

### In `allarme_core_batterie.yaml`

| Entity                                        | Tipo          | Scopo                                                 |
|-----------------------------------------------|---------------|-------------------------------------------------------|
| `sensor.allarme_core_batteria_minima`         | template sensor | Batteria minima tra tutti i wireless (con dettaglio/sensori_bassi) |
| (sensori per-wrapper)                         | template sensor | Batteria e connettività per ogni wrapper wireless |

### In `allarme_core_anomalie_sensori.yaml`

| Entity                                        | Tipo          | Scopo                                                 |
|-----------------------------------------------|---------------|-------------------------------------------------------|
| `binary_sensor.allarme_core_anomalia_rilevata`| template        | On se almeno un sensore fisico in stato anomalo (unknown/unavailable) |

---

## 6. VARIABILI PERSISTENTI (var component)

File: `var/allarme_core.yaml`

| Var entity                                       | Contenuto                                                    |
|--------------------------------------------------|--------------------------------------------------------------|
| `var.allarme_core_sensori_esclusi_var`           | Lista entity_id sensori aperti al momento dell'arming (esclusi da triggered) |
| `var.allarme_core_sensori_attivazione_allarme_var` | Stringa comma-separated entity_id causa allarme + attr profilo |
| `var.allarme_core_timeline_md`                   | Testo markdown timeline completa (long_state)                |
| `var.allarme_core_timeline_critici_md`           | Solo eventi critici 🚨 (long_state, max 25 righe)            |
| `var.allarme_core_timeline_operazioni_md`        | Operazioni normali (long_state, max 25 righe)                |
| `var.allarme_core_timeline_note_md`              | Note manuali 📝 (long_state, max 25 righe)                   |
| `var.allarme_core_ultimo_allarme_per_sensore`    | JSON {entity_id: timestamp_str} ultimo allarme per sensore   |
| `var.allarme_core_last_snapshot_urls`            | URL snapshot Frigate (long_state, una per riga)              |
| `var.allarme_core_last_clip_urls`                | URL clip Frigate (long_state, una per riga)                  |

---

## 7. SCRIPTS PRINCIPALI

| Script                                        | File            | Scopo                                                    |
|-----------------------------------------------|-----------------|----------------------------------------------------------|
| `script.arm_allarme_core`                     | allarme_core.yaml | arming → delay → armed (mode: restart)                |
| `script.disarm_allarme_core`                  | allarme_core.yaml | Stoppa arm script → disarmed                           |
| `script.allarme_core_attiva_panico`           | allarme_core.yaml | Forza triggered + logga PANICO (indipendente da stato) |
| `script.allarme_core_logbook_emit`            | allarme_core_log.yaml | Scrive in logbook HA + aggiorna input_text ultimo evento |
| `script.allarme_core_logbook_nota`            | allarme_core_log.yaml | Nota manuale via campo msg                            |
| `script.allarme_core_logbook_nota_da_ui`      | allarme_core_log.yaml | Nota da campo UI (input_text.allarme_core_nota_manuale) |
| `script.allarme_core_logbook_cancella_tutto`  | allarme_core_log.yaml | Reset tutte le var timeline                           |
| `script.allarme_core_snapshot_cameras`        | allarme_core_log.yaml | Salva snapshot da lista camera entity_id              |
| `script.allarme_core_attiva_test_mode`        | allarme_core_test.yaml | Attiva flag test + registra timestamp                |
| `script.allarme_core_disattiva_test_mode`     | allarme_core_test.yaml | Disattiva flag test + logga                          |

---

## 8. AUTOMAZIONI PRINCIPALI

### `allarme_core_automazioni.yaml`

| Alias                                              | Trigger                        | Azione principale                                          |
|----------------------------------------------------|--------------------------------|------------------------------------------------------------|
| Registra timestamp inizio arming                   | stato → arming                 | Set input_datetime.allarme_core_arming_started             |
| Registra timestamp inizio triggered                | stato → triggered              | Set input_datetime.allarme_core_triggered_started          |
| Reset sensori esclusi al disarm                    | stato → disarmed               | Clear input_text.allarme_core_sensori_esclusi              |
| Memorizza sensori esclusi all'arming               | stato → armed                  | Snapshot sensori aperti → input_text                       |
| Memorizza sensori esclusi con Var                  | stato → armed                  | Snapshot sensori aperti → var (doppio per disponibilità)   |
| Reset sensori esclusi al disarm con Var            | stato → disarmed               | Clear var.allarme_core_sensori_esclusi_var                 |
| Memorizza sensori che provocano allarme            | stato → triggered              | FIX-C: lettura diretta binary_sensor → var_attivazione     |
| Reset panico al disarm                             | stato → disarmed               | Turn off input_boolean.allarme_core_panico_attivo          |
| Reset guardia avviso auto-reset                    | stato → triggered              | Turn off avviso_loggato flag                               |
| Log avviso pre auto-reset triggered                | time_pattern /5s               | Log avviso 30sec prima del reset                           |
| Auto-reset da triggered                            | time_pattern /5s               | Disarma dopo N minuti se auto_reset abilitato              |
| Auto-disattiva modalità test al timeout            | template + homeassistant start | Disattiva test mode (resiliente al riavvio HA)             |

### `allarme_core_log.yaml`

| Alias                                              | Trigger                              | Azione principale                                          |
|----------------------------------------------------|--------------------------------------|------------------------------------------------------------|
| Logbook Sensori Esclusi                            | state var.sensori_esclusi_var        | Log aggiornamento lista esclusi                            |
| Logbook Cambio Stato                               | state input_select.allarme_core_stato | Log stato (escluso triggered — ha log dedicato)          |
| Logbook Cambio Profilo                             | state input_select.allarme_core_profilo | Log cambio profilo                                      |
| Logbook Allarme Scattato                           | stato → triggered                    | Log con sensori aperti e esclusi                           |
| Snapshot e clips su Triggered                      | stato → triggered                    | Attende var, calcola camere da sensori aperti, snapshot + var URL |
| UI Timeline Append                                 | state input_text.ultimo_evento       | Append riga a var.timeline_md                              |
| UI Timeline Sezioni (Livello 4)                    | state input_text.ultimo_evento       | Append a critici/operazioni/note in base a emoji           |

---

## 9. INTEGRAZIONE FRIGATE

- `input_text.allarme_core_url_base_ha` — URL esterno Home Assistant (es. https://mia-casa.duckdns.org)
- `input_text.allarme_core_url_base_frigate` — URL esterno Frigate (es. http://192.168.x.x:5000)
- `input_text.allarme_core_frigate_token` — Token autenticazione Frigate
- `input_number.allarme_core_clip_secondi_prima` — Secondi di clip prima del trigger (5-120, default 30) — definito in `allarme_core_log.yaml`
- `input_number.allarme_core_clip_secondi_dopo` — Secondi di clip dopo il trigger (0-60, default 10) — definito in `allarme_core_log.yaml`

**Flusso su triggered:**
1. Automazione aspetta che `var.allarme_core_sensori_attivazione_allarme_var` venga aggiornata (wait_for_trigger 5s)
2. Estrae entity_id sensori causa allarme dalla var
3. Per ogni sensore legge attributo `camera` (lista entity_id camera)
4. Chiama `script.allarme_core_snapshot_cameras` per salvare JPG in `/config/www/allarme_core_snapshots/`
5. Salva URL snapshot e clip in `var.allarme_core_last_snapshot_urls` e `var.allarme_core_last_clip_urls`
6. `binary_sensor.allarme_core_dati_pronti` diventa on quando tutto è pronto (o scatta timeout configurabile)
7. Node-RED si triggera su fronte off→on di `allarme_core_dati_pronti` per inviare notifiche

---

## 10. SISTEMA BATTERIE

**3 percorsi di risoluzione batteria per wrapper:**
- **Percorso A** — `sensori_config` (JSON lista): itera ogni sotto-sensore individualmente
- **Percorso B** — `sensori_origine` (lista flat, legacy): calcola minimo del gruppo
- **Percorso C** — `sensore_origine` (stringa singola): singolo sensore

**Configurazione:**
- `input_number.allarme_core_soglia_batteria_bassa` — soglia % (5-50)
- `input_number.allarme_core_debounce_offline_minuti` — debounce per offline (1-30 min)

**Output:**
- `sensor.allarme_core_batteria_minima` — valore minimo globale con attributi dettaglio e sensori_bassi

---

## 11. SISTEMA ANOMALIE

`binary_sensor.allarme_core_anomalia_rilevata` itera dinamicamente su tutti i `binary_sensor.allarme_core_*` escludendo i sensori aggregati (`_attivo`, `_filtrati`, `_zone_valida`, `_aperto`, `_anomalia`).

Per ogni wrapper legge:
- `sensore_origine` → verifica stato non in ['on','off']
- `sensori_origine` → itera lista, stessa verifica

Attributi: `sensori_anomali` (lista dict {fisico, wrapper, stato}), `contatore`

---

## 12. MODALITÀ TEST

- `input_boolean.allarme_core_modalita_test` — flag letto da Node-RED
- `input_number.allarme_core_test_timeout_minuti` — timeout auto-disattivazione (5-120 min)
- `input_datetime.allarme_core_test_attivato_alle` — timestamp attivazione
- Automazione usa `platform: template` + `platform: homeassistant event: start` (resiliente al riavvio)

---

## 13. SISTEMA PANICO

- `input_boolean.allarme_core_panico_attivo` — flag canale separato letto da Node-RED
- `script.allarme_core_attiva_panico` — attiva flag panico, registra timestamp triggered, forza stato triggered, logga
- Auto-reset del flag al disarm tramite automazione dedicata

---

## 14. VALIDAZIONE ZONE (allarme_core_supporto.yaml)

Per ogni sensore esiste un `binary_sensor.allarme_core_<nome>_zone_valida` che verifica:
- La stringa in `input_text.allarme_core_<nome>_zone` non è vuota
- Ogni zona (split per virgola) esiste in `input_select.allarme_core_zone_disponibili`

---

## 15. DASHBOARD — PLANCIA OPERATIVA (`plancia_operativa.yaml`)

**Tipo:** `type: sections` — layout a colonne (max 4), path: `configurazione-allarme`

**Dipendenze custom card:**
- `card_mod` — stili CSS dinamici con Jinja2
- `custom:timeline-card` — timeline visuale stato allarme
- `browser_mod` — popup modal per dettagli e impostazioni

**Header (sempre visibile):**
- Alert `ha-alert warning` se modalità test attiva (con minuti rimanenti)
- Alert `ha-alert error` se `binary_sensor.allarme_core_batteria_critica` è on (soglia 5%)
- Barra progress arming: visibile solo se `alarm_control_panel.allarme_core` è in arming → mostra secondi rimanenti + barra ASCII con emoji colore (🟢/🟡/🔴)

**Sezione 1 — Controllo Allarme:**

| Card | Entity/contenuto | Note |
|------|-----------------|------|
| tile | `sensor.allarme_core_zone_attive` | Mostra stato + attributo zones |
| tile | `input_select.allarme_core_profilo` | Con feature select-options |
| tile | `alarm_control_panel.allarme_core` | Con feature alarm-modes |
| tile | `input_select.allarme_core_stato` | Visualizzazione stato raw |
| markdown | sensori esclusi | Lista da `sensor.allarme_core_sensori_esclusi_memoria`.sensori |
| button | **PANICO** | Rosso (#c0392b), tap_action con confirmation, chiama `script.allarme_core_attiva_panico` |
| tile x5 | count zona sera/giorno/perimetrali/volumetrici/notte | border rosso se zona on, verde se off |
| tile | `sensor.allarme_core_sensori_attivi` | Border rosso se sensori attivi, verde se nessuno |
| tile | `sensor.allarme_core_ultimo_trigger` | tap_action popup con dettagli (data, profilo, sensori, snapshot/clip links) |
| tile | `input_boolean.allarme_core_modalita_test` | Border arancio se on |
| tile | `sensor.allarme_core_test_minuti_rimanenti` | Countdown test |
| tile | `input_number.allarme_core_test_timeout_minuti` | Slider |
| tile x2 | script attiva/disattiva test | Pulsanti azione |
| tile | `input_boolean.allarme_core_auto_reset_triggered` | Border arancio se on |
| tile | `input_number.allarme_core_auto_reset_minuti` | Slider |
| button | Impostazioni Variabili | Popup con URL HA/Frigate/token/delay/timeout/clip_prima/clip_dopo — border verde/arancio/rosso in base a compilazione campi |
| button | Debug Variabili | Popup con var esclusi, var attivazione, dati_pronti, triggered_started |

**Sezione 2 — Log:**

| Card | Contenuto |
|------|-----------|
| tile | `script.allarme_core_logbook_cancella_tutto` |
| tile | `script.allarme_core_logbook_nota_da_ui` → popup con `input_text.allarme_core_nota_manuale` |
| `custom:timeline-card` | Visualizza `input_select.allarme_core_stato` ultime 24h (con state_map e icon_color_map) |
| markdown | `sensor.allarme_core_timeline_render`.long_state — timeline completa con colori |
| markdown | Registro unificato: Ultimo evento + sezioni Critici 🚨 / Operazioni ⚙️ / Note 📝 da var |
| markdown | Ultimo evento semplice |
| logbook | Filtra su `input_text.allarme_core_logbook_event` — ultime 24h |

**Sezione 3 — Immagini:**

| Card | Contenuto |
|------|-----------|
| markdown | Snapshot: legge `var.allarme_core_last_snapshot_urls`.long_state, mostra max 4 immagini come `[![](url)](url)` |
| markdown | Clip: legge `var.allarme_core_last_clip_urls`.long_state, mostra max 4 link apertura |

**Sezione 4 — Batterie & Connettività:**

| Card | Entity/contenuto |
|------|-----------------|
| gauge | `sensor.allarme_core_batteria_minima` (0-100, soglie green 30 / yellow 20 / red 0) |
| entity | `sensor.allarme_core_batterie_basse` |
| entity | `sensor.allarme_core_sensori_offline` |
| markdown | Tabella batterie basse da `.lista` |
| markdown | Tabella sensori offline da `.lista_offline` |
| markdown | Tabella anomalie hardware da `binary_sensor.allarme_core_anomalia_rilevata`.sensori_anomali |
| entities | Soglia batteria + debounce offline |

---

## 16. DASHBOARD — PLANCIA SENSORI (`plancia_sensori.yaml`)

**Tipo:** `type: sections` — path: `allarme-mappa-sensori`, title: `Mappa Sensori`

**Organizzazione:** Sensori raggruppati per piano e stanza con heading title/subtitle.

**Pattern card per ogni sensore (tile + card_mod avanzato):**

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
    # Logica CSS dinamica:
    # - Legge enabled (input_boolean), zone, camera, zone_valida, sensore_origine/sensori_origine
    # - Border color: rosso=disabilitato | arancio=fisico anomalo | verde=ok
    # - Badge ::after: ❌=zona invalida (zone_valida=off) | 🔌=fisico anomalo | ⚠️=zona/camera vuota
```

**Logica badge per ogni card sensore:**
- `❌` — `binary_sensor.*_zone_valida` è off (zona configurata non valida)
- `🔌` — sensore fisico (`sensore_origine` o `sensori_origine`) in stato not in ['on','off'] AND enabled
- `⚠️` — zona o camera non configurata (campo vuoto/unknown)
- Nessun badge — tutto configurato correttamente

**Struttura grouping plancia sensori:**
- Primo Piano: Sala, Cucina, Bagno, Lavanderia, Ufficio, Camera, Doccia, Studio, Tecp1
- Piano Terra: Taverna, Sala/ingresso PT, Tecnologico PT
- Esterno/Accessori: Esterno, Locale Attrezzi, Locali Tecnici, Solaio

**Dipendenze:** `card_mod`, `browser_mod`

---

## 17. ENTITÀ BATTERIE EXTRA (da plancia_operativa)

Entità rilevate nell'uso della plancia ma definite in `allarme_core_batterie.yaml`:

| Entity | Tipo | Scopo |
|--------|------|-------|
| `sensor.allarme_core_batterie_basse` | template sensor | Count sensori sotto soglia — attr `lista` [{nome, batteria}] |
| `sensor.allarme_core_sensori_offline` | template sensor | Count sensori offline — attr `lista_offline` [{nome, fisico}] |
| `binary_sensor.allarme_core_batteria_critica` | template | On se almeno un sensore ha batteria < 5% (soglia fissa critica, diversa dalla soglia configurabile) |

---

## 18. BUG CORRETTI (storico)

| ID       | Descrizione                                                         | File                    |
|----------|---------------------------------------------------------------------|-------------------------|
| AC-BUG-01 | entity_id in `data:` deprecato → spostato in `target:`            | allarme_core.yaml, automazioni |
| AC-BUG-04 | Sensori template indentati a 4 spazi invece di 6 → non caricati   | allarme_core_log.yaml   |
| AC-BUG-05 | input_datetime.allarme_core_arming_started non dichiarato → sensore aperti filtrati unavailable | allarme_core.yaml |
| AC-BUG-06 | script arm continuava in background dopo disarm → si armava da solo | allarme_core.yaml      |
| FIX-B    | Sensori aperti filtrati ignorava stato "triggered"                  | allarme_core.yaml       |
| FIX-C    | Race condition lettura causa allarme (template invece di lettura diretta binary_sensor) | allarme_core_automazioni.yaml |

---

## 19. REGOLE PER AGGIUNTA NUOVO SENSORE

Vedi procedura dettagliata in `.claude/CLAUDE.md` sezione "Procedura: Aggiunta Nuovo Sensore in Allarme Core".

**Riepilogo file da modificare:**
1. `allarme_core_sensori.yaml` — 4 inserimenti (input_boolean, input_text zone, input_text camera, template binary_sensor)
2. `allarme_core_supporto.yaml` — 1 inserimento (binary_sensor zone_valida)
3. `plancia_sensori.yaml` — 1 inserimento (heading subtitle + tile card)

**File che si aggiornano automaticamente:**
- `allarme_core_batterie.yaml` — template itera su tutti i wrapper
- `allarme_core_anomalie_sensori.yaml` — template itera su tutti i wrapper

---

## 20. DIPENDENZE ESTERNE

- **var component** — necessario per tutte le variabili persistenti
- **Frigate** — opzionale, per snapshot/clip su triggered
- **Node-RED** — per notifiche e azioni fisiche (sirena, relay)
- `/config/www/allarme_core_snapshots/` — directory snapshot (deve esistere)
- **card_mod** (custom card HACS) — stili CSS dinamici in entrambe le plance
- **browser_mod** (custom card HACS) — popup modal nelle plance
- **custom:timeline-card** (custom card HACS) — timeline visuale in plancia operativa

---

*Aggiornare questo file dopo ogni modifica strutturale al progetto.*
