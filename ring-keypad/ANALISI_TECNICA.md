# Ring Keypad V2 — Analisi Tecnica

> Ultima revisione: 2026-04-22
> Versione architettura: 5.13

---

## Sezione 0 — Panoramica

Ring Keypad V2 via **Z-Wave / zwave2mqtt**. Fornisce:
- Inserimento/disinserimento PIN
- Feedback visivo (LED) e sonoro
- Multi-tastiera senza duplicare codice
- Integrazione con qualsiasi sistema allarme esterno
- Configurazione completa da UI, senza `secrets.yaml`

**Autonomo e indipendente** — nessuna logica allarme propria. Traduce comandi fisici in stati HA e viceversa.

---

## Sezione 1 — Struttura File

```
ring-keypad/
├── packages/
│   ├── ring_keypad_globals.yaml        # Helper globali: stato, utenti, PIN, config
│   ├── ring_keypad_scripts.yaml        # Script parametrici (accettano keypad_id)
│   ├── ring_keypad_automations.yaml    # Automazioni con wildcard MQTT
│   ├── ring_keypad_keypads.yaml        # Sensori MQTT fisici per tastiera
│   ├── ring_keypad_integration.yaml   # Bridge bidirezionale verso sistemi esterni
│   └── ring_keypad_log.yaml            # Log: logbook, timeline, note
├── var/ring_keypad.yaml                # Var timeline
└── ANALISI_TECNICA.md
```

**Mappatura file → entità:**

| File | Entità |
|------|--------|
| `ring_keypad_globals.yaml` | `input_select`, `input_text`, `input_number`, `input_boolean` |
| `ring_keypad_scripts.yaml` | `script.*` |
| `ring_keypad_automations.yaml` | `automation.*` (trigger MQTT) |
| `ring_keypad_keypads.yaml` | `mqtt.sensor`, `mqtt.binary_sensor` per tastiera |
| `ring_keypad_integration.yaml` | `automation.*` (bridge esterno) |
| `ring_keypad_log.yaml` | `input_text` logbook, `script.*` log, `automation.*` log, template sensor timeline |
| `var/ring_keypad.yaml` | `var.ring_keypad_timeline_md/critici/operazioni/note` |

---

## Sezione 2 — Protocollo Z-Wave / MQTT

### Entry Control CC (111) — Input tastiera

Topic: `zwave2mqtt/{keypad_id}/unknownClass_111/endpoint_0/{event_type}/{data_type}`

| event_type | data_type | Significato |
|------------|-----------|-------------|
| 3 | 2 | Disarm + codice |
| 6 | 2 | Arm Home + codice |
| 6 | 0 | Arm Home senza codice |
| 5 | 2 | Arm Away + codice |
| 5 | 0 | Arm Away senza codice |
| 2 | 2 | V (Enter) + codice → arma sera (valida PIN) |
| 2 | 0 | V (Enter) senza codice → arma sera se require_code=off |
| 25 | 2 | X (Cancel) + codice |
| 25 | 0 | X (Cancel) senza codice |
| 17 | 0 | Anti-rapina Police hold 3s |
| 19 | 0 | Medical hold 3s |
| 16 | 0 | Fire hold 3s |

Payload: `{"time":..., "value":"1234"}` con codice; `{"time":...}` senza.

### Indicator CC — Output verso tastiera

Topic: `zwave2mqtt/{keypad_id}/indicator/endpoint_0/{Name}/{sub_cmd}/set`

| sub_cmd | Significato | Payload |
|---------|-------------|---------|
| 1 | On/Off indicatore | 0 (off) o 99 (on) |
| 9 | Livello suono/LED | 1 (silenzioso) o 99 (con suono) |
| timeout | Countdown | stringa `"XmYs"` |

**Indicatori:**

| Indicatore | sub_cmd | Payload | Descrizione |
|------------|---------|---------|-------------|
| `Not_armed_-_disarmed` | 9 | 1/99 voice-aware | Disarmato |
| `Armed_Stay` | 9 | 1/99 voice-aware | Armato home |
| `Armed_Away` | 9 | 1/99 voice-aware | Armato away |
| `Code_not_accepted` | 9 | 99 fisso | Codice errato |
| `Bypass_challenge` | 1 | 99 fisso | Richiesta bypass |
| `Exit_Delay` | timeout | `"Xm Ys"` | Countdown uscita |
| `Entry_Delay` | timeout | `"Xm Ys"` | Countdown entrata |
| `Alarming` | 9 | 99 fisso | Intrusione CON suono |
| `Alarming_Burglar` | 9 | 1 fisso | Anti-rapina SILENZIOSO |
| `Alarming_Smoke_-_Fire` | 9 | 99 fisso | Allarme fuoco |
| `Alarming_Medical` | 9 | 99 fisso | Allarme medico |
| `Alarming_Water_leak` | 9 | 99 fisso | Perdita acqua |

### LED Display Configuration

Topic: `zwave2mqtt/{keypad_id}/configuration/endpoint_0/System_Security_Mode_Display/set`

| Payload | Descrizione |
|---------|-------------|
| `0` | LED spenti |
| `1` | Fast blink ~1s (armato sera) |
| `3` | Lampeggio 3s |
| `601` | Sempre accesi |

### Status nodo

- Topic: `zwave2mqtt/{keypad_id}/status`
- `value_template: "{{ value_json.status }}"` → `"Alive"` o `"Dead"`

### Batteria

- Topic: `zwave2mqtt/{keypad_id}/battery/endpoint_0/level`
- `value_template: "{{ value_json.value }}"` — 0-100%

---

## Sezione 3 — Multi-Tastiera (Wildcard MQTT)

### 3.1 Ricezione

Wildcard `+` cattura qualsiasi tastiera:
```
zwave2mqtt/+/unknownClass_111/endpoint_0/3/2
```
```yaml
variables:
  keypad_id: "{{ trigger.topic.split('/')[1] }}"
```

### 3.2 Trasmissione

Script costruiscono topic dinamicamente da `keypad_id`:
```yaml
topic: "zwave2mqtt/{{ keypad_id }}/indicator/endpoint_0/Not_armed_-_disarmed/9/set"
```

### 3.3 Registro tastiere attive

`input_text.ring_keypad_active_keypads` (CSV).

**Tastiere attive (2026-04-19):**
```
tastiera_allarme_camera,tastiera_allarme_sala
```

Usato da `sync_all` e `broadcast_alarm` per iterare su tutte.

### 3.4 Aggiungere nuova tastiera

1. UI HA — aggiungi topic a `ring_keypad_active_keypads` (CSV)
2. `ring_keypad_keypads.yaml` — blocchi sensor batteria + binary_sensor online con `default_entity_id` esplicito. Pattern: `sensor.tastiera_<stanza>_allarme_batteria` / `binary_sensor.tastiera_<stanza>_allarme_online`
3. `plancia_controllo.yaml` — tile Online + Batteria nella sezione "Stato Hardware"
4. Riavviare HA
5. Nessuna altra modifica — wildcard + `_all` scripts gestiscono tutto

---

## Sezione 4 — Entità Helper

### `input_select`

| Entità | Valori possibili | Descrizione |
|--------|-----------------|-------------|
| `input_select.ring_keypad_alarm_state` | disarmed, armed_home, armed_away, armed_sera, arming, triggered_burglar, triggered_burglar_sera, triggered_rapina, triggered_fire, triggered_medical, triggered_water | Stato interno keypad (unica fonte verità) |
| `input_select.ring_keypad_led_mode` | Spenti, Lampeggio (3s), Sempre accesi | Controlla System_Security_Mode_Display |

### `input_text`

| Entità | Descrizione | Note |
|--------|-------------|------|
| `ring_keypad_active_keypads` | Topic base tastiere (CSV) | Modifica dalla UI |
| `ring_keypad_zwave_entity_ids` | JSON `{"topic_base": "entity_id_zwave_js"}` — usato da `ping_verify` | RK-59 |
| `ring_keypad_last_user` | Ultimo utente | Log |
| `ring_keypad_last_event` | Ultimo evento | Log |
| `ring_keypad_last_keypad` | Topic base ultima tastiera | Log |
| `ring_keypad_arming_target_mode` | Target mode (armed_home/away) salvato all'inizio arming | Per integration: profilo allarme-core |
| `ring_keypad_bypass_mode` | Mode del bypass pendente | Impostato da `arm_or_challenge`; reset da `bypass_cancel` |
| `ring_keypad_bypass_keypad` | Topic base tastiera che avviato bypass | Impostato da `arm_or_challenge`; reset da `bypass_cancel` |
| `ring_keypad_pin_master` | PIN Master | mode: password; pattern `^$\|^[0-9]{4,8}$` |
| `ring_keypad_pin_user1..3` | PIN Utenti 1-3 | mode: password; pattern `^$\|^[0-9]{4,8}$` |
| `ring_keypad_name_master` | Nome Master | Log |
| `ring_keypad_name_user1..3` | Nomi 1-3 | Log |

### `input_number`

| Entità | Default | Descrizione |
|--------|---------|-------------|
| `ring_keypad_bypass_timeout` | 5-60s | Timeout conferma bypass |
| `ring_keypad_exit_delay` | 30s | Countdown uscita (sincronizzato da allarme_core_arming_delay) |
| `ring_keypad_siren_volume` | 5 | Volume sirena (1-10), via `Siren_Volume` CC |
| `ring_keypad_failed_attempts` | 0 (`initial: 0` accettato — tecnico interno) | Contatore PIN falliti |
| `ring_keypad_max_failed_attempts` | 5 | Soglia lockout (3-10) |

### `input_boolean`

| Entità | Default | Descrizione |
|--------|---------|-------------|
| `ring_keypad_debug` | false | Debug mode — blocca propagazione a sistemi core |
| `ring_keypad_require_code_to_arm` | false | PIN obbligatorio per armare |
| `ring_keypad_bypass_pending` | false | Flag bypass attivo |
| `ring_keypad_voice_feedback` | true | on=payload 99 (LED+suono); off=payload 1 (solo LED) per disarmed/armed |
| `ring_keypad_lockout_active` | false (`initial: false` — reset garantito al riavvio, fix C-7) | Lockout brute-force; reset auto dopo 5 min |

### Template binary_sensor (in `ring_keypad_globals.yaml`)

| Entità | Descrizione |
|--------|-------------|
| `binary_sensor.keypad_collisione_pin` | On se 2+ utenti hanno stesso PIN (≥4 cifre) — fix C-4 |

### Sensori MQTT (in `ring_keypad_keypads.yaml`)

| Entità | Tipo | Descrizione |
|--------|------|-------------|
| `sensor.tastiera_<id>_batteria` | sensor | Batteria 0-100% |
| `binary_sensor.tastiera_<id>_online` | binary_sensor | Connettività Z-Wave |

**Tastiere attive:**
- `tastiera_allarme_camera` (nodeId 5) → `sensor.tastiera_camera_allarme_batteria`, `binary_sensor.tastiera_camera_allarme_online`
- `tastiera_allarme_sala` → `sensor.tastiera_sala_allarme_batteria`, `binary_sensor.tastiera_sala_allarme_online`

---

## Sezione 5 — Gestione PIN e Sicurezza

PIN in `input_text` con `mode: password`. Non cifrati (file `.storage/core.restore_state`). Nessun `secrets.yaml`.

**Validazione:**
```jinja2
{% set ns = namespace(valid=false) %}
{% for pin in [pin_master, pin_user1, pin_user2, pin_user3] %}
  {% if pin | length >= 4 and pin == code %}
    {% set ns.valid = true %}
  {% endif %}
{% endfor %}
{{ ns.valid }}
```
PIN < 4 chars ignorati. PIN vuoto non fa mai match.

`ring_keypad_require_code_to_arm=on`: tasti Arm senza codice → `code_rejected`.

---

## Sezione 6 — Script

### Script indicazione (hardware)

| Script | Parametri | Azione MQTT |
|--------|-----------|-------------|
| `ring_keypad_send_indicator` | keypad_id, indicator, sub_cmd, value | Payload generico |
| `ring_keypad_show_disarmed` | keypad_id | `Not_armed_-_disarmed/9` — voice-aware |
| `ring_keypad_show_armed_home` | keypad_id | `Armed_Stay/9` — voice-aware |
| `ring_keypad_show_armed_away` | keypad_id | `Armed_Away/9` — voice-aware |
| `ring_keypad_show_armed_sera` | keypad_id | `Armed_Stay/9` payload 1 + `System_Security_Mode_Display` payload 1 — silenzioso + fast blink (~1s); bypassa `ring_keypad_led_mode` |
| `ring_keypad_code_rejected` | keypad_id | `Code_not_accepted/9` payload 99 |
| `ring_keypad_bypass_challenge` | keypad_id | `Bypass_challenge/1` payload 99 + avvia `bypass_keepalive` |
| `ring_keypad_show_exit_delay` | keypad_id | `Exit_Delay/9/set` volume (1 se armed_home, 99 altrimenti) + `Exit_Delay/timeout` stringa `Xm Ys` |
| `ring_keypad_alarm_burglar` | keypad_id | `Alarming/9` payload 99 — CON suono |
| `ring_keypad_alarm_burglar_sera` | keypad_id | `Alarming/9` payload 1 — SILENZIOSO |
| `ring_keypad_alarm_rapina` | keypad_id | `Alarming_Burglar/9` payload 1 — SILENZIOSO |
| `ring_keypad_alarm_fire` | keypad_id | `Alarming_Smoke_-_Fire/9` payload 99 |
| `ring_keypad_alarm_medical` | keypad_id | `Alarming_Medical/9` payload 99 |
| `ring_keypad_alarm_water` | keypad_id | `Alarming_Water_leak/9` payload 99 |

### Script LED

| Script | Parametri | Descrizione |
|--------|-----------|-------------|
| `ring_keypad_set_led_mode` | keypad_id | Pubblica LED da `ring_keypad_led_mode` su tastiera singola |
| `ring_keypad_set_led_mode_all` | — | Itera tastiere → `set_led_mode` |

### Script volume sirena

| Script | Parametri | Descrizione |
|--------|-----------|-------------|
| `ring_keypad_set_siren_volume` | keypad_id | Pubblica `ring_keypad_siren_volume` su `configuration/endpoint_0/Siren_Volume/set` |
| `ring_keypad_set_siren_volume_all` | — | Itera tastiere → `set_siren_volume` |

### Script sincronizzazione

| Script | Parametri | Descrizione |
|--------|-----------|-------------|
| `ring_keypad_sync_state` | keypad_id | Dispatcher: prima `set_led_mode`, poi indicatore corretto per `ring_keypad_alarm_state` (11 stati) |
| `ring_keypad_sync_all` | — | `sync_state` su tutte le tastiere; pre-check disponibilità nodi (RK-63) |
| `ring_keypad_broadcast_alarm` | alarm_type | Allarme specifico a tutte le tastiere (burglar/rapina/fire/medical/water); pre-check + post-ping (RK-64) |

### Script logica

| Script | Parametri | Descrizione |
|--------|-----------|-------------|
| `ring_keypad_resolve_user` | code | Identifica utente dal PIN → `ring_keypad_last_user` |
| `ring_keypad_validate_and_disarm` | code, keypad_id | Valida PIN → disarma o rifiuta |
| `ring_keypad_validate_and_arm` | code, mode, keypad_id | **Puro PIN→arm**: PIN valido → `resolve_user` + `arm_or_challenge`; errato → `code_rejected` |
| `ring_keypad_arm_or_challenge` | mode, keypad_id | Controlla sensori aperti zone target: pulito → `do_arm`; aperti → `bypass_pending` + `bypass_challenge` + `bypass_timeout_script` |
| `ring_keypad_bypass_confirm` | keypad_id | `stored_mode`←`bypass_mode`; turn_off `do_arm`; `bypass_cancel`; `do_arm(stored_mode)` — senza PIN |
| `ring_keypad_bypass_keepalive` | keypad_id | `mode: restart` — ogni 5s invia `Bypass_challenge/9` payload 1 (solo LED) finché `bypass_pending=on AND bypass_keypad==keypad_id` |
| `ring_keypad_bypass_cancel` | — | Reset `bypass_pending`, `bypass_mode`, `bypass_keypad`; ferma `bypass_timeout_script` e `bypass_keepalive` |
| `ring_keypad_bypass_timeout_script` | — | `mode: restart` — dopo timeout: **prima** inline-reset bypass, **poi** `sync_state` (ordine critico: evita race RK-45) |
| `ring_keypad_do_disarm` | keypad_id | **Prima** `bypass_cancel`; poi ferma `do_arm` (RK-30); stato=disarmed + log |
| `ring_keypad_do_arm` | mode, keypad_id | Stato=arming → delay exit → stato=mode + log; `mode: restart` (fix C-3) |
| `ring_keypad_ping_verify` | keypad_id, manual | 2s delay → `zwave_js.ping` → controlla success → log fail (o ok se manual=true) |
| `ring_keypad_verify_all` | manual | `mode: restart` — itera tastiere sequenzialmente → `ping_verify` |

---

## Sezione 7 — Automazioni

### Automazioni input (da tastiera)

| ID | Trigger MQTT | Azione |
|----|-------------|--------|
| `ring_keypad_input_disarm` | `zwave2mqtt/+/.../3/2` | `validate_and_disarm` |
| `ring_keypad_input_arm_away` | `.../5/2` | Se bypass_pending → `code_rejected`; altrimenti `validate_and_arm mode=armed_away` |
| `ring_keypad_input_arm_away_nocode` | `.../5/0` | Se bypass_pending → `code_rejected`; se require_code=off → `arm_or_challenge`; altrimenti `code_rejected` |
| `ring_keypad_input_arm_home` | `.../6/2` | Se bypass_pending → `code_rejected`; altrimenti `validate_and_arm mode=armed_home` |
| `ring_keypad_input_arm_home_nocode` | `.../6/0` | Se bypass_pending → `code_rejected`; se require_code=off → `arm_or_challenge`; altrimenti `code_rejected` |
| `ring_keypad_input_enter` | `.../2/2` | Se bypass_pending → `bypass_confirm`; altrimenti `validate_and_arm mode=armed_sera` |
| `ring_keypad_input_enter_nocode` | `.../2/0` | Se bypass_pending → `bypass_confirm`; se require_code=off → `arm_or_challenge` (con doppia guard RK-52); altrimenti `code_rejected` |
| `ring_keypad_input_cancel` | `25/2` + `25/0` | Se bypass_pending per tastiera → `bypass_cancel` + `sync_state`; altrimenti log "Tasto X" |
| `ring_keypad_input_rapina` | `.../17/0` | Stato=triggered_rapina + broadcast rapina |
| `ring_keypad_input_medical` | `.../19/0` | Stato=triggered_medical + broadcast |
| `ring_keypad_input_fire` | `.../16/0` | Stato=triggered_fire + broadcast |

### Automazioni di sistema

| ID | Trigger | Azione |
|----|---------|--------|
| `ring_keypad_sync_on_state_change` | State: `ring_keypad_alarm_state` | `sync_all` |
| `ring_keypad_restore_on_connect` | MQTT `zwave2mqtt/+/status` = Alive | `set_siren_volume` + `sync_state` |
| `ring_keypad_sync_on_ha_start` | HA start | Delay 15s → safety-net lockout reset (C-8) → `set_siren_volume_all` + `sync_all` |
| `ring_keypad_sync_led_on_mode_change` | State: `ring_keypad_led_mode` | `set_led_mode_all` **solo se** `alarm_state != 'armed_sera'` |
| `ring_keypad_sync_siren_volume_on_change` | State: `ring_keypad_siren_volume` | `set_siren_volume_all` |
| `ring_keypad_lockout_reset` | State: `lockout_active` = on | Delay 5 min in action → se ancora on: disattiva + azzera failed_attempts + log; `mode: restart` (fix C-9) |
| `ring_keypad_failed_attempts_reset` | State: `failed_attempts` invariato 60s (+ lockout off + valore > 0) | Azzera contatore |

### Automazioni integrazione (`ring_keypad_integration.yaml`)

| ID | Trigger | Condizione | Azione |
|----|---------|------------|--------|
| `ring_keypad_sync_from_allarme_core` | State: `allarme_core_stato` o `allarme_core_profilo` | stato==armed per trigger profilo | Aggiorna `ring_keypad_alarm_state` profilo-aware |
| `ring_keypad_command_to_allarme_core` | State: `alarm_state` → disarmed/arming/armed_home/armed_away/armed_sera | **debug OFF** + anti-loop | Imposta profilo + arm/disarm allarme-core |
| `ring_keypad_sync_exit_delay` | State: `allarme_core_arming_delay` + HA start | — | Copia in `ring_keypad_exit_delay` |
| `ring_keypad_sync_from_safety_core` | State: `safety_core_stato` | — | Aggiorna `alarm_state` (fire/water/reset) |
| `ring_keypad_command_to_safety_core` | State: `alarm_state` → triggered_fire | **debug OFF** + safety_core non già triggered | safety_core triggered + log manuale |

> **Debug mode on:** `command_to_allarme_core` e `command_to_safety_core` bloccate. Le `sync_from_*` continuano normalmente.

---

## Sezione 8 — Flusso Operativo

### Inserimento da tastiera

```
[ARM AWAY] + PIN
    ↓ MQTT 5/2
ring_keypad_validate_and_arm (code, mode="armed_away", keypad_id)
    ├─ PIN valido → resolve_user → arm_or_challenge
    │       ├─ no sensori aperti → do_arm
    │       │     → alarm_state="arming" → sync_all → show_exit_delay tutte tastiere
    │       │     → (parallelo) ring_keypad_command_to_allarme_core chiama arm_allarme_core
    │       │     → delay exit_delay → alarm_state="armed_away" → sync_all → show_armed_away
    │       │
    │       └─ sensori aperti → bypass_pending=on; bypass_mode=mode; bypass_keypad=id
    │                → bypass_challenge (feedback tastiera)
    │                → bypass_timeout_script (timer software)
    │                    ↓ Utente preme V → bypass_confirm → stored_mode; do_arm
    │                    ↓ Utente preme X → bypass_cancel + sync_state (LED ripristinato)
    │                    ↓ Timeout → inline-reset bypass → sync_state
    │
    └─ PIN errato → code_rejected + log (N/M tentativi)
```

### Disinserimento da tastiera

```
[DISARM] + PIN
    ↓ MQTT 3/2
validate_and_disarm
    ├─ PIN valido → do_disarm → alarm_state="disarmed" → tutte tastiere LED disarmed
    │                          → script.disarm_allarme_core
    └─ PIN errato → code_rejected
```

### Allarme da sistema esterno → tastiere

```
allarme_core_stato="triggered"
    → sync_from_allarme_core → alarm_state="triggered_burglar"
    → sync_on_state_change → sync_all → alarm_burglar su OGNI tastiera
```

---

## Sezione 9 — Integrazione Sistemi Esterni

Ring-keypad **disaccoppiato** dal sistema allarme. Integration legge/scrive tramite `ring_keypad_integration.yaml`.

**Anti-loop bidirezionale:**
```
allarme-core cambia → sync_from aggiorna alarm_state → command_to attivato
    → CONTROLLO: allarme-core già nello stato corretto?
        ├─ SI → nessun comando (loop interrotto)
        └─ NO → comando inviato (caso reale da tastiera)
```

### Integrazione allarme-core

**Mappatura stati → keypad:**

| `allarme_core_stato` | `allarme_core_profilo` | `ring_keypad_alarm_state` |
|----------------------|------------------------|---------------------------|
| `disarmed` | qualsiasi | `disarmed` |
| `arming` | qualsiasi | `arming` |
| `armed` | `notte` | `armed_home` |
| `armed` | `giorno` | `armed_away` |
| `armed` | `tutti` | `armed_away` |
| `armed` | `sera` | `armed_sera` |
| `triggered` | `sera` | `triggered_burglar_sera` |
| `triggered` | altri | `triggered_burglar` |

**Comandi keypad → allarme-core:**

| `ring_keypad_alarm_state` | Profilo | Script | Note |
|---------------------------|---------|--------|------|
| `disarmed` | — | `disarm_allarme_core` | — |
| `arming` | da `arming_target_mode` | `arm_allarme_core` | fire-and-forget (turn_on) |
| `armed_home` | `notte` | `arm_allarme_core` | Safety fallback |
| `armed_away` | `giorno` | `arm_allarme_core` | Safety fallback |

Anti-loop arming: invia solo se `stato not in ['arming','armed']`
Anti-loop armed_home: invia solo se `stato != 'arming' AND (stato != 'armed' OR profilo != 'notte')`
Anti-loop armed_away: invia solo se `stato != 'arming' AND (stato != 'armed' OR profilo not in ['giorno','tutti'])`

`ring_keypad_exit_delay` = `allarme_core_arming_delay` — sincronizzati da `ring_keypad_sync_exit_delay`.

### Integrazione safety-core

**safety-core → keypad:**

| `safety_core_stato` | `safety_core_ultima_categoria` | `alarm_state` |
|---------------------|-------------------------------|---------------|
| `triggered` | fumo/gas/carbonio | `triggered_fire` |
| `triggered` | acqua | `triggered_water` |
| `ok` (reset) | qualsiasi | re-sync da allarme-core |

**keypad → safety-core (Fire hold 3s → triggered_fire):**
- `safety_core_stato=triggered`, `safety_core_ultima_categoria=fumo`
- Log: `"🔥 Allarme fuoco manuale da tastiera Ring Keypad"`
- Anti-loop: condizione `safety_core_stato != 'triggered'`

### Integrazione alarm_control_panel generico (commentata, sezione 2)

Disponibile in `ring_keypad_integration.yaml`. Decommentare sezione 2, commentare sezione 1.

---

## Sezione 10 — Installazione

**Requisiti:** HA 2022.3+, zwave2mqtt configurato, MQTT attivo in HA

**Passi:**
1. Copiare `packages/*.yaml` in `config/packages/ring_keypad/`
2. `configuration.yaml`: `homeassistant.packages: !include_dir_named packages`
3. Riavviare HA
4. UI HA: configurare `ring_keypad_active_keypads`, PIN master/user1-3, nomi

**Topic base** = nome nodo in zwave2mqtt.

---

## Sezione 11 — Note Architetturali

**Perché no secrets.yaml:** `input_text mode:password` → modifica PIN senza riavvio.

**Perché wildcard MQTT:** N tastiere = 10 automazioni fisse (non N×10).

**Mappatura profilo → tastiera:**

| Profilo | Tasto tastiera | LED |
|---------|----------------|-----|
| `notte` | ARM HOME (6/2, 6/0) | Armed Stay |
| `giorno` | ARM AWAY (5/2, 5/0) | Armed Away |
| `sera` | V/Enter (2/2, 2/0) | Armed Stay silenzioso + fast blink |
| `tutti` | — (da allarme-core) | Armed Away |

**Doppio delay e bypass:** tastiera e allarme-core armano in parallelo. `ring_keypad_command_to_allarme_core` chiama `arm_allarme_core` all'inizio di `arming`, non alla fine. `exit_delay` e `arming_delay` sincronizzati → armed simultaneo, nessun doppio countdown.

---

## Sezione 12 — Bug e Limitazioni

| # | Descrizione | Stato | Fix |
|---|-------------|-------|-----|
| RK-01 | armed_home/away non distinguevano profilo → sempre profilo precedente | **2026-04-13** | Integration: notte→armed_home, giorno→armed_away |
| RK-02 | Delay in `do_arm` non interrompibile | Aperto | Limitazione HA scripts |
| RK-04 | `sync_on_state_change` si attiva per transizioni →unknown | Accettato | sync_all idempotente |
| RK-05 | `restore_on_connect` `value_template` rimosso → trigger mai scattato | **2026-04-11** | Ripristinato `value_template` |
| RK-06 | `code_rejected` payload "1" non-standard | **2026-04-11** | Payload "99" |
| RK-07 | `bypass_challenge` payload "1" non-standard | **2026-04-11** | Payload "99" |
| RK-08 | `tastiera_camera_online` mancava `value_template` → sempre off | **2026-04-11** | Aggiunto `value_template` |
| RK-09 | Battery topic errato `battery/0/isLow` (bool) | **2026-04-11** | Topic `battery/endpoint_0/level` + value_template |
| RK-10 | Nome nodo errato `tastiera_camera` invece di `tastiera_allarme_camera` | **2026-04-11** | Aggiornato in tutti i file |
| RK-11 | Entity ID duplicati con `_2` da zwave2mqtt discovery | **2026-04-11** | `default_entity_id` espliciti (fix in RK-57) |
| RK-12 | Dashboard: `header.card` (singolo) invece di `header.cards` | **2026-04-11** | Cambiato a `cards:` |
| RK-13 | `condition: template` con block scalar `>` → `"True\n"` invece di `"true"` | **2026-04-11** | Inline con `| string | lower` |
| RK-14 | Topic Entry CC errati: `111/0/` invece di `unknownClass_111/endpoint_0/` | **2026-04-12** | Topic corretti |
| RK-15 | Topic Indicator CC errati: `135/0/` invece di `indicator/endpoint_0/` | **2026-04-12** | Topic corretti |
| RK-16 | Exit/Entry Delay: payload int in secondi invece di stringa `XmYs` | **2026-04-12** | Calcolo `(s//60)m(s%60)s` |
| RK-17 | `alarm_burglar` puntava a `Alarming_Burglar` (silenzioso) invece di `Alarming` | **2026-04-12** | alarm_burglar→Alarming/99; nuovo alarm_rapina→Alarming_Burglar/1 |
| RK-18 | `input_cancel` ascoltava solo `25/2` (X+code) invece di anche `25/0` | **2026-04-12** | Topic aggiornato a `25/2` (con codice) |
| RK-19 | `input_police` rinominata `input_rapina`, stato `triggered_rapina` | **2026-04-12** | Distinzione burglar vs rapina |
| RK-20 | Doppio countdown: `do_arm` delay + `arm_allarme_core` delay in sequenza | **2026-04-13** | Integration chiama arm_allarme_core all'inizio di arming; parallelo |
| RK-21 | `sync_from_allarme_core` mappava armed sempre a `armed_away` | **2026-04-13** | Mappatura profilo-aware |
| RK-22 | Non reagiva a cambi `allarme_core_profilo` mentre armato | **2026-04-13** | Trigger su profilo con condizione stato==armed |
| RK-23 | `exit_delay` e `arming_delay` indipendenti | **2026-04-13** | Automazione `sync_exit_delay` |
| RK-24 | Anti-loop armed_away non considerava profilo `tutti` | **2026-04-13** | Condizione `not in ['giorno','tutti']` |
| RK-25 | `sync_from_allarme_core` senza `mode:` → doppio trigger poteva scartare | **2026-04-13** | `mode: queued, max: 5` |
| RK-26 | `| float`/`| int` senza default → 0 se unknown al boot | **2026-04-13** | Default: `| float(20)`, `| int(30)` |
| RK-27 | Safety-core non integrato | **2026-04-13** | Sezione 3: `sync_from_safety_core` + `command_to_safety_core` |
| RK-28 | Integration chiamava `arm_allarme_core_immediate` saltando stato arming → bypass non calcolato | **2026-04-13** | Chiama `arm_allarme_core` normale; `arm_immediate` eliminato |
| RK-29 | Exit delay countdown non partiva: `| int` senza default + race condition sync chain | **2026-04-13** | `do_arm` chiama `show_exit_delay` direttamente; `| int(30)`; sync_from blocca verso armed se do_arm in esecuzione |
| RK-30 | Disarmo durante exit delay: `do_disarm` non fermava `do_arm` | **2026-04-13** | `script.turn_off ring_keypad_do_arm` come primo passo di `do_disarm` |
| RK-31 | Disarmo durante exit delay non disarmava allarme-core: 3 bug sovrapposti (mode:single, anti-loop race, action bloccante) | **2026-04-13** | mode:queued; anti-loop su trigger.from_state; tutti arm_allarme_core → script.turn_on (fire-and-forget) |
| RK-32 | Exit/Entry Delay payload senza virgolette JSON → zwave2mqtt non lo accettava | **2026-04-13** | Payload `'"XmYs"'` (singole+doppie quotes) |
| RK-33 | Debug mode non bloccava propagazione keypad→core | **2026-04-13** | `condition: state ring_keypad_debug off` nelle command_to_* |
| RK-34 | `ring_keypad_exit_delay` editabile in dashboard ma è read-only (sync automatico) | **2026-04-13** | Rimosso `numeric-input`; stile opacizzato; rimosso da popup config |
| RK-35 | Allarmi sicurezza vitale (fire/medical/water) con payload voice-aware → potevano essere silenziosi | **2026-04-13** | Payload fisso 99 per tutti e 4; voice_feedback solo per stati normali |
| RK-36 | `sync_from_allarme_core` ramo disarmed non fermava `do_arm` | **2026-04-13** | `script.turn_off ring_keypad_do_arm` nel ramo disarmed |
| RK-37 | Nuove tastiere in keypads.yaml senza `object_id` → conflitti con discovery | **2026-04-13** | Template con `object_id`/`default_entity_id` |
| RK-38 | Profilo `sera` armato mostrava tastiera come disarmed | **2026-04-14** | Stato `armed_sera`: Armed_Stay silenzioso + fast blink LED |
| RK-39 | Cambio LED mode sovrascriveva fast blink di armed_sera | **2026-04-14** | `sync_led_on_mode_change` bloccata se `alarm_state == 'armed_sera'` |
| RK-40 | `restore_on_connect` + `sync_on_ha_start`: `set_led_mode` separato sovrascriveva fast blink | **2026-04-14** | `sync_state` chiama `set_led_mode` internamente; rimossa chiamata separata |
| RK-41 | Tasto V non armava profilo sera | **2026-04-14** | `input_enter` → `validate_and_arm mode=armed_sera`; integration estesa con caso armed_sera |
| RK-42 | Nessun controllo sensori aperti prima dell'armo da tastiera | **2026-04-14** | Script `arm_or_challenge`: controlla binary_sensor zone target; bypass challenge con timer |
| RK-43 | Tasti ARM nocode chiamavano `do_arm` direttamente senza `arm_or_challenge` | **2026-04-14** | 3 automazioni nocode ora chiamano `arm_or_challenge` |
| RK-44 | `bypass_pending` non resettato al disarmo da dashboard/esterno | **2026-04-14** | `bypass_cancel` nel ramo disarmed di `sync_from_allarme_core` |
| RK-45 | **CRITICO** — `bypass_timeout_script` chiamava `sync_state` prima di resettare bypass → race window | **2026-04-14** | Inline-reset **prima** di `sync_state` |
| RK-46 | **CRITICO** — ARM HOME/AWAY + PIN valido durante bypass → secondo challenge sovrapposto | **2026-04-14** | `validate_and_arm` a 3 casi: CASO 2a → `code_rejected` se bypass attivo |
| RK-47 | **CRITICO** — `do_arm` `mode:single` → conferma bypass scartata silenziosamente se do_arm in esecuzione | **2026-04-14** | `turn_off ring_keypad_do_arm` prima della nuova chiamata in CASO 1 |
| RK-48 | Commento errato in integration (legacy `sera→disarmed`) | **2026-04-14** | Commento corretto |
| RK-49 | **MEDIO** — Allarme safety durante exit delay: `do_arm` completava sovrascrivendo triggered_fire/water | **2026-04-14** | `turn_off ring_keypad_do_arm` nei rami triggered di `sync_from_safety_core` |
| RK-50 | **UX** — Logica bypass accoppiata a `validate_and_arm` | **2026-04-14** | `bypass_confirm` script separato; `validate_and_arm` puro PIN→arm |
| RK-51 | **MINOR** — Guard bypass in ARM HOME/AWAY: `code_rejected` senza aggiornamento `last_event` | **2026-04-14** | `ring_keypad_last_event` aggiornato in tutti e 4 i branch bypass |
| RK-52 | **CRITICO** — Bypass loop con require_code=off: `bypass_confirm` resettava pending PRIMA che `do_arm` impostasse arming → messaggio 2/0 duplicato riavviava bypass | **2026-04-15** | Doppia guard in `input_enter_nocode`: not `bypass_confirm running` + `alarm_state==disarmed` |
| RK-53 | **MEDIO** — `Timeout_Display_on_Status_Change` non configurabile via MQTT | **2026-04-15** | Rimosso publish; finestra gestita da `bypass_timeout_script` software (range aggiornato 5-60s) |
| RK-54 | **UX** — Indicatore bypass scompare dopo ~5s → nessun feedback visivo | **2026-04-15** | `bypass_keepalive` (`mode:restart`): ogni 5s sub_cmd 9 payload 1 (solo LED, no voce) finché pending=on |
| RK-55 | **BUG** — Tasto X senza codice (25/0) non cancellava bypass | **2026-04-15** | Aggiunto trigger `25/0` a `input_cancel` |
| RK-56 | **BUG** — Doppio comando MQTT Exit_Delay: `do_arm` impostava arming (→ sync) + chiamava show_exit_delay direttamente | **2026-04-15** | Rimossa chiamata diretta; sync_on_state_change → sync_all → show_exit_delay su TUTTE |
| RK-57 | **DEPRECAZIONE** — `object_id` deprecato in HA 2024.x → warning al boot | **2026-04-20** | Sostituito con `default_entity_id: <dominio>.<nome>` |
| RK-58 | **UX** — Latenza LED disarmed: hop extra via sync_on_state_change | **2026-04-20** | `sync_all` diretto in `do_disarm`, ramo disarmed `sync_from_allarme_core`, ramo ok `sync_from_safety_core` |
| RK-59 | **OSSERVABILITÀ** — Nessun feedback se LED non ricevuto (Z-Wave fire-and-forget) | **2026-04-20/21** | `ring_keypad_zwave_entity_ids` (JSON {topic:entity}); `ping_verify(keypad_id, manual)`; `verify_all(manual)` |
| RK-60 | **LOG** — `ping_verify` loggava OK e FAIL indistintamente → logbook pieno | **2026-04-21** | manual=false: logga solo FAIL; manual=true: logga OK e FAIL |
| RK-61 | **BUG** — `is_manual` non accessibile in template `msg`; JSON malformato → crash silenzioso | **2026-04-21** | `log_fail_msg` pre-calcolato; guard `mapping is not none`; `| trim` su `zwave_entity_id` |
| RK-62 | **BUG** — `states[zwave_entity_id].last_updated` → UndefinedError se entity non esiste; `manual` propagato come "True"/"False" fragile | **2026-04-21** | Guard `s is not none`; manual normalizzato `| bool | lower` |
| RK-63 | **OSSERVABILITÀ** — `sync_all` senza check pre-invio: log FAIL arrivavano ~5s dopo comandi | **2026-04-21** | Pre-check a 0ms in `sync_all` su ogni tastiera; log `⚠️ NODO NON DISPONIBILE prima dei comandi` |
| RK-64 | **OSSERVABILITÀ** — `broadcast_alarm` senza check disponibilità nodi | **2026-04-21** | Pre-check + post-ping fire-and-forget dopo invio comandi |
| RK-65 | **OSSERVABILITÀ** — `bypass_challenge` senza check nodo | **2026-04-21** | Pre-check + log `⚠️ NODO NON DISPONIBILE prima del bypass`; post-ping fire-and-forget |
| RK-66 | **OSSERVABILITÀ** — `bypass_timeout_script` chiamava `sync_state` senza check | **2026-04-21** | Pre-check a 0ms prima di sync_state; log `⚠️ NODO NON DISPONIBILE al ripristino LED post-bypass` |
| C-2 | **CRITICO** — `trigger.payload_json` è None se payload non JSON → `.get()` → UndefinedError | **2026-04-17** | `(trigger.payload_json \| default({})).get('value', '')` nelle 4 automazioni con codice |
| C-3 | **CRITICO** — `do_arm` `mode:single` → cambio profilo durante exit delay scartato silenziosamente | **2026-04-17** | `mode: restart` — nuovo profilo riavvia script |
| C-4 | **CRITICO** — PIN duplicati non rilevati | **2026-04-17** | `binary_sensor.keypad_collisione_pin` in globals.yaml |
| C-5 | **CRITICO** — PIN < 4 cifre ignorati silenziosamente | **2026-04-17** | Pattern `^$\|^[0-9]{4,8}$` su tutti i 4 PIN |
| C-6 | **CRITICO** — Nessuna protezione brute-force (RK-03) | **2026-04-17** | failed_attempts + max_failed_attempts + lockout_active; lockout_reset 5min; sliding window 60s |
| C-7 | **CRITICO** — `lockout_active` senza `initial: false` → lockout permanente dopo riavvio | **2026-04-18** | `initial: false` (entità tecnica interna, non modificata dall'utente) |
| C-8 | **DIFESA** — Safety-net lockout reset mancante in `sync_on_ha_start` | **2026-04-18** | Blocco if/then in `sync_on_ha_start` dopo delay 15s |
| C-9 | **BUG** — `lockout_reset` con `for: minutes: 5` nel trigger non scattava affidabilmente | **2026-04-18** | Trigger immediato; delay 5min in action; `mode: restart`; condizione post-delay |
| C-10 | **BUG** — Messaggi PIN errato costanti → `from_state==to_state` → log non scattava | **2026-04-19** | `new_count` incluso nel messaggio: `"PIN errato (N/M)"` → ogni tentativo unico |
| C-11 | **BUG** — LOCKOUT message → sezione Operazioni invece di Critici | **2026-04-19** | `{% elif 'LOCKOUT' in m %} 🔐` nella routing; `or '🔐' in msg` per critici; colore rosso timeline |
| RK-67 | **UX** — Exit delay sonoro anche con profilo notte (armed_home): il beep disturbava in casa | **2026-04-22** | `show_exit_delay`: pubblica `Exit_Delay/9/set = 1` (silenzioso) se `arming_target_mode==armed_home`, `99` altrimenti. Volume prima del timeout. |
| C-12 | **BUG** — Tentativi PIN falliti non loggati dopo sliding window reset: `ring_keypad_failed_attempts_reset` azzerava solo il counter; `last_event` e `logbook_ultimo_evento` conservavano i valori precedenti → prossimo tentativo produceva stessa stringa `(1/5)` → condizioni `from_state != to_state` in `log_da_last_event` e `log_timeline_append` FAIL → non loggato | **2026-04-22** | `ring_keypad_failed_attempts_reset`: dopo reset counter, svuota anche `last_event=""` e `logbook_ultimo_evento=""`. Le automazioni log ignorano valori vuoti (condizione `trim != ''`). Prossimo tentativo sempre diverso da `""` → loggato. |

---

## Sezione 13 — Sistema Log

### Architettura

`ring_keypad_log.yaml` — autonomo, zero impatto su altri 5 package.

### Entità helper

| Entità | Tipo | Descrizione |
|--------|------|-------------|
| `input_text.ring_keypad_logbook_event` | fittizia | Filter Logbook card in UI |
| `input_text.ring_keypad_logbook_ultimo_evento` | trigger | Aggiornata da `logbook_emit`; trigger timeline append |
| `input_text.ring_keypad_nota_manuale` | UI | Note manuali da dashboard |

### Script

| Script | Descrizione |
|--------|-------------|
| `ring_keypad_logbook_emit` | `msg` → logbook.log + update `ultimo_evento` |
| `ring_keypad_logbook_nota` | Prefissa `📝 Nota:` → `logbook_emit` |
| `ring_keypad_logbook_nota_da_ui` | Legge `nota_manuale` → `logbook_emit` → svuota campo |
| `ring_keypad_logbook_cancella_tutto` | Azzera 4 `var.*` timeline |

### Automazioni log

| ID | Trigger | Copertura |
|----|---------|-----------|
| `ring_keypad_log_da_last_event` | `ring_keypad_last_event` | arm, disarm, PIN errato, bypass, tasto X, RAPINA, MEDICAL, FIRE |
| `ring_keypad_log_da_alarm_state_esterno` | `alarm_state` → triggered_burglar/water | Allarmi da sistemi esterni |
| `ring_keypad_log_tastiera_online` | MQTT `zwave2mqtt/+/status` Alive | Tastiera tornata online |
| `ring_keypad_log_timeline_append` | `ring_keypad_logbook_ultimo_evento` | Append a `var.*_timeline_md` |

### Routing emoji

| Contenuto messaggio | Emoji | Sezione |
|---------------------|-------|---------|
| `LOCKOUT` | 🔐 | Critici |
| `PIN errato` | ❌ | Operazioni |
| `ANTI-RAPINA` | 🚨 | Critici |
| `MEDICAL` | 🚑 | Critici |
| `FIRE` | 🔥 | Critici |
| `triggered_burglar` | 🚨 | Critici |
| `triggered_water` | 💧 | Critici |
| `Bypass attivo` | ⚠️ | Operazioni |
| `Tasto X` / `annullamento` | ↩️ | Operazioni |
| `Disarmato` | 🔓 | Operazioni |
| `Armo ... armed_sera` | 🌙 | Operazioni |
| `Armo` | 🔒 | Operazioni |
| Tastiera online | 📡 | Operazioni |
| Nota manuale | 📝 | Note |
| Altro | ℹ️ | Operazioni |

### Var timeline (`var/ring_keypad.yaml`)

| Var | Uso |
|-----|-----|
| `var.ring_keypad_timeline_md` | Timeline globale |
| `var.ring_keypad_timeline_critici_md` | Critici (popup) |
| `var.ring_keypad_timeline_operazioni_md` | Operazioni (popup) |
| `var.ring_keypad_timeline_note_md` | Note (popup) |

Max 25 righe per var.

### Template sensor

| Sensor | `unique_id` | Descrizione |
|--------|-------------|-------------|
| `Ring Keypad – Timeline Render` | `ring_keypad_timeline_render` | Colora righe per tipo, espone `long_state` |
| `Ring Keypad – Count Critici` | `ring_keypad_count_critici` | Conta righe critici |
| `Ring Keypad – Count Operazioni` | `ring_keypad_count_operazioni` | Conta righe operazioni |
| `Ring Keypad – Count Note` | `ring_keypad_count_note` | Conta righe note |

---

## Sezione 14 — Dashboard (`plancia_controllo.yaml`)

`sections view`, path: `ring-keypad`.

### Header (banner condizionali)

| Condizione | Tipo | Messaggio |
|-----------|------|-----------|
| `ring_keypad_debug = on` | warning | Debug attivo |
| `alarm_state` in triggered_* | error | Allarme in corso + tipo |
| `bypass_pending = on` | warning | Bypass in attesa — premere V o X |
| `lockout_active = on` | error | Lockout attivo — reset dopo 5 min |
| `keypad_collisione_pin = on` | warning | Due utenti con stesso PIN |

### Sezione 1 — Controllo

| Card | Descrizione |
|------|-------------|
| Tile Stato | `ring_keypad_alarm_state` + bordo colorato |
| Mushroom Template | Stato leggibile + utente + tastiera, icona dinamica 11 stati |
| Conditional bypass | Visibile se bypass_pending=on: tastiera + profilo |
| Conditional allarme | Visibile se triggered_*: banner rosso |

### Sezione 2 — Stato Hardware

| Card | Entità |
|------|--------|
| Tile Online Camera | `binary_sensor.tastiera_camera_allarme_online` |
| Tile Batteria Camera | `sensor.tastiera_camera_allarme_batteria` |
| Tile Online Sala | `binary_sensor.tastiera_sala_allarme_online` |
| Tile Batteria Sala | `sensor.tastiera_sala_allarme_batteria` |
| Mushroom tastiere | `ring_keypad_active_keypads` — lista e conteggio |

> Aggiungere tastiera: copiare due tile (Online + Batteria) nella sezione.

### Sezione 3 — Configurazione

| Card | Entità |
|------|--------|
| Feedback vocale | `input_boolean.ring_keypad_voice_feedback` |
| Richiedi PIN armo | `input_boolean.ring_keypad_require_code_to_arm` |
| Volume Sirena | `input_number.ring_keypad_siren_volume` |
| Modalità LED | `input_select.ring_keypad_led_mode` |
| Ritardo Uscita (read-only) | `input_number.ring_keypad_exit_delay` (opacizzato, sync da allarme-core) |
| Timeout Bypass | `input_number.ring_keypad_bypass_timeout` |
| Debug Mode | `input_boolean.ring_keypad_debug` |
| Collisione PIN | `binary_sensor.keypad_collisione_pin` (warning se on) |
| Soglia Lockout | `input_number.ring_keypad_max_failed_attempts` |
| Tentativi falliti | `input_number.ring_keypad_failed_attempts` (read-only + pulsante reset) |
| Lockout Attivo | `input_boolean.ring_keypad_lockout_active` (+ pulsante reset) |

Popup "Configurazione Avanzata": PIN master/user1-3 + nomi + tastiere attive.

### Sezione 4 — Log & Attività

| Card | Descrizione |
|------|-------------|
| Pulsante Registro | Popup: 3 markdown (Critici / Operazioni / Note), max 25 righe ciascuno |
| Tile Cancella Log | `ring_keypad_logbook_cancella_tutto` |
| Tile Aggiungi Nota | Popup `ring_keypad_nota_manuale` + pulsante salva |
| `custom:timeline-card` | 24h: `ring_keypad_alarm_state`, `ring_keypad_last_user`, `ring_keypad_lockout_active` |
| Tile Ultimo Evento | `input_text.ring_keypad_logbook_ultimo_evento` |

**Colori timeline:**

| Entità | Stato | Colore |
|--------|-------|--------|
| `ring_keypad_alarm_state` | disarmed | verde |
| `ring_keypad_alarm_state` | armed_*/arming | giallo/arancio/viola/grigio |
| `ring_keypad_alarm_state` | triggered_* | rosso |
| `ring_keypad_lockout_active` | on | rosso (banda) |
