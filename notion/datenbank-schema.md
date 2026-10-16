# Notion-Datenbanken – Schema

Der Pilot nutzt zwei Notion-Datenbanken. Beide müssen mit der Notion-Integration verbunden sein (*… → Verbindungen*), sonst findet n8n sie nicht.

## 1. Wissensdatenbank IT Service Desk

Enthält geprüfte Lösungsartikel. n8n liest **nur Artikel mit Status „freigegeben“**; neue Lösungen aus dem First- oder Second-Level-Support werden als **„Entwurf“** angelegt und erst nach fachlicher Prüfung freigegeben.

| Eigenschaft | Notion-Typ | Werte / Format | Zweck |
|---|---|---|---|
| Titel | Title | Freitext | Kurzes Problem in den Worten der Anfragenden |
| Kategorie | Select | Passwort-Selbsthilfe, M365, Drucker/Scanner, Standardsoftware, How-to, außerhalb Pilot | Pilotkategorien; „außerhalb Pilot“ für Gegenproben |
| Status | Text | Entwurf, in Prüfung, freigegeben, veraltet | Nur „freigegeben“ wird von n8n genutzt (im Workflow als Text-Feld gefiltert bzw. geschrieben) |
| Schlagwörter | Multi-select | z. B. Passwort, Outlook, Scanner | Grundlage der Top-3-Suche im Code-Node |
| Fehlerbild | Text | Freitext | Was die anfragende Person sieht |
| Lösung (Schritte) | Text | Nummerierte Schritte | Basis für den KI-Antwortvorschlag |
| Gilt für | Text | Systeme, Standorte, Zielgruppe | Geltungsbereich |
| Eskalation wenn | Text | Freitext | Wann die Selbsthilfe endet und wer übernimmt |
| Selbsthilfe geeignet | Checkbox | Ja / Nein | Lösbar ohne Adminrechte? |
| Sensible Inhalte | Checkbox | muss „Nein“ sein | Artikel mit sensiblen Inhalten sind vom KI-Pfad ausgeschlossen |
| Owner | Text / Person | Name (Rolle) | Inhaltlich verantwortlich |
| Fachlich geprüft von | Text / Person | Name (Rolle) | Freigabe nach fachlicher Prüfung |
| Prüfdatum | Date | JJJJ-MM-TT | Letzte Prüfung |
| Nächste Prüfung | Date | Prüfdatum + 12 Monate | Wiedervorlage |
| Ursprungsticket | Text | z. B. INC-2026-0412 / PILOT-123 | Ticketnummer, **keine Personennamen** |

Im Workflow wird die Datenbank über den Platzhalter `NOTION_DATENBANK_ID_WISSENSDATENBANK` angesprochen (Nodes *Get many database pages* und *Create a database page – Entwurf Artikel*).

Testdaten: [`wissensdatenbank-testeintraege.csv`](wissensdatenbank-testeintraege.csv) – 3 freigegebene Artikel und 1 Entwurf als Gegenprobe (CNC-Fehler F739, muss „kein Treffer“ ergeben). Import über *Notion → Import → CSV*; danach die Typen der Spalten wie oben einstellen.

## 2. Offene Freigaben

Arbeitsliste für das Support-Team. Für jeden menschlichen Schritt legt n8n einen Eintrag mit dem Link zum wartenden Formular (Wait-Node) an. So kommt der Ablauf weitgehend ohne E-Mail aus.

Im Workflow über den Platzhalter `NOTION_DATENBANK_ID_OFFENE_FREIGABEN` angesprochen (drei Nodes *Create a database page – offene Freigaben / First Level Support / Second Level Support* im Hauptworkflow sowie *Create a database page – Störung* im Fehler-Workflow).

| Eigenschaft | Notion-Typ | Werte / Format | Zweck |
|---|---|---|---|
| Ticket ID | Title | `PILOT-<Ausführungs-ID>` | Eindeutige Kennung je Anfrage |
| Status | Select | offen, erledigt | n8n setzt „offen“; „erledigt“ wird im Pilot noch manuell gesetzt |
| Art | Select | Freigabe, Lösung First Level, Lösung Second Level, Störung | Welcher Schritt wartet; „Störung“ kommt aus dem Fehler-Workflow |
| Problem | Text | Kategorie / Kurzbeschreibung | Überblick ohne Personennamen |
| Formular-Link | URL | Resume-URL des Wait-Nodes bzw. Link zur Ausführung | Öffnet das Freigabe- bzw. Lösungsformular; bei „Störung“ die fehlgeschlagene Ausführung in n8n |
| erstellt am | Date | Eingangszeitpunkt der Anfrage | Wird von n8n aus dem Formular übernommen |

> **Hinweis:** Der Formular-Link funktioniert nur, solange die zugehörige n8n-Ausführung wartet. Im Pilot liefen die Links nur lokal (`localhost`); für den Einsatz im Netz muss n8n unter einer erreichbaren Adresse laufen (`WEBHOOK_URL`).

## Datenschutz

- In beiden Datenbanken keine Namen oder E-Mail-Adressen von Anfragenden – nur Ticket-IDs.
- Keine Auswertung je Person, nur Team-Kennzahlen.
