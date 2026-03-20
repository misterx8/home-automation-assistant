template:
  - binary_sensor:

      - name: "Disponibilita Contatto Porta Gas Fuori"
        unique_id: disponibilita_contatto_porta_gas_fuori
        device_class: connectivity
        state: >
          {{ states('binary_sensor.contatto_porta_gas_tempo_scaduto') == 'off' }}

      - name: "Disponibilita Contatto Porta Atrezzi"
        unique_id: disponibilita_contatto_porta_atrezzi
        device_class: connectivity
        state: >
          {{ states('binary_sensor.contatto_porta_atrezzi_tempo_scaduto') == 'off' }}

      # --- Acqua Vasca ---
      - name: "Disponibilita Acqua Vasca"
        unique_id: disponibilita_acqua_vasca
        device_class: connectivity
        icon: >
          {% if this.state == 'on' %}
            mdi:bluetooth-connect
          {% else %}
            mdi:bluetooth-off
          {% endif %}
        state: >
          {% set entita = 'sensor.acqua_vasca_segnale_bluetooth' %}
          {% set timeout_secondi = 900 %}
          {% set s = states(entita) %}
          {% if s in ['unavailable', 'unknown', 'none', ''] %}
            {{ false }}
          {% elif (as_timestamp(now()) - as_timestamp(states[entita].last_updated)) > timeout_secondi %}
            {{ false }}
          {% else %}
            {{ true }}
          {% endif %}

      # --- Acqua Bagno ---
      - name: "Disponibilita Acqua Bagno"
        unique_id: disponibilita_acqua_bagno
        device_class: connectivity
        icon: >
          {% if this.state == 'on' %}
            mdi:bluetooth-connect
          {% else %}
            mdi:bluetooth-off
          {% endif %}
        state: >
          {% set entita = 'sensor.acqua_bagno_segnale_bluetooth' %}
          {% set timeout_secondi = 900 %}
          {% set s = states(entita) %}
          {% if s in ['unavailable', 'unknown', 'none', ''] %}
            {{ false }}
          {% elif (as_timestamp(now()) - as_timestamp(states[entita].last_updated)) > timeout_secondi %}
            {{ false }}
          {% else %}
            {{ true }}
          {% endif %}

      # --- Acqua Cucina Principale ---
      - name: "Disponibilita Acqua Cucina Principale"
        unique_id: disponibilita_acqua_cucina_principale
        device_class: connectivity
        icon: >
          {% if this.state == 'on' %}
            mdi:bluetooth-connect
          {% else %}
            mdi:bluetooth-off
          {% endif %}
        state: >
          {% set entita = 'sensor.acqua_cucina_principale_segnale_bluetooth' %}
          {% set timeout_secondi = 900 %}
          {% set s = states(entita) %}
          {% if s in ['unavailable', 'unknown', 'none', ''] %}
            {{ false }}
          {% elif (as_timestamp(now()) - as_timestamp(states[entita].last_updated)) > timeout_secondi %}
            {{ false }}
          {% else %}
            {{ true }}
          {% endif %}

      # --- Acqua Doccia ---
      - name: "Disponibilita Acqua Doccia"
        unique_id: disponibilita_acqua_doccia
        device_class: connectivity
        icon: >
          {% if this.state == 'on' %}
            mdi:bluetooth-connect
          {% else %}
            mdi:bluetooth-off
          {% endif %}
        state: >
          {% set entita = 'sensor.acqua_doccia_segnale_bluetooth' %}
          {% set timeout_secondi = 900 %}
          {% set s = states(entita) %}
          {% if s in ['unavailable', 'unknown', 'none', ''] %}
            {{ false }}
          {% elif (as_timestamp(now()) - as_timestamp(states[entita].last_updated)) > timeout_secondi %}
            {{ false }}
          {% else %}
            {{ true }}
          {% endif %}

      # --- Acqua Lavanderia ---
      - name: "Disponibilita Acqua Lavanderia"
        unique_id: disponibilita_acqua_lavanderia
        device_class: connectivity
        icon: >
          {% if this.state == 'on' %}
            mdi:bluetooth-connect
          {% else %}
            mdi:bluetooth-off
          {% endif %}
        state: >
          {% set entita = 'sensor.acqua_lavanderia_segnale_bluetooth' %}
          {% set timeout_secondi = 900 %}
          {% set s = states(entita) %}
          {% if s in ['unavailable', 'unknown', 'none', ''] %}
            {{ false }}
          {% elif (as_timestamp(now()) - as_timestamp(states[entita].last_updated)) > timeout_secondi %}
            {{ false }}
          {% else %}
            {{ true }}
          {% endif %}

      # --- Acqua Lavello Cucina ---
      - name: "Disponibilita Acqua Lavello Cucina"
        unique_id: disponibilita_acqua_lavello_cucina
        device_class: connectivity
        icon: >
          {% if this.state == 'on' %}
            mdi:bluetooth-connect
          {% else %}
            mdi:bluetooth-off
          {% endif %}
        state: >
          {% set entita = 'sensor.acqua_lavello_cucina_segnale_bluetooth' %}
          {% set timeout_secondi = 900 %}
          {% set s = states(entita) %}
          {% if s in ['unavailable', 'unknown', 'none', ''] %}
            {{ false }}
          {% elif (as_timestamp(now()) - as_timestamp(states[entita].last_updated)) > timeout_secondi %}
            {{ false }}
          {% else %}
            {{ true }}
          {% endif %}

      # --- Acqua Studio ---
      - name: "Disponibilita Acqua Studio"
        unique_id: disponibilita_acqua_studio
        device_class: connectivity
        icon: >
          {% if this.state == 'on' %}
            mdi:bluetooth-connect
          {% else %}
            mdi:bluetooth-off
          {% endif %}
        state: >
          {% set entita = 'sensor.acqua_studio_segnale_bluetooth' %}
          {% set timeout_secondi = 900 %}
          {% set s = states(entita) %}
          {% if s in ['unavailable', 'unknown', 'none', ''] %}
            {{ false }}
          {% elif (as_timestamp(now()) - as_timestamp(states[entita].last_updated)) > timeout_secondi %}
            {{ false }}
          {% else %}
            {{ true }}
          {% endif %}

      # --- Acqua Stufa ---
      - name: "Disponibilita Acqua Stufa"
        unique_id: disponibilita_acqua_stufa
        device_class: connectivity
        icon: >
          {% if this.state == 'on' %}
            mdi:bluetooth-connect
          {% else %}
            mdi:bluetooth-off
          {% endif %}
        state: >
          {% set entita = 'sensor.acqua_stufa_segnale_bluetooth' %}
          {% set timeout_secondi = 900 %}
          {% set s = states(entita) %}
          {% if s in ['unavailable', 'unknown', 'none', ''] %}
            {{ false }}
          {% elif (as_timestamp(now()) - as_timestamp(states[entita].last_updated)) > timeout_secondi %}
            {{ false }}
          {% else %}
            {{ true }}
          {% endif %}

      # --- Acqua Tecnologico PT ---
      - name: "Disponibilita Acqua Tecnologico PT"
        unique_id: disponibilita_acqua_tecnologico_pt
        device_class: connectivity
        icon: >
          {% if this.state == 'on' %}
            mdi:bluetooth-connect
          {% else %}
            mdi:bluetooth-off
          {% endif %}
        state: >
          {% set entita = 'sensor.acqua_tecnologico_pt_segnale_bluetooth' %}
          {% set timeout_secondi = 900 %}
          {% set s = states(entita) %}
          {% if s in ['unavailable', 'unknown', 'none', ''] %}
            {{ false }}
          {% elif (as_timestamp(now()) - as_timestamp(states[entita].last_updated)) > timeout_secondi %}
            {{ false }}
          {% else %}
            {{ true }}
          {% endif %}
