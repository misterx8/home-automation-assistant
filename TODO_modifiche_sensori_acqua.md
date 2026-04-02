# Piano modifiche — Sensori Vibrazione (allarme-core)

Generato il: 2026-04-02  
Basato su: `TODO_sensori.md`

---

## Sensori fisici da integrare

### 🔹 Sensori vibrazione (`binary_sensor.*`)
| Entità fisica | Stanza | Tipo modifica |
|---|---|---|
| `binary_sensor.vibrazione_cucina_sottoscala` | cucina | **nuovo wrapper multi-sensore** |
| `binary_sensor.vibrazione_cucina_lavello` | cucina | **nuovo wrapper multi-sensore** |
| `binary_sensor.vibrazione_taverna_divano` | taverna | **aggiunto al wrapper esistente** `allarme_core_vibrazione_ingresso_taverna` (→ diventa 3 sensori) |
| `binary_sensor.vibrazione_bagno` | bagno | **nuovo wrapper singolo** |
| `binary_sensor.vibrazione_lavanderia` | lavanderia | **nuovo wrapper singolo** |

### 🔹 Disponibilità e batteria
| Entità | Usata in |
|---|---|
| `binary_sensor.disponibilita_vibrazione_cucina_sottoscala` | cucina `sensori_config[0]` |
| `binary_sensor.disponibilita_vibrazione_cucina_lavello` | cucina `sensori_config[1]` |
| `binary_sensor.disponibilita_vibrazione_taverna_divano` | taverna `sensori_config[2]` (entry aggiunta) |
| `binary_sensor.disponibilita_vibrazione_bagno` | bagno attr `disponibilita_origine` |
| `binary_sensor.disponibilita_vibrazione_lavanderia` | lavanderia attr `disponibilita_origine` |
| `sensor.batteria_vibrazione_cucina_sottoscala` | cucina `sensori_config[0]` |
| `sensor.batteria_vibrazione_cucina_lavello` | cucina `sensori_config[1]` |
| `sensor.batteria_vibrazione_taverna_divano` | taverna `sensori_config[2]` (entry aggiunta) |
| `sensor.batteria_vibrazione_bagno` | bagno attr `batteria_origine` |
| `sensor.batteria_vibrazione_lavanderia` | lavanderia attr `batteria_origine` |

---

## Wrapper risultanti

| Wrapper | Sensori fisici | Operazione |
|---|---|---|
| `allarme_core_vibrazione_cucina` | sottoscala + lavello | **NUOVO** (multi) |
| `allarme_core_vibrazione_ingresso_taverna` | entrata + tapparella + **divano** | **MODIFICA** wrapper esistente |
| `allarme_core_vibrazione_bagno` | bagno | **NUOVO** (singolo) |
| `allarme_core_vibrazione_lavanderia` | lavanderia | **NUOVO** (singolo) |

---

## FILE 1: `allarme-core/allarme_core_sensori.yaml`

### Modifica 1.1 — `input_boolean` (3 nuovi, nessuno per taverna)

**In `#region cucina`** (dopo `allarme_core_finestra_cucina_abilitato`, ~riga 30):
```yaml
  allarme_core_vibrazione_cucina_abilitato:
    name: "Allarme Vibrazione Cucina Abilitato"
    icon: mdi:shield-check
```

**In `#region bagno`** (dopo `allarme_core_finestra_bagno_abilitato`, ~riga 39):
```yaml
  allarme_core_vibrazione_bagno_abilitato:
    name: "Allarme Vibrazione Bagno Abilitato"
    icon: mdi:shield-check
```

**In `#region lavanderia`** (dopo `allarme_core_finestra_lavanderia_abilitato`, ~riga 48):
```yaml
  allarme_core_vibrazione_lavanderia_abilitato:
    name: "Allarme Vibrazione Lavanderia Abilitato"
    icon: mdi:shield-check
```

---

### Modifica 1.2 — `input_text` zone (3 nuovi, nessuno per taverna)

**In `#region cucina`** (dopo `allarme_core_finestra_cucina_zone`):
```yaml
  allarme_core_vibrazione_cucina_zone:
    name: "Zone Vibrazione Cucina"
    pattern: "^[^,]+(,[^,]+)*$"
```

**In `#region bagno`** (dopo `allarme_core_finestra_bagno_zone`):
```yaml
  allarme_core_vibrazione_bagno_zone:
    name: "Zone Vibrazione Bagno"
    pattern: "^[^,]+(,[^,]+)*$"
```

**In `#region lavanderia`** (dopo `allarme_core_finestra_lavanderia_zone`):
```yaml
  allarme_core_vibrazione_lavanderia_zone:
    name: "Zone Vibrazione Lavanderia"
    pattern: "^[^,]+(,[^,]+)*$"
```

---

### Modifica 1.3 — `input_text` camera (3 nuovi, nessuno per taverna)

**In `#region cucina`** (dopo `allarme_core_finestra_cucina_camera`):
```yaml
  allarme_core_vibrazione_cucina_camera:
    name: "Camera Vibrazione Cucina Allarme"
    pattern: "^[^,]+(,[^,]+)*$"
```

**In `#region bagno`** (dopo `allarme_core_finestra_bagno_camera`):
```yaml
  allarme_core_vibrazione_bagno_camera:
    name: "Camera Vibrazione Bagno Allarme"
    pattern: "^[^,]+(,[^,]+)*$"
```

**In `#region lavanderia`** (dopo `allarme_core_finestra_lavanderia_camera`):
```yaml
  allarme_core_vibrazione_lavanderia_camera:
    name: "Camera Vibrazione Lavanderia Allarme"
    pattern: "^[^,]+(,[^,]+)*$"
```

---

### Modifica 1.4 — `template: binary_sensor` — NUOVO wrapper CUCINA (multi)

Aggiungere in `#region cucina` dopo il wrapper `Allarme Core Finestre Cucina` (~riga 854):

```yaml
    - name: "Allarme Core Vibrazione Cucina"
      unique_id: allarme_core_vibrazione_cucina
      device_class: vibration
      state: >
        {% set sorgenti = [
          'binary_sensor.vibrazione_cucina_sottoscala',
          'binary_sensor.vibrazione_cucina_lavello'
        ] %}
        {% set abilitato = is_state('input_boolean.allarme_core_vibrazione_cucina_abilitato','on') %}
        {% if not abilitato %}
          off
        {% else %}
          {{ 'on' if sorgenti | select('is_state','on') | list | count > 0 else 'off' }}
        {% endif %}
      attributes:
        sensori_origine: >
          {{ [
            'binary_sensor.vibrazione_cucina_sottoscala',
            'binary_sensor.vibrazione_cucina_lavello'
          ] | to_json }}
        sensori_config: >
          {{ [
            {'entity_id': 'binary_sensor.vibrazione_cucina_sottoscala', 'tipo_connessione': 'wireless', 'batteria_origine': 'sensor.batteria_vibrazione_cucina_sottoscala', 'disponibilita_origine': 'binary_sensor.disponibilita_vibrazione_cucina_sottoscala'},
            {'entity_id': 'binary_sensor.vibrazione_cucina_lavello', 'tipo_connessione': 'wireless', 'batteria_origine': 'sensor.batteria_vibrazione_cucina_lavello', 'disponibilita_origine': 'binary_sensor.disponibilita_vibrazione_cucina_lavello'}
          ] | to_json }}
        sensori_aperti: >
          {% set sorgenti = [
            'binary_sensor.vibrazione_cucina_sottoscala',
            'binary_sensor.vibrazione_cucina_lavello'
          ] %}
          {{ sorgenti | select('is_state','on') | list }}
        nomi_aperti: >
          {% set sorgenti = [
            'binary_sensor.vibrazione_cucina_sottoscala',
            'binary_sensor.vibrazione_cucina_lavello'
          ] %}
          {{ sorgenti | select('is_state','on') | map('state_attr','friendly_name') | list }}
        tipo_connessione: wireless
        tipo: vibrazione
        priorita: media
        ritardo: 0
        zone: "{{ states('input_text.allarme_core_vibrazione_cucina_zone').split(',') }}"
        camera: "{{ states('input_text.allarme_core_vibrazione_cucina_camera').split(',') }}"
        abilitato: "{{ is_state('input_boolean.allarme_core_vibrazione_cucina_abilitato','on') }}"
```

---

### Modifica 1.5 — `template: binary_sensor` — MODIFICA wrapper TAVERNA esistente

Nel wrapper `allarme_core_vibrazione_ingresso_taverna` (~riga 708–757) aggiungere
`binary_sensor.vibrazione_taverna_divano` come **3° elemento** in tutti e 4 i punti:

**`state` — nella lista `sorgenti`:**
```yaml
        # aggiungere dopo 'binary_sensor.vibrazione_tapparella_taverna':
          'binary_sensor.vibrazione_taverna_divano'
```

**attributo `sensori_origine` — nella lista JSON:**
```yaml
        # aggiungere come 3° elemento:
            'binary_sensor.vibrazione_taverna_divano'
```

**attributo `sensori_config` — nuova entry JSON:**
```yaml
        # aggiungere come 3° elemento del array:
            {'entity_id': 'binary_sensor.vibrazione_taverna_divano', 'tipo_connessione': 'wireless', 'batteria_origine': 'sensor.batteria_vibrazione_taverna_divano', 'disponibilita_origine': 'binary_sensor.disponibilita_vibrazione_taverna_divano'}
```

**attributi `sensori_aperti` e `nomi_aperti` — nella lista `sorgenti`:**
```yaml
        # aggiungere dopo 'binary_sensor.vibrazione_tapparella_taverna':
          'binary_sensor.vibrazione_taverna_divano'
```

---

### Modifica 1.6 — `template: binary_sensor` — NUOVO wrapper BAGNO (singolo)

Aggiungere in `#region bagno` dopo il wrapper `Allarme Core Finestra bagno`:

```yaml
    - name: "Allarme Core Vibrazione Bagno"
      unique_id: allarme_core_vibrazione_bagno
      device_class: vibration
      state: >
        {% set porta = states('binary_sensor.vibrazione_bagno') %}
        {% set abilitato = is_state('input_boolean.allarme_core_vibrazione_bagno_abilitato','on') %}
        {% if porta in ['on','off'] %}
          {% if abilitato %}
            {{ porta }}
          {% else %}
            off
          {% endif %}
        {% else %}
          off
        {% endif %}
      attributes:
        sensore_origine: binary_sensor.vibrazione_bagno
        tipo_connessione: wireless
        batteria_origine: sensor.batteria_vibrazione_bagno
        disponibilita_origine: binary_sensor.disponibilita_vibrazione_bagno
        tipo: vibrazione
        priorita: media
        ritardo: 0
        zone: "{{ states('input_text.allarme_core_vibrazione_bagno_zone').split(',') }}"
        camera: "{{ states('input_text.allarme_core_vibrazione_bagno_camera').split(',') }}"
        abilitato: "{{ is_state('input_boolean.allarme_core_vibrazione_bagno_abilitato','on') }}"
```

---

### Modifica 1.7 — `template: binary_sensor` — NUOVO wrapper LAVANDERIA (singolo)

Aggiungere in `#region lavanderia` dopo il wrapper `Allarme Core Finestra lavanderia`:

```yaml
    - name: "Allarme Core Vibrazione Lavanderia"
      unique_id: allarme_core_vibrazione_lavanderia
      device_class: vibration
      state: >
        {% set porta = states('binary_sensor.vibrazione_lavanderia') %}
        {% set abilitato = is_state('input_boolean.allarme_core_vibrazione_lavanderia_abilitato','on') %}
        {% if porta in ['on','off'] %}
          {% if abilitato %}
            {{ porta }}
          {% else %}
            off
          {% endif %}
        {% else %}
          off
        {% endif %}
      attributes:
        sensore_origine: binary_sensor.vibrazione_lavanderia
        tipo_connessione: wireless
        batteria_origine: sensor.batteria_vibrazione_lavanderia
        disponibilita_origine: binary_sensor.disponibilita_vibrazione_lavanderia
        tipo: vibrazione
        priorita: media
        ritardo: 0
        zone: "{{ states('input_text.allarme_core_vibrazione_lavanderia_zone').split(',') }}"
        camera: "{{ states('input_text.allarme_core_vibrazione_lavanderia_camera').split(',') }}"
        abilitato: "{{ is_state('input_boolean.allarme_core_vibrazione_lavanderia_abilitato','on') }}"
```

---

## FILE 2: `allarme-core/allarme_core_supporto.yaml`

### Modifica 2.1 — 3 nuovi `*_zone_valida` (alla fine del file, nessuno per taverna)

```yaml
    - name: "allarme_core_vibrazione_cucina_zone_valida"
      state: >
          {% set zone_list = states('input_text.allarme_core_vibrazione_cucina_zone').split(',') | map('trim') | list %}
          {% set available_zones = state_attr('input_select.allarme_core_zone_disponibili','options') %}
          {% if zone_list == [''] %}
            false
          {% else %}
            {{ zone_list | select('in', available_zones) | list | length == zone_list | length }}
          {% endif %}

    - name: "allarme_core_vibrazione_bagno_zone_valida"
      state: >
          {% set zone_list = states('input_text.allarme_core_vibrazione_bagno_zone').split(',') | map('trim') | list %}
          {% set available_zones = state_attr('input_select.allarme_core_zone_disponibili','options') %}
          {% if zone_list == [''] %}
            false
          {% else %}
            {{ zone_list | select('in', available_zones) | list | length == zone_list | length }}
          {% endif %}

    - name: "allarme_core_vibrazione_lavanderia_zone_valida"
      state: >
          {% set zone_list = states('input_text.allarme_core_vibrazione_lavanderia_zone').split(',') | map('trim') | list %}
          {% set available_zones = state_attr('input_select.allarme_core_zone_disponibili','options') %}
          {% if zone_list == [''] %}
            false
          {% else %}
            {{ zone_list | select('in', available_zones) | list | length == zone_list | length }}
          {% endif %}
```

---

## FILE 3: `allarme-core/plancia_controllo.yaml`

### Modifica 3.1 — 3 nuove tile card (nessuna per taverna — il wrapper esistente rimane)

Il pattern è identico per tutti i sensori esistenti: `tile` con `card_mod style` che legge
`sensore_origine` / `sensori_origine` dal wrapper per rilevare anomalie fisiche automaticamente.

---

#### Card CUCINA — inserire prima di `- type: heading` Bagno (~riga 2335)

```yaml
      - type: tile
        grid_options:
          columns: 6
          rows: 1
        entity: binary_sensor.allarme_core_vibrazione_cucina
        name: " "
        vertical: false
        tap_action:
          action: call-service
          service: browser_mod.popup
          service_data:
            title: Dettagli avanzati
            content:
              type: entities
              entities:
                - entity: input_boolean.allarme_core_vibrazione_cucina_abilitato
                - entity: input_text.allarme_core_vibrazione_cucina_zone
                  secondary_info: last-updated
                - entity: input_text.allarme_core_vibrazione_cucina_camera
                  secondary_info: last-updated
        features_position: bottom
        card_mod:
          style: >
            {% set enabled =
            states('input_boolean.allarme_core_vibrazione_cucina_abilitato')
            %} {% set zone =
            states('input_text.allarme_core_vibrazione_cucina_zone')
            %} {% set camera =
            states('input_text.allarme_core_vibrazione_cucina_camera')
            %} {% set errore =
            states('binary_sensor.allarme_core_vibrazione_cucina_zone_valida')
            %}

            {% set invalid_zone = zone in ['unknown','unavailable',''] %} {% set
            invalid_camera = camera in ['unknown','unavailable',''] %}

            {% set fisico_s = state_attr(config.entity, 'sensore_origine') %}
            {% set fisici_json = state_attr(config.entity, 'sensori_origine') %}
            {% set fisici_l = fisici_json | from_json if fisici_json is string else [] %}
            {% set ns_fa = namespace(v=false) %}
            {% if fisico_s is not none and states(fisico_s) not in ['on','off'] %}
              {% set ns_fa.v = true %}
            {% endif %}
            {% for f in fisici_l %}
              {% if states(f) not in ['on','off'] %}{% set ns_fa.v = true %}{% endif %}
            {% endfor %}
            {% set fisico_anomalo = enabled == 'on' and ns_fa.v %}

            {% set border_color = 'red' if enabled == 'off' else ('orange' if fisico_anomalo else 'green') %}

            ha-card {
              border-radius: 30px;
              box-shadow: 0 0 0 2px {{ border_color }};
              padding-left: 12px;
              position: relative;
            }

            /* badge icona warning / errore */ {% if errore == 'off' %}
            ha-card::after {
              content: "❌";
              position: absolute;
              top: 12px;
              right: 12px;
              font-size: 1.2em;
            } {% elif fisico_anomalo %} ha-card::after {
              content: "🔌";
              position: absolute;
              top: 12px;
              right: 12px;
              font-size: 1.2em;
            } {% elif invalid_zone or invalid_camera %} ha-card::after {
              content: "⚠️";
              position: absolute;
              top: 12px;
              right: 12px;
              font-size: 1.2em;
            } {% endif %}
```

---

#### Card BAGNO — inserire prima di `- type: heading` Lavanderia (~riga 2497)

```yaml
      - type: tile
        grid_options:
          columns: 6
          rows: 1
        entity: binary_sensor.allarme_core_vibrazione_bagno
        name: " "
        vertical: false
        tap_action:
          action: call-service
          service: browser_mod.popup
          service_data:
            title: Dettagli avanzati
            content:
              type: entities
              entities:
                - entity: input_boolean.allarme_core_vibrazione_bagno_abilitato
                - entity: input_text.allarme_core_vibrazione_bagno_zone
                  secondary_info: last-updated
                - entity: input_text.allarme_core_vibrazione_bagno_camera
                  secondary_info: last-updated
        features_position: bottom
        card_mod:
          style: >
            {% set enabled =
            states('input_boolean.allarme_core_vibrazione_bagno_abilitato')
            %} {% set zone =
            states('input_text.allarme_core_vibrazione_bagno_zone')
            %} {% set camera =
            states('input_text.allarme_core_vibrazione_bagno_camera')
            %} {% set errore =
            states('binary_sensor.allarme_core_vibrazione_bagno_zone_valida')
            %}

            {% set invalid_zone = zone in ['unknown','unavailable',''] %} {% set
            invalid_camera = camera in ['unknown','unavailable',''] %}

            {% set fisico_s = state_attr(config.entity, 'sensore_origine') %}
            {% set fisici_json = state_attr(config.entity, 'sensori_origine') %}
            {% set fisici_l = fisici_json | from_json if fisici_json is string else [] %}
            {% set ns_fa = namespace(v=false) %}
            {% if fisico_s is not none and states(fisico_s) not in ['on','off'] %}
              {% set ns_fa.v = true %}
            {% endif %}
            {% for f in fisici_l %}
              {% if states(f) not in ['on','off'] %}{% set ns_fa.v = true %}{% endif %}
            {% endfor %}
            {% set fisico_anomalo = enabled == 'on' and ns_fa.v %}

            {% set border_color = 'red' if enabled == 'off' else ('orange' if fisico_anomalo else 'green') %}

            ha-card {
              border-radius: 30px;
              box-shadow: 0 0 0 2px {{ border_color }};
              padding-left: 12px;
              position: relative;
            }

            /* badge icona warning / errore */ {% if errore == 'off' %}
            ha-card::after {
              content: "❌";
              position: absolute;
              top: 12px;
              right: 12px;
              font-size: 1.2em;
            } {% elif fisico_anomalo %} ha-card::after {
              content: "🔌";
              position: absolute;
              top: 12px;
              right: 12px;
              font-size: 1.2em;
            } {% elif invalid_zone or invalid_camera %} ha-card::after {
              content: "⚠️";
              position: absolute;
              top: 12px;
              right: 12px;
              font-size: 1.2em;
            } {% endif %}
```

---

#### Card LAVANDERIA — inserire prima di `- type: heading` Tecnologico (~riga 2663)

```yaml
      - type: tile
        grid_options:
          columns: 6
          rows: 1
        entity: binary_sensor.allarme_core_vibrazione_lavanderia
        name: " "
        vertical: false
        tap_action:
          action: call-service
          service: browser_mod.popup
          service_data:
            title: Dettagli avanzati
            content:
              type: entities
              entities:
                - entity: input_boolean.allarme_core_vibrazione_lavanderia_abilitato
                - entity: input_text.allarme_core_vibrazione_lavanderia_zone
                  secondary_info: last-updated
                - entity: input_text.allarme_core_vibrazione_lavanderia_camera
                  secondary_info: last-updated
        features_position: bottom
        card_mod:
          style: >
            {% set enabled =
            states('input_boolean.allarme_core_vibrazione_lavanderia_abilitato')
            %} {% set zone =
            states('input_text.allarme_core_vibrazione_lavanderia_zone')
            %} {% set camera =
            states('input_text.allarme_core_vibrazione_lavanderia_camera')
            %} {% set errore =
            states('binary_sensor.allarme_core_vibrazione_lavanderia_zone_valida')
            %}

            {% set invalid_zone = zone in ['unknown','unavailable',''] %} {% set
            invalid_camera = camera in ['unknown','unavailable',''] %}

            {% set fisico_s = state_attr(config.entity, 'sensore_origine') %}
            {% set fisici_json = state_attr(config.entity, 'sensori_origine') %}
            {% set fisici_l = fisici_json | from_json if fisici_json is string else [] %}
            {% set ns_fa = namespace(v=false) %}
            {% if fisico_s is not none and states(fisico_s) not in ['on','off'] %}
              {% set ns_fa.v = true %}
            {% endif %}
            {% for f in fisici_l %}
              {% if states(f) not in ['on','off'] %}{% set ns_fa.v = true %}{% endif %}
            {% endfor %}
            {% set fisico_anomalo = enabled == 'on' and ns_fa.v %}

            {% set border_color = 'red' if enabled == 'off' else ('orange' if fisico_anomalo else 'green') %}

            ha-card {
              border-radius: 30px;
              box-shadow: 0 0 0 2px {{ border_color }};
              padding-left: 12px;
              position: relative;
            }

            /* badge icona warning / errore */ {% if errore == 'off' %}
            ha-card::after {
              content: "❌";
              position: absolute;
              top: 12px;
              right: 12px;
              font-size: 1.2em;
            } {% elif fisico_anomalo %} ha-card::after {
              content: "🔌";
              position: absolute;
              top: 12px;
              right: 12px;
              font-size: 1.2em;
            } {% elif invalid_zone or invalid_camera %} ha-card::after {
              content: "⚠️";
              position: absolute;
              top: 12px;
              right: 12px;
              font-size: 1.2em;
            } {% endif %}
```

---

## FILE 4 & 5 — Nessuna modifica

- `allarme_core_anomalie_sensori.yaml`: logica dinamica, include automaticamente i nuovi wrapper
- `allarme_core_batterie.yaml`: logica dinamica, include automaticamente i nuovi wrapper wireless

---

## Riepilogo finale

| # | File | Tipo | Elementi |
|---|---|---|---|
| 1.1 | `allarme_core_sensori.yaml` | `input_boolean` | **+3** (cucina, bagno, lavanderia) |
| 1.2 | `allarme_core_sensori.yaml` | `input_text` zone | **+3** |
| 1.3 | `allarme_core_sensori.yaml` | `input_text` camera | **+3** |
| 1.4 | `allarme_core_sensori.yaml` | nuovo `binary_sensor` cucina (multi) | **+1** |
| 1.5 | `allarme_core_sensori.yaml` | modifica `binary_sensor` taverna (+3° sensore) | **modifica** |
| 1.6 | `allarme_core_sensori.yaml` | nuovo `binary_sensor` bagno (singolo) | **+1** |
| 1.7 | `allarme_core_sensori.yaml` | nuovo `binary_sensor` lavanderia (singolo) | **+1** |
| 2.1 | `allarme_core_supporto.yaml` | `zone_valida` | **+3** |
| 3.1 | `allarme_core/plancia_controllo.yaml` | tile card cucina | **+1** (prima heading Bagno) |
| 3.2 | `allarme_core/plancia_controllo.yaml` | tile card bagno | **+1** (prima heading Lavanderia) |
| 3.3 | `allarme_core/plancia_controllo.yaml` | tile card lavanderia | **+1** (prima heading Tecnologico) |
| — | `allarme_core_anomalie_sensori.yaml` | — | **nessuna modifica** |
| — | `allarme_core_batterie.yaml` | — | **nessuna modifica** |

**Totale elementi nuovi: 18 + 1 modifica esistente**  
**File da toccare: 3**
