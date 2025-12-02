# Arista WiFi PSK Rotator + Web UI

Rotátor Wi‑Fi PSK pro **Arista CloudVision Wireless Manager** + jednoduché **web UI** s QR kódem a aktuálním heslem.

Repozitář obsahuje:

- `rotate_psk.py` – logika rotace PSK přes Arista WM API + generování QR a `data/current_psk.json`
- `status_server.py` – Flask web server pro zobrazení SSID / hesla / QR
- `arista_psk_rotator_service.py` – Windows služba pro plánovanou rotaci PSK
- `arista_psk_web_service.py` – Windows služba pro web UI
- `deploy.py` – instalační skript (vytvoří config, uloží API klíče, zaregistruje služby)
- `requirements.txt` – Python závislosti

> ❗ Všechno níže počítá s **Windows Server 2022/2025** a **Pythonem 3.12** nainstalovaným přes *Python Installation Manager* (`py`).

---

## 1. Jak získat soubory z GitHubu

Na cílovém serveru:

1. Otevři repozitář v prohlížeči (GitHub).
2. Klikni na **Code → Download ZIP**.
3. ZIP rozbal třeba do  
   `C:\AristaPskRotator` nebo `C:\Users\<user>\Downloads\ARISTA_PSK`.

Dál v návodu budu adresář nazývat prostě **`C:\AristaPskRotator`**.

---

## 2. Přehled architektury

- **rotate_psk.py**
  - přihlásí se do Arista WM REST API pomocí API key
  - najde konfigurační profil pro dané SSID
  - vygeneruje nové heslo ve formátu `Slovo-Slovo-Slovo7`
  - pošle změnu na WM
  - uloží stav do `data/current_psk.json`
  - vygeneruje QR PNG `data/wifi_qr_<SSID>.png`

- **status_server.py**
  - čte `data/current_psk.json`
  - na HTTP portu (z `config.json`, default 8081) renderuje Wi‑Fi kartu s heslem a QR

- **Windows služby**
  - `AristaPskRotate` – 1× denně (nebo v test režimu á 2 minuty) zavolá `rotate_once()` z `rotate_psk.py`
  - `AristaPskWeb` – spustí Flask server ze `status_server.py`

- **API credentials**
  - bezpečně uloženy v registrech:
    - `HKLM\SOFTWARE\AristaPskRotator`
      - `WM_KEY_ID` – API key ID
      - `WM_KEY_VALUE` – API key value

---

## 3. Doporučené zabezpečení

### 3.1 Service account

Silně doporučeno **dedikované službové konto**, např.:

```bat
net user svc_psk_Rotator SuperSilneHeslo123! /add
```

Dále:

1. Otevři **Local Security Policy**  
   `secpol.msc` → **Local Policies → User Rights Assignment**
2. V položce **Log on as a service** přidej účet `.\svc_psk_Rotator` (nebo doménový).

> Účet nemusí umět interaktivní logon, stačí “log on as a service”.

### 3.2 Práva na složku aplikace

Na `C:\AristaPskRotator`:

- `Administrators` – Full control
- `svc_psk_Rotator` – Modify (čtení + zápis – logy, data)
- Odeber (pokud možno) `Users` / `Everyone`.

### 3.3 Práva na registry

Klíč vytváří deploy skript:

```text
HKLM\SOFTWARE\AristaPskRotator
  WM_KEY_ID    (REG_SZ)
  WM_KEY_VALUE (REG_SZ)
```

Doporučené ACL:

- `Administrators` – Full control
- `svc_psk_Rotator` – Read

Hodnoty jsou prostý text, proto omez přístup pouze na adminy + službový účet.

---

## 4. Instalace Pythonu pro service account

### 4.1 Přihlášení jako service account (poprvé)

Poprvé je ideální se přihlásit **přímo jako** `svc_psk_Rotator`:

- buď přes RDP
- nebo lokálně na serveru (Switch user → Other user).

Důvod: Python i `pip` se nainstalují do profilu tohoto uživatele.

### 4.2 Instalace Pythonu 3.12

V PowerShellu / CMD:

```bat
py install 3.12
py -0p
```

Uvidíš něco jako:

```text
 -V:3.14[-64] *   C:\Users\svc_psk_Rotator\AppData\Local\Python\pythoncore-3.14-64\python.exe
 -V:3.12[-64]     C:\Users\svc_psk_Rotator\AppData\Local\Python\pythoncore-3.12-64\python.exe
```

### 4.3 Instalace Python závislostí

Přepni se do adresáře aplikace:

```bat
cd C:\AristaPskRotator
py -3.12 -m pip install -r requirements.txt
```

To nainstaluje např.:

- `requests`
- `flask`
- `pywin32`
- `qrcode`
- `wordfreq`
- atd.

---

## 5. Konfigurace `config.json`

Po prvním spuštění deploy skriptu se vytvoří `config.json`. Typická šablona:

```json
{
  "WM_BASE_URL": "https://awm15001-c4.srv.wifi.arista.com",
  "WM_LOCATION_NAME": "ALTEPRO_LAB",
  "WM_NODE_ID": 0,
  "SSID_PROFILE_NAME": "TSK_TEST",
  "BACKEND_PORT": 8081,
  "ROTATION_HOUR": 2,
  "ROTATION_MINUTE": 0,
  "ROTATION_EVERY_MINUTES": 0,
  "LOG_LEVEL": "INFO"
}
```

Vysvětlení hlavních položek:

- `WM_BASE_URL` – URL Arista WM.
- `WM_LOCATION_NAME` – název lokace; skript si poprvé sám najde `WM_LOCATION_ID` a doplní ho do configu.
- `WM_NODE_ID` – ID node (AP group / zařízení) – dle Arista WM.
- `SSID_PROFILE_NAME` – název SSID profilu v WM.
- `BACKEND_PORT` – port web UI (status server).
- `ROTATION_HOUR`, `ROTATION_MINUTE` – běžný plán: 1× denně.
- `ROTATION_EVERY_MINUTES` – **testovací režim** (např. 2 = každé 2 minuty).  
  Pro produkci nastav `0` nebo položku smaž.
- `LOG_LEVEL` – `INFO` / `DEBUG` / `WARNING` / `ERROR`.

---

## 6. Uložení API klíčů do registru

Stále přihlášen jako **`svc_psk_Rotator`**:

```bat
cd C:\AristaPskRotator
py -3.12 deploy.py
```

Skript:

1. Připraví `config.json`, pokud ještě neexistuje.
2. Zeptá se na:
   - `WM_KEY_ID`
   - `WM_KEY_VALUE`
3. Uloží je do:
   - `HKLM\SOFTWARE\AristaPskRotator`
4. Zkontroluje `pywin32`.
5. Zaregistruje služby:
   - `AristaPskRotate`
   - `AristaPskWeb`
6. Pokusí se je rovnou spustit.

---

## 7. Nastavení služeb

V **services.msc** najdeš:

- **Arista WiFi PSK Rotation Service** (`AristaPskRotate`)
- **Arista WiFi PSK Web UI** (`AristaPskWeb`)

U obou:

1. Otevři **Properties → Log On**.
2. Nastav účet:
   - `.\svc_psk_Rotator`  
   - zadej heslo.
3. Startup type:
   - nejdříve **Manual** (test)
   - po ověření **Automatic**.

> Pokud deploy běžel přímo pod `svc_psk_Rotator`, měl by už Log On nastavit správně – ale je dobré to zkontrolovat.

---

## 8. Ověření provozu

### 8.1 Rotátor

1. Pro test nastav v `config.json`:

```json
"ROTATION_EVERY_MINUTES": 2
```

2. Restartuj **Arista WiFi PSK Rotation Service**.
3. Sleduj log:

```text
C:\AristaPskRotator\logs\service_rotate.log
C:\AristaPskRotator\logs\rotate.log
```

Měl bys vidět něco jako:

```text
... Service loop starting — mode=interval, next_run=...
... Starting scheduled PSK rotation…
... Generated new PSK passphrase: Something-Something-Word7
... PSK rotation SUCCESS: ...
```

### 8.2 Web UI

- Zkontroluj, že běží služba **Arista WiFi PSK Web UI**.
- V prohlížeči na serveru (nebo z LAN) otevři:

```
http://<server-name>:8081/
```

Měl by se zobrazit panel:

- SSID
- Password (Fráze-Slova-1234)
- QR kód
- “Last rotated: … UTC”

---

## 9. Produkční nastavení

Až přestaneš testovat:

1. V `config.json` nastav:

```json
"ROTATION_EVERY_MINUTES": 0,
"ROTATION_HOUR": 2,
"ROTATION_MINUTE": 0
```

2. Restartuj **Arista WiFi PSK Rotation Service**.
3. Ověř v `service_rotate.log`, že režim je `daily` a plán je správný.

---

## 10. Firewall

Nezapomeň přidat výjimku pro port web UI:

```bat
netsh advfirewall firewall add rule name="Arista PSK Web UI" dir=in action=allow protocol=TCP localport=8081
```

Nebo použij jiný port / interní firewall dle firemní politiky.

---

## 11. Odinstalace

### 11.1 Zastavení a odstranění služeb

V adresáři aplikace:

```bat
cd C:\AristaPskRotator
py -3.12 arista_psk_rotator_service.py remove
py -3.12 arista_psk_web_service.py remove
```

Nebo přes `sc delete AristaPskRotate` / `sc delete AristaPskWeb`.

### 11.2 Smazání registru

V RegEditu:

- smaž klíč `HKLM\SOFTWARE\AristaPskRotator`

### 11.3 Smazání aplikace

- smaž složku `C:\AristaPskRotator` (případně archivuj logy).

---

## 12. Troubleshooting

### 12.1 Služba nejde spustit – Error 5: Access is denied

Zkontroluj:

- **Log On** účet má právo *“Log on as a service”*.
- Účet má **Read** na `HKLM\SOFTWARE\AristaPskRotator`.
- Účet má **Modify** na složku `C:\AristaPskRotator` (logy, data).

### 12.2 Služba běží, ale PSK se nemění

- Podívej se do `logs/rotate.log` – typické chyby:
  - špatné `WM_KEY_ID` / `WM_KEY_VALUE`
  - špatné `WM_BASE_URL`
  - špatný `SSID_PROFILE_NAME` / `WM_LOCATION_NAME`

### 12.3 Web UI neukazuje heslo, jen chybu

- Zkontroluj, že už proběhla aspoň 1 úspěšná rotace  
  → musí existovat `data/current_psk.json` a `data/wifi_qr_*.png`.

---

Pokud budeš chtít, můžeme do README ještě přidat příklady pro více SSID / více lokací nebo tipy, jak to sledovat přes externí monitoring.
