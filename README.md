# engineering-hospitation

Hospitationsaufgabe bei NoscAI: **Laborbefunde (LDT & PDF) → strukturierte, geprüfte Laborwerte**

Dauer: **3 Tage** · Schwerpunkt: **AI-Automatisierung & Evaluation** (Full Stack nur als schlanke Review-Oberfläche)

---

## Kontext

ClinicOS ist ein Praxisverwaltungssystem für Arztpraxen. Laborbefunde kommen in der Praxis auf zwei Wegen an:

1. **LDT** (Labordatentransfer): der KBV-Standard für den elektronischen Austausch zwischen Labor und Praxis. Das ist
   ein zeilenbasiertes Textformat, in dem jedes Feld über eine vierstellige **Feldkennung** identifiziert wird. Oft
   hängt der gedruckte Befund zusätzlich als **eingebettetes PDF** in der LDT-Datei.
2. **PDF / Scan**: von Laboren ohne LDT-Anbindung, per Fax, E-Mail oder vom Patienten mitgebracht.

In der Praxis ist LDT der Standard, aber keineswegs sauber: Labore nutzen eigene Testkürzel, Einheiten und
Referenzbereiche werden unterschiedlich angegeben, und manche Felder stehen dort, wo sie laut Spezifikation nicht
hingehören. Bei PDFs gibt es gar keine Struktur.

Ein LLM kann solche Befunde gut lesen. Aber ein falsch extrahierter oder erfundener Laborwert ist in der Medizin
schlimmer als gar keiner. Die eigentliche Aufgabe ist deshalb nicht „LLM aufrufen“. Gebaut werden soll eine Pipeline,
der man **messbar vertrauen** kann und die unsichere Fälle an einen Menschen weitergibt.

## Was im Repo liegt

| Pfad | Inhalt |
|------|--------|
| `docs/LDT_3.2.15_Spezifikation.pdf` | Die offizielle KBV-Spezifikation LDT 3.2.15: Satzarten, Feldkennungen, Regeln |
| `data/ldt/kbv-testdaten/*.ldt` | Offizielle KBV-Testdateien mit fiktiven Patienten (Musterpatient/-in), die meisten mit eingebettetem Befund-PDF |

Die KBV-Testdateien sind klein: wenige Tests pro Befund, und nicht jeder Befund enthält überhaupt Laborwerte (z. B.
Zytologie, Molekulargenetik). Sie zeigen, **wie die Realität aussieht**, reichen aber nicht als Eval-Datensatz. Den
baust du selbst (siehe Tag 1).

## Die Aufgabe

Baue eine kleine Pipeline, die

1. **LDT-Dateien** deterministisch parst und die Laborwerte sowie das eingebettete PDF herauszieht,
2. aus **PDF-Befunden ohne LDT** die Laborwerte per LLM extrahiert,
3. alle Werte **normalisiert** (Analyt → LOINC-Code, Einheiten vereinheitlicht),
4. sie **plausibilisiert** (Regeln, Referenzbereiche, unmögliche Werte, Widersprüche LDT ↔ PDF),
5. die eigene Qualität gegen einen Gold-Datensatz **misst**, und
6. einer Ärztin oder einem Arzt die Ergebnisse zur **Bestätigung oder Korrektur** vorlegt.

Der Kniff: Ein LDT mit eingebettetem PDF ist ein **natürliches Eval-Paar**. Die strukturierten LDT-Werte sind die
Wahrheit, das PDF ist das, was ein Modell ohne LDT zu sehen bekäme.

---

## Tag 1: LDT verstehen, Datensatz bauen

**LDT-Parser (deterministisch, kein LLM)**

- Lies die Spezifikation. Relevant sind vor allem der Zeilenaufbau (Länge + Feldkennung + Inhalt), die Satzarten und
  die Felder für Test-Ident, Testbezeichnung, Ergebniswert, Einheit, Normalwerte und Grenzwertindikator sowie die
  Felder für eingebettete Anhänge.
- Lies Felder **über ihre Feldkennung**, nicht über ihre Position oder Beschriftung. Achte auf die Zeichenkodierung.
- Extrahiere das eingebettete PDF aus den KBV-Testdateien.
- Felder, die laut Spezifikation an einer Stelle nicht vorkommen dürfen, sind ein Fehler des Labors. Nicht raten,
  sondern melden.

**Synthetischer Datensatz**

- Erzeuge 15–20 Befunde, und zwar **jeweils als LDT und als dazu passendes PDF**. Keine echten Patientendaten.
- Tipp: Erzeuge zuerst die Gold-Daten (strukturiert, z. B. zufällige Werte je Analyt) und **rendere daraus** sowohl
  das LDT als auch das PDF (über mehrere Layout-Vorlagen). Dann stimmen alle drei per Konstruktion überein. Ein LLM
  darf beim Generieren höchstens für Variation sorgen (z. B. Freitext-Kommentare). Die Werte selbst sollen nicht vom
  LLM kommen.
- Baue bewusst Variation und Fallen ein, z. B.:
  - verschiedene PDF-Layouts (Tabelle, Fließtext, zweispaltig), mehrseitige Befunde
  - labor-eigene Kürzel und Synonyme: „Krea“ / „Kreatinin“, „HbA1c“ / „Hämoglobin A1c“, „GPT“ / „ALT“
  - Einheiten-Varianten: mg/dl vs. µmol/l, mmol/l vs. mg/dl, % vs. mmol/mol
  - fehlende Referenzbereiche, Grenzwertindikatoren (`H` / `L` / `+`), Werte mit `<` / `>`, deutsches Dezimalkomma
  - Freitext, der wie ein Wert aussieht, aber keiner ist
  - ein paar LDTs mit typischen Laborfehlern (Feld an falscher Stelle, fehlende Einheit)
- Optional: 2–3 PDFs zusätzlich „verschmutzen“ (als Bild rendern, leicht drehen, Rauschen), um einen Scan zu simulieren.

## Tag 2: PDF-Extraktion, Normalisierung & Eval

**PDF-Extraktion**

- Definiere ein Pydantic-Schema für einen Laborwert (mindestens: Analyt wie gedruckt, Wert, Einheit, Referenzbereich,
  Flag, Seite, Confidence bzw. Begründung). Nutze dasselbe Schema für LDT- und PDF-Ergebnisse.
- Extrahiere per LLM mit **Structured Output**. Das Modell darf nichts erfinden: fehlende Angaben bleiben leer.

**Normalisierung**

- Mapping Analyt / labor-eigenes Kürzel → LOINC-Code und Einheitenumrechnung auf eine Zieleinheit pro Analyt.
- Entscheide bewusst, **was deterministisch** gelöst werden sollte und **wo ein LLM** sinnvoll ist (z. B. nur als
  Fallback für unbekannte Kürzel). Begründe die Entscheidung im README.

**Plausibilitätsregeln** (Beispiele, erweiterbar)

- Wert außerhalb des Referenzbereichs: Passt das vom Labor gesetzte Flag dazu?
- Physiologisch unmöglicher Wert (z. B. Natrium 1400 mmol/l, negativer Wert)
- Einheit passt nicht zum Analyten
- LDT und eingebettetes PDF widersprechen sich

**Evaluation**

- Ein Eval-Skript, das die PDF-Extraktion gegen die Gold-Daten laufen lässt und pro Feld **Precision / Recall**
  (bzw. Exact Match für Wert + Einheit) ausgibt, gesamt und pro Befund. Genauso für die Normalisierung (richtiger
  LOINC-Code, richtige Zieleinheit).
- Vergleiche **mindestens zwei Varianten** (z. B. zwei Prompts, zwei Modelle, Text-PDF vs. Bild-Input) und dokumentiere
  das Ergebnis mit Zahlen.
- Eine kurze **Fehleranalyse**: Welche Fehlerklassen treten auf, und warum?

## Tag 3: Human-in-the-Loop & Demo

- Ein **FastAPI-Endpoint**, der eine LDT- oder PDF-Datei annimmt und die geprüften Werte zurückgibt.
- Eine kleine **Review-Oberfläche** (Streamlit reicht völlig; React ist Bonus, kein Muss):
  - Befund-PDF und extrahierte Werte nebeneinander, Quelle sichtbar (LDT oder LLM)
  - Werte mit niedriger Confidence oder verletzter Regel sind hervorgehoben
  - Werte können bestätigt oder korrigiert werden
  - Korrekturen werden gespeichert und können als **neue Eval-Fälle** in den Gold-Datensatz übernommen werden
- **Demo (15 Min.)**: Pipeline live mit einem KBV-Testbefund und einem eigenen PDF, Eval-Zahlen, die
  interessantesten Fehler und was du als Nächstes tun würdest.

---

## Deliverables

- Code in diesem Repo (Feature-Branch + Pull Request auf `main`)
- `README`-Abschnitt „Ergebnisse“: Setup in ≤ 5 Befehlen, Architektur-Skizze, Eval-Tabelle, Designentscheidungen,
  bekannte Grenzen
- Datensatz (LDT + PDF + Gold-JSON) im Repo
- Demo am Ende von Tag 3

## Worauf wir achten

Wichtiger als die Anzahl der Features:

| Kriterium | Frage |
|-----------|-------|
| **Ehrliche Evaluation** | Gibt es ein Eval, dem man glaubt? Weißt du, wo das Modell scheitert? |
| **LLM vs. deterministisch** | Wird alles, was sich mit Regeln lösen lässt (LDT-Parsing, Einheitenumrechnung), auch mit Regeln gelöst? |
| **Umgang mit Unsicherheit** | Keine erfundenen Werte; unsichere Fälle landen beim Menschen statt stillschweigend im System |
| **Standards ernst nehmen** | Wird LDT nach Spezifikation gelesen, und werden Abweichungen gemeldet statt „repariert“? |
| **Datensatzdesign** | Testet der Datensatz die schweren Fälle oder nur den Happy Path? |
| **Code & Kommunikation** | Lesbarer Code, sinnvolle Struktur, klares README, gute Demo |

Nicht alles muss fertig werden. Eine saubere, gemessene Pipeline mit einfacher UI schlägt eine halbfertige
Feature-Sammlung. Wenn du priorisieren musst: **Tag 1 und 2 sind der Kern.**

## Rahmen

- **Nur synthetische bzw. KBV-Testdaten.** Keine echten Befunde, keine echten Patientendaten, auch nicht als Beispiel.
- Sprache: Python (empfohlen: `uv`, FastAPI, Pydantic). Freie Wahl der Libraries.
- LLM-Zugang (API-Key und Modell) stellen wir bereit. Keys gehören in eine `.env` und werden **niemals committet**
  (`.env` steht in `.gitignore`, Vorlage: `.env.example`).
- AI-Coding-Tools (Claude Code, Cursor, Copilot, …) sind ausdrücklich erlaubt. Du solltest aber jede Zeile erklären
  können.
- Fragen stellen ist erwünscht. Rückfragen zu Anforderungen gehören zum Job.

## Quellen

- LDT-Spezifikation und Testdateien: Kassenärztliche Bundesvereinigung (KBV), [update.kbv.de](https://update.kbv.de)
- LOINC: [loinc.org](https://loinc.org)

## Ansprechpartner

Akram Askar (CTO). Bei Fragen jederzeit melden.
