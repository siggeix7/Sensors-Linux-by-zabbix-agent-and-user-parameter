# Zabbix – Linux `lm-sensors` template

*(Zabbix agent & user parameter, with LLD and temperature triggers)*

## 1. Overview

Questo repository contiene due template Zabbix per monitorare le temperature di un host Linux usando il comando `sensors` di **lm-sensors** e un **UserParameter** dell’agente.

Tutto si basa su un singolo item che esegue `sensors` e restituisce l’output grezzo; da lì, tramite **Low Level Discovery (LLD)** e **dependent item**, il template crea automaticamente:

* un item per ogni temperatura trovata;
* un grafico per ogni temperatura;
* trigger a 3 livelli di severità configurabili tramite macro.

Funziona con **Zabbix 7.4** (agent classico o agent2).

---

## 2. Template inclusi

Nel file YAML trovi due template:

1. **`Sensors Linux by zabbix agent active and user parameters`**

   * Usa un item di tipo **Zabbix agent (active)**.
   * Adatto se preferisci gli **active checks** (nessuna connessione in ingresso al client).

2. **`Sensors Linux by zabbix agent and user parameter`**

   * Usa un item di tipo **Zabbix agent** (passive).
   * Adatto per installazioni classiche con agent passivo. ([zabbix.com][1])

Entrambi i template:

* usano la chiave **`sensors.raw`** come item principale;
* definiscono la discovery rule `Sensors Linux: temperature discovery`;
* creano item dependent `sensors.linux.temp[{#CHIP},{#LABEL}]`;
* includono 3 macro di soglia:

  * `{$TEMP.WARN}` = 60
  * `{$TEMP.HIGH}` = 70
  * `{$TEMP.CRIT}` = 80
* includono un **graph prototype** `Temperature [{#CHIP} - {#LABEL}]`.

---

## 3. Requisiti

### 3.1. Software

* **Zabbix Server / Proxy**: 7.4.x
* **Zabbix agent** sull’host:

  * `zabbix_agentd` (agent classico) **oppure**
  * `zabbix_agent2` (agent 2) ([zabbix.com][1])
* **lm-sensors** installato e funzionante sull’host.

### 3.2. Sistema operativo

Qualsiasi Linux supportato da Zabbix agent, ad esempio:

* Debian / Ubuntu / Proxmox
* RHEL / Rocky / Alma / CentOS
* openSUSE / SLES
* Arch Linux / Manjaro
* Alpine Linux
* NAS basati su Debian/Ubuntu.

---

## 4. Installare `lm-sensors`

Di seguito alcuni esempi di installazione per distro comuni.

> **Nota:** comandi da eseguire come `root` o con `sudo`.

### 4.1. Debian / Ubuntu / Proxmox

```bash
apt update
apt install lm-sensors
```

### 4.2. RHEL / Rocky / Alma / CentOS

```bash
# Sulle versioni recenti (8/9):
dnf install lm_sensors
# oppure:
yum install lm_sensors
```

### 4.3. openSUSE / SLES

```bash
zypper install lm_sensors
```

### 4.4. Arch Linux / Manjaro

```bash
pacman -S lm_sensors
```

### 4.5. Alpine Linux

```bash
apk add lm-sensors
```

---

## 5. Configurare `lm-sensors`

Dopo l’installazione, rileva i sensori hardware:

```bash
sensors-detect
```

* Rispondi alle domande (di solito puoi confermare i default).
* Al termine, se ti propone di aggiungere i moduli a `/etc/modules-load.d`, conferma.

Verifica che `sensors` funzioni:

```bash
sensors
```

Devi vedere un output simile a:

```output sensors
coretemp-isa-0000
Adapter: ISA adapter
Package id 0:  +51.0°C
Core 0:        +50.0°C

nvme-pci-0100
Adapter: PCI adapter
Composite:    +32.9°C
...
```

Se `sensors` non mostra nulla o mostra errori, risolvi prima qui: il template dipende completamente da questo comando.

---

## 6. Configurare Zabbix agent (UserParameter)

### 6.1. Trovare il path di `sensors`

```bash
which sensors
```

Di solito è `/usr/bin/sensors`. Se il percorso è diverso, aggiornalo nei comandi sotto.

---

### 6.2. Agent classico (`zabbix_agentd`)

Percorsi tipici:

* Config principale: `/etc/zabbix/zabbix_agentd.conf`
* Directory dei file `.d`: `/etc/zabbix/zabbix_agentd.d/`

Crea un file, ad esempio: `/etc/zabbix/zabbix_agentd.d/sensors.conf`

```bash
sudo nano /etc/zabbix/zabbix_agentd.d/sensors.conf
```

Contenuto:

```zabbix conf
### UserParameter per output grezzo di "sensors"
UserParameter=sensors.raw,/usr/bin/sensors
```

Opzionale: se usi `UserParameterDir`, puoi mettere solo `sensors`:

```zabbix conf
UserParameterDir=/usr/bin
UserParameter=sensors.raw,sensors
```

---

### 6.3. Agent 2 (`zabbix_agent2`)

Percorsi tipici:

* Config principale: `/etc/zabbix/zabbix_agent2.conf`
* Directory dei file `.d`: `/etc/zabbix/zabbix_agent2.d/`

Stesso concetto:

```bash
sudo nano /etc/zabbix/zabbix_agent2.d/sensors.conf
```

```zabbix conf
UserParameter=sensors.raw,/usr/bin/sensors
```

---

### 6.4. Abilitare UserParameter / AllowKey

In Zabbix 7.x puoi restringere le chiavi con `AllowKey` / `DenyKey`. ([zabbix.com][2])

Se usi queste regole, assicurati di **permettere** la chiave `sensors.raw`. Esempi:

```zabbix conf
# Permette solo sensors.raw (e magari altre chiavi)
AllowKey=sensors.raw
# oppure
AllowKey=sensors.*
DenyKey=*
```

Nella maggior parte delle installazioni, se non hai toccato `AllowKey`/`DenyKey`, non serve fare nulla.

> **Nota su `UnsafeUserParameters`**
> Il nostro comando è semplice (`/usr/bin/sensors`, nessun parametro con caratteri speciali), quindi **non è necessario** impostare `UnsafeUserParameters=1`. ([zabbix.com][3])

---

### 6.5. Riavviare l’agente

Dopo aver aggiunto il file:

```bash
# agent classico
sudo systemctl restart zabbix-agent

# agent2
sudo systemctl restart zabbix-agent2
```

Oppure, se vuoi evitare il restart completo, puoi usare:

```bash
sudo zabbix_agentd -R userparameter_reload
```

dove supportato. ([zabbix.com][4])

---

## 7. Permessi: l’utente `zabbix` deve poter eseguire `sensors`

Per default, l’agente gira come utente `zabbix`. ([zabbix.com][3])

Verifica:

```bash
sudo -u zabbix /usr/bin/sensors
```

* Se ottieni un output normale → **perfetto**.
* Se vedi errori tipo “Permission denied” su `/sys` o `/dev`:

### Possibili soluzioni (in ordine di “pulizia”)

1. **Regole udev**
   Alcuni sistemi limitano l’accesso a `/dev/hwmon*` o simili a un certo gruppo.

   * Controlla `ls -l /dev/hwmon*` e imposta regole udev per aggiungere il gruppo dell’utente `zabbix`.

2. **Far girare l’agente come root (non raccomandato)**

   In `zabbix_agentd.conf` / `zabbix_agent2.conf`:

   ```zabbix conf
   AllowRoot=1
   User=root
   ```

   Questo dà più privilegi a tutto l’agente, quindi valuta bene la sicurezza.

3. **Usare sudo (solo per sensors)**

   * Aggiungi in `/etc/sudoers.d/zabbix` (con `visudo -f /etc/sudoers.d/zabbix`):

     ```text
     zabbix ALL=(root) NOPASSWD:/usr/bin/sensors
     ```

   * Modifica il tuo `UserParameter`:

     ```zabbix conf
     UserParameter=sensors.raw,sudo /usr/bin/sensors
     ```

   Attenzione: anche questo aumenta un po’ la superficie di attacco, usalo solo se necessario.

---

## 8. Importare il template in Zabbix

1. Vai in **Data collection → Templates**.
2. Click su **Import**.
3. Seleziona il file `Sensors_Linux_by_zabbix_agent_and_user_parameter.yaml`.
4. Conferma le opzioni di import (in genere default vanno bene).
5. Verifica che compaiano i due template:

   * `Sensors Linux by zabbix agent active and user parameters`
   * `Sensors Linux by zabbix agent and user parameter`

---

## 9. Collegare il template all’host

1. Vai in **Data collection → Hosts**.
2. Apri l’host Linux/NAS che vuoi monitorare.
3. Tab **Templates** → **Link new template**.
4. Scegli:

   * **Versione active** se l’host usa Zabbix agent (active).
   * **Versione passive** se usi Zabbix agent classico.
5. Salva.

Dopo qualche minuto dovresti vedere:

* item `Sensors Linux: RAW output` (Text);
* discovery rule `Sensors Linux: temperature discovery`;
* item generati del tipo `Sensors Linux: Temperature [chip - label]`.

---

## 10. Cosa fa il template

### 10.1. Item principale

* `Sensors Linux: RAW output`

  * Key: `sensors.raw`
  * Type: Zabbix agent / Zabbix agent (active)
  * Delay: 20s (modificabile)
  * Preprocessing: `Discard unchanged with heartbeat (60s)`

Questo item esegue il comando `sensors` e salva l’output testuale.

### 10.2. Discovery & item dependent

La discovery rule `Sensors Linux: temperature discovery`:

* legge l’output di `sensors`;
* riconosce ogni sezione (chip) e riga con temperatura (es. `Core 0`, `Composite`, `temp1`);
* crea LLD con:

  * `{#CHIP}` (es. `coretemp-isa-0000`)
  * `{#LABEL}` (es. `Core 0`)

Da qui, viene creato un item dependent:

* Key: `sensors.linux.temp[{#CHIP},{#LABEL}]`
* Tipo: Dependent item, float, unità `°C`
* Preprocessing: JavaScript che estrae il valore numerico della temperatura dalla riga corrispondente.

---

## 11. Soglie & trigger (macro)

Il template definisce 3 macro di soglia:

* `{$TEMP.WARN}` – soglia di warning (default: 60°C)
* `{$TEMP.HIGH}` – soglia alta (default: 70°C)
* `{$TEMP.CRIT}` – soglia critica (default: 80°C)

Puoi personalizzarle a livello di:

* template (per tutti gli host)
* host (override per singolo server/NAS).

I trigger prototype (che puoi vedere/modificare nel template) usano queste macro in combinazione con funzioni tipo `min(1m)` per evitare falsi positivi dovuti a spike singoli.

---

## 12. Grafici

Ogni temperatura ha un **graph prototype**:

* Nome: `Temperature [{#CHIP} - {#LABEL}]`
* Contiene l’item `sensors.linux.temp[{#CHIP},{#LABEL}]`.

Inoltre, puoi usare una **Dashboard** Zabbix 7.4 con widget **Graph** e **Item pattern** per avere un unico grafico con **tutte le temperature**:

* Host pattern: ad es. il nome del tuo host (`nas-*` oppure `*`).
* Item pattern:
  `*sensors.linux.temp*`

Questo ti permette di tracciare fino a 50 item per grafico, automatizzando il monitoraggio di tutte le temperature discoverate. ([zabbix.com][5])

---

## 13. Troubleshooting

### 13.1. `Sensors Linux: RAW output` è “Unsupported”

* Verifica che l’agent veda la chiave:

  ```bash
  zabbix_agentd -t sensors.raw
  # oppure
  zabbix_agent2 -t sensors.raw
  ```

* Se vedi: `Unsupported item key.`:

  * controlla `AllowKey` / `DenyKey` nel config;
  * controlla di aver scritto correttamente `UserParameter=sensors.raw,/usr/bin/sensors`.

### 13.2. Nessun item discoverato

* Esegui `sensors` manualmente sul server: se non ci sono temperature, il template non potrà creare nulla.
* Verifica nel log dell’agente (`/var/log/zabbix/zabbix_agentd.log` o `zabbix_agent2.log`) eventuali errori di permesso.

### 13.3. Alcune temperature mancano

* Il parser prende solo le righe che contengono effettivamente qualcosa come `+XX.X°C`.
  Valori solo in altre unità o senza `°C` non vengono considerati; eventuali adattamenti richiedono modifiche allo script JS nel template.

---

## 14. Licenza / Uso

* Il template è pensato per essere **usato liberamente**, modificato e condiviso.
* Per l’utilizzo di Zabbix stesso, fai riferimento alla licenza **AGPLv3** e alla documentazione ufficiale. ([zabbix.com][6])

---
