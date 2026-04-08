# Technische Dokumentation - Anker Solix API

> **Version:** 3.4.0 | **Python:** >= 3.12 | **Lizenz:** MIT

## Inhaltsverzeichnis

1. [Projektuebersicht und Architektur](#1-projektuebersicht-und-architektur)
2. [Authentifizierung und Session-Management](#2-authentifizierung-und-session-management)
3. [Cloud API Kommunikation](#3-cloud-api-kommunikation)
4. [Datenmodell und Typsystem](#4-datenmodell-und-typsystem)
5. [Cache-Management](#5-cache-management)
6. [Data Polling](#6-data-polling)
7. [MQTT-Kommunikationsschicht](#7-mqtt-kommunikationsschicht)
8. [MQTT Device Control System](#8-mqtt-device-control-system)
9. [Feature-Module](#9-feature-module)
10. [Fehlerbehandlung](#10-fehlerbehandlung)
11. [Hilfsmodule](#11-hilfsmodule)
12. [Ausfuehrbare Skripte](#12-ausfuehrbare-skripte)

---

## 1. Projektuebersicht und Architektur

### 1.1 Zweck

Diese Bibliothek ermoeglicht die Kommunikation mit Anker Solix Power-Geraeten (Solarbank, Wechselrichter, Smart Meter, Portable Power Stations, EV-Charger u.a.) ueber reverse-engineerte Cloud- und MQTT-APIs. Die gesamte Bibliothek ist **async-first** und basiert auf `aiohttp` fuer HTTP- und `paho-mqtt` fuer MQTT-Kommunikation.

### 1.2 Moduluebersicht

Das `api/`-Verzeichnis enthaelt 24 Module mit insgesamt ca. 30.000 Zeilen Code:

| Modul | Zeilen | Beschreibung |
|---|---|---|
| `api.py` | 2.356 | Haupt-API-Klasse `AnkerSolixApi` |
| `apibase.py` | 2.275 | Basisklasse `AnkerSolixBaseApi` mit Cache-Management |
| `session.py` | 1.090 | HTTP-Client-Session und Authentifizierung |
| `apitypes.py` | 1.860 | Typen, Enums, Dataclasses, API-Endpoints |
| `errors.py` | 134 | Exceptions und Fehlercode-Mapping |
| `energy.py` | 1.484 | Energieanalyse und Statistiken |
| `schedule.py` | 2.958 | Geraete-Zeitplanung |
| `vehicle.py` | 846 | Fahrzeugverwaltung (EV-Charger) |
| `export.py` | 2.190 | JSON-Datenexport |
| `hesapi.py` | 1.727 | HES (Home Energy System) Endpoints |
| `powerpanel.py` | 1.452 | Power Panel Endpoints |
| `poller.py` | 1.571 | Daten-Polling fuer Cache-Aktualisierung |
| `helpers.py` | 131 | Hilfsfunktionen |
| `mqtt.py` | 1.088 | MQTT-Session-Management |
| `mqtt_device.py` | 849 | Basis-MQTT-Geraetesteuerung |
| `mqtt_solarbank.py` | 116 | Solarbank-spezifische MQTT-Steuerung |
| `mqtt_charger.py` | 114 | Charger-spezifische MQTT-Steuerung |
| `mqtt_pps.py` | 555 | PPS-spezifische MQTT-Steuerung |
| `mqtt_various.py` | 80 | Sonstige MQTT-Geraete (SmartPlug) |
| `mqtt_factory.py` | 66 | Factory fuer MQTT-Geraeteobjekte |
| `mqttmap.py` | 4.201 | MQTT-Feld-Konvertierungsmappings |
| `mqttcmdmap.py` | 1.298 | MQTT-Command-Mappings |
| `mqtttypes.py` | 1.492 | MQTT-Datentypen und Strukturen |

### 1.3 Klassenhierarchie

```
AnkerSolixClientSession          (session.py)
    |                                HTTP-Client, Authentifizierung, Request-Handling
    |
AnkerSolixBaseApi                (apibase.py)
    |                                Cache-Management, MQTT-Session, Callbacks
    |
    +-- AnkerSolixApi            (api.py)
    |       Haupt-API fuer Balkon-Kraftwerke (power_service Endpoints)
    |       Importiert Mixin-Module: energy, schedule, vehicle
    |
    +-- AnkerSolixHesApi         (hesapi.py)
    |       API fuer Home Energy Systeme (X1, charging_hes_svc Endpoints)
    |
    +-- AnkerSolixPowerpanelApi  (powerpanel.py)
            API fuer Power Panels (charging_energy_service Endpoints)


AnkerSolixMqttSession            (mqtt.py)
    |                                MQTT-Client, Topic-Management, Nachrichtenverarbeitung
    |
SolixMqttDevice                  (mqtt_device.py)
    |                                Basis-Geraeteklasse mit Command-Validierung
    |
    +-- SolixMqttDeviceSolarbank (mqtt_solarbank.py)
    +-- SolixMqttDeviceCharger   (mqtt_charger.py)
    +-- SolixMqttDevicePps       (mqtt_pps.py)
    +-- SolixMqttDeviceVarious   (mqtt_various.py)

SolixMqttDeviceFactory           (mqtt_factory.py)
                                     Erzeugt die richtige Geraeteklasse nach Geraetetyp
```

### 1.4 Mixin-Pattern

Die `AnkerSolixApi`-Klasse verwendet ein Mixin-Pattern, um Feature-Module einzubinden. Methoden aus `energy.py`, `schedule.py` und `vehicle.py` werden direkt in den Klassenkoerper importiert:

```python
class AnkerSolixApi(AnkerSolixBaseApi):
    from .energy import energy_daily, energy_analysis, ...
    from .schedule import get_device_load, set_home_load, ...
    from .vehicle import get_vehicle_list, create_vehicle, ...
```

Jedes Mixin-Modul definiert `async`-Funktionen, die `self: AnkerSolixApi` als ersten Parameter nehmen. Dadurch koennen die Methoden direkt auf den API-Cache (`self.sites`, `self.devices`) und die Session (`self.apisession`) zugreifen.

### 1.5 Abhaengigkeiten

| Paket | Version | Verwendung |
|---|---|---|
| `aiohttp` | >= 3.10.11 | Asynchrone HTTP-Kommunikation |
| `aiofiles` | >= 23.2.0 | Asynchrone Datei-I/O |
| `cryptography` | >= 3.4.8 | ECDH-Schluesselaustausch, AES-Verschluesselung |
| `paho-mqtt` | >= 2.1.0 | MQTT-Protokoll-Client |
| `python-dotenv` | >= 1.2.1 | `.env`-Datei-Unterstuetzung (optional, fuer Skripte) |

---

## 2. Authentifizierung und Session-Management

**Datei:** `api/session.py` — Klasse `AnkerSolixClientSession`

### 2.1 Ueberblick

Die Authentifizierung gegenueber den Anker-Servern basiert auf einem asymmetrischen Schluesselaustausch (ECDH), gefolgt von einer symmetrischen Passwortverschluesselung (AES-256-CBC). Der daraus resultierende Token wird fuer alle nachfolgenden API-Anfragen verwendet.

### 2.2 ECDH Key Exchange

Bei der Initialisierung der Session wird ein **Elliptic Curve Diffie-Hellman (ECDH)** Schluesselpaar generiert:

1. **Kurve:** NIST P-256 (SECP256R1 / prime256v1)
2. **Privater Schluessel:** Zufaellig generiert ueber `ec.generate_private_key()`
3. **Oeffentlicher Schluessel:** Abgeleitet vom privaten Schluessel
4. **Anker-Server Public Key:** Fest kodiert als unkomprimierter Punkt (65 Bytes, Format `04` + 32 Byte X + 32 Byte Y)
5. **Shared Secret:** Berechnet via `private_key.exchange(ec.ECDH(), server_public_key)`

Der oeffentliche Schluessel des Anker-Servers ist fuer EU- und COM-Server identisch:
```
04c5c00c4f8d1197cc7c3167c52bf7acb054d722f0ef08dcd7e0883236e0d72a3868d9750cb47fa4619248f3d83f0f662671dadc6e2d31c2f41db0161651c7c076
```

### 2.3 Passwortverschluesselung

Das Passwort wird vor dem Login-Request verschluesselt:

1. Das ECDH Shared Secret dient als Schluessel
2. **Algorithmus:** AES-256-CBC
3. **Seed/IV:** 16 zufaellige Bytes
4. **Padding:** PKCS7 (128-Bit Blockgroesse)
5. Das verschluesselte Passwort wird Base64-kodiert uebertragen

### 2.4 Login-Ablauf

```
Client                                    Anker Server
  |                                           |
  |-- POST passport/login ------------------>|
  |   {                                       |
  |     "ab": "DE",                           |
  |     "client_secret_info": {               |
  |       "public_key": "<client_pub_hex>"    |
  |     },                                    |
  |     "email": "user@example.com",          |
  |     "password": "<aes_encrypted_b64>",    |
  |     "time_zone": 3600000,                 |
  |     "transaction": "<unix_ts_ms>"         |
  |   }                                       |
  |                                           |
  |<-- Response ----------------------------  |
  |   {                                       |
  |     "auth_token": "...",                  |
  |     "user_id": "...",                     |
  |     "token_expires_at": <unix_ts>,        |
  |     "nick_name": "..."                    |
  |   }                                       |
```

### 2.5 Token-Management

- **auth_token:** Wird als `x-auth-token` Header in allen API-Anfragen mitgesendet
- **gtoken:** MD5-Hash der `user_id` aus der Login-Antwort, wird als `gtoken` Header gesendet
- **Token-Ablauf:** 7 Tage — wird automatisch erneuert wenn < 60 Sekunden verbleiben
- **Auth-Caching:** Login-Antwort wird in `api/authcache/{email}.json` gespeichert
- **Cache-Refresh:** Bei jeder Request-Ausfuehrung wird geprueft, ob die Cache-Datei extern aktualisiert wurde (z.B. von einer parallelen API-Instanz). Falls ja, werden die Credentials automatisch aktualisiert.

### 2.6 Payload-Verschluesselung (experimentell)

Die Klasse `AnkerEncryptionHandler` (ebenfalls in `session.py`) implementiert eine zusaetzliche Payload-Verschluesselung fuer API-Requests, die von neueren App-Versionen verwendet wird:

1. **Key Exchange:** POST an `openapi/oauth/key/exchange` zum Austausch eines neuen ECDH-Schluessels
2. **HKDF Key Derivation:** Ableitung von Verschluesselungs- und MAC-Schluessel aus dem Shared Secret mittels HKDF (SHA-256)
3. **Request-Verschluesselung:** AES-256-CBC Verschluesselung des JSON-Payloads
4. **Signatur:** SHA-256 HMAC ueber den verschluesselten Payload
5. **Zusaetzliche Header:** `x-encryption-info`, `x-key-ident`, `x-request-once`, `x-request-ts`, `x-signature`

---

## 3. Cloud API Kommunikation

**Dateien:** `api/session.py`, `api/apitypes.py`

### 3.1 Server-Architektur

Die API nutzt zwei regionale Server:

| Region | Server-URL |
|---|---|
| EU | `https://ankerpower-api-eu.anker.com` |
| COM | `https://ankerpower-api.anker.com` |

Die Zuordnung erfolgt ueber den `countryId`-Parameter. Laender wie DE, FR, AT, CH, etc. werden dem EU-Server zugeordnet, waehrend US, CA, AU, JP, etc. den COM-Server nutzen. Bei unbekanntem Laendercode wird standardmaessig der EU-Server verwendet.

### 3.2 Request-Verarbeitung

Alle API-Aufrufe laufen ueber die zentrale Methode `AnkerSolixClientSession.request()`:

1. **Token-Validierung:** Pruefung, ob das Token noch gueltig ist (> 60s Restlaufzeit)
2. **Automatische Authentifizierung:** Falls nicht eingeloggt oder Cache aktualisiert
3. **Header-Generierung:** Standard-Header plus optionale Verschluesselungs-Header
4. **Request-Delay:** Konfigurierbarer Mindestabstand zwischen Requests (Standard: 0,3s)
5. **Endpoint-Throttling:** Optionales Limit pro Endpoint pro Minute
6. **Retry-Logik:** Automatischer Retry bei Authentifizierungsfehlern (Token abgelaufen/gekickt)

### 3.3 Standard-Header

```json
{
  "content-type": "application/json",
  "model-type": "DESKTOP",
  "app-name": "anker_power",
  "os-type": "android",
  "country": "<countryId>",
  "timezone": "GMT+01:00",
  "gtoken": "<md5_user_id>",
  "x-auth-token": "<auth_token>"
}
```

### 3.4 Rate-Limiting

- **Request Delay:** Konfigurierbar von 0,0s bis 10,0s (Standard: 0,3s). Wird ueber `requestDelay()` gesetzt.
- **Request Timeout:** Konfigurierbar von 10s bis 60s (Standard: 30s)
- **Endpoint-Limit:** Maximale Anfragen pro Endpoint pro Minute (Standard: 10). Bei Ueberschreitung wird der naechste Request fuer bis zu 65 Sekunden verzoegert.
- **RequestCounter:** Hilfsklasse (`helpers.py`), die Timestamps aller Requests speichert und Statistiken fuer letzte Minute/Stunde liefert.

### 3.5 API-Endpoint-Kategorien

Die Bibliothek implementiert ueber **79 Endpoints** aus verschiedenen Service-Kategorien:

| Service | Prefix | Beschreibung |
|---|---|---|
| Power Service | `power_service/v1/` | Haupt-Service fuer Balkon-Kraftwerke |
| App Service | `app/` | Geraeteverwaltung, OTA, MQTT-Info |
| Charging PV Service | `charging_pv_svc/` | Standalone-Wechselrichter |
| Charging Energy Service | `charging_energy_service/` | Power Panels |
| Charging HES Service | `charging_hes_svc/` | Home Energy Systeme (X1) |
| Mini Power Service | `mini_power/v1/` | Prime Charger Geraete |

Wichtige Endpoint-Gruppen im `power_service`:

- **Site-Management:** `get_site_homepage`, `get_site_list`, `get_site_detail`, `get_scen_info`
- **Geraete-Einstellungen:** `get_site_device_param`, `set_site_device_param`, `get_device_attrs`, `set_device_attrs`
- **Energie-Daten:** `energy_analysis`, `get_home_load_chart`, `get_device_income`
- **Firmware:** `get_ota_info`, `get_ota_update`, `get_auto_upgrade`
- **Dynamische Preise:** `check_available`, `support_option`, `price_detail`
- **Fahrzeuge:** `get_vehicle_list`, `add_vehicle`, `update_vehicle`, `set_charging_vehicle`

---

## 4. Datenmodell und Typsystem

**Datei:** `api/apitypes.py`

### 4.1 Enumerationen

Die zentrale Typendefinition umfasst zahlreiche Enums:

**`SolixDeviceType`** — Geraetkategorien:
```
SOLARBANK, COMBINER_BOX, INVERTER, SMARTMETER, SMARTPLUG, PPS,
POWERPANEL, POWERCOOLER, HES, HOME_BACKUP, GENERATOR, CHARGER,
POWERBANK, EV_CHARGER, VEHICLE, SOLARBANK_PPS, VIRTUAL, SYSTEM, ACCOUNT
```

**`SolixParmType`** — Parameter-Typen fuer Schedule-Abfragen:
```
SOLARBANK_SCHEDULE = "4"
SOLARBANK_2_SCHEDULE = "6"
SOLARBANK_TARIFF_SCHEDULE = "12"
SOLARBANK_3_SCHEDULE = "26"
```

Weitere wichtige Enums: `SolarbankStatus`, `SolarbankUsageMode`, `SolixDeviceStatus`, `SolixGridStatus`, `SolarbankPowerMode`, `SolixNetworkStatus`.

### 4.2 Dataclasses

**`SolixDefaults`** — Zentrale Standardwerte:
- Leistungs-Presets: `PRESET_MIN` (100W) bis `PRESET_MAX` (800W)
- Lade-Prioritaet: `CHARGE_PRIORITY_MIN` (0%) bis `CHARGE_PRIORITY_MAX` (100%), Standard 80%
- Request-Konfiguration: `REQUEST_DELAY_DEF` (0.3s), `REQUEST_TIMEOUT_DEF` (30s), `ENDPOINT_LIMIT_DEF` (10)

**`SolarbankTimeslot`** / **`Solarbank2Timeslot`** — Zeitslot-Definitionen fuer Geraete-Schedules mit Start-/Endzeit, Leistung, Ladeprioritat und Betriebsmodus.

### 4.3 API-Endpoint-Dictionaries

- **`API_ENDPOINTS`** — Dictionary mit symbolischen Namen zu REST-Pfaden (z.B. `"homepage"` -> `"power_service/v1/site/get_site_homepage"`)
- **`API_CHARGING_ENDPOINTS`** — Endpoints fuer `charging_energy_service`
- **`API_HES_SVC_ENDPOINTS`** — Endpoints fuer `charging_hes_svc`
- **`API_FILEPREFIXES`** — Zuordnung von Endpoint-Namen zu Datei-Praefixen fuer den JSON-Export

### 4.4 Geraeteuebersicht

Die Bibliothek unterstuetzt ueber 60 verschiedene Geraetemodelle, darunter:

| Modell-Praefix | Geraetetyp |
|---|---|
| `A17C0`-`A17C5` | Solarbanks (E1600, E2700) |
| `A17X7`-`A17X8` | Smart Meter, Smart Plug |
| `A17E0`-`A17E1` | Micro-Wechselrichter (MI60, MI80) |
| `A17B1` | Home Power Panel |
| `A176x`-`A179x` | Portable Power Stations (C1000, F2000, F3800 etc.) |
| `A17Ax` | Powered Cooler (Everfrost) |
| `A2345` | Prime Charger |
| `A7320` | Generator |
| `AE100` | Power Dock |

---

## 5. Cache-Management

**Datei:** `api/apibase.py` — Klasse `AnkerSolixBaseApi`

### 5.1 Cache-Struktur

Die API verwaltet drei zentrale Cache-Dictionaries, die von allen Abfragen befuellt und aktualisiert werden:

```python
self.account: dict[str, dict]   # Konto-Informationen (Schluessel: Email)
self.sites: dict[str, dict]     # Standort-/System-Daten (Schluessel: site_id)
self.devices: dict[str, dict]   # Geraete-Daten (Schluessel: device_sn)
```

**Account-Cache:** Benutzer-Profildaten, Token-Status, Fahrzeugliste, MQTT-Verbindungsstatus

**Sites-Cache:** Fuer jeden Standort: `site_info`, `site_details`, Energiedaten, Schedule-Informationen, Topologie, Preisdaten, dynamische Preise, Solar-Forecast

**Devices-Cache:** Fuer jedes Geraet: Typ, Status, Firmware, Leistungsdaten, Batterie-Stand, MQTT-Daten, Schedule, Attribute

### 5.2 Cache-Aktualisierung

- **`_update_site()`** / **`_update_dev()`** / **`_update_account()`:** Interne Methoden, die einzelne Cache-Eintraege mit neuen Daten zusammenfuehren
- **`recycleDevices()`:** Entfernt Geraete aus dem Cache, die nicht mehr in aktiven Sites vorhanden sind
- **`recycleSites()`:** Entfernt Sites, die nicht mehr als aktiv gemeldet werden
- **`clearCaches()`:** Leert alle Caches und stoppt die MQTT-Session
- **`customizeCacheId()`:** Erlaubt manuelle Anpassung von Cache-Werten (z.B. Batteriekapazitaet, dynamische Preise)

### 5.3 Callback-System

Das Callback-System ermoeglicht es externen Komponenten (z.B. MQTT-Device-Instanzen), ueber Cache-Aenderungen benachrichtigt zu werden:

```python
self._device_callbacks: dict[str, dict]
# Struktur: {device_sn: {"functions": {callback1, callback2, ...}, "dynamic_descriptions": {...}}}
```

- **`register_device_callback()`:** Registriert eine Callback-Funktion fuer ein Geraet
- **`notify_device()`:** Benachrichtigt alle registrierten Callbacks mit den aktuellen Geraetedaten
- Beim Entfernen eines Geraets aus dem Cache wird der Callback mit einem leeren Dictionary aufgerufen

### 5.4 MQTT-Session-Integration

Die Methode `startMqttSession()` in `apibase.py` initialisiert und verbindet die MQTT-Session:

1. Erstellt `AnkerSolixMqttSession`-Instanz falls nicht vorhanden
2. Verbindet den MQTT-Client mit dem Server
3. Registriert den Message-Callback (`mqtt_received()`)
4. Initialisiert `mqtt_data`-Felder fuer alle MQTT-faehigen Geraete

Die Methode `mqtt_received()` wird bei jeder MQTT-Nachricht aufgerufen und aktualisiert den Device-Cache mit den extrahierten MQTT-Daten.

---

## 6. Data Polling

**Datei:** `api/poller.py`

### 6.1 Ueberblick

Das Poller-Modul stellt vier Hauptfunktionen bereit, die die API-Endpoints abfragen und den internen Cache aktualisieren:

### 6.2 Polling-Funktionen

**`poll_sites()`** — Hauptfunktion zum Abrufen aller Standort-Informationen:
- Ruft `get_site_list` und `get_scene_info` fuer jeden Standort ab
- Optional: `get_site_homepage` fuer detaillierte Geraetedaten
- Identifiziert Geraetetypen und ordnet sie den korrekten API-Klassen zu
- Aktualisiert `sites` und `devices` Cache

**`poll_site_details()`** — Detaillierte Informationen pro Standort:
- Firmware-Status (`get_ota_batch`)
- Gebundene Geraete (`bind_devices`)
- Site-Preise und dynamische Preise
- Geraete-Attribute und Fittings
- WiFi-Informationen

**`poll_device_details()`** — Geraetespezifische Details:
- Schedules (`get_device_parm`)
- Solar-Kompatibilitaet und OTA-Status
- Wechselrichter-Status fuer Standalone-Geraete
- Charger-Einstellungen und Screensaver

**`poll_device_energy()`** — Energiedaten pro Geraet/Standort:
- Tages-Energiedaten nach Geraetetyp
- Aufgeschluesselt nach: Solarproduktion, Batterie-Laden/Entladen, Netzeinspeisung/Bezug, Hausverbrauch

### 6.3 Exclude-Mechanismus

Alle Polling-Funktionen akzeptieren ein `exclude`-Set mit `ApiCategories`, um bestimmte Abfragen zu ueberspringen und die Anzahl der API-Calls zu reduzieren:

```python
await poll_sites(api, exclude={ApiCategories.DEVICE_DETAILS, ApiCategories.ENERGY})
```

---

## 7. MQTT-Kommunikationsschicht

**Dateien:** `api/mqtt.py`, `api/mqtttypes.py`, `api/mqttmap.py`, `api/mqttcmdmap.py`

### 7.1 Verbindungsaufbau

Die MQTT-Verbindung wird ueber TLS (Port 8883) hergestellt:

1. MQTT-Server-Informationen werden via Cloud API abgefragt (`app/devicemanage/get_user_mqtt_info`)
2. TLS-Zertifikate (CA-Cert, Client-Cert, Client-Key) werden aus der API-Antwort extrahiert und lokal gecacht (`api/authcache/`)
3. Der `paho-mqtt`-Client wird mit TLSv1.2 konfiguriert
4. Client-ID-Format: `android-{app_name}-{user_id}-{certificate_id}`
5. Keepalive: 60 Sekunden (Standard)

### 7.2 Topic-Struktur

```
Empfang:  dt/{app_name}/{model_pn}/{device_sn}/...
Senden:   cmd/{app_name}/{model_pn}/{device_sn}/req
```

- `dt` = Data Topic (Telemetriedaten vom Geraet)
- `cmd` = Command Topic (Befehle an das Geraet)
- `app_name` = Anker-App-Bezeichner aus der MQTT-Info
- `model_pn` = Produkt-Nummer des Geraets (z.B. `A17C1`)
- `device_sn` = Seriennummer des Geraets

### 7.3 Nachrichtenformat

Jede MQTT-Nachricht hat folgende JSON-Struktur:

```json
{
  "head": {
    "version": "1.0.0.1",
    "client_id": "android-{app_name}-{user_id}-{cert_id}",
    "sess_id": "1234-5678",
    "msg_seq": 1,
    "seed": "<random_hex_oder_1>",
    "timestamp": 1700000000,
    "cmd_status": 2,
    "cmd": 17,
    "sign_code": 1,
    "device_pn": "A17C1",
    "device_sn": "SERIALNUMBER"
  },
  "payload": "{\"device_sn\":\"...\",\"account_id\":\"...\",\"data\":\"<base64>\"}"
}
```

Das `payload`-Feld ist ein JSON-String, der ein weiteres JSON-Objekt enthaelt. Das `data`-Feld darin ist Base64-kodiert und enthaelt entweder:
- **Hex-Daten** (binaer) — fuer die meisten Geraete
- **JSON-Strings** (`trans`-Feld statt `data`) — fuer X1/HES-Geraete

### 7.4 Hex-Datenprotokoll

**Datei:** `api/mqtttypes.py` — Klasse `DeviceHexDataHeader`

Binaere MQTT-Nachrichten folgen einem festen Header-Format:

```
Offset  Laenge  Beschreibung
------  ------  ----------------------------------------
0       2       Fester Praefix: FF 09
2       2       Nachrichtenlaenge (Little-Endian, inkl. Praefix und XOR-Checksum)
4       3       Pattern: 03 00/01 0F (00=Senden, 01=Empfangen)
7       2       Message-Typ (z.B. 04 05 = Telemetrie, 04 08 = Sonstiges)
9       0-1     Optionales Inkrement-Byte
10+     n       Datenfelder
n       1       XOR-Checksumme
```

**Datenfelder** werden gemaess dem `SOLIXMQTTMAP` dekodiert. Jeder Datentyp hat eine Kennung (1 Byte, z.B. `0xA0`-`0xA9`) gefolgt von typspezifischen Wertbytes:

- `0xA0`: 1-Byte Integer
- `0xA1`: 2-Byte Integer (Big-Endian)
- `0xA2`: 4-Byte Integer (Big-Endian)
- `0xA4`: String (Laenge im ersten Byte)
- `0xA7`: Nested Data (Laenge in 2 Bytes)
- `0xA8`: Boolean (1 Byte)
- usw.

### 7.5 MQTT-Mappings

**Datei:** `api/mqttmap.py` — Dictionary `SOLIXMQTTMAP`

Das zentrale Mapping-Dictionary ordnet jedem Geraetemodell und jedem Message-Typ die erwarteten Datenfelder zu:

```python
SOLIXMQTTMAP = {
    "A17C1": {           # Solarbank 2 E1600 Pro
        "0405": {        # Message-Typ: Telemetrie
            "fields": [
                {"name": "solar_power_1", "type": 0xA1, "factor": 1},
                {"name": "solar_power_2", "type": 0xA1, "factor": 1},
                {"name": "battery_soc", "type": 0xA0},
                ...
            ]
        },
        "0407": {...},   # Weiterer Message-Typ
        ...
    },
    ...
}
```

### 7.6 Command-Mappings

**Datei:** `api/mqttcmdmap.py`

Definiert die verfuegbaren MQTT-Befehle mit ihren Parametern:

- **`SolixMqttCommands`:** Dataclass mit allen bekannten Befehlsnamen (z.B. `realtime_trigger`, `ac_charge_switch`, `sb_max_load`)
- **`COMMAND_LIST`:** Innerhalb der MQTT-Mappings verschachtelt, ordnet Befehlen ihre Hex-Kodierung, Wertebereiche, Standardwerte und Validierungsregeln zu

Command-Eigenschaften umfassen:
- `VALUE_MIN` / `VALUE_MAX` / `VALUE_STEP` — Wertebereich
- `VALUE_OPTIONS` — Erlaubte Optionen (Liste oder Dict)
- `VALUE_DEFAULT` — Standardwert
- `VALUE_STATE` — Name des zugehoerigen Status-Felds
- `STATE_CONVERTER` — Lambda-Funktion zur Konvertierung zwischen Einstellwert und Status
- `COMMAND_ENCODING` — Encoding-Typ fuer die Nachricht

### 7.7 Callback-Pattern

Die `AnkerSolixMqttSession`-Klasse verwendet folgende paho-mqtt Callbacks:

| Callback | Beschreibung |
|---|---|
| `on_connect` | Bei erfolgreicher Verbindung: Re-Subscribe aller Topics |
| `on_message` | Nachrichtenverarbeitung: Dekodierung, Wertextraktion, Cache-Update |
| `on_disconnect` | Logging bei Verbindungsabbruch |
| `on_subscribe` | Fehlerbehandlung bei fehlgeschlagenen Subscriptions |
| `on_unsubscribe` | Fehlerbehandlung bei fehlgeschlagenen Unsubscriptions |
| `on_publish` | Fehlerbehandlung bei fehlgeschlagenen Publishes |

Die Methode `on_message` fuehrt folgende Schritte aus:
1. Payload-Dekodierung (UTF-8 JSON)
2. Extraktion von Timestamp, Model-PN und Device-SN
3. Base64-Dekodierung des `data`- oder `trans`-Felds
4. Erzeugung eines `DeviceHexData`- oder `DeviceJsonData`-Objekts
5. Extraktion der Werte gemaess dem MQTT-Mapping
6. Aktualisierung des `mqtt_data`-Cache
7. Aufruf des registrierten Message-Callbacks

---

## 8. MQTT Device Control System

**Dateien:** `api/mqtt_device.py`, `api/mqtt_factory.py`, `api/mqtt_solarbank.py`, `api/mqtt_charger.py`, `api/mqtt_pps.py`, `api/mqtt_various.py`

### 8.1 Factory Pattern

Die Klasse `SolixMqttDeviceFactory` erzeugt die passende Geraeteinstanz basierend auf dem Geraetetyp und der Produkt-Nummer:

```python
factory = SolixMqttDeviceFactory(api_instance=api, device_sn="SERIAL123")
device = factory.create_device()
# Gibt zurueck: SolixMqttDeviceSolarbank, SolixMqttDevicePps, etc.
```

Die Zuordnung erfolgt nach:
1. **Geraetekategorie** (`SolixDeviceType`): Solarbank, PPS, Charger, SmartPlug, etc.
2. **Produkt-Nummer** (PN): Muss in der `MODELS`-Menge der jeweiligen Klasse enthalten sein
3. **MQTT-Mapping**: PN muss im `SOLIXMQTTMAP` vorhanden sein

Fallback: Falls kein spezialisiertes Mapping existiert, wird die Basisklasse `SolixMqttDevice` zurueckgegeben (unterstuetzt nur `realtime_trigger`).

### 8.2 Basisklasse SolixMqttDevice

Jede Geraeteinstanz bietet:

- **`models`:** Set der unterstuetzten Modellnummern
- **`features`:** Dictionary, das jeden Command auf die Menge der unterstuetzten Modelle abbildet
- **`controls`:** Dynamisch generiertes Dictionary der verfuegbaren Steuerungen mit Wertebereichen und Optionen
- **`mqttdata`:** Zwischengespeicherte MQTT-Telemetriedaten
- **`device`:** Referenz auf den Device-Cache-Eintrag

Zentrale Methoden:
- **`update_device()`:** Callback fuer Cache-Aenderungen, aktualisiert interne Referenzen
- **`run_command()`:** Validiert und sendet einen MQTT-Befehl
- **`update_controls()`:** Aktualisiert die Control-Beschreibungen basierend auf aktuellem Geraetestatus
- **`get_entity_descriptions()`:** Liefert Beschreibungen fuer Home-Automation-Integrationen

### 8.3 Spezialisierte Klassen

**`SolixMqttDeviceSolarbank`** — Solarbanks (A17C0, A17C1, A17C2, A17C3, A17C5, AE100):
- Temperatureinheit umschalten
- Min-SOC (Batteriereserve) setzen
- AC-Steckdose ein/ausschalten
- LED-Licht ein/ausschalten und Modus waehlen
- Max-Last und Parallelbetrieb konfigurieren
- Netzeinspeisung deaktivieren
- AC-Eingangslimit und PV-Limit setzen

**`SolixMqttDevicePps`** — Portable Power Stations (C200, C300, C800, C1000, F1500, F2000, F3800, etc.):
- AC/DC/USB-Ausgaenge schalten
- Lademodus und -limits setzen
- Eco-Modus und Auto-Off konfigurieren
- Display-Helligkeit und Timeout einstellen

**`SolixMqttDeviceCharger`** — Ladegeraete (Prime Charger A2345):
- Modellspezifische Steuerung

**`SolixMqttDeviceVarious`** — Sonstige (SmartPlug A17X8):
- Ein/Ausschalten

---

## 9. Feature-Module

### 9.1 Energy-Modul

**Datei:** `api/energy.py`

Stellt Methoden zur Abfrage und Aufbereitung von Energiedaten bereit:

- **`energy_daily()`:** Tagesweise Energiedaten fuer beliebige Zeitraeume (max. 366 Tage)
  - Solar-Produktion, Batterie-Laden/-Entladen, Netzeinspeisung/-Bezug, Hausverbrauch
  - Optimierte Abfragen je nach angefordertem Detailgrad (`dayTotals`)
- **`energy_analysis()`:** Direkte Abfrage des `energy_analysis`-Endpoints
- **`home_load_chart()`:** Daten fuer die Lastanzeige auf der App-Startseite
- **`device_pv_energy_daily()`:** PV-Energiedaten fuer Standalone-Wechselrichter
- **`refresh_pv_forecast()`:** Aktualisierung der PV-Vorhersagedaten
- **`get_device_charge_order_stats()`:** Ladestatistiken fuer EV-Charger

### 9.2 Schedule-Modul

**Datei:** `api/schedule.py`

Verwaltet Geraete-Zeitplaene (Schedules) fuer verschiedene Solarbank-Generationen:

- **`get_device_parm()`** / **`set_device_parm()`:** Generische Parameter-Abfrage/-Setzung mit verschiedenen `SolixParmType`-Werten (Schedule Typ 4, 6, 12, 26 etc.)
- **`get_device_load()`** / **`set_device_load()`:** SB1-spezifische Schedule-Verwaltung
- **`set_home_load()`:** Komplexe SB1-Schedule-Verwaltung mit Zeitslot-Validierung
- **`set_sb2_home_load()`:** SB2-Schedule-Verwaltung mit Unterstuetzung fuer verschiedene Usage-Modi (manual, smart_mode, auto_mode)
- **`set_sb2_ac_charge()`:** AC-Ladesteuerung fuer SB2-Geraete
- **`set_sb2_use_time()`:** Zeitbasierte Nutzungssteuerung fuer SB2+ mit Tarif-Integration

### 9.3 Vehicle-Modul

**Datei:** `api/vehicle.py`

Verwaltet Fahrzeuge fuer EV-Charger-Integration:

- **`get_vehicle_list()`:** Liste aller registrierten Fahrzeuge
- **`get_vehicle_details()`:** Details eines Fahrzeugs (Batteriekapazitaet, Ladeleistung, Verbrauch)
- **`create_vehicle()`:** Neues Fahrzeug registrieren
- **`manage_vehicle()`:** Fahrzeug aktualisieren oder loeschen
- **`get_brand_list()`** / **`get_brand_models()`** / **`get_model_years()`:** Fahrzeugkatalog-Abfragen

### 9.4 Export-Modul

**Datei:** `api/export.py`

Exportiert System-Daten als JSON-Dateien:

- Export aller API-Endpoint-Antworten in separate JSON-Dateien
- Optionale **Anonymisierung** von Seriennummern, Site-IDs, E-Mail-Adressen etc.
- Optionale **ZIP-Komprimierung** der exportierten Dateien
- Die exportierten JSON-Dateien koennen als Testdaten fuer Offline-Tests verwendet werden (`testDir()`)

### 9.5 HES API

**Datei:** `api/hesapi.py` — Klasse `AnkerSolixHesApi`

Eigene API-Klasse fuer Home Energy Systeme (X1), die ebenfalls von `AnkerSolixBaseApi` erbt:

- Geraeteinformationen, Installationsdaten, WiFi-Info
- Energiestatistiken (Solar, HES, Grid, Home, PPS, EV-Charger)
- Systemgewinn-Berechnungen
- Waermepumpen-Plaene

### 9.6 PowerPanel API

**Datei:** `api/powerpanel.py` — Klasse `AnkerSolixPowerpanelApi`

Eigene API-Klasse fuer Power Panels, die ebenfalls von `AnkerSolixBaseApi` erbt:

- Geraeteinformationen und Fehlerstatus
- Energiestatistiken (Solar, HES, PPS, Home, Grid, Diesel)
- Firmware-Updates und Konfiguration

---

## 10. Fehlerbehandlung

**Datei:** `api/errors.py`

### 10.1 Exception-Hierarchie

```
AnkerSolixError (Basis-Exception)
    |
    +-- AuthorizationError          (401, 403)
    +-- ConnectError                (502, 504, 997)
    +-- NetworkError                (998)
    +-- ServerError                 (999)
    +-- RequestError                (10000, 10003, 10007, 26161)
    |   +-- ItemNotFoundError       (10004)
    |   +-- ItemExistsError         (31001)
    |   +-- ItemLimitExceededError  (31003)
    +-- BusyError                   (21105)
    +-- RequestLimitError           (429)
    +-- VerifyCodeError             (26050)
    +-- VerifyCodeExpiredError      (26051)
    +-- NeedVerifyCodeError         (26052)
    +-- VerifyCodeMaxError          (26053)
    +-- VerifyCodeNoneMatchError    (26054)
    +-- VerifyCodePasswordError     (26055)
    +-- ClientPublicKeyError        (26070)
    +-- TokenKickedOutError         (26084)
    +-- InvalidCredentialsError     (26108, 26156)
    +-- RetryExceeded               (100053)
    +-- NoAccessPermission          (160003)
```

### 10.2 Fehlercode-Mapping

Die Funktion `raise_error(data, prefix)` wird nach jeder API-Antwort aufgerufen:

1. Extrahiert den `code`-Wert aus der JSON-Antwort
2. Schlaegt den Code im `ERRORS`-Dictionary nach
3. Falls gefunden: Wirft die zugehoerige Exception mit Fehlermeldung
4. Fuer unbekannte Codes >= 10000: Wirft die generische `AnkerSolixError`

---

## 11. Hilfsmodule

**Datei:** `api/helpers.py`

### 11.1 RequestCounter

Zaehlt API-Requests mit Zeitstempeln fuer Rate-Limiting:

- **`add()`:** Fuegt einen neuen Request-Eintrag hinzu
- **`recycle()`:** Entfernt Eintraege, die aelter als 1 Stunde sind
- **`last_minute()`** / **`last_hour()`:** Zaehlt Requests im jeweiligen Zeitfenster
- **`add_throttle()`:** Markiert einen Endpoint als gedrosselt
- **`get_details()`:** Formatierte Ausgabe aller Requests mit Timestamps

### 11.2 Utility-Funktionen

| Funktion | Beschreibung |
|---|---|
| `md5(data)` | Berechnet MD5-Hash fuer String oder Bytes |
| `getTimezoneGMTString()` | Erzeugt Zeitzonen-String (z.B. `GMT+01:00`) |
| `generateTimestamp(in_ms)` | Generiert Unix-Timestamp in Sekunden oder Millisekunden |
| `convertToKwh(val, unit)` | Konvertiert Wh/MWh nach kWh |
| `get_enum_name(enum, value)` | Findet den Namen eines Enum-Werts |
| `round_by_factor(value, factor)` | Rundet einen Wert auf den naechsten Faktor |

---

## 12. Ausfuehrbare Skripte

Im Root-Verzeichnis befinden sich mehrere ausfuehrbare Skripte, die die Bibliothek nutzen:

| Skript | Beschreibung |
|---|---|
| `monitor.py` | Interaktives Hauptmonitor-Tool mit umfangreicher Konsolenausgabe aller System- und Geraetedaten |
| `mqtt_monitor.py` | MQTT-spezifischer Monitor fuer Echtzeit-Telemetriedaten und Geraetesteuerung |
| `export_system.py` | Exportiert alle System-Daten als JSON-Dateien (fuer Debugging/Testing) |
| `energy_csv.py` | Exportiert Energiedaten als CSV-Dateien |
| `test_api.py` | API-Testskript mit konfigurierbaren Testmodi (`TESTAUTHENTICATE`, `TESTAPIMETHODS`, `TESTAPIENDPOINTS`, `TESTAPIFROMJSON`) |
| `test_c1000x_mqtt_controls.py` | Testskript fuer C1000X MQTT-Steuerungsbefehle |
| `example_c1000x_mqtt_integration.py` | Beispiel-Integration fuer C1000X MQTT-Steuerung |
| `grep_mqtt_cmd.py` | Hilfsskript zum Durchsuchen von MQTT-Command-Definitionen |
| `common.py` | Gemeinsame Utilities fuer alle Skripte (Logging-Setup, Credential-Management via `.env`) |

### 12.1 Credential-Management

Die Skripte nutzen `python-dotenv` zum Laden von Zugangsdaten aus einer `.env`-Datei:

```env
ANKERUSER=user@example.com
ANKERPASSWORD=geheim
ANKERCOUNTRY=DE
```

### 12.2 Offline-Testing

Die Bibliothek unterstuetzt Offline-Testing ueber exportierte JSON-Dateien:

1. Export mit `export_system.py` in ein Verzeichnis unter `examples/`
2. Setzen des Test-Verzeichnisses via `api.testDir("examples/example1")`
3. Aufruf von API-Methoden mit `fromFile=True` — Daten werden aus lokalen JSON-Dateien statt vom Server geladen

Das `examples/`-Verzeichnis enthaelt ueber 22 Beispiel-Datensaetze fuer verschiedene Systemkonfigurationen.
