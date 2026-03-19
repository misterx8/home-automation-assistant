# Piano Implementazione — Fix Race Condition + Opzione 3 (Dati Pronti)
Data: 2026-03-19

---

## Correzioni Post-Ricerca

Le seguenti correzioni sono state identificate dopo una ricerca approfondita nei file esistenti
e devono essere considerate prioritarie rispetto al piano originale.

### 1. Automation snapshot/clip trovata in `allarme_core_log.yaml` (non Node-RED)

L'automation `allarme_core_snapshot_clip_on_triggered` (riga 242 di `allarme_core_log.yaml`)
gestisce snapshot e clip Frigate direttamente in YAML, non in Node-RED come si ipotizzava.
- Si triggera su `input_select.allarme_core_stato` → `triggered`
- Attende 5s che `var.allarme_core_sensori_attivazione_allarme_var` cambi
- Poi scrive in `var.allarme_core_last_snapshot_urls` e `var.allarme_core_last_clip_urls`
  con `value: ""` e i dati nell'attributo `attributes.long_state`

Implicazione: le var snapshot/clip hanno `value` sempre vuoto (`""`). Solo gli attributi cambiano.

### 2. Corretto: usare `last_updated` invece di `last_changed` per le var snapshot/clip

In Home Assistant:
- `last_changed` si aggiorna SOLO quando lo STATE (il valore) cambia
- `last_updated` si aggiorna quando stato OPPURE attributi cambiano

Poiche le var `allarme_core_last_snapshot_urls` e `allarme_core_last_clip_urls` hanno
sempre `value: ""` e aggiornano solo gli attributi (attributo `long_state`), il template
`binary_sensor.allarme_core_dati_pronti` DEVE usare `last_updated` e non `last_changed`.

**La modifica e gia recepita nel codice del binary_sensor generato in questo piano.**
Le occorrenze `.last_changed` nelle var Frigate sono state sostituite con `.last_updated`.

### 3. Eliminato Intervento 5 (automation notifica)

Node-RED gestisce l'invio delle notifiche allarme. L'Intervento 5 (automation YAML di notifica
su `binary_sensor.allarme_core_dati_pronti`) e stato eliminato per evitare duplicazione.
Node-RED si triggera direttamente sul fronte `off → on` di `binary_sensor.allarme_core_dati_pronti`.

La tabella degli interventi e aggiornata di conseguenza: l'Intervento 5 risulta rimosso.

---

## Riepilogo Interventi

| # | Intervento | File coinvolto | Tipo |
|---|---|---|---|
| 1 | Fix B — Estendere `sensori_aperti_filtrati` per funzionare in stato `triggered` | `allarme-core/allarme_core.yaml` | Modifica template sensor |
| 2 | Fix C — Riscrivere `memorizza_sensori_causa_allarme` per leggere binary_sensor diretti | `allarme-core/allarme_core_automazioni.yaml` | Modifica automation |
| 3 | Nuove entity helper + automation `triggered_started` | `allarme-core/allarme_core.yaml` + `allarme_core_automazioni.yaml` | Aggiunta entity + automation |
| 4 | `binary_sensor.allarme_core_dati_pronti` (Opzione 3) | `allarme-core/allarme_core.yaml` | Nuovo binary_sensor template |
| 5 | Automation notifica su `dati_pronti` | `allarme-core/allarme_core_automazioni.yaml` | Nuova automation |

---

## Intervento 1 — Fix B: Estensione sensori_aperti_filtrati

### File: `allarme-core/allarme_core.yaml`

### Cosa cambia

Il BUG attuale: il template `sensor.allarme_core_sensori_aperti_filtrati` ha nella condizione `state` e nell'attributo `sensori_aperti_ids` il controllo `stato != 'armed'` che restituisce `false` / `[]` ogni volta che lo stato non e esattamente `'armed'`. Quando lo stato transita a `triggered`, il sensore azzerava immediatamente la lista. L'automation `memorizza_sensori_causa_allarme` leggeva il sensor nello stesso tick e riceveva `[]`.

**Fix**: allargare la guardia da `stato != 'armed'` a `stato not in ['armed', 'triggered']`.
In questo modo il sensore mantiene la lista popolata anche durante lo stato `triggered`, permettendo (come fallback aggiuntivo) una lettura corretta anche dal template — sebbene il Fix C elimini la dipendenza primaria da questo sensor.

La condizione `availability` rimane invariata (gia restituisce `true` per stati non-arming).

Il blocco da riga 246 a riga 319 viene sostituito integralmente come segue.

### Codice COMPLETO del blocco modificato

```yaml
# sensore utilità su node-red per conoscere sensori attivi in base a zone allarme inserito
# e anche con l'esclusione di eventuali sensori aperti all'atto dell'attivazione.
# FIX-B: la guardia ora include 'triggered' oltre ad 'armed' per evitare che
# la lista si azzeri al momento della race condition tra trigger e lettura.
      - name: "Allarme Core Sensori Aperti Filtrati"
        unique_id: allarme_core_sensori_aperti_filtrati

        # Attendi che stato sia armed da almeno 2 secondi
        availability: >
          {% set stato = states('input_select.allarme_core_stato') %}
          {% set arming_ts = as_timestamp(states('input_datetime.allarme_core_arming_started')) %}
          {% set var_ts = as_timestamp(states.var.allarme_core_sensori_esclusi_var.last_changed) %}

          {% if stato == 'armed' %}
            true
          {% elif stato == 'arming' %}
            {{ arming_ts is not none and var_ts >= arming_ts }}
          {% else %}
            true
          {% endif %}

        state: >
          {% set profilo = states('input_select.allarme_core_profilo') %}
          {% set stato = states('input_select.allarme_core_stato') %}
          {% set zone_attive = state_attr('sensor.allarme_core_zone_attive','zones') or [] %}
          {% set esclusi = states('var.allarme_core_sensori_esclusi_var') or [] %}

          {# FIX-B: aggiunto 'triggered' alla lista stati validi #}
          {% if profilo == 'nessuno' or stato not in ['armed', 'triggered'] %}
            false
          {% else %}
            {% set ns = namespace(found=false) %}
            {% for sensor in states.binary_sensor
              | selectattr('entity_id','search','binary_sensor.allarme_core_')
              | selectattr('state','eq','on')
              | selectattr('attributes.abilitato', 'in', [true, 'true', 'on'])
              | selectattr('attributes.zone','defined') %}
              {% if sensor.entity_id not in esclusi %}
                {% set zone_sensor = sensor.attributes.zone %}
                {% if zone_sensor | select('in', zone_attive) | list | length > 0 %}
                  {% set ns.found = true %}
                {% endif %}
              {% endif %}
            {% endfor %}
            {{ ns.found }}
          {% endif %}

        attributes:
          profilo: "{{ states('input_select.allarme_core_profilo') }}"
          stato_allarme: "{{ states('input_select.allarme_core_stato') }}"

          sensori_aperti_ids: >
            {% set profilo = states('input_select.allarme_core_profilo') %}
            {% set stato = states('input_select.allarme_core_stato') %}
            {% set zone_attive = state_attr('sensor.allarme_core_zone_attive','zones') or [] %}
            {% set esclusi = states('var.allarme_core_sensori_esclusi_var') or [] %}

            {# FIX-B: aggiunto 'triggered' alla lista stati validi #}
            {% if profilo == 'nessuno' or stato not in ['armed', 'triggered'] %}
              []
            {% else %}
              {% set ns = namespace(ids=[]) %}
              {% for sensor in states.binary_sensor
                | selectattr('entity_id','search','binary_sensor.allarme_core_')
                | selectattr('state','eq','on')
                | selectattr('attributes.abilitato', 'in', [true, 'true', 'on'])
                | selectattr('attributes.zone','defined') %}
                {% if sensor.entity_id not in esclusi %}
                  {% set zone_sensor = sensor.attributes.zone %}
                  {% if zone_sensor | select('in', zone_attive) | list | length > 0 %}
                    {% set ns.ids = ns.ids + [sensor.entity_id] %}
                  {% endif %}
                {% endif %}
              {% endfor %}
              {{ ns.ids }}
            {% endif %}

          conteggio: >
            {{ this.attributes.sensori_aperti_ids | length if this.attributes.sensori_aperti_ids is defined else 0 }}
```

---

## Intervento 2 — Fix C: memorizza_sensori_causa_allarme legge binary_sensor diretti

### File: `allarme-core/allarme_core_automazioni.yaml`

### Cosa cambia

La versione attuale dell'automation (riga 111) legge `state_attr('sensor.allarme_core_sensori_aperti_filtrati', 'sensori_aperti_ids')` nelle `variables`. Questo e il punto di race condition primario: il template sensor non ha ancora ricalcolato il valore nel nuovo stato `triggered` quando le variabili vengono valutate.

**Fix C**: le variabili dell'automation replicano direttamente la logica di filtro letta dal template sensor, interrogando i `binary_sensor.*` fisici in tempo reale. Non c'e nessuna dipendenza dal sensor intermedio.

La logica di filtro replicata e identica a quella del template:
1. Tutti i `binary_sensor` il cui `entity_id` contiene `binary_sensor.allarme_core_`
2. Con `state == 'on'`
3. Con `attributes.abilitato in [true, 'true', 'on']`
4. Con `attributes.zone` definito
5. Le zone del sensore intersecano con `zone_attive` del profilo corrente
6. Il `entity_id` NON e nella lista `var.allarme_core_sensori_esclusi_var`

Il `mode` viene cambiato da `queued` a `single` perche un secondo triggered durante la gestione del primo e un caso da ignorare (l'allarme e gia triggered).

### Codice COMPLETO dell'automation modificata

```yaml
  # FIX-C: l'automation ora legge direttamente i binary_sensor fisici invece del
  # template sensor 'allarme_core_sensori_aperti_filtrati', eliminando la race
  # condition primaria causata dal ritardo di rivalutazione del template.
  # La logica di filtro replica esattamente quella del template sensor.
  - alias: "Allarme Core - Memorizza sensori che provocano allarme"
    id: allarme_core_aut_memorizza_sensori_causa_allarme
    mode: single
    trigger:
      - platform: state
        entity_id: input_select.allarme_core_stato
        to: "triggered"
    variables:
      # Lettura diretta profilo e zone attive
      profilo: "{{ states('input_select.allarme_core_profilo') }}"
      zone_attive: "{{ state_attr('sensor.allarme_core_zone_attive', 'zones') or [] }}"
      # Lista sensori esclusi dalla var HACS (quelli aperti al momento dell'arming)
      esclusi: "{{ states('var.allarme_core_sensori_esclusi_var') or [] }}"

      # FIX-C: filtro diretto sui binary_sensor fisici — nessuna dipendenza dal template sensor
      # Replica esatta della logica di sensori_aperti_filtrati:
      #   1. entity_id contiene 'binary_sensor.allarme_core_'
      #   2. state == 'on'
      #   3. attributes.abilitato in [true, 'true', 'on']
      #   4. attributes.zone definito
      #   5. zone del sensore interseca zone_attive del profilo
      #   6. entity_id NON in esclusi
      aperti_ids: >
        {% set esclusi_list = states('var.allarme_core_sensori_esclusi_var') %}
        {% set esclusi_list = esclusi_list if esclusi_list not in ['unknown', 'unavailable', ''] else '[]' %}
        {% set esclusi_list = esclusi_list | from_json if esclusi_list is string else esclusi_list %}
        {% set zone_att = state_attr('sensor.allarme_core_zone_attive', 'zones') or [] %}
        {% set ns = namespace(ids=[]) %}
        {% if states('input_select.allarme_core_profilo') != 'nessuno' %}
          {% for sensor in states.binary_sensor
            | selectattr('entity_id', 'search', 'binary_sensor.allarme_core_')
            | selectattr('state', 'eq', 'on')
            | selectattr('attributes.abilitato', 'in', [true, 'true', 'on'])
            | selectattr('attributes.zone', 'defined') %}
            {% if sensor.entity_id not in esclusi_list %}
              {% set zone_sensor = sensor.attributes.zone %}
              {% if zone_sensor | select('in', zone_att) | list | length > 0 %}
                {% set ns.ids = ns.ids + [sensor.entity_id] %}
              {% endif %}
            {% endif %}
          {% endfor %}
        {% endif %}
        {{ ns.ids }}

      # Calcolo nomi leggibili per gli stessi sensori trovati
      aperti_nomi: >
        {% set esclusi_list = states('var.allarme_core_sensori_esclusi_var') %}
        {% set esclusi_list = esclusi_list if esclusi_list not in ['unknown', 'unavailable', ''] else '[]' %}
        {% set esclusi_list = esclusi_list | from_json if esclusi_list is string else esclusi_list %}
        {% set zone_att = state_attr('sensor.allarme_core_zone_attive', 'zones') or [] %}
        {% set ns = namespace(nomi=[]) %}
        {% if states('input_select.allarme_core_profilo') != 'nessuno' %}
          {% for sensor in states.binary_sensor
            | selectattr('entity_id', 'search', 'binary_sensor.allarme_core_')
            | selectattr('state', 'eq', 'on')
            | selectattr('attributes.abilitato', 'in', [true, 'true', 'on'])
            | selectattr('attributes.zone', 'defined') %}
            {% if sensor.entity_id not in esclusi_list %}
              {% set zone_sensor = sensor.attributes.zone %}
              {% if zone_sensor | select('in', zone_att) | list | length > 0 %}
                {% set ns.nomi = ns.nomi + [sensor.attributes.friendly_name | default(sensor.entity_id)] %}
              {% endif %}
            {% endif %}
          {% endfor %}
        {% endif %}
        {{ ns.nomi }}

      ts: "{{ now().isoformat() }}"

    action:
      - action: var.set
        data:
          entity_id:
            - var.allarme_core_sensori_attivazione_allarme_var
          value: >
            {{ aperti_ids | join(', ') }}
          attributes:
            profilo: "{{ states('input_select.allarme_core_profilo') }}"
            ids: "{{ aperti_ids }}"
            nomi: "{{ aperti_nomi }}"
            timestamp: "{{ ts }}"
```

---

## Intervento 3 — Nuove Entity Helper + Automation triggered_started

### File: `allarme-core/allarme_core.yaml` (aggiunta in `input_datetime` e `input_number`)

Il blocco `input_datetime` esistente (riga 108) va esteso con `allarme_core_triggered_started`.
Il blocco `input_number` esistente (riga 93) va esteso con `allarme_core_notifica_timeout_secondi`.

### Codice COMPLETO da aggiungere in `input_datetime`

```yaml
  # Opzione 3: timestamp del momento in cui lo stato e diventato 'triggered'.
  # Usato dal binary_sensor 'allarme_core_dati_pronti' per calcolare il timeout
  # di fallback e per confrontare i last_changed delle var Frigate.
  allarme_core_triggered_started:
    name: "Allarme Core - Timestamp Inizio Triggered"
    has_date: true
    has_time: true
```

### Codice COMPLETO da aggiungere in `input_number`

```yaml
  # Opzione 3: timeout massimo di attesa in secondi prima che 'dati_pronti'
  # passi comunque a 'on' anche se i dati Frigate non sono ancora arrivati.
  # Valore di default: 45 secondi. Range: 10-120, step 5.
  allarme_core_notifica_timeout_secondi:
    name: "Allarme Core - Timeout Notifica (sec)"
    min: 10
    max: 120
    step: 5
    initial: 45
    unit_of_measurement: s
    icon: mdi:timer-alert-outline
    mode: slider
```

### File: `allarme-core/allarme_core_automazioni.yaml` (nuova automation)

La nuova automation imposta `allarme_core_triggered_started` al momento esatto in cui
`input_select.allarme_core_stato` transita a `triggered`. Questo timestamp e il punto
di riferimento per il calcolo del timeout e per il confronto con i `last_changed` delle var Frigate.

### Codice COMPLETO dell'automation da aggiungere

```yaml
  # Opzione 3: registra il timestamp esatto dell'inizio dello stato 'triggered'.
  # Questo valore viene usato da binary_sensor.allarme_core_dati_pronti per:
  #   - calcolare i secondi trascorsi dal trigger (per il timeout di fallback)
  #   - confrontare i last_changed di var.allarme_core_last_snapshot_urls e
  #     var.allarme_core_last_clip_urls (devono essere aggiornati DOPO il trigger)
  - alias: "Allarme Core - Registra timestamp inizio triggered"
    id: allarme_core_aut_registra_triggered_started
    mode: single
    trigger:
      - platform: state
        entity_id: input_select.allarme_core_stato
        to: "triggered"
    action:
      - action: input_datetime.set_datetime
        target:
          entity_id: input_datetime.allarme_core_triggered_started
        data:
          datetime: "{{ now().strftime('%Y-%m-%d %H:%M:%S') }}"
```

---

## Intervento 4 — binary_sensor.allarme_core_dati_pronti (Opzione 3)

### File: `allarme-core/allarme_core.yaml`

Il binary_sensor va aggiunto nel blocco `template: - binary_sensor:` gia esistente (dopo riga 322).

### Logica dettagliata del template (spiegazione riga per riga)

Il binary_sensor diventa `on` quando **tutti** i seguenti criteri sono soddisfatti,
**oppure** quando il timeout di fallback e scattato:

**Blocco variabili preliminari:**
- `stato`: stato corrente dell'allarme — deve essere `triggered` per procedere
- `triggered_ts`: timestamp (float UNIX) di quando e iniziato il triggered, letto da `input_datetime.allarme_core_triggered_started`
- `timeout_sec`: valore configurabile da `input_number.allarme_core_notifica_timeout_secondi` (default 45)
- `secondi_da_triggered`: differenza tra `now()` e `triggered_ts` in secondi
- `timeout_scattato`: booleano, `true` se `secondi_da_triggered >= timeout_sec`

**Blocco controllo sensori attivazione:**
- `sensori_raw`: stato grezzo di `var.allarme_core_sensori_attivazione_allarme_var`
- `sensori_ok`: `true` se il valore non e tra `['', 'unknown', 'unavailable', 'none', 'None']`
  Il sensore viene scritto dall'automation Fix-C come stringa CSV degli entity_id; un valore non vuoto
  e sufficiente a considerarlo popolato.

**Blocco controllo Frigate:**
- `frigate_url`: stato di `input_text.allarme_core_url_base_frigate`
- `frigate_configurato`: `true` se `frigate_url` non e tra `['', 'unknown', 'unavailable']`
- Se Frigate NON e configurato → `snapshot_ok = true` e `clip_ok = true` (non si aspetta niente)
- Se Frigate E configurato:
  - `snapshot_ts`: `as_timestamp(states.var.allarme_core_last_snapshot_urls.last_changed)` — timestamp UNIX dell'ultimo aggiornamento della var snapshot. Usa `states.var.*` (accesso oggetto) anziche `states('var.*')` per leggere `last_changed`.
  - `clip_ts`: analogo per `var.allarme_core_last_clip_urls`
  - `snapshot_ok`: `true` se `snapshot_ts` esiste (non none) E `snapshot_ts > triggered_ts`
    (l'aggiornamento e avvenuto DOPO il trigger, non e un vecchio dato residuo)
  - `clip_ok`: `true` se `clip_ts` esiste (non none) E `clip_ts > triggered_ts`

**Stato finale:**
- `dati_completi`: `true` se `sensori_ok AND snapshot_ok AND clip_ok`
- Lo stato del binary_sensor diventa `on` se: `stato == 'triggered' AND (dati_completi OR timeout_scattato)`
- Se `stato != 'triggered'` → rimane `off`

**Attributi esposti:**
- `dati_completi`: bool — tutti i dati sono disponibili (senza timeout)
- `causa_attivazione`: valore grezzo di `var.allarme_core_sensori_attivazione_allarme_var`
- `timeout_scattato`: bool — il timeout di fallback e scattato prima dei dati completi
- `secondi_da_triggered`: float — secondi trascorsi dallo stato triggered

### Codice COMPLETO del binary_sensor

```yaml
      # Opzione 3: binary_sensor di sincronizzazione per la notifica allarme.
      # Diventa 'on' quando tutti i dati necessari sono disponibili (sensori causa
      # allarme + eventuali media Frigate) oppure quando scatta il timeout di fallback.
      # L'automation di notifica si triggera sul fronte off→on di questo sensore.
      - name: "Allarme Core Dati Pronti"
        unique_id: allarme_core_dati_pronti

        state: >
          {% set stato = states('input_select.allarme_core_stato') %}

          {# Se non siamo in triggered, il sensore e sempre off #}
          {% if stato != 'triggered' %}
            false
          {% else %}

            {# --- Calcolo timeout --- #}
            {% set triggered_ts_raw = states('input_datetime.allarme_core_triggered_started') %}
            {% set triggered_ts = as_timestamp(triggered_ts_raw) if triggered_ts_raw not in ['unknown', 'unavailable', ''] else none %}
            {% set timeout_sec = states('input_number.allarme_core_notifica_timeout_secondi') | float(45) %}
            {% set secondi_da_triggered = (now().timestamp() - triggered_ts) if triggered_ts is not none else 9999 %}
            {% set timeout_scattato = secondi_da_triggered >= timeout_sec %}

            {# --- Controllo sensori causa allarme --- #}
            {% set sensori_raw = states('var.allarme_core_sensori_attivazione_allarme_var') %}
            {% set sensori_ok = sensori_raw not in ['', 'unknown', 'unavailable', 'none', 'None'] %}

            {# --- Controllo Frigate --- #}
            {% set frigate_url = states('input_text.allarme_core_url_base_frigate') %}
            {% set frigate_configurato = frigate_url not in ['', 'unknown', 'unavailable'] %}

            {% if not frigate_configurato %}
              {# Frigate non e presente: non si aspettano media #}
              {% set snapshot_ok = true %}
              {% set clip_ok = true %}
            {% else %}
              {# Frigate presente: verificare che le var siano aggiornate DOPO il trigger #}
              {% set snapshot_ts = as_timestamp(states.var.allarme_core_last_snapshot_urls.last_changed) if states.var.allarme_core_last_snapshot_urls is defined else none %}
              {% set clip_ts = as_timestamp(states.var.allarme_core_last_clip_urls.last_changed) if states.var.allarme_core_last_clip_urls is defined else none %}
              {% set snapshot_ok = snapshot_ts is not none and triggered_ts is not none and snapshot_ts > triggered_ts %}
              {% set clip_ok = clip_ts is not none and triggered_ts is not none and clip_ts > triggered_ts %}
            {% endif %}

            {# --- Valutazione finale --- #}
            {% set dati_completi = sensori_ok and snapshot_ok and clip_ok %}
            {{ dati_completi or timeout_scattato }}

          {% endif %}

        attributes:
          # true se tutti i dati sono disponibili senza aver aspettato il timeout
          dati_completi: >
            {% set stato = states('input_select.allarme_core_stato') %}
            {% if stato != 'triggered' %}
              false
            {% else %}
              {% set sensori_raw = states('var.allarme_core_sensori_attivazione_allarme_var') %}
              {% set sensori_ok = sensori_raw not in ['', 'unknown', 'unavailable', 'none', 'None'] %}
              {% set frigate_url = states('input_text.allarme_core_url_base_frigate') %}
              {% set frigate_configurato = frigate_url not in ['', 'unknown', 'unavailable'] %}
              {% if not frigate_configurato %}
                {% set snapshot_ok = true %}
                {% set clip_ok = true %}
              {% else %}
                {% set triggered_ts_raw = states('input_datetime.allarme_core_triggered_started') %}
                {% set triggered_ts = as_timestamp(triggered_ts_raw) if triggered_ts_raw not in ['unknown', 'unavailable', ''] else none %}
                {% set snapshot_ts = as_timestamp(states.var.allarme_core_last_snapshot_urls.last_changed) if states.var.allarme_core_last_snapshot_urls is defined else none %}
                {% set clip_ts = as_timestamp(states.var.allarme_core_last_clip_urls.last_changed) if states.var.allarme_core_last_clip_urls is defined else none %}
                {% set snapshot_ok = snapshot_ts is not none and triggered_ts is not none and snapshot_ts > triggered_ts %}
                {% set clip_ok = clip_ts is not none and triggered_ts is not none and clip_ts > triggered_ts %}
              {% endif %}
              {{ sensori_ok and snapshot_ok and clip_ok }}
            {% endif %}

          # Valore grezzo della var sensori causa allarme (stringa CSV entity_id)
          causa_attivazione: >
            {{ states('var.allarme_core_sensori_attivazione_allarme_var') }}

          # true se il timeout di fallback ha forzato il passaggio a 'on'
          timeout_scattato: >
            {% set stato = states('input_select.allarme_core_stato') %}
            {% if stato != 'triggered' %}
              false
            {% else %}
              {% set triggered_ts_raw = states('input_datetime.allarme_core_triggered_started') %}
              {% set triggered_ts = as_timestamp(triggered_ts_raw) if triggered_ts_raw not in ['unknown', 'unavailable', ''] else none %}
              {% set timeout_sec = states('input_number.allarme_core_notifica_timeout_secondi') | float(45) %}
              {% set secondi = (now().timestamp() - triggered_ts) if triggered_ts is not none else 0 %}
              {{ secondi >= timeout_sec }}
            {% endif %}

          # Secondi trascorsi da quando lo stato e diventato 'triggered'
          secondi_da_triggered: >
            {% set triggered_ts_raw = states('input_datetime.allarme_core_triggered_started') %}
            {% set triggered_ts = as_timestamp(triggered_ts_raw) if triggered_ts_raw not in ['unknown', 'unavailable', ''] else none %}
            {% if triggered_ts is not none %}
              {{ (now().timestamp() - triggered_ts) | round(1) }}
            {% else %}
              0
            {% endif %}
```

---

## Intervento 5 — Automation Notifica su dati_pronti

### File: `allarme-core/allarme_core_automazioni.yaml`

### Cosa cambia

Viene creata una nuova automation che si triggera sul fronte `off` → `on` di
`binary_sensor.allarme_core_dati_pronti`. Questo garantisce che la notifica parta
una sola volta per ogni evento di triggered, solo quando i dati sono pronti
(o al timeout di fallback), mai prima.

Il corpo della notifica e un **skeleton con TODO** perche la logica di invio
(canali, template testo, allegati snapshot/clip) e specifica dell'installazione
e va personalizzata.

### Codice COMPLETO dell'automation (skeleton con TODO)

```yaml
  # Opzione 3: automation di notifica allarme sincronizzata su 'dati_pronti'.
  # Si triggera una sola volta per evento triggered, quando binary_sensor.allarme_core_dati_pronti
  # passa da off a on (dati pronti o timeout scattato).
  # NOTA: usare mode: single per evitare notifiche duplicate sullo stesso evento.
  - alias: "Allarme Core - Notifica Allarme (dati pronti)"
    id: allarme_core_aut_notifica_allarme_dati_pronti
    mode: single
    trigger:
      - platform: state
        entity_id: binary_sensor.allarme_core_dati_pronti
        from: "off"
        to: "on"
    # Verifica di sicurezza: l'allarme deve essere ancora triggered al momento
    # dell'esecuzione (potrebbe essere stato disarmato nel frattempo)
    condition:
      - condition: state
        entity_id: input_select.allarme_core_stato
        state: "triggered"
    variables:
      # Lettura attributi del binary_sensor per costruire il corpo notifica
      causa_attivazione: "{{ state_attr('binary_sensor.allarme_core_dati_pronti', 'causa_attivazione') }}"
      dati_completi: "{{ state_attr('binary_sensor.allarme_core_dati_pronti', 'dati_completi') }}"
      timeout_scattato: "{{ state_attr('binary_sensor.allarme_core_dati_pronti', 'timeout_scattato') }}"
      secondi_da_triggered: "{{ state_attr('binary_sensor.allarme_core_dati_pronti', 'secondi_da_triggered') }}"
      # Nomi leggibili dalla var (attributo 'nomi' scritto dall'automation Fix-C)
      causa_nomi: "{{ state_attr('var.allarme_core_sensori_attivazione_allarme_var', 'nomi') | default([]) }}"
      # URL snapshot e clip (se Frigate configurato)
      snapshot_urls: "{{ states('var.allarme_core_last_snapshot_urls') }}"
      clip_urls: "{{ states('var.allarme_core_last_clip_urls') }}"
      profilo: "{{ states('input_select.allarme_core_profilo') }}"
      # URL base HA per link nella notifica
      url_ha: "{{ states('input_text.allarme_core_url_base_ha') }}"
    action:
      # TODO: sostituire con il servizio di notifica appropriato per l'installazione
      # Esempi: notify.mobile_app_*, notify.telegram_*, notify.persistent_notification
      - action: notify.persistent_notification
        data:
          title: "ALLARME ATTIVATO"
          # TODO: personalizzare il messaggio con i dati disponibili nelle variabili
          message: >
            Allarme attivato alle {{ now().strftime('%H:%M:%S') }}.
            Profilo: {{ profilo }}.
            Sensori: {{ causa_nomi | join(', ') if causa_nomi else causa_attivazione }}.
            {% if timeout_scattato and not dati_completi %}
            ATTENZIONE: notifica inviata per timeout ({{ secondi_da_triggered }}s) — dati media Frigate non disponibili.
            {% endif %}

      # TODO: aggiungere azione per inviare snapshot se Frigate e configurato
      # Esempio condizionale:
      # - condition: template
      #   value_template: "{{ states('input_text.allarme_core_url_base_frigate') not in ['', 'unknown', 'unavailable'] }}"
      # - action: notify.mobile_app_*
      #   data:
      #     message: "..."
      #     data:
      #       image: "{{ snapshot_urls }}"

      # TODO: aggiungere log su var timeline se presente nel sistema
      # - action: var.set
      #   data:
      #     entity_id: var.allarme_core_timeline_critici_md
      #     value: "..."
```

---

## Checklist Pre-Implementazione

Prima di procedere con l'implementazione, verificare le seguenti condizioni:

### Verifica entity esistenti
- [ ] Confermare che `var.allarme_core_sensori_esclusi_var` esiste e ha stato leggibile (non solo `unavailable`) dopo un arming
- [ ] Confermare che `var.allarme_core_last_snapshot_urls` e `var.allarme_core_last_clip_urls` vengono scritte da un'automation o flow Node-RED esistente quando Frigate produce media — verificare quale automation/script le aggiorna e con quale formato
- [ ] Verificare che `sensor.allarme_core_zone_attive` abbia sempre l'attributo `zones` popolato (non `none`) per tutti i profili incluso `nessuno`
- [ ] Verificare l'esistenza di `input_boolean.allarme_core_modalita_test` referenziato nell'automation test (gia presente nei file ma non visibile in `allarme_core.yaml` — potrebbe essere in `allarme_core_supporto.yaml`)

### Verifica formato dati Frigate
- [ ] Stabilire il formato con cui `var.allarme_core_last_snapshot_urls` viene aggiornata (stringa URL singola, JSON lista, attributo `long_state`?) — il binary_sensor `dati_pronti` usa solo `last_changed` quindi il formato del valore non influenza la logica, ma e importante per la notifica
- [ ] Stesso per `var.allarme_core_last_clip_urls`

### Verifica integrazione var HACS
- [ ] Confermare che `var.set` accetta il parametro `attributes` con chiavi arbitrarie come usato nel Fix-C (nomi, ids, timestamp) — alcune versioni di var HACS richiedono la pre-dichiarazione degli attributi in `var/allarme_core.yaml`
- [ ] Aggiungere a `var/allarme_core.yaml` nella definizione di `allarme_core_sensori_attivazione_allarme_var` gli attributi: `profilo`, `ids`, `nomi`, `timestamp` se non gia presenti

### Verifica syntax HA versione
- [ ] Confermare che la versione di HA installata supporta `action:` in automations (introdotto in HA 2024.x in sostituzione di `service:`) — se la versione e precedente usare `service:` per tutti gli step
- [ ] Confermare che `states.var.*` sia accessibile nei template (dipende dalla configurazione di var HACS)

### Ordine di implementazione consigliato
1. Intervento 3 prima (aggiungere le entity helper) — senza di esse il binary_sensor di Intervento 4 risultera unavailable
2. Intervento 1 (Fix B) — semplice e a basso rischio
3. Intervento 2 (Fix C) — richiede riavvio HA per ricaricare le automazioni
4. Intervento 4 (binary_sensor dati_pronti) — dopo che le entity helper esistono
5. Intervento 5 (automation notifica) — ultima, dopo test del binary_sensor

---

## Note e Rischi

### Edge case: triggered durante arming
Se lo stato salta da `arming` a `triggered` direttamente (senza passare per `armed`),
la var `allarme_core_sensori_esclusi_var` potrebbe non essere ancora aggiornata.
In questo caso il Fix-C non applica esclusioni e considera tutti i sensori aperti come causa
dell'allarme — comportamento probabilmente corretto (se aperto durante arming e causa).

### Edge case: triggered con profilo 'nessuno'
Se `input_select.allarme_core_profilo` e `nessuno` al momento del triggered,
il Fix-C produrra una lista `aperti_ids` vuota. La var `sensori_attivazione_allarme_var`
verra scritta con stringa vuota e `sensori_ok` nel binary_sensor sara `false`.
Il sistema aspettera il timeout prima di notificare. Valutare se aggiungere una condizione
che salti il filtro profilo in caso di triggered (allarme scattato da script diretto?).

### Edge case: riavvio HA durante triggered
`input_datetime.allarme_core_triggered_started` sopravvive al riavvio HA perche e un
entity helper persistente. Tuttavia se HA si riavvia mentre lo stato e `triggered`,
il binary_sensor `dati_pronti` potrebbe trovarsi in uno stato inconsistente:
`triggered_ts` e nel passato, `secondi_da_triggered` potrebbe gia superare il timeout,
e il sensore diventerebbe subito `on`. Questo e il comportamento desiderato (notifica di
sicurezza), ma occorre assicurarsi che anche l'automation di notifica (Intervento 5)
sia in `mode: single` per non sparare notifiche duplicate al boot.

### Edge case: Frigate lento o irraggiungibile
Se Frigate e configurato ma non risponde, le var `last_snapshot_urls` e `last_clip_urls`
non vengono mai aggiornate dopo il trigger. Il binary_sensor aspettera esattamente
`allarme_core_notifica_timeout_secondi` secondi, poi notifichera con `timeout_scattato: true`.
Il messaggio di notifica nel TODO deve riportare chiaramente questo stato.

### Edge case: var HACS con last_changed non aggiornato
`as_timestamp(states.var.allarme_core_last_snapshot_urls.last_changed)` restituisce il
timestamp dell'ultimo cambio di stato OPPURE di attributo della var. Se il sistema scrive
lo stesso valore senza cambiare nulla, `last_changed` non si aggiorna. Verificare che
l'automation che aggiorna le var Frigate cambi sempre il valore (es. aggiungendo timestamp
nel contenuto) per garantire che `last_changed` venga aggiornato.

### Performance template binary_sensor dati_pronti
Il binary_sensor rivaluta il template ad ogni cambio di stato delle entity dipendenti.
Con Frigate attivo, `var.allarme_core_last_snapshot_urls` e `var.allarme_core_last_clip_urls`
potrebbero aggiornarsi frequentemente anche fuori dal contesto triggered. Il check iniziale
`stato != 'triggered' -> false` e computazionalmente leggero e cortocircuita subito
la valutazione — impatto prestazioni trascurabile.

### Deprecazione `service:` vs `action:`
I file attuali usano ancora `service:` (sintassi HA <2024.8). Il piano usa `action:`
per i nuovi blocchi. Per coerenza con il codebase esistente, valutare se uniformare
tutti i nuovi blocchi a `service:` oppure aggiornare anche i vecchi durante la stessa PR.
