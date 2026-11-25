# Überweisung

Zwei optionale Module: **Überweisungs-Anfragen** (`referral_requests` in `GET /v1/info`) und
**Überweisungs-Verlauf** (`referral_history`). Ein PVS kann Überweisungs-Anfragen anbieten, ohne den Verlauf zu
liefern; Folgeüberweisungen sind dann nur in Freitext möglich.

**Grundsatz:** atmOS stellt **keine Überweisung aus**. atmOS nimmt den Wunsch des Patienten auf und legt ihn im PVS
als Anfrage an. Ob, wohin und mit welcher Diagnose und welchem Auftrag überwiesen wird, entscheidet allein der Arzt
im PVS. Das PVS erzeugt aus der Anfrage keine Überweisung automatisch.

## Überweisung anfragen – `POST /v1/referral-requests`

**Wofür:** Ein Patient bittet am Telefon oder per Nachricht um eine Überweisung, z. B. zum Kardiologen, oder braucht
für eine laufende Behandlung bei einem Facharzt eine Folgeüberweisung. Der Arzt sieht die Anfrage im PVS und
entscheidet.

| Feld | Pflicht | Beschreibung |
| --- | --- | --- |
| `patient_id` | ja | Patientennummer |
| `kind` | ja | `first` (Erstüberweisung) oder `follow_up` (Folgeüberweisung) |
| `previous_referral_id` | nein | Nur bei `follow_up`: die `referral_id` einer früheren Überweisung dieses Patienten aus dem Verlauf |
| `to` | ja, außer bei `follow_up` mit `previous_referral_id` | Wohin: Fachrichtung oder Arzt bzw. Praxis, als Freitext, 1–200 Zeichen |
| `reason` | bei `first` ja, sonst nein | Der Grund in den Worten des Patienten, 1–2000 Zeichen |
| `findings` | nein | Befunde für den weiterbehandelnden Arzt, bis 2000 Zeichen |
| `note` | nein | Hinweis für die Praxis |
| `user_code` | ja | Kürzel der Person, die die Anfrage freigegeben hat |

Das PVS antwortet mit `422`, wenn

- bei `kind: first` eine `previous_referral_id` mitkommt oder `to` bzw. `reason` fehlt,
- bei `kind: follow_up` weder `to` noch `previous_referral_id` angegeben ist,
- die `previous_referral_id` keine Überweisung dieses Patienten ist.

Gibt es den Patienten nicht, antwortet es mit `404`.

**Antwort** (`201`): `request_id`, `patient_id`, `status` (nach dem Anlegen immer `pending`: liegt dem Arzt vor).

### Folgeüberweisung

Eine Überweisung gilt für das Quartal, in dem sie ausgestellt wurde. Läuft eine Behandlung beim Facharzt weiter,
braucht der Patient deshalb in jedem neuen Quartal eine neue Überweisung, die Folgeüberweisung.

Gibt atmOS bei `kind: follow_up` eine `previous_referral_id` an, übernimmt das PVS Fachrichtung bzw. Empfänger,
Diagnose und Auftrag aus der früheren Überweisung in die Anfrage für das neue Quartal; der Arzt muss nur noch
bestätigen. Ohne `previous_referral_id` (z. B. wenn die frühere Überweisung nicht im Verlauf steht) ist die
Folgeüberweisung eine Anfrage in Freitext mit `to`.

### Beispiele

Erstüberweisung:

```http
POST /v1/referral-requests
Authorization: Bearer <token>
Content-Type: application/json

{
  "patient_id": "4711",
  "kind": "first",
  "to": "Kardiologie",
  "reason": "Herzstolpern seit einigen Wochen, vor allem abends",
  "user_code": "BSP"
}
```

Folgeüberweisung mit Verweis auf die frühere Überweisung:

```http
POST /v1/referral-requests
Authorization: Bearer <token>
Content-Type: application/json

{
  "patient_id": "4711",
  "kind": "follow_up",
  "previous_referral_id": "u-2201",
  "note": "Kontrolltermin beim Kardiologen ist schon vereinbart",
  "user_code": "BSP"
}
```

Antwort in beiden Fällen, z. B.:

```json
{ "request_id": "ue-89", "patient_id": "4711", "status": "pending" }
```

Unbekannte frühere Überweisung (`422`):

```json
{
  "type": "https://atmos.example.org/probleme/fachlich-nicht-ausfuehrbar",
  "title": "Fachlich nicht ausführbar",
  "status": 422,
  "detail": "previous_referral_id: unbekannte Überweisung für diesen Patienten",
  "errors": [{ "parameter": "previous_referral_id", "code": "unknown_referral" }]
}
```

## Ausgestellte Überweisungen – `GET /v1/patients/{patient_id}/referrals`

Modul **Überweisungs-Verlauf** (optional).

**Wofür:** Bittet ein Patient um eine Folgeüberweisung, kann atmOS nachfragen („Die Überweisung zur Kardiologie aus
dem letzten Quartal?“) und die gewählte Überweisung als `previous_referral_id` weitergeben. Die Diagnosen helfen den
Mitarbeitenden, zwei Überweisungen zur selben Fachrichtung zu unterscheiden.

Filter `from` (nur Überweisungen, die an oder nach diesem Tag ausgestellt wurden), seitenweise mit
`limit`/`cursor`, neueste zuerst.

| Feld | Pflicht | Beschreibung |
| --- | --- | --- |
| `referral_id` | ja | Kennung; für Folgeüberweisungen als `previous_referral_id` verwendbar |
| `issued_on` | ja | Ausstellungsdatum |
| `quarter` | ja | Quartal, für das die Überweisung gilt, als `JJJJ-Q`, z. B. `2026-4` |
| `to` | ja | Fachrichtung und/oder Empfänger, als Text |
| `type` | ja | `order` (Auftragsleistung), `consultation` (Konsiliaruntersuchung), `co_treatment` (Mitbehandlung), `further_treatment` (Weiterbehandlung), `other` |
| `diagnoses[]` | ja | Diagnosen der Überweisung als Text; leere Liste, wenn keine angegeben sind |
| `issued_by` | nein | Kürzel des Arztes |

```json
{
  "items": [
    { "referral_id": "u-2201", "issued_on": "2026-10-02", "quarter": "2026-4", "to": "Kardiologie",
      "type": "co_treatment", "diagnoses": ["Herzrhythmusstörung, nicht näher bezeichnet"], "issued_by": "MUS" }
  ],
  "next_cursor": null
}
```
