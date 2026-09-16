# Multiprobe Thermal Monitor

Firmware [ESPHome](https://esphome.io) per **ESP32-S3 Super Mini** che legge **3 sonde di temperatura DS18B20** e pubblica le misure **via MQTT** su Home Assistant.

## Come funziona

| Ogni | Cosa succede |
|---|---|
| **3 s** | Le tre sonde vengono lette insieme, nello stesso ciclo. |
| **30 s** | Per ogni sonda si pubblica su MQTT la **mediana** degli ultimi 10 campioni. |

- La prima temperatura arriva **30 s dopo l'accensione**, quando la finestra di 10 campioni è piena.
- La mediana scarta le letture errate occasionali del bus. Se una sonda non risponde per un'intera finestra, pubblica `NaN` e il LED passa a rosso.
- Home Assistant trova le entità da solo tramite **MQTT discovery**: niente da configurare lato HA oltre all'integrazione MQTT.

### Modalità debug

Con un ponticello tra `GPIO7` e `GND` inserito all'avvio, la scheda parte in modalità debug. La logica è la stessa, cambiano solo i tempi:

| | Normale | Debug |
|---|---|---|
| Lettura sonde | ogni 3 s | ogni 2 s |
| Pubblicazione mediana | ogni 30 s | ogni 2 s |
| Valori grezzi (*Sonda N grezzo*) | non pubblicati | ogni 2 s |

Il ponticello si legge solo all'avvio: dopo averlo spostato bisogna riavviare la scheda.

Entità esposte: le tre temperature (**Sonda 1**, **Sonda 2**, **Sonda 3**, in °C).

### Diagnostica

Entità separate dalle misure (categoria *diagnostic* in Home Assistant), utili quando qualcosa non va:

| Entità | Serve a capire | Aggiornamento |
|---|---|---|
| Uptime | se la scheda si è riavviata | 60 s |
| Causa ultimo riavvio | perché: alimentazione (brownout), crash, spegnimento | all'avvio |
| WiFi signal | se la connessione è debole | 60 s |
| Letture fallite | se un bus sta peggiorando, prima che la sonda smetta del tutto | 60 s |
| ESPHome version | quale firmware è in esecuzione dopo un aggiornamento | all'avvio |

Il contatore delle letture fallite conta gli errori che la mediana nasconde, e si azzera a ogni riavvio.

### LED di stato

| LED | Significato |
|---|---|
| ⚪ bianco, lampeggio lento | tutto ok |
| 🔴 rosso, lampeggio veloce | WiFi assente, broker MQTT irraggiungibile o sonda guasta |
| ⚫ spento | scheda senza alimentazione o bloccata |

## Hardware

| Componente | Quantità | Note |
|---|---|---|
| ESP32-S3 Super Mini | 1 | alimentata dalla presa USB-C |
| Sonda DS18B20 waterproof | 3 | cavo a 3 fili |
| Resistenza 4.7 kΩ ¼ W | 3 | un pull-up per sonda |
| Condensatore elettrolitico 100 µF | 1 | ≥ 10 V, tra `5V` e `GND` |
| Caricatore USB | 1 | 5 V, almeno 1 A |
| Header 2 pin + jumper | 1 | ponticello debug su `GPIO7` |

Collegamenti:

| Sonda | Pin dati | Entità |
|---|---|---|
| 1 | `GPIO4` | Sonda 1 |
| 2 | `GPIO5` | Sonda 2 |
| 3 | `GPIO6` | Sonda 3 |

Ogni sonda ha il proprio bus 1-Wire, quindi non servono gli indirizzi delle DS18B20. VDD delle sonde a **3.3 V**, non a 5 V.

Schema completo: [`DOCS/wiring.png`](DOCS/wiring.png) (sorgente [`DOCS/wiring.yaml`](DOCS/wiring.yaml)). Il perché delle scelte di cablaggio è in [`DOCS/note-hardware.md`](DOCS/note-hardware.md).

## Installazione

Serve [uv](https://docs.astral.sh/uv/).

```bash
uv sync
cp secrets.yaml.example secrets.yaml   # credenziali WiFi e broker MQTT
```

Lato Home Assistant servono un broker MQTT (per esempio l'add-on Mosquitto) e l'integrazione MQTT attiva.

## Uso

```bash
uv run esphome run multiprobe_thermal_monitor.yaml    # compila, flasha e mostra i log
uv run esphome logs multiprobe_thermal_monitor.yaml   # solo log
```

Il primo flash va fatto via USB. Gli aggiornamenti successivi possono passare via WiFi (OTA).

## Configurazione

I parametri principali sono nelle `substitutions` in testa a `multiprobe_thermal_monitor.yaml`:

| Parametro | Default | Significato |
|---|---|---|
| `sample_ms_normal` / `sample_ms_debug` | `3000` / `2000` | ms tra due letture; minimo 1000 (vedi sotto) |
| `publish_every_normal` / `publish_every_debug` | `10` / `1` | letture tra due pubblicazioni |
| `window_size` | `10` | letture su cui si calcola la mediana |
| `debug_pin` | `GPIO7` | pin del ponticello debug |
| `probe_1_pin` … `probe_3_pin` | `GPIO4` … `GPIO6` | pin dati delle sonde |
| `led_brightness_ok` / `led_brightness_alert` | `8%` / `40%` | luminosità del LED |

## Struttura del repository

| Percorso | Contenuto |
|---|---|
| `multiprobe_thermal_monitor.yaml` | firmware da flashare |
| `common/status_led.yaml` | modulo riutilizzabile per il LED di stato |
| `DOCS/` | schema di cablaggio e note hardware |
| `bench/` | varianti di test (vuota) |

## Scelte di progetto

- **MQTT invece dell'API nativa ESPHome**: le misure vanno su un broker già in uso. Senza `api:` i log via rete dalla dashboard ESPHome non sono disponibili.
- **Mediana invece di media**: l'errore della DS18B20 (±0.5 °C) è di calibrazione e la media non lo riduce. La mediana serve invece a ignorare le letture errate isolate.
- **Un bus per sonda**: niente indirizzi da configurare, una sonda si sostituisce senza riflashare e un guasto non blocca le altre due.
- **Almeno 1 s tra due letture**: a 12 bit una lettura della DS18B20 richiede 750 ms. Con un intervallo più corto ESPHome annulla ogni lettura prima che finisca e il sensore non pubblica nulla.

## Licenza

MIT, vedi [`LICENSE`](LICENSE).
