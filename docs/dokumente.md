# Dokumente

Modul **Dokumente** (optional).

**Wofür:** Ein Patient fragt nach seinem Befund oder Arztbrief; Mitarbeitende sehen in atmOS, welche Dokumente
vorliegen, und senden eines an den Patienten. Umgekehrt schickt ein Patient einen Befund, ein Foto oder ein
ausgefülltes Formular; nach der Zuordnung legt atmOS die Datei im Archiv des Patienten im PVS ab.

## Dokumente auflisten – `GET /v1/patients/{patient_id}/documents`

Filter `from`, `to` (Dokumentdatum), seitenweise mit `limit`/`cursor`. Neueste zuerst. Die Liste enthält nur
Metadaten, nicht den Inhalt.

| Feld | Pflicht | Beschreibung |
| --- | --- | --- |
| `document_id` | ja | Kennung |
| `title` | ja | Titel, wie im PVS |
| `content_type` | ja | MIME-Typ, z. B. `application/pdf` |
| `created_at` | ja | Wann das Dokument im PVS abgelegt wurde |
| `document_type` | nein | z. B. `letter` (Arztbrief), `finding` (Befund), `lab`, `image`, `form`, `other` |
| `document_date` | nein | Datum des Dokuments selbst (z. B. Briefdatum) |
| `size_bytes` | nein | Größe |
| `tags[]` | nein | Schlagworte bzw. Kategorien des PVS |

```json
{
  "items": [
    {
      "document_id": "d-9001",
      "title": "Arztbrief Kardiologie",
      "document_type": "letter",
      "content_type": "application/pdf",
      "size_bytes": 183422,
      "document_date": "2026-11-10",
      "created_at": "2026-11-11T08:00:00+01:00"
    }
  ],
  "next_cursor": null
}
```

## Inhalt abrufen – `GET /v1/patients/{patient_id}/documents/{document_id}/content`

Liefert die Datei selbst, mit passendem `Content-Type` und `Content-Disposition` (Dateiname).

## Dokument ablegen – `POST /v1/patients/{patient_id}/documents`

`multipart/form-data` mit zwei Teilen:

| Teil | Pflicht | Inhalt |
| --- | --- | --- |
| `metadata` | ja | JSON: `title` (Pflicht), `file_name` (Pflicht), `user_code` (Pflicht), `document_type`, `document_date`, `tags` |
| `file` | ja | Die Datei. Das PVS nimmt mindestens PDF, JPEG und PNG an |

Antwort: das abgelegte Dokument mit `document_id` (`201`). Zu groß: `413`; Dateityp nicht angenommen: `415`.

```http
POST /v1/patients/4711/documents
Authorization: Bearer <token>
Content-Type: multipart/form-data; boundary=grenze

--grenze
Content-Disposition: form-data; name="metadata"
Content-Type: application/json

{ "title": "Befund Hausarzt-Vertretung", "file_name": "befund.pdf", "document_type": "finding", "user_code": "BSP" }
--grenze
Content-Disposition: form-data; name="file"; filename="befund.pdf"
Content-Type: application/pdf

<Dateiinhalt>
--grenze--
```
