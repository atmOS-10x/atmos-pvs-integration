# Kartei

Modul **Kartei** (Pflicht).

## Textgruppen – `GET /v1/casefile/text-groups`

**Wofür:** In der Einrichtung wählt die Praxis, in welche Textgruppe (Karteikategorie) atmOS schreibt und welche
Textgruppen atmOS lesen soll.

| Feld | Pflicht | Beschreibung |
| --- | --- | --- |
| `code` | ja | Kürzel der Textgruppe, wie im PVS, z. B. `B` |
| `label` | ja | Voller Name, z. B. `Befund` |
| `writable` | ja | Ob atmOS in diese Gruppe schreiben darf |
| `hidden` | nein | Ob Einträge dieser Gruppe in der normalen Karteiansicht ausgeblendet sind |

Eine **ausgeblendete Textgruppe** ist nützlich für Einträge, die zur Nachvollziehbarkeit im PVS stehen sollen, aber
die normale Karteiansicht nicht füllen sollen (z. B. das vollständige Protokoll eines Anrufs). Kennt ein PVS keine
ausgeblendeten Textgruppen, lässt es `hidden` weg.

## Kartei lesen – `GET /v1/patients/{patient_id}/casefile`

**Wofür:** atmOS zeigt Mitarbeitenden zu einer Anfrage die passenden Karteieinträge, z. B. die letzten Laborwerte,
wenn ein Patient nach seinem Befund fragt, oder prüft, ob ein Anliegen schon dokumentiert ist.

**Filter** (alle optional, mit UND verknüpft):

| Parameter | Beschreibung | Beispiel |
| --- | --- | --- |
| `q` | Volltextsuche im Eintragstext | `Blutdruck` |
| `text_group` | Nur diese Textgruppen, mehrfach angebbar | `text_group=B&text_group=L` |
| `from` | Frühestes Eintragsdatum (einschließlich) | `2026-01-01` |
| `to` | Spätestes Eintragsdatum (einschließlich) | `2026-11-30` |
| `limit`, `cursor` | Seitenweise Abfrage, siehe [Grundlagen](grundlagen.md) | |

**Antwort:** Einträge, neueste zuerst.

| Feld | Pflicht | Beschreibung |
| --- | --- | --- |
| `entry_id` | ja | Kennung des Eintrags |
| `text_group` | ja | Code der Textgruppe |
| `text_group_label` | nein | Voller Name der Textgruppe |
| `text` | ja | Eintragstext |
| `user_code` | nein | Kürzel des Verfassers |
| `written_at` | ja | Datum und Uhrzeit des Eintrags |
| `fields[]` | nein | Bei strukturierten Einträgen (Diagnose, Medikament …) die Felder als `label`/`value` in der Reihenfolge des PVS |

```http
GET /v1/patients/4711/casefile?text_group=B&from=2026-01-01&limit=20
Authorization: Bearer <token>
```

```json
{
  "items": [
    {
      "entry_id": "k-1001",
      "text_group": "B",
      "text_group_label": "Befund",
      "text": "RR 130/85 mmHg, Puls 72/min",
      "user_code": "MUS",
      "written_at": "2026-11-02T10:15:00+01:00"
    }
  ],
  "next_cursor": null
}
```

## In die Kartei schreiben – `POST /v1/patients/{patient_id}/casefile`

**Wofür:** atmOS dokumentiert erledigte Anliegen in der Kartei: eine telefonische Rezeptbestellung, eine
Terminabsage, die Zusammenfassung eines Anrufs. So sieht die Praxis im PVS, was über atmOS gelaufen ist.

| Feld | Pflicht | Beschreibung |
| --- | --- | --- |
| `text_group` | ja | Code einer Textgruppe mit `writable: true` |
| `text` | ja | Eintragstext, bis 20 000 Zeichen, mehrzeilig möglich |
| `user_code` | ja | Kürzel der Person, die den Eintrag verantwortet |
| `written_at` | ja | Datum und Uhrzeit, unter denen der Eintrag stehen soll |

Das PVS legt den Eintrag so an, wie er kommt, und kürzt ihn nicht stillschweigend. Antwort: der angelegte Eintrag
mit `entry_id` (`201`). Nicht beschreibbare Textgruppe oder unbekanntes Kürzel: `422`.

```http
POST /v1/patients/4711/casefile
Authorization: Bearer <token>
Content-Type: application/json

{
  "text_group": "ATM",
  "text": "Patient bittet telefonisch um Folgerezept Ramipril 5 mg. An Praxis weitergeleitet.",
  "user_code": "BSP",
  "written_at": "2026-11-24T09:41:00+01:00"
}
```
