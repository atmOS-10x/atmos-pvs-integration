# Arbeitsunfähigkeitsbescheinigung (AU und eAU)

Zwei optionale Module: **AU-Anfragen** (`sick_note_requests` in `GET /v1/info`) und **AU-Verlauf**
(`sick_note_history`). Ein PVS kann AU-Anfragen anbieten, ohne den Verlauf zu liefern.

**Grundsatz:** atmOS stellt **keine AU aus**. atmOS nimmt die Bitte des Patienten auf und legt sie im PVS als Anfrage
an. Ob, ab wann, bis wann und mit welcher Diagnose eine AU ausgestellt wird (und ob als eAU), entscheidet allein der
Arzt im PVS. Das PVS erzeugt aus der Anfrage keine Bescheinigung automatisch.

**AU und eAU:** Beide Module gelten für die AU auf Papier und für die elektronische AU (eAU). Es gibt nur **eine**
Anfrage; atmOS wählt nicht, ob auf Papier oder elektronisch ausgestellt wird. Die Praxis stellt die Bescheinigung so
aus, wie sie es muss, bei gesetzlich Versicherten in der Regel als eAU.

## AU anfragen – `POST /v1/sick-note-requests`

**Wofür:** Ein Patient meldet sich krank, am Telefon oder per Nachricht, und bittet um eine Erst- oder
Folgebescheinigung. Der Arzt sieht die Anfrage im PVS mit den Beschwerden und entscheidet.

| Feld | Pflicht | Beschreibung |
| --- | --- | --- |
| `patient_id` | ja | Patientennummer |
| `kind` | ja | `first` (Erstbescheinigung) oder `follow_up` (Folgebescheinigung) |
| `unable_since` | ja | Seit wann der Patient nach eigener Angabe arbeitsunfähig ist |
| `requested_until` | nein | Bis wann der Patient die AU wünscht; der Arzt entscheidet |
| `complaints` | ja | Die Beschwerden in den Worten des Patienten, 1–2000 Zeichen |
| `note` | nein | Hinweis für die Praxis |
| `user_code` | ja | Kürzel der Person, die die Anfrage freigegeben hat |

Liegt `requested_until` vor `unable_since`, antwortet das PVS mit `422`.

**Antwort** (`201`): `request_id`, `patient_id`, `status` (nach dem Anlegen immer `pending`: liegt dem Arzt vor).

```http
POST /v1/sick-note-requests
Authorization: Bearer <token>
Content-Type: application/json

{
  "patient_id": "4711",
  "kind": "first",
  "unable_since": "2026-11-24",
  "requested_until": "2026-11-27",
  "complaints": "Fieber seit gestern, Husten, Gliederschmerzen",
  "user_code": "BSP"
}
```

```json
{ "request_id": "au-77", "patient_id": "4711", "status": "pending" }
```

## Ausgestellte AU – `GET /v1/patients/{patient_id}/sick-notes`

Modul **AU-Verlauf** (optional).

**Wofür:** Bittet ein Patient um eine Folgebescheinigung, zeigt atmOS den Mitarbeitenden und dem Arzt, ob und bis
wann zuletzt eine AU bestand.

Filter `from` (nur AU, die an oder nach diesem Tag enden), seitenweise mit `limit`/`cursor`, neueste zuerst.
Diagnosen und Angaben zur Übermittlung an die Krankenkasse werden **nicht** geliefert.

| Feld | Pflicht | Beschreibung |
| --- | --- | --- |
| `sick_note_id` | ja | Kennung |
| `kind` | ja | `first` oder `follow_up` |
| `unable_from` | ja | Arbeitsunfähig seit |
| `unable_until` | ja | Voraussichtlich arbeitsunfähig bis einschließlich |
| `issued_at` | nein | Wann die AU ausgestellt wurde |
| `issued_by` | nein | Kürzel des Arztes |
| `issued_as` | nein | `electronic` (als eAU ausgestellt) oder `paper` (auf Papier) |

```json
{
  "items": [
    { "sick_note_id": "s-301", "kind": "first", "unable_from": "2026-10-12", "unable_until": "2026-10-16",
      "issued_at": "2026-10-12T09:20:00+02:00", "issued_by": "MUS", "issued_as": "electronic" }
  ],
  "next_cursor": null
}
```
