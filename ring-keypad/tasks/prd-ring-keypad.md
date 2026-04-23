# PRD: Ring Keypad V2 — Home Assistant Integration

> Versione: 1.1 — 2026-04-23
> Sostituisce: `ring-keypad/ANALISI_TECNICA.md`
> Pubblico: Claude (agente AI per modifiche future)

---

## Introduction / Overview

Ring Keypad V2 è un tastierino Z-Wave (via zwave2mqtt) integrato in Home Assistant. Il sistema traduce input fisici (PIN, tasti) in stati HA e propaga feedback visivo/sonoro verso il tastierino. È **autonomo e indipendente**: non contiene logica d'allarme propria, ma funge da bridge bidirezionale tra l'utente fisico e qualsiasi sistema d'allarme esterno (allarme-core, safety-core o alarm_control_panel generico).

**Problema risolto:** senza questo sistema, il Ring Keypad V2 non ha integrazione nativa con Home Assistant tramite zwave2mqtt. Il protocollo Entry Control CC (111) e Indicator CC (135) richiede logica custom per ogni comando.

**Principi fondamentali:**
- Multi-tastiera con wildcard MQTT: N tastiere = codice fisso, zero duplicazioni
- Unica fonte di verità: `input_select.ring_keypad_alarm_state`
- Stato passato **sempre** attraverso `sync_state` → mai comandi MQTT diretti fuori dai script
- Anti-loop bidirezionale: evita che keypad→core e core→keypad si rincorrano

---

## Goals

- Supportare inserimento/disinserimento allarme tramite PIN da tastiera fisica
- Feedback visivo (LED) e sonoro immediato su ogni tastiera
- Supportare N tastiere senza duplicare codice (wildcard MQTT)
- Integrazione bidirezionale con allarme-core (profili: notte, giorno, sera, tutti)
- Integrazione con safety-core (fumo, acqua)
- Protezione brute-force PIN con lockout automatico
- Bypass sensori aperti prima dell'armo con conferma fisica
- Exit delay silenzioso per profilo notte (armed_home), sonoro altrimenti
- Log timeline con categorizzazione eventi (operazioni, critici, note)
- Configurazione completa da UI HA senza modificare YAML

---

## User Stories

### US-001: Inserimento allarme da tastiera con PIN
**Description:** As a utente, I want to insert the alarm by entering a PIN on the keypad so that the system arms with the correct profile.

**Acceptance Criteria:**
- [ ] ARM HOME (6/2) + PIN valido → arma profilo notte (armed_home)
- [ ] ARM AWAY (5/2) + PIN valido → arma profilo giorno (armed_away)
- [ ] Tasto V/Enter (2/2) + PIN valido → arma profilo sera (armed_sera)
- [ ] PIN errato → feedback sonoro `Code_not_accepted` + log tentativo
- [ ] Exit delay visivo su tutte le tastiere durante arming
- [ ] Exit delay silenzioso per profilo notte, sonoro per gli altri

### US-002: Disinserimento allarme da tastiera
**Description:** As a utente, I want to disarm the alarm entering a PIN so that I can enter safely.

**Acceptance Criteria:**
- [ ] DISARM (3/2) + PIN valido → disarma, LED disarmed su tutte le tastiere
- [ ] PIN errato durante entry delay → `Code_not_accepted` + log
- [ ] Disarmo durante exit delay → interrompe arming (do_arm fermato)

### US-003: Bypass sensori aperti
**Description:** As a utente, I want to arm even with open sensors so that I can arm quickly when needed.

**Acceptance Criteria:**
- [ ] Sensore aperto nella zona target → `Bypass_challenge` LED + keepalive ogni 5s
- [ ] Tasto V → conferma bypass e arma
- [ ] Tasto X → annulla bypass, LED ripristinato
- [ ] Timeout (configurabile 5-60s) → annulla bypass automaticamente

### US-004: Feedback visivo di stato su tutte le tastiere
**Description:** As a utente, I want all keypads to show the correct LED state so that I can see the alarm status at a glance.

**Acceptance Criteria:**
- [ ] disarmed → `Not_armed_-_disarmed/9` voice-aware
- [ ] armed_home → `Armed_Stay/9` voice-aware
- [ ] armed_away → `Armed_Away/9` voice-aware
- [ ] armed_sera → `Armed_Stay/9` payload `1` fisso + LED fast blink (System_Security_Mode_Display=1)
- [ ] arming → `Exit_Delay/9` (1 notte, 99 altrimenti) + `Exit_Delay/timeout` countdown
- [ ] triggered_burglar → `Alarming/9` payload `99` (con suono)
- [ ] triggered_burglar_sera → `Alarming/9` payload `1` (silenzioso)
- [ ] triggered_rapina → `Alarming_Burglar/9` payload `1` (sempre silenzioso)
- [ ] triggered_fire → `Alarming_Smoke_-_Fire/9` payload `99`
- [ ] triggered_medical → `Alarming_Medical/9` payload `99`
- [ ] triggered_water → `Alarming_Water_leak/9` payload `99`

### US-005: Integrazione bidirezionale con allarme-core
**Description:** As a sistema, I want keypad and allarme-core to stay in sync so that changes from either side are reflected everywhere.

**Acceptance Criteria:**
- [ ] Armo da tastiera → allarme-core arma con profilo corretto
- [ ] Armo da allarme-core → tastiera mostra LED corretto + exit delay silenzioso/sonoro corretto
- [ ] Disarmo da tastiera → allarme-core disarma
- [ ] Disarmo da allarme-core → tastiera LED disarmed
- [ ] Anti-loop: nessun rimbalzo stato keypad↔core

### US-006: Protezione brute-force PIN
**Description:** As a amministratore, I want the system to block repeated wrong PINs so that brute-force attacks are prevented.

**Acceptance Criteria:**
- [ ] Contatore tentativi falliti incrementato ad ogni PIN errato
- [ ] Lockout attivato dopo N tentativi (N configurabile 3-10, default 5)
- [ ] Durante lockout: risposta identica a PIN errato (nessun feedback diverso)
- [ ] Lockout reset automatico dopo 5 minuti
- [ ] Contatore reset dopo 60s di inattività (sliding window)

### US-007: Log e timeline eventi
**Description:** As a utente, I want a searchable log of all keypad events so that I can audit who armed/disarmed and when.

**Acceptance Criteria:**
- [ ] Ogni armo/disarmo loggato con utente e tastiera
- [ ] PIN errati loggati con contatore (N/M tentativi)
- [ ] Allarmi critici (rapina, fuoco, medico, intrusione) in sezione Critici
- [ ] Timeline visuale ultimi 24h in dashboard
- [ ] Note manuali aggiungibili da UI

---

## Functional Requirements

**Protocollo Z-Wave:**
- FR-1: Entry Control CC (unknownClass_111/endpoint_0) per input tastiera
- FR-2: Indicator CC (indicator/endpoint_0) per output LED/suono
- FR-3: Configuration CC (configuration/endpoint_0/System_Security_Mode_Display) per LED display
- FR-4: Battery CC (battery/endpoint_0/level) per livello batteria
- FR-5: Status topic (zwave2mqtt/{id}/status) per connettività nodo

**State machine:**
- FR-6: 11 stati possibili: `disarmed | arming | armed_home | armed_away | armed_sera | triggered_burglar | triggered_burglar_sera | triggered_rapina | triggered_fire | triggered_medical | triggered_water`
- FR-7: Unica fonte di verità: `input_select.ring_keypad_alarm_state`
- FR-8: Ogni cambio stato → `sync_all` su tutte le tastiere (via `sync_on_state_change`)

**Multi-tastiera:**
- FR-9: Wildcard MQTT `+` cattura qualsiasi tastiera; `keypad_id = trigger.topic.split('/')[1]`
- FR-10: Lista tastiere attive in `input_text.ring_keypad_active_keypads` (CSV)
- FR-11: Aggiunta nuova tastiera: solo CSV + blocco sensor in `ring_keypad_keypads.yaml`

**PIN e sicurezza:**
- FR-12: 4 slot PIN (master + user1/2/3), pattern `^$|^[0-9]{4,8}$`
- FR-13: PIN < 4 cifre ignorati; PIN vuoti mai validi
- FR-14: Rilevazione collisione PIN tramite `binary_sensor.keypad_collisione_pin`
- FR-15: Lockout con reset automatico 5 min + sliding window 60s

**Armo:**
- FR-16: `ring_keypad_arming_target_mode` impostato PRIMA di `alarm_state = "arming"` in entrambi i flussi (tastiera e esterno)
- FR-17: Exit delay volume: payload `1` se `arming_target_mode == armed_home`, `99` altrimenti
- FR-18: Payload sub_cmd 9 (level): intero senza virgolette (`1` o `99`)
- FR-19: Payload sub_cmd timeout: stringa JSON con virgolette (`"XmYs"`)
- FR-20: `ring_keypad_exit_delay` sempre sincronizzato con `allarme_core_arming_delay`

**Bypass:**
- FR-21: Verifica sensori aperti zone target prima di armare
- FR-22: Bypass keepalive ogni 5s (sub_cmd 9, payload 1 = solo LED) durante attesa
- FR-23: `bypass_cancel` chiamato prima di qualsiasi reset di stato

**LED:**
- FR-24: `armed_sera`: forza `System_Security_Mode_Display = 1` (fast blink), bypassa preferenza utente
- FR-25: Cambio `ring_keypad_led_mode` non applicato se `alarm_state == armed_sera`
- FR-26: `sync_state` chiama `set_led_mode` all'inizio (ripristina preferenza), poi override per armed_sera

**Voice feedback:**
- FR-27: `ring_keypad_voice_feedback = on` → payload `99` per stati normali (LED + suono)
- FR-28: `ring_keypad_voice_feedback = off` → payload `1` (solo LED)
- FR-29: Eccezioni fisse: `code_rejected` sempre `99`; `armed_sera` sempre `1`; sicurezza vitale (fire/medical/water) sempre `99`; anti-rapina sempre `1`

**Log:**
- FR-30: Ogni evento loggato via `ring_keypad_logbook_emit` → logbook HA + `last_event`
- FR-31: Routing emoji per categoria (operazioni / critici / note)
- FR-32: `logbook_emit` emette prima stringa vuota poi messaggio (fix C-12: garantisce `from_state != to_state` sempre)

**Debug mode:**
- FR-33: `ring_keypad_debug = on` blocca `command_to_allarme_core` e `command_to_safety_core`
- FR-34: `sync_from_*` continuano normalmente con debug on

---

## Non-Goals

- Nessuna logica d'allarme propria (zone, timer, profili): delegata ad allarme-core
- Nessuna cifratura PIN (stored in `input_text mode:password`, non in secrets.yaml)
- Nessun supporto Entry Delay hardware (gestito da allarme-core)
- Nessuna integrazione con sistemi diversi da allarme-core e safety-core senza modifiche esplicite
- Nessuna modifica automatica del profilo allarme-core (solo impostazione profilo + arm)

---

## Technical Considerations

### Struttura file

```
ring-keypad/
├── packages/
│   ├── ring_keypad_globals.yaml        # Helper: input_select, input_text, input_number, input_boolean
│   ├── ring_keypad_scripts.yaml        # Script parametrici (accettano keypad_id)
│   ├── ring_keypad_automations.yaml    # Automazioni MQTT wildcard + sistema
│   ├── ring_keypad_keypads.yaml        # Sensori MQTT fisici per tastiera (batteria + online)
│   ├── ring_keypad_integration.yaml    # Bridge bidirezionale: allarme-core + safety-core
│   └── ring_keypad_log.yaml            # Log: logbook, timeline, note
├── var/ring_keypad.yaml                # Var timeline (4 var: globale, critici, operazioni, note)
├── tasks/prd-ring-keypad.md            # Questo file
└── ANALISI_TECNICA.md                  # Deprecato — sostituito da questo PRD
```

**Mappatura file → entità:**

| File | Entità |
|------|--------|
| `ring_keypad_globals.yaml` | `input_select`, `input_text`, `input_number`, `input_boolean`, `binary_sensor` template |
| `ring_keypad_scripts.yaml` | `script.*` |
| `ring_keypad_automations.yaml` | `automation.*` (trigger MQTT + sistema) |
| `ring_keypad_keypads.yaml` | `mqtt.sensor`, `mqtt.binary_sensor` per tastiera |
| `ring_keypad_integration.yaml` | `automation.*` (bridge esterno) |
| `ring_keypad_log.yaml` | `input_text` logbook, `script.*` log, `automation.*` log, template sensor timeline |

**Regola:** ogni nuova entità va dichiarata nel file che la usa. Non aggiungere entità a file arbitrari.

### Protocollo Z-Wave / MQTT

#### Entry Control CC (111) — Input da tastiera

Topic: `zwave2mqtt/{keypad_id}/unknownClass_111/endpoint_0/{event_type}/{data_type}`

| event_type | data_type | Significato |
|------------|-----------|-------------|
| 3 | 2 | Disarm + codice |
| 6 | 2 | Arm Home + codice |
| 6 | 0 | Arm Home senza codice |
| 5 | 2 | Arm Away + codice |
| 5 | 0 | Arm Away senza codice |
| 2 | 2 | Tasto V + codice → arma sera |
| 2 | 0 | Tasto V senza codice |
| 25 | 2 | Tasto X + codice |
| 25 | 0 | Tasto X senza codice |
| 17 | 0 | Police hold 3s → anti-rapina |
| 19 | 0 | Medical hold 3s |
| 16 | 0 | Fire hold 3s |

Payload con codice: `{"time":..., "value":"1234"}` — estrai con `(trigger.payload_json | default({})).get('value', '')`

#### Indicator CC — Output verso tastiera

Topic: `zwave2mqtt/{keypad_id}/indicator/endpoint_0/{Name}/{sub_cmd}/set`

| sub_cmd | Tipo payload | Significato |
|---------|-------------|-------------|
| `1` | intero `0` o `99` | On/Off indicatore (voce) |
| `9` | intero `1` o `99` | Level: `1`=silenzioso, `99`=con suono |
| `timeout` | stringa JSON `"XmYs"` | Countdown (virgolette obbligatorie) |

**Indicatori e payload:**

| Indicatore | sub_cmd | Payload | Nota |
|------------|---------|---------|------|
| `Not_armed_-_disarmed` | 9 | voice-aware | |
| `Armed_Stay` | 9 | voice-aware | |
| `Armed_Away` | 9 | voice-aware | |
| `Armed_Stay` (sera) | 9 | `1` fisso | |
| `Code_not_accepted` | 9 | `99` fisso | |
| `Bypass_challenge` | 1 | `99` (primo) / 9+`1` (keepalive) | |
| `Exit_Delay` | 9 | `1` se notte, `99` altrimenti | |
| `Exit_Delay` | timeout | `"XmYs"` | |
| `Alarming` | 9 | `99` (burglar) / `1` (sera) | |
| `Alarming_Burglar` | 9 | `1` fisso | anti-rapina |
| `Alarming_Smoke_-_Fire` | 9 | `99` fisso | |
| `Alarming_Medical` | 9 | `99` fisso | |
| `Alarming_Water_leak` | 9 | `99` fisso | |

**CRITICO:** sub_cmd `9` e `1` usano payload numerico senza virgolette. Sub_cmd `timeout` usa stringa JSON con virgolette. Non confondere i due formati.

#### LED Display

Topic: `zwave2mqtt/{keypad_id}/configuration/endpoint_0/System_Security_Mode_Display/set`

| Payload | Descrizione |
|---------|-------------|
| `0` | LED spenti |
| `1` | Fast blink ~1s (armed_sera) |
| `3` | Lampeggio 3s |
| `601` | Sempre accesi |

### Entità Helper Complete

#### `input_select`

| Entità | Valori | Descrizione |
|--------|--------|-------------|
| `ring_keypad_alarm_state` | disarmed, armed_home, armed_away, armed_sera, arming, triggered_burglar, triggered_burglar_sera, triggered_rapina, triggered_fire, triggered_medical, triggered_water | **Unica fonte di verità** |
| `ring_keypad_led_mode` | Spenti, Lampeggio (3s), Sempre accesi | Controlla System_Security_Mode_Display |

#### `input_text`

| Entità | Descrizione | Note |
|--------|-------------|------|
| `ring_keypad_active_keypads` | CSV topic base tastiere attive | Es. `tastiera_allarme_camera,tastiera_allarme_sala` |
| `ring_keypad_zwave_entity_ids` | JSON `{"topic_base": "entity_id_zwave_js"}` | Per ping_verify |
| `ring_keypad_arming_target_mode` | Target mode corrente (`armed_home/away/sera`) | Impostato da `do_arm` E da `sync_from_allarme_core` caso arming |
| `ring_keypad_bypass_mode` | Profilo del bypass pendente | Reset da `bypass_cancel` |
| `ring_keypad_bypass_keypad` | Tastiera che ha avviato il bypass | Reset da `bypass_cancel` |
| `ring_keypad_last_user` | Ultimo utente autenticato | Log |
| `ring_keypad_last_event` | Ultimo evento testuale | Trigger log timeline |
| `ring_keypad_last_keypad` | Topic base ultima tastiera | Log |
| `ring_keypad_pin_master` | PIN master | mode: password |
| `ring_keypad_pin_user1/2/3` | PIN utenti | mode: password |
| `ring_keypad_name_master` | Nome master | Log |
| `ring_keypad_name_user1/2/3` | Nomi utenti | Log |
| `ring_keypad_logbook_event` | Dummy per Logbook card | |
| `ring_keypad_logbook_ultimo_evento` | Trigger timeline append | |
| `ring_keypad_nota_manuale` | Note manuali da UI | |

#### `input_number`

| Entità | Default | Descrizione |
|--------|---------|-------------|
| `ring_keypad_exit_delay` | 30s | Read-only — sincronizzato da `allarme_core_arming_delay` |
| `ring_keypad_bypass_timeout` | 30s (range 5-60) | Timeout conferma bypass |
| `ring_keypad_siren_volume` | 5 (range 1-10) | Volume sirena via Siren_Volume CC |
| `ring_keypad_failed_attempts` | 0 (`initial: 0` ok — tecnico interno) | Contatore PIN falliti |
| `ring_keypad_max_failed_attempts` | 5 (range 3-10) | Soglia lockout |

#### `input_boolean`

| Entità | Default | Descrizione |
|--------|---------|-------------|
| `ring_keypad_voice_feedback` | true | on=99 (LED+suono); off=1 (solo LED) |
| `ring_keypad_require_code_to_arm` | false | PIN obbligatorio per armare |
| `ring_keypad_bypass_pending` | false | Flag bypass attivo |
| `ring_keypad_debug` | false | Blocca propagazione keypad→core |
| `ring_keypad_lockout_active` | false (`initial: false` ok — tecnico interno) | Lockout brute-force |

#### `binary_sensor` template (in globals.yaml)

| Entità | Descrizione |
|--------|-------------|
| `binary_sensor.keypad_collisione_pin` | On se 2+ utenti hanno stesso PIN (≥4 cifre) |

#### Sensori MQTT (in keypads.yaml)

Pattern `default_entity_id`:
- `sensor.tastiera_{nome}_allarme_batteria`
- `binary_sensor.tastiera_{nome}_allarme_online`

Tastiere attive:
- `tastiera_allarme_camera` → `sensor.tastiera_camera_allarme_batteria`, `binary_sensor.tastiera_camera_allarme_online`
- `tastiera_allarme_sala` → `sensor.tastiera_sala_allarme_batteria`, `binary_sensor.tastiera_sala_allarme_online`

### Script Reference (ring_keypad_scripts.yaml)

**Script hardware — indicatori:**

| Script | Azione |
|--------|--------|
| `ring_keypad_send_indicator` | Invia Indicator CC generico |
| `ring_keypad_show_disarmed` | `Not_armed_-_disarmed/9` voice-aware |
| `ring_keypad_show_armed_home` | `Armed_Stay/9` voice-aware |
| `ring_keypad_show_armed_away` | `Armed_Away/9` voice-aware |
| `ring_keypad_show_armed_sera` | `Armed_Stay/9` payload `1` + `System_Security_Mode_Display=1` |
| `ring_keypad_show_exit_delay` | `Exit_Delay/9` volume (1/99) + `Exit_Delay/timeout` countdown |
| `ring_keypad_code_rejected` | `Code_not_accepted/9` payload `99` fisso |
| `ring_keypad_bypass_challenge` | `Bypass_challenge/1` payload `99` + avvia keepalive + post-ping |
| `ring_keypad_bypass_keepalive` | `mode:restart` — ogni 5s `Bypass_challenge/9` payload `1` finché bypass_pending=on |
| `ring_keypad_alarm_burglar` | `Alarming/9` payload `99` |
| `ring_keypad_alarm_burglar_sera` | `Alarming/9` payload `1` |
| `ring_keypad_alarm_rapina` | `Alarming_Burglar/9` payload `1` fisso |
| `ring_keypad_alarm_fire` | `Alarming_Smoke_-_Fire/9` payload `99` |
| `ring_keypad_alarm_medical` | `Alarming_Medical/9` payload `99` |
| `ring_keypad_alarm_water` | `Alarming_Water_leak/9` payload `99` |

**Script LED:**

| Script | Azione |
|--------|--------|
| `ring_keypad_set_led_mode` | Pubblica LED da `ring_keypad_led_mode` su tastiera singola |
| `ring_keypad_set_led_mode_all` | Itera tastiere → `set_led_mode` |

**Script volume sirena:**

| Script | Azione |
|--------|--------|
| `ring_keypad_set_siren_volume` | Pubblica volume sirena su singola tastiera |
| `ring_keypad_set_siren_volume_all` | Itera tastiere → `set_siren_volume` |

**Script sincronizzazione:**

| Script | Azione |
|--------|--------|
| `ring_keypad_sync_state` | Dispatcher per singola tastiera: `set_led_mode` poi indicatore per stato |
| `ring_keypad_sync_all` | Pre-check disponibilità nodi → `sync_state` su tutte le tastiere → `verify_all` fire-and-forget |
| `ring_keypad_broadcast_alarm` | Pre-check → allarme specifico su tutte le tastiere → post-ping |

**Script logica:**

| Script | Azione |
|--------|--------|
| `ring_keypad_resolve_user` | Identifica utente dal PIN → `last_user` |
| `ring_keypad_validate_and_disarm` | Valida PIN → disarma o rifiuta; gestisce lockout e brute-force |
| `ring_keypad_validate_and_arm` | Valida PIN → `arm_or_challenge`; gestisce lockout |
| `ring_keypad_arm_or_challenge` | Controlla sensori aperti zone target: pulito → `do_arm`; aperti → bypass |
| `ring_keypad_bypass_confirm` | `turn_off do_arm` → `bypass_cancel` → `do_arm(stored_mode)` |
| `ring_keypad_bypass_cancel` | Reset `bypass_pending/mode/keypad`; ferma `bypass_timeout_script` e `bypass_keepalive` |
| `ring_keypad_bypass_timeout_script` | `mode:restart` — dopo timeout: reset inline bypass → `sync_state` |
| `ring_keypad_do_disarm` | `bypass_cancel` → `turn_off do_arm` → `alarm_state=disarmed` → `sync_all` → log |
| `ring_keypad_do_arm` | `mode:restart` — `arming_target_mode=mode` → `alarm_state=arming` → delay → `alarm_state=mode` |
| `ring_keypad_ping_verify` | `mode:queued` — ping Z-Wave singola tastiera; logga FAIL (+ OK se manual=true) |
| `ring_keypad_verify_all` | `mode:restart` — itera tastiere → `ping_verify` |

**Script log:**

| Script | Azione |
|--------|--------|
| `ring_keypad_logbook_emit` | Emette stringa vuota poi `msg` su `logbook_ultimo_evento` + logbook.log |
| `ring_keypad_logbook_nota` | Prefissa `📝 Nota:` → `logbook_emit` |
| `ring_keypad_logbook_nota_da_ui` | Legge `nota_manuale` → `logbook_emit` → svuota campo |
| `ring_keypad_logbook_cancella_tutto` | Azzera 4 `var.*` timeline |

### Automazioni Reference

**Automazioni input (ring_keypad_automations.yaml):**

| ID | Trigger MQTT | Azione |
|----|-------------|--------|
| `ring_keypad_input_disarm` | `3/2` | `validate_and_disarm` |
| `ring_keypad_input_arm_away` | `5/2` | Se bypass_pending → `code_rejected`; altrimenti `validate_and_arm armed_away` |
| `ring_keypad_input_arm_away_nocode` | `5/0` | Se bypass_pending → `code_rejected`; se require_code=off → `arm_or_challenge`; altrimenti `code_rejected` |
| `ring_keypad_input_arm_home` | `6/2` | Se bypass_pending → `code_rejected`; altrimenti `validate_and_arm armed_home` |
| `ring_keypad_input_arm_home_nocode` | `6/0` | Se bypass_pending → `code_rejected`; se require_code=off → `arm_or_challenge`; altrimenti `code_rejected` |
| `ring_keypad_input_enter` | `2/2` | Se bypass_pending → `bypass_confirm`; altrimenti `validate_and_arm armed_sera` |
| `ring_keypad_input_enter_nocode` | `2/0` | Se bypass_pending → `bypass_confirm`; se require_code=off + doppia guard → `arm_or_challenge armed_sera` |
| `ring_keypad_input_cancel` | `25/2` + `25/0` | Se bypass_pending per questa tastiera → `bypass_cancel` + `sync_state`; sempre log |
| `ring_keypad_input_rapina` | `17/0` | `alarm_state=triggered_rapina` + `broadcast_alarm rapina` |
| `ring_keypad_input_medical` | `19/0` | `alarm_state=triggered_medical` + `broadcast_alarm medical` |
| `ring_keypad_input_fire` | `16/0` | `alarm_state=triggered_fire` + `broadcast_alarm fire` |

**Automazioni sistema:**

| ID | Trigger | Azione |
|----|---------|--------|
| `ring_keypad_sync_on_state_change` | State: `alarm_state` | `sync_all` |
| `ring_keypad_restore_on_connect` | MQTT `zwave2mqtt/+/status` = Alive | delay 3s → `set_siren_volume` → `sync_state` |
| `ring_keypad_sync_on_ha_start` | HA start | delay 15s → safety-net lockout reset → `set_siren_volume_all` → `sync_all` |
| `ring_keypad_sync_led_on_mode_change` | State: `led_mode` | `set_led_mode_all` solo se `alarm_state != armed_sera` |
| `ring_keypad_sync_siren_volume_on_change` | State: `siren_volume` | `set_siren_volume_all` |
| `ring_keypad_lockout_reset` | State: `lockout_active=on` | delay 5 min → se ancora on: disattiva + azzera + log; `mode:restart` |
| `ring_keypad_failed_attempts_reset` | State: `failed_attempts` invariato 60s (+ lockout off + > 0) | Azzera contatore + svuota `last_event` e `logbook_ultimo_evento` |
| `ring_keypad_log_debug_mode` | State: `ring_keypad_debug` → on/off | `logbook_emit` con messaggio attivazione/disattivazione |

### Integrazione Sistemi Esterni (ring_keypad_integration.yaml)

#### allarme-core → keypad (`sync_from_allarme_core`)

`mode: queued, max: 5`. Trigger su `allarme_core_stato` SEMPRE e su `allarme_core_profilo` SOLO se `stato == armed`.

| `allarme_core_stato` | `allarme_core_profilo` | `ring_keypad_alarm_state` |
|----------------------|------------------------|---------------------------|
| `disarmed` | qualsiasi | `disarmed` + `bypass_cancel` + `turn_off do_arm` + `sync_all` |
| `arming` | qualsiasi | imposta `arming_target_mode` dal profilo → `arming` |
| `armed` | `notte` | `armed_home` (con guardia anti-race do_arm) |
| `armed` | `giorno`/`tutti` | `armed_away` (con guardia anti-race do_arm) |
| `armed` | `sera` | `armed_sera` (con guardia anti-race do_arm) |
| `triggered` | `sera` | `triggered_burglar_sera` |
| `triggered` | altri | `triggered_burglar` |

**Guardia anti-race armed:** non sovrascrive se `alarm_state == arming` E `do_arm` è in esecuzione.

#### keypad → allarme-core (`command_to_allarme_core`)

`mode: queued, max: 5`. Trigger su `alarm_state → disarmed/arming/armed_home/armed_away/armed_sera`. Bloccato se `debug = on`.

| `alarm_state` | Profilo | Script | Anti-loop |
|---------------|---------|--------|-----------|
| `disarmed` | — | `disarm_allarme_core` | `from_state not in [disarmed, unknown]` OR `allarme_core != disarmed` |
| `arming` | da `arming_target_mode` | `arm_allarme_core` (fire-and-forget) | `allarme_core_stato not in [arming, armed]` |
| `armed_home` | notte | `arm_allarme_core` (fire-and-forget) | `stato != arming AND (stato != armed OR profilo != notte)` |
| `armed_away` | giorno | `arm_allarme_core` (fire-and-forget) | `stato != arming AND (stato != armed OR profilo not in [giorno, tutti])` |
| `armed_sera` | sera | `arm_allarme_core` (fire-and-forget) | `stato != arming AND (stato != armed OR profilo != sera)` |

#### safety-core → keypad (`sync_from_safety_core`)

| `safety_core_stato` | `safety_core_ultima_categoria` | `alarm_state` |
|---------------------|-------------------------------|---------------|
| `triggered` | fumo/gas/carbonio | `triggered_fire` + `turn_off do_arm` |
| `triggered` | acqua | `triggered_water` + `turn_off do_arm` |
| `ok` | qualsiasi | Re-sync da allarme-core (stessa mappatura di `sync_from_allarme_core`) + `sync_all` |

#### keypad → safety-core (`command_to_safety_core`)

Trigger: `alarm_state → triggered_fire`. Anti-loop: `safety_core_stato != triggered`. Bloccato se `debug = on`.

#### Sync exit delay (`ring_keypad_sync_exit_delay`)

Trigger: `allarme_core_arming_delay` cambia O HA start → copia valore in `ring_keypad_exit_delay`.

### Anti-Loop Bidirezionale

```
allarme-core cambia → sync_from aggiorna alarm_state → command_to si attiva
    → CONTROLLO: allarme-core già nello stato corretto?
        ├─ SI → nessun comando (loop interrotto)
        └─ NO → comando inviato (caso reale da tastiera)
```

### Aggiungere una Nuova Tastiera

1. UI HA → aggiungi topic a `ring_keypad_active_keypads` (CSV)
2. `ring_keypad_keypads.yaml` → blocco sensor batteria + binary_sensor online con `default_entity_id` esplicito
3. `ring_keypad_zwave_entity_ids` (UI) → aggiungi `{"topic_base": "zwave_js_entity_id"}` per ping
4. Riavviare HA
5. Nessuna altra modifica — wildcard + `_all` scripts gestiscono tutto

---

## Dashboard — `plancia_controllo.yaml`

File: `ring-keypad/plancia_controllo.yaml`
Tipo: `sections view`, path: `ring-keypad`, `subview: true`, max_columns: 4.
Back path: `/home-new/controllo-generale`.
Richiede: `card_mod`, `mushroom-template-card`, `custom:timeline-card`, `browser_mod` (per popup).

### Header — Banner condizionali

I banner appaiono in cima alla view in base allo stato. Sono `conditional` cards dentro `header.cards`.

| Condizione | Tipo alert | Messaggio |
|-----------|------------|-----------|
| `ring_keypad_debug = on` | `warning` | Debug attivo |
| `alarm_state` in triggered_burglar/rapina/fire/medical/water | `error` | Allarme in corso + tipo (mappa stato→etichetta leggibile) |
| `ring_keypad_bypass_pending = on` | `warning` | Bypass in attesa — premere V o X |
| `ring_keypad_lockout_active = on` | `error` | Lockout attivo — reset automatico dopo 5 min |
| `keypad_collisione_pin = on` | `warning` | Collisione PIN — verificare configurazione |

**Nota:** il banner triggered NON include `triggered_burglar_sera` (omesso nella condizione OR del header).

### Sezione 1 — Controllo Tastiere Ring

| Card | Tipo | Dettaglio |
|------|------|-----------|
| Tile Stato Allarme | `tile` su `alarm_state` | Bordo colorato: rosso=triggered, arancio=arming/armed_*, verde=disarmed; `columns: 12` |
| Mushroom Template | `mushroom-template-card` | Primary: stato leggibile (mappa 10 stati → IT); Secondary: utente + tastiera; Icona + colore dinamici per stato; `columns: 12` |
| Conditional bypass | `mushroom-template-card` dentro conditional | Visibile se `bypass_pending=on`; mostra tastiera e profilo bypass; `columns: 12` |
| Conditional allarme | `markdown` dentro conditional | Visibile se triggered_*; `ha-alert error` con testo differenziato per rapina; `columns: 12` |

**Icone stato mushroom:**
- `triggered_rapina` → `mdi:alarm-light-off` (silenzioso)
- altri triggered_* → `mdi:alarm-light`
- armed_* → `mdi:shield-lock`
- arming → `mdi:shield-sync`
- disarmed → `mdi:shield-home-outline`

**Colori icona mushroom:**
- triggered_* → rosso
- armed_sera → `#8B6FDB` (viola)
- armed_home/away/arming → arancio
- disarmed → verde

### Sezione 2 — Stato Hardware

| Card | Entità | Dettaglio |
|------|--------|-----------|
| Tile Online Camera | `binary_sensor.tastiera_camera_allarme_online` | Bordo verde/rosso; `columns: 6` |
| Tile Batteria Camera | `sensor.tastiera_camera_allarme_batteria` | Bordo verde>30%, arancio>15%, rosso≤15%; `columns: 6` |
| Tile Online Sala | `binary_sensor.tastiera_sala_allarme_online` | Stessa logica Camera; `columns: 6` |
| Tile Batteria Sala | `sensor.tastiera_sala_allarme_batteria` | Stessa logica Camera; `columns: 6` |
| Mushroom Tastiere attive | template su `ring_keypad_active_keypads` | Lista CSV → join con `•` + conteggio; icona verde/rosso; `columns: 12` |

**Aggiungere tastiera in hardware:** replicare i due tile (Online + Batteria) nella sezione con le entity ID della nuova tastiera.

### Sezione 3 — Configurazione

| Card | Entità | Bordo | Dettaglio |
|------|--------|-------|-----------|
| Tile Feedback vocale | `ring_keypad_voice_feedback` | verde/grigio | `columns: 6` |
| Tile Richiedi PIN | `ring_keypad_require_code_to_arm` | arancio(on)/verde(off) | `columns: 6` |
| Tile Modalità LED | `ring_keypad_led_mode` | grigio/arancio/blu | Colore per valore (Spenti/Lampeggio/Sempre) |
| Tile Debug Mode | `ring_keypad_debug` | arancio/grigio | `columns: 6` |
| Tile Exit Delay | `ring_keypad_exit_delay` | grigio, opacity 0.75 | Read-only (sync da allarme-core); nessun `numeric-input` |
| Tile Volume Sirena | `ring_keypad_siren_volume` | blu≤3/arancio≤7/rosso>7 | Feature `numeric-input buttons`; `rows: 2` |
| Tile Timeout Bypass | `ring_keypad_bypass_timeout` | blu≤30/arancio≤60/viola>60 | Feature `numeric-input buttons`; `rows: 2` |
| Tile Soglia Lockout | `ring_keypad_max_failed_attempts` | rosso≤3/arancio≤5/blu>5 | Feature `numeric-input buttons`; `rows: 2` |
| Tile Lockout Brute-Force | `ring_keypad_lockout_active` | rosso(on)/grigio(off) | `columns: 6` |
| Tile Tentativi falliti | `ring_keypad_failed_attempts` | rosso≥max/arancio>0/grigio | Read-only; `columns: 6` |
| Button Sync Forzato | script `sync_all` | blu | Tap: sync_all; Hold: set_siren_volume_all; `columns: 6` |
| Button Configurazione Avanzata | popup `browser_mod` | blu | Popup: PIN + nomi utenti, tastiere attive, LED, voice, zwave_entity_ids; `rows: 2` |
| Button Debug | popup `browser_mod` | grigio opacity 0.55 | Popup: debug mode + tutte le entità diagnostiche (last_user/event/keypad, arming_target_mode, bypass, lockout, collisione); `rows: 2` |
| Button Test Ping Z-Wave | popup `browser_mod` | blu | Popup: bottone avvia `verify_all manual=true`; sezione risultati OK e FAIL da timeline_md; `rows: 2` |

**Popup Configurazione Avanzata** contiene:
- entities card: nomi e PIN master + user1/2/3
- conditional: warning collisione PIN se `keypad_collisione_pin=on`
- entities card: `ring_keypad_active_keypads`, `led_mode`, `voice_feedback`
- markdown istruzioni + entities card: `ring_keypad_zwave_entity_ids`

**Popup Debug** contiene entities card con: `debug`, `last_user`, `last_event`, `last_keypad`, `alarm_state`, `arming_target_mode`, `bypass_pending`, `bypass_mode`, `bypass_keypad`, divider, `lockout_active`, `failed_attempts`, `keypad_collisione_pin`.

### Sezione 4 — Log & Attività

| Card | Tipo | Dettaglio |
|------|------|-----------|
| Mushroom Ultimo Evento | template su `last_event` | Primary: "Ultimo Evento"; Secondary: valore o "Nessun evento"; icona arancio/grigio; `columns: 12, rows: 2` |
| Tile Ultima Tastiera | `last_keypad` | Bordo grigio; `columns: 4` |
| Tile Ultimo Utente | `last_user` | Bordo grigio; `columns: 4` |
| Button Registro | popup `browser_mod` | `columns: 6, rows: 2` — popup con 3 markdown (Critici/Operazioni/Note) da `var.*_timeline_md`; max-height 200px ciascuno |
| Tile Cancella Log | script `logbook_cancella_tutto` | `columns: 3` |
| Tile Aggiungi Nota | popup nota manuale | `columns: 3` — popup: `nota_manuale` input + `logbook_nota_da_ui` save |
| Timeline Card | `custom:timeline-card` | 24h, limit 50, overflow scroll, max_height 480px, refresh 30s; entità: `alarm_state` (colori + icone per stato), `last_user`, `lockout_active` |
| Tile Ultimo Evento Log | `ring_keypad_logbook_ultimo_evento` | `columns: 12` |

**Colori timeline alarm_state:**
- disarmed → verde
- armed_home → giallo
- armed_away → arancio
- armed_sera → viola
- arming → grigio
- triggered_* → rosso

**Nota implementativa:** quando si aggiunge una tastiera, replicare i tile Online + Batteria nella Sezione 2. La dashboard non richiede altre modifiche (le sezioni Log e Controllo sono agnostiche al numero di tastiere).

## Success Metrics

- Inserimento allarme da tastiera: LED arming visibile entro 1s dalla pressione tasto
- Disinserimento: LED disarmed su tutte le tastiere entro 2s
- Exit delay silenzioso per profilo notte: nessun beep durante countdown
- Nessuna propagazione loop keypad↔core verificabile da logbook
- Lockout attivo dopo N PIN errati, reset automatico dopo 5 min
- Ogni evento loggato nel logbook HA con utente e tastiera

---

## Open Questions

- **Entry Delay visivo:** attualmente non implementato (gestito da allarme-core). Da valutare se aggiungere `Entry_Delay/timeout` quando `allarme_core_stato` passa a `triggered` dopo arming.
- **Notifiche push:** nessuna notifica mobile per allarmi critici dalla tastiera (rapina, medical, fire). Da valutare integrazione con sistema notifiche HA.
- **Terza tastiera:** procedura documentata; verificare comportamento `sync_all` con 3+ tastiere e delay cumulativi.
- **Autenticazione senza PIN per sera:** comportamento `require_code_to_arm=off` con profilo sera non testato sistematicamente.
