# Demo-Ablauf: Laborbefund-Pipeline in 15 Minuten

Ziel der Demo ist, die gesamte Kette einmal sichtbar zu machen: LDT/PDF-Upload,
Extraktion, Normalisierung, Plausibilitätsprüfung, Review-Oberfläche,
Evaluation, Fehleranalyse und nächste Schritte.

## Vorbereitung

Backend starten:

```sh
cd loesung
python3 -m uv run uvicorn src.api:app --reload
```

Frontend starten:

```sh
cd loesung/frontend
npm install
npm run dev
```

Browser öffnen:

```text
http://localhost:5173
```

Eval-Zahlen bei Bedarf neu erzeugen:

```sh
cd loesung
python3 -m uv run python -m src.eval_pdf --output reports/tag2-evaluation.json
```

Mit neuen LLM-Aufrufen, falls gespeicherte Ausgaben fehlen:

```sh
python3 -m uv run python -m src.eval_pdf --run-llm --output reports/tag2-evaluation.json
```

## Demo-Dateien

| Zweck | Datei | Warum diese Datei |
| --- | --- | --- |
| KBV-LDT mit eingebettetem PDF | `../data/ldt/kbv-testdaten/Z01_UseCase15_Befund_mit_PDF.ldt` | enthält klinisch-chemische `Obj_0060`-Werte mit Einheiten |
| Eigenes Text-PDF | `data/synthetisch/SYN-001/befund.pdf` | sauberes synthetisches PDF mit Gold-Daten |
| Eigenes Scan-PDF | `data/synthetisch/SYN-004/befund_scan.pdf` | verschmutzter Scan für Claude Vision / direkten Bild-Input |
| Fehlerhafter LDT-Fall | `data/synthetisch/SYN-016/befund.ldt` | absichtlich fehlende Einheit, demonstriert Regelverletzung |
| Eval-Report | `reports/tag2-evaluation.json` | enthält die dokumentierten Precision-/Recall-Zahlen |

## 15-Minuten-Ablauf

### 0:00–2:00 — Scope und Architektur

Kurz erklären:

- Wir verarbeiten Laborbefunde aus LDT und PDF.
- Der erste fachliche Scope sind numerische klinisch-chemische Laborwerte.
- LDT wird deterministisch geparst und validiert.
- PDFs laufen entweder regelbasiert, per Text-LLM oder bei Scan-PDFs per Vision-LLM.
- Alle Ergebnisse nutzen dasselbe Pydantic-Schema und laufen danach durch Normalisierung und Plausibilität.

Wichtiger Satz:

> Wir zwingen nicht jeden LDT-Ergebnistyp in ein Laborwertschema. Humangenetik/`Obj_0073` wird erkannt und gemeldet, aber nicht als numerischer Laborwert erfunden.

### 2:00–5:00 — KBV-Testbefund live hochladen

In der Review-UI hochladen:

```text
../data/ldt/kbv-testdaten/Z01_UseCase15_Befund_mit_PDF.ldt
```

Zeigen:

- Die API erkennt LDT.
- Eingebettetes PDF erscheint links.
- Extrahierte Werte erscheinen rechts.
- Quelle ist `LDT`.
- Normalisierung und Plausibilität laufen im Backend mit.

Falls nach Einheiten gefragt wird:

- Einheiten werden nur direkt aus dem Ergebnisobjekt übernommen.
- Wir reparieren fehlende Einheiten nicht aus PDF oder Referenzbereich, damit die Extraktion auditierbar bleibt.

### 5:00–8:00 — Eigenes Text-PDF hochladen

Hochladen:

```text
data/synthetisch/SYN-001/befund.pdf
```

Zuerst ohne LLM-Haken zeigen:

- Regelbasierte Baseline ist schnell und lokal.
- Sie funktioniert gut bei einfachen Layouts.

Dann mit LLM-Haken erklären:

- Text-PDF läuft durch `pdf_llm_extractor.py`.
- Structured Output nutzt dasselbe Schema wie LDT.
- Das Modell darf nichts normalisieren und nichts erfinden.

### 8:00–10:00 — Scan-PDF / Vision-LLM zeigen

Hochladen:

```text
data/synthetisch/SYN-004/befund_scan.pdf
```

LLM-Haken aktivieren.

Erklären:

- Das PDF hat keinen zuverlässig extrahierbaren Text.
- Die API rendert die Seiten als Bilder.
- `pdf_vision_extractor.py` schickt diese Bilder an Claude Vision über OpenRouter.
- Das ersetzt in dieser Variante klassische OCR.

Wichtiger Unterschied:

- `scan_ocr_llm` ist die OCR-Pipeline mit Tesseract oder Simulation.
- `scan_vision_llm` ist direkter Bild-Input an ein multimodales Modell.

### 10:00–12:00 — Eval-Zahlen zeigen

Öffnen:

```text
reports/tag2-evaluation.json
```

Oder README-Tabelle zeigen.

Kernaussagen:

| Variante | Aussage |
| --- | --- |
| Regex-Baseline | Fehler bei komplexen Layouts, besonders zweispaltig |
| Claude Text | stabilste Text-PDF-Variante, 1.0 bei allen Kernmetriken |
| Gemini Text | Werte/Einheiten/LOINC gut, aber 18 Flag-Abweichungen |
| OCR-Pipeline | downstream stabil, aber OCR selbst ist hier simuliert |
| Claude Vision | echte Bildinterpretation, messbar, aber schwächer als Text-PDF |
| Gemini Vision | getestet, aber Ausgabeform nicht stabil schema-konform |

Guter Erklärungssatz:

> Gemini Vision hat die Werte im Bild grundsätzlich erkannt, lieferte aber über OpenRouter mehrfach Stringlisten statt Laborwert-Objekten. Deshalb haben wir es dokumentiert, aber nicht als faire Precision-/Recall-Zahlenreihe gezählt.

### 12:00–13:30 — Interessanteste Fehler

Zeigen oder erklären:

- Regex-Baseline: Spalten und Zeilenumbrüche erzeugen fehlende/zusätzliche Werte.
- Gemini Text: Flag wird teils in ein anderes Feld gesetzt oder weggelassen, während Wert und Einheit stimmen.
- Claude Vision: Bildinterpretationsfehler, z. B. ein Wert fehlt, ein Wert wird zusätzlich gelesen, Einheiten/Referenzbereich können abweichen.
- LDT: absichtliche Regelverletzungen wie fehlende Einheit werden gemeldet und nicht still repariert.

Optionaler LDT-Fehlerfall:

```text
data/synthetisch/SYN-016/befund.ldt
```

### 13:30–15:00 — Nächste Schritte

Kurz und konkret:

1. Weitere LDT-Ergebnistypen modellieren, besonders `Obj_0073`/Humangenetik als eigenes Schema statt als Laborwert.
2. Review-Korrekturen automatisch in Gold-/Eval-Fälle überführen.
3. Mehr echte Scan-PDFs testen und Vision/OCR-Varianten vergleichen.
4. UI erweitern: Normalisierte Werte, LOINC-Codes und Plausibilitätsdetails direkt in der Tabelle anzeigen.
5. Persistenz und Deployment ergänzen: Datenbank, Authentifizierung, Versionierung der Reviews.

Abschlusssatz:

> Der aktuelle Stand zeigt eine nachvollziehbare Ende-zu-Ende-Pipeline: synthetische Gold-Daten, LDT/PDF-Extraktion, LLM-Vergleich, Normalisierung, Plausibilität, Evaluation und eine Review-Oberfläche für Korrekturen.
