# Patienten

Modul **Patienten** (Pflicht) und Modul **Kontaktdaten** (optional).

## Patientensuche – `GET /v1/patients/search` (Pflicht)

**Wofür:** atmOS ordnet eingehende Anfragen dem richtigen Patienten zu: eine Nachricht im Posteingang, einen Anruf
(über Rufnummer und Geburtsdatum), eine Anfrage über das Patientenportal. Mitarbeitende suchen Patienten nach Name und Geburtsdatum.

**Suchparameter** (mindestens einer; mehrere werden mit UND verknüpft):

| Parameter | Beschreibung | Beispiel |
| --- | --- | --- |
| `first_name` | Vorname oder dessen Anfang, Groß-/Kleinschreibung egal | `Max` |
| `last_name` | Nachname oder dessen Anfang | `Muster` |
| `birth_date` | Geburtsdatum, auch unvollständig: `YYYY-MM-DD`, `YYYY-MM`, `YYYY` oder `--MM-DD` (Tag und Monat) | `1970-01` |
| `phone` | Telefonnummer, bevorzugt E.164. Das PVS findet sie unabhängig von der Schreibweise in der Kartei | `+493023125000` |
| `email` | E-Mail-Adresse, Groß-/Kleinschreibung egal | `max.mustermann@example.org` |
| `limit` | Höchstzahl der Treffer (Standard 50) | `10` |

**Antwort:** eine **nach Übereinstimmung sortierte** Trefferliste, beste zuerst. Eine leere Liste heißt: kein Treffer.

| Feld | Pflicht | Beschreibung |
| --- | --- | --- |
| `patient_id` | ja | Patientennummer im PVS |
| `first_name` | ja | Vorname |
| `last_name` | ja | Nachname |
| `birth_date` | ja | Geburtsdatum (`null`, wenn unbekannt) |
| `score` | nein | Güte der Übereinstimmung von 0 bis 1 |

```http
GET /v1/patients/search?phone=%2B493023125000&birth_date=1970-01-01
Authorization: Bearer <token>
```

```json
{
  "items": [
    { "patient_id": "4711", "first_name": "Max", "last_name": "Mustermann", "birth_date": "1970-01-01", "score": 1.0 }
  ]
}
```

Hinweis zum Abgleich am Telefon: Eine Rufnummer allein identifiziert einen Patienten nicht sicher (Familien teilen
Festnetznummern, Rufnummern werden unterdrückt). atmOS kombiniert sie deshalb mit dem Geburtsdatum. Das PVS soll
alle Patienten liefern, auf die die Nummer passt, nicht nur den ersten.

## Stammdaten – `GET /v1/patients/{patient_id}` (Pflicht)

**Wofür:** atmOS zeigt die Stammdaten zur Anfrage an, leitet Anrufe je nach Versicherung unterschiedlich (GKV, PKV,
HZV) und prüft vor einer Rezeptbestellung, ob die Versichertenkarte im laufenden Quartal eingelesen wurde.

| Feld | Pflicht | Beschreibung |
| --- | --- | --- |
| `patient_id` | ja | Patientennummer |
| `first_name`, `last_name` | ja | Name |
| `title`, `name_prefix`, `name_suffix` | nein | Titel, Namensvorsatz, Namenszusatz |
| `birth_date` | ja | Geburtsdatum |
| `sex` | nein | `female`, `male`, `diverse`, `unknown` |
| `insurance.type` | ja | `public` (GKV), `private` (PKV), `other` (z. B. Selbstzahler), `unknown` |
| `insurance.hzv` | ja | Teilnahme an der Hausarztzentrierten Versorgung: `active`, `requested`, `ended`, `none`, `unknown` |
| `insurance.contracts[]` | nein | Verträge, in die der Patient eingeschrieben ist (siehe unten) |
| `insurance.card_last_read_at` | ja | Wann die Versichertenkarte zuletzt eingelesen wurde (`null`, wenn nie) |
| `insurance.insurer_name` | nein | Name der Krankenkasse |
| `insurance.insurance_number` | nein | Versichertennummer |
| `phones[]` | ja | Alle Telefonnummern (leere Liste, wenn keine) |
| `phones[].phone_id` | ja | Kennung der Nummer, für Änderungen |
| `phones[].number` | ja | Nummer in E.164, `null`, wenn das PVS sie nicht normalisieren kann |
| `phones[].number_raw` | nein | Nummer, wie sie im PVS steht |
| `phones[].kind` | nein | `mobile`, `landline`, `work`, `fax`, `other` |
| `phones[].primary` | nein | Hauptnummer |
| `emails[]` | ja | Alle E-Mail-Adressen (`email_id`, `address`, `primary`) |
| `address` | nein | `street`, `zip_code`, `city`, `country` |

**Verträge.** `insurance.contracts[]` listet die Selektivverträge, in die der Patient eingeschrieben ist: HZV und
weitere Verträge, z. B. Facharztverträge.

| Feld | Pflicht | Beschreibung |
| --- | --- | --- |
| `type` | ja | `hzv` (Hausarztzentrierte Versorgung), `specialist` (Facharztvertrag) oder `other` |
| `name` | ja | Name des Vertrags, wie im PVS |
| `status` | ja | `active`, `requested` (beantragt), `ended` |
| `since` | nein | Seit wann der Patient eingeschrieben ist |
| `until` | nein | Bis wann, bei beendeten Verträgen |

`insurance.hzv` ist die Zusammenfassung für die HZV und muss zu `contracts` passen: `active`, `requested` oder
`ended`, wenn ein HZV-Vertrag mit diesem Stand gelistet ist (bei mehreren gilt `active` vor `requested` vor
`ended`); `none`, wenn das PVS weiß, dass keiner besteht; `unknown`, wenn es das nicht weiß.

Telefonnummern bitte **normalisiert** (E.164) liefern. Wenn das nicht für alle Nummern geht, `number: null` und die
Originalschreibweise in `number_raw`.

Statt eines Merkmals „Karte in diesem Quartal eingelesen“ liefert das PVS den **Zeitpunkt des letzten Einlesens**;
atmOS leitet das Quartal daraus ab.

```json
{
  "patient_id": "4711",
  "first_name": "Max",
  "last_name": "Mustermann",
  "birth_date": "1970-01-01",
  "insurance": {
    "type": "public",
    "insurer_name": "Beispiel-Krankenkasse",
    "hzv": "active",
    "contracts": [
      { "type": "hzv", "name": "Beispiel-HZV-Vertrag", "status": "active", "since": "2024-01-01" }
    ],
    "card_last_read_at": "2026-10-06T08:12:00+02:00"
  },
  "phones": [
    { "phone_id": "p1", "number": "+493023125000", "number_raw": "030 / 23125 000", "kind": "landline", "primary": true }
  ],
  "emails": [
    { "email_id": "e1", "address": "max.mustermann@example.org", "primary": true }
  ]
}
```

## Kontaktdaten ändern (optional)

**Wofür:** Ein Patient ruft von einer neuen Nummer an oder nennt eine neue E-Mail-Adresse; Mitarbeitende übernehmen
sie mit einem Klick ins PVS, statt sie dort abzutippen.

| Endpunkt | Wirkung |
| --- | --- |
| `POST /v1/patients/{patient_id}/phones` | Nummer hinzufügen: `number` (E.164, Pflicht), `kind`, `primary`, `user_code` (Pflicht) |
| `PATCH /v1/patients/{patient_id}/phones/{phone_id}` | Nummer ändern: nur die mitgesandten Felder, dazu `user_code` |
| `POST /v1/patients/{patient_id}/emails` | E-Mail-Adresse hinzufügen: `address` (Pflicht), `primary`, `user_code` (Pflicht) |
| `PATCH /v1/patients/{patient_id}/emails/{email_id}` | E-Mail-Adresse ändern |

Setzt ein Aufruf `primary: true`, ist danach nur diese Nummer bzw. Adresse die Hauptnummer bzw. -adresse.

```http
POST /v1/patients/4711/phones
Authorization: Bearer <token>
Content-Type: application/json

{ "number": "+491713920000", "kind": "mobile", "primary": false, "user_code": "BSP" }
```

Löschen von Nummern und Adressen ist in dieser Version nicht vorgesehen.
