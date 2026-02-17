# IMB Intervision (IV) Modul – Änderungsanfrage
## Systemweite Datumsnormalisierung & IV-Erweiterungen – Konzept & Ist-Zustand

**Status:** Vereinbarte Richtung mit offenen Fragen  
**Zielgruppe:** Auftraggeber, Kammervertreter, Implementierungsteam  
**Umfang:** Systemweit (IMB-Kern, IV-Modul, Fortbildung, Berichterstattung)  
**Kernentscheidung:** Unix-Timestamp-Speicherung für alle Datums- und Zeitwerte

---

## 1. Kontext & Gesamtrichtung

Während der laufenden Entwicklung und des Betriebs des IMB-Systems wurden verschiedene strukturelle Einschränkungen identifiziert, u. a. bei:

- der Handhabung von Datum und Uhrzeit
- der Teilnehmeridentitätskonsistenz
- der historischen Teilnahme- und Punkt-Berichterstattung
- der Nachvollziehbarkeit und gesetzlichen Aufbewahrungspflichten
- der UI-Skalierbarkeit für lang laufende IV-Gruppen

Nach Rücksprache mit dem Auftraggeber wurde folgende **systemweite Entscheidung** getroffen:

> **Alle Datums- und Zeitwerte im IMB-System werden als Unix-Timestamps gespeichert.**

Diese Änderung geht über das IV-Modul hinaus und erfordert daher:
- einen eigenen Projektumfang
- eine kontrollierte Migration
- einen zeitweisen System-Stopp während des Produktiv-Rollouts

Die nachfolgend beschriebenen IV-bezogenen Änderungen sind **unter der Voraussetzung** dieser neuen timestamp-basierten Grundlage konzipiert.

---

## 2. Systemweite Datumsnormalisierung (Grundlagenänderung)

### 2.1 Kanonische Speicherung
- Alle Datums- und Zeitangaben werden intern als **Unix-Timestamps (UTC)** gespeichert.
- Dies gilt systemweit für:
  - IV-Sitzungen
  - Fortbildungsveranstaltungen
  - Berichtszeiträume
  - Punkteberechnungen
  - jede andere datumsbasierte Logik

### 2.2 Anzeigeformat
- Im UI werden Datumsangaben im deutschen Format angezeigt:
  - `tt.mm.jjjj` (sowie Uhrzeit, falls relevant)
- Die Anzeigeformatierung ist rein präsentational und wird nie für die Logik verwendet.

### 2.3 Migrationsstrategie
- Alle Änderungen werden in einem **Entwicklungs-/Staging-System** entwickelt und getestet.
- Produktiv-Rollout:
  1. Systemstopp (kein Lese-/Schreibzugriff) in der Nacht bzw. frühmorgens
  2. Bereitstellung des aktualisierten Codes
  3. Ausführung der Migrationsskripte
  4. Verifizierungstests
  5. Wiederaufnahme des Systems

Diese Migration wird als **eigenständiges Projekt** mit eigenem Plan und Risikomanagement behandelt.

---

## 3. Legacy-Migration: Hamburg-zentrische Meta-Keys

Während sich die Änderungsanfrage auf die Weiterentwicklung des Systems konzentriert, erfordert das bestehende System Bereinigung und Migration. Die Codebasis verwendet derzeit Hamburg-zentrische Meta-Keys, die in generische, kammerunabhängige Keys überführt werden sollen.

### 3.1 Umfang
- **Systemweit**: Alle Referenzen auf diese Keys in Felddefinitionen, Programmlogik und Datenbankspeicherung
- **Codebasis**: Aktualisierung der Felddefinitionen und Programmlogik
- **Datenbank**: Migration bestehender `meta_key`-Werte in Post-Meta und zugehörigen Tabellen

### 3.2 Key-Zuordnungen (Beispiele)

| Aktuell (Hamburg-spezifisch) | Ziel (Generisch) |
|------------------------------|------------------|
| `teil_ptkhh_yes`             | `teil_ptk_yes`   |
| `teilnehmer_ptkhh`           | `teilnehmer_ptk` |

*Vor der Migration ist eine vollständige Prüfung aller `ptkhh`- bzw. Hamburg-spezifischen Keys erforderlich.*

### 3.3 Anforderungen
- Identifikation aller Vorkommen Hamburg-zentrischer Keys in der Codebasis
- Aktualisierung der Felddefinitionen und aller Programmlogik, die diese Keys liest/schreibt
- Bereitstellung eines Datenbank-Migrationsskripts zur Umbenennung der `meta_key`-Werte
- Die Migration muss dieselbe Einfrieren/Bereitstellen/Migrieren/Verifizieren/Wiederaufnehmen-Prozedur wie andere systemweite Änderungen befolgen

---

## 4. Rollen & Terminologie

### 4.1 IMB-Mitglied
- Hat ein IMB-Benutzerkonto (`user_id`)
- Berechtigt für automatische Punkteberechnung
- Kann inaktiv werden (nicht gelöscht)

### 4.2 Gast-Teilnehmer (Nicht-IMB)
- Kein IMB-Benutzerkonto
- Externer Teilnehmer (andere Kammern, Ärztekammer etc.)
- Teilnahme wird erfasst, aber keine automatischen IMB-Punkte

### 4.3 IV-Teilnehmerdatensatz
- Repräsentiert eine konkrete Gruppenmitgliedschaft
- Status:
  - `pending` — in Prüfung
  - `approved` — Genehmigt
  - `denied` — abgelehnt
  - `archived` — Archiviert (inaktiv); wird verwendet, wenn der Teilnehmer ausgetreten ist (z. B. IMB-Nutzer wurde inaktiv) und der Koordinator ihn entfernen und ggf. als Gast neu hinzufügen soll
- Die Genehmigung erfolgt **pro IV-Gruppe**, nicht global

### 4.4 Gruppenkoordinator (GK)
- Verwaltet die Gruppenzusammensetzung
- Reicht Teilnehmer zur Genehmigung ein
- **Darf genehmigte Teilnehmer nicht mehr ändern**

### 4.5 Kammer / Kammer-Admin
- Genehmigt Teilnehmer
- Verwaltet Namensänderungen bei Gast-Teilnehmern
- Kann historische Berichte erstellen
- Verantwortlich für Prüfung und Compliance

---

## 5. Teilnehmer-Unveränderbarkeit & Namensänderungen

### 5.1 Allgemeine Regel
Sobald ein Teilnehmer **genehmigt** ist, dürfen Gruppenkoordinatoren **keine Änderungen** mehr vornehmen.

Dies gilt für:
- IMB-Mitglieder
- Gast-Teilnehmer
- alle Teilnehmerdatenfelder

### 5.2 IMB-Mitglieder
- Namensänderungen werden ausschließlich über den **IMB-Namensänderungs-Workflow** abgewickelt.
- Nach Genehmigung:
  - wird der aktualisierte Name überall (aktuelle und historische Ansichten) angezeigt.
- Logs bleiben unverändert und speichern den ursprünglichen Namen zum Zeitpunkt jeder Aktion.

### 5.3 Gast-Teilnehmer
- Namensänderungen dürfen **nur** von folgenden Rollen vorgenommen werden:
  - Kammer / chamber_admin
- Gruppenkoordinatoren müssen Änderungswünsche an die Kammer weiterleiten.
- Namensänderungen:
  - aktualisieren den kanonischen Teilnehmernamen
  - werden rückwirkend in historischen Teilnahmeansichten angezeigt
  - **ändern keine Logs**
- In den Logs erscheinen:
  - der ursprüngliche Name zum Zeitpunkt der Aktion
  - explizite „Namensänderung durch Kammer-Nutzer X“-Einträge

---

## 6. IMB → Nicht-IMB Übergänge

Wenn ein IMB-Mitglied die Kammer verlässt und später als Gast-Teilnehmer mitmacht:

- Es erfolgt **keine Verknüpfung** zwischen dem IMB-Datensatz und dem Gast-Teilnehmer-Datensatz.
- Das System behandelt sie als **zwei getrennte Teilnehmeridentitäten**.
- Die Berichterstattung ist dementsprechend getrennt:
  - IMB-Berichte (punkterelevant)
  - Gast-Teilnehmer-Berichte (teilnahmeorientiert)
- Die Kammer **kann nicht manuell** ein PTK-Mitglied durch Anpassung eines genehmigten Teilnehmers in einen Gast umwandeln.
- Nach Genehmigung ist der Teilnehmerdatensatz **eingefroren**; Details können nicht bearbeitet werden.
- Bei Namensänderung eines PTK-Mitglieds wird dies in IMB aktualisiert und erscheint in IV, da die `user_id` dieselbe ist.
- Der Teilnehmer-**typ** (PTK-Mitglied vs. Gast) kann nicht geändert werden, auch nicht durch Kammer-Admins.
- Offene Frage an PTK-HH (Auftraggeber): Darf der Name eines Gast-Teilnehmers manuell geändert werden?

Diese Trennung ist bewusst und explizit.

---

## 7. Intervisionspunkte & Inaktive Mitglieder

### 7.1 Kernregel
Wenn ein Mitglied inaktiv wird:
- **verschwinden** zuvor erworbene Intervisionspunkte **nicht**
- historische Punkte bleiben:
  - gespeichert
  - prüfbar
  - druckbar

Dies unterstützt Fälle, in denen ehemalige Mitglieder Punktebescheinigungen für eine neue Kammer anfordern.

### 7.2 Berechtigungen
- Nur die **Kammer** darf Punktebescheinigungen (PDFs) für **inaktive Mitglieder** erstellen.

### 7.3 IV-Sitzung-Punkte-Check
Bei der Erstellung einer IV-Sitzung werden automatisch Punkte für jedes PTK-Mitglied generiert und benötigen die Kammer-Genehmigung.

- Prüfung einbauen, dass das Mitglied **nicht** `user_status = inactive` ist, bevor Punkte zugewiesen werden.
- Wenn ein Nutzer inaktiv ist, aber weiterhin als PTK-Mitglied zugeordnet ist, dies im IV-Sitzungs-Log protokollieren.

---

## 8. IV-Gruppenmitgliedschaft Wenn IMB-Nutzer Inaktiv Wird

### 8.1 Für die Zukunft

Wenn ein IMB-Nutzer ein Austrittsdatum erhält und im System auf inaktiv gesetzt wird, stellt sich die Frage, wie mit dessen IV-Gruppenmitgliedschaft umgegangen werden soll.

**Offene Fragen an PTK-HH (Auftraggeber):**

> Dies wirft die Frage auf, wie wir damit künftig umgehen sollen: Wenn ein PTK-Mitglied ein Austrittsdatum erhält und auf inaktiv gesetzt wird, sollte das System diesen Nutzer dann automatisch aus allen IV-Gruppen entfernen? Wie kommunizieren wir das an die Gruppenkoordinatoren? Wäre es sinnvoll, einen neuen Status für IV-Gruppenmitglieder („inaktiv") einzuführen, der den Koordinator informiert, dass der Nutzer entfernt und anschließend als Gast neu hinzugefügt werden muss?

### 8.2 Historische Bereinigung: Bestehende Inaktive Mitglieder in IV-Gruppen

Es gibt einen Rückstand an Mitgliedern, die inzwischen inaktiv geworden sind. Derzeit sind sie entweder:

- manuell in Gäste umgewandelt worden (derzeit von der Kammer erlaubt, wird nach dieser Änderung **gesperrt**), oder
- weiterhin als PTK-Mitglieder in IV-Gruppen gelistet, mit zugewiesenen Punkten usw.

Es wurde ein Hilfsprogramm entwickelt, das alle inaktiven Mitglieder exportiert, die IV-Gruppen als PTK-Mitglied angehören. Die resultierende CSV-Datei kann für eine Bereinigung verwendet werden.

**Offene Frage an PTK-HH (Auftraggeber):**

Können wir einen neuen genehmigten Gast ohne den üblichen Genehmigungsprozess anlegen? Die Zielkammer, der sie beigetreten sind, ist unbekannt, aber es kann eine Spalte in der CSV für manuelle Aktualisierung ergänzt werden; diese angepasste Liste würde dann die Bereinigung steuern. Dieser Ansatz würde ermöglichen, alle besuchten Sitzungen **nach** ihrem Austritt dem neuen Gast-Datensatz zuzuordnen, während die bisherige PTK-Teilnahme erhalten bleibt.

---

## 9. Druck von Teilnahme & Punkten für Gast-Teilnehmer

### 9.1 Aktueller Zustand
- Gruppenkoordinatoren können Berichte für **aktive Gast-Teilnehmer** über das Frontend drucken.
- Kammern können bereits Berichte für **jedes IMB-Mitglied** (aktiv oder inaktiv) drucken.

### 9.2 Erforderliche Erweiterung
- Die **Kammer** soll folgende Berichte erstellen können:
  - Teilnahmeberichte
  - Punkteberichte (wo zutreffend)
  - für **jeden Gast-Teilnehmer** (aktiv oder inaktiv)

### 9.3 Admin-UI Platzierung
- WordPress-Admin → IV-Gruppe bearbeiten
- Metabox ergänzen, die:
  - alle Gast-Teilnehmer auflistet
  - Tabs bereitstellt:
    - Aktiv
    - Inaktiv
  - beim Auswählen eines Teilnehmers einen neuen Tab öffnet
  - Daten an den bestehenden Frontend-Zeitraum-PDF-Generator übergibt

---

## 10. Offene Datenschutzfrage (Ausstehend)

Zur Reduzierung der Kammer-Arbeitslast wurde vorgeschlagen, Gruppenkoordinatoren ebenfalls zu erlauben:

- Teilnahme-/Punkteberichte zu drucken
- für **inaktive Gast-Teilnehmer**
- analog zu dem, was bereits für aktive Gast-Teilnehmer erlaubt ist

Dies wird derzeit vom Auftraggeber im Hinblick auf **deutsche Datenschutzvorschriften** geprüft.

**Status:** Offene Frage – Entscheidung ausstehend.

---

## 11. IV-Sitzungen UI – Listen- & Paginierungsregeln

### 11.1 IV-Gruppen-Details → Sitzungsliste
- Paginierte Liste
- **10 Sitzungen pro Seite**
- Nur Sitzungen aus den **letzten 6 Jahren**

Die Paginierung soll in der GK-Ansicht bereits existieren und muss verifiziert werden.

### 11.2 Sitzungen Älter Als 6 Jahre
**Offene Frage (Auftraggeber prüft):**
- Sollen Sitzungen älter als 6 Jahre:
  - in der UI ausgeblendet, aber im System behalten werden, oder
  - vollständig aus dem System entfernt werden?

### 11.3 Teilnehmeransicht
- Dieselben Regeln müssen gelten:
  - paginiert
  - 10 pro Seite
  - max. 6 Jahre
- Dies ist **derzeit nicht implementiert** und muss ergänzt werden.

---

## 12. Logs vs. Angezeigte Daten

- Logs sind **unveränderlich** und spiegeln stets wider:
  - den Zustand zum Zeitpunkt der Aktion
- UI-Ansichten, Berichte und Druckausgaben:
  - verwenden stets den **aktuellen kanonischen Namen**
- Dies gewährleistet:
  - Prüfkorrektheit
  - rechtliche Nachvollziehbarkeit
  - saubere nutzerorientierte Dokumente

---

## 13. Zusammenfassung Offener Fragen

1. Dürfen Gruppenkoordinatoren Berichte für **inaktive Gast-Teilnehmer** drucken (Datenschutz)?
2. Sollen IV-Sitzungen älter als 6 Jahre **ausgeblendet** oder **gelöscht** werden?
3. Endgültige Bestätigung, dass die systemweite Timestamp-Migration als **eigenständiges Projekt** mit eigenem Rollout behandelt wird.
4. Wenn ein IMB-Nutzer inaktiv wird (Austrittsdatum gesetzt): Soll das System diesen **automatisch** aus IV-Gruppen entfernen oder einen **„inaktiv"**-Status für Gruppenmitglieder einführen? Wie soll diese Änderung und erforderliche Maßnahmen den Gruppenkoordinatoren mitgeteilt werden?
5. **Historische Bereinigung**: Für bestehende inaktive Mitglieder, die weiterhin als PTK-Mitglieder in IV-Gruppen gelistet sind: Können wir genehmigte Gäste ohne den üblichen Genehmigungsprozess anlegen? Hierfür würde eine CSV bestehender inaktiver Nutzer, die IV-Gruppen als PTK-Mitglied angehören (mit manuell ausgefüllter Spalte für die Kammer), die Bereinigung steuern sowie Sitzungen nach Austritt dem neuen Gast zuordnen, während die bisherige PTK-Teilnahme erhalten bleibt.

---

## 14. Nächste Schritte

1. Auftraggeber-Bestätigung der offenen Fragen
2. Eigenes Konzept und Plan für die systemweite Timestamp-Migration
3. Vollständige Prüfung der Hamburg-zentrischen Meta-Keys (`ptkhh` etc.) und Migrationsplan
4. Implementierung der IV- und UI-bezogenen Änderungen auf Basis des neuen Datumsmodells
5. Produktiv-Rollout gemäß Einfrieren/Migrieren/Testen/Wiederaufnehmen-Prozedur (inkl. Meta-Key-Migration)

---
