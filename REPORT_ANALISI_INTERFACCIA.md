# Report di Analisi — Interfacce Lovelace (plancia_controllo.yaml)

**Data analisi:** 2026-03-16
**Ultima modifica:** 2026-03-16
**Agenti utilizzati:** lovelace-dashboard-designer, homeassistant-yaml-expert, homeassistant-debugger
**File analizzati:**
- `allarme-core/plancia_controllo.yaml` (~2670 righe)
- `safety-core/plancia_controllo.yaml` (~1070 righe)
- `allarme-core/allarme_core_log.yaml` (modificato per FR-02)
- `safety-core/safety_core_log.yaml` (modificato per FR-02)

> **Legenda stato:** ✅ Corretto · 🔴 Aperto · 🟡 Parzialmente corretto
> **Legenda severità:** 🚨 Critico · 🟠 Medio · 🔵 Basso

---

## Indice

1. [Bug Critici — allarme-core](#1-bug-critici--allarme-core)
2. [Bug Medi/Bassi — allarme-core](#2-bug-medibassi--allarme-core)
3. [Bug Critici — safety-core](#3-bug-critici--safety-core)
4. [Bug Medi/Bassi — safety-core](#4-bug-medibassi--safety-core)
5. [Incoerenze tra i due file](#5-incoerenze-tra-i-due-file)
6. [Funzionalità Raccomandate](#6-funzionalit%C3%A0-raccomandate)
7. [Piano di Intervento](#7-piano-di-intervento)

---

## 1. Bug Critici — allarme-core

---

### 🔴 AC-UI-01 · 🚨 CRITICO — `action: toggle` su "Cancella tutti i log"
**File:** `allarme-core/plancia_controllo.yaml` · righe 2552–2555

La card che punta a `script.allarme_core_logbook_cancella_tutto` usa `action: toggle` come `tap_action` **e** come `icon_tap_action`. Il toggle su uno script lo **interrompe se è già in esecuzione**, invece di ri-avviarlo. Se lo script impiega qualche secondo (es. cancella N entries) e l'utente tocca di nuovo, lo ferma a metà. Inoltre l'`icon_tap_action: action: toggle` è ridondante e raddoppia il rischio.

La forma corretta è `action: call-service` con `service: script.turn_on` e `target.entity_id`, come già usato in safety-core.

```yaml
# ❌ ATTUALE (errato)
tap_action:
  action: toggle
icon_tap_action:
  action: toggle

# ✅ CORRETTO
tap_action:
  action: call-service
  service: script.turn_on
  target:
    entity_id: script.allarme_core_logbook_cancella_tutto
```

---

## 2. Bug Medi/Bassi — allarme-core

---

### 🔴 AC-UI-02 · 🟠 MEDIO — Heading "Controllo Allarme" senza icona
**File:** `allarme-core/plancia_controllo.yaml` · riga 11

L'heading di apertura del primo pannello non ha il campo `icon:`. Tutti gli altri heading principali del file lo hanno valorizzato (`mdi:home-floor-1`, `mdi:home-floor-0`, `mdi:pine-tree`, `mdi:math-log`), e la dashboard di safety-core usa `icon: mdi:shield-alert`. L'assenza crea incoerenza visiva evidente.

```yaml
# ❌ ATTUALE
- type: heading
  heading_style: title
  heading: Controllo Allarme

# ✅ CORRETTO
- type: heading
  heading_style: title
  heading: Controllo Allarme
  icon: mdi:shield-lock
```

---

### 🔴 AC-UI-03 · 🟠 MEDIO — Icona `far:image` — dipendenza da hacs-fontawesome
**File:** `allarme-core/plancia_controllo.yaml` · riga 2634

L'heading "Immagini" usa `icon: far:image` (Font Awesome Regular). Questa sintassi richiede obbligatoriamente il custom component `hacs-fontawesome`. Se non installato, l'icona è invisibile senza errori espliciti in UI. L'alternativa nativa MDI `mdi:image-multiple` non richiede dipendenze aggiuntive.

```yaml
# ❌ ATTUALE
icon: far:image

# ✅ CORRETTO
icon: mdi:image-multiple
```

---

### 🔴 AC-UI-04 · 🟠 MEDIO — Card "Ultimo Evento" ridondante nella sezione Log
**File:** `allarme-core/plancia_controllo.yaml` · righe 2620–2623

Esiste una card markdown standalone "Allarme Core – Ultimo Evento" che mostra esclusivamente:
```
**{{ states('input_text.allarme_core_logbook_ultimo_evento') }}**
```
La stessa informazione è già visibile come prima riga `### Ultimo evento` nella card "Registro" immediatamente precedente (riga 2591). Si tratta di ridondanza pura che occupa spazio verticale senza aggiungere valore.

---

### 🔴 AC-UI-05 · 🟠 MEDIO — Potenziale duplicazione tra "Time line Eventi" e "Registro"
**File:** `allarme-core/plancia_controllo.yaml` · righe 2584–2619

La card "Time line Eventi" (riga 2584) mostra `sensor.allarme_core_timeline_render` via `state_attr`. La card "Registro" (riga 2589) mostra le stesse sezioni Critici/Operazioni/Note usando `var.allarme_core_timeline_critici_md`, `var.allarme_core_timeline_operazioni_md`, `var.allarme_core_timeline_note_md`. Se queste variabili derivano dallo stesso sensor, la sovrapposizione informativa è totale. Verificare se entrambe le card sono necessarie.

---

### 🔴 AC-UI-06 · 🔵 BASSO — `name: " "` su decine di tile sensori
**File:** `allarme-core/plancia_controllo.yaml` · diffuso

La quasi totalità dei tile sensori usa `name: " "` (spazio singolo) per nascondere il nome. Alcuni sensori hanno invece un nome esplicito e descrittivo (es. riga 909: `name: Armadio`, riga 1327: `name: Corridoio`, riga 1388: `name: Barriera Tap`). L'incoerenza riduce la leggibilità: l'utente non può identificare il sensore senza aprire il popup. Si consiglia di valorizzare `name:` esplicitamente con nomi brevi e descrittivi per tutti i tile.

---

### 🔴 AC-UI-07 · 🔵 BASSO — `state_content: [state, zones]` presuppone attributo `zones`
**File:** `allarme-core/plancia_controllo.yaml` · righe 14–17

La tile `sensor.allarme_core_zone_attive` usa:
```yaml
state_content:
  - state
  - zones
```
Questo presuppone che l'entità abbia un attributo chiamato esattamente `zones`. Se l'attributo non esiste o ha nome diverso, il secondo elemento viene mostrato vuoto senza errore. Verificare che `sensor.allarme_core_zone_attive` esponga effettivamente l'attributo `zones`.

---

## 3. Bug Critici — safety-core

---

### 🔴 SC-UI-01 · 🚨 CRITICO — `service_data.entity_id` deprecato su "Reset Manuale"
**File:** `safety-core/plancia_controllo.yaml` · righe 43–46

La card "Reset Manuale" usa la sintassi `service_data.entity_id` per specificare la destinazione di `script.turn_on`, deprecata da HA 2022.8. In versioni recenti di HA (2024+) questo genera warning nei log e in alcune configurazioni può causare comportamento non garantito.

```yaml
# ❌ ATTUALE (deprecato)
tap_action:
  action: call-service
  service: script.turn_on
  service_data:
    entity_id: script.safety_core_reset

# ✅ CORRETTO
tap_action:
  action: call-service
  service: script.turn_on
  target:
    entity_id: script.safety_core_reset
```

---

### 🔴 SC-UI-02 · 🚨 CRITICO — `service_data.entity_id` deprecato su "Cancella tutti i log"
**File:** `safety-core/plancia_controllo.yaml` · righe 944–946

Identico a SC-UI-01. La card "Cancella tutti i log" usa `service_data.entity_id: script.safety_core_logbook_cancella_tutto`. Fix analogo: spostare in `target.entity_id`.

---

### 🔴 SC-UI-03 · 🚨 CRITICO — Popup "Aggiungi Nota" non funzionale
**File:** `safety-core/plancia_controllo.yaml` · righe 956–963

La card "Aggiungi Nota" apre un popup con `content.type: entities` che espone solo `entity: script.safety_core_logbook_nota`. Il risultato è un popup che mostra unicamente un pulsante esegui-script, **senza alcun campo testuale** in cui l'utente possa scrivere il testo della nota. La funzionalità è de facto inutilizzabile: l'utente non può inserire testo, può solo avviare lo script che probabilmente scrive una nota vuota o predefinita.

**Fix consigliato:** Aggiungere `input_text.safety_core_nota_manuale` nel popup prima dello script, permettendo all'utente di compilare il testo prima di eseguire lo script.

```yaml
# ✅ FIX CONSIGLIATO
tap_action:
  action: call-service
  service: browser_mod.popup
  service_data:
    title: Aggiungi nota al log
    content:
      type: entities
      entities:
        - entity: input_text.safety_core_nota_manuale
          name: Testo della nota
        - entity: script.safety_core_logbook_nota
          name: Salva nota
```

---

## 4. Bug Medi/Bassi — safety-core

---

### 🔴 SC-UI-04 · 🟠 MEDIO — Logica CSS bordo ambigua: disabilitato = allarme
**File:** `safety-core/plancia_controllo.yaml` · diffuso (tutti i sensori con card_mod)

In tutti i sensori Gas, Fumo, Acqua e Carbonio, la logica del bordo CSS è:
```jinja2
{% set border_color = 'green' if (enabled == 'on' and cat == 'on') else 'red' %}
```
Il bordo è **rosso** sia quando un sensore è in **allarme attivo** sia quando è semplicemente **disabilitato intenzionalmente** dall'utente. L'utente non può distinguere visivamente tra le due condizioni — un pannello pieno di bordi rossi per sensori disabilitati sembra un sistema in emergenza.

**Fix consigliato:** Introdurre tre stati distinti:
```jinja2
{% if is_state(entity, 'on') %}
  {% set border_color = 'red' %}       {# allarme attivo #}
{% elif enabled == 'off' or cat == 'off' %}
  {% set border_color = 'grey' %}      {# disabilitato #}
{% else %}
  {% set border_color = 'green' %}     {# operativo e silenzioso #}
{% endif %}
```

---

### 🔴 SC-UI-05 · 🟠 MEDIO — Icona `far:image` — dipendenza da hacs-fontawesome
**File:** `safety-core/plancia_controllo.yaml` · riga 1032

Identico a AC-UI-03. Sostituire con `mdi:image-multiple`.

---

### 🔴 SC-UI-06 · 🟠 MEDIO — Card "Ultimo Evento" ridondante nella sezione Log
**File:** `safety-core/plancia_controllo.yaml` · righe 1018–1021

Identico a AC-UI-04. La card standalone "Safety Core – Ultimo Evento" duplica l'informazione già presente nel "Registro" soprastante.

---

### 🔴 SC-UI-07 · 🟠 MEDIO — "Stato Sistema" non gestisce stati oltre `triggered`/`ok`
**File:** `safety-core/plancia_controllo.yaml` · righe 22–28

La logica CSS dello "Stato Sistema" è:
```jinja2
{% set color = 'red' if stato == 'triggered' else 'green' %}
```
Non gestisce stati intermedi come `unknown`, `unavailable`, o stati custom dell'`input_select`. Se il sistema ha stati come `resetting` o `testing`, verranno mostrati in verde come se tutto fosse OK.

```yaml
# ✅ FIX CONSIGLIATO
{% set color = 'red' if stato == 'triggered' else
               'grey' if stato in ['unknown','unavailable'] else 'green' %}
```

---

### 🔴 SC-UI-08 · 🔵 BASSO — CSS sensori Acqua compresso su singola riga
**File:** `safety-core/plancia_controllo.yaml` · righe 396–402, 429–435, ecc.

I sensori Acqua hanno il CSS `card_mod` compresso su singola riga (probabile copia-incolla senza riformattazione), mentre i sensori Gas, Fumo e Carbonio usano CSS con indentazione e blocchi separati. Il comportamento è identico ma la manutenibilità del YAML è drasticamente ridotta per la sezione Acqua.

---

### 🔴 SC-UI-09 · 🔵 BASSO — Icona `mdi:view-dashboard` semanticamente errata
**File:** `safety-core/plancia_controllo.yaml` · riga 54

L'heading "Stato Categorie" usa `icon: mdi:view-dashboard`, icona generica senza correlazione semantica con le categorie di sicurezza. Suggerito: `mdi:shield-home` o `mdi:shape-outline`.

---

## 5. Incoerenze tra i due file

---

### INC-01 · 🚨 CRITICO — Azione "Cancella log" completamente diversa tra i due file

| File | Comportamento | Correttezza |
|------|--------------|-------------|
| `allarme-core` | `action: toggle` + `icon_tap_action: toggle` | ❌ Errato — toggling di script inaffidabile |
| `safety-core` | `action: call-service` + `service_data.entity_id` | 🟡 Intento corretto, sintassi deprecata |

Il pattern corretto univoco è: `action: call-service` + `target.entity_id`.

---

### INC-02 · 🟠 MEDIO — Heading principale con/senza icona

| File | Heading principale | Icona |
|------|-------------------|-------|
| `allarme-core` | "Controllo Allarme" | ❌ Assente |
| `safety-core` | "Controllo Safety" | ✅ `mdi:shield-alert` |

---

### INC-03 · 🟠 MEDIO — Logica CSS bordo sensori individuali

| File | Logica bordo | Gestione disabilitato |
|------|-------------|----------------------|
| `allarme-core` | Verde = abilitato, Rosso = disabilitato. Badge ❌ per zone non valide, ⚠️ per unavailable | Distingue correttamente "disabilitato" da "errore zona" |
| `safety-core` | Verde = `(enabled AND cat)`, Rosso = tutto il resto | Non distingue "disabilitato" da "allarme attivo" |

`allarme-core` ha una logica più robusta; `safety-core` dovrebbe allinearsi introducendo un terzo stato.

---

### INC-04 · 🟠 MEDIO — Feature "Aggiungi Nota" asimmetrica

| File | Presenza |
|------|---------|
| `allarme-core` | ❌ Assente nella sezione Log |
| `safety-core` | 🟡 Presente ma non funzionale (SC-UI-03) |

Una volta corretto SC-UI-03, la stessa feature andrebbe aggiunta anche ad `allarme-core`.

---

### INC-05 · 🟠 MEDIO — Card "Time line render" aggiuntiva solo in allarme-core

| File | Card aggiuntiva |
|------|----------------|
| `allarme-core` | ✅ "Time line Eventi" su `sensor.allarme_core_timeline_render` |
| `safety-core` | ❌ Assente — solo la timeline-card custom |

Valutare se aggiungere la stessa card testo anche in `safety-core` per coerenza.

---

### INC-06 · 🔵 BASSO — Formattazione CSS card_mod

| File | CSS sensori |
|------|------------|
| `allarme-core` | Uniforme — sempre indentato e leggibile |
| `safety-core` | Misto — Gas/Fumo/Carbonio formattati, Acqua compresso su riga singola |

---

## 6. Funzionalità Raccomandate

---

### PRIORITÀ ALTA

#### ✅ FR-01 — Terzo stato colore per sensori disabilitati (safety-core) — IMPLEMENTATO 2026-03-16
~~Attualmente il bordo rosso è ambiguo (disabilitato = allarme). Introdurre tre stati: **rosso** per allarme attivo, **grigio** per disabilitato intenzionalmente, **verde** per operativo e silenzioso.~~

**Implementato:** La logica `border_color = 'green' if (enabled == 'on' and cat == 'on') else 'red'` è stata rimossa da tutti i 19 sensori individuali di `safety-core/plancia_controllo.yaml` e sostituita con logica a tre stati:
- `grey` → sensore o categoria disabilitati intenzionalmente
- `red` → sensore in stato `on` (allarme attivo)
- `green` → operativo e silenzioso

I 9 sensori Acqua con CSS compresso su singola riga sono stati anche riformattati con indentazione standard.

#### ✅ FR-02 — Input testuale reale nel popup "Aggiungi Nota" (safety-core + allarme-core) — IMPLEMENTATO 2026-03-16
~~Il popup attuale è inutilizzabile. Aggiungere `input_text.safety_core_nota_manuale` nel popup prima dello script. Poi replicare la stessa funzionalità in allarme-core che ne è privo.~~

**Implementato su 4 file:**

*safety-core/safety_core_log.yaml:*
- Aggiunto `input_text.safety_core_nota_manuale` (campo helper UI)
- Aggiunto `script.safety_core_logbook_nota_da_ui` (wrapper che legge dall'helper, blocca se vuoto, delega a `logbook_emit`, azzera il campo dopo l'invio)

*safety-core/plancia_controllo.yaml:*
- Popup "Aggiungi Nota" aggiornato: mostra `input_text.safety_core_nota_manuale` + `script.safety_core_logbook_nota_da_ui`

*allarme-core/allarme_core_log.yaml:*
- Aggiunto `input_text.allarme_core_nota_manuale`
- Aggiunto `script.allarme_core_logbook_nota_da_ui` (stesso pattern)

*allarme-core/plancia_controllo.yaml:*
- Tile "Cancella tutti i log" ridotta da 12 a 6 colonne
- Aggiunta nuova tile "Aggiungi Nota" affiancata, con popup funzionale

#### FR-03 — Sostituzione `far:image` con `mdi:image-multiple` (entrambi)
Eliminare la dipendenza da hacs-fontawesome su entrambi i file.

---

### PRIORITÀ MEDIA

#### FR-04 — Stato sistema header sempre visibile con chip (entrambi)
Sostituire l'header testuale con `custom:mushroom-chips-card` che mostri lo stato attuale (ARMATO/DISARMATO/TRIGGERED) sempre visibile senza scroll. Richiede HACS: `mushroom`.

#### FR-05 — `hold_action` per abilitazione rapida sensori (entrambi)
Aggiungere `hold_action: action: toggle` sull'`input_boolean._abilitato` corrispondente a ogni tile sensore. Riduce a 1 click la disabilitazione temporanea di un sensore senza aprire il popup.

#### FR-06 — Contatore sensori attivi per stanza/categoria (allarme-core)
Aggiungere sotto ogni heading di stanza (Sala, Camera, Ufficio, ecc.) una `mushroom-template-card` o badge che mostri "X/Y sensori abilitati in questa stanza". Permette di identificare anomalie per area senza scorrere tutti i tile.

#### FR-07 — Animazione CSS per stato triggered (entrambi)
Aggiungere un'animazione `@keyframes pulse` sulla card "Stato Sistema" e "Qualsiasi Allerta" quando lo stato è `triggered`, per rendere l'allerta visivamente evidente anche perifericamente senza guardare il colore.

```yaml
# Esempio CSS card_mod per stato triggered
card_mod:
  style: >
    {% if is_state('input_select.safety_core_stato','triggered') %}
    ha-card {
      animation: pulse 1s infinite;
    }
    @keyframes pulse {
      0% { box-shadow: 0 0 0 3px red; }
      50% { box-shadow: 0 0 0 8px rgba(255,0,0,0.3); }
      100% { box-shadow: 0 0 0 3px red; }
    }
    {% endif %}
```

#### FR-08 — Feature `select-options` su "Stato Sistema" safety-core
Aggiungere la feature `select-options` al tile `input_select.safety_core_stato`, analogamente a quanto già fatto per `input_select.allarme_core_profilo` in allarme-core (riga 23). Permette di cambiare lo stato direttamente dalla card.

---

### PRIORITÀ BASSA

#### FR-09 — Nome esplicito su tutti i tile sensori (allarme-core)
Valorizzare `name:` con etichette brevi e descrittive (es. "Finestra", "Porta", "Movimento") per tutti i tile che usano `name: " "`. Migliora la leggibilità senza aprire popup.

#### FR-10 — Auto-entities per generazione dinamica sensori (entrambi)
Usare `custom:auto-entities` (HACS) con filtro su area o attributo per generare automaticamente i tile. La dashboard si auto-aggiorna al'aggiunta di nuovi sensori senza modifiche manuali — particolarmente utile in safety-core dove i sensori sono categorizzati per tipo.

#### FR-11 — Indicatore timestamp ultimo test sensore (safety-core)
Aggiungere nel popup browser_mod di ogni sensore un campo `input_datetime.safety_core_<sensore>_ultimo_test` che permetta di tracciare quando un sensore è stato verificato l'ultima volta.

---

## 7. Bug Scoperti durante Implementazione

---

### ✅ DB-01 · 🚨 CRITICO (pre-esistente) — Indentazione errata `allarme_core_snapshot_cameras`
**File:** `allarme-core/allarme_core_log.yaml` · righe 97–116
**Stato:** CORRETTO il 2026-03-16

Il corpo dello script `allarme_core_snapshot_cameras` era indentato a 6 spazi invece dei corretti 4 (chiavi `alias`, `mode`, `max`, `fields`, `sequence`). Home Assistant non caricava correttamente lo script o lo interpretava in modo imprevedibile. Bug pre-esistente, non correlato alle implementazioni FR-01/FR-02, scoperto dalla validazione automatica.

```yaml
# ❌ PRIMA (6 spazi — errato)
  allarme_core_snapshot_cameras:
      alias: "Allarme Core - Snapshot Cameras"
      mode: queued

# ✅ DOPO (4 spazi — corretto)
  allarme_core_snapshot_cameras:
    alias: "Allarme Core - Snapshot Cameras"
    mode: queued
```

---

## 8. Piano di Intervento

| Priorità | ID | File | Descrizione | Stato |
|----------|----|------|-------------|-------|
| 1 | AC-UI-01 | allarme-core | `toggle` → `call-service` su Cancella Log | 🔴 Aperto |
| 2 | SC-UI-01/02 | safety-core | `service_data` → `target` su Reset + Cancella Log | 🔴 Aperto |
| 3 | SC-UI-03 + FR-02 | entrambi | Popup Aggiungi Nota funzionale con input_text | ✅ Implementato 2026-03-16 |
| 4 | FR-01 | safety-core | Logica CSS bordo 3 stati (verde/grigio/rosso) | ✅ Implementato 2026-03-16 |
| 5 | DB-01 | allarme-core | Fix indentazione `allarme_core_snapshot_cameras` | ✅ Corretto 2026-03-16 |
| 6 | AC-UI-02 | allarme-core | Icona mancante heading "Controllo Allarme" | 🔴 Aperto |
| 7 | AC-UI-03 + SC-UI-05 | entrambi | `far:image` → `mdi:image-multiple` | 🔴 Aperto |
| 8 | AC-UI-04 + SC-UI-06 | entrambi | Rimuovere card "Ultimo Evento" ridondante | 🔴 Aperto |
| 9 | SC-UI-07 | safety-core | Gestire `unknown/unavailable` in CSS Stato Sistema | 🔴 Aperto |
| 10 | FR-04 | entrambi | Chip stato sempre visibile in header | 🔴 Aperto |
| 11 | FR-05 | entrambi | `hold_action` per abilitazione rapida | 🔴 Aperto |

---

*Report generato automaticamente dall'analisi agenti: lovelace-dashboard-designer + homeassistant-debugger*
*Progetto: home-assistant | Branch: main | Data: 2026-03-16*
