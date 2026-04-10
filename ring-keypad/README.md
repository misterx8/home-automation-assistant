# Ring Keypad V2 — Package Home Assistant

Gestione multi-tastiera Ring Keypad V2 via Z-Wave (zwave2mqtt).  
Supporta N tastiere senza duplicare il codice. Parametrizzabile completamente dalla UI HA.

## Struttura file

```
ring-keypad/
├── packages/
│   ├── ring_keypad_globals.yaml        # Helper globali: stato, utenti, PIN
│   ├── ring_keypad_scripts.yaml        # Script parametrici (accettano keypad_id)
│   ├── ring_keypad_automations.yaml    # Automazioni con wildcard MQTT
│   ├── ring_keypad_keypads.yaml        # Sensori MQTT per tastiera fisica
│   └── ring_keypad_integration.yaml   # Bridge verso allarme-core (o altri sistemi)
└── ANALISI_TECNICA.md                  # Documentazione tecnica completa
```

## Installazione rapida

1. Copiare i 5 file `packages/*.yaml` in `config/packages/ring_keypad/`
2. Aggiungere in `configuration.yaml`:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
3. Riavviare HA
4. Configurare dalla UI:
   - **Tastiere attive**: `input_text.ring_keypad_active_keypads` → topic base Z-Wave (es. `tastiera_camera`)
   - **PIN**: `ring_keypad_pin_master`, `ring_keypad_pin_user1..3` (campi password)
   - **Nomi**: `ring_keypad_name_master`, `ring_keypad_name_user1..3`

## Aggiungere una tastiera

1. Aggiungere il topic base a `ring_keypad_active_keypads` (separato da virgola)
2. Decommentare il blocco template in `ring_keypad_keypads.yaml` e adattarlo
3. Nessun'altra modifica necessaria — il wildcard MQTT gestisce tutto automaticamente

## Integrazione con allarme-core

Attiva di default in `ring_keypad_integration.yaml` (sezione 1).  
Per sistemi alternativi (Alarmo, alarm_control_panel): vedere sezione 2 commentata.

## Documentazione

Vedere `ANALISI_TECNICA.md` per:
- Protocollo Z-Wave/MQTT completo
- Flusso operativo dettagliato
- Mappature stati
- Note architetturali
