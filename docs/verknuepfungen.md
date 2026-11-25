# Verknüpfungen zwischen PVS und atmOS

Modul **Verknüpfungen** (optional). Hier gibt es keinen REST-Endpunkt: Es geht darum, mit einem Klick denselben
Patienten im jeweils anderen System zu öffnen.

## Patient aus atmOS im PVS öffnen

**Wofür:** Mitarbeitende bearbeiten in atmOS eine Anfrage und wollen den Patienten im PVS sehen, ohne ihn dort erneut
zu suchen.

**Läuft das PVS im Browser,** genügt eine Adresse, unter der ein Patient direkt aufgeht. Der Hersteller nennt atmOS
dafür eine Vorlage mit dem Platzhalter `{patient_id}`, z. B.:

```
https://pvs.example.org/patienten/{patient_id}
```

**Läuft das PVS als Programm auf dem Praxisrechner,** schlagen wir vor, dass das PVS bei der Installation ein
eigenes URL-Schema im Betriebssystem registriert (URL-Handler), z. B.:

```
beispielpvs://patient/{patient_id}
```

atmOS öffnet diese Adresse; das Betriebssystem übergibt sie an das laufende PVS, das den Patienten anzeigt. Ist das
PVS nicht gestartet, startet es und öffnet dann den Patienten.

In beiden Fällen nennt der Hersteller atmOS die Vorlage; atmOS setzt die Patientennummer ein.

## Patient aus dem PVS in atmOS öffnen

**Wofür:** Mitarbeitende haben einen Patienten im PVS offen und wollen in atmOS seine Anfragen, Nachrichten und
Termine sehen.

Viele PVS haben frei belegbare Schaltflächen, die eine Adresse mit Platzhaltern öffnen. Für atmOS lautet die Vorlage:

```
https://<atmOS-Adresse>/<Praxis-Kürzel>/patient?patient=<Patientennummer>
```

| Teil | Bedeutung |
| --- | --- |
| `<atmOS-Adresse>` | Die Adresse der atmOS-Anwendung, die die Praxis nutzt; sie wird in der Einrichtung mitgeteilt |
| `<Praxis-Kürzel>` | Das Kürzel der Praxis in atmOS, z. B. `praxis-mustermann` |
| `<Patientennummer>` | Die `patient_id` aus dem PVS, die das PVS beim Klick einsetzt |

Beispiel: `https://<atmOS-Adresse>/praxis-mustermann/patient?patient=4711`

Die Adresse enthält nur die Patientennummer, nie Name, Geburtsdatum oder Versichertennummer. Wer die Schaltfläche
nutzt, meldet sich in atmOS wie gewohnt an; ohne Berechtigung für diesen Patienten öffnet sich nichts.
