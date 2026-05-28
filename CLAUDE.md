# CCS Karty – Tracker aplikace

## Problém

Uživatel používá předplacené CCS karty vázané na SPZ automobilu. Každá karta má měsíční limit čerpání. Uživatel potřebuje vědět, za kolik ještě může natankovat.

- CCS karty jsou firemní — emaily o transakcích chodí pro **všechna firemní SPZ**, uživatel potřebuje filtrovat jen svou SPZ
- Zůstatek v emailu (např. 130 345 Kč) je **celkový účet firmy**, ne limit na konkrétní SPZ
- Měsíční limit na SPZ je znám uživateli, není v emailu

---

## Zdroje dat

### CCS e-maily (Outlook / Microsoft 365)
E-maily chodí od `market@ccs.cz`. Formát těla emailu (HTML):

```
Vážený zákazníku, zůstatek na účtu číslo 8000288962 se snížil o částku 997,1 CZK.
Dostupný zůstatek k 26.05.2026 je 130 345,3 CZK.
Pro úplnost uvádíme detaily této platby:
Platba kartou XXXXXXX XXXXX7232004 9AP7292 (nezaúčtováno) v Pha 5,Plzeňská
Částka: 997,1 CZK
Datum provedení: 26.05.2026
Kód transakce: 5490646
```

**Klíčová pole pro parsing:**
- **SPZ**: součást řádku `Platba kartou … {SPZ} (…)` — 7 znaků, alfanumerické (př. `9AP7292`)
- **Částka**: regex `snížil o částku (X,X) CZK` nebo `Částka: (X,X) CZK`
- **Datum**: `receivedDateTime` z Graph API
- **Lokace**: `v {město/ulice}` za identifikátorem SPZ

**Ověřené regex vzory (Node.js):**
```js
// Částka
const m1 = body.match(/sn[íi][žz]il\s+o\s+[čc][áa]stku\s+([\d\s]+[,.]\d+)\s*CZK/i);
const m2 = body.match(/[Čč][áa]stka:\s*([\d\s]+[,.]\d+)\s*CZK/i);

// Lokace
const locM = body.match(/\bv\s+([A-ZÁČĎÉĚÍŇÓŘŠŤÚŮÝŽ][^\n,]{2,30})/);
```

---

## Architektura řešení

**PWA (Progressive Web App)** — single HTML soubor, žádný backend, funguje jako nativní appka na Androidu.

### Stack
- **Auth**: MSAL.js 2.38.3 (`https://alcdn.msauth.net/browser/2.38.3/js/msal-browser.min.js`)
- **Data**: Microsoft Graph API (`/me/messages`) — čte emaily přímo z Outlooku
- **Storage**: `localStorage` — transakce, nastavení, cache
- **Hosting**: lokální soubor nebo libovolný statický hosting (GitHub Pages apod.)

### Graph API dotaz
```
GET /me/messages
  ?$filter=receivedDateTime ge {ISO_datum} and from/emailAddress/address eq 'market@ccs.cz'
  &$select=receivedDateTime,body
  &$top=200
  &$orderby=receivedDateTime desc
```

### Datový model v localStorage
```
ccs_client_id       — Azure App Client ID (string)
ccs_spz             — SPZ uživatele, uppercase (string)
ccs_limit           — měsíční limit v Kč (float)
ccs_last_sync       — datum posledního syncu (string)
ccs_tx_{YYYY}_{MM}  — pole transakcí pro daný měsíc (JSON array)
```

Struktura jedné transakce:
```json
{
  "ts": 1748304000000,
  "date": "2026-05-26T14:00:00.000Z",
  "amount": 997.1,
  "location": "Pha 5"
}
```

---

## Soubory v projektu

```
index.html    — kompletní PWA aplikace (self-contained)
CLAUDE.md     — tento soubor
```

---

## Nastavení Azure App Registration (nutné pro auth)

1. Jdi na `portal.azure.com`
2. **App registrations → New registration**
   - Name: `CCS Tracker`
   - Supported account types: *Accounts in any organizational directory and personal Microsoft accounts*
   - Redirect URI: typ **SPA** → URL kde běží appka (nebo `http://localhost` pro vývoj)
3. Zkopíruj **Application (client) ID**
4. **API permissions → Add permission → Microsoft Graph → Delegated → `Mail.Read`**
5. Client ID zadej do appky při prvním spuštění

---

## UX flow

1. První spuštění → obrazovka Azure setup (zadej Client ID)
2. → Login přes Microsoft (popup OAuth)
3. → Setup (zadej SPZ + měsíční limit)
4. → Dashboard (přehled zbývajícího limitu)
5. Tlačítko **Sync** → načte emaily za posledních 3 měsíce, naparsuje transakce pro danou SPZ
6. Navigace: **Přehled** (dashboard) | **Nastavení** (změna SPZ/limitu, logout, reset)

---

## Možná vylepšení / TODO

- [ ] Automatický sync při otevření appky
- [ ] Push notifikace když zbývá méně než X %
- [ ] Podpora více SPZ / karet najednou
- [ ] Export transakcí do CSV
- [ ] Offline PWA manifest + service worker pro plné offline fungování
- [ ] Testování s reálnými emaily — ověřit edge cases v parsingu (různé formáty lokace, odmítnuté transakce, dobití atd.)
- [ ] Rozlišení `(nezaúčtováno)` vs finálně zaúčtované transakce
