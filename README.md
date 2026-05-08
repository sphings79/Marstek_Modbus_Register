# 🔋 Marstek Venus D – Modbus Register Map (Community Research)

> **Sprache / Language:** [Deutsch](#deutsch) · [English](#english)

---

## Deutsch

### Was ist das hier?

Dieses Repository dokumentiert die Ergebnisse einer vollständigen **Modbus-TCP-Registeranalyse** des Heimspeichers **Marstek Venus D** (2 Packs, 16 Zellen je Pack, LFP-Chemie). Die Daten wurden durch systematische Scans aller Registerbereiche 30000–49999 gewonnen und mit der offiziellen Registertabelle des [ViperRNMC/marstek_venus_modbus](https://github.com/ViperRNMC/marstek_venus_modbus) Home-Assistant-Integration abgeglichen.

Ziel ist es, der Community eine **vollständige, praxiserprobte Referenz** bereitzustellen — mit interpretierten Werten, Scale-Faktoren, Quell-Spalten und Hinweisen auf Register, die in der offiziellen Dokumentation noch fehlen oder falsch zugeordnet sind.

---

### Gerät / Testsetup

| Eigenschaft | Wert |
|:------------|:-----|
| Modell | Marstek Venus D |
| Packs | 2 × 16 Zellen (LFP) |
| Kommunikation | RS485 → Modbus TCP via WiFi-Gateway |
| IP / Port / Unit | `192.168.181.154` / `502` / `1` |
| Gerätename (Reg. 31000) | `VNSD-0` |
| MAC (Reg. 30304) | `68:24:99:EE:F7:5C` |
| Komm.-Modul-FW (Reg. 30350) | `20240909_0159` |
| EMS-Version (Reg. 30200) | `147` |
| Scan-Tool | `modbus_scanner.py` (custom Python) |

---

### Wichtige Hinweise zur Venus-D-Architektur

#### Registermodell — Prioritäten

Die [ViperRNMC-Integration](https://github.com/ViperRNMC/marstek_venus_modbus) unterscheidet vier Geräte-Varianten mit unterschiedlichen Register-Layouts. Für den Venus D gilt folgende Lookup-Reihenfolge:

```
d → e_v3 → a      (niemals e_v12!)
```

Der **Ev12** hat ein vollständig anderes Registermodell und darf für den Venus D **nicht** als Referenz verwendet werden.

#### Batterie-Packs — Blockstruktur

Der Venus D mit 2 Packs verwendet eine **+100-Register-Verschiebung** zwischen den Packs:

| Pack | Block-Start | Beispiel: Zellspannungen |
|:----:|:-----------:|:------------------------:|
| Pack 1 | 34000 | 34018–34033 (16 Zellen) |
| Pack 2 | 34100 | 34118–34133 (16 Zellen) |
| Pack 3–7 | 34200–34600 | (Erweiterungen, im Scan leer) |

> Das A-Modell hat nur **13 Zellen** pro Pack (Register 34018–34030). Das D- und E-Modell haben **16 Zellen** (34018–34033). Der Ev12 hat ein abweichendes Layout.

#### RS485-Steuermodus

Schreibzugriff auf Steuerregister (42010–42021) ist nur möglich, wenn das Gerät aktiv im RS485-Steuermodus ist:

```
Register 42000 = 0x55AA (21930) → RS485-Steuerung AKTIV
Register 42000 = 0x55BB (21947) → RS485-Steuerung INAKTIV
```

> ⚠️ Steuerregister niemals schreiben, ohne den aktuellen Modus zu prüfen!

---

### Inhalt dieses Repositories

| Datei | Inhalt |
|:------|:-------|
| `scan_30000_matched.csv` | Alle bekannten Register aus dem 30000-Block mit interpretierten Werten |
| `scan_30000_unknown.csv` | Unbekannte/undokumentierte Register aus dem 30000-Block mit Best-Guess |
| `scan_40000_matched.csv` | Alle bekannten Register aus dem 40000-Block (Steuer-/Konfigurationsregister) |
| `scan_40000_unknown.csv` | Unbekannte Register aus dem 40000-Block inkl. String-Dekodierung |
| `scan_30000_analysis.md` | Vollständige Analyse 30000-Block (zweisprachig DE/EN) |
| `scan_40000_analysis.md` | Vollständige Analyse 40000-Block (zweisprachig DE/EN) |

---

### Highlights & neue Erkenntnisse

Die Scans ergaben eine Reihe von Registern, die in der aktuellen Integration noch nicht oder nicht vollständig dokumentiert sind:

#### 🆕 Fehlende Pack-Header-Register (30000-Block)

| Register | Interpretation | Wert im Scan |
|:--------:|:---------------|:------------:|
| 34000 | Pack-1-Gesamtspannung (×0.01 V) | 49.82 V |
| 34005 | Pack-1-Max-Zellspannung (×0.001 V) | 3.119 V |
| 34006 | Pack-1-Min-Zellspannung (×0.001 V) | 3.112 V |
| 34010 | Pack-1-Temperatur (×0.1 °C) | 11.6 °C |
| 34011 | Pack-1-Max-Zelltemperatur (×0.1 °C) | 24.6 °C |
| 34012–34016 | Pack-1-Zellentemperatursensoren 1–5 | 16.8–17.9 °C |
| 35011 | Min. Zelltemperatur (×0.1 °C) | 21.4 °C |

#### 🆕 IP-Adressen im Registerblock (30400–30403)

Die Netzwerkkonfiguration des Kommunikationsmoduls ist über Modbus auslesbar:

| Register | Inhalt | Wert |
|:--------:|:-------|:----:|
| 30400–30401 | Geräte-IP (uint16-Paar) | `192.168.181.154` |
| 30402–30403 | Gateway-IP (uint16-Paar) | `192.168.181.1` |

#### 🆕 WiFi-SSID im 40000-Block (41504–41509)

Die Register 41500–41515 enthalten ASCII-kodierte Strings des Kommunikationsmoduls. Register 41504–41509 dekodieren zur SSID `Hame147258` — vermutlich der eingebaute WiFi-Hotspot des Moduls.

#### 🔍 Register, die laut README nur für e_v12 gelten, auf dem D aber aktiv sind

| Register | Laut README | Tatsächlich auf D |
|:--------:|:------------|:-----------------:|
| 32100 | battery_voltage (e_v12 only) | 51.27 V (aktiv!) |
| 32104 | battery_soc (e_v12 only) | 11 % (aktiv!) |

---

### Scan-Methodik

Der Scan wurde mit einem eigenen Python-Skript durchgeführt, das Register einzeln abfragt, mit Heartbeat-Register und adaptivem Backoff:

```bash
python modbus_scanner.py \
  --ip 192.168.181.154 \
  --port 502 \
  --start 30000 \
  --count 10000 \
  -v \
  --heartbeat-reg 41000 \
  --csv scan_30000.csv
```

| Parameter | Wert |
|:----------|:-----|
| Delay | 0.3 s |
| Timeout | 3.0 s |
| Pause | alle 50 Register → 3.0 s |
| Backoff | max. 5 Versuche → 30 s Cooldown |
| Modus | Einmaliger Scan |

---

### Disclaimer

Die Inhalte dieses Repositories basieren auf **Reverse Engineering** durch Modbus-TCP-Scans eines einzelnen Geräts (Venus D, 2 Packs). Alle Interpretationen unbekannter Register sind **Vermutungen** und nicht offiziell bestätigt. Schreibzugriffe auf Steuerregister erfolgen auf eigenes Risiko. Dieses Repository steht in keiner Verbindung zu Marstek oder dem ViperRNMC-Projekt, referenziert es aber als Ausgangsbasis.

---

## English

### What is this?

This repository documents the results of a complete **Modbus TCP register analysis** of the **Marstek Venus D** home battery system (2 packs, 16 cells per pack, LFP chemistry). Data was gathered through systematic scans of all register ranges 30000–49999 and cross-referenced against the official register table of the [ViperRNMC/marstek_venus_modbus](https://github.com/ViperRNMC/marstek_venus_modbus) Home Assistant integration.

The goal is to provide the community with a **complete, field-tested reference** — including interpreted values, scale factors, source columns, and notes on registers that are missing from or incorrectly documented in the official documentation.

---

### Device / Test Setup

| Property | Value |
|:---------|:------|
| Model | Marstek Venus D |
| Packs | 2 × 16 cells (LFP) |
| Communication | RS485 → Modbus TCP via WiFi gateway |
| IP / Port / Unit | `192.168.181.154` / `502` / `1` |
| Device name (Reg. 31000) | `VNSD-0` |
| MAC (Reg. 30304) | `68:24:99:EE:F7:5C` |
| Comm module FW (Reg. 30350) | `20240909_0159` |
| EMS version (Reg. 30200) | `147` |
| Scan tool | `modbus_scanner.py` (custom Python) |

---

### Key Notes on Venus D Architecture

#### Register model — lookup priority

The [ViperRNMC integration](https://github.com/ViperRNMC/marstek_venus_modbus) distinguishes four device variants with different register layouts. For the Venus D, the lookup order is:

```
d → e_v3 → a      (never e_v12!)
```

The **Ev12** has a completely different register model and must **not** be used as a reference for the Venus D.

#### Battery packs — block structure

The Venus D with 2 packs uses a **+100 register offset** between packs:

| Pack | Block start | Example: cell voltages |
|:----:|:-----------:|:----------------------:|
| Pack 1 | 34000 | 34018–34033 (16 cells) |
| Pack 2 | 34100 | 34118–34133 (16 cells) |
| Pack 3–7 | 34200–34600 | (expansion slots, empty in scan) |

> The A model has only **13 cells** per pack (registers 34018–34030). The D and E models have **16 cells** (34018–34033). The Ev12 has a different layout.

#### RS485 control mode

Write access to control registers (42010–42021) is only possible when the device is actively in RS485 control mode:

```
Register 42000 = 0x55AA (21930) → RS485 control ACTIVE
Register 42000 = 0x55BB (21947) → RS485 control INACTIVE
```

> ⚠️ Never write to control registers without checking the current mode first!

---

### Repository Contents

| File | Contents |
|:-----|:---------|
| `scan_30000_matched.csv` | All known registers from the 30000 block with interpreted values |
| `scan_30000_unknown.csv` | Unknown/undocumented registers from the 30000 block with best-guess interpretations |
| `scan_40000_matched.csv` | All known registers from the 40000 block (control/config registers) |
| `scan_40000_unknown.csv` | Unknown registers from the 40000 block incl. string decoding |
| `scan_30000_analysis.md` | Full analysis of the 30000 block (bilingual DE/EN) |
| `scan_40000_analysis.md` | Full analysis of the 40000 block (bilingual DE/EN) |

---

### Highlights & New Findings

The scans revealed a number of registers not yet fully documented in the current integration:

#### 🆕 Missing pack header registers (30000 block)

| Register | Interpretation | Value in scan |
|:--------:|:---------------|:-------------:|
| 34000 | Pack 1 total voltage (×0.01 V) | 49.82 V |
| 34005 | Pack 1 max cell voltage (×0.001 V) | 3.119 V |
| 34006 | Pack 1 min cell voltage (×0.001 V) | 3.112 V |
| 34010 | Pack 1 temperature (×0.1 °C) | 11.6 °C |
| 34011 | Pack 1 max cell temperature (×0.1 °C) | 24.6 °C |
| 34012–34016 | Pack 1 cell temperature sensors 1–5 | 16.8–17.9 °C |
| 35011 | Min cell temperature (×0.1 °C) | 21.4 °C |

#### 🆕 IP addresses in register block (30400–30403)

The network configuration of the communication module is readable via Modbus:

| Register | Content | Value |
|:--------:|:--------|:-----:|
| 30400–30401 | Device IP (uint16 pair) | `192.168.181.154` |
| 30402–30403 | Gateway IP (uint16 pair) | `192.168.181.1` |

#### 🆕 WiFi SSID in 40000 block (41504–41509)

Registers 41500–41515 contain ASCII-encoded strings from the communication module. Registers 41504–41509 decode to the SSID `Hame147258` — likely the built-in WiFi hotspot of the module.

#### 🔍 Registers documented as e_v12-only that are active on the D model

| Register | Per README | Actually on D |
|:--------:|:-----------|:-------------:|
| 32100 | battery_voltage (e_v12 only) | 51.27 V (active!) |
| 32104 | battery_soc (e_v12 only) | 11 % (active!) |

---

### Disclaimer

The contents of this repository are based on **reverse engineering** via Modbus TCP scans of a single device (Venus D, 2 packs). All interpretations of unknown registers are **best-effort guesses** and are not officially confirmed. Write access to control registers is at your own risk. This repository has no affiliation with Marstek or the ViperRNMC project, though it references it as a baseline.

---

## Contributing

If you own a Marstek Venus A, D, or E and have run similar scans, contributions are very welcome! Please open an issue or pull request with:
- Your device model and pack count
- Scan output (CSV or raw text)
- Any confirmed register interpretations

The more devices we cover, the better the register map becomes for everyone.
