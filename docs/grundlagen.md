# Grundlagen

Diese Regeln gelten für alle Endpunkte. Die Einzelheiten je Endpunkt stehen in
[openapi/openapi.yaml](../openapi/openapi.yaml).

## Server und Client

- Das **PVS** stellt die Schnittstelle bereit (Server).
- **atmOS Health** ruft sie auf (Client).

## Erreichbarkeit

- Nur HTTPS (TLS 1.2 oder neuer). Alle Pfade beginnen mit `/v1`.
- Die Basisadresse legt der Hersteller fest, z. B. `https://pvs.example.org/atmos`.
- Die Schnittstelle kann auf zwei Arten erreichbar sein:
  - **Öffentlich**, also über das Internet erreichbar (z. B. bei einem PVS, das in der Cloud läuft). atmOS ruft sie
    dann direkt auf.
  - **Nur im Praxisnetz**, also nur von Rechnern in der Praxis erreichbar. Dann installiert atmOS seinen Connector
    auf einem Rechner im Praxisnetz; der Connector ruft die Schnittstelle dort auf. Die Schnittstelle muss dafür
    nicht aus dem Internet erreichbar sein.
- Für beide Fälle gelten dieselben Endpunkte, Formate und dieselbe Authentifizierung.

## Authentifizierung

- Jeder Aufruf trägt `Authorization: Bearer <token>`.
- **Die Praxis legt das Token selbst fest**, in den Einstellungen des PVS, ohne den Hersteller einschalten zu müssen,
  und trägt es danach in atmOS ein.
- Das Token gilt für genau eine Praxis bzw. einen Standort (siehe unten). Die Praxis kann es jederzeit ändern oder
  widerrufen, ohne dass sich die Adresse ändert.
- Ohne oder mit ungültigem Token antwortet das PVS mit `401`.

## Mehrere Standorte

- atmOS bevorzugt **eine Installation je Praxis bzw. Standort**: jede mit eigener Basisadresse und eigenem Token.
  Praxen mit mehreren Standorten oder Abteilungen (z. B. ein MVZ) richten für jeden Standort eine eigene Anbindung
  ein.
- Hält ein PVS alle Praxen oder Standorte in **einer zentralen Datenbank** (z. B. ein PVS in der Cloud), unterscheidet
  es sie über eine **Praxis-ID in der Basisadresse**:
  `https://pvs.example.org/atmos/{practice_id}`, also z. B. `https://pvs.example.org/atmos/p-1001/v1/patients/4711`.
  Die Praxis-ID vergibt das PVS. Die Endpunkte bleiben dieselben.
- Das Token gilt nur für die Praxis-ID, für die es festgelegt wurde. Ein Aufruf mit einer anderen Praxis-ID wird mit
  `401` abgelehnt.

## Testzugang

Für die Entwicklung der Anbindung und für einen Pilotbetrieb braucht atmOS einen **Testzugang**: eine Testpraxis im
PVS mit erfundenen Patienten, Terminen, Medikationsplänen und Karteieinträgen, mit eigener Basisadresse und eigenem
Token. Echte Patientendaten werden für die Entwicklung nicht verwendet.

## Formate

| Was | Format | Beispiel |
| --- | --- | --- |
| Inhalt | JSON, UTF-8 (`application/json`) | |
| Datum | `YYYY-MM-DD` | `1970-01-01` |
| Zeitpunkt | RFC 3339 mit Zeitzone | `2026-12-01T09:30:00+01:00` |
| Telefonnummer | E.164 | `+493023125000` |
| Feldnamen | `snake_case`, englisch | `birth_date` |
| Kennungen | Zeichenketten, auch wenn sie im PVS Zahlen sind | `"4711"` |

- Fehlt einem optionalen Feld der Wert, lässt das PVS das Feld weg oder sendet `null`.
- Unbekannte Felder in Antworten ignoriert atmOS. Das PVS darf also zusätzliche Felder senden.
- Kennungen (`patient_id`, `entry_id`, `document_id`, `appointment_id` …) sind dauerhaft: Dieselbe Kennung bezeichnet
  immer dasselbe Objekt.

## Fehler

Fehler werden nach [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457) als `application/problem+json` gemeldet:

```json
{
  "type": "https://atmos.example.org/probleme/ungueltiger-parameter",
  "title": "Ungültiger Parameter",
  "status": 400,
  "detail": "birth_date: erwartet YYYY-MM-DD, YYYY-MM, YYYY oder --MM-DD",
  "errors": [{ "parameter": "birth_date", "code": "invalid_format" }]
}
```

| Status | Bedeutung |
| --- | --- |
| `400` | Anfrage falsch geformt (fehlender Parameter, falsches Format) |
| `401` | Kein oder ungültiges Token |
| `404` | Patient, Dokument, Termin usw. gibt es nicht |
| `409` | Konflikt, z. B. der Termin ist nicht mehr frei |
| `422` | Fachlich nicht ausführbar, z. B. Textgruppe nicht beschreibbar, Kürzel unbekannt |
| `5xx` | Fehler im PVS; atmOS versucht es später erneut |

## Seitenweise Abfrage

Listen, die lang werden können (Kartei, Dokumente, Termine), werden seitenweise geliefert:

- atmOS gibt `limit` an (1–200, Standard 50).
- Gibt es weitere Einträge, enthält die Antwort `next_cursor`. atmOS ruft dann denselben Endpunkt mit
  `cursor=<next_cursor>` und sonst gleichen Parametern erneut auf.
- `next_cursor: null` heißt: das war die letzte Seite.

Der Cursor ist für atmOS undurchsichtig; das PVS darf hineinschreiben, was es braucht.

## Schreibende Aufrufe

- Jeder schreibende Aufruf nennt in `user_code` das PVS-Kürzel der Person, die den Vorgang in atmOS freigegeben hat.
  Das PVS vermerkt den Vorgang unter diesem Kürzel. Ist das Kürzel unbekannt, antwortet es mit `422`.

## Versionierung

- Die Hauptversion steht im Pfad (`/v1`). Innerhalb von `v1` kommen nur rückwärtskompatible Änderungen hinzu
  (neue optionale Felder, neue Endpunkte, neue Werte in Aufzählungen, auf die atmOS vorbereitet ist).
- Inkompatible Änderungen bekommen einen neuen Pfad (`/v2`); `v1` bleibt dann eine Übergangszeit erhalten.
- `GET /v1/info` meldet in `api_version` die genaue Version dieser Spezifikation, die das PVS umsetzt.

## Datenschutz

- Patientendaten gehören nicht in Adressen (Pfade, Parameter), soweit es sich vermeiden lässt. Ausnahme ist die
  Patientensuche, deren Suchbegriffe in der Adresse stehen; das PVS sollte Suchparameter nicht im Klartext
  protokollieren.
