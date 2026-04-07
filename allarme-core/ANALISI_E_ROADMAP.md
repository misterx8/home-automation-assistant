# Allarme Core — Analisi Approfondita e Roadmap Implementazioni

> Generato il 2026-04-06

---

## 1. STATO ATTUALE DEL SISTEMA

Il sistema **Allarme Core** è solido, ben strutturato e modulare. Copre già la maggior parte dei casi d'uso reali:

- Arming/disarming con delay configurabile e countdown visuale
- 5 profili zona (sera, giorno, tutti, notte, nessuno)
- ~50 sensori wrapper con attributi zona, camera, enable/disable
- Esclusione automatica sensori aperti al momento dell'arming
- Integrazione Frigate per snapshot e clip su triggered
- Monitoraggio batterie wireless con soglia configurabile e debounce offline
- Rilevamento anomalie sensori fisici (unknown/unavailable)
- Logging e timeline markdown con note manuali
- Modalità test con auto-scadenza
- Header dashboard con alert contestuali

---

## 2. BUG CORRETTI (storico)

| ID | Descrizione | File | Stato |
|----|-------------|------|-------|
| AC-BUG-01 | entity_id in `data:` deprecato, failure silenzioso | allarme_core.yaml, automazioni | ✅ Fixato |
| AC-BUG-05 | input_datetime.allarme_core_arming_started non dichiarato → sensore aperti filtrati unavailable | allarme_core.yaml | ✅ Fixato |
| AC-BUG-06 | script arm continuava in background dopo disarm → si armava da solo dopo il delay | allarme_core.yaml | ✅ Fixato |
| FIX-B | Sensori aperti filtrati ignorava stato "triggered" | allarme_core.yaml | ✅ Applicato |
| FIX-C | Race condition lettura sensori causa allarme (usava template invece di lettura diretta) | allarme_core_automazioni.yaml | ✅ Applicato |

---

## 3. NUOVE IMPLEMENTAZIONI CONSIGLIATE

Le implementazioni sono ordinate per priorità e impatto reale.

---

### 🔴 PRIORITÀ ALTA

---

#### 3.1 Bottone PANICO / SOS

**Problema:** Non esiste un modo rapido per attivare l'allarme manualmente in caso di emergenza, indipendentemente dal profilo e dallo stato corrente.

**Implementazione:**
- Aggiungere `input_button.allarme_core_panico` nella plancia operativa
- Script dedicato `script.allarme_core_attiva_panico` che:
  1. Forza stato → `triggered`
  2. Logga "🚨 PANICO manuale attivato da UI"
  3. Scrive in var causa allarme "Panico manuale"
  4. Node-RED rileva e attiva sirena + notifiche senza filtri

**Plancia operativa:** Card bottone rosso prominente nella prima sezione accanto al pannello di controllo, visibile sempre, con conferma via hold action per evitare falsi trigger.

```yaml
# Idea card:
type: button
name: "⚠️ PANICO"
icon: mdi:alarm-light
hold_action:
  action: call-service
  service: script.turn_on
  target:
    entity_id: script.allarme_core_attiva_panico
card_mod:
  style: |
    ha-card {
      background: red;
      color: white;
      border-radius: 30px;
    }
```

---

#### 3.2 Storico Allarmi Persistente

**Problema:** `sensor.allarme_core_ultimo_trigger` salva solo l'ultimo evento. Non c'è memoria storica degli allarmi passati.

**Implementazione:**
- Aggiungere `var.allarme_core_storico_allarmi` con attributo `lista` (array)
- Automazione che all'ogni `triggered` fa append del nuovo evento alla lista (max 20 elementi)
- Ogni elemento storico contiene: timestamp, profilo, sensori, urls_snapshot, urls_clip

**Plancia operativa:** Nuova sezione "Storico Allarmi" con markdown card che mostra la lista in formato tabella:
```
| Data | Profilo | Sensori | Media |
|------|---------|---------|-------|
| 05/04 18:23 | sera | Porta sala | 📷 2 📹 1 |
```

---

#### 3.3 Auto-Reset da Triggered Configurabile

**Problema:** Una volta scattato l'allarme, rimane in `triggered` indefinitamente. L'utente deve sempre intervenire manualmente.

**Implementazione:**
- Aggiungere `input_boolean.allarme_core_auto_reset_triggered` (default off)
- Aggiungere `input_number.allarme_core_auto_reset_minuti` (1-30 min, default 5)
- Automazione: se triggered da più di N minuti E auto_reset abilitato → disarma
- Logga l'evento come "reset automatico dopo timeout"

**Plancia operativa:** Nella sezione Modalità Test o in una nuova sezione "Configurazione Avanzata".

---

#### 3.4 Timeout Sicurezza su Stato "Arming"

**Problema:** Se l'arming viene interrotto o lo script rimane bloccato, il sistema potrebbe restare in `arming` indefinitamente senza mai armarsi né disarmarsi.

**Implementazione:**
- Automazione con trigger template: `stato == 'arming'` da più di `arming_delay + 30 secondi`
- Azione: disarma e logga "⚠️ Arming timeout — stato resettato a disarmed"
- Questo è una rete di sicurezza, non il comportamento normale

---

### 🟡 PRIORITÀ MEDIA

---

#### 3.5 Sensore Test Connettività Frigate

**Problema:** Se URL o token Frigate sono errati, lo snapshot fallisce silenziosamente. Non c'è feedback all'utente.

**Implementazione:**
- `sensor.allarme_core_frigate_status` — template sensor che testa la raggiungibilità
- Alternativa più semplice: `binary_sensor` che verifica se i 3 campi (url_ha, url_frigate, token) sono compilati
- Alert nell'header della plancia se Frigate non configurato

**Plancia operativa:** Il bottone "Impostazioni Variabili" già mostra il border colorato (verde/arancio/rosso) in base allo stato dei 3 campi. È sufficiente migliorare il tooltip/badge per indicare esattamente quale campo manca.

---

#### 3.6 Disarmo con Codice PIN (opzionale)

**Problema:** Chiunque abbia accesso alla dashboard può disarmare l'allarme senza autenticazione aggiuntiva.

**Implementazione via Node-RED:**
- Input numerico per PIN in Node-RED flow
- Se PIN corretto → chiama `script.disarm_allarme_core`
- Se PIN errato → logga tentativo e notifica

**Alternativa HA nativa:**
- `alarm_control_panel` già supporta `code:` per il disarmo
- Rimuovere `code_arm_required: false` e aggiungere `code:` con PIN
- Il pannello tile mostrerà tastierino PIN automaticamente

---

#### 3.7 Profili Automatici via Presenza (Geofencing)

**Problema:** I profili (sera, giorno, notte, tutti) sono solo manuali. Nessuna automazione li cambia in base alla presenza in casa.

**Implementazione:**
- Utilizzare `person.xxx` o `device_tracker.xxx` esistenti in HA
- Automazione: se tutti lontani da casa → profilo "tutti"
- Automazione: se qualcuno torna → profilo "nessuno" (o profilo configurato)
- `input_select.allarme_core_profilo_auto` per configurare il comportamento

**Note:** Richiede che i device tracker siano già configurati in HA.

---

#### 3.8 Dashboard: Indicatore Visuale Stato Globale

**Problema:** Nella plancia operativa non c'è un indicatore grande e immediato dello stato corrente dell'allarme (armato/disarmato/allarme).

**Implementazione — card nella plancia operativa:**
```yaml
type: markdown
content: |
  {% set stato = states('input_select.allarme_core_stato') %}
  {% set profilo = states('input_select.allarme_core_stato') %}
  {% if stato == 'triggered' %}
  <ha-alert alert-type="error">🚨 ALLARME IN CORSO</ha-alert>
  {% elif stato == 'armed' %}
  <ha-alert alert-type="success">🔒 ARMATO — profilo {{ states('input_select.allarme_core_profilo') }}</ha-alert>
  {% elif stato == 'arming' %}
  <ha-alert alert-type="warning">⏳ INSERIMENTO IN CORSO…</ha-alert>
  {% else %}
  <ha-alert alert-type="info">🔓 DISINSERITO</ha-alert>
  {% endif %}
```

---

#### 3.9 Dashboard: Sezione Statistiche e Trend

**Problema:** Non c'è una visione storica o di trend (quante volte la settimana si arma, ore di attivazione, ecc.).

**Implementazione:**
- `type: statistics-graph` card per sensori come `sensor.allarme_core_batteria_minima` (trend nel tempo)
- `type: history-graph` per `input_select.allarme_core_stato` (visualizza arming/armed/disarmed nel tempo)
- Nuova sezione nella plancia operativa: "Andamento"

```yaml
- type: history-graph
  title: Storico Stato Allarme
  hours_to_show: 72
  entities:
    - entity: input_select.allarme_core_stato
      name: Stato
    - entity: input_select.allarme_core_profilo
      name: Profilo
```

---

#### 3.10 Dashboard Plancia Sensori: Filtri per Zona

**Problema:** Con ~50 sensori, la plancia sensori è molto lunga. Non è possibile vedere rapidamente solo i sensori di una zona specifica (es. solo perimetrali).

**Implementazione:**
- Aggiungere una sezione "Vista per Zona" in cima alla plancia sensori
- Una card markdown per zona che elenca solo i sensori di quella zona e il loro stato
- Oppure usare `custom:auto-entities` per filtrare dinamicamente

```yaml
type: markdown
title: "Sensori Zona Perimetrali"
content: |
  {% for s in states.binary_sensor
    | selectattr('entity_id','search','binary_sensor.allarme_core_')
    | selectattr('attributes.zone','defined')
    | selectattr('attributes.zone','contains','perimetrali') %}
  - {{ s.attributes.friendly_name }}: {{ '🔴 APERTO' if s.state == 'on' else '🟢 chiuso' }}
  {% endfor %}
```

---

#### 3.11 Dashboard Plancia Sensori: Riepilogo Stato per Stanza

**Problema:** Nella plancia sensori si vedono i singoli sensori, ma non c'è un colpo d'occhio rapido che dica "cucina: tutto ok" o "taverna: 1 sensore aperto".

**Implementazione:**
- Prima sezione della plancia: griglia di card, una per stanza
- Ogni card mostra il nome stanza e il conteggio sensori aperti
- Card_mod con border rosso se almeno un sensore aperto, verde se tutti chiusi

```yaml
type: tile
entity: binary_sensor.allarme_core_zona_[xxx]
name: Stanza
grid_options:
  columns: 4
  rows: 1
card_mod:
  style: |
    {% set on = is_state('binary_sensor.allarme_core_zona_xxx', 'on') %}
    ha-card { box-shadow: 0 0 0 3px {{ 'red' if on else 'green' }}; }
```

---

### 🟢 PRIORITÀ BASSA / QUALITY OF LIFE

---

#### 3.12 Notifica Push Diretta da HA (senza Node-RED)

**Problema:** Le notifiche dipendono interamente da Node-RED. Se Node-RED è down, nessuna notifica arriva.

**Implementazione:**
- Aggiungere `input_boolean.allarme_core_notifiche_ha_dirette` per attivare la modalità fallback
- Automazione: se `binary_sensor.allarme_core_dati_pronti` → on E Node-RED non disponibile → invia notifica HA nativa tramite `notify.mobile_app_xxx`

---

#### 3.13 Controllo Validità Zone Migliorato

**Problema:** I binary sensor `*_zone_valida` verificano solo il formato della stringa (regex), non se le zone scritte esistono effettivamente nell'input_select delle zone disponibili.

**Implementazione:**
- Aggiornare la logica in `allarme_core_supporto.yaml`
- Aggiungere controllo: ogni zona nella stringa deve essere in `state_attr('sensor.allarme_core_zone_attive','zones_disponibili')`

---

#### 3.14 Card Mappa Planimetrica

**Problema:** Non c'è una visualizzazione planimetrica della casa con i sensori sovrapposti.

**Implementazione con custom card `picture-elements`:**
- Caricare una planimetria come immagine in `/www/`
- Usare `type: picture-elements` con ogni sensore posizionato sulle coordinate corrispondenti
- Ogni elemento è un cerchio colorato (rosso/verde) che mostra lo stato del sensore

```yaml
type: picture-elements
image: /local/planimetria_casa.png
elements:
  - type: state-badge
    entity: binary_sensor.allarme_core_porta_ingresso_sala
    style:
      top: 45%
      left: 30%
```

---

#### 3.15 Export Configurazione Sensori

**Problema:** La configurazione di zona e camera per ogni sensore è distribuita in decine di `input_text` e `input_boolean`. Non c'è modo di esportarla tutta insieme per backup o migrazione.

**Implementazione:**
- Script che legge tutti gli attributi di configurazione e compone un JSON
- Salva in una `var` o `input_text` (base64 se lungo)
- Card markdown che mostra il JSON per copia manuale

---

## 4. MIGLIORAMENTI DASHBOARD SPECIFICI

### Plancia Operativa — Modifiche Consigliate

| Sezione | Proposta | Tipo |
|---------|----------|------|
| Header | Aggiungere indicatore stato critico anche per sensori offline | Modifica header |
| Prima sezione | Aggiungere bottone PANICO con hold action | Nuova card |
| Prima sezione | Aggiungere alert visuale (ha-alert) stato globale allarme | Nuova card markdown |
| Sensori esclusi (esistente) | Aggiungere pulsante "Deseleziona tutti" per reset manuale | Nuova tile |
| Nuova sezione | Storico allarmi (ultimi 20) in tabella markdown | Nuova sezione |
| Nuova sezione | Andamento stato (history-graph 72h) | Nuova sezione |
| Nuova sezione | Configurazione avanzata (auto-reset triggered, timeout arming) | Nuova sezione |

### Plancia Sensori — Modifiche Consigliate

| Sezione | Proposta | Tipo |
|---------|----------|------|
| Prima sezione (nuova) | Riepilogo per stanza: griglia di tile una per stanza | Nuova sezione |
| Seconda sezione (nuova) | Vista per zona: markdown per zona perimetrali/volumetrici/ecc | Nuova sezione |
| Per ogni sensore | Aggiungere badge batteria bassa accanto al badge anomalia | Modifica card_mod |
| Nuova sezione | Mappa planimetrica con picture-elements | Nuova sezione opzionale |

---

## 5. DIPENDENZE NECESSARIE PER LE NUOVE IMPLEMENTAZIONI

| Funzionalità | Dipendenza aggiuntiva |
|---|---|
| Storico allarmi persistente | `var` component (già presente) |
| Geofencing profili | `person` o `device_tracker` già configurati |
| Notifiche HA dirette | `notify.mobile_app_xxx` già configurato |
| Mappa planimetrica | Immagine planimetria casa in `/config/www/` |
| PIN disarmo | `alarm_control_panel` con campo `code:` |

---

## 6. ORDINE DI IMPLEMENTAZIONE CONSIGLIATO

```
1. [ALTA]   Bottone PANICO — impatto sicurezza immediato
2. [ALTA]   Auto-reset da triggered — qualità di vita
3. [ALTA]   Storico allarmi — visibilità eventi passati
4. [ALTA]   Timeout sicurezza su "arming" — rete di sicurezza
5. [MEDIA]  Indicatore visuale stato globale in plancia operativa
6. [MEDIA]  Riepilogo per stanza in plancia sensori
7. [MEDIA]  Filtri per zona in plancia sensori
8. [MEDIA]  Sezione statistiche/andamento in plancia operativa
9. [MEDIA]  Profili automatici via presenza
10. [BASSA] Mappa planimetrica
11. [BASSA] Export configurazione
12. [BASSA] Notifiche HA dirette come fallback
```

---

*File generato automaticamente da analisi del progetto. Aggiornare manualmente dopo ogni implementazione.*
