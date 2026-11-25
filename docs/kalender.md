# Kalender und Termine

Modul **Kalender** (optional).

**Wofür:** Patienten buchen und sagen Termine ab, z. B. am Telefon. atmOS sucht freie Termine, bucht sie im
PVS-Kalender, sagt sie ab und zeigt, welche Termine ein Patient hat und welche Termine insgesamt gebucht sind.

## Kalender und Terminarten – `GET /v1/calendars`

Alle Kalender (je Behandler, Raum oder Sprechstunde) mit den Terminarten (Besuchsgründen), die darin buchbar sind.
In der Einrichtung wählt die Praxis, welche davon atmOS anbieten darf.

| Feld | Pflicht | Beschreibung |
| --- | --- | --- |
| `calendar_id` | ja | Kennung |
| `name` | ja | Name |
| `reasons[].reason_id` | ja | Kennung der Terminart |
| `reasons[].name` | ja | Name der Terminart |
| `reasons[].duration_min` | nein | Übliche Dauer in Minuten |

## Freie Termine – `GET /v1/appointments/available`

| Parameter | Pflicht | Beschreibung |
| --- | --- | --- |
| `from`, `to` | ja | Zeitraum (Datum, einschließlich), höchstens 31 Tage |
| `calendar_id` | nein | Nur diese Kalender, mehrfach angebbar |
| `reason_id` | nein | Nur Zeiten, die für diese Terminart passen |

Antwort: freie Termine, früheste zuerst, je mit `calendar_id`, `start_at`, `duration_min` und `reason_ids`.

```json
{ "items": [{ "calendar_id": "cal-1", "start_at": "2026-12-01T09:30:00+01:00", "duration_min": 10, "reason_ids": ["r-akut"] }] }
```

## Gebuchte Termine

Es gibt zwei Abfragen:

| Endpunkt | Wofür |
| --- | --- |
| `GET /v1/patients/{patient_id}/appointments` | Die Termine **eines** Patienten („Wann ist mein nächster Termin?“). Ohne `from` ab heute |
| `GET /v1/appointments` | **Alle** gebuchten Termine im Zeitraum, über alle Patienten, filterbar nach `calendar_id` und Zeitraum (`from`, `to` Pflicht, höchstens 31 Tage) |

Beide liefern abgesagte Termine nur mit `include_cancelled=true`, früheste zuerst, seitenweise.

| Feld | Pflicht | Beschreibung |
| --- | --- | --- |
| `appointment_id` | ja | Kennung, dauerhaft |
| `calendar_id` | ja | Kalender |
| `calendar_name` | nein | Name des Kalenders |
| `reason_id`, `reason_name` | nein | Terminart |
| `start_at` | ja | Beginn |
| `duration_min` | ja | Dauer |
| `patient_id` | nein | `null` bei Terminen ohne angelegten Patienten |
| `patient_name` | nein | Name, wenn kein Patient angelegt ist |
| `status` | ja | `booked` oder `cancelled` |
| `note` | nein | Freitext der Praxis am Termin |
| `room` | nein | Raum |

## Termin buchen – `POST /v1/appointments`

| Feld | Pflicht | Beschreibung |
| --- | --- | --- |
| `calendar_id` | ja | Kalender |
| `start_at` | ja | Beginn |
| `reason_id` | nein | Terminart |
| `duration_min` | nein | Ohne Angabe die Dauer der Terminart |
| `patient_id` | eins von beiden | Bekannter Patient |
| `new_patient` | eins von beiden | Neupatient ohne Patientennummer: `first_name`, `last_name`, `birth_date`, `phone` (Pflicht), `email` |
| `note` | nein | Anliegen des Patienten |
| `user_code` | ja | Kürzel der Person, die die Buchung freigegeben hat |

Antwort: der gebuchte Termin (`201`). Ist die Zeit nicht mehr frei: `409`.

```http
POST /v1/appointments
Authorization: Bearer <token>
Content-Type: application/json

{
  "calendar_id": "cal-1",
  "reason_id": "r-akut",
  "start_at": "2026-12-01T09:30:00+01:00",
  "patient_id": "4711",
  "note": "Halsschmerzen seit drei Tagen",
  "user_code": "BSP"
}
```

## Termin absagen – `POST /v1/appointments/{appointment_id}/cancel`

Körper (optional): `cancel_reason` (Grund, wie der Patient ihn nennt), `user_code`. Antwort: der abgesagte Termin.
Eine wiederholte Absage desselben Termins antwortet mit `200` und dem schon abgesagten Termin.

Verschieben ist in dieser Version eine Absage und eine neue Buchung.
