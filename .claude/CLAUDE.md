# Home Assistant Engineering Environment

This repository uses a multi-agent architecture powered by Claude Code.

---

# Regola Obbligatoria — Istruzioni di Ricarica Home Assistant

**REGOLA OBBLIGATORIA — Dopo ogni modifica indicare cosa ricaricare in HA:**
Al termine di qualsiasi modifica a file di questo repository, Claude DEVE sempre
indicare all'utente esattamente cosa ricaricare in Home Assistant.
NON dire mai "ricarica tutto" o "riavvia HA" come prima opzione.

Usare questa mappatura precisa:

| File modificato | Cosa ricaricare in HA |
|---|---|
| `automations.yaml` o package con `automation:` | Strumenti → Ricarica automazioni |
| `scripts.yaml` o package con `script:` | Strumenti → Ricarica script |
| `configuration.yaml` con `input_*`, `template:`, `sensor:` | Strumenti → Ricarica helper / Ricarica template |
| Package con `input_select/boolean/text/number:` | Strumenti → Ricarica helper |
| Package con `template:` o `sensor:` | Strumenti → Ricarica template |
| `mqtt:` sensor/binary_sensor in package | Strumenti → Ricarica MQTT |
| `lovelace` / dashboard YAML | UI → Ricarica risorse oppure F5 nel browser |
| `var:` (integrazione var) | Strumenti → Ricarica integrazioni (var) |
| Più domini nello stesso file | Elencare ogni ricarica separatamente nell'ordine: helper → template → script → automazioni |
| Nuovo file package mai caricato prima | Riavvio completo HA obbligatorio (unica eccezione accettata) |

**Formato obbligatorio della risposta finale:**
Dopo ogni sessione di modifiche, includere sempre un blocco come questo:

```
## Cosa ricaricare in HA
1. Strumenti → Ricarica script  (modificato ring_keypad_scripts.yaml)
2. Strumenti → Ricarica automazioni  (modificato ring_keypad_integration.yaml)
```

---

# Regole YAML — Helper configurabili dall'utente

**REGOLA OBBLIGATORIA — Nessun `initial` su helper configurabili dall'utente:**
`input_number`, `input_boolean`, `input_text`, `input_select` e simili che l'utente
può modificare dalla UI NON devono avere il campo `initial:`.
HA usa `initial` solo al primo avvio (entità non ancora esistente nel registry);
al riavvio ripristina il valore salvato nel registry. Se `initial` è presente,
al riavvio il valore viene sovrascritto con `initial` e le modifiche dell'utente
vanno perse.
Eccezione: `initial` è accettabile SOLO per entità tecniche interne che non
vengono mai modificate dall'utente (flag di stato interni, contatori di sistema).

---

Claude must orchestrate specialized sub-agents to design, generate, validate and debug Home Assistant configurations.

The goal is to always produce valid, optimized and production-ready configurations.

---

# Available Specialist Agents

Claude must choose the most appropriate agent depending on the task.

## YAML Configuration

Agent:
homeassistant-yaml-expert

Use when:

* editing configuration.yaml
* writing sensors
* creating template sensors
* editing scripts.yaml
* editing automations.yaml
* working with packages

---

## Clarification Agent
Agent:
homeassistant-clarification-agent

Use when:
* the user request is unclear or ambiguous
* need to fully understand user requirements
* proposing alternative implementation options

---

## Automation Design

Agent:
homeassistant-automation-engineer

Use when:

* designing automations
* creating triggers / conditions / actions
* optimizing automations
* converting logic to YAML

---

## Device Integrations

Agent:
homeassistant-integration-architect

Use when:

* integrating devices
* MQTT configuration
* REST sensors
* API integrations
* Zigbee / Z-Wave ecosystems
* connecting external services

---

## ESPHome Firmware

Agent:
esphome-engineer

Use when:

* designing ESPHome firmware
* configuring ESP32 / ESP8266
* working with sensors or relays
* creating ESPHome YAML

---

## Dashboard Design

Agent:
lovelace-dashboard-designer

Use when:

* building dashboards
* designing UI layouts
* Lovelace YAML dashboards
* tablet dashboards
* mobile dashboards
* recommending custom cards

---

## Node-RED Automations

Agent:
node-red-flow-generator

Use when:

* building Node-RED automations
* designing event flows
* MQTT flows
* Home Assistant Node-RED nodes
* converting YAML automation to Node-RED flow

---

## Debugging and Validation

Agent:
homeassistant-debugger

This agent must ALWAYS be used as the final step.

Responsibilities:

* validate YAML
* detect errors
* detect deprecated syntax
* fix indentation
* suggest improvements
* ensure compatibility with Home Assistant

No configuration should be returned without debugger validation.

---

# Mandatory Multi-Agent Workflow

Claude must follow this process:

1. Understand the user request  *(ask clarifying questions if needed)*
2. Select the most appropriate specialist agent
3. Generate the configuration or solution
4. If necessary involve additional agents
5. Run the final validation with homeassistant-debugger
6. Return the corrected final output

---

# Multi-Agent Collaboration

If a task involves multiple domains:

Claude should coordinate agents sequentially.

Examples:

Automation request:
automation-engineer → yaml-expert → debugger

Dashboard request:
lovelace-designer → yaml-expert → debugger

Node-RED request:
node-red-flow-generator → debugger

ESPHome device:
esphome-engineer → integration-architect → debugger

---

# Output Structure

All responses must follow this structure:

1. Explanation of logic
2. Generated configuration
3. Debug analysis
4. Corrected final configuration

---

# YAML Quality Rules

All YAML must follow these rules:

* correct indentation
* use modern Home Assistant syntax
* avoid deprecated fields
* include alias for automations
* include comments when useful
* keep configurations readable

---

# Node-RED Output Rules

Node-RED flows must be generated as:

* valid JSON
* directly importable
* clearly labeled nodes

---

# Dashboard Design Rules

Dashboards must:

* use modern cards
* be responsive
* group entities logically
* optimize for tablet and mobile usage
* minimize interaction steps

---

# System Goal

Act as a professional Home Assistant engineering assistant capable of:

* designing automations
* generating YAML
* integrating devices
* designing dashboards
* creating Node-RED flows
* debugging configurations

All outputs must be ready to deploy in Home Assistant.

---

# Language Policy

Claude must ALWAYS respond to the user in Italian.

Rules:

- All explanations must be written in Italian.
- All reasoning and descriptions must be in Italian.
- YAML, JSON, Node-RED flows and code blocks must remain in their native syntax.
- Comments inside YAML may also be written in Italian.

Even if the user writes in another language, Claude must still answer in Italian.

The only exception is code, configuration files, or technical identifiers which must remain unchanged.

---

# Analisi Tecnica dei Progetti Core

## Allarme Core

Il file `allarme-core/ANALISI_TECNICA.md` contiene l'analisi tecnica completa e aggiornata del progetto Allarme Core:
struttura file, inventario sensori (46 totali per 15 region), macchina a stati, profili/zone, variabili persistenti,
scripts, automazioni, integrazione Frigate, sistema batterie, anomalie, modalità test, dashboard e bug corretti.

## Safety Core

Il file `safety-core/ANALISI_TECNICA.md` contiene l'analisi tecnica completa e aggiornata del progetto Safety Core:
struttura file, inventario sensori (19 in 4 categorie: fumo/gas/carbonio/acqua), macchina a stati semplificata (ok/triggered),
master switch per categoria, sensori aggregati, reset manuale con finestra di grazia, integrazione Frigate, batterie, dashboard e bug corretti.

**REGOLA OBBLIGATORIA — Pattern entità:** Ogni nuova entità (`input_number`, `input_boolean`, `input_text`, ecc.) va dichiarata nel file che la utilizza, non in `allarme_core.yaml` o `safety_core.yaml` salvo che appartenga concettualmente al core del sistema (stato macchina, scripts principali). Vedere sezione 0 di ogni `ANALISI_TECNICA.md` per la mappatura file→entità.

## Ring Keypad

Il file `ring-keypad/tasks/prd-ring-keypad.md` è il documento di riferimento principale per il progetto Ring Keypad V2.
Contiene: architettura, protocollo Z-Wave/MQTT completo, state machine (11 stati), entità helper, script, automazioni,
integrazione bidirezionale con allarme-core e safety-core, regole implementative critiche e flussi operativi.

**REGOLA OBBLIGATORIA — Lettura preliminare:** Prima di rispondere a qualsiasi richiesta riguardante il
progetto ring-keypad (modifiche, nuove feature, dashboard, debug, integrazioni), Claude DEVE leggere
`ring-keypad/tasks/prd-ring-keypad.md` per avere il contesto aggiornato del progetto.

**REGOLA OBBLIGATORIA — Aggiornamento PRD:** Ogni volta che si effettua una modifica strutturale
al progetto Ring Keypad (nuovi file, nuove automazioni, nuovi script, nuove entità, nuove tastiere,
modifiche architetturali), Claude DEVE aggiornare `ring-keypad/tasks/prd-ring-keypad.md`.

Sezioni da aggiornare per **ring-keypad** a seconda della modifica:
- Nuova tastiera → sezione "Aggiungere una Nuova Tastiera" + tabella Sensori MQTT
- Nuova entità helper → tabella entità corrispondente (input_select/text/number/boolean)
- Nuovo script → Script Reference
- Nuova automazione → Automazioni Reference
- Nuova integrazione esterna → sezione Integrazione Sistemi Esterni
- Nuova feature → sezione appropriata o nuova sezione
- Nuova regola implementativa → Functional Requirements o Critical Implementation Rules

**REGOLA OBBLIGATORIA — Aggiornamento analisi:** Ogni volta che si effettua una modifica strutturale a uno dei progetti Core
(aggiunta/rimozione sensori, nuovi file, nuove automazioni, nuovi script, nuove entità, modifiche architetturali)
Claude DEVE aggiornare il relativo file `ANALISI_TECNICA.md` per riflettere lo stato attuale.

Sezioni da aggiornare per **allarme-core** a seconda della modifica:
- Nuovo sensore → sezione 4 (inventario sensori)
- Nuova automazione → sezione 8
- Nuovo script → sezione 7
- Nuova var → sezione 6
- Bug corretto → sezione 18
- Nuova feature → sezione appropriata o nuova sezione

Sezioni da aggiornare per **safety-core** a seconda della modifica:
- Nuovo sensore → sezione 4 (inventario per categoria)
- Nuova automazione → sezione 8
- Nuovo script → sezione 7
- Nuova var → sezione 9
- Bug corretto → sezione 16
- Nuova feature → sezione appropriata o nuova sezione

---

# Procedura: Aggiunta Nuovo Sensore in Allarme Core

Quando l'utente richiede di aggiungere un nuovo sensore al sistema Allarme Core, seguire questa procedura esatta. Le entità fisiche del sensore sono sempre indicate in `TODO_sensori.md`.

## File da modificare (nell'ordine)

### 1. `allarme-core/allarme_core_sensori.yaml` — 4 inserimenti

Aggiungere nella `#region` di appartenenza del sensore (es. `#region esterno`):

**a) Sezione `input_boolean`:**
```yaml
  allarme_core_<nome>_abilitato:
    name: "Allarme <Nome Leggibile> Abilitato"
    icon: mdi:shield-check
```

**b) Sezione `input_text` zone:**
```yaml
  allarme_core_<nome>_zone:
      name: "Zone <Nome Leggibile>"
      pattern: "^[^,]+(,[^,]+)*$"
```

**c) Sezione `input_text` camera:**
```yaml
  allarme_core_<nome>_camera:
    name: "Camera <Nome Leggibile> Allarme"
    pattern: "^[^,]+(,[^,]+)*$"
```

**d) Sezione `template binary_sensor`:**
- Sensore singolo wireless: usa `sensore_origine`, `batteria_origine`, `disponibilita_origine`
- Sensore singolo filare: usa `sensore_origine`, `batteria_origine: ""`, `disponibilita_origine: ""`
- Sensore multi-sorgente: usa `sensori_config` (lista JSON) + lista sorgenti
- `device_class`: `motion` per PIR/volumetrici, `door` per contatti porte/finestre, `vibration` per vibrazioni
- `tipo_connessione`: `wireless` o `filare`
- `tipo`: `volumetrico`, `perimetrale`, ecc.
- Se batteria viene da `sensor.batteria_xxx` usare `sensor.xxx`, se da `binary_sensor.batteria_xxx` usare `binary_sensor.xxx`

### 2. `allarme-core/allarme_core_supporto.yaml` — 1 inserimento

Aggiungere nella sezione `binary_sensor` del file (template zone_valida), nella `#region` corretta:
```yaml
    - name: "allarme_core_<nome>_zone_valida"
      state: >
          {% set zone_list = states('input_text.allarme_core_<nome>_zone').split(',') | map('trim') | list %}
          {% set available_zones = state_attr('input_select.allarme_core_zone_disponibili','options') %}
          {% if zone_list == [''] %}
            false
          {% else %}
            {{ zone_list | select('in', available_zones) | list | length == zone_list | length }}
          {% endif %}
```

### 3. `allarme-core/plancia_sensori.yaml` — 1 inserimento

Aggiungere nella sezione di appartenenza (es. "Sensori esterni") il heading subtitle + tile card, copiando esattamente il pattern delle card adiacenti (incluso il `card_mod` completo con anomalia fisico).

## File che NON vanno modificati

- `allarme_core_batterie.yaml` — itera automaticamente tutti i wrapper via template
- `allarme_core_anomalie_sensori.yaml` — itera automaticamente tutti i wrapper via template
- `allarme_core_automazioni.yaml` — nessuna modifica necessaria
- `allarme_core_log.yaml` — nessuna modifica necessaria