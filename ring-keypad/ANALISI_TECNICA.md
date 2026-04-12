# Ring Keypad V2 — Analisi Tecnica

> Ultima revisione: 2026-04-12  
> Versione architettura: 3.0 (protocollo Z-Wave corretto, voice feedback, LED mode, anti-rapina, water)

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
│   └── ring_keypad_integration.yaml   # Bridge bidirezionale verso sistemi esterni
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
| 2 | 2 | Tasto V (Enter) + codice — solo log |
| 25 | 2 | Tasto X (Cancel) + codice — solo log |
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
| timeout | Countdown formato stringa | `"{min}m{sec}s"` es. `"0m30s"` |

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

1. Aggiungere il suo topic base a `input_text.ring_keypad_active_keypads` dalla UI HA
2. Decommentare e adattare il blocco sensore in `ring_keypad_keypads.yaml`  
3. **Nessun'altra modifica necessaria**: le automazioni con wildcard e gli script parametrici gestiscono tutto automaticamente

---

## Sezione 4 — Entità Helper

### `input_select`

| Entità | Valori possibili | Descrizione |
|--------|-----------------|-------------|
| `input_select.ring_keypad_alarm_state` | disarmed, armed_home, armed_away, arming, pending, triggered_burglar, triggered_rapina, triggered_fire, triggered_medical, triggered_water | Stato interno del keypad (unica fonte di verità per gli indicatori) |
| `input_select.ring_keypad_led_mode` | Spenti, Lampeggio (3s), Sempre accesi | Controlla System_Security_Mode_Display sulla tastiera |

### `input_text`

| Entità | Descrizione | Note |
|--------|-------------|------|
| `ring_keypad_active_keypads` | Elenco topic base tastiere (CSV) | Modifica dalla UI |
| `ring_keypad_last_user` | Ultimo utente che ha agito | Log |
| `ring_keypad_last_event` | Descrizione ultimo evento | Log |
| `ring_keypad_last_keypad` | Topic base dell'ultima tastiera usata | Log |
| `ring_keypad_pin_master` | PIN Master | mode: password |
| `ring_keypad_pin_user1..3` | PIN Utenti 1-3 | mode: password |
| `ring_keypad_name_master` | Nome Master | Per log |
| `ring_keypad_name_user1..3` | Nomi Utenti 1-3 | Per log |

### `input_number`

| Entità | Default | Descrizione |
|--------|---------|-------------|
| `ring_keypad_exit_delay` | 30s | Countdown uscita (arming → armed) |
| `ring_keypad_entry_delay` | 30s | Countdown entrata (pending → triggered o disarm) |

### `input_boolean`

| Entità | Default | Descrizione |
|--------|---------|-------------|
| `ring_keypad_debug` | false | Modalità debug |
| `ring_keypad_require_code_to_arm` | false | Se true, richiede PIN anche per armare |
| `ring_keypad_voice_feedback` | true | Se on: payload 99 (LED+suono) per disarmed/armed; se off: payload 1 (solo LED) |

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
| `ring_keypad_code_rejected` | keypad_id | `Code_not_accepted/9` payload 99 | Sempre con suono |
| `ring_keypad_bypass_challenge` | keypad_id | `Bypass_challenge/1` payload 99 | sub_cmd 1! |
| `ring_keypad_show_exit_delay` | keypad_id | `Exit_Delay/timeout` stringa `Xm Ys` | — |
| `ring_keypad_show_entry_delay` | keypad_id | `Entry_Delay/timeout` stringa `Xm Ys` | — |
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

### Script di sincronizzazione

| Script | Parametri | Descrizione |
|--------|-----------|-------------|
| `ring_keypad_sync_state` | keypad_id | Dispatcher: invia indicatore corretto in base a `ring_keypad_alarm_state` (10 stati) |
| `ring_keypad_sync_all` | — | Chiama `sync_state` su tutte le tastiere in `ring_keypad_active_keypads` |
| `ring_keypad_broadcast_alarm` | alarm_type | Invia allarme specifico a tutte le tastiere (burglar/rapina/fire/medical/water) |

### Script di logica

| Script | Parametri | Descrizione |
|--------|-----------|-------------|
| `ring_keypad_resolve_user` | code | Identifica l'utente dal PIN, aggiorna `ring_keypad_last_user` |
| `ring_keypad_validate_and_disarm` | code, keypad_id | Valida PIN → disarma o rifiuta |
| `ring_keypad_validate_and_arm` | code, mode, keypad_id | Valida PIN → arma o rifiuta |
| `ring_keypad_do_disarm` | keypad_id | Imposta stato `disarmed` + log |
| `ring_keypad_do_arm` | mode, keypad_id | Imposta `arming` → delay exit → imposta `mode` + log |

---

## Sezione 7 — Automazioni

### Automazioni input (da tastiera)

| ID Automazione | Trigger MQTT | Azione |
|----------------|-------------|--------|
| `ring_keypad_input_disarm` | `zwave2mqtt/+/unknownClass_111/endpoint_0/3/2` | `validate_and_disarm` |
| `ring_keypad_input_arm_away` | `zwave2mqtt/+/unknownClass_111/endpoint_0/5/2` | `validate_and_arm mode=armed_away` |
| `ring_keypad_input_arm_away_nocode` | `zwave2mqtt/+/unknownClass_111/endpoint_0/5/0` | `do_arm` o `code_rejected` |
| `ring_keypad_input_arm_home` | `zwave2mqtt/+/unknownClass_111/endpoint_0/6/2` | `validate_and_arm mode=armed_home` |
| `ring_keypad_input_arm_home_nocode` | `zwave2mqtt/+/unknownClass_111/endpoint_0/6/0` | `do_arm` o `code_rejected` |
| `ring_keypad_input_enter` | `zwave2mqtt/+/unknownClass_111/endpoint_0/2/2` | Aggiorna log "Tasto V" |
| `ring_keypad_input_cancel` | `zwave2mqtt/+/unknownClass_111/endpoint_0/25/2` | Aggiorna log "Tasto X" |
| `ring_keypad_input_rapina` | `zwave2mqtt/+/unknownClass_111/endpoint_0/17/0` | Stato `triggered_rapina` + broadcast rapina |
| `ring_keypad_input_medical` | `zwave2mqtt/+/unknownClass_111/endpoint_0/19/0` | Stato `triggered_medical` + broadcast |
| `ring_keypad_input_fire` | `zwave2mqtt/+/unknownClass_111/endpoint_0/16/0` | Stato `triggered_fire` + broadcast |

### Automazioni di sistema

| ID Automazione | Trigger | Azione |
|----------------|---------|--------|
| `ring_keypad_sync_on_state_change` | State: `ring_keypad_alarm_state` | `sync_all` |
| `ring_keypad_restore_on_connect` | MQTT: `zwave2mqtt/+/status` = Alive | `sync_state` + `set_led_mode` su tastiera tornata online |
| `ring_keypad_sync_on_ha_start` | HA start | Delay 15s → `sync_all` + `set_led_mode_all` |
| `ring_keypad_sync_led_on_mode_change` | State: `ring_keypad_led_mode` | `set_led_mode_all` |

### Automazioni di integrazione (in `ring_keypad_integration.yaml`)

| ID Automazione | Trigger | Azione |
|----------------|---------|--------|
| `ring_keypad_sync_from_allarme_core` | State: `allarme_core_stato` | Aggiorna `ring_keypad_alarm_state` |
| `ring_keypad_command_to_allarme_core` | State: `ring_keypad_alarm_state` → disarmed/armed_* | Chiama script arm/disarm allarme-core |

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
    │        ring_keypad_do_arm (mode, keypad_id)
    │            ↓
    │        ring_keypad_alarm_state = "arming"
    │            ↓ (sync automatico via ring_keypad_sync_on_state_change)
    │        ring_keypad_sync_all → show_exit_delay su tutte le tastiere
    │            ↓ (delay exit_delay secondi)
    │        ring_keypad_alarm_state = "armed_away"
    │            ↓ (sync automatico)
    │        ring_keypad_sync_all → show_armed_away su tutte le tastiere
    │            ↓ (via ring_keypad_command_to_allarme_core)
    │        script.arm_allarme_core
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

**Mappatura stati:**

| allarme-core (`allarme_core_stato`) | ring_keypad (`ring_keypad_alarm_state`) |
|-------------------------------------|----------------------------------------|
| `disarmed` | `disarmed` |
| `arming` | `arming` |
| `armed` | `armed_away` |
| `triggered` | `triggered_burglar` |

**Comandi dal keypad verso allarme-core:**

| ring_keypad stato | Azione su allarme-core |
|-------------------|------------------------|
| `disarmed` | `script.disarm_allarme_core` |
| `armed_home` | `script.arm_allarme_core` (allarme-core non distingue home/away) |
| `armed_away` | `script.arm_allarme_core` |

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

1. Aggiungere il topic base a `ring_keypad_active_keypads` separato da virgola
2. In `ring_keypad_keypads.yaml`: decommentare e adattare il blocco sensore template
3. Nessun'altra modifica necessaria

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

### Stato `armed_home` vs allarme-core

allarme-core ha un solo stato armato (`armed`). Il keypad supporta `armed_home` e `armed_away`.
La mappatura corrente: entrambi i modi dal keypad chiamano `script.arm_allarme_core`.
Per un'integrazione futura con sistemi che distinguono i due modi, la struttura dell'integration file supporta già la biforcazione.

### `ring_keypad_do_arm` e delay interno

Il delay di exit delay è interno allo script. Se HA si riavvia durante il delay, lo script termina ma lo stato rimane `arming`. Al riavvio, `ring_keypad_sync_on_ha_start` re-sincronizza i LED. Lo stato interno va corretto manualmente (o via integrazione esterna che riporta lo stato corretto).

---

## Sezione 12 — Bug e Limitazioni Conosciute

| # | Descrizione | Stato | Fix |
|---|-------------|-------|-----|
| RK-01 | `armed_home` mappato a `arm_allarme_core` (stesso di armed_away) | Aperto | Per sistemi con distinzione, personalizzare `ring_keypad_integration.yaml` |
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
| RK-16 | Exit/Entry Delay: payload era un intero in secondi (sub_cmd 7) invece della stringa `Xm Ys` richiesta dal campo `timeout` | **Corretto 2026-04-12** | Script aggiornati con calcolo `(s // 60)m(s % 60)s` |
| RK-17 | `ring_keypad_alarm_burglar` puntava a `Alarming_Burglar` (anti-rapina silenzioso) invece di `Alarming` (intrusione con suono) | **Corretto 2026-04-12** | `alarm_burglar` → `Alarming/9` payload 99; nuovo `alarm_rapina` → `Alarming_Burglar/9` payload 1 |
| RK-18 | Automazione `ring_keypad_input_cancel`: data_type era `0` invece di `2` (il manuale documenta il Cancel con codice come 25/2) | **Corretto 2026-04-12** | Topic aggiornato a `25/2` |
| RK-19 | Automazione `ring_keypad_input_police`: rinominata in `ring_keypad_input_rapina`, stato cambiato da `triggered_burglar` a `triggered_rapina` | **Corretto 2026-04-12** | Distinzione semantica: burglar=intrusione esterna, rapina=minaccia con presenza |
