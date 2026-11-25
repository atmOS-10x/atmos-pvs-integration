# Medikation

Modul **Medikation** (optional).

## Medikationsplan – `GET /v1/patients/{patient_id}/medication-plan`

**Wofür:** Ein Patient bestellt am Telefon ein Folgerezept und wählt dabei aus seinem
Medikationsplan. atmOS erinnert rechtzeitig, bevor ein Medikament aufgebraucht ist.

### Freigabe durch den Arzt

Das Wichtigste an diesem Endpunkt ist die **Freigabe** (`approval`): Hat ein **Arzt** bestätigt, dass der
Medikationsplan aktuell ist?

- Nur ein Arzt kann den Plan freigeben, **keine MFA**. Dafür sorgt das PVS: Es liefert `approved: true` nur, wenn
  die Freigabe von einem Arzt stammt. `approved_by` nennt dessen Kürzel aus [`GET /v1/staff`](mitarbeitende.md).
- Ist der Plan freigegeben, kann eine Bestellung aus diesem Plan ohne Vorprüfung durch die MFA direkt beim Arzt
  landen. Ist er nicht freigegeben, prüft zuerst die Praxis.
- Ändert sich der Plan nach der Freigabe, gilt er als **nicht freigegeben**, bis ein Arzt ihn erneut bestätigt.

Hat das PVS noch keinen solchen Freigabevermerk, ist das die wichtigste Ergänzung für diese Anbindung. Bis dahin
liefert das PVS `approved: false`.

| Feld | Pflicht | Beschreibung |
| --- | --- | --- |
| `approval.approved` | ja | `true`, wenn ein Arzt den aktuellen Stand bestätigt hat |
| `approval.approved_by` | nein | Kürzel des Arztes |
| `approval.approved_at` | nein | Zeitpunkt der Freigabe |
| `updated_at` | nein | Letzte Änderung des Plans |

### Einträge

| Feld | Pflicht | Beschreibung |
| --- | --- | --- |
| `entry_id` | ja | Kennung des Eintrags |
| `pzn` | nein | Pharmazentralnummer (8 Ziffern); `null` bei Einträgen ohne PZN |
| `name` | ja | Handelsname mit Stärke und Darreichungsform |
| `active_substances[]` | nein | Wirkstoffe mit `name` und `strength` |
| `dosage_form` | nein | Darreichungsform, z. B. `Tabletten` |
| `unit` | nein | Dosiereinheit, z. B. `Stück`, `ml` |
| `dosage` | ja | `morning`, `noon`, `evening`, `night` (wie auf dem bundeseinheitlichen Medikationsplan) oder `free_text` |
| `reason` | nein | Grund der Einnahme |
| `note` | nein | Hinweis |
| `start_date` | nein | Beginn der Einnahme |
| `runs_out_at` | nein, aber dringend erwünscht | Tag, an dem das Medikament nach der letzten Verordnung voraussichtlich aufgebraucht ist |
| `prescribable` | ja | Ob es über atmOS nachbestellt werden darf (`false` z. B. bei BtM oder auf Anweisung des Arztes) |
| `last_order.status` | nein | Stand der letzten Bestellung: `pending`, `processing`, `ready`, `rejected`, `cancelled` |
| `last_order.placed_at` | nein | Datum der letzten Bestellung |

```json
{
  "patient_id": "4711",
  "approval": { "approved": true, "approved_by": "MUS", "approved_at": "2026-10-06T11:02:00+02:00" },
  "updated_at": "2026-10-06T10:55:00+02:00",
  "entries": [
    {
      "entry_id": "m-1",
      "pzn": "00000001",
      "name": "Ramipril Beispiel 5 mg Tabletten",
      "active_substances": [{ "name": "Ramipril", "strength": "5 mg" }],
      "dosage_form": "Tabletten",
      "unit": "Stück",
      "dosage": { "morning": "1", "noon": "0", "evening": "0", "night": "0" },
      "reason": "Bluthochdruck",
      "runs_out_at": "2026-12-15",
      "prescribable": true,
      "last_order": { "status": "ready", "placed_at": "2026-09-12" }
    }
  ]
}
```

## Medikamente bestellen – `POST /v1/medication-orders`

**Wofür:** Ein Patient bestellt am Telefon oder per Nachricht ein Medikament nach. Nach der
Freigabe legt atmOS die Bestellung im PVS an, so wie die Praxis Rezeptanforderungen aus dem PVS kennt (z. B. auf der
Arbeitsliste des Arztes).

atmOS übergibt je Medikament **die PZN** (und, wenn vorhanden, den Planeintrag). Alles Weitere entscheidet das PVS
bzw. die Praxis im PVS, wie bei einer Bestellung, die dort direkt angelegt wird: Packungsgröße, Austausch gegen ein
wirkstoffgleiches Präparat (aut idem), Rezeptart, Weg zum Patienten (eRezept, Papier).

| Feld | Pflicht | Beschreibung |
| --- | --- | --- |
| `patient_id` | ja | Patientennummer |
| `items[].pzn` | ja | PZN des Medikaments |
| `items[].entry_id` | nein | Eintrag im Medikationsplan |
| `user_code` | ja | Kürzel der Person, die die Bestellung freigegeben hat |
| `note` | nein | Hinweis für die Praxis |

**Antwort** (`201`): je Position ein Ergebnis. Einzelne Positionen können fehlschlagen, ohne dass die ganze
Bestellung fehlschlägt.

| Feld | Beschreibung |
| --- | --- |
| `order_id` | Kennung der Bestellung |
| `items[].result` | `issued` (Rezept direkt erzeugt), `queued_for_review` (liegt der Praxis zur Prüfung vor), `failed` |
| `items[].status` | Stand wie bei `last_order.status` |
| `items[].failure_reason` | Grund bei `failed`, z. B. „PZN nicht im Medikationsplan“ |

```http
POST /v1/medication-orders
Authorization: Bearer <token>
Content-Type: application/json

{ "patient_id": "4711", "items": [{ "pzn": "00000001", "entry_id": "m-1" }], "user_code": "BSP" }
```

```json
{
  "order_id": "o-501",
  "patient_id": "4711",
  "items": [{ "pzn": "00000001", "result": "queued_for_review", "status": "pending" }]
}
```

## Medikament als Freitext bestellen – `POST /v1/medication-orders/free-text`

Modul **Freitext-Bestellung** (optional, in `GET /v1/info` als `medication_free_text`). Es hängt nicht vom Modul
Medikation ab: Ein PVS kann Freitext-Bestellungen anbieten, auch wenn es keinen Medikationsplan liefert.

**Wofür:**

- Ein Patient bestellt ein Medikament, das nicht in seinem Medikationsplan steht oder dessen PZN er nicht kennt
  („das Blutdruckmittel, das ich im Krankenhaus bekommen habe“).
- Das PVS nimmt Rezeptanforderungen ohnehin als Text an und nicht als Liste von PZN.

**Was das PVS mit dem Text macht:** Es legt ihn als **offene Rezeptanforderung** an, die Mitarbeitende oder der Arzt
im PVS bearbeiten, so wie eine Anforderung, die in der Praxis von Hand notiert wurde. Das PVS erzeugt daraus **kein
Rezept automatisch** und versucht nicht, den Text selbst einem Medikament zuzuordnen.

| Feld | Pflicht | Beschreibung |
| --- | --- | --- |
| `patient_id` | ja | Patientennummer |
| `text` | ja | Das Medikament, wie der Patient es nennt (Name, Stärke, Menge, soweit bekannt), 1–500 Zeichen |
| `user_code` | ja | Kürzel der Person, die die Bestellung freigegeben hat |
| `note` | nein | Hinweis für die Praxis |

**Antwort** (`201`): `order_id`, `patient_id`, `result` (`queued_for_review` oder `failed`), `status` (wie bei
`last_order.status`, nach dem Anlegen `pending`), bei `failed` ein `failure_reason`.

```http
POST /v1/medication-orders/free-text
Authorization: Bearer <token>
Content-Type: application/json

{ "patient_id": "4711", "text": "Ramipril 5 mg, 100 Stück, wie bisher", "user_code": "BSP" }
```

```json
{ "order_id": "o-502", "patient_id": "4711", "result": "queued_for_review", "status": "pending" }
```
