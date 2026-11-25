# atmOS Health – PVS-Schnittstelle

Diese Spezifikation beschreibt, welche Schnittstelle eine Praxissoftware (PVS) bereitstellen muss, damit
atmOS Health sie anbinden kann: welche Endpunkte, welche Daten, in welcher Form und wofür atmOS sie nutzt.

> **Stand:** Entwurf, Version 0.1.0. Bis zur Version 1.0 können sich Endpunkte und Felder noch ändern.

## Worum es geht

atmOS Health nimmt Anfragen von Patienten an die Praxis entgegen, etwa Anrufe oder Nachrichten im Posteingang, und
hilft, sie zu erledigen: Termine buchen und absagen, Rezepte bestellen, AU und Überweisungen anfragen, Dokumente senden und empfangen. Dafür braucht
atmOS Zugriff auf einige Daten im PVS: Wer ist der
Patient, wie ist er versichert, was steht im Medikationsplan, welche Termine hat er. Und atmOS muss Ergebnisse ins
PVS zurückschreiben können, vor allem in die Kartei.

Für die Daten dieser Schnittstelle ist das PVS die führende Quelle: atmOS liest sie dort und schreibt Änderungen
dorthin zurück.

## So funktioniert die Anbindung

```
 Patient                atmOS Health                         PVS
 (Anruf,            ──▶ Posteingang,            ──REST──▶  diese Schnittstelle
  Nachricht)            Mitarbeitende der Praxis ◀─JSON──   (Hersteller stellt sie bereit)
```

1. **Einrichtung.** Der PVS-Hersteller stellt die Schnittstelle bereit, öffentlich erreichbar oder nur im Praxisnetz
   (dann über den atmOS-Connector in der Praxis). Die Praxis legt im PVS selbst ein Token fest und trägt Adresse und
   Token in atmOS ein. atmOS ruft `GET /v1/info` auf und sieht, welche Module das PVS
   anbietet. Die Praxis ordnet ihre Mitarbeitenden in atmOS den PVS-Kürzeln zu (`GET /v1/staff`) und wählt die
   Textgruppe, in die atmOS schreibt (`GET /v1/casefile/text-groups`).
2. **Betrieb.** atmOS ist immer der aufrufende Teil: Es sucht Patienten, liest Stammdaten, Medikationsplan, Kartei,
   Dokumente und Termine und schreibt Karteieinträge, Bestellungen, Termine und Dokumente zurück. Das PVS muss
   atmOS nicht von sich aus benachrichtigen.
3. **Freigabe.** Was atmOS ins PVS schreibt, hat in der Regel eine Person in der Praxis freigegeben oder eine Regel,
   die die Praxis selbst festgelegt hat. Jeder Schreibaufruf nennt das PVS-Kürzel der verantwortlichen Person.

Ein typischer Ablauf, Rezeptbestellung am Telefon:

1. Ein Anruf kommt an. atmOS sucht den Patienten über Rufnummer und Geburtsdatum (`GET /v1/patients/search`).
2. atmOS liest die Stammdaten (Versicherung, ob die Versichertenkarte im Quartal eingelesen ist) und den
   Medikationsplan (`GET /v1/patients/{patient_id}`, `GET /v1/patients/{patient_id}/medication-plan`).
3. Der Patient wählt am Telefon sein Medikament. Ist der Plan vom Arzt freigegeben, geht die Bestellung direkt an
   den Arzt, sonst zuerst zur Prüfung an die MFA.
4. Nach der Freigabe legt atmOS die Bestellung im PVS an (`POST /v1/medication-orders`) und vermerkt den Vorgang in
   der Kartei (`POST /v1/patients/{patient_id}/casefile`).

## Was ein PVS bereitstellen muss

Die Schnittstelle ist in Module gegliedert. **Pflichtmodule** braucht jede Anbindung. **Optionale Module** schalten
weitere Funktionen in atmOS frei; ein PVS kann sie später nachreichen.

| Modul | Pflicht? | Endpunkte | Beschreibung |
| --- | --- | --- | --- |
| Allgemein | Pflicht | `GET /v1/info` | Version und angebotene Module |
| Patienten | Pflicht | `GET /v1/patients/search`<br>`GET /v1/patients/{patient_id}` | Patientensuche, Stammdaten |
| Mitarbeitende | Pflicht | `GET /v1/staff` | Behandler, MFA und weitere Benutzer mit Kürzel und Namen |
| Kartei | Pflicht | `GET /v1/casefile/text-groups`<br>`GET /v1/patients/{patient_id}/casefile`<br>`POST /v1/patients/{patient_id}/casefile` | Textgruppen, Kartei lesen und schreiben |
| Kontaktdaten | optional | `POST /v1/patients/{patient_id}/phones`<br>`PATCH /v1/patients/{patient_id}/phones/{phone_id}`<br>`POST /v1/patients/{patient_id}/emails`<br>`PATCH /v1/patients/{patient_id}/emails/{email_id}` | Telefonnummern und E-Mail-Adressen ergänzen und ändern |
| Medikation | optional | `GET /v1/patients/{patient_id}/medication-plan`<br>`POST /v1/medication-orders` | Medikationsplan mit Freigabe, Rezeptbestellung |
| Freitext-Bestellung | optional | `POST /v1/medication-orders/free-text` | Medikament als Freitext bestellen, ohne PZN; landet als offene Anforderung in der Praxis |
| AU-Anfragen | optional | `POST /v1/sick-note-requests` | Arbeitsunfähigkeitsbescheinigung (AU bzw. eAU) anfragen; der Arzt entscheidet im PVS |
| AU-Verlauf | optional | `GET /v1/patients/{patient_id}/sick-notes` | Ausgestellte AU und eAU eines Patienten (ohne Diagnosen) |
| Überweisungs-Anfragen | optional | `POST /v1/referral-requests` | Überweisung anfragen, auch als Folgeüberweisung; der Arzt entscheidet im PVS |
| Überweisungs-Verlauf | optional | `GET /v1/patients/{patient_id}/referrals` | Ausgestellte Überweisungen eines Patienten, mit Diagnosen als Text |
| Dokumente | optional | `GET /v1/patients/{patient_id}/documents`<br>`GET /v1/patients/{patient_id}/documents/{document_id}/content`<br>`POST /v1/patients/{patient_id}/documents` | Dokumente auflisten, abrufen, ablegen |
| Kalender | optional | `GET /v1/calendars`<br>`GET /v1/appointments/available`<br>`GET /v1/appointments`<br>`GET /v1/patients/{patient_id}/appointments`<br>`POST /v1/appointments`<br>`POST /v1/appointments/{appointment_id}/cancel` | Kalender, freie und gebuchte Termine, buchen, absagen |
| Verknüpfungen | optional | kein REST-Endpunkt | Patient aus atmOS im PVS öffnen und umgekehrt |

Bietet ein PVS ein optionales Modul an, stellt es alle Endpunkte dieses Moduls bereit.

## Inhalt dieses Repositorys

| Datei | Inhalt |
| --- | --- |
| [openapi/openapi.yaml](openapi/openapi.yaml) | Die maschinenlesbare Spezifikation (OpenAPI 3.1) mit allen Endpunkten, Feldern und Beispielen |
| [docs/grundlagen.md](docs/grundlagen.md) | Erreichbarkeit, Authentifizierung, mehrere Standorte, Testzugang, Formate, Fehler, seitenweise Abfrage, Versionierung |
| [docs/patienten.md](docs/patienten.md) | Patientensuche, Stammdaten, Kontaktdaten |
| [docs/mitarbeitende.md](docs/mitarbeitende.md) | Benutzer und Kürzel des PVS |
| [docs/kartei.md](docs/kartei.md) | Kartei lesen und schreiben |
| [docs/medikation.md](docs/medikation.md) | Medikationsplan, Freigabe, Rezeptbestellung nach PZN und als Freitext |
| [docs/au.md](docs/au.md) | Arbeitsunfähigkeitsbescheinigung (AU und eAU) anfragen, ausgestellte AU lesen |
| [docs/ueberweisung.md](docs/ueberweisung.md) | Überweisung und Folgeüberweisung anfragen, ausgestellte Überweisungen lesen |
| [docs/dokumente.md](docs/dokumente.md) | Dokumente |
| [docs/kalender.md](docs/kalender.md) | Kalender und Termine |
| [docs/verknuepfungen.md](docs/verknuepfungen.md) | Patienten zwischen PVS und atmOS öffnen |

Die OpenAPI-Datei lässt sich mit gängigen Werkzeugen (z. B. Swagger UI, Redoc, OpenAPI Generator) anzeigen oder in
Server-Code für die eigene Implementierung übersetzen. Die Schnittstelle kann in jeder Sprache umgesetzt werden.

## Beispieldaten

Alle Namen, Nummern und Adressen in dieser Spezifikation sind erfunden (Max Mustermann, `example.org`,
Beispiel-PZN `00000001`). Ähnlichkeiten mit echten Personen oder Praxen sind zufällig.
