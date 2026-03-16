# Feature: Modalità Test — allarme-core

**Data progettazione:** 2026-03-16
**Stato:** ✅ Implementato — 2026-03-16 — tutti i file validati PASS
**Scope:** allarme-core (non tocca safety-core)

---

## 1. Obiettivo

Permettere di attivare una **modalità test** dell'impianto di allarme in cui:

- I sensori rilevano e cambiano stato normalmente
- `sensor.allarme_core_sensori_aperti_filtrati` si aggiorna normalmente
- Node-RED riceve l'evento, lo elabora e può transitare lo stato a `triggered`
- Le automazioni HA girano normalmente (logbook, snapshot, memorizzazione sensori)
- **Le sole conseguenze reali (sirene, notifiche push, relay fisici) vengono soppresse da Node-RED** leggendo il flag HA

La modalità si disattiva automaticamente dopo un timeout configurabile dall'utente (default 30 min) o manualmente dalla plancia.

---

## 2. Architettura della soluzione

```
Sensore aperto
    │
    ▼
sensor.allarme_core_sensori_aperti_filtrati  (HA — invariato)
    │
    ▼
Node-RED legge il sensore
    │
    ├─► [controlla input_boolean.allarme_core_modalita_test]
    │        │
    │        ├─ OFF → esegue tutto normalmente (sirena, notifiche, ecc.)
    │        └─ ON  → imposta triggered MA salta le conseguenze fisiche
    │
    ▼
input_select.allarme_core_stato = triggered  (HA — invariato)
    │
    ▼
Automazioni HA girano normalmente (logbook, var, memorizzazione)
```

**Principio chiave:** HA non sa nulla del test mode a livello di state machine.
Il flag è un `input_boolean` che HA espone; Node-RED lo legge prima di eseguire
le azioni con effetti fisici (sirena, notifiche).

---

## 3. Nuove entità introdotte

| Entità | Tipo | Scopo |
|--------|------|-------|
| `input_boolean.allarme_core_modalita_test` | input_boolean | Flag ON/OFF — Node-RED lo legge |
| `input_number.allarme_core_test_timeout_minuti` | input_number | Timeout configurabile (5–120 min, default 30) |
| `input_datetime.allarme_core_test_attivato_alle` | input_datetime | Timestamp attivazione, usato dall'automazione timeout |
| `sensor.allarme_core_test_minuti_rimanenti` | template sensor | Mostra i minuti restanti prima della disattivazione auto |
| `script.allarme_core_attiva_test_mode` | script | Attiva il flag, salva timestamp, scrive nel logbook |
| `script.allarme_core_disattiva_test_mode` | script | Disattiva il flag, scrive nel logbook |

---

## 4. Modifiche chirurgiche per file

---

### FILE 1 — NUOVO: `allarme-core/allarme_core_test.yaml`

> File completamente nuovo da creare. Non tocca nessun file esistente.

```yaml
# ============================================================================
# ALLARME CORE — MODALITÀ TEST
# Permette di testare il flusso completo (sensori → Node-RED → triggered)
# senza far scattare conseguenze fisiche (sirene, notifiche, relay).
#
# Il flag input_boolean.allarme_core_modalita_test viene letto da Node-RED
# prima di eseguire azioni con effetti reali.
# Il timeout è configurabile tramite input_number.allarme_core_test_timeout_minuti.
# ============================================================================

# ----------------------------------------------------------------------------
# FLAG E CONFIGURAZIONE
# ----------------------------------------------------------------------------
input_boolean:
  allarme_core_modalita_test:
    name: "Allarme Core - Modalità Test"
    icon: mdi:test-tube

input_number:
  allarme_core_test_timeout_minuti:
    name: "Allarme Core - Timeout Test"
    min: 5
    max: 120
    step: 5
    initial: 30
    unit_of_measurement: min
    icon: mdi:timer

input_datetime:
  allarme_core_test_attivato_alle:
    name: "Allarme Core - Test Attivato Alle"
    has_date: true
    has_time: true

# ----------------------------------------------------------------------------
# SCRIPT
# ----------------------------------------------------------------------------
script:

  allarme_core_attiva_test_mode:
    alias: "Allarme Core - Attiva Modalità Test"
    icon: mdi:test-tube
    mode: single
    sequence:
      - service: input_boolean.turn_on
        target:
          entity_id: input_boolean.allarme_core_modalita_test
      - service: input_datetime.set_datetime
        target:
          entity_id: input_datetime.allarme_core_test_attivato_alle
        data:
          datetime: "{{ now().strftime('%Y-%m-%d %H:%M:%S') }}"
      - service: script.allarme_core_logbook_emit
        data:
          msg: >
            🧪 Modalità Test ATTIVATA —
            timeout {{ states('input_number.allarme_core_test_timeout_minuti') | int }} min

  allarme_core_disattiva_test_mode:
    alias: "Allarme Core - Disattiva Modalità Test"
    icon: mdi:test-tube-off
    mode: single
    sequence:
      - service: input_boolean.turn_off
        target:
          entity_id: input_boolean.allarme_core_modalita_test
      - service: script.allarme_core_logbook_emit
        data:
          msg: "🧪 Modalità Test DISATTIVATA"

# ----------------------------------------------------------------------------
# SENSOR — minuti rimanenti prima della disattivazione automatica
# ----------------------------------------------------------------------------
template:
  - sensor:
    - name: "Allarme Core Test Minuti Rimanenti"
      unique_id: allarme_core_test_minuti_rimanenti
      unit_of_measurement: "min"
      icon: mdi:timer-outline
      state: >
        {% if is_state('input_boolean.allarme_core_modalita_test', 'on') %}
          {% set attivato = states('input_datetime.allarme_core_test_attivato_alle') %}
          {% if attivato not in ['unknown', 'unavailable'] %}
            {% set timeout = states('input_number.allarme_core_test_timeout_minuti') | int %}
            {% set elapsed = (now() - attivato | as_datetime).total_seconds() / 60 %}
            {{ [timeout - elapsed | int, 0] | max }}
          {% else %}
            0
          {% endif %}
        {% else %}
          0
        {% endif %}
```

---

### FILE 2 — MODIFICA: `allarme-core/allarme_core_automazioni.yaml`

> Aggiungere **una sola nuova automazione** in fondo alla sezione `automation:`.
> Non toccare nessuna automazione esistente.

#### ⚠️ Perché NON usare `delay`

Un'automazione con `delay:` muore al riavvio di HA o al `reload config`.
Se il test mode è attivo e HA viene riavviato, il delay scompare e il flag
rimane `on` per sempre — esattamente il problema da evitare.

**Soluzione:** due trigger indipendenti che coprono entrambi i casi:

| Trigger | Caso coperto |
|---------|-------------|
| `platform: template` | Normale operatività — scatta quando il tempo calcolato da `input_datetime` raggiunge il timeout |
| `platform: homeassistant event: start` | Riavvio HA / reload config — se il flag è ancora `on`, lo disattiva subito |

Il `platform: template` **non usa delay**: HA rivaluta il template periodicamente
e lo rivaluta anche al boot. Se il timeout è già scaduto quando HA si riavvia,
il template è già `true` e l'automazione parte immediatamente.

Il trigger `homeassistant: start` è la rete di sicurezza: se HA riavvia
mentre il test è attivo, il test viene disattivato al boot indipendentemente
da quando era stato avviato.

**Aggiungere dopo l'ultima automazione esistente:**

```yaml
  # Disattiva automaticamente la modalità test.
  # Due trigger per coprire tutti i casi:
  #  - timeout_scaduto: template rivalutato da HA periodicamente e al boot
  #  - riavvio_ha:      se HA riavvia con il flag attivo, disattiva subito
  #    (il delay è morto al riavvio — questo trigger è la rete di sicurezza)
  - alias: "Allarme Core - Auto-disattiva modalità test al timeout"
    id: allarme_core_aut_test_auto_disattiva
    mode: single
    trigger:
      # Caso normale: timeout scaduto durante operatività continua
      - platform: template
        id: timeout_scaduto
        value_template: >
          {% if is_state('input_boolean.allarme_core_modalita_test', 'on') %}
            {% set attivato = states('input_datetime.allarme_core_test_attivato_alle') %}
            {% if attivato not in ['unknown', 'unavailable'] %}
              {% set timeout = states('input_number.allarme_core_test_timeout_minuti') | int %}
              {% set elapsed = (now() - attivato | as_datetime).total_seconds() / 60 %}
              {{ elapsed >= timeout }}
            {% else %}
              false
            {% endif %}
          {% else %}
            false
          {% endif %}
      # Rete di sicurezza: HA riavviato o config ricaricata mentre il test era attivo
      - platform: homeassistant
        id: riavvio_ha
        event: start
    condition:
      - condition: state
        entity_id: input_boolean.allarme_core_modalita_test
        state: "on"
    action:
      - choose:
          # Riavvio: messaggio specifico nel logbook
          - conditions:
              - condition: trigger
                id: riavvio_ha
            sequence:
              - service: script.allarme_core_logbook_emit
                data:
                  msg: "🧪 Modalità Test DISATTIVATA — riavvio HA rilevato (timer perso)"
              - service: input_boolean.turn_off
                target:
                  entity_id: input_boolean.allarme_core_modalita_test
        # Timeout normale: usa lo script standard (scrive nel logbook)
        default:
          - service: script.allarme_core_disattiva_test_mode
```

**Note importanti:**

- `mode: single` è corretto: non serve `restart` perché il template trigger
  si aggiorna da solo al variare del flag — non c'è un'istanza precedente da cancellare.
- Al riavvio, il test mode viene **sempre disattivato** se era attivo.
  È il comportamento più sicuro per un sistema di allarme: dopo un riavvio
  l'utente riattiva esplicitamente il test se necessario.
- Il `input_datetime.allarme_core_test_attivato_alle` persiste su disco (come tutti
  gli `input_datetime` in HA), quindi il calcolo del tempo scaduto è sempre corretto.

---

### FILE 3 — MODIFICA: `allarme-core/plancia_controllo.yaml`

> Tre modifiche chirurgiche alla dashboard esistente.

#### 3a — Banner di allerta test mode nell'header

**Sostituire** il blocco `header:` esistente (in fondo al file):

```yaml
# ❌ PRIMA
header:
  card:
    type: markdown
    content: |
      # Configurazione Allarme
```

**Con:**

```yaml
# ✅ DOPO
header:
  card:
    type: conditional
    conditions:
      - condition: state
        entity: input_boolean.allarme_core_modalita_test
        state: "on"
    card:
      type: markdown
      content: >
        <ha-alert alert-type="warning">
        🧪 **MODALITÀ TEST ATTIVA** —
        {{ states('sensor.allarme_core_test_minuti_rimanenti') }} min rimanenti.
        Sirene e notifiche sono disabilitate in Node-RED.
        </ha-alert>
```

> Nota: l'header mostra il banner **solo quando il test è attivo**; in condizioni normali
> non è visibile (comportamento atteso — meno rumore visivo in produzione).
> Se preferisci avere sempre un header con titolo, usa una `vertical-stack` card
> con il titolo fisso + la conditional per il banner.

#### 3b — Sezione controllo test mode nella prima grid (sezione "Controllo Allarme")

**Aggiungere** i seguenti card dopo l'ultimo tile esistente nella prima `- type: grid`
(quella con heading "Controllo Allarme", prima riga del file):

```yaml
      - type: heading
        heading: Modalità Test
        heading_style: title
        icon: mdi:test-tube
      - type: tile
        entity: input_boolean.allarme_core_modalita_test
        name: Test Mode
        vertical: false
        features_position: bottom
        grid_options:
          columns: 4
          rows: 1
        card_mod:
          style: >
            {% set on = is_state('input_boolean.allarme_core_modalita_test','on') %}
            ha-card {
              border-radius: 30px;
              box-shadow: 0 0 0 3px {{ 'orange' if on else 'green' }};
              padding-left: 12px;
            }
      - type: tile
        entity: sensor.allarme_core_test_minuti_rimanenti
        name: Scade tra
        vertical: false
        features_position: bottom
        grid_options:
          columns: 4
          rows: 1
      - type: tile
        entity: input_number.allarme_core_test_timeout_minuti
        name: Timeout
        vertical: false
        features_position: bottom
        features:
          - type: numeric-input
            mode: slider
        grid_options:
          columns: 4
          rows: 1
      - type: tile
        entity: script.allarme_core_attiva_test_mode
        name: Attiva Test
        icon: mdi:test-tube
        vertical: false
        tap_action:
          action: call-service
          service: script.turn_on
          target:
            entity_id: script.allarme_core_attiva_test_mode
        features_position: bottom
        grid_options:
          columns: 6
          rows: 1
      - type: tile
        entity: script.allarme_core_disattiva_test_mode
        name: Disattiva Test
        icon: mdi:test-tube-off
        vertical: false
        tap_action:
          action: call-service
          service: script.turn_on
          target:
            entity_id: script.allarme_core_disattiva_test_mode
        features_position: bottom
        grid_options:
          columns: 6
          rows: 1
```

#### 3c — Voce nel logbook (sezione Log)

Nessuna modifica necessaria: `script.allarme_core_attiva_test_mode` e
`script.allarme_core_disattiva_test_mode` chiamano già `allarme_core_logbook_emit`,
quindi gli eventi compaiono automaticamente nella timeline e nel logbook esistenti.

---

### FILE 4 — MODIFICA: Node-RED (indicazione, non YAML HA)

> Questa modifica è **nel flow Node-RED**, non in HA.
> Viene riportata qui per completezza del piano.

In ogni nodo del flow che esegue conseguenze fisiche
(sirena, notifica push, relay, TTS, ecc.), aggiungere **prima** un nodo
`current state` che legge `input_boolean.allarme_core_modalita_test`:

```
[sensor.allarme_core_sensori_aperti_filtrati cambia]
    │
    ▼
[current state: input_boolean.allarme_core_modalita_test]
    │
    ├─ state == "on"  →  [set triggered in HA]  →  [STOP — no conseguenze]
    └─ state == "off" →  [set triggered in HA]  →  [sirena] [notifica] [relay...]
```

**Entità HA da leggere in Node-RED:** `input_boolean.allarme_core_modalita_test`

---

## 5. Comportamento atteso end-to-end

| Evento | Test OFF | Test ON |
|--------|----------|---------|
| Sensore aperto → `sensori_aperti_filtrati` si aggiorna | ✅ | ✅ |
| Node-RED rileva il cambio | ✅ | ✅ |
| `input_select.allarme_core_stato` → `triggered` | ✅ | ✅ |
| Automazioni HA (logbook, var, memorizzazione) | ✅ | ✅ |
| Sirena fisica | ✅ | ❌ bloccata da Node-RED |
| Notifica push / TTS | ✅ | ❌ bloccata da Node-RED |
| Relay / attuatori fisici | ✅ | ❌ bloccata da Node-RED |
| Logbook HA: evento "Test ATTIVATO/DISATTIVATO" | — | ✅ |
| Dashboard: banner arancione | ❌ nascosto | ✅ visibile |
| Auto-disattivazione al timeout | — | ✅ dopo N min |

---

## 6. Riepilogo file da toccare

| File | Tipo modifica | Righe stimate |
|------|--------------|---------------|
| `allarme-core/allarme_core_test.yaml` | **Nuovo file** | ~80 righe |
| `allarme-core/allarme_core_automazioni.yaml` | Aggiungi automazione in fondo | +15 righe |
| `allarme-core/plancia_controllo.yaml` | Modifica header + aggiungi sezione test | +50 righe |
| Node-RED flow | Nodo condizionale (fuori scope HA) | — |

**File NON toccati:** `allarme_core.yaml`, `allarme_core_sensori.yaml`,
`allarme_core_supporto.yaml`, `allarme_core_log.yaml`, `safety-core/*`

---

## 7. Checklist deploy

- [x] Approvato il design da parte dell'utente
- [x] Verificato che `script.allarme_core_logbook_emit` esiste in `allarme_core_log.yaml`
- [x] `allarme_core_test.yaml` creato (97 righe) — validato PASS
- [x] Automazione `allarme_core_aut_test_auto_disattiva` aggiunta — validata PASS
- [x] `plancia_controllo.yaml` aggiornato (header + sezione test) — validato PASS
- [x] Warning cosmetico corretto: `default` usa `script.turn_on` + `target:` per uniformità
- [ ] **TODO utente** — Aggiungere `allarme_core_test.yaml` al package o incluso in `configuration.yaml`, poi riavviare HA
- [ ] **TODO utente** — Verificare in Developer Tools → Stati che le 6 nuove entità compaiono
- [ ] **TODO utente** — Aggiornare il flow Node-RED: nodo condizionale che legge `input_boolean.allarme_core_modalita_test` prima di sirena/notifiche/relay
- [ ] **TODO utente** — Test funzionale: attiva test → apri sensore → verifica `triggered` in HA → verifica blocco in Node-RED
- [ ] **TODO utente** — Test timeout: attiva test → attendi scadenza → verifica disattivazione automatica
- [ ] **TODO utente** — Test resilienza riavvio: attiva test → riavvia HA → verifica disattivazione al boot + messaggio logbook "riavvio HA rilevato"
- [ ] **TODO utente** — Test resilienza reload: attiva test → Reload Automations → verifica disattivazione

---

*Documento di progettazione — nessun file modificato fino ad approvazione esplicita*
