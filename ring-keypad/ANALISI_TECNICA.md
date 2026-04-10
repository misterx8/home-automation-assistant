# Ring Keypad V2 — Analisi Tecnica

> Ultima revisione: 2026-04-10  
> Versione architettura: 2.0 (multi-tastiera, parametrica)

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

Topic ricevuti: `zwave2mqtt/{keypad_id}/111/0/{event_type}/{data_type}`

| event_type | data_type | Significato |
|------------|-----------|-------------|
| 2 | 2 | Tasto Enter + codice |
| 3 | 2 | Tasto Disarm + codice |
| 5 | 2 | Tasto Arm Away + codice |
| 5 | 0 | Tasto Arm Away senza codice |
| 6 | 2 | Tasto Arm Home + codice |
| 6 | 0 | Tasto Arm Home senza codice |
| 16 | 0 | Tasto Fire (tenuto premuto) |
| 17 | 0 | Tasto Police (tenuto premuto) |
| 19 | 0 | Tasto Medical (tenuto premuto) |
| 25 | 0 | Tasto Cancel |

Il payload è JSON: `{"value": "1234"}` per eventi con codice, `{}` per eventi senza.

### Indicator CC (135) — Output verso la tastiera

Topic pubblicati: `zwave2mqtt/{keypad_id}/135/0/{indicator}/{sub_cmd}/set`

| sub_cmd | Significato | Payload |
|---------|-------------|---------|
| 1 | On/Off indicatore | 0 (off) o 99 (on) |
| 7 | Countdown secondi | numero intero |
| 9 | Livello suono/LED | 0-99 |

**Indicatori disponibili:**

| Indicatore | Descrizione |
|------------|-------------|
| `Not_armed__disarmed` | Sistema disarmato |
| `Armed_Stay` | Armato home |
| `Armed_Away` | Armato away |
| `Code_not_accepted` | Codice errato (bip di errore) |
| `Bypass_challenge` | Richiesta bypass |
| `Exit_Delay` | Countdown uscita (sub_cmd=7) |
| `Entry_Delay` | Countdown entrata (sub_cmd=7) |
| `Alarming_Burglar` | Allarme intrusione |
| `Alarming_Smoke__Fire` | Allarme fuoco |
| `Alarming_Carbon_Monoxide` | Allarme CO |
| `Alarming_Medical` | Allarme medico |
| `Generic_event_sound_notification_1` | Beep generico |
| `Generic_event_sound_notification_4` | Chime |

### Status nodo

- Topic: `zwave2mqtt/{keypad_id}/status`
- Payload JSON: `{"status": "Alive"}` o `{"status": "Dead"}`

### Batteria

- Topic: `zwave2mqtt/{keypad_id}/battery/0/isLow`
- Payload JSON: `{"value": true}` (scarica) o `{"value": false}` (carica)

---

## Sezione 3 — Principio Multi-Tastiera (Wildcard MQTT)

Il supporto multi-tastiera si basa su **due meccanismi**:

### 3.1 Ricezione (input)

Le automazioni usano wildcard MQTT `+` per catturare eventi da qualsiasi tastiera:

```
zwave2mqtt/+/111/0/3/2   ← Disarm da QUALSIASI tastiera
```

Il `keypad_id` viene estratto automaticamente dal topic:

```yaml
variables:
  keypad_id: "{{ trigger.topic.split('/')[1] }}"
```

### 3.2 Trasmissione (output/feedback)

Gli script accettano `keypad_id` come parametro e costruiscono il topic dinamicamente:

```yaml
topic: "zwave2mqtt/{{ keypad_id }}/135/0/Not_armed__disarmed/1/set"
```

### 3.3 Registro tastiere attive

L'elenco dei topic base è in `input_text.ring_keypad_active_keypads` (CSV):

```
tastiera_camera,tastiera_ingresso,tastiera_garage
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
| `input_select.ring_keypad_alarm_state` | disarmed, armed_home, armed_away, arming, pending, triggered_burglar, triggered_fire, triggered_co, triggered_medical | Stato interno del keypad (unica fonte di verità per gli indicatori) |

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

### Sensori MQTT (per tastiera, dichiarati in `ring_keypad_keypads.yaml`)

| Entità | Tipo | Descrizione |
|--------|------|-------------|
| `sensor.tastiera_<id>_batteria` | sensor | Livello batteria (10% = bassa, 100% = ok) |
| `binary_sensor.tastiera_<id>_online` | binary_sensor | Connettività Z-Wave |

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

| Script | Parametri | Azione MQTT |
|--------|-----------|-------------|
| `ring_keypad_send_indicator` | keypad_id, indicator, sub_cmd, value | Invia payload generico |
| `ring_keypad_show_disarmed` | keypad_id | LED disarmed |
| `ring_keypad_show_armed_home` | keypad_id | LED armed stay |
| `ring_keypad_show_armed_away` | keypad_id | LED armed away |
| `ring_keypad_code_rejected` | keypad_id | Bip errore codice |
| `ring_keypad_bypass_challenge` | keypad_id | Richiesta bypass |
| `ring_keypad_show_exit_delay` | keypad_id | Countdown exit delay |
| `ring_keypad_show_entry_delay` | keypad_id | Countdown entry delay |
| `ring_keypad_alarm_burglar` | keypad_id | LED/suono allarme intrusione |
| `ring_keypad_alarm_fire` | keypad_id | LED/suono allarme fuoco |
| `ring_keypad_alarm_co` | keypad_id | LED/suono allarme CO |
| `ring_keypad_alarm_medical` | keypad_id | LED/suono allarme medico |
| `ring_keypad_sound_beep` | keypad_id | Beep generico |
| `ring_keypad_sound_chime` | keypad_id | Chime |

### Script di sincronizzazione

| Script | Parametri | Descrizione |
|--------|-----------|-------------|
| `ring_keypad_sync_state` | keypad_id | Dispatcher: invia indicatore corretto in base a `ring_keypad_alarm_state` |
| `ring_keypad_sync_all` | — | Chiama `sync_state` su tutte le tastiere in `ring_keypad_active_keypads` |
| `ring_keypad_broadcast_alarm` | alarm_type | Invia allarme specifico a tutte le tastiere |

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
| `ring_keypad_input_disarm` | `zwave2mqtt/+/111/0/3/2` | `validate_and_disarm` |
| `ring_keypad_input_enter` | `zwave2mqtt/+/111/0/2/2` | `validate_and_disarm` |
| `ring_keypad_input_arm_away` | `zwave2mqtt/+/111/0/5/2` | `validate_and_arm mode=armed_away` |
| `ring_keypad_input_arm_away_nocode` | `zwave2mqtt/+/111/0/5/0` | `do_arm` o `code_rejected` |
| `ring_keypad_input_arm_home` | `zwave2mqtt/+/111/0/6/2` | `validate_and_arm mode=armed_home` |
| `ring_keypad_input_arm_home_nocode` | `zwave2mqtt/+/111/0/6/0` | `do_arm` o `code_rejected` |
| `ring_keypad_input_cancel` | `zwave2mqtt/+/111/0/25/0` | Aggiorna log |
| `ring_keypad_input_fire` | `zwave2mqtt/+/111/0/16/0` | Stato `triggered_fire` + broadcast |
| `ring_keypad_input_police` | `zwave2mqtt/+/111/0/17/0` | Stato `triggered_burglar` + broadcast |
| `ring_keypad_input_medical` | `zwave2mqtt/+/111/0/19/0` | Stato `triggered_medical` + broadcast |

### Automazioni di sistema

| ID Automazione | Trigger | Azione |
|----------------|---------|--------|
| `ring_keypad_sync_on_state_change` | State: `ring_keypad_alarm_state` | `sync_all` |
| `ring_keypad_restore_on_connect` | MQTT: `zwave2mqtt/+/status` = Alive | `sync_state` su tastiera tornata online |
| `ring_keypad_sync_on_ha_start` | HA start | Delay 15s → `sync_all` |

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

| # | Descrizione | Workaround |
|---|-------------|------------|
| RK-01 | `armed_home` mappato a `arm_allarme_core` (stesso di armed_away) | Per sistemi con distinzione, personalizzare `ring_keypad_integration.yaml` |
| RK-02 | Delay in `do_arm` non interrompibile dall'esterno | Nessuno (limitazione HA scripts) |
| RK-03 | Nessuna protezione brute-force su PIN | Implementazione futura con `counter` helper |
| RK-04 | `ring_keypad_sync_on_state_change` si attiva anche per transizioni `→unknown` | Accettato: il sync_all è idempotente |
