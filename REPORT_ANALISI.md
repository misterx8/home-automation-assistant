# Report di Analisi — allarme-core & safety-core

**Data analisi:** 2026-03-16
**Ultima modifica:** 2026-03-16
**Agenti utilizzati:** homeassistant-debugger (x2), homeassistant-automation-engineer
**File analizzati:** 11 file YAML tra i due progetti

> **Legenda stato:** ✅ Corretto · 🔴 Aperto · 🟡 Parzialmente corretto

---

## Indice

1. [Bug Critici — allarme-core](#1-bug-critici--allarme-core)
2. [Bug Medi/Bassi — allarme-core](#2-bug-medibassi--allarme-core)
3. [Bug Critici — safety-core](#3-bug-critici--safety-core)
4. [Bug Medi/Bassi — safety-core](#4-bug-medibassi--safety-core)
5. [Funzionalità Raccomandate](#5-funzionalità-raccomandate)
6. [Piano di Intervento](#6-piano-di-intervento)

---

## 1. Bug Critici — allarme-core

---

### ✅ AC-BUG-01 · CRITICO — `disarm_allarme_core`: `entity_id` dentro `data` invece di `target`
**File:** `allarme_core.yaml` · `allarme_core_automazioni.yaml`
**Stato:** CORRETTO il 2026-03-16

Il disarmo dell'allarme usava una sintassi deprecata dal 2022.x che in versioni recenti di HA causa un **failure silenzioso**: lo script viene eseguito senza errori ma lo stato non cambia.

**Fix applicato** in 3 punti: script `disarm_allarme_core`, automazione reset sensori esclusi al disarm, automazione memorizza sensori esclusi. `entity_id` spostato da `data:` a `target:` in tutti e tre.

```yaml
# CORRETTO (applicato)
- service: input_select.select_option
  target:
    entity_id: input_select.allarme_core_stato
  data:
    option: disarmed
```

---

### AC-BUG-02 · CRITICO — Trigger su `armed` nell'automazione sensori esclusi è logicamente errato
**File:** `allarme_core_automazioni.yaml` righe 19–26

Le automazioni `allarme_core_aut_memorizza_sensori_esclusi` e `allarme_core_aut_memorizza_sensori_esclusi_var` si triggherano su **entrambi** `arming` e `armed`. Il trigger su `armed` è sbagliato: a quel punto l'armamento è già completato e un sensore che si apre subito dopo viene erroneamente escluso, causando **falsi negativi** (sensori reali aperti da intruso vengono ignorati).

**Fix:** Rimuovere il trigger `to: "armed"` da entrambe le automazioni — il trigger corretto è solo `to: "arming"`.

---

### ✅ AC-BUG-03 · CRITICO — `map('state_attr', 'friendly_name')` su lista di stringhe non funziona
**File:** `allarme_core_automazioni.yaml`
**Stato:** CORRETTO il 2026-03-16

`map('state_attr', 'friendly_name')` non è un filtro Jinja2 valido su una lista di stringhe. La variabile `aperti_nomi` risultava sempre vuota.

**Fix applicato:** sostituito con loop esplicito `for eid in ids` con `state_attr(eid, 'friendly_name') | default(eid)`.

```yaml
# CORRETTO (applicato)
aperti_nomi: >
  {% set ids = state_attr('sensor.allarme_core_sensori_aperti_filtrati', 'sensori_aperti_ids') | default([]) %}
  {% set ns = namespace(nomi=[]) %}
  {% for eid in ids %}
    {% set ns.nomi = ns.nomi + [state_attr(eid, 'friendly_name') | default(eid)] %}
  {% endfor %}
  {{ ns.nomi }}
```

---

### ✅ AC-BUG-04 · CRITICO — Indentazione errata nei sensori template di `allarme_core_log.yaml`
**File:** `allarme_core_log.yaml`
**Stato:** CORRETTO il 2026-03-16

I sensori template (`Timeline Render`, `Count Critici`, `Count Operazioni`, `Count Note`) erano indentati a 4 spazi invece di 6 — HA non li caricava, la UI della timeline era completamente non funzionante.

**Fix applicato:** corretta l'indentazione a 6 spazi. Aggiunti contestualmente i `unique_id` mancanti a tutti e 4 i sensori (`allarme_core_timeline_render`, `allarme_core_count_critici`, `allarme_core_count_operazioni`, `allarme_core_count_note`).

---

### ✅ AC-BUG-05 · CRITICO — `input_datetime.allarme_core_arming_started` non definito in nessun file
**File:** `allarme_core.yaml` · `allarme_core_automazioni.yaml`
**Stato:** CORRETTO il 2026-03-16

Il sensore `Allarme Core Sensori Aperti Filtrati` usava questa entity nell'attributo `availability` ma non era definita da nessuna parte. Durante lo stato `arming`, il sensore risultava sempre `unavailable`.

**Fix applicato in 2 parti:**
1. Aggiunto blocco `input_datetime` in `allarme_core.yaml`:
```yaml
input_datetime:
  allarme_core_arming_started:
    name: "Allarme Core - Timestamp Inizio Arming"
    has_date: true
    has_time: true
```
2. Aggiunta automazione `allarme_core_aut_registra_arming_started` in `allarme_core_automazioni.yaml` che imposta il datetime al momento esatto in cui lo stato passa ad `arming`.

---

### AC-BUG-06 · CRITICO — `service:` deprecato in tutto il progetto (HA 2024.8+)
**File:** `allarme_core.yaml`, `allarme_core_automazioni.yaml`, `allarme_core_log.yaml`

Tutte le action usano la chiave `service:` invece di `action:`. In HA 2024.8+ questa sintassi è deprecata e genera warning al boot. In versioni future cesserà di funzionare.

**Fix:** Sostituire sistematicamente `service:` con `action:` in tutti i blocchi `sequence:` e `action:`.

---

## 2. Bug Medi/Bassi — allarme-core

---

### AC-BUG-07 · MEDIO — Accesso fragile a `states.var.xxx.last_changed` nel boot
**File:** `allarme_core.yaml` riga 217

```yaml
{% set var_ts = as_timestamp(states.var.allarme_core_sensori_esclusi_var.last_changed) %}
```

Se l'integrazione `var` non ha ancora caricato le entità al boot, questo accesso genera `UndefinedError` e manda il sensore in `unavailable` permanente. Aggiungere un controllo `is defined` o usare `default(now())`.

---

### AC-BUG-08 · MEDIO — Automazione `UI Timeline Append` senza `id` e `mode`
**File:** `allarme_core_log.yaml` riga 323

Questa automazione non ha `id:` né `mode:`. Senza `id` non è gestibile via UI HA. Senza `mode`, usa il default `single` e può perdere eventi di log ravvicinati. Aggiungere `id: allarme_core_ui_timeline_append` e `mode: queued` con `max: 50`.

---

### AC-BUG-09 · MEDIO — Variabile `now` sovrascrive la funzione built-in Jinja2
**File:** `allarme_core_log.yaml` riga 328

```yaml
variables:
  now: "{{ now().strftime('%H:%M:%S') }}"  # ← sovrascrive now()
```

In alcune versioni di HA questo crea un conflitto di scope. Rinominare in `ts_ora` o `ora_corrente`.

---

### AC-BUG-10 · MEDIO — Tutti i 35+ binary_sensor in `allarme_core_supporto.yaml` senza `unique_id`
**File:** `allarme_core_supporto.yaml` (tutto il file)

Nessun sensore `_zone_valida` ha `unique_id`. Impossibile gestirli dalla UI, rinominarli o tracciarli nel registro delle entità. In caso di modifica del `name:`, HA crea duplicati nel database.

**Fix:** Aggiungere `unique_id:` a ogni sensore usando lo stesso valore del `name:` in snake_case.

---

### AC-BUG-11 · MEDIO — Doppia memorizzazione sensori esclusi: rischio inconsistenza
**File:** `allarme_core_automazioni.yaml` righe 15–69

Due automazioni (`input_text` + `var`) fanno la stessa cosa con gli stessi trigger. Se una fallisce, i due storage sono inconsistenti. Consolidare in un'unica sorgente di verità (preferibilmente `var`).

---

### AC-BUG-12 · BASSO — `var` senza valore iniziale
**File:** `var/allarme_core.yaml` righe 1–6

Le var `sensori_esclusi_var` e `sensori_attivazione_allarme_var` non hanno `value:` iniziale. Al primo boot restituiscono `unknown`. Aggiungere `value: "[]"` come valore iniziale.

---

## 3. Bug Critici — safety-core

---

### SC-BUG-01 · CRITICO — Falsi allarmi acqua su `unknown`/`unavailable` (8 sensori su 9)
**File:** `safety_core_sensori.yaml` — pattern ripetuto su Bagno, Cucina Principale, Doccia, Lavanderia, Lavello Cucina, Studio, Stufa, Tecnologico PT

I sensori acqua usano una logica **diversa** e **sbagliata** rispetto a fumo/gas/carbonio (e anche rispetto al corretto `safety_core_acqua_vasca`):

```yaml
# ERRATO (8 sensori acqua)
{% if not abilitato %}
  off
{% else %}
  {{ 'on' if fisico != 'off' else 'off' }}  # ← unknown/unavailable != 'off' → FALSO ON
{% endif %}

# CORRETTO (già usato da fumo/gas/carbonio/acqua_vasca)
{% if fisico in ['on','off'] %}
  {{ 'on' if (abilitato and fisico == 'on') else 'off' }}
{% else %}
  off  # ← unknown/unavailable → sempre off
{% endif %}
```

**Impatto:** Ad ogni riavvio di HA o disconnessione WiFi di un sensore acqua, il sensore wrapper diventa `on`, propagando un falso triggered a tutta la catena. La condizione di guardia nell'automazione mitiga parzialmente il danno ma il sistema rimane in stato inconsistente con dashboard errate.

---

### SC-BUG-02 · CRITICO — Pattern regex `input_text` camera non accetta stringa vuota
**File:** `safety_core_sensori.yaml` righe 110–171 (tutti i 19 `input_text` camera)

```yaml
pattern: "^[^,]+(,[^,]+)*$"  # ← non accetta "" (valore default di input_text)
```

Al primo avvio, tutti i 19 campi camera sono in stato di validazione fallita perché il valore default (`""`) non matcha il pattern. Questo può causare warning UI e rendere i campi non editabili normalmente.

**Fix:**
```yaml
pattern: "^$|^[^,]+(,[^,]+)*$"
```

---

### SC-BUG-03 · CRITICO — IP Frigate hardcoded in due file separati
**File:** `safety_core_automazioni.yaml` riga 209 + `allarme_core_log.yaml` riga 267

```yaml
frigate_base: "http://192.168.2.92:5000"  # duplicato nei due progetti
```

Se l'IP cambia, va aggiornato in due posti con rischio di dimenticare uno. Le URL dei clip saranno errate silenziosamente. Esternalizzare in una `var` o `input_text` condivisa.

---

## 4. Bug Medi/Bassi — safety-core

---

### SC-BUG-04 · MEDIO — `mode: single` sull'automazione triggered perde eventi ravvicinati
**File:** `safety_core_automazioni.yaml` riga 19

Con `mode: single`, se due categorie scattano quasi-simultaneamente (es. fumo + gas), solo la prima viene processata. La seconda categoria non viene registrata nella `var` né fotografata. **Fix:** Usare `mode: queued` con `max: 3`.

---

### SC-BUG-05 · MEDIO — Sensori template in `safety_core_log.yaml` senza `unique_id`
**File:** `safety_core_log.yaml` righe 126, 149–162

I sensori `Timeline Render`, `Count Critici`, `Count Operazioni`, `Count Note` non hanno `unique_id`. Impossibile gestirli dalla UI di HA, assegnarli ad aree o disabilitarli individualmente.

---

### SC-BUG-06 · MEDIO — Var commentate nel file principale senza istruzioni di include
**File:** `safety_core.yaml` righe 89–118

Le variabili `var` sono commentate nel file principale ma essenziali per le automazioni. Non è documentato come includere `var/safety_core.yaml`. Se le var non sono incluse, tutto il sistema di log e notifiche fallisce silenziosamente.

**Aggiungere** nel README o nel file stesso:
```yaml
# Nel tuo configuration.yaml aggiungere:
# var: !include_dir_merge_named safety-core/var/
```

---

### SC-BUG-07 · MEDIO — Attributo `camera` restituisce `['']` invece di `[]` su input vuoto
**File:** `safety_core_sensori.yaml` riga 202 (e tutti gli equivalenti)

```yaml
camera: "{{ states('input_text.safety_core_fumo_studio_camera').split(',') }}"
```

Con input_text vuoto, `"".split(',')` → `['']` (lista con elemento vuoto). Questo elemento vuoto può essere passato al servizio `camera.snapshot` causando errori.

**Fix:**
```yaml
camera: >
  {% set raw = states('input_text.safety_core_fumo_studio_camera') | trim %}
  {{ raw.split(',') | map('trim') | reject('equalto','') | list if raw else [] }}
```

---

### SC-BUG-08 · BASSO — `service:` deprecato in tutto il progetto (HA 2024.8+)
**File:** `safety_core_automazioni.yaml`, `safety_core_log.yaml`, `safety_core.yaml`

Stessa situazione di allarme-core: sostituire `service:` con `action:` ovunque.

---

### SC-BUG-09 · BASSO — `input_select.safety_core_ultima_categoria` senza `initial` e `icon`
**File:** `safety_core.yaml` righe 26–33

Al primo avvio il valore non è predeterminato. Aggiungere `initial: nessuna` e `icon: mdi:shield-alert-outline`.

---

### SC-BUG-10 · BASSO — `var.safety_core_sensori_attivazione_var` senza `default()` nella lettura
**File:** `safety_core_log.yaml` riga 113

```yaml
{% set raw = states('var.safety_core_sensori_attivazione_var') %}
```

Al primo avvio la var è `unknown`. La chiamata `.split(', ')` su `unknown` genera errori di template. Aggiungere `| default('', true)`.

---

## 5. Funzionalità Raccomandate

---

### FEAT-01 · ALTA — Notifiche Push con Azioni Interattive
**Sistema:** Entrambi

Nessuno dei due sistemi invia notifiche push. In assenza dall'abitazione un evento triggered non viene comunicato.

**Logica:**
- Automazione su `input_select.*_stato` → `triggered`
- `notify.mobile_app_*` con `data.actions`: `[{action: "DISARM_ALARM", title: "Disarma"}]` per allarme-core, `[{action: "RESET_SAFETY", title: "Reset"}]` per safety-core
- Seconda automazione su `mobile_app_notification_action` per gestire le azioni
- Per fumo/gas/carbonio: usare `channel: alarm_stream` (Android) per bypass modalità silenziosa
- Allegare la prima snapshot disponibile (`var.*_last_snapshot_urls`)

---

### FEAT-02 · ALTA — Monitoraggio Disponibilità Sensori
**Sistema:** Entrambi

Un sensore fisico che va `unavailable` è protetto silenziosamente senza che nessuno se ne accorga.

**Logica:**
- Template `binary_sensor` aggregato che itera sui sensori `*_core_*` e verifica lo stato del loro `sensore_origine`
- `on` se almeno un sensore fisico è `unavailable`/`unknown`
- Log immediato nella timeline + notifica push dedicata
- Per allarme-core: opzionalmente bloccare l'arming se sensori critici sono offline (`input_number.allarme_core_soglia_sensori_offline`)

---

### FEAT-03 · ALTA — Codice PIN per il Disarmo
**Sistema:** allarme-core

Il disarmo avviene attualmente senza alcuna verifica. Aggiungere al blocco `alarm_control_panel`:

```yaml
code: !secret allarme_core_pin
code_format: number
```

Usare `secrets.yaml` per il PIN. Per il disarmo via notifica push (FEAT-01) il PIN non è necessario (autenticazione tramite device mobile).

---

### FEAT-04 · ALTA — Sirena su Triggered
**Sistema:** allarme-core

Il triggered è attualmente silenzioso. Aggiungere:
- Script `allarme_core_attiva_sirena` con timeout configurabile (`input_number.allarme_core_durata_sirena_minuti`)
- Delay configurabile prima dell'attivazione (es. 30s per permettere il disarmo)
- Automazione che ferma la sirena su `disarmed`
- File separato `allarme_core_sirena.yaml` per mantenere la modularità

---

### FEAT-05 · ALTA — Esternalizzazione IP Frigate e Parametri Temporali
**Sistema:** Entrambi

Creare un file `shared_config.yaml` o un package con:

```yaml
input_text:
  shared_frigate_base_url:
    name: "Frigate Base URL"
    initial: "http://192.168.2.92:5000"

input_number:
  allarme_core_delay_arming:
    name: "Allarme - Delay Arming (secondi)"
    initial: 20
    min: 5
    max: 120
    step: 5
  safety_core_delay_reset_grazia:
    name: "Safety - Delay Reset Grazia (secondi)"
    initial: 10
    min: 5
    max: 60
    step: 5
```

Sostituire tutti i valori hardcoded con `states('input_text.shared_frigate_base_url')` e i delay con template dinamici.

---

### FEAT-06 · MEDIA — Modalità Manutenzione (Bypass Temporaneo Sensore)
**Sistema:** Entrambi

Il sistema permette solo disable permanente. Manca il bypass temporaneo (es. "escludi questo sensore per 2 ore durante lavori").

**Logica:**
- `input_datetime.*_manutenzione_fino_a` per ogni sensore (o globale)
- Template `binary_sensor` `*_in_manutenzione` che verifica `now() < as_datetime(...)`
- Script `imposta_manutenzione` con campo `entity_id` e `durata_ore`
- Log nella timeline con emoji 🔧

---

### FEAT-07 · MEDIA — Arm Automatico Basato su Presenza
**Sistema:** allarme-core

**Logica:**
- Trigger: conteggio persone `home` scende a 0
- Condizione: stato è `disarmed`
- Action: chiama `script.arm_allarme_core` con profilo da `input_select.allarme_core_profilo_uscita`
- `input_boolean.allarme_core_arm_auto_presenza` per abilitare/disabilitare
- Automazione inversa: disarmo al rientro della prima persona (solo se non `triggered`)

---

### FEAT-08 · MEDIA — Storico Eventi Persistente
**Sistema:** Entrambi

Le var HACS si perdono ad ogni riavvio. Aggiungere:
- `input_text` per ultimo evento critico + timestamp (sopravvive ai riavvii)
- `input_number` contatore triggered totali
- Opzionale: `shell_command` per scrivere log su file CSV in `/config/www/`

---

### FEAT-09 · MEDIA — Tasto Panico
**Sistema:** allarme-core

- Script `allarme_core_panic` che forza `triggered` indipendentemente dallo stato corrente
- Log dedicato "🆘 PANICO MANUALE"
- Attivazione sirena immediata (senza delay)
- Protezione da attivazione accidentale: doppia conferma in Lovelace (`confirmation: true`)

---

### FEAT-10 · MEDIA — Bridge di Integrazione tra i Due Sistemi
**Sistema:** Entrambi (file `bridge_allarme_safety.yaml`)

Scenari da gestire:
- **Safety triggered + Allarme disarmato** → notifica massima priorità
- **Safety fumo/gas/CO + Allarme armed** → disarmo automatico di emergenza (opt-in via `input_boolean.bridge_disarmo_emergenza_safety`) per permettere evacuazione sicura
- **Safety acqua + Allarme armed** → solo notifica, non disarmare
- **Arming in corso + Safety triggered** → bloccare l'arming

---

### FEAT-11 · MEDIA — Stato `acknowledged` in Safety-Core
**Sistema:** safety-core

Aggiungere un terzo stato all'`input_select.safety_core_stato`:
- `ok` → `triggered` → `acknowledged` → `ok`

Lo stato `acknowledged` significa "evento visto, situazione gestita". Differisce dal reset completo perché:
- I sensori fisici possono ancora essere `on`
- Non accetta nuovi trigger per la stessa categoria
- Logga chi ha fatto l'acknowledge e quando

---

### FEAT-12 · BASSA — Test Periodico Automatico
**Sistema:** Entrambi

Automazione settimanale che:
1. Verifica tutti i sensori fisici (no `unavailable`/`unknown`)
2. Verifica accessibilità var HACS
3. Verifica accessibilità Frigate (GET su `/api/stats`)
4. Logga "🔬 Test settimanale: X/Y sensori OK"
5. Invia notifica riassuntiva

---

## 6. Piano di Intervento

### Fase 1 — Correzioni Bloccanti (da fare subito)

| ID | Descrizione | File | Stato |
|----|-------------|------|-------|
| AC-BUG-01 | Correggere `disarm_allarme_core` con `target:` | `allarme_core.yaml` + `allarme_core_automazioni.yaml` | ✅ Corretto 2026-03-16 |
| AC-BUG-02 | Rimuovere trigger `to: armed` dalla memorizzazione esclusi | `allarme_core_automazioni.yaml` | 🔴 Aperto |
| AC-BUG-04 | Correggere indentazione sensori timeline + aggiungere `unique_id` | `allarme_core_log.yaml` | ✅ Corretto 2026-03-16 |
| AC-BUG-05 | Creare `input_datetime.allarme_core_arming_started` + automazione | `allarme_core.yaml` + `allarme_core_automazioni.yaml` | ✅ Corretto 2026-03-16 |
| SC-BUG-01 | Correggere logica acqua su 8 sensori (falsi allarmi) | `safety_core_sensori.yaml` | 🔴 Aperto |
| SC-BUG-02 | Correggere pattern regex camera su 19 `input_text` | `safety_core_sensori.yaml` | 🔴 Aperto |

### Fase 2 — Correzioni Importanti (entro breve)

| ID | Descrizione | Stato |
|----|-------------|-------|
| AC-BUG-03 | Fix `map('state_attr')` errato | ✅ Corretto 2026-03-16 |
| AC-BUG-06 / SC-BUG-08 | Migrazione da `service:` ad `action:` (tutto il codice) | 🔴 Aperto |
| AC-BUG-08 | Aggiungere `id` e `mode: queued` a `UI Timeline Append` | 🔴 Aperto |
| AC-BUG-10 | Aggiungere `unique_id` ai 35+ sensori in `allarme_core_supporto.yaml` | 🔴 Aperto |
| SC-BUG-03 | Esternalizzare IP Frigate (FEAT-05) | 🔴 Aperto |
| SC-BUG-06 | Documentare l'include delle var | 🔴 Aperto |
| SC-BUG-07 | Fix attributo `camera` con split su stringa vuota | 🔴 Aperto |

### Fase 3 — Nuove Funzionalità (pianificazione)

| Priorità | Funzionalità |
|----------|-------------|
| ALTA | FEAT-01: Notifiche push con azioni interattive |
| ALTA | FEAT-02: Monitoraggio disponibilità sensori |
| ALTA | FEAT-03: PIN disarmo allarme |
| ALTA | FEAT-04: Sirena su triggered |
| MEDIA | FEAT-10: Bridge integrazione tra sistemi |
| MEDIA | FEAT-11: Stato `acknowledged` in safety-core |
| MEDIA | FEAT-06: Modalità manutenzione bypass |
| MEDIA | FEAT-07: Arm automatico su presenza |
| MEDIA | FEAT-08: Storico eventi persistente |
| MEDIA | FEAT-09: Tasto panico |
| BASSA | FEAT-12: Test periodico automatico |

---

## Riepilogo Numerico

| Categoria | Critico | Medio | Basso | Totale | Corretti |
|-----------|---------|-------|-------|--------|----------|
| allarme-core | 6 | 5 | 1 | **12** | ✅ 4 (AC-BUG-01, 03, 04, 05) |
| safety-core | 3 | 5 | 3 | **11** | — |
| **Totale bug** | **9** | **10** | **4** | **23** | **4 corretti / 19 aperti** |
| Nuove funzionalità raccomandate | — | — | — | **12** | — |

---

*Report generato con homeassistant-debugger e homeassistant-automation-engineer*
