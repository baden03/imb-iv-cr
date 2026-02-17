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
Sobald ein Teilnehmer **genehmigt** ist, dürfen Gruppenkoordinatoren **keine Änderungen** mehr vornehmen — mit einer Ausnahme: Das **Kammer**-Feld (Zielkammer) darf von Kammer-Admins und Gruppenkoordinatoren aktualisiert werden, wenn es leer gelassen wurde (z. B. nach automatischer PTK→Gast-Umwandlung).

Dies gilt für:
- IMB-Mitglieder
- Gast-Teilnehmer
- alle übrigen Teilnehmerdatenfelder

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
- Nach Genehmigung ist der Teilnehmerdatensatz **eingefroren**; Details können nicht bearbeitet werden (außer dem Kammer-Feld, wenn es nach automatischer PTK→Gast-Umwandlung leer gelassen wurde — siehe §8.1).
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

### 8.1 Automatische Verarbeitung (Vereinbart)

Wenn ein PTK-Nutzer mit Austrittsdatum (Pflichtangabe) als inaktiv markiert wird, führt das System **automatisch** Folgendes aus:

1. **`valid_to` aktualisieren** in der Tabelle `group_membership` für alle IV-Gruppen, denen der Nutzer angehört — setzen auf das Austrittsdatum (Timestamp).
2. **Neuen Eintrag erstellen** in der Tabelle `group_membership` für jede `group_id`, der der inaktive Nutzer angehört: `valid_from` = heute (Timestamp), `valid_to` = 20 Jahre in der Zukunft (oder der verwendete Timestamp für „weit in der Zukunft“).
3. **Nutzer entfernen** (nicht als inaktiv markieren) vollständig aus allen `group_member_details` aller IV-Gruppen.
4. **Neuen Gast-Teilnehmer erstellen** in den `group_member_details` jeder Gruppe mit:
   - `teilnehmer_name` — vom ehemaligen PTK-Mitglied
   - `teilnehmer_ptkhh` (wird zu `teilnehmer_ptk` umbenannt) — nein (Gast, kein PTK-Mitglied)
   - `teil_kammer_other` — **leer** (vom Gruppenkoordinator auszufüllen)
   - `teilnehmer_type` — vom alten PTK-Mitglied übernommen
   - `teil_appurkunde` — leer (bereits als PTK-Mitglied genehmigt)
   - `status` — approved

5. **Postmeta aktualisieren** für alle `intervision_meeting`-Beiträge, die **nach** dem Austrittsdatum des Nutzers stattgefunden haben, wo `meta_key` = `'attended'` und `meta_value` = die inaktive `user_id`: die `user_id` durch Vor- und Nachname ersetzen (als neuer `member_key` in `group_membership`).
6. **Punkte löschen**, die dem Nutzer zugewiesen wurden und **nach** dem Austrittsdatum erstellt wurden.

Das **Kammer**-Feld (`teil_kammer_other`) ist das **einzige** Feld, das von Kammer-Admins und Gruppenkoordinatoren für einen genehmigten Teilnehmer geändert werden darf, aber nur wenn es leer ist. Alle übrigen genehmigten Teilnehmerfelder bleiben eingefroren.

### 8.2 Historische Bereinigung: Bestehende Inaktive Mitglieder in IV-Gruppen

Es gibt einen Rückstand an Mitgliedern, die inzwischen inaktiv geworden sind. Derzeit sind sie entweder:

- manuell in Gäste umgewandelt worden (derzeit von der Kammer erlaubt, wird nach dieser Änderung **gesperrt**), oder
- weiterhin als PTK-Mitglieder in IV-Gruppen gelistet, mit zugewiesenen Punkten usw.

Es wurde ein Hilfsprogramm entwickelt, das alle inaktiven Mitglieder exportiert, die IV-Gruppen als PTK-Mitglied angehören. Die resultierende CSV-Datei kann für eine Bereinigung verwendet werden.

**Offene Frage an PTK-HH (Auftraggeber):**

Bei der Bereinigung ist die Zielkammer, der sie beigetreten sind, unbekannt; es kann jedoch eine Spalte in der CSV für manuelle Aktualisierung ergänzt werden. Diese angepasste Liste würde dann die Bereinigung steuern. Dieser Ansatz würde ermöglichen, alle besuchten Sitzungen **nach** ihrem Austritt dem neuen Gast-Datensatz zuzuordnen, während die bisherige PTK-Teilnahme erhalten bleibt.

### 8.3 PTK-Mitglied Als Gruppenkoordinator Tretend

**Offene Frage an PTK-HH (Auftraggeber):**

Was geschieht, wenn das austretende PTK-Mitglied ein Gruppenkoordinator ist? Die Möglichkeit, einen GK als inaktiv zu markieren, soll nur erlaubt sein, wenn er bereits seine GK-Rolle an ein anderes Mitglied übertragen hat.

**Vorgeschlagene Implementierung (vorläufig):** Vor dem Erlauben, den Nutzer als inaktiv zu markieren, immer prüfen, ob der Nutzer offene IV-Gruppen besitzt. Wenn der Nutzer offene IV-Gruppen besitzt, die Aktion blockieren, bis das Eigentum übertragen wurde.

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
4. ~~Wenn ein IMB-Nutzer inaktiv wird~~ **Gelöst**: Automatische Verarbeitung gemäß §8.1 (Entfernung aus Gruppen, Erstellung Gast, Kammer-Feld bearbeitbar).
5. **Historische Bereinigung**: Für bestehende inaktive Mitglieder, die weiterhin als PTK-Mitglieder in IV-Gruppen gelistet sind: Können wir genehmigte Gäste ohne den üblichen Genehmigungsprozess anlegen? Ja. Hierfür würde eine CSV bestehender inaktiver Nutzer, die IV-Gruppen als PTK-Mitglied angehören (mit manuell ausgefüllter Spalte für die Kammer), die Bereinigung steuern sowie Sitzungen nach Austritt dem neuen Gast zuordnen, während die bisherige PTK-Teilnahme erhalten bleibt.
6. **PTK-Mitglied als Gruppenkoordinator austretend**: Was geschieht, wenn das austretende PTK-Mitglied ein Gruppenkoordinator ist? Vorschlag: Markierung als inaktiv blockieren, bis das Eigentum an offenen IV-Gruppen an einen anderen Nutzer übertragen wurde. Beim Auftraggeber bestätigen.

---

## 14. Nächste Schritte

1. Auftraggeber-Bestätigung der offenen Fragen
2. Eigenes Konzept und Plan für die systemweite Timestamp-Migration
3. Vollständige Prüfung der Hamburg-zentrischen Meta-Keys (`ptkhh` etc.) und Migrationsplan
4. Implementierung der IV- und UI-bezogenen Änderungen auf Basis des neuen Datumsmodells
5. Produktiv-Rollout gemäß Einfrieren/Migrieren/Testen/Wiederaufnehmen-Prozedur (inkl. Meta-Key-Migration)

---
