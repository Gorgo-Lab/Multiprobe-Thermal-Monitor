# Multiprobe Thermal Monitor

Firmware ESPHome per **ESP32-S3 Super Mini**: legge **3 sonde di temperatura** nello stesso ciclo di campionamento, a intervallo regolare, e pubblica le letture **via MQTT** verso Home Assistant (che le scopre da solo tramite MQTT discovery).

> **Stato: firmware e wiring completi, mai provati su hardware reale.** Non c'è nulla da configurare prima del primo flash.

## Struttura del progetto

| Percorso | Cos'è |
|---|---|
| `multiprobe_thermal_monitor.yaml` | Flagship: il file da flashare per l'uso reale. ESP32-S3 + 3 sonde + MQTT. |
| `DOCS/wiring.yaml` (+ `.html`/`.png`) | Wiring: schemi collegamenti. `.html`/`.png` sono artefatti rigenerati, non editarli a mano. |
| `DOCS/note-hardware.md` | Il perché delle scelte di cablaggio. Le note del wiring rimandano qui con riferimenti tipo `[condensatore]`. |
| `common/status_led.yaml` | Modulino riutilizzabile: pilota il LED RGB onboard come indicatore di stato (codice colore documentato in testa al file). |
| `bench/` | Varianti di sviluppo/test — vuota per ora, verrà popolata man mano che servono (es. una variante con una sola sonda per validare il cablaggio, o una che pubblica valori simulati per testare il lato Home Assistant senza hardware). |

## Hardware

- **Board**: ESP32-S3 "Super Mini" (board cinese generica, 22.5 × 18 mm, USB-C nativo). Caratterizzata con esptool nel repo gemello [`../esp32-S3-from-china`](../esp32-S3-from-china): ESP32-S3 QFN56 rev v0.2, 4 MB flash, 2 MB PSRAM, USB-Serial/JTAG **senza bridge UART esterno**.
  - Conseguenza pratica: su macOS la board si presenta come `/dev/cu.usbmodem*`, e il `logger` **deve** usare `hardware_uart: USB_SERIAL_JTAG`.
  - GPIO liberi sugli header: `1`–`13`, più TX/RX. Il LED RGB WS2812 onboard è su `GPIO48`.
- **Sonde**: 3 × **DS18B20** waterproof (1-Wire digitale, ±0.5 °C, −55/+125 °C), ognuna sul **proprio bus** — `GPIO4`, `GPIO5`, `GPIO6` — con la **propria** resistenza di pull-up da 4.7 kΩ verso 3V3 (tre in totale, una per bus). Alimentazione condivisa, normale a 3 fili (non parassita). Vedi `DOCS/wiring.yaml`.
  - **Quale sonda è quale entità lo decide il pin**: `GPIO4` → *Sonda 1*, `GPIO5` → *Sonda 2*, `GPIO6` → *Sonda 3*. Etichetta i cavi: scambiare due tratte sui GPIO scambia due entità e nulla te lo segnala.
- **Alimentazione**: caricatore USB da rete (5V, ≥1A) sulla porta USB-C, con un elettrolitico da 100 µF tra i pin `5V` e `GND`. Scelta del cavo e motivazioni in [`DOCS/note-hardware.md`](DOCS/note-hardware.md).

## Indicatore di stato (LED RGB onboard)

Il LED onboard segnala i guasti che **non si possono vedere in nessun altro modo**: se Home Assistant riceve i dati, i problemi si vedono lì; il LED serve esattamente quando i dati non arrivano, cioè quando è rotto il canale che te lo avrebbe detto.

Codice binario, due soli stati:

| Segnale | Significato |
|---|---|
| ⚪ **bianco, lampeggio lento** (1 ogni 2s) | Tutto ok |
| 🔴 **rosso, lampeggio veloce** (~3 Hz) | Qualcosa non va |
| ⚫ **spento del tutto** | Scheda senza alimentazione o firmware bloccato |

L'informazione sta nella **velocità** prima ancora che nel colore: lento e veloce si distinguono con la coda dell'occhio, senza ricordare una legenda e senza dipendere dalla percezione dei colori.

Il lampeggio lento quando tutto va bene non è decorativo: è ciò che rende **non ambiguo** il terzo caso. Senza, "LED spento" significherebbe sia "tutto bene" sia "scheda morta" — il guasto più importante da notare.

**Cosa accende il rosso**: WiFi non connesso, broker MQTT irraggiungibile, o almeno una sonda muta. Il rosso dice *che* qualcosa non va, non *cosa*: il dettaglio sta nei log e in Home Assistant. È una scelta deliberata — un indicatore che va interpretato non si guarda passando.

Il rosso non scatta per un glitch occasionale sul bus: il filtro `median` scarta i NaN dalla finestra e il sensore pubblica NaN solo se *tutti* e 10 i campioni lo sono, quindi serve una sonda muta per l'intera finestra di 30s.

> **Conseguenza**: il LED è `internal`, quindi **non è più una luce controllabile da Home Assistant**. Non può fare entrambe le cose: se HA lo accendesse, starebbe sovrascrivendo un avviso.

---

## Installazione ambiente

Il progetto usa [uv](https://docs.astral.sh/uv/) per la gestione dell'ambiente Python/dipendenze.

```bash
# 1. installa uv (se non già presente)
curl -LsSf https://astral.sh/uv/install.sh | sh

# 2. installa le dipendenze (crea automaticamente il venv)
uv sync
```

### Secrets

Copia `secrets.yaml.example` in `secrets.yaml` (gitignored) e inserisci credenziali WiFi e broker MQTT:

```bash
cp secrets.yaml.example secrets.yaml
```

### Lato Home Assistant

Servono un broker MQTT raggiungibile (es. l'add-on Mosquitto) e l'integrazione **MQTT** configurata in HA. Nessuna entità da dichiarare a mano: ESPHome pubblica i messaggi di discovery sul prefisso `homeassistant/` e le tre temperature compaiono da sole come sensori del dispositivo.

---

## Primo flash

Nessuna configurazione da ricavare dall'hardware: cabla, flasha, funziona. Ogni DS18B20 ha un indirizzo univoco a 64 bit inciso in fabbrica, ma essendo sola sul proprio bus viene auto-selezionata da ESPHome — non c'è niente da leggere dai log e incollare nel firmware.

Se una sonda non pubblica, il problema è nel cablaggio di **quella** tratta (pull-up assente sul suo pin, massa, sonda guasta): le altre due continuano a funzionare, ed è esattamente il motivo per cui i bus sono separati.

---

## Utilizzo

### Flash + log (tutto in uno)

```bash
uv run esphome run multiprobe_thermal_monitor.yaml
```

### Solo log (device già flashato)

```bash
uv run esphome logs multiprobe_thermal_monitor.yaml
```

### Porta seriale esplicita

Se ESPHome non rileva automaticamente la porta:

```bash
uv run esphome run multiprobe_thermal_monitor.yaml --device /dev/cu.usbmodem1101   # macOS
uv run esphome run multiprobe_thermal_monitor.yaml --device /dev/ttyACM0           # Linux
```

Per usare una delle varianti in `bench/` (quando presenti), sostituisci il percorso del file.

---

## Sviluppo

### Validare un config senza flashare

```bash
uv run esphome config multiprobe_thermal_monitor.yaml
```

### Compilare senza flashare

Verifica che il C++ generato compili davvero — cattura errori che `esphome config` non vede:

```bash
uv run esphome compile multiprobe_thermal_monitor.yaml
```

### Rigenerare i diagrammi di wiring (WireViz via Docker)

`DOCS/wiring.yaml` è sorgente [WireViz](https://github.com/wireviz/WireViz). Per rigenerare `.html`/`.png` dopo una modifica:

```bash
docker run --rm -v "$(pwd)":/root/src znibb/wireviz:latest "wireviz DOCS/wiring.yaml -f hp"
```

`-f hp` genera sia HTML che PNG.

---

## Note di progettazione

Qui le scelte firmware. Il perché del cablaggio (bus separati, pull-up, nodi, alimentazione, cavo, condensatore) sta in [`DOCS/note-hardware.md`](DOCS/note-hardware.md).

**Perché MQTT e non l'API nativa ESPHome**: scelta esplicita, il monitor pubblica su un broker già presente in casa. Il prezzo è che senza il blocco `api:` la dashboard ESPHome non può leggere i log via rete e Home Assistant non usa l'integrazione nativa; i due trasporti possono comunque coesistere nello stesso firmware, se in futuro servisse.

**Cosa significa "contemporaneamente"**: le 3 sonde condividono lo stesso `update_interval`, quindi ESPHome le aggiorna nello stesso ciclo e le tre conversioni (750 ms a 12 bit) si sovrappongono sul bus invece di andare in sequenza. Anche le finestre della mediana restano allineate, quindi i tre valori pubblicati si riferiscono sempre allo stesso intervallo di tempo. È simultaneità *ai fini della misura* — per una grandezza lenta come la temperatura è abbondante. Non è invece un campionamento atomico con timestamp unico: se servisse (es. un ΔT istantaneo tra due sonde con requisiti stretti), il design andrebbe rivisto.

**Perché la mediana e non la media**: il filtro non migliora l'accuratezza della sonda. I ±0.5 °C della DS18B20 sono errore *sistematico* di calibrazione — quella sonda legge sempre 0.3 °C in più, e mediare mille campioni restituisce 0.3 °C in più con più cifre decimali. Mediare abbatte solo il rumore *casuale*, che su un sensore digitale con ADC e filtro interni vale ~1 LSB (0.0625 °C): guadagneresti 0.02 °C su uno strumento che sbaglia di 0.5 °C. Quello che il filtro fa davvero è **scartare le letture sballate** che un bus 1-Wire disturbato può produrre — e per quello la mediana è nettamente meglio della media: un singolo campione spazzatura non la sposta, mentre trascinerebbe la media.

**Perché `sample_interval` non può scendere sotto 1s**: a 12 bit la conversione della DS18B20 dura 750 ms. Sotto quella soglia il sensore non pubblica *nulla* — non "meno spesso": nulla. ESPHome programma la lettura con `set_timeout(nome_sensore, …)`, e registrare un timeout con un nome già in uso **cancella** quello pendente: ogni nuovo ciclo di campionamento annullerebbe la lettura del ciclo precedente prima che scatti.

**Perché la diagnostica (RSSI, uptime, versione) è nel flagship e non solo nelle varianti di banco**: su questa board l'RSSI è la prima cosa da guardare quando il device sparisce da HA — girano esemplari con l'antenna montata al contrario.

---

## Licenza

Questo progetto è distribuito con licenza MIT (vedi `LICENSE`).
