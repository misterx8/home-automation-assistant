# Report: Implementazione Batteria e Stato Online/Offline per Sensori Wireless
## Progetto: Allarme Core — Home Assistant + Zigbee2MQTT

**Data:** 2026-03-18
**Branch:** `claude/sensor-battery-status-dVaHj`
**Stato:** Pianificazione

---

## 1. Contesto e Obiettivo

Il progetto **allarme-core** gestisce attualmente circa 50+ sensori fisici tramite wrapper template
(`binary_sensor.allarme_core_*`). Ogni wrapper espone già gli attributi `sensore_origine` /
`sensori_origine` che puntano alle entità fisiche reali sottostanti.

**Zigbee2MQTT (Z2M)**, quando integrato con Home Assistant, espone automaticamente per ogni
dispositivo Zigbee:
- `sensor.<nome_dispositivo>_battery` → livello batteria in %
- `binary_sensor.<nome_dispositivo>_availability` → stato online/offline

L'obiettivo è integrare queste due entità per ogni sensore wireless nel sistema allarme-core,
senza impattare i sensori cablati (filari) e senza modificare i file esistenti.

---

## 2. Come Zigbee2MQTT Espone le Entità

Z2M utilizza il **friendly name** del dispositivo configurato in `zigbee2mqtt/configuration.yaml`
come base per i nomi delle entità HA:

```
Dispositivo Z2M: "porta_ingresso_taverna"
  → binary_sensor.porta_ingresso_taverna             (contatto aperto/chiuso)
  → sensor.porta_ingresso_taverna_battery            (batteria %)
  → binary_sensor.porta_ingresso_taverna_availability (online/offline)
```

La condizione necessaria è che il **friendly name Z2M coincida con la parte centrale**
dell'entity_id fisico già usato nei wrapper (quello in `sensore_origine`).

---

## 3. Strategia Raccomandata: Convenzione di Naming Automatica

### Approccio scelto: Derivazione automatica dal `sensore_origine`

Anziché modificare i 50+ wrapper esistenti in `allarme_core_sensori.yaml`, il nuovo modulo
**deriva automaticamente** le entità batteria e disponibilità dal valore di `sensore_origine`
già presente su ogni wrapper.

**Logica di derivazione:**
```
sensore_origine: "binary_sensor.porta_ingresso_taverna"
  → strip prefisso dominio: "porta_ingresso_taverna"
  → batteria:       "sensor.porta_ingresso_taverna_battery"
  → disponibilità:  "binary_sensor.porta_ingresso_taverna_availability"
```

Questo è possibile perché Z2M assegna batteria e disponibilità usando lo **stesso nome base**
del sensore binario.

**Vantaggi:**
- Zero modifiche ai file esistenti
- Funziona automaticamente per tutti i sensori presenti e futuri
- Coerente con il pattern già usato per `allarme_core_anomalie_sensori.yaml`
- I sensori cablati (senza entità `_battery` o `_availability`) vengono ignorati (stato `unknown`)

---

## 4. Architettura del Nuovo Modulo

### 4.1 Nuovi File da Creare

```
allarme-core/
├── allarme_core_batterie.yaml               ← Sensori template aggregati + input_number soglia
├── allarme_core_batterie_automazioni.yaml   ← Automazioni alert batteria/offline
└── plancia_controllo.yaml                   ← Aggiornamento sezione dashboard (file esistente)
```

### 4.2 Nessun file esistente da modificare (Fase 1)

Nella fase iniziale, tutti i nuovi sensori vengono aggiunti nei nuovi file. La dashboard viene
aggiornata come estensione della sezione esistente.

---

## 5. Entità da Creare (`allarme_core_batterie.yaml`)

### 5.1 Controllo Soglia Configurabile

```yaml
input_number:
  allarme_core_soglia_batteria_bassa:
    name: "Allarme Core Soglia Batteria Bassa"
    min: 5
    max: 50
    step: 5
    unit_of_measurement: "%"
    initial: 20
    icon: mdi:battery-alert
```

### 5.2 Sensori Aggregati Globali

| Entità | Tipo | Descrizione |
|--------|------|-------------|
| `sensor.allarme_core_batteria_minima` | `sensor` | % batteria più bassa tra tutti i sensori wireless |
| `sensor.allarme_core_batterie_basse` | `sensor` | Numero di sensori con batteria < soglia (default 20%) |
| `sensor.allarme_core_sensori_offline` | `sensor` | Numero di sensori attualmente offline |
| `binary_sensor.allarme_core_batteria_critica` | `binary_sensor` | ON se almeno un sensore ha batteria critica |
| `binary_sensor.allarme_core_sensore_offline` | `binary_sensor` | ON se almeno un sensore è offline |

### 5.3 Attributi dei Sensori Aggregati

**`sensor.allarme_core_batteria_minima`:**
```yaml
attributes:
  dettaglio:      # lista completa sensori wireless con batteria rilevata
    - wrapper: binary_sensor.allarme_core_porta_ingresso_taverna
      fisico:  binary_sensor.porta_ingresso_taverna
      batteria: 87
      nome: "Porta Ingresso Taverna"
  sensori_bassi:  # solo quelli sotto soglia
    - wrapper: ...
      fisico:  ...
      batteria: 15
```

**`binary_sensor.allarme_core_sensore_offline`:**
```yaml
attributes:
  sensori_offline:
    - wrapper: binary_sensor.allarme_core_movimento_cucina
      fisico:  binary_sensor.movimento_cucina
      nome: "Movimento Cucina"
  contatore: 1
```

---

## 6. Logica Template — Batteria Minima (pseudocodice Jinja2)

```jinja2
{# Itera su tutti i wrapper allarme-core che hanno sensore_origine #}
{% set ns = namespace(batterie=[], dettaglio=[], bassi=[]) %}
{% set soglia = states('input_number.allarme_core_soglia_batteria_bassa') | int(20) %}

{% for s in states.binary_sensor
    | selectattr('entity_id', 'search', 'binary_sensor\.allarme_core_')
    | rejectattr('entity_id', 'search', '_attivo|_filtrati|_zone_valida|_aperto|_anomalia|_critica|_offline') %}

  {% set fisico = state_attr(s.entity_id, 'sensore_origine') %}
  {% if fisico is not none %}
    {% set nome_base = fisico.split('.')[1] %}
    {% set batteria_entity = 'sensor.' ~ nome_base ~ '_battery' %}
    {% set batt = states(batteria_entity) %}
    {% if batt not in ['unknown', 'unavailable', 'none'] %}
      {# Sensore wireless con batteria disponibile #}
      {% set batt_int = batt | int(0) %}
      {% set ns.batterie = ns.batterie + [batt_int] %}
      {% set ns.dettaglio = ns.dettaglio + [{'wrapper': s.entity_id, 'fisico': fisico, 'batteria': batt_int}] %}
      {% if batt_int < soglia %}
        {% set ns.bassi = ns.bassi + [{'wrapper': s.entity_id, 'fisico': fisico, 'batteria': batt_int}] %}
      {% endif %}
    {% endif %}
  {% endif %}

{% endfor %}

{{ ns.batterie | min if ns.batterie | count > 0 else 'unknown' }}
```

---

## 7. Automazioni (`allarme_core_batterie_automazioni.yaml`)

### 7.1 Alert Batteria Bassa

| # | Alias | Trigger | Debounce | Azione |
|---|-------|---------|----------|--------|
| 1 | `allarme_core_alert_batteria_bassa` | `binary_sensor.allarme_core_batteria_critica` → ON | — | Log timeline + notifica |
| 2 | `allarme_core_alert_batteria_risolta` | `binary_sensor.allarme_core_batteria_critica` → OFF | — | Log risoluzione timeline |

### 7.2 Alert Sensore Offline

| # | Alias | Trigger | Debounce | Azione |
|---|-------|---------|----------|--------|
| 3 | `allarme_core_alert_sensore_offline` | `binary_sensor.allarme_core_sensore_offline` → ON | 30s | Log timeline + notifica |
| 4 | `allarme_core_alert_sensore_tornato_online` | `binary_sensor.allarme_core_sensore_offline` → OFF | — | Log risoluzione |

**Nota:** Il debounce di 30 secondi sull'offline evita falsi allarmi durante riavvii del
coordinatore Zigbee o brevi interruzioni di rete mesh.

Tutte le automazioni riusano `script.allarme_core_logbook_emit` già esistente per integrarsi
nella timeline attuale.

---

## 8. Aggiornamento Dashboard (`plancia_controllo.yaml`)

### Nuova Sezione: "Batterie & Connettività"

Layout consigliato:

```
┌─────────────────────────────────────────────────────┐
│  🔋 BATTERIE & CONNETTIVITÀ                         │
├──────────────────┬──────────────┬───────────────────┤
│  Batteria Min    │ Basse (<20%) │ Sensori Offline   │
│  [gauge 0-100%]  │  [badge N]   │    [badge N]      │
├──────────────────┴──────────────┴───────────────────┤
│  Lista sensori con batteria sotto soglia            │
│  [entities card — da attributo sensori_bassi]       │
├─────────────────────────────────────────────────────┤
│  Lista sensori offline                              │
│  [entities card — da attributo sensori_offline]     │
├─────────────────────────────────────────────────────┤
│  Soglia batteria bassa:  [slider 5% ──────── 50%]  │
└─────────────────────────────────────────────────────┘
```

**Card gauge batteria minima:**
```yaml
type: gauge
entity: sensor.allarme_core_batteria_minima
min: 0
max: 100
severity:
  green: 30
  yellow: 20
  red: 0
```

---

## 9. Gestione Sensori Cablati (Filari)

I sensori cablati **non hanno entità `_battery` o `_availability`** in Z2M.
Il template gestisce questo automaticamente: se `states('sensor.xyz_battery')` restituisce
`unknown` o `unavailable`, il sensore viene semplicemente **ignorato** senza errori.

Il risultato: i sensori cablati non compaiono nella lista batterie, senza alcuna modifica
al loro wrapper.

---

## 10. Prerequisiti e Configurazione Z2M

### 10.1 Abilitare `availability` in Zigbee2MQTT

In `zigbee2mqtt/configuration.yaml`:

```yaml
# Opzione semplice (tutti i dispositivi)
availability: true

# Opzione granulare (timeout configurabili)
availability:
  active:
    timeout: 10    # minuti dopo cui un dispositivo attivo è offline
  passive:
    timeout: 1500  # minuti per dispositivi passivi (sensori porta/finestra)
```

### 10.2 Naming Convention Z2M → Entità HA

| Z2M Friendly Name | `sensore_origine` (wrapper) | Batteria derivata | Disponibilità derivata |
|---|---|---|---|
| `porta_ingresso_taverna` | `binary_sensor.porta_ingresso_taverna` | `sensor.porta_ingresso_taverna_battery` | `binary_sensor.porta_ingresso_taverna_availability` |
| `movimento_cucina` | `binary_sensor.movimento_cucina` | `sensor.movimento_cucina_battery` | `binary_sensor.movimento_cucina_availability` |
| `finestra_studio` | `binary_sensor.finestra_studio` | `sensor.finestra_studio_battery` | `binary_sensor.finestra_studio_availability` |

### 10.3 Fallback con Attributi Espliciti

Se alcuni dispositivi Z2M non seguono la convenzione di naming, si possono aggiungere attributi
espliciti opzionali ai wrapper interessati in `allarme_core_sensori.yaml`:

```yaml
# Esempio per sensore con naming non standard
binary_sensor.allarme_core_vibrazione_ingresso_taverna:
  attributes:
    batteria_origine: "sensor.sensore_vibrazione_xyz_battery"
    disponibilita_origine: "binary_sensor.sensore_vibrazione_xyz_availability"
```

Il template aggregatore legge prima `batteria_origine` (esplicito), e solo se assente
deriva automaticamente dal `sensore_origine`.

---

## 11. Riepilogo Entità Prodotte

| Entità | Tipo | Ruolo UI |
|--------|------|----------|
| `sensor.allarme_core_batteria_minima` | sensor | Gauge principale |
| `sensor.allarme_core_batterie_basse` | sensor | Badge count + lista attributo |
| `sensor.allarme_core_sensori_offline` | sensor | Badge count + lista attributo |
| `binary_sensor.allarme_core_batteria_critica` | binary_sensor | Trigger automazioni |
| `binary_sensor.allarme_core_sensore_offline` | binary_sensor | Trigger automazioni |
| `input_number.allarme_core_soglia_batteria_bassa` | input_number | Slider configurazione |

**Totale: 6 nuove entità — zero modifiche alle entità esistenti.**

---

## 12. Piano di Implementazione — Fasi

### Fase 1 — Infrastruttura Base
- [ ] Creare `allarme-core/allarme_core_batterie.yaml`
  - `input_number` soglia
  - `sensor.allarme_core_batteria_minima` con attributi dettaglio
  - `sensor.allarme_core_batterie_basse` con attributi sensori_bassi
  - `sensor.allarme_core_sensori_offline` con attributi sensori_offline
  - `binary_sensor.allarme_core_batteria_critica`
  - `binary_sensor.allarme_core_sensore_offline`
- [ ] Verificare naming convention Z2M sui dispositivi reali installati

### Fase 2 — Automazioni
- [ ] Creare `allarme-core/allarme_core_batterie_automazioni.yaml`
  - 4 automazioni alert (batteria bassa/risolta, offline/tornato online)
  - Integrazione con `script.allarme_core_logbook_emit`

### Fase 3 — Dashboard
- [ ] Aggiungere sezione "Batterie & Connettività" in `plancia_controllo.yaml`
  - Gauge batteria minima
  - Badge contatori
  - Liste sensori filtrate da attributi
  - Slider soglia

### Fase 4 — Ottimizzazione (opzionale)
- [ ] Aggiungere `batteria_origine` esplicito ai wrapper che non seguono la convenzione
- [ ] Valutare integrazione con il sistema anomalie esistente (`allarme_core_anomalie_sensori.yaml`)
- [ ] Aggiungere variabile `var` persistente per storico batterie basse

---

## 13. Compatibilità con il Sistema Esistente

| File / Componente | Impatto | Note |
|---|---|---|
| `allarme_core_sensori.yaml` | Nessuna modifica | I wrapper esistenti non cambiano |
| `allarme_core_anomalie_sensori.yaml` | Nessuna modifica | Sistemi paralleli e indipendenti |
| `allarme_core_log.yaml` | Riuso diretto | Le automazioni chiamano `allarme_core_logbook_emit` |
| `plancia_controllo.yaml` | Estensione | Nuova sezione aggiunta |
| `var/allarme_core.yaml` | Eventuale (Fase 4) | Solo se si decide di persistere lo storico |

---

*Report generato il 2026-03-18 — Claude Code — claude/sensor-battery-status-dVaHj*
