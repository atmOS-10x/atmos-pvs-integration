# Mitarbeitende

Modul **Mitarbeitende** (Pflicht).

## Alle Benutzer – `GET /v1/staff`

**Wofür:** In der Einrichtung ordnet die Praxis jeden Mitarbeitenden in atmOS seinem PVS-Kürzel zu. Danach

- schreibt atmOS Karteieinträge, Bestellungen und Termine unter dem Kürzel der Person, die sie freigegeben hat;
- zeigt atmOS Kürzel aus der Kartei mit vollem Namen an.

Geliefert werden **alle** Behandler, MFA und sonstigen Benutzer, die in dieser PVS-Installation angelegt sind.

| Feld | Pflicht | Beschreibung |
| --- | --- | --- |
| `user_code` | ja | Kürzel im PVS, z. B. `MUS` |
| `full_name` | ja | Voller Name, z. B. `Dr. med. Erika Musterfrau` |
| `active` | nein | `false` für Benutzer, die nicht mehr in der Praxis arbeiten |

```json
{
  "items": [
    { "user_code": "MUS", "full_name": "Dr. med. Erika Musterfrau", "active": true },
    { "user_code": "BSP", "full_name": "Bernd Beispiel", "active": true }
  ]
}
```
