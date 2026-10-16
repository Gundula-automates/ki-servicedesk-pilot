# Testprotokoll Pilot-Workflow

| | |
|---|---|
| **Datum** | 06.10.2026 |
| **Testumgebung** | n8n lokal, Notion-Testdatenbank, Mail-Versand deaktiviert |
| **Getestet von** | KI Integration Manager (Rolle) |
| **Testdaten** | 3 Artikel freigegeben, 1 Entwurf; fiktive Anfragende, keine Echtdaten |

## Testfälle

| Nr. | Testfall | Eingabe | Erwarteter Pfad | Erwartetes Ergebnis | Tatsächliches Ergebnis | Status |
|---|---|---|---|---|---|---|
| 1 | Treffer + Freigabe | „scan nicht in Mail“; Freigabe: freigeben | Notion-Suche → Code (Top 3) → KI-Vorschlag → Freigabe-Formular | KI-Antwort; Eintrag in „Offene Freigaben“ | wie erwartet | ✓ ok |
| 2 | Vorschlag abgelehnt | wie Nr. 1; Freigabe: ablehnen | Freigabe-Formular → IF freigegeben (nein) → Lösungs-Formular First Level | Fall landet bei First Level, keine KI-Antwort | wie erwartet, neuer Teilpfad erkannt | ! Nacharbeit |
| 3 | Kein Treffer, First Level löst | „Kaffeemaschine kaputt“; gelöst = ja | IF Treffer (nein) → Lösungs-Formular First Level → IF gelöst (ja) | Mail mit Bearbeitungsschritten; Entwurf in Wissensdatenbank | Pfad ok, Entwurf Artikel funktioniert nicht | ! Nacharbeit |
| 4 | Weiterleitung Second Level | wie Nr. 3; gelöst = nein | IF gelöst (nein) → Lösungs-Formular Second Level | Second Level sieht Schritte von First Level; Mail; Entwurf | Pfad ok, Entwurf Artikel fehlerhaft, Schritte nicht übergeben | ! Nacharbeit |
| 5 | Gegenprobe Entwurf | „CNC F739“ (Artikel nur als Entwurf) | Notion findet nur freigegebene Artikel → kein Treffer | Kein KI-Vorschlag, Fall geht an First Level | wie erwartet | ✓ ok |
| 6 | Sensible Anfrage | Beschreibung enthält „Gehalt“ | IF sensibel (ja) → vertrauliche Mail | Mail an Service Desk; keine KI, kein Notion-Eintrag | wie erwartet | ✓ ok |

**Ergebnis der ersten Testrunde:** 3 von 6 Tests bestanden, 3 mit Nacharbeit.

### Gefundene Probleme

| Nr. | Problem | Geplante Lösung |
|---|---|---|
| 2 | „Entwurf Artikel“ bricht ab, wenn die Lösung nicht als Artikel geeignet ist | Pfad „nicht als Artikel geeignet“ ergänzen |
| 3 | Fehler im Node „Entwurf Artikel“ | Node überarbeiten |
| 4 | Übergabe First → Second Level; Node „Entwurf Artikel“ | Übergabe und Node überarbeiten |

## Fehlerbehebung

### 1 · Node „Entwurf Artikel“ repariert – ✓ behoben

- **Problem:** Der Node *Create database page – Entwurf Artikel* brach in Test 3 und 4 ab.
- **Ursache und Lösung:** Fehlerhafte Syntax im Ausdruck – korrigiert.
- **Ergebnis:** Der Pfad läuft korrekt durch; der Artikelentwurf landet mit Status „Entwurf“ in der Wissensdatenbank (Beispiel: neuer Artikel „Display aus“).

### 2 · Übergabe an Second Level und Artikel-Filter – ✓ behoben

- **Befund 1 – Übergabe First → Second Level:** Funktioniert. Im ersten Test waren lediglich keine Arbeitsschritte eingetragen.
- **Befund 2 – Artikel trotz „nicht geeignet“:** Die Lösung wurde ohne Prüfung als Artikelentwurf angelegt (Beispiel: Lösung aus PILOT-291).
- **Lösung:** IF-Node „Artikel geeignet?“ vor dem Anlegen des Entwurfs – Test erfolgreich.

**Ergebnis nach Fehlerbehebung:** 3 Fehler gefunden und behoben; die betroffenen Pfade laufen korrekt durch.

## Nachtest nach Code-Review (15.10.2026)

Bei der Vorbereitung für GitHub fielen drei weitere Fehler auf. Sie wurden behoben, der Fehler-Workflow wurde ergänzt und alles erneut getestet (Production-URL, Notion verbunden, Mail-Versand deaktiviert).

| Problem | Korrektur |
|---|---|
| Sensibel-Filter enthielt `hr` und traf damit auch „mehr“, „Uhr“, „fährt“ | Begriff entfernt |
| IF „als Artikel geeignet?“ hatte eine leere zweite Bedingung (ODER) und war damit immer wahr | Leere Bedingung gelöscht |
| Mail-Nodes First/Second Level verwiesen auf einen nicht vorhandenen Node; Second Level hätte den Text von First Level verschickt | Verweis auf `Wait1` bzw. `Wait2`, Feld „Lösung“ |

| Nr. | Testfall | Eingabe | Erwartetes Ergebnis | Status |
|---|---|---|---|---|
| A | Kein Fehlalarm im Sensibel-Filter | „Drucker druckt nicht mehr“ | nicht sensibel → Notion-Suche und KI-Pfad | ✓ ok |
| B | Sensible Anfrage | „Frage zu meiner Gehaltsabrechnung“ | vertraulicher Pfad, keine KI, kein Notion-Eintrag | ✓ ok |
| C | Lösung nicht als Artikel geeignet | „Kaffeemaschine kaputt“; First Level: gelöst, „nicht geeignet“ | kein Artikelentwurf in der Wissensdatenbank | ✓ ok |
| D | Fehler-Workflow | Testfehler im Code-Node, Anfrage über Production-URL | Eintrag in „Offene Freigaben“ mit Art „Störung“ | ✓ ok |

Hinweis: Der Fehler-Workflow startet nur bei Produktions-Ausführungen, nicht bei Tests im Editor.

## Offene Punkte

- Formular-Links funktionieren nur lokal (`localhost`)
- Status „erledigt“ in „Offene Freigaben“ wird noch manuell gesetzt
- Mail-Eingang nur konzeptionell umgesetzt (Email Trigger deaktiviert)
- Mail-Versand deaktiviert, kein Postfach angebunden
