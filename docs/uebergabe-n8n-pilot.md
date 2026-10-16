# Übergabe: n8n-Pilot IT-Helpdesk (Übung, fiktives Unternehmen)

> **Hinweis:** Dieses Dokument zeigt den **Planungsstand vor dem Bau** des Workflows. In der finalen Umsetzung wurden u. a. die Notion-Suche (alle freigegebenen Artikel + Code-Node Top 3 per Schlagwort), das Sprachmodell (AI Agent mit Groq) und die menschlichen Schritte (Wait-Nodes mit Formular, Link in „Offene Freigaben“) anders gelöst – siehe [README](../README.md) und [Testprotokoll](testprotokoll.md).

## Ziel
Funktionsfähiger n8n-Prototyp: Anfrage per Formular (optional E-Mail) → Suche in Notion → bei Treffer freizugebender Antwortvorschlag, sonst strukturierte Mail an First-Level-Support → ggf. Second Level → Dokumentation in Notion. Fallback: manueller E-Mail-Prozess.

## Stand
- Notion-Datenbank „Wissensdatenbank IT Service Desk“ angelegt (CSV-Import). Spalten: Titel, Kategorie, Status, Schlagwörter, Fehlerbild, Lösung (Schritte), Gilt für, Eskalation wenn, Selbsthilfe geeignet, Sensible Inhalte, Owner, Fachlich geprüft von, Prüfdatum, Nächste Prüfung, Ursprungsticket.
- 3 freigegebene Einträge: Passwort/Konto gesperrt (Passwort-Selbsthilfe), Outlook „Getrennt“ (M365), Scan-to-Mail kommt nicht an (Drucker/Scanner). 1 Gegenprobe: CNC-Fehler F739, Status „Entwurf“ → muss „kein Treffer“ ergeben.
- n8n Form Trigger läuft. Formularfelder: Produktionskritisch (Mehrfachauswahl, Array), Standort, Gerät, Problem, Fehlernummer, Problembeschreibung, Mailadresse, Name, Abteilung, Datum (+ submittedAt, formMode).
- Edit Fields (Formular) funktioniert (Mode JSON, Feld auf Expression, Ausdruck unten).
- Offen: E-Mail-Trigger. Bei Microsoft 365 → Microsoft Outlook Trigger (OAuth); falls „Admin-Genehmigung erforderlich“ → eigenes Test-Postfach (z. B. Gmail Trigger oder IMAP mit App-Passwort).

## Node-Reihenfolge
1. n8n Form Trigger („On form submission“) | optional: Microsoft Outlook Trigger / Gmail Trigger / Email Trigger (IMAP)
2. Edit Fields (Set) – je Eingang einer, gleiche Feldnamen
3. If – sensibel? (true → direkt Mail an Service Desk, nicht in KI-Pfad); optional If produktionskritisch → Schichtleitung/Hotline
4. Notion – „Get many database pages“ (o. ä., Aktion MIT Property-Filtern, nicht „Search“): Status equals freigegeben, Kategorie equals {{ $json.problem }}, Limit 3, Settings → Always Output Data = an
5. If – Treffer? ({{ $json.id }} is not empty)
   - Treffer: Basic LLM Chain + OpenAI Chat Model (oder Edit Fields) → Antwortvorschlag → Microsoft Outlook/Gmail/Send Email „Send and Wait for Response“ (Approval) an First Level → If approved → Send an Anfragende → Notion Create (Protokoll, keine Namen)
   - Kein Treffer: Send and Wait for Response (Custom Form: gelöst ja/nein, Lösungsschritte) an First Level → If gelöst → Send an Anfragende → Notion Create (neuer Eintrag, Status Entwurf); nein → Send and Wait an Second Level → Send an Anfragende → Notion Create
6. Fallback: eigener Workflow mit Error Trigger → Mail an Service-Desk-Postfach; im Hauptworkflow Settings → Error Workflow eintragen.

## Edit Fields – Formular (Mode JSON, Expression)
```
{{
{
  "ticket_id": "PILOT-" + $execution.id,
  "eingang": $json["submittedAt"],
  "datum": $json["Datum"] ?? "",
  "kanal": "Formular",
  "testmodus": $json["formMode"] === "test",
  "name": $json["Name"] ?? "",
  "email": $json["Mailadresse"] ?? "",
  "abteilung": $json["Abteilung"] ?? "",
  "standort": $json["Standort"] ?? "",
  "geraet": $json["Gerät"] ?? "",
  "problem": $json["Problem"] ?? "",
  "fehlernummer": $json["Fehlernummer"] ?? "",
  "beschreibung": $json["Problembeschreibung"] ?? "",
  "produktionskritisch_auswahl": ($json["Produktionskritisch"] ?? []).join(", "),
  "produktionskritisch": /produktion|maschine|anlage|steht/i.test(($json["Produktionskritisch"] ?? []).join(" ")),
  "suchtext": [$json["Problem"], $json["Fehlernummer"], $json["Gerät"], $json["Problembeschreibung"]].filter(x => x).join(" ").toLowerCase().trim(),
  "sensibel": /kündig|gehalt|lohn|krank|abmahnung|betriebsrat|personalakte|bewerbung/i.test(($json["Problem"] ?? "") + " " + ($json["Problembeschreibung"] ?? ""))
}
}}
```

## Wichtige Hinweise
- Formularfeld „Problem“ als Dropdown mit exakt den Notion-Kategorien (Passwort-Selbsthilfe, M365, Drucker/Scanner, Standardsoftware, How-to, Sonstiges), sonst greift der Notion-Filter nicht.
- Notion-Integration (notion.so/my-integrations) muss per „… → Verbindungen“ mit der Datenbank verbunden sein.
- Mail-Trigger: Action „Mark as Read“, eigenes Test-Postfach, Schleifen vermeiden (Antworten nicht aus demselben Postfach senden).
- Keine ungeprüften KI-Antworten, keine sensiblen Tickets im KI-Pfad, Logs ohne Namen (pseudonymisiert), keine Auswertung je Person.
- Erst Nodes 1–5 bauen und testen (Execute step), dann die Zweige anhängen.
