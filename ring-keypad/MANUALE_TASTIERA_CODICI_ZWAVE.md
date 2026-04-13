disarmare allare senza voce:
Topic: zwave2mqtt/tastiera_allarme_camera/indicator/endpoint_0/Not_armed_-_disarmed/9/set
payload: 1
disarmare allare con voce:
Topic: zwave2mqtt/tastiera_allarme_camera/indicator/endpoint_0/Not_armed_-_disarmed/9/set
payload: 99

codice inserito errato con suono
Topic: zwave2mqtt/tastiera_allarme_camera/indicator/endpoint_0/Code_not_accepted/9/set
payload: 99
codice inserito errato senza suono
Topic: zwave2mqtt/tastiera_allarme_camera/indicator/endpoint_0/Code_not_accepted/9/set
payload: 1


allarme armato in casa con voce
Topic: zwave2mqtt/tastiera_allarme_camera/indicator/endpoint_0/Armed_Stay/9/set
payload: 99
allarme armato in casa senza voce
Topic: zwave2mqtt/tastiera_allarme_camera/indicator/endpoint_0/Armed_Stay/9/set
payload: 1

allarme armato fuori casa con voce
Topic: zwave2mqtt/tastiera_allarme_camera/indicator/endpoint_0/Armed_Away/9/set
payload: 99
allarme armato fuori casa senza voce
Topic: zwave2mqtt/tastiera_allarme_camera/indicator/endpoint_0/Armed_Away/9/set
payload: 1

allarme scattato silenzioso
Topic: zwave2mqtt/tastiera_allarme_camera/indicator/endpoint_0/Alarming/9/set
payload: 1
allarme scattato con suono
Topic: zwave2mqtt/tastiera_allarme_camera/indicator/endpoint_0/Alarming/9/set
payload: 99

allarme panico silenzioso
Topic: zwave2mqtt/tastiera_allarme_camera/indicator/endpoint_0/Alarming_Burglar/9/set
payload: 1
allarme panico con suono
Topic: zwave2mqtt/tastiera_allarme_camera/indicator/endpoint_0/Alarming_Burglar/9/set
payload: 99

allarme scattato silenzioso fire
Topic: zwave2mqtt/tastiera_allarme_camera/indicator/endpoint_0/Alarming_Smoke_-_Fire/9/set
payload: 1
allarme scattato con suono fire
Topic: zwave2mqtt/tastiera_allarme_camera/indicator/endpoint_0/Alarming_Smoke_-_Fire/9/set
payload: 99

allarme medico attivato con voce
Topic: zwave2mqtt/tastiera_allarme_camera/indicator/endpoint_0/Alarming_Medical/9/set
payload: 99
allarme medico attivato senza voce
Topic: zwave2mqtt/tastiera_allarme_camera/indicator/endpoint_0/Alarming_Medical/9/set
payload: 1

allarme Perdita acqua attivato con voce
Topic: zwave2mqtt/tastiera_allarme_camera/indicator/endpoint_0/Alarming_Water_leak/9/set
payload: 99
allarme Perdita acqua attivato senza voce
Topic: zwave2mqtt/tastiera_allarme_camera/indicator/endpoint_0/Alarming_Water_leak/9/set
payload: 1

allarme bassa temperatura attivato con voce
Topic: zwave2mqtt/tastiera_allarme_camera/indicator/endpoint_0/Alarming_Freeze_warning/9/set
payload: 99
allarme bassa temperatura attivato senza voce
Topic: zwave2mqtt/tastiera_allarme_camera/indicator/endpoint_0/Alarming_Freeze_warning/9/set
payload: 1

bypass in caso di sensori aperti
Topic: zwave2mqtt/tastiera_allarme_camera/indicator/endpoint_0/Bypass_challenge/1/set
payload: 99

conto alla rovescia ingresso (il formato del payload deve essere stringa "0m30s" questo esempio per 30 secondi)
Topic: zwave2mqtt/tastiera_allarme_camera/indicator/endpoint_0/Entry_Delay/timeout/set
payload: "0m5s"
conto alla rovescia uscita (il formato del payload deve essere stringa "0m30s" questo esempio per 30 secondi)
Topic: zwave2mqtt/tastiera_allarme_camera/indicator/endpoint_0/Exit_Delay/timeout/set
payload: "0m30s"

impostare il volume della sirena della tastiera da 1 a  10
topic: zwave2mqtt/tastiera_allarme_camera/configuration/endpoint_0/Siren_Volume/set
payload: 5


imposta le luci degli allarmi quanto devono stare accese o spente oppure la durata del lampeggio:
1-600 durata del lampeggio in secondi
601 sempre accese
0 spente
topic: zwave2mqtt/tastiera_allarme_camera/configuration/endpoint_0/System_Security_Mode_Display/set
payload: 601





Topic per leggere codice inserito per disattivare allarme: zwave2mqtt/tastiera_allarme_camera/unknownClass_111/endpoint_0/3/2 ----- esempio: {"time":1775987201571,"value":"1234"}
Topic per leggere codice inserito per attivare allarme in casa: zwave2mqtt/tastiera_allarme_camera/unknownClass_111/endpoint_0/6/2 ----- esempio: {"time":1775987284614,"value":"9876"}
Topic per leggere codice inserito per attivare allarme fuori casa: zwave2mqtt/tastiera_allarme_camera/unknownClass_111/endpoint_0/5/2 ----- esempio: {"time":1775987329622,"value":"6543"}
Topic per leggere codice inserito premendo poi la v sulla tastiera: zwave2mqtt/tastiera_allarme_camera/unknownClass_111/endpoint_0/2/2 ----- esempio: {"time":1775987372275,"value":"8585"}
Topic per leggere codice inserito premendo poi la x sulla tastiera: zwave2mqtt/tastiera_allarme_camera/unknownClass_111/endpoint_0/25/2 ----- esempio: {"time":1775987396617,"value":"2323"}



topic per leggere se è stato premuto per piu di 3 secondi allarme anti rapina e quindi attivato: zwave2mqtt/tastiera_allarme_camera/unknownClass_111/endpoint_0/17/0 ----- esempio: {"time":1775988051557}
topic per leggere se è stato premuto per piu di 3 secondi allarme medico e quindi attivato: zwave2mqtt/tastiera_allarme_camera/unknownClass_111/endpoint_0/19/0 ----- esempio: {"time":1775988051557}
topic per leggere se è stato premuto per piu di 3 secondi allarme incendio e quindi attivato: zwave2mqtt/tastiera_allarme_camera/unknownClass_111/endpoint_0/16/0 ----- esempio: {"time":1775988051557}