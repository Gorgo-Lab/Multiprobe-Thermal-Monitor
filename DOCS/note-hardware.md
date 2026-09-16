# Note hardware

Il perché delle scelte di cablaggio. Le note di `DOCS/wiring.yaml` sono appunti da banco e rimandano qui con riferimenti tra parentesi quadre, per esempio `[condensatore]`: ognuno corrisponde a una sezione con lo stesso nome.

Le istruzioni da seguire mentre si salda restano scritte nelle note del diagramma. Qui c'è solo la spiegazione.

---

## [bus-separati]

Ogni sonda sta sul **proprio** bus 1-Wire, su un GPIO dedicato (`GPIO4`, `GPIO5`, `GPIO6`), invece di tutte e tre sulla stessa linea.

1-Wire nasce per far convivere più dispositivi sulla stessa linea, quindi il bus condiviso sarebbe l'uso "canonico". Qui però costa più di quanto rende:

- **Niente indirizzi da configurare.** Con più sonde sullo stesso pin gli indirizzi a 64 bit diventano obbligatori: vanno letti dai log e incollati nel firmware. ESPHome sceglie da solo il dispositivo solo se sul bus ce n'è uno (`one_wire.cpp:22-35`).
- **Sostituire una sonda è un'operazione solo fisica**: si stacca, si attacca e si riavvia la scheda, perché la ricerca sul bus avviene solo all'avvio. Con gli indirizzi, ogni sonda nuova richiederebbe di rileggere i log e riflashare.
- **Un guasto resta confinato.** Una sonda che tiene la linea dati bassa blocca solo il proprio bus. Sul bus condiviso sparirebbero tutte e tre le letture.
- **Ogni tratta è punto-punto**, quindi non c'è il problema delle riflessioni di un bus a stella.

Costa due GPIO e due resistenze in più, su una scheda con 13 pin liberi. Il bus condiviso tornerebbe conveniente solo con molte più sonde, o con pochi pin disponibili.

I pin sono adiacenti sull'header, tutti liberi e nessuno di strapping. `GPIO3` lo è, per questo si parte da `GPIO4`.

## [pin-entita]

Quale sonda corrisponde a quale entità in Home Assistant lo decide **il pin**: `GPIO4` → *Sonda 1*, `GPIO5` → *Sonda 2*, `GPIO6` → *Sonda 3*.

Se si scambiano due tratte, si scambiano anche le due entità, e nulla lo segnala: le letture restano tutte plausibili. Con gli indirizzi questo errore era impossibile, perché l'indirizzo seguiva la sonda. È il prezzo di [bus-separati]: **etichettare i cavi**.

## [pull-up]

Serve **una resistenza da 4.7 kΩ per ogni bus**, quindi tre in totale, ognuna tra la propria linea dati e il nodo 3V3.

- Non sono in parallelo tra loro: ognuna serve il proprio bus. Con una sola resistenza condivisa, due linee dati resterebbero flottanti.
- Più resistenze sulla **stessa** linea sono invece un errore: tre da 4.7 kΩ in parallelo fanno ~1.6 kΩ, fuori specifica.
- La resistenza va **dal lato master**, cioè sulla scheda, non in fondo alla linea vicino alla sonda: è la convenzione 1-Wire.

Nel diagramma i "cavi" tra resistenza, GPIO e nodo 3V3 sono i reofori della resistenza. WireViz richiede un cavo per ogni collegamento, ma fisicamente non c'è nessun filo da tagliare.

## [nodi]

`N_3V3` e `N_GND` sono **nodi elettrici** (net), non componenti. Dicono solo cosa è collegato con cosa:

- su `N_3V3` arrivano il pin `3V3` della scheda, i tre VDD delle sonde e i tre pull-up: **7 collegamenti**;
- su `N_GND` arrivano il pin `GND` della scheda e i tre GND delle sonde.

Un punto comune serve anche con le linee dati punto-punto, perché sette conduttori non stanno su un pin di header.

**Sulla millefori** un nodo si fa con un filo bus: un filo rigido nudo (va bene un reoforo avanzato) appoggiato lungo una fila di fori e saldato a ogni piazzola. Tutto ciò che deve stare sul nodo si salda in un punto qualsiasi di quel filo.

"Nodo" qui significa net, **non topologia a stella**. Lo stesso net si può realizzare a stella o a catena, e in questo circuito non fa differenza: sul nodo 3V3 passano ~4.5 mA (tre DS18B20 in conversione) più ~0.7 mA per pull-up attivo, che su 5 cm di pista fanno mezzo millivolt di caduta. La messa a terra a stella serve quando ci sono correnti alte e un riferimento analogico sensibile sulla stessa massa. Qui mancano entrambi: la DS18B20 è digitale.

## [3v3]

Le sonde vanno alimentate a **3.3 V, non a 5 V**. La linea dati arriva direttamente a un GPIO dell'ESP32, che non tollera 5 V.

## [parassita]

Si usa l'alimentazione normale a 3 fili, non quella parassita (VDD a massa, sonda alimentata dalla sola linea dati). La modalità parassita serve a risparmiare un conduttore, che nel cavo delle sonde c'è già. In cambio è più fragile, soprattutto in conversione a 12 bit.

## [colori-sonde]

Nero = GND, giallo = DQ, rosso = VDD è la combinazione più diffusa, ma **non è uno standard**. Va verificata sul modello che si ha in mano prima di collegare: invertire VDD e GND distrugge la sonda.

Il cavo di una sonda è uno solo, a 3 fili, ma i conduttori finiscono in tre punti diversi (nodo GND, GPIO, nodo 3V3). Per questo nel diagramma compare spezzato in due voci.

## [alimentazione]

La scheda si alimenta dalla **propria presa USB-C**, con un normale cavo USB da un caricatore di rete (5 V, almeno 1 A). Non è un collegamento da saldare, per questo non compare nel diagramma.

- **Perché un caricatore e non 220 V → 12 V → buck → 5 V.** Il carico è un solo ESP32 da ~150 mA, meno di 1 W: due stadi di conversione aggiungono componenti e punti di guasto senza vantaggi. Il caricatore tiene la 220 V dentro un guscio certificato, fuori dalla scatola. Il 12 V + buck avrebbe senso solo con carichi a 12 V reali, o con tratte DC oltre i ~10 m.
- **Il pin `5V` dell'header è lo stesso rail del VBUS dell'USB.** Qui non alimenta: serve solo per attaccare il condensatore. **Non collegarci mai una seconda sorgente**, sarebbero due alimentazioni in parallelo.
- **La presa USB-C della scheda è saldata in superficie** e si stacca dal PCB se il cavo viene tirato. Fissare il cavo alla scatola con una fascetta, così lo strappo non arriva al connettore.

## [cavo]

Sul cavo USB conta la caduta di tensione, non la lunghezza in sé.

Il regolatore della scheda vuole almeno ~3.6 V in ingresso: partendo da 5 V si possono perdere fino a ~1.4 V. A 0.5 A di picco su 5 m:

| Conduttore | Sezione | Caduta | Esito |
|---|---|---|---|
| AWG24 (cavo ethernet) | 0.205 mm² | 0.42 V | abbondante |
| AWG28 (USB economico) | 0.081 mm² | 1.06 V | al limite |
| AWG30 (USB ultrasottile) | 0.051 mm² | 1.70 V | brownout |

Il condensatore assorbe localmente i picchi, quindi in pratica il cavo vede i ~150 mA medi. Il caso che fallisce davvero è il cavetto USB ultrasottile e lungo, con reset che sembrano casuali.

Contano anche due cose che non si calcolano: caricatori che sotto carico danno 4.75-4.80 V invece di 5.00 V, e la resistenza di contatto di connettori USB-C economici o usurati.

Per tratte lunghe è meglio una **prolunga 220 V** con il caricatore vicino alla scatola, invece di un cavo USB lungo.

## [condensatore]

Un **elettrolitico da 100 µF** tra `5V` e `GND`, e basta.

**Perché serve.** La radio WiFi assorbe picchi da 350-500 mA per pochi millisecondi. L'alimentatore non li serve, perché sta dall'altra parte di un cavo con resistenza e induttanza: li serve la capacità vicina alla scheda. Senza, la tensione cala, il brownout detector scatta sotto ~2.8 V e la scheda si riavvia. Il sintomo tipico è il riavvio proprio mentre si connette al WiFi, o riavvii casuali ogni poche ore.

**Perché senza il ceramico da 100 nF in parallelo.** La coppia "elettrolitico + 100 nF" è la regola per il disaccoppiamento accanto a un integrato, dove il ceramico serve i fronti da nanosecondo. Qui il condensatore sta all'ingresso di un regolatore LDO: i fronti veloci sono sul rail 3.3 V a valle, dove la scheda ha già i suoi condensatori. Un LDO è un elemento in serie, quindi un gradino di 300 mA in uscita è un gradino di 300 mA in ingresso, con un tempo di salita dell'ordine del microsecondo. A quella scala l'induttanza dell'elettrolitico (~10 nH, ~0.02 Ω) è trascurabile e conta la sua resistenza serie, che con un low-ESR a 300 mA vale qualche decina di millivolt. Il ceramico servirebbe sopra qualche MHz, dove su questo nodo non c'è richiesta.

**Posizionamento.** Deve stare sulla scheda, non a metà del cavo di alimentazione. Qualche centimetro di reoforo in più non cambia nulla, per lo stesso motivo: su transitori da microsecondo conta la resistenza serie, non l'induttanza. Il passo tipico è 5 mm (due spaziature di fori), mentre `5V` e `GND` sono pin adiacenti (2.54 mm): meglio due pontini corti che forzare i reofori.

## [polarita]

L'elettrolitico è polarizzato: `+` al `5V`, `−` al `GND`.

- La **banda sul cilindro** indica il **negativo**. Il `+` spesso non è stampato.
- Il reoforo più lungo è il positivo, ma solo finché non si tagliano le gambe.
- Montato al contrario si scalda, si gonfia e sfiata. A 5 V da caricatore non è pericoloso, ma il condensatore è da buttare.

## [debug]

Un ponticello su due pin tra `GPIO13` e `GND` sceglie la modalità all'avvio:

- **scollegato**: modalità normale;
- **inserito**: modalità debug, per verificare il cablaggio e sviluppare.

Il pin ha il pull-up interno attivo, quindi non serve una resistenza. Il firmware legge il ponticello **solo all'avvio**: dopo averlo spostato bisogna riavviare la scheda.

`GPIO13` sta sulla stessa fila dell'header del pin `GND`, a due posizioni di distanza, ed è libero e non di strapping. Non è adiacente al `GND`: in mezzo c'è `3V3`. Per questo l'header del ponticello va sulla millefori con due fili corti, e non va chiuso direttamente sui pin del modulo, dove si rischierebbe di cortocircuitare `3V3`. Non si usa `GPIO4` come negli altri progetti perché qui è la sonda 1.
