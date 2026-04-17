# Ring Keypad V2 — Analisi Tecnica

> Ultima revisione: 2026-04-17  
> Versione architettura: 5.0 (Fix critici C-2/C-3/C-4/C-5/C-6: protezione payload MQTT, mode:restart su do_arm, rilevamento PIN duplicati, validazione pattern PIN, sistema anti brute-force)

---

## Sezione 0 — Panoramica e Obiettivi

Il progetto **Ring Keypad V2** gestisce una o più tastiere fisiche Ring Keypad V2 collegate via **Z-Wave** a Home Assistant attraverso **zwave2mqtt**. Fornisce:

- Inserimento/disinserimento dell'allarme tramite PIN
- Feedback visivo (LED indicatori) e sonoro sulla tastiera
- Supporto a più tastiere contemporaneamente senza duplicare il codice
- Integrazione con qualsiasi sistema di allarme esterno (allarme-core, Alarmo, ecc.)
- Configurazione completa da UI, senza `secrets.yaml`

Il progetto è **autonomo e indipendente**: non contiene logica d'allarme propria. Agisce come **interfaccia hardware** verso un sistema di allarme esterno, traducendo i comandi fisici in aggiornamenti di stato HA e viceversa.

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
│   └── ring_keypad_log.yaml            # Sistema log: logbook, timeline, note
├── var/
│   └── ring_keypad.yaml                # Var timeline (da aggiungere al var.yaml di HA)
└── ANALISI_TECNICA.md                  # Questo file
```

### Mappatura file → entità

| File | Entità dichiarate |
|------|-------------------|
| `ring_keypad_globals.yaml` | `input_select`, `input_text`, `input_number`, `input_boolean` |
| `ring_keypad_scripts.yaml` | `script.*` |
| `ring_keypad_automations.yaml` | `automation.*` (trigger MQTT) |
| `ring_keypad_keypads.yaml` | `mqtt.sensor`, `mqtt.binary_sensor` per tastiera |
| `ring_keypad_integration.yaml` | `automation.*` (bridge esterno) |
| `ring_keypad_log.yaml` | `input_text` logbook, `script.*` log, `automation.*` log, `template sensor` timeline |
| `var/ring_keypad.yaml` | `var.ring_keypad_timeline_md`, `var.ring_keypad_timeline_critici_md`, `var.ring_keypad_timeline_operazioni_md`, `var.ring_keypad_timeline_note_md` |

---

## Sezione 2 — Protocollo Z-Wave / MQTT

Le tastiere Ring Keypad V2 comunicano via zwave2mqtt su due Command Class (CC) Z-Wave:

### Entry Control CC (111) — Input dalla tastiera

Topic ricevuti: `zwave2mqtt/{keypad_id}/unknownClass_111/endpoint_0/{event_type}/{data_type}`

| event_type | data_type | Significato |
|------------|-----------|-------------|
| 3 | 2 | Tasto Disarm + codice |
| 6 | 2 | Tasto Arm Home + codice |
| 6 | 0 | Tasto Arm Home senza codice |
| 5 | 2 | Tasto Arm Away + codice |
| 5 | 0 | Tasto Arm Away senza codice |
| 2 | 2 | Tasto V (Enter) + codice → arma profilo sera (valida PIN) |
| 2 | 0 | Tasto V (Enter) senza codice → arma sera se `require_code_to_arm=off` |
| 25 | 2 | Tasto X (Cancel) + codice — annulla bypass o solo log |
| 25 | 0 | Tasto X (Cancel) senza codice — annulla bypass o solo log |
| 17 | 0 | Anti-rapina Police hold 3s |
| 19 | 0 | Medical hold 3s |
| 16 | 0 | Fire hold 3s |

Il payload è JSON: `{"time":..., "value":"1234"}` per eventi con codice, `{"time":...}` per eventi senza.

### Indicator CC — Output verso la tastiera

Topic pubblicati: `zwave2mqtt/{keypad_id}/indicator/endpoint_0/{Name}/{sub_cmd}/set`

| sub_cmd | Significato | Payload |
|---------|-------------|---------|
| 1 | On/Off indicatore | 0 (off) o 99 (on) |
| 9 | Livello suono/LED | 1 (silenzioso) o 99 (con suono) |
| timeout | Countdown formato stringa `"XmYs"` | es. `"0m30s"` (30s), `"0m40s"` (40s), `"1m30s"` (90s) |

**Indicatori disponibili:**

| Indicatore | sub_cmd | Payload | Descrizione |
|------------|---------|---------|-------------|
| `Not_armed_-_disarmed` | 9 | 1/99 voice-aware | Sistema disarmato |
| `Armed_Stay` | 9 | 1/99 voice-aware | Armato home |
| `Armed_Away` | 9 | 1/99 voice-aware | Armato away |
| `Code_not_accepted` | 9 | 99 fisso | Codice errato |
| `Bypass_challenge` | 1 | 99 fisso | Richiesta bypass |
| `Exit_Delay` | timeout | stringa `"Xm Ys"` | Countdown uscita |
| `Entry_Delay` | timeout | stringa `"Xm Ys"` | Countdown entrata |
| `Alarming` | 9 | 99 fisso | Allarme intrusione CON suono |
| `Alarming_Burglar` | 9 | 1 fisso | Anti-rapina SEMPRE SILENZIOSO |
| `Alarming_Smoke_-_Fire` | 9 | 99 fisso | Allarme fuoco |
| `Alarming_Medical` | 9 | 99 fisso | Allarme medico |
| `Alarming_Water_leak` | 9 | 99 fisso | Allarme perdita acqua |

### LED Display Configuration

Topic: `zwave2mqtt/{keypad_id}/configuration/endpoint_0/System_Security_Mode_Display/set`

| Payload | Descrizione |
|---------|-------------|
| `0` | LED spenti |
| `1` | Fast blink ~1s (usato da `armed_sera`) |
| `3` | Lampeggio 3s |
| `601` | Sempre accesi |

### Status nodo

- Topic: `zwave2mqtt/{keypad_id}/status`
- Payload JSON: `{"time": <timestamp>, "value": true, "status": "Alive", "nodeId": N}`
- Usare `value_template: "{{ value_json.status }}"` per estrarre `"Alive"` o `"Dead"`

### Batteria

- Topic: `zwave2mqtt/{keypad_id}/battery/endpoint_0/level`
- Payload JSON: `{"time": <timestamp>, "value": 100}` — il campo `value` è la percentuale reale (0-100)
- Usare `value_template: "{{ value_json.value }}"` — NON il vecchio topic `battery/0/isLow` (non esiste)

---

## Sezione 3 — Principio Multi-Tastiera (Wildcard MQTT)

Il supporto multi-tastiera si basa su **due meccanismi**:

### 3.1 Ricezione (input)

Le automazioni usano wildcard MQTT `+` per catturare eventi da qualsiasi tastiera:

```
zwave2mqtt/+/unknownClass_111/endpoint_0/3/2   ← Disarm da QUALSIASI tastiera
```

Il `keypad_id` viene estratto automaticamente dal topic:

```yaml
variables:
  keypad_id: "{{ trigger.topic.split('/')[1] }}"
```

### 3.2 Trasmissione (output/feedback)

Gli script accettano `keypad_id` come parametro e costruiscono il topic dinamicamente:

```yaml
topic: "zwave2mqtt/{{ keypad_id }}/indicator/endpoint_0/Not_armed_-_disarmed/9/set"
```

### 3.3 Registro tastiere attive

L'elenco dei topic base è in `input_text.ring_keypad_active_keypads` (CSV):

```
tastiera_allarme_camera,tastiera_ingresso,tastiera_garage
```

Usato da `ring_keypad_sync_all` e `ring_keypad_broadcast_alarm` per iterare su tutte le tastiere.

### 3.4 Aggiungere una seconda tastiera

1. **UI HA** — Aggiungere il topic base a `input_text.ring_keypad_active_keypads` separato da virgola  
   (es. `tastiera_allarme_camera,tastiera_ingresso`)

2. **`ring_keypad_keypads.yaml`** — Decommentare e adattare il blocco template per la nuova tastiera.  
   **Obbligatorio**: impostare `object_id` seguendo il pattern `<topic_base>_batteria` / `<topic_base>_online`  
   per evitare conflitti con la discovery zwave2mqtt (bug RK-11).  
   Esempio:
   ```yaml
   object_id: tastiera_ingresso_batteria   # → sensor.tastiera_ingresso_batteria
   object_id: tastiera_ingresso_online     # → binary_sensor.tastiera_ingresso_online
   ```

3. **`plancia_controllo.yaml`** — Decommentare e adattare il blocco template nella sezione "Stato Hardware"  
   sostituendo `<topic_base>` con il topic base reale.

4. **Riavviare HA** per registrare i nuovi sensori MQTT.

5. **Nessuna altra modifica necessaria**: le automazioni con wildcard, tutti gli script `_all`  
   (`sync_all`, `broadcast_alarm`, `set_led_mode_all`, `set_siren_volume_all`) e  
   `restore_on_connect` gestiscono tutto automaticamente in base alla lista `ring_keypad_active_keypads`.

---

## Sezione 4 — Entità Helper

### `input_select`

| Entità | Valori possibili | Descrizione |
|--------|-----------------|-------------|
| `input_select.ring_keypad_alarm_state` | disarmed, armed_home, armed_away, armed_sera, arming, triggered_burglar, triggered_rapina, triggered_fire, triggered_medical, triggered_water | Stato interno del keypad (unica fonte di verità per gli indicatori) |
| `input_select.ring_keypad_led_mode` | Spenti, Lampeggio (3s), Sempre accesi | Controlla System_Security_Mode_Display sulla tastiera |

### `input_text`

| Entità | Descrizione | Note |
|--------|-------------|------|
| `ring_keypad_active_keypads` | Elenco topic base tastiere (CSV) | Modifica dalla UI |
| `ring_keypad_last_user` | Ultimo utente che ha agito | Log |
| `ring_keypad_last_event` | Descrizione ultimo evento | Log |
| `ring_keypad_last_keypad` | Topic base dell'ultima tastiera usata | Log |
| `ring_keypad_arming_target_mode` | Target mode (armed_home/armed_away) salvato all'inizio dell'arming | Usato dall'integration per impostare profilo allarme-core e chiamare arm_allarme_core (normale) con calcolo bypass |
| `ring_keypad_bypass_mode` | Mode (armed_home/armed_away/armed_sera) del bypass pendente | Impostato da `arm_or_challenge`; resettato da `bypass_cancel` |
| `ring_keypad_bypass_keypad` | Topic base della tastiera che ha avviato il bypass | Impostato da `arm_or_challenge`; resettato da `bypass_cancel` |
| `ring_keypad_pin_master` | PIN Master | mode: password; pattern `^$\|^[0-9]{4,8}$` (fix C-5) |
| `ring_keypad_pin_user1..3` | PIN Utenti 1-3 | mode: password; pattern `^$\|^[0-9]{4,8}$` (fix C-5) |
| `ring_keypad_name_master` | Nome Master | Per log |
| `ring_keypad_name_user1..3` | Nomi Utenti 1-3 | Per log |

### `input_number`

| Entità | Default | Descrizione |
|--------|---------|-------------|
| `ring_keypad_bypass_timeout` | slider 5-60s | Timeout conferma bypass: dopo quanti secondi il `Bypass_challenge` scade e il LED viene ripristinato |
| `ring_keypad_exit_delay` | 30s | Countdown uscita (arming → armed) |
| `ring_keypad_siren_volume` | 5 | Volume sirena hardware (1-10), sincronizzato via `Siren_Volume` configuration CC |
| `ring_keypad_failed_attempts` | 0 (tecnico, `initial: 0` accettato) | Contatore tentativi PIN falliti — reset dopo 60s inattività o PIN corretto |
| `ring_keypad_max_failed_attempts` | configurabile (default HA: 5) | Soglia tentativi prima del lockout (3-10) |

### `input_boolean`

| Entità | Default | Descrizione |
|--------|---------|-------------|
| `ring_keypad_debug` | false | Modalità debug |
| `ring_keypad_require_code_to_arm` | false | Se true, richiede PIN anche per armare |
| `ring_keypad_bypass_pending` | false | Flag attivo durante l'attesa di conferma bypass (sensori aperti) |
| `ring_keypad_voice_feedback` | true | Se on: payload 99 (LED+suono) per disarmed/armed; se off: payload 1 (solo LED) |
| `ring_keypad_lockout_active` | — | Lockout brute-force attivo; reset automatico dopo 5 min (fix C-6) |

### Template binary_sensor (dichiarati in `ring_keypad_globals.yaml`)

| Entità | Descrizione |
|--------|-------------|
| `binary_sensor.keypad_collisione_pin` | `on` se due o più utenti hanno lo stesso PIN (≥4 cifre) — fix C-4 |

### Sensori MQTT (per tastiera, dichiarati in `ring_keypad_keypads.yaml`)

| Entità | Tipo | Descrizione |
|--------|------|-------------|
| `sensor.tastiera_<id>_batteria` | sensor | Livello batteria reale 0-100% (topic: `battery/endpoint_0/level`) |
| `binary_sensor.tastiera_<id>_online` | binary_sensor | Connettività Z-Wave (topic: `status`, field JSON `status`) |

**Tastiera attiva:** `tastiera_allarme_camera` (nodeId 5)

---

## Sezione 5 — Gestione PIN e Sicurezza

### Archiviazione PIN

I PIN sono in `input_text` con `mode: password`:
- **Nascosti nell'UI** di HA (campo mascherato)
- **Non cifrati** a livello di storage HA (file `.storage/core.restore_state`)
- **Non richiedono `secrets.yaml`**
- Il valore iniziale è vuoto: impostare dalla UI dopo il primo avvio

### Validazione PIN

Il pattern di validazione esclude automaticamente i PIN non configurati (vuoti):

```jinja2
{% set ns = namespace(valid=false) %}
{% for pin in [pin_master, pin_user1, pin_user2, pin_user3] %}
  {% if pin | length >= 4 and pin == code %}
    {% set ns.valid = true %}
  {% endif %}
{% endfor %}
{{ ns.valid }}
```

- PIN con meno di 4 caratteri sono ignorati (previene match accidentali di PIN vuoti)
- Un PIN non configurato (stringa vuota) non fa mai match
- L'utente che ha agito viene identificato e salvato in `ring_keypad_last_user`

### `ring_keypad_require_code_to_arm`

Quando attivo (`on`), i tasti Arm Away/Home senza codice (`data_type=0`) attivano `code_rejected` invece di armare. Utile in contesti ad alta sicurezza.

---

## Sezione 6 — Script

### Script di indicazione (hardware)

| Script | Parametri | Azione MQTT | Note |
|--------|-----------|-------------|------|
| `ring_keypad_send_indicator` | keypad_id, indicator, sub_cmd, value | Invia payload generico | — |
| `ring_keypad_show_disarmed` | keypad_id | `Not_armed_-_disarmed/9` | Voice-aware |
| `ring_keypad_show_armed_home` | keypad_id | `Armed_Stay/9` | Voice-aware |
| `ring_keypad_show_armed_away` | keypad_id | `Armed_Away/9` | Voice-aware |
| `ring_keypad_show_armed_sera` | keypad_id | `Armed_Stay/9` payload 1 + `System_Security_Mode_Display` payload 1 | Silenzioso + fast blink LED (~1s); bypassa `ring_keypad_led_mode` |
| `ring_keypad_code_rejected` | keypad_id | `Code_not_accepted/9` payload 99 | Sempre con suono |
| `ring_keypad_bypass_challenge` | keypad_id | `Bypass_challenge/1` payload 99 + avvia `bypass_keepalive` | Primo invio con voce; keepalive mantiene il LED acceso ogni 5s |
| `ring_keypad_show_exit_delay` | keypad_id | `Exit_Delay/timeout` stringa `Xm Ys` | — |
| `ring_keypad_alarm_burglar` | keypad_id | `Alarming/9` payload 99 | Intrusione CON suono |
| `ring_keypad_alarm_rapina` | keypad_id | `Alarming_Burglar/9` payload 1 | SEMPRE SILENZIOSO |
| `ring_keypad_alarm_fire` | keypad_id | `Alarming_Smoke_-_Fire/9` payload 99 | — |
| `ring_keypad_alarm_medical` | keypad_id | `Alarming_Medical/9` payload 99 | — |
| `ring_keypad_alarm_water` | keypad_id | `Alarming_Water_leak/9` payload 99 | — |

### Script LED

| Script | Parametri | Descrizione |
|--------|-----------|-------------|
| `ring_keypad_set_led_mode` | keypad_id | Pubblica valore LED da `ring_keypad_led_mode` su singola tastiera |
| `ring_keypad_set_led_mode_all` | — | Itera `ring_keypad_active_keypads` e chiama `set_led_mode` |

### Script volume sirena

| Script | Parametri | Descrizione |
|--------|-----------|-------------|
| `ring_keypad_set_siren_volume` | keypad_id | Pubblica `ring_keypad_siren_volume` su `configuration/endpoint_0/Siren_Volume/set` per una tastiera |
| `ring_keypad_set_siren_volume_all` | — | Itera `ring_keypad_active_keypads` e chiama `set_siren_volume` |

### Script di sincronizzazione

| Script | Parametri | Descrizione |
|--------|-----------|-------------|
| `ring_keypad_sync_state` | keypad_id | Dispatcher: chiama prima `set_led_mode` (reset preferenza utente), poi invia indicatore corretto in base a `ring_keypad_alarm_state` (11 stati, incluso `armed_sera`) |
| `ring_keypad_sync_all` | — | Chiama `sync_state` su tutte le tastiere in `ring_keypad_active_keypads` |
| `ring_keypad_broadcast_alarm` | alarm_type | Invia allarme specifico a tutte le tastiere (burglar/rapina/fire/medical/water) |

### Script di logica

| Script | Parametri | Descrizione |
|--------|-----------|-------------|
| `ring_keypad_resolve_user` | code | Identifica l'utente dal PIN, aggiorna `ring_keypad_last_user` |
| `ring_keypad_validate_and_disarm` | code, keypad_id | Valida PIN → disarma o rifiuta |
| `ring_keypad_validate_and_arm` | code, mode, keypad_id | **Puro PIN→arm**: PIN valido → `resolve_user` + `arm_or_challenge`; PIN errato → `code_rejected`. Nessuna logica bypass (gestita a monte nelle automazioni) |
| `ring_keypad_arm_or_challenge` | mode, keypad_id | Controlla sensori aperti nelle zone del profilo target: se pulito → `do_arm`; se ci sono sensori → attiva `bypass_pending` + `bypass_challenge` + avvia `bypass_timeout_script` |
| `ring_keypad_bypass_confirm` | keypad_id | `stored_mode` ← `bypass_mode`; `turn_off do_arm`; `bypass_cancel`; `do_arm(stored_mode)` — nessuna verifica PIN, il tasto V è sufficiente |
| `ring_keypad_bypass_keepalive` | keypad_id | `mode: restart` — loop `repeat/while`: ogni 5s invia `Bypass_challenge/9` payload `1` (sub_cmd 9 = solo LED, no voce) finché `bypass_pending=on` AND `bypass_keypad==keypad_id`. sub_cmd 1 attiva la voce; sub_cmd 9 solo LED. |
| `ring_keypad_bypass_cancel` | — | Resetta `bypass_pending`, `bypass_mode`, `bypass_keypad`; ferma `bypass_timeout_script` e `bypass_keepalive` |
| `ring_keypad_bypass_timeout_script` | — | `mode: restart` — allo scadere di `ring_keypad_bypass_timeout` secondi: **prima** inline-reset bypass (turn_off pending + clear mode/keypad), **poi** `sync_state` (ordine critico: evita race condition RK-45); non chiama `bypass_cancel` per evitare `turn_off` su se stesso |
| `ring_keypad_do_disarm` | keypad_id | **Prima** chiama `bypass_cancel`; poi ferma `do_arm` (RK-30); imposta stato `disarmed` + log |
| `ring_keypad_do_arm` | mode, keypad_id | Imposta `arming` → delay exit → imposta `mode` + log |

---

## Sezione 7 — Automazioni

### Automazioni input (da tastiera)

| ID Automazione | Trigger MQTT | Azione |
|----------------|-------------|--------|
| `ring_keypad_input_disarm` | `zwave2mqtt/+/unknownClass_111/endpoint_0/3/2` | `validate_and_disarm` |
| `ring_keypad_input_arm_away` | `zwave2mqtt/+/unknownClass_111/endpoint_0/5/2` | Se bypass_pending → `code_rejected`; altrimenti `validate_and_arm mode=armed_away` |
| `ring_keypad_input_arm_away_nocode` | `zwave2mqtt/+/unknownClass_111/endpoint_0/5/0` | Se bypass_pending → `code_rejected`; se `require_code=off` → `arm_or_challenge`; altrimenti `code_rejected` |
| `ring_keypad_input_arm_home` | `zwave2mqtt/+/unknownClass_111/endpoint_0/6/2` | Se bypass_pending → `code_rejected`; altrimenti `validate_and_arm mode=armed_home` |
| `ring_keypad_input_arm_home_nocode` | `zwave2mqtt/+/unknownClass_111/endpoint_0/6/0` | Se bypass_pending → `code_rejected`; se `require_code=off` → `arm_or_challenge`; altrimenti `code_rejected` |
| `ring_keypad_input_enter` | `zwave2mqtt/+/unknownClass_111/endpoint_0/2/2` | Se bypass_pending → `bypass_confirm`; altrimenti `validate_and_arm mode=armed_sera` |
| `ring_keypad_input_enter_nocode` | `zwave2mqtt/+/unknownClass_111/endpoint_0/2/0` | Se bypass_pending → `bypass_confirm`; se `require_code=off` → `arm_or_challenge`; altrimenti `code_rejected` |
| `ring_keypad_input_cancel` | `25/2` (con codice) + `25/0` (senza codice) | Se `bypass_pending=on` per questa tastiera → `bypass_cancel` + `sync_state` (annulla bypass attivo); altrimenti solo log "Tasto X" |
| `ring_keypad_input_rapina` | `zwave2mqtt/+/unknownClass_111/endpoint_0/17/0` | Stato `triggered_rapina` + broadcast rapina |
| `ring_keypad_input_medical` | `zwave2mqtt/+/unknownClass_111/endpoint_0/19/0` | Stato `triggered_medical` + broadcast |
| `ring_keypad_input_fire` | `zwave2mqtt/+/unknownClass_111/endpoint_0/16/0` | Stato `triggered_fire` + broadcast |

### Automazioni di sistema

| ID Automazione | Trigger | Azione |
|----------------|---------|--------|
| `ring_keypad_sync_on_state_change` | State: `ring_keypad_alarm_state` | `sync_all` |
| `ring_keypad_restore_on_connect` | MQTT: `zwave2mqtt/+/status` = Alive | `set_siren_volume` + `sync_state` (gestisce LED internamente) su tastiera tornata online |
| `ring_keypad_sync_on_ha_start` | HA start | Delay 15s → `set_siren_volume_all` + `sync_all` (`sync_state` gestisce LED internamente per ogni tastiera) |
| `ring_keypad_sync_led_on_mode_change` | State: `ring_keypad_led_mode` | `set_led_mode_all` **solo se** `ring_keypad_alarm_state != 'armed_sera'` (fast blink non viene sovrascritto) |
| `ring_keypad_sync_siren_volume_on_change` | State: `ring_keypad_siren_volume` | `set_siren_volume_all` |
| `ring_keypad_lockout_reset` | State: `ring_keypad_lockout_active` = on per 5 min | Disattiva lockout + azzera `failed_attempts` + log (fix C-6) |
| `ring_keypad_failed_attempts_reset` | State: `ring_keypad_failed_attempts` invariato per 60s (+ lockout off + valore > 0) | Azzera contatore tentativi falliti — sliding window anti brute-force (fix C-6) |

### Automazioni di integrazione (in `ring_keypad_integration.yaml`)

| ID Automazione | Trigger | Condizione | Azione |
|----------------|---------|------------|--------|
| `ring_keypad_sync_from_allarme_core` | State: `allarme_core_stato` o `allarme_core_profilo` | stato==armed per trigger profilo | Aggiorna `ring_keypad_alarm_state` con mappatura profilo-aware |
| `ring_keypad_command_to_allarme_core` | State: `ring_keypad_alarm_state` → disarmed/arming/armed_home/armed_away/armed_sera | **debug OFF** + anti-loop stato/profilo | Imposta profilo (notte/giorno/sera) + chiama arm/disarm allarme-core |
| `ring_keypad_sync_exit_delay` | State: `allarme_core_arming_delay` + HA start | — | Copia valore in `ring_keypad_exit_delay` |
| `ring_keypad_sync_from_safety_core` | State: `safety_core_stato` | — | Aggiorna `ring_keypad_alarm_state` (fire/water/reset) |
| `ring_keypad_command_to_safety_core` | State: `ring_keypad_alarm_state` → triggered_fire | **debug OFF** + safety_core non già triggered | Imposta safety_core triggered + log manuale |

> **Debug mode (`input_boolean.ring_keypad_debug = on`):** le automazioni `ring_keypad_command_to_allarme_core` e `ring_keypad_command_to_safety_core` non propagano nulla verso i sistemi core. Le automazioni inverse (`sync_from_allarme_core`, `sync_from_safety_core`, `sync_exit_delay`) continuano a funzionare normalmente: la tastiera fisica riceve comunque i LED/suoni di feedback.

---

## Sezione 8 — Flusso Operativo

### Inserimento allarme da tastiera

```
Utente preme [ARM AWAY] + PIN
    ↓
MQTT: zwave2mqtt/tastiera_camera/111/0/5/2
    ↓
Automazione: ring_keypad_input_arm_away
    ↓
Script: ring_keypad_validate_and_arm (code, mode="armed_away", keypad_id)
    ↓
    [PIN valido?]
    ├─ SI → ring_keypad_resolve_user (aggiorna last_user)
    │        ring_keypad_arm_or_challenge (mode, keypad_id)
    │            ↓
    │        [Sensori aperti nelle zone del profilo?]
    │        ├─ NO → ring_keypad_do_arm (mode, keypad_id)
    │        │            ↓
    │        │        ring_keypad_alarm_state = "arming"
    │        │            ↓ (sync automatico via ring_keypad_sync_on_state_change)
    │        │        ring_keypad_sync_all → show_exit_delay su tutte le tastiere
    │        │            ↓ (delay exit_delay secondi)
    │        │        ring_keypad_alarm_state = "arming"
    │        │            ↓ (via ring_keypad_command_to_allarme_core, IMMEDIATAMENTE)
    │        │        script.arm_allarme_core chiamato subito (calcola bypass sensori aperti)
    │        │            ↓ (delay exit_delay secondi in parallelo tra keypad e allarme-core)
    │        │        ring_keypad_alarm_state = "armed_away"
    │        │            ↓ (sync automatico)
    │        │        ring_keypad_sync_all → show_armed_away su tutte le tastiere
    │        │
    │        └─ SI → bypass_pending = on; bypass_mode = mode; bypass_keypad = keypad_id
    │                 ring_keypad_bypass_challenge (Bypass_challenge/1 payload 99)
    │                 ring_keypad_bypass_timeout_script (avviato fire-and-forget)
    │                     ↓
    │                 [Utente preme tasto V — con o senza PIN]
    │                     ↓
    │                 Intercettato PRIMA di validate_and_arm in entrambe le automazioni
    │                     ↓
    │                 ring_keypad_bypass_confirm(keypad_id)
    │                     ↓
    │                 stored_mode ← bypass_mode; turn_off do_arm; bypass_cancel; do_arm(stored_mode)
    │
    │                 [Utente preme tasto ARM HOME/AWAY + PIN durante bypass attivo]
    │                     ↓
    │                 validate_and_arm CASO 2a (bypass_pending=on, stessa tastiera, mode≠armed_sera)
    │                     ↓
    │                 code_rejected + log "Bypass attivo - usare tasto V + PIN per confermare"
    │
    │                 [Utente preme tasto X durante bypass attivo]
    │                     ↓
    │                 ring_keypad_input_cancel → bypass_cancel + sync_state (LED ripristinato)
    │
    │                 [Timeout scade]
    │                     ↓
    │                 bypass_timeout_script: inline-reset bypass → sync_state (LED ripristinato)
    │
    └─ NO → ring_keypad_code_rejected (tastiera sorgente)
             aggiorna ring_keypad_last_event
```

### Disinserimento allarme da tastiera

```
Utente preme [DISARM] + PIN
    ↓
MQTT: zwave2mqtt/tastiera_ingresso/111/0/3/2
    ↓
keypad_id = "tastiera_ingresso"
    ↓
Script: ring_keypad_validate_and_disarm
    ↓
    [PIN valido?]
    ├─ SI → ring_keypad_do_disarm
    │        ring_keypad_alarm_state = "disarmed"
    │            ↓ (sync + integrazione)
    │        Tutte le tastiere: LED disarmed
    │        script.disarm_allarme_core
    │
    └─ NO → code_rejected feedback sulla tastiera sbagliata
```

### Allarme da sistema esterno → tastiere

```
allarme_core_stato = "triggered"
    ↓
ring_keypad_sync_from_allarme_core
    ↓
ring_keypad_alarm_state = "triggered_burglar"
    ↓ (sync automatico)
ring_keypad_sync_all
    ↓
ring_keypad_alarm_burglar su OGNI tastiera attiva
```

---

## Sezione 9 — Integrazione con Sistemi Esterni

### Architettura di integrazione

Il ring-keypad è **disaccoppiato** dal sistema di allarme. L'integrazione avviene tramite `ring_keypad_integration.yaml` che:

1. **Legge** lo stato del sistema esterno e aggiorna `ring_keypad_alarm_state`
2. **Scrive** comandi al sistema esterno quando il keypad cambia stato

### Prevenzione loop bidirezionale

```
allarme-core cambia stato
    → ring_keypad_sync_from_allarme_core aggiorna ring_keypad_alarm_state
    → ring_keypad_command_to_allarme_core si attiva
    → CONTROLLO ANTI-LOOP: allarme-core è già nello stato corretto?
        ├─ SI → nessun comando inviato (loop interrotto)
        └─ NO → comando inviato (caso reale da tastiera)
```

### Integrazione allarme-core (attiva, sezione 1)

**Mappatura stati allarme-core → keypad (profilo-aware):**

| `allarme_core_stato` | `allarme_core_profilo` | `ring_keypad_alarm_state` | LED tastiera |
|----------------------|------------------------|---------------------------|--------------|
| `disarmed` | qualsiasi | `disarmed` | Disarmed |
| `arming` | qualsiasi | `arming` | Exit Delay countdown |
| `armed` | `notte` | `armed_home` | Armed Stay |
| `armed` | `giorno` | `armed_away` | Armed Away |
| `armed` | `tutti` | `armed_away` | Armed Away |
| `armed` | `sera` | `armed_sera` | Armed Stay silenzioso + fast blink LED (~1s) |
| `triggered` | qualsiasi | `triggered_burglar` | Alarming |

Il trigger su `allarme_core_profilo` è condizionato a `stato == 'armed'` per evitare flickering durante la sequenza di armo dalla tastiera.

**Comandi dal keypad verso allarme-core:**

| `ring_keypad_alarm_state` | Profilo impostato | Script chiamato | Note |
|---------------------------|-------------------|-----------------|------|
| `disarmed` | — | `script.disarm_allarme_core` | — |
| `arming` | da `ring_keypad_arming_target_mode` | `script.arm_allarme_core` | Calcola bypass sensori aperti |
| `armed_home` | `notte` | `script.arm_allarme_core` | Safety fallback (anti-loop se già arming) |
| `armed_away` | `giorno` | `script.arm_allarme_core` | Safety fallback (anti-loop se già arming) |

**Anti-loop per `arming`:** invia solo se `allarme_core_stato not in ['arming', 'armed']`  
**Anti-loop per `armed_home`:** invia solo se `stato != 'arming' AND (stato != 'armed' OR profilo != 'notte')`  
**Anti-loop per `armed_away`:** invia solo se `stato != 'arming' AND (stato != 'armed' OR profilo not in ['giorno','tutti'])`

**Sync exit delay:** `ring_keypad_exit_delay` è mantenuto uguale ad `allarme_core_arming_delay` dall'automazione `ring_keypad_sync_exit_delay`. Unica sorgente di verità: `allarme_core_arming_delay`.

### Automazioni di integrazione aggiornate

| ID Automazione | Trigger | Azione |
|----------------|---------|--------|
| `ring_keypad_sync_from_allarme_core` | State: `allarme_core_stato` o `allarme_core_profilo` (se armed) | Aggiorna `ring_keypad_alarm_state` con mappatura profilo-aware |
| `ring_keypad_command_to_allarme_core` | State: `ring_keypad_alarm_state` → disarmed/arming/armed_home/armed_away/armed_sera | Imposta profilo (notte/giorno/sera) + chiama arm_allarme_core (arming) o disarm_allarme_core |
| `ring_keypad_sync_exit_delay` | State: `allarme_core_arming_delay` + HA start | Copia valore in `ring_keypad_exit_delay` |

### Integrazione safety-core (attiva, sezione 3)

**Mappatura safety-core → keypad:**

| `safety_core_stato` | `safety_core_ultima_categoria` | `ring_keypad_alarm_state` | LED tastiera |
|---------------------|-------------------------------|---------------------------|--------------|
| `triggered` | `fumo` / `gas` / `carbonio` | `triggered_fire` | Alarming Smoke-Fire |
| `triggered` | `acqua` | `triggered_water` | Alarming Water Leak |
| `ok` (reset) | qualsiasi | re-sync da allarme-core | LED allarme-core corrente |

Al reset di safety-core (`ok`), il keypad ripristina il LED in base allo stato attuale di allarme-core (armato/disarmato/triggered burglar).

**Mappatura keypad → safety-core:**

| Evento tastiera | Condizione | Azione su safety-core | Log |
|-----------------|------------|----------------------|-----|
| Tasto Fire hold 3s → `triggered_fire` | `safety_core_stato != 'triggered'` | stato=`triggered`, categoria=`fumo` | "🔥 Allarme fuoco manuale da tastiera Ring Keypad" |

Il log distingue lo scatto manuale da tastiera rispetto all'attivazione automatica dei sensori fisici.

| ID Automazione | Trigger | Azione |
|----------------|---------|--------|
| `ring_keypad_sync_from_safety_core` | State: `safety_core_stato` | Aggiorna keypad in base a stato+categoria; al reset ripristina allarme-core |
| `ring_keypad_command_to_safety_core` | State: `ring_keypad_alarm_state` → `triggered_fire` | Se safety-core non già triggered: stato=triggered, categoria=fumo, log manuale |

### Integrazione alarm_control_panel generico (commentata, sezione 2)

Disponibile in `ring_keypad_integration.yaml` per sistemi come Alarmo, Manuel Alp, ecc.
Decommentare la sezione 2 e commentare la sezione 1.

---

## Sezione 10 — Installazione e Configurazione

### Requisiti

- Home Assistant 2022.3+ (per `to: [lista]` in trigger state)
- zwave2mqtt configurato con il nodo Ring Keypad V2
- Integration MQTT attiva in HA
- (opzionale) allarme-core installato per l'integrazione

### Passi installazione

1. Copiare i 5 file `packages/*.yaml` in `config/packages/ring_keypad/`

2. Aggiungere in `configuration.yaml`:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```

3. Riavviare HA

4. Dalla UI HA, configurare:
   - `ring_keypad_active_keypads`: topic base della/e tastiera/e (es. `tastiera_camera`)
   - `ring_keypad_pin_master`, `ring_keypad_pin_user1..3`: impostare i PIN
   - `ring_keypad_name_master`, `ring_keypad_name_user1..3`: impostare i nomi

5. Verificare che il topic base corrisponda al nome del nodo in zwave2mqtt

### Aggiungere una nuova tastiera

1. **UI HA**: aggiungere il topic base a `ring_keypad_active_keypads` separato da virgola  
   (es. `tastiera_allarme_camera,tastiera_ingresso`)

2. **`ring_keypad_keypads.yaml`**: decommentare e adattare il blocco template.  
   **Obbligatorio**: includere `object_id` con pattern `<topic_base>_batteria` / `<topic_base>_online`.  
   Senza `object_id` HA genera entity ID auto che possono collidere con zwave2mqtt discovery (RK-11).

3. **`plancia_controllo.yaml`**: decommentare e adattare il blocco template nella sezione "Stato Hardware",  
   sostituendo `<topic_base>` con il topic base reale.

4. **Riavviare HA** per registrare i sensori MQTT.

5. **Nessuna modifica a script, automazioni o integrazione**: tutto gestito automaticamente.

### Trovare il topic base di una tastiera

In zwave2mqtt, il topic base è il **nome del nodo** configurato. Verificare:
- Dashboard zwave2mqtt → nomi nodi
- MQTT broker → topic browser → `zwave2mqtt/+/#`

---

## Sezione 11 — Note Architetturali

### Perché non secrets.yaml

I `secrets.yaml` richiedono riavvio HA per modificare i PIN. Con `input_text mode:password`:
- Modifica PIN senza riavvio
- Visibilità controllata nell'UI (campo mascherato)
- Nessuna dipendenza da file esterno da gestire manualmente

### Perché wildcard MQTT invece di automazioni per tastiera

Con N tastiere, l'approccio classico richiederebbe N×10 automazioni duplicate.
Il wildcard + estrazione `keypad_id` dal topic riduce a 10 automazioni indipendenti dal numero di tastiere.

### Mappatura profilo ↔ modalità tastiera

`allarme_core_profilo` è la sorgente di verità per la distinzione casa/fuori-casa:

| Profilo | Significato | Tasto tastiera | LED tastiera |
|---------|-------------|----------------|--------------|
| `notte` | Inserimento in casa (perimetrale + notte) | ARM HOME (6/2 o 6/0) | Armed Stay |
| `giorno` | Inserimento fuori casa (tutti i sensori) | ARM AWAY (5/2 o 5/0) | Armed Away |
| `tutti` | Inserimento fuori casa completo | — (da allarme-core) | Armed Away |
| `sera` | Profilo serale (silenzioso, perimetrale leggero) | V/Enter (2/2 o 2/0) | Armed Stay silenzioso + fast blink LED (~1s) |

Quando si arma da tastiera: il profilo viene impostato dall'integration nel caso `arming`, prima di chiamare `arm_allarme_core`.  
Quando si arma da allarme-core: il profilo è già impostato, la tastiera lo legge e mostra il LED corretto.

### Gestione doppio delay e bypass sensori

**I due sistemi armano in parallelo:** quando la tastiera entra in `arming`, `ring_keypad_command_to_allarme_core` chiama subito `arm_allarme_core` (normale). allarme-core entra nel proprio stato `arming`, calcola i bypass dei sensori aperti, poi arma. Il countdown visivo della tastiera (`ring_keypad_exit_delay`) è sincronizzato con `allarme_core_arming_delay` → i due sistemi raggiungono `armed` contemporaneamente senza doppio delay.

**Safety fallback:** se ring_keypad raggiunge `armed_home`/`armed_away` e allarme-core è ancora `disarmed` (scenario anomalo), i rispettivi casi dell'integration chiamano `arm_allarme_core` come fallback.

**Sync delay:** `ring_keypad_exit_delay` = `allarme_core_arming_delay` (tenuti in sync dall'automazione `ring_keypad_sync_exit_delay`).

### `ring_keypad_do_arm` e delay interno

Il delay di exit delay è interno allo script. Se HA si riavvia durante il delay, lo script termina ma lo stato rimane `arming`. Al riavvio, `ring_keypad_sync_on_ha_start` re-sincronizza i LED. Lo stato interno va corretto manualmente (o via integrazione esterna che riporta lo stato corretto).

---

## Sezione 12 — Bug e Limitazioni Conosciute

| # | Descrizione | Stato | Fix |
|---|-------------|-------|-----|
| RK-01 | `armed_home` e `armed_away` non distinguevano il profilo → allarme-core armava sempre con profilo precedente | **Corretto 2026-04-13** | Integration aggiornata: armed_home→profilo notte, armed_away→profilo giorno prima di arm_immediate |
| RK-02 | Delay in `do_arm` non interrompibile dall'esterno | Aperto | Nessuno (limitazione HA scripts) |
| RK-03 | Nessuna protezione brute-force su PIN | Aperto | Implementazione futura con `counter` helper |
| RK-04 | `ring_keypad_sync_on_state_change` si attiva anche per transizioni `→unknown` | Accettato | Il sync_all è idempotente, nessun impatto operativo |
| RK-05 | `ring_keypad_restore_on_connect`: `value_template` rimosso per errore (il topic pubblica JSON, non stringa) — trigger mai scattato | **Corretto 2026-04-11** | Ripristinato `value_template: "{{ value_json.status }}"` |
| RK-06 | `ring_keypad_code_rejected`: payload `"1"` non-standard per Indicator CC sub_cmd 1 | **Corretto 2026-04-11** | Payload cambiato a `"99"` (0xFF = on, secondo specifica Z-Wave) |
| RK-07 | `ring_keypad_bypass_challenge`: payload `"1"` non-standard per Indicator CC sub_cmd 1 | **Corretto 2026-04-11** | Payload cambiato a `"99"` |
| RK-08 | `binary_sensor.tastiera_camera_online`: mancava `value_template` → sensor sempre `off` (payload è JSON, non stringa) | **Corretto 2026-04-11** | Aggiunto `value_template: "{{ value_json.status }}"` |
| RK-09 | Battery sensor: topic errato `battery/0/isLow` (booleano) invece di `battery/endpoint_0/level` (percentuale reale) | **Corretto 2026-04-11** | Topic e `value_template` aggiornati, ora fornisce 0-100% |
| RK-10 | Nome nodo tastiera errato `tastiera_camera` invece di `tastiera_allarme_camera` in keypads.yaml e globals.yaml | **Corretto 2026-04-11** | Aggiornato in tutti i file |
| RK-11 | Entity ID sensori MQTT duplicati con suffisso `_2` — zwave2mqtt MQTT discovery creava entità in conflitto con quelle YAML | **Corretto 2026-04-11** | Disabilitata discovery zwave2mqtt; aggiunti `object_id` espliciti in keypads.yaml: `tastiera_camera_allarme_batteria`, `tastiera_camera_allarme_online` |
| RK-12 | Dashboard `plancia_controllo.yaml`: header usava `header.card` (singolo) invece di `header.cards` (lista) — errore configurazione sections view | **Corretto 2026-04-11** | Cambiato `card:` in `cards:` e rimosso `vertical-stack` wrapper non supportato nell'header |
| RK-13 | `condition: template` con `value_template` usava block scalar `>` — template renderizzava `"True\n"` invece di `"true"` | **Corretto 2026-04-11** | Convertito a stringa inline con `| string | lower` |
| RK-14 | Topic Entry Control CC errati: usavano `111/0/` invece del path reale `unknownClass_111/endpoint_0/` — nessun trigger MQTT funzionante | **Corretto 2026-04-12** | Tutti i topic input aggiornati al formato verificato |
| RK-15 | Topic Indicator CC errati: usavano `135/0/` invece di `indicator/endpoint_0/` — nessun feedback hardware funzionante | **Corretto 2026-04-12** | Tutti i topic output aggiornati al formato verificato |
| RK-16 | Exit/Entry Delay: payload era un intero in secondi (sub_cmd 7) invece della stringa `XmYs` richiesta dal campo `timeout` | **Corretto 2026-04-12** | Script aggiornati con calcolo `(s // 60)m(s % 60)s` — formato confermato corretto da zwave2mqtt (es. `"0m30s"`, `"1m30s"`) |
| RK-17 | `ring_keypad_alarm_burglar` puntava a `Alarming_Burglar` (anti-rapina silenzioso) invece di `Alarming` (intrusione con suono) | **Corretto 2026-04-12** | `alarm_burglar` → `Alarming/9` payload 99; nuovo `alarm_rapina` → `Alarming_Burglar/9` payload 1 |
| RK-18 | Automazione `ring_keypad_input_cancel`: data_type era `0` invece di `2` (il manuale documenta il Cancel con codice come 25/2) | **Corretto 2026-04-12** | Topic aggiornato a `25/2` |
| RK-19 | Automazione `ring_keypad_input_police`: rinominata in `ring_keypad_input_rapina`, stato cambiato da `triggered_burglar` a `triggered_rapina` | **Corretto 2026-04-12** | Distinzione semantica: burglar=intrusione esterna, rapina=minaccia con presenza |
| RK-20 | Doppio countdown all'armo da tastiera: `ring_keypad_do_arm` (exit_delay) + `arm_allarme_core` (arming_delay) causavano due delay in sequenza e flickering del LED | **Corretto 2026-04-13** | Integration chiama `arm_allarme_core` all'inizio della fase `arming` (non alla fine); i due sistemi armano in parallelo con delay sincronizzato → nessun doppio countdown |
| RK-21 | `ring_keypad_sync_from_allarme_core`: stato `armed` sempre mappato a `armed_away` indipendentemente dal profilo — tastiera non distingueva casa/fuori | **Corretto 2026-04-13** | Mappatura profilo-aware: notte→armed_home, giorno/tutti→armed_away, sera→armed_sera (2026-04-14) |
| RK-22 | `ring_keypad_sync_from_allarme_core`: non reagiva ai cambi di `allarme_core_profilo` mentre armato — cambio profilo da UI non aggiornava LED tastiera | **Corretto 2026-04-13** | Aggiunto trigger su `allarme_core_profilo` con condizione `stato==armed` |
| RK-23 | `ring_keypad_exit_delay` e `allarme_core_arming_delay` erano indipendenti — countdown tastiera poteva divergere dal delay allarme-core | **Corretto 2026-04-13** | Aggiunta automazione `ring_keypad_sync_exit_delay` che mantiene i due valori sincronizzati |
| RK-24 | Anti-loop `armed_away` in `ring_keypad_command_to_allarme_core`: controllava solo `profilo != 'giorno'` ma non `!= 'tutti'` — con profilo `tutti` armato, premere armed_away dalla tastiera cambiava profilo a giorno inaspettatamente | **Corretto 2026-04-13** | Condizione aggiornata a `not in ['giorno', 'tutti']` |
| RK-25 | `ring_keypad_sync_from_allarme_core` senza `mode:` — trigger doppio (stato+profilo) in rapida successione poteva scartare esecuzioni | **Corretto 2026-04-13** | Aggiunto `mode: queued, max: 5` |
| RK-26 | `| float` e `| int` senza default — se input_number in stato `unknown` al boot, valore diventava 0 | **Corretto 2026-04-13** | Aggiunti default: `| float(20)` nel sync delay, `| int(30)` negli script exit/entry delay (payload stringa `XmYs` confermato corretto) |
| RK-27 | Sezione 3 integration: safety-core non integrato — allarmi fumo/gas/carbonio/acqua non mostrati su tastiera; tasto Fire non triggerava safety-core | **Corretto 2026-04-13** | Attivata sezione 3: `ring_keypad_sync_from_safety_core` + `ring_keypad_command_to_safety_core` |
| RK-28 | Bypass sensori aperti non calcolato all'armo da tastiera: l'integration chiamava `arm_allarme_core_immediate` saltando lo stato `arming` di allarme-core — il calcolo bypass avviene durante `arming` → sensori aperti scattavano immediatamente | **Corretto 2026-04-13** | `ring_keypad_do_arm` salva `ring_keypad_arming_target_mode` prima di entrare in `arming`; `ring_keypad_command_to_allarme_core` intercetta `arming` e chiama `arm_allarme_core` (normale); `arm_allarme_core_immediate` eliminato da allarme-core; anti-loop `armed_home`/`armed_away` aggiornato con `!= 'arming'` |
| RK-29 | Exit delay countdown non partiva da tastiera: `ring_keypad_do_arm` usava `\| int` senza default nel delay (0s se unknown) e il countdown dipendeva da race condition nel sync chain; inoltre `ring_keypad_sync_from_allarme_core` poteva sovrascrivere il countdown con LED "armed" se `arm_allarme_core` completava prima del delay di `ring_keypad_do_arm` | **Corretto 2026-04-13** | `ring_keypad_do_arm` chiama `ring_keypad_show_exit_delay` direttamente prima del delay (indipendente da sync chain) e usa `\| int(30)` come default; `ring_keypad_sync_from_allarme_core` blocca l'aggiornamento verso armed_home/away se ring_keypad è ancora in `arming` e `ring_keypad_do_arm` è in esecuzione |
| RK-30 | Disarmo durante exit delay non efficace: `ring_keypad_do_disarm` non fermava `ring_keypad_do_arm` → il delay continuava in background, al termine sovrascriveva lo stato con `armed_home/away` e ri-armava allarme-core | **Corretto 2026-04-13** | Aggiunto `script.turn_off ring_keypad_do_arm` come primo passo di `ring_keypad_do_disarm` |
| RK-32 | Exit/Entry Delay countdown non partiva dalla tastiera: payload MQTT pubblicato senza virgolette doppie (`1m0s`) invece del formato JSON stringa richiesto da zwave2mqtt (`"1m0s"`) | **Corretto 2026-04-13** | Payload cambiato a `'"{{ s // 60 }}m{{ s % 60 }}s"'` (singole quotes YAML per includere le doppie nel valore raw) in `ring_keypad_show_exit_delay` e `ring_keypad_show_entry_delay` |
| RK-31 | Disarmo durante exit delay non disarmava allarme-core: catena di 3 bug sovrapposti. (a) `mode: single`: trigger `disarmed` scartato. (b) Anti-loop `allarme_core_stato != 'disarmed'`: race condition se arm_allarme_core non aggiornava ancora stato. (c) `action: script.arm_allarme_core` bloccante: il caso `arming` aspettava il completamento di arm_allarme_core (10s); quando terminava allarme-core era già `armed`, sync_from_allarme_core impostava ring_keypad_alarm_state=armed_home, il caso `armed_home` in coda ri-armava allarme-core svuotando il disarmo. | **Corretto 2026-04-13** | (a) `mode: queued` su ring_keypad_command_to_allarme_core; (b) anti-loop basato su `trigger.from_state.state`; (c) tutti i casi arm_allarme_core cambiati a `script.turn_on` (fire-and-forget): il caso `disarmed` parte immediatamente mentre arm_allarme_core è nel delay e disarm_allarme_core lo ferma prima del completamento |
| RK-33 | Debug mode non bloccava la propagazione keypad→core: con `input_boolean.ring_keypad_debug = on`, i comandi da tastiera (arm/disarm/fire) venivano comunque inviati ad allarme-core e safety-core — impossibile testare la tastiera senza modificare lo stato del sistema | **Corretto 2026-04-13** | Aggiunta `condition: state ring_keypad_debug off` come prima condizione in `ring_keypad_command_to_allarme_core` e `ring_keypad_command_to_safety_core`. Le automazioni inverse (sync_from_*) non sono coinvolte: la tastiera continua a ricevere LED/suoni normalmente |
| RK-34 | `ring_keypad_exit_delay` editabile dalla dashboard: il tile in `plancia_controllo.yaml` aveva `features: numeric-input` e il popup "Configurazione Avanzata" includeva il campo nel form Timing — ma il valore è read-only (sincronizzato automaticamente da `allarme_core_arming_delay`) | **Corretto 2026-04-13** | Rimosso `features: numeric-input` dal tile (nome aggiornato a "Ritardo Uscita (da allarme-core)", stile opacizzato); rimosso da popup Configurazione Avanzata sezione Timing con commento esplicativo |
| RK-35 | `ring_keypad_alarm_burglar/fire/medical/water`: payload voice-aware (`99` se voice_feedback=on, `1` se off) — allarmi di sicurezza vitale potevano risultare silenziosi se voice_feedback era disabilitato | **Corretto 2026-04-13** | Payload fisso a `"99"` per tutti e quattro gli script; `ring_keypad_voice_feedback` rimane attivo solo per gli stati operativi normali (disarmed/armed_home/armed_away) |
| RK-36 | `ring_keypad_sync_from_allarme_core` ramo `disarmed`: `ring_keypad_do_arm` non veniva fermato al disarmo da dashboard/esterno — il delay dell'exit delay continuava in background e al termine sovrascriveva lo stato con `armed_home/away`, ri-armando allarme-core | **Corretto 2026-04-13** | Aggiunto `script.turn_off ring_keypad_do_arm` come primo passo del ramo `disarmed` in `ring_keypad_sync_from_allarme_core`; copre il caso "disarmo da dashboard/esterno" che `ring_keypad_do_disarm` non gestisce (viene chiamato solo da tastiera) |
| RK-37 | Template blocchi per nuove tastiere in `ring_keypad_keypads.yaml` mancavano di `object_id` — aggiungendo una seconda tastiera, HA avrebbe generato entity ID auto dal nome, potenzialmente in conflitto con zwave2mqtt discovery (stessa causa di RK-11) | **Corretto 2026-04-13** | Aggiunti `object_id` con pattern `<topic_base>_batteria` / `<topic_base>_online` nei template commentati; aggiornate sezioni 3.4 e 10 della documentazione; aggiunto template pronto nella dashboard sezione "Stato Hardware" |
| RK-38 | Profilo `sera` armato mostrava tastiera come `disarmed` — nessun indicatore visivo che il sistema fosse armato | **Corretto 2026-04-14** | Introdotto stato `armed_sera` con `Armed_Stay` silenzioso (payload 1) + fast blink LED (`System_Security_Mode_Display=1`, ~1s); nuovo script `ring_keypad_show_armed_sera` che bypassa `ring_keypad_led_mode` senza sovrascriverlo in modo permanente |
| RK-39 | `ring_keypad_sync_led_on_mode_change`: cambio preferenza LED sovrascriveva il fast blink di `armed_sera` — rimpiazzava il blink con il valore salvato dall'utente | **Corretto 2026-04-14** | Aggiunta condizione: l'automazione si blocca se `ring_keypad_alarm_state == 'armed_sera'`; il LED corretto viene ripristinato automaticamente al prossimo cambio di stato tramite `sync_state` |
| RK-40 | Sequenza `ring_keypad_restore_on_connect` e `ring_keypad_sync_on_ha_start`: chiamavano `sync_state` e poi separatamente `set_led_mode` — `set_led_mode` poteva sovrascrivere il fast blink di `armed_sera` impostato da `sync_state` | **Corretto 2026-04-14** | `sync_state` ora chiama `set_led_mode` internamente come primo passo prima di applicare l'indicatore; rimossa la chiamata separata a `set_led_mode` nelle due automazioni |
| RK-41 | Tasto V (Enter) non attivava il profilo sera — `ring_keypad_input_enter` aggiornava solo il log senza armare; non esisteva percorso per armare con profilo sera dalla tastiera fisica | **Corretto 2026-04-14** | `ring_keypad_input_enter` sostituito con chiamata a `validate_and_arm mode=armed_sera`; `ring_keypad_command_to_allarme_core` esteso con `armed_sera` nel trigger `to:`, caso `armed_sera→profilo sera` nel blocco `arming` e nuovo case safety fallback per `armed_sera` |
| RK-42 | Nessuna verifica sensori aperti prima dell'armo da tastiera: il sistema armava immediatamente anche con porte/finestre aperte nelle zone del profilo target — sensori escludevano zone per il tempo di esclusione, ma l'utente non aveva visibilità della situazione | **Corretto 2026-04-14** | Nuovo script `ring_keypad_arm_or_challenge`: controlla `binary_sensor.allarme_core_*` con `state=on` e `attributes.abilitato=true` nelle zone del profilo target; se tutto ok → `do_arm`; se sensori aperti → `bypass_pending=on` + `bypass_challenge` + timer `bypass_timeout_script`. Conferma bypass: il successivo `validate_and_arm` con stessa tastiera+mode+PIN valido chiama `bypass_cancel` + `do_arm`. PIN sempre obbligatorio per la conferma bypass. `validate_and_arm` ristrutturato a 2 casi. |
| RK-43 | Tasti ARM nocode (5/0, 6/0, 2/0) bypassavano il check sensori: chiamavano `do_arm` direttamente senza passare da `arm_or_challenge` — con `require_code_to_arm=off` il bypass poteva partire anche se c'erano sensori aperti, e non era possibile confermare il bypass senza PIN | **Corretto 2026-04-14** | Le 3 automazioni nocode (`arm_away_nocode`, `arm_home_nocode`, `enter_nocode`) ora chiamano `arm_or_challenge` invece di `do_arm`. Aggiunta guardia: se `bypass_pending=on` per quella tastiera → `code_rejected` (la conferma bypass richiede sempre PIN, quindi senza codice è impossibile). |
| RK-44 | `bypass_pending` non veniva resettato al disarmo da dashboard/esterno: se l'utente disarmava da allarme-core mentre un bypass era in attesa, il flag rimaneva `on` — al successivo tentativo di armo il sistema entrava nel CASO 1 di `validate_and_arm` con dati inconsistenti | **Corretto 2026-04-14** | Aggiunto `script.ring_keypad_bypass_cancel` nel ramo `disarmed` di `ring_keypad_sync_from_allarme_core`, subito dopo `turn_off ring_keypad_do_arm`. |
| RK-45 | **CRITICO** — `ring_keypad_bypass_timeout_script` race condition: al timeout chiamava `sync_state` prima di resettare `bypass_pending` — se l'utente premeva un tasto durante l'esecuzione di `sync_state`, `validate_and_arm` trovava ancora `bypass_pending=on` e interpretava l'azione come conferma bypass (CASO 1), armando con il vecchio `bypass_mode` anche se l'utente aveva intenzionato un armo normale | **Corretto 2026-04-14** | Ordine invertito nel timeout script: inline-reset (`turn_off bypass_pending` + clear `bypass_mode` + clear `bypass_keypad`) eseguito **prima** di chiamare `sync_state`; in questo modo `bypass_pending=off` è garantito prima che eventuali input possano essere processati |
| RK-46 | **CRITICO** — `validate_and_arm` con bypass_pending attivo + tasto diverso da V: premere ARM HOME/AWAY + PIN valido durante un bypass in corso chiamava `arm_or_challenge` → generava un secondo challenge sovrapposto al primo, con `bypass_mode` sovrascritto e timer riavviato — il sistema perdeva il contesto del bypass originale | **Corretto 2026-04-14** | Ristrutturato `validate_and_arm` da 2 a **3 casi**: CASO 2a (bypass_pending=on + stessa tastiera + mode≠`armed_sera`) → `code_rejected` + log "Bypass attivo - usare tasto V + PIN per confermare"; il secondo `arm_or_challenge` non viene mai chiamato. La conferma bypass è sempre e solo via tasto V. |
| RK-47 | **CRITICO** — `ring_keypad_do_arm` ha `mode: single, max_exceeded: silent`: se al momento della conferma bypass `do_arm` era ancora in esecuzione (residuo di una chiamata precedente), la nuova chiamata di `do_arm` da conferma bypass veniva scartata silenziosamente — sistema restava in `disarmed` senza alcun feedback | **Corretto 2026-04-14** | Aggiunto `script.turn_off ring_keypad_do_arm` come passo esplicito in `validate_and_arm` CASO 1, prima di `bypass_cancel` e della nuova chiamata a `do_arm`; garantisce che l'armo post-bypass parta sempre da zero |
| RK-48 | **MEDIO** — Commento in `ring_keypad_integration.yaml` descriveva il trigger `armed_sera` come `sera→disarmed` (legacy dall'implementazione precedente dove V faceva solo log) — documentazione interna ingannevole | **Corretto 2026-04-14** | Commento corretto in `sera→armed_sera` nel blocco automazione `ring_keypad_command_to_allarme_core` |
| RK-49 | **MEDIO** — `ring_keypad_sync_from_safety_core` rami triggered (fuoco/acqua): non fermavano `ring_keypad_do_arm` — se un allarme safety scattava durante l'exit delay, `do_arm` completava il countdown e impostava `armed_home`/`armed_away` sovrascrivendo `triggered_fire`/`triggered_water`; allarme-core veniva poi ri-armato, mascherando l'allarme attivo | **Corretto 2026-04-14** | Aggiunto `script.turn_off ring_keypad_do_arm` come primo passo in entrambi i rami triggered di `ring_keypad_sync_from_safety_core` (fuoco e acqua) |
| RK-51 | **MINOR** — Guard bypass in ARM HOME/AWAY (con e senza codice): `code_rejected` inviato correttamente ma `ring_keypad_last_event` non aggiornato — beep di rifiuto senza traccia nel log | **Corretto 2026-04-14** | Aggiunto `input_text.set_value ring_keypad_last_event = "Bypass attivo - premere V per confermare"` in tutti e quattro i branch bypass: `ring_keypad_input_arm_away` (5/2), `ring_keypad_input_arm_home` (6/2), `ring_keypad_input_arm_away_nocode` (5/0), `ring_keypad_input_arm_home_nocode` (6/0) |
| RK-50 | **UX/Refactor** — Logica bypass accoppiata a `validate_and_arm`: CASO 1/2a/2b rendevano lo script complesso e il PIN era richiesto anche quando non necessario. Il bypass non appartiene alla validazione PIN. | **Corretto 2026-04-14** | Nuovo script `ring_keypad_bypass_confirm`: unica responsabilità, conferma bypass senza PIN. `validate_and_arm` semplificato a puro PIN→arm. Le automazioni V (2/2 e 2/0) intercettano il bypass prima, chiamano `bypass_confirm`. Le automazioni ARM HOME/AWAY (5/2, 6/2) aggiungono guard bypass→`code_rejected` prima di `validate_and_arm`. |
| RK-53 | **MEDIO** — `Timeout_Display_on_Status_Change` non configurabile via MQTT su questo dispositivo: il tentativo di impostarlo in `ring_keypad_bypass_challenge` non aveva effetto, introducendo una chiamata MQTT inutile. L'indicatore hardware scompare secondo il valore Z-Wave salvato in NVRAM (non modificabile da software). La finestra di conferma bypass deve essere gestita interamente via software. | **Corretto 2026-04-15** | Rimosso il publish `Timeout_Display_on_Status_Change` da `ring_keypad_bypass_challenge`. La finestra è gestita da `ring_keypad_bypass_timeout_script` (delay software): anche dopo che l'indicatore hardware scompare, premere V entro `ring_keypad_bypass_timeout` secondi arma il sistema. Range aggiornato a 5-60s (non più vincolato al limite hardware 30s). |
| RK-52 | **CRITICO** — Bypass loop con `require_code_to_arm=off`: `bypass_confirm` chiama `bypass_cancel` (bypass_pending=off) PRIMA che `do_arm` imposti lo stato su `arming`. In quella finestra, un messaggio `2/0` duplicato dall'hardware o premuto una seconda volta trovava bypass_pending=off → Branch 2 (`require_code_to_arm=off`) → `arm_or_challenge` con sensori aperti → nuovo bypass challenge. Il sistema sembrava "riprovare l'inserimento sera" all'infinito. | **Corretto 2026-04-15** | Guardia doppia al Branch 2 di `ring_keypad_input_enter_nocode`: (1) `not is_state('script.ring_keypad_bypass_confirm', 'on')` — blocca il branch mentre `bypass_confirm` è in esecuzione, copre la race window tra `bypass_cancel` e `do_arm`; (2) `ring_keypad_alarm_state == 'disarmed'` — blocca il branch durante arming/armed. |
| RK-55 | **BUG** — Tasto X non interrompeva il bypass: `ring_keypad_input_cancel` ascoltava solo `25/2` (X + codice). Durante un bypass attivo l'utente preme X senza inserire codice → la tastiera emette `25/0`, evento non gestito → bypass continua fino al timeout software. | **Corretto 2026-04-15** | Aggiunto secondo trigger `25/0` all'automazione `ring_keypad_input_cancel`. Ora sia X+codice che X senza codice cancellano il bypass attivo e ripristinano i LED. |
| RK-56 | **BUG** — Doppio comando MQTT `Exit_Delay` durante l'armo: `ring_keypad_do_arm` impostava lo stato su `arming` (triggera `sync_on_state_change` → `sync_all` → `sync_state` → `show_exit_delay`) e poi chiamava `show_exit_delay` anche direttamente sulla tastiera. Risultato: la tastiera riceveva il countdown due volte in rapida successione. | **Corretto 2026-04-15** | Rimossa la chiamata diretta a `show_exit_delay` da `ring_keypad_do_arm`. Il cambio stato su `arming` innesca automaticamente `sync_on_state_change` → `sync_all` → `show_exit_delay` su TUTTE le tastiere attive, il che è anche più corretto (multi-tastiera). |
| RK-54 | **UX** — L'indicatore hardware `Bypass_challenge` scompare dopo ~5s (timeout gestito dall'NVRAM del dispositivo, non modificabile via MQTT): l'utente non aveva più feedback visivo per la maggior parte della finestra di bypass software, rendendo difficile capire che poteva ancora premere V. | **Implementato 2026-04-15** | Nuovo script `ring_keypad_bypass_keepalive` (`mode: restart`): loop `repeat/while` che ogni 5s invia `Bypass_challenge/9` payload `1` (**sub_cmd 9 = solo LED, nessuna voce** — sub_cmd 1 attiva sempre la voce indipendentemente dal payload) finché `bypass_pending=on` AND `bypass_keypad==keypad_id`. `ring_keypad_bypass_challenge` lo avvia subito dopo il primo invio sub_cmd 1 payload `99`. `ring_keypad_bypass_cancel` lo ferma insieme a `bypass_timeout_script`. |
| C-2 | **CRITICO** — `trigger.payload_json` è `None` se il payload MQTT non è JSON valido (es. messaggio corrotto, retained obsoleto). Chiamare `.get(...)` su `None` causa `UndefinedError` e l'automazione fallisce silenziosamente: l'evento viene perso. | **Corretto 2026-04-17** | Sostituito `trigger.payload_json.get('value', '')` con `(trigger.payload_json \| default({})).get('value', '')` nelle 4 automazioni con codice: `ring_keypad_input_disarm`, `ring_keypad_input_arm_away`, `ring_keypad_input_arm_home`, `ring_keypad_input_enter`. |
| C-3 | **CRITICO** — `ring_keypad_do_arm` con `mode: single` e `max_exceeded: silent`: durante l'exit delay (30s), qualsiasi richiesta di cambio profilo (es. Arm Home → Arm Away) viene scartata silenziosamente. Il sistema arma nel profilo originale senza feedback. | **Corretto 2026-04-17** | Cambiato `mode: single` → `mode: restart` e rimosso `max_exceeded: silent`. Con `mode: restart`, premere un nuovo profilo durante l'exit delay riavvia lo script con il nuovo profilo (comportamento desiderato). |
| C-4 | **CRITICO** — PIN duplicati tra utenti non rilevati: `ring_keypad_resolve_user` valuta sequenzialmente, quindi se due utenti condividono lo stesso PIN, il log viene sempre attribuito all'utente con priorità più alta (master per primo). Nessun warning visibile. | **Implementato 2026-04-17** | Aggiunto template `binary_sensor.keypad_collisione_pin` in `ring_keypad_globals.yaml`: `on` se due o più PIN configurati (≥4 cifre) sono identici. Da mostrare in dashboard come warning. |
| C-5 | **CRITICO** — PIN < 4 cifre configurati dall'utente vengono silenziosamente ignorati (guard `length >= 4`). L'utente crede di avere un PIN attivo ma non funziona mai. Nessuna validazione UI. | **Corretto 2026-04-17** | Aggiunto `pattern: "^$\|^[0-9]{4,8}$"` a tutti i 4 `input_text` PIN in `ring_keypad_globals.yaml`. HA mostra errore UI se il PIN non rispetta il pattern (vuoto OK, oppure 4-8 cifre numeriche). |
| C-6 | **CRITICO** — Nessuna protezione brute-force su PIN (RK-03 aperto): nessun rate limiting, tentativi illimitati. Un attaccante con accesso fisico può provare decine di PIN al minuto. | **Implementato 2026-04-17** | Sistema completo: `input_number.ring_keypad_failed_attempts` (contatore interno, `initial: 0`), `input_number.ring_keypad_max_failed_attempts` (soglia configurabile, 3-10), `input_boolean.ring_keypad_lockout_active` (flag lockout). `validate_and_disarm` e `validate_and_arm` incrementano il contatore su PIN errato, attivano lockout al raggiungimento della soglia, resettano il contatore su PIN corretto. In lockout: `code_rejected` immediato (indistinguibile da PIN errato, per non rivelare il lockout). Automazione `ring_keypad_lockout_reset`: reset automatico dopo 5 min. Automazione `ring_keypad_failed_attempts_reset`: sliding window — reset contatore dopo 60s di inattività se lockout non attivo. |

---

## Sezione 13 — Sistema Log

### Architettura

Il sistema log è implementato in `ring_keypad_log.yaml` (**Opzione A** — zero impatto sui file esistenti). Il file è autonomo: non modifica nessuno degli altri 5 package.

### Entità helper

| Entità | Tipo | Descrizione |
|--------|------|-------------|
| `input_text.ring_keypad_logbook_event` | fittizia | Usata come entity_id in `logbook.log` per filtrare la Logbook card in UI |
| `input_text.ring_keypad_logbook_ultimo_evento` | trigger | Aggiornata da `logbook_emit`; osservata dall'automazione timeline append |
| `input_text.ring_keypad_nota_manuale` | UI | Campo testo per note manuali da dashboard |

### Script

| Script | Descrizione |
|--------|-------------|
| `ring_keypad_logbook_emit` | `msg` → `logbook.log` + update `ultimo_evento` |
| `ring_keypad_logbook_nota` | Prefissa `📝 Nota:` e chiama `logbook_emit` |
| `ring_keypad_logbook_nota_da_ui` | Legge `ring_keypad_nota_manuale`, chiama `logbook_emit`, svuota il campo |
| `ring_keypad_logbook_cancella_tutto` | Azzera le 4 `var.*` timeline (globale + 3 sezioni) |

### Sorgenti di log

Il log è completamente autonomo grazie a tre sorgenti di osservazione:

| ID Automazione | Trigger | Copertura |
|----------------|---------|-----------|
| `ring_keypad_log_da_last_event` | `input_text.ring_keypad_last_event` | Tutte le azioni da tastiera: arm, disarm, PIN errato, bypass, tasto X, ANTI-RAPINA, MEDICAL, FIRE |
| `ring_keypad_log_da_alarm_state_esterno` | `ring_keypad_alarm_state` → `triggered_burglar`, `triggered_water` | Allarmi da sistemi esterni (allarme-core → burglar; safety-core → acqua) — questi stati non hanno corrispondente scrittura su `last_event` |
| `ring_keypad_log_tastiera_online` | MQTT `zwave2mqtt/+/status` | Tastiera tornata online (payload `status = Alive`) |
| `ring_keypad_log_timeline_append` | `input_text.ring_keypad_logbook_ultimo_evento` | Append a `var.*_timeline_md` (globale + sezione appropriata) |

### Decorazione emoji

| Contenuto messaggio | Emoji | Sezione timeline |
|---------------------|-------|-----------------|
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

### Var timeline

Dichiarate in `ring-keypad/var/ring_keypad.yaml` (da aggiungere al `var.yaml` di HA):

| Var | Uso |
|-----|-----|
| `var.ring_keypad_timeline_md` | Timeline globale — letta dal template sensor "Timeline Render" |
| `var.ring_keypad_timeline_critici_md` | Sezione critici — popup dashboard |
| `var.ring_keypad_timeline_operazioni_md` | Sezione operazioni — popup dashboard |
| `var.ring_keypad_timeline_note_md` | Sezione note — popup dashboard |

Ogni var mantiene al massimo 25 righe (le più recenti).

### Template sensor

| Sensor | `unique_id` | Descrizione |
|--------|-------------|-------------|
| `Ring Keypad – Timeline Render` | `ring_keypad_timeline_render` | Legge `var.ring_keypad_timeline_md`, colora le righe per tipo, espone in `long_state` |
| `Ring Keypad – Count Critici` | `ring_keypad_count_critici` | Conta le righe in `var.ring_keypad_timeline_critici_md` |
| `Ring Keypad – Count Operazioni` | `ring_keypad_count_operazioni` | Conta le righe in `var.ring_keypad_timeline_operazioni_md` |
| `Ring Keypad – Count Note` | `ring_keypad_count_note` | Conta le righe in `var.ring_keypad_timeline_note_md` |

### Dashboard (plancia_controllo.yaml)

La sezione "Log & Attività" è stata estesa con:
- **Pulsante Registro** — popup con 3 markdown card (Critici / Operazioni / Note) scrollabili
- **Tile Cancella Log** — chiama `ring_keypad_logbook_cancella_tutto`
- **Tile Aggiungi Nota** — popup con campo testo + pulsante salva
- **`custom:timeline-card`** — storico visivo 24h di `ring_keypad_alarm_state` e `last_user`
- **Tile Ultimo Evento Log** — mostra `ring_keypad_logbook_ultimo_evento`

### Installazione

1. Copiare `packages/ring_keypad_log.yaml` in `config/packages/ring_keypad/` (o nella directory packages HA)
2. Aggiungere il contenuto di `var/ring_keypad.yaml` al `var.yaml` esistente in HA
3. Riavviare HA
4. Il log parte immediatamente a raccogliere eventi senza configurazione aggiuntiva
