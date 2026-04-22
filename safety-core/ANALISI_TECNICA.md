# Safety Core — Analisi Tecnica

> Ultimo aggiornamento: 2026-04-22
> Aggiornare dopo ogni modifica strutturale.

---

## 1. STRUTTURA FILE

```
safety-core/
├── safety_core.yaml                      # Core: stato macchina, script reset, input_select/boolean/text
├── safety_core_sensori.yaml              # 19 wrapper fisici + 5 aggregati per categoria
├── safety_core_automazioni.yaml          # Automazioni: triggered, log, snapshot
├── safety_core_log.yaml                  # Log: script, template sensor timeline
├── safety_core_batterie.yaml             # Monitoraggio batterie wireless
├── safety_core_batterie_automazioni.yaml # Automazioni alert batterie
├── safety_core_anomalie_sensori.yaml     # binary_sensor aggregato anomalie
├── safety_core_anomalie_automazioni.yaml
├── plancia_controllo.yaml
└── var/safety_core.yaml                  # Variabili persistenti (var component)
```

Sistema **completamente indipendente** da allarme_core. Gestisce pericoli ambientali (fumo, gas, monossido, acqua).

---

## 2. MACCHINA A STATI

```
ok ──[binary_sensor.safety_core_qualsiasi_attivo=on, for 3s]──► triggered
 ▲                                                                    │
 └──────────────[script.safety_core_reset (SOLO MANUALE)]────────────┘
```

**Stati `input_select.safety_core_stato`:** `ok` | `triggered`

**Differenze vs allarme_core:**
- Nessun arming/disarming — sempre attivo
- Nessun profilo/zone
- Reset **solo manuale**
- Finestra di grazia post-reset: 10 secondi (blocca ri-trigger)
- Trigger debounce: `for: 00:00:03`

---

## 3. CATEGORIE E MASTER SWITCH

4 categorie, ognuna con master switch:

| Categoria | Master Switch | Device Class |
|-----------|---------------|--------------|
| fumo | `input_boolean.safety_core_categoria_fumo_abilitato` | smoke |
| gas | `input_boolean.safety_core_categoria_gas_abilitato` | gas |
| carbonio | `input_boolean.safety_core_categoria_carbonio_abilitato` | carbon_monoxide |
| acqua | `input_boolean.safety_core_categoria_acqua_abilitato` | moisture |

**Logica attivazione wrapper (AND tripla):**
```
stato wrapper = ON se:
  1. input_boolean.safety_core_categoria_<cat>_abilitato = on
  2. input_boolean.safety_core_<nome>_abilitato = on
  3. sensore fisico = on
```
Fisico in unknown/unavailable → wrapper = `off`.

---

## 4. SENSORI WRAPPER — INVENTARIO (19 totali)

Entity_id convention: `binary_sensor.safety_core_<categoria>_<luogo>`

**Attributi standard:**

| Attributo | Fonte |
|-----------|-------|
| `sensore_origine` | sensore fisico singolo |
| `tipo` | 'fumo'\|'gas'\|'carbonio'\|'acqua' |
| `priorita` | 'alta' (tutti) |
| `camera` | input_text.safety_core_<nome>_camera |
| `abilitato` | input_boolean.safety_core_<nome>_abilitato |
| `categoria_abilitata` | input_boolean.safety_core_categoria_<cat>_abilitato |
| `tipo_connessione` | 'wireless'\|'filare' hardcoded |
| `batteria_origine` | entity_id\|"" |
| `disponibilita_origine` | entity_id\|"" |

**Nessun attributo `zone`** — safety_core non usa zone/profili.

#### Fumo (6) — device_class: smoke

| Entity | Sensore Fisico | Conn. |
|--------|---------------|-------|
| `binary_sensor.safety_core_fumo_studio` | binary_sensor.sensore_fumo_studio | wireless |
| `binary_sensor.safety_core_fumo_ufficio` | binary_sensor.sensore_fumo_ufficio | wireless |
| `binary_sensor.safety_core_fumo_tecnologico_p1` | binary_sensor.sensore_fumo_tecnologico_p1 | wireless |
| `binary_sensor.safety_core_fumo_cucina` | binary_sensor.sensore_fumo_cucina | wireless |
| `binary_sensor.safety_core_fumo_solaio` | binary_sensor.sensore_fumo_solaio | **filare** |
| `binary_sensor.safety_core_fumo_tecnologico_pt` | binary_sensor.sensore_fumo_tecnologico_pt | **filare** |

#### Gas (3) — device_class: gas

| Entity | Sensore Fisico | Conn. |
|--------|---------------|-------|
| `binary_sensor.safety_core_gas_cucina` | binary_sensor.gas_cucina_vd4 | **filare** |
| `binary_sensor.safety_core_gas_caldaia` | binary_sensor.gas_caldaia_vd4 | **filare** |
| `binary_sensor.safety_core_gas_studio` | binary_sensor.sensore_gas_studio | **filare** |

#### Monossido di Carbonio (1) — device_class: carbon_monoxide

| Entity | Sensore Fisico | Conn. |
|--------|---------------|-------|
| `binary_sensor.safety_core_carbonio_taverna` | binary_sensor.sensore_carbonio_taverna | **filare** |

#### Acqua (9) — device_class: moisture

| Entity | Sensore Fisico | Conn. |
|--------|---------------|-------|
| `binary_sensor.safety_core_acqua_vasca` | binary_sensor.sensore_acqua_vasca | wireless |
| `binary_sensor.safety_core_acqua_bagno` | binary_sensor.sensore_acqua_bagno | wireless |
| `binary_sensor.safety_core_acqua_cucina_principale` | binary_sensor.sensore_acqua_cucina_principale | wireless |
| `binary_sensor.safety_core_acqua_doccia` | binary_sensor.sensore_acqua_doccia | wireless |
| `binary_sensor.safety_core_acqua_lavanderia` | binary_sensor.sensore_acqua_lavanderia | wireless |
| `binary_sensor.safety_core_acqua_lavello_cucina` | binary_sensor.sensore_acqua_lavello_cucina | wireless |
| `binary_sensor.safety_core_acqua_studio` | binary_sensor.sensore_acqua_studio | wireless |
| `binary_sensor.safety_core_acqua_stufa` | binary_sensor.sensore_acqua_stufa | wireless |
| `binary_sensor.safety_core_acqua_tecnologico_pt` | binary_sensor.sensore_acqua_tecnologico_pt | wireless |

---

## 5. SENSORI AGGREGATI PER CATEGORIA

In `safety_core_sensori.yaml`, iterano su `binary_sensor.safety_core_<cat>_*`:

| Entity | Logica |
|--------|--------|
| `binary_sensor.safety_core_fumo_attivo` | OR su tutti safety_core_fumo_* |
| `binary_sensor.safety_core_gas_attivo` | OR su tutti safety_core_gas_* |
| `binary_sensor.safety_core_carbonio_attivo` | OR su tutti safety_core_carbonio_* |
| `binary_sensor.safety_core_acqua_attivo` | OR su tutti safety_core_acqua_* |
| `binary_sensor.safety_core_qualsiasi_attivo` | OR su tutti e 4 gli _attivo — **trigger principale** |

Ogni aggregato ha attributo `sensori_attivi` (lista entity_id on).

---

## 6. ENTITÀ STATO E CONFIGURAZIONE

### Da `safety_core.yaml`

| Entity | Tipo | Scopo |
|--------|------|-------|
| `input_select.safety_core_stato` | input_select | ok\|triggered |
| `input_select.safety_core_ultima_categoria` | input_select | Categoria che ha triggerato (nessuna\|fumo\|gas\|carbonio\|acqua) |
| `input_boolean.safety_core_reset_in_corso` | input_boolean | Finestra di grazia 10s post-reset |
| `input_text.safety_core_ultimo_evento` | input_text | Ultimo evento (mirror) |

### Da `safety_core_log.yaml`

| Entity | Tipo | Scopo |
|--------|------|-------|
| `input_text.safety_core_logbook_event` | input_text | Entità fittizia per Logbook card |
| `input_text.safety_core_logbook_ultimo_evento` | input_text | Trigger timeline |
| `input_text.safety_core_nota_manuale` | input_text | Note manuali da UI |
| `input_text.safety_core_url_base_ha` | input_text | URL HA — link snapshot |
| `input_text.safety_core_url_base_frigate` | input_text | URL Frigate — URL clip |
| `input_text.safety_core_frigate_token` | input_text | Token JWT Frigate |
| `input_number.safety_core_clip_secondi_prima` | input_number | Secondi pre-triggered (5-120, def 30) |
| `input_number.safety_core_clip_secondi_dopo` | input_number | Secondi post-triggered (5-60, def 10) |

---

## 7. SCRIPT

| Script | File | Scopo |
|--------|------|-------|
| `script.safety_core_reset` | safety_core.yaml | Reset manuale: ok + categoria nessuna + delay 10s grazia |
| `script.safety_core_logbook_emit` | safety_core_log.yaml | Logbook HA + update input_text |
| `script.safety_core_logbook_nota` | safety_core_log.yaml | Nota via campo msg |
| `script.safety_core_logbook_nota_da_ui` | safety_core_log.yaml | Nota da UI + svuota campo |
| `script.safety_core_logbook_cancella_tutto` | safety_core_log.yaml | Reset tutte le var timeline |
| `script.safety_core_snapshot_cameras` | safety_core_log.yaml | Snapshot in `/config/www/safety_core_snapshots/` |

---

## 8. AUTOMAZIONI

### `safety_core_automazioni.yaml`

| Alias | Trigger | Condizioni | Azione |
|-------|---------|------------|--------|
| Triggered su sensore attivo | `qualsiasi_attivo` → on, for 3s | Non già triggered; fisico realmente on; reset_in_corso=off | stato=triggered + ultima categoria + var sensori |
| Log sensori tornati normali | `qualsiasi_attivo` → off | stato=triggered | Log "tornati normali — reset rimane manuale" |
| Logbook Cambio Stato | `safety_core_stato` cambia | from≠to | Log triggered (cat+sensori) o ok |
| Snapshot Cameras su Triggered | stato → triggered | cams_all.length > 0 | Snapshot + var URL |
| UI Timeline Append | `logbook_ultimo_evento` cambia | valore non vuoto e cambiato | Append a var timeline |

**Fix Bug 1:** condizione verifica fisico realmente `on` — evita falsi allarmi post-riavvio.
**Fix Bug 2:** `reset_in_corso=off` — blocca ri-trigger nei 10s post-reset.
**Fix Bug 3:** `action variables` invece di `automation variables` per `ent` (fix timeline append).

Automazioni timeline append in `safety_core_automazioni.yaml` (non in log.yaml come in allarme_core).

---

## 9. VARIABILI PERSISTENTI (var component)

File: `var/safety_core.yaml`

| Var | Contenuto |
|-----|-----------|
| `var.safety_core_sensori_attivazione_var` | Comma-separated entity_id causa triggered |
| `var.safety_core_timeline_md` | Timeline completa (long_state) |
| `var.safety_core_timeline_critici_md` | Solo critici 🚨 (max 25 righe) |
| `var.safety_core_timeline_operazioni_md` | Operazioni (max 25 righe) |
| `var.safety_core_timeline_note_md` | Note 📝 (max 25 righe) |
| `var.safety_core_last_snapshot_urls` | URL snapshot (long_state) |
| `var.safety_core_last_clip_urls` | URL clip Frigate (long_state) |

**Nessuna** `var.safety_core_sensori_esclusi_var` — safety_core non esclude sensori.

---

## 10. TEMPLATE SENSOR (`safety_core_log.yaml`)

| Entity | Scopo |
|--------|-------|
| `sensor.safety_core_sensori_attivazione_nomi` | Nomi leggibili sensori causa triggered |
| `sensor.safety_core_timeline_render` | Timeline con emoji colore |
| `sensor.safety_core_count_critici` | Conteggio eventi critici |
| `sensor.safety_core_count_operazioni` | Conteggio operazioni |
| `sensor.safety_core_count_note` | Conteggio note |

---

## 11. SISTEMA BATTERIE (`safety_core_batterie.yaml`)

**Percorso C unico** — `sensore_origine` singolo per tutti i wrapper.

- `input_number.safety_core_soglia_batteria_bassa` — soglia % (5-50, def 20)
- `input_number.safety_core_debounce_offline_minuti` — debounce (1-30 min, def 2)
- `sensor.safety_core_batteria_minima` — output valore minimo globale

---

## 12. SISTEMA ANOMALIE (`safety_core_anomalie_sensori.yaml`)

`binary_sensor.safety_core_anomalia_rilevata` — logica identica ad allarme_core:
- Itera `binary_sensor.safety_core_*` escludendo `_attivo` e `_anomalia`
- Legge `sensore_origine`/`sensori_origine` → stato not in ['on','off']
- Attributi: `sensori_anomali` [{fisico, wrapper, stato}], `contatore`

---

## 13. INTEGRAZIONE FRIGATE

- Snapshot in `/config/www/safety_core_snapshots/`
- Clip -30s/+10s dal triggered (configurabile via input_number)
- Var `safety_core_last_snapshot_urls` e `safety_core_last_clip_urls`
- **Nessun** `binary_sensor.safety_core_dati_pronti` — nessun gate sincronizzazione

---

## 14. DIFFERENZE CHIAVE VS ALLARME_CORE

| Aspetto | safety_core | allarme_core |
|---------|-------------|--------------|
| Stati | ok / triggered | disarmed / arming / armed / triggered |
| Reset | Solo manuale | Manuale + auto-reset |
| Profili/Zone | Nessuno | 5 profili, 5 zone |
| Esclusione sensori | No | Sì (all'arming) |
| Modalità test | No | Sì |
| Master switch | Per categoria (4) | Per singolo sensore |
| Finestra grazia reset | 10 sec | No |
| Gate notifica dati_pronti | No | Sì |
| Trigger debounce | `for: 00:00:03` | No |
| Elettrovalvole | Sì (gas + acqua) | No |
| Batterie percorsi | C unico | A+B+C |

---

## 15. DASHBOARD — PLANCIA CONTROLLO (`plancia_controllo.yaml`)

`type: sections`, path: `safety-core`. Dipendenze: `card_mod`, `browser_mod`

**Sezione 1 — Controllo Safety:**

| Card | Entity/contenuto |
|------|-----------------|
| tile | `input_select.safety_core_stato` |
| tile | `input_select.safety_core_ultima_categoria` |
| tile | `script.safety_core_reset` |
| tile x4 | binary_sensor fumo/gas/carbonio/acqua _attivo |
| tile | `binary_sensor.safety_core_qualsiasi_attivo` |
| tile x4 | Master switch categorie |
| tile x2 | `switch.eth_rly16_board1_relay_1` (GAS) + `switch.eth_rly16_board2_relay_3` (Acqua) |

**Sezioni 2+ — Sensori per categoria:** tile per ogni sensore.

**Bottoni configurazione:**

| Card | Popup |
|------|-------|
| "Configurazione Avanzata" (9 cols) | URL HA/Frigate / Clip secondi / Batterie soglia+debounce |
| "Debug" (3 cols) | var sensori attivazione, reset_in_corso, ultima_categoria, stato |

**Pattern card_mod sensori:**
```
Border: grigio=disabilitato | rosso=in allarme | arancio=fisico anomalo | verde=ok
Badge: 🔕=disabilitato | ⚠️=fisico anomalo
```

tap_action → popup (browser_mod) con `input_boolean` abilitato + `input_text` camera.

**Elettrovalvole:**
- `switch.eth_rly16_board1_relay_1` — Gas
- `switch.eth_rly16_board2_relay_3` — Acqua

**Sezione Log & Attività:**

| Card | Contenuto |
|------|-----------|
| button | Registro (popup 3 markdown: Critici/Operazioni/Note) |
| tile | `script.safety_core_logbook_cancella_tutto` |
| tile | `script.safety_core_logbook_nota_da_ui` |
| custom:timeline-card | `safety_core_stato` + `safety_core_ultima_categoria`, 24h |
| tile | `input_text.safety_core_logbook_ultimo_evento` |
| logbook | `input_text.safety_core_logbook_event` |

---

## 15b. INTEGRAZIONE CON RING KEYPAD

Via `ring-keypad/packages/ring_keypad_integration.yaml` (sezione 3).

**safety-core → tastiera:**

| Stato + Categoria | LED tastiera |
|-------------------|--------------|
| `triggered` + `fumo`/`gas`/`carbonio` | Alarming Smoke-Fire |
| `triggered` + `acqua` | Alarming Water Leak |
| `ok` (dopo reset) | Ripristino LED allarme-core corrente |

**tastiera → safety-core (tasto Fire hold 3s):**
- `safety_core_stato=triggered`, `safety_core_ultima_categoria=fumo`
- Log: `"🔥 Allarme fuoco manuale da tastiera Ring Keypad"`
- Anti-loop: non invia se safety-core già triggered

---

## 16. BUG CORRETTI

| ID | Descrizione | File |
|----|-------------|------|
| Bug 1 | Falsi allarmi post-riavvio da sensori unknown/unavailable — aggiunta condizione fisico realmente on | safety_core_automazioni.yaml |
| Bug 2 | Ri-trigger immediato dopo reset — finestra grazia 10s con reset_in_corso | safety_core_automazioni.yaml + safety_core.yaml |
| Bug 3 | Timeline append: `ent` non valorizzato con automation variables → spostato in action variables | safety_core_automazioni.yaml |
| Bug 4 | `initial: 20` e `initial: 2` su soglia/debounce → reset al default ad ogni riavvio. Rimossi | safety_core_batterie.yaml |

---

## 17. PROCEDURA AGGIUNTA NUOVO SENSORE

File da modificare:

**1. `safety_core_sensori.yaml` — 3 inserimenti:**
- `input_boolean.safety_core_<cat>_<nome>_abilitato`
- `input_text.safety_core_<cat>_<nome>_camera`
- template `binary_sensor` (sensore_origine, tipo, priorita, camera, abilitato, categoria_abilitata, tipo_connessione, batteria_origine, disponibilita_origine)

**2. `plancia_controllo.yaml` — 1 inserimento:**
- Tile nella sezione categoria + tap_action popup + card_mod (border grigio/rosso/arancio/verde + badge)

**File automatici:** `safety_core_batterie.yaml`, `safety_core_anomalie_sensori.yaml`, aggregati `_attivo`

---

## 18. DIPENDENZE ESTERNE

- **var component** (HACS)
- **Frigate** — opzionale
- **Node-RED** — notifiche + azionamento elettrovalvole
- `/config/www/safety_core_snapshots/`
- **card_mod** (HACS)
- **browser_mod** (HACS)
- `switch.eth_rly16_board1_relay_1` e `switch.eth_rly16_board2_relay_3` — elettrovalvole fisiche (ETH-RLY16)
