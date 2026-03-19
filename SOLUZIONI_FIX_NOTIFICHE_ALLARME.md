# Soluzioni Fix Notifiche Allarme — Race Condition & Dati Pronti

Data analisi: 2026-03-18

---

## Problemi identificati

1. **Race condition** — Il template `sensori_aperti_filtrati` non funziona in stato `triggered`, causando la scrittura del var sensori vuoto
2. **Notifiche premature** — Le notifiche vengono inviate prima che screenshot e clip siano disponibili
3. **Nessun gate "tutto pronto"** — Non esiste un punto unico che certifica che sensori + immagini + clip siano disponibili prima di notificare

---

## Fix Race Condition

### Opzione B — Estendere sensori_aperti_filtrati

Modificare il template sensor `sensori_aperti_filtrati` affinché funzioni anche in stato `triggered`:

```yaml
# Prima:
# stato not in ['armed_home', 'armed_away'] → []
# Dopo:
# stato not in ['armed_home', 'armed_away', 'triggered'] → []
```

- Minima modifica, elimina la race condition
- Il template sensor ha semantica leggermente allargata (funziona anche in triggered)

### Opzione C — Leggere direttamente i binary_sensor fisici

L'automation che memorizza i sensori legge direttamente `states.binary_sensor.*` invece del template sensor:

- I sensori fisici non cambiano stato in risposta all'allarme → nessuna race condition possibile
- Più robusto ma duplica la logica di filtro (deve replicare i criteri di inclusione/esclusione)

### Raccomandazione: B + C insieme

- **B** come rete di sicurezza sul template sensor
- **C** come fix primario nell'automation

---

## Fix Notifiche — Binary Sensor "Dati Pronti"

### Logica del sensore

`binary_sensor.allarme_core_dati_pronti` diventa `on` solo quando:

```
stato == triggered
  AND var sensori non vuoto
  AND (nessuna camera OR snapshot aggiornati dopo l'ultimo trigger)
  AND (nessuna camera OR clip aggiornate dopo l'ultimo trigger)
```

### Opzione 1 — Template sensor reattivo

Il binary sensor è un template che si rivaluta automaticamente man mano che i var vengono aggiornati.
Diventa `true` nel momento esatto in cui l'ultimo dato viene scritto.

**Pro**: automatico, zero overhead, nessuna automation aggiuntiva
**Contro**: se una telecamera fallisce non diventa mai `true` → serve un timeout esterno

### Opzione 2 — input_boolean controllato da automation

- Reset a `off` quando `stato → triggered`
- L'automation snapshot/clip setta a `on` al termine
- Se non ci sono camere, settato direttamente dalla automation `memorizza_sensori`

**Pro**: esplicito, controllabile, gestisce nativamente il caso "nessuna camera"
**Contro**: dipende dall'automation che lo setta correttamente — se l'automation fallisce rimane `off`

### Opzione 3 — Template sensor con timeout di fallback ⭐ CONSIGLIATA

Combina la reattività del template con una rete di sicurezza:
- Se dopo N secondi da `triggered` le condizioni non sono tutte soddisfatte → diventa `true` comunque
- Aggiunge un attributo `dati_completi: false` per segnalare che qualcosa mancava

**Pro**: non blocca mai le notifiche, segnala se i dati sono incompleti
**Contro**: complessità leggermente maggiore

---

## Flusso notifiche con binary sensor "dati pronti"

```
stato → triggered
    │
    ├─ [fix B+C] var sensori scritto correttamente (< 1s)
    │
    ├─ automation snapshot/clip (già esistente, con wait_for_trigger)
    │     → scarica immagini e clip
    │     → aggiorna var.allarme_core_last_snapshot_urls
    │     → aggiorna var.allarme_core_last_clip_urls
    │
    └─ binary_sensor.allarme_core_dati_pronti
          → diventa ON solo quando tutto è pronto
          │
          └─ automation notifica
                → invia con sensori + immagini + clip garantiti disponibili
```

---

## Raccomandazione finale

| Fix | Priorità |
|-----|----------|
| Fix B — estendere template sensori_aperti_filtrati | Alta |
| Fix C — leggere binary_sensor fisici nell'automation | Alta |
| Opzione 3 — template sensor con timeout di fallback | Alta |

Implementare nell'ordine:
1. Fix B+C per eliminare la race condition
2. Binary sensor con Opzione 3 come gate per le notifiche
3. Aggiornare l'automation notifica per triggerarsi sul binary sensor invece che su `triggered`
