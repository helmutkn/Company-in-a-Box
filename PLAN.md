# nanobot – Entwicklungsplan

> Stand: 5. März 2026 · Branch: `cursor/neuer-plan-f015` · Version: 0.1.4.post3

---

## Zusammenfassung

nanobot ist ein ultra-leichtgewichtiger KI-Assistent (~4.000 Zeilen Kerncode) mit Multi-Channel-Support, LLM-Integration über LiteLLM und einem Skills-System. Dieses Dokument beschreibt den Entwicklungsplan für die nächsten Iterationen – aufgeteilt in vier Phasen, priorisiert nach Auswirkung und Aufwand.

---

## Phase 1 – Stabilisierung & Qualität (Kurzfristig)

Fokus: Bestehende Bugs beheben, Testabdeckung erhöhen, Code-Konsistenz verbessern.

### 1.1 Schema- und Konfigurationsfixes

| Aufgabe | Dateien | Beschreibung |
|---------|---------|--------------|
| Doppelte `MatrixConfig` entfernen | `nanobot/config/schema.py` | Zwei identische `MatrixConfig`-Klassen existieren (Zeilen 67–80 und 188–202). Eine davon entfernen. |
| QQ `allow_from`-Semantik klären | `nanobot/config/schema.py`, `nanobot/channels/qq.py` | Kommentar sagt "empty = public access", aber `BaseChannel.is_allowed()` behandelt leere Liste als Deny-All. Konsistenz herstellen. |
| `tools_used` in Memory-Konsolidierung befüllen | `nanobot/agent/loop.py`, `nanobot/agent/memory.py` | `tools_used` wird in `_save_turn()` nie gesetzt – die Konsolidierung bekommt immer leere Tool-Listen. Aus `assistant.tool_calls` extrahieren. |
| Ungenutzten `skill_names`-Parameter bereinigen | `nanobot/agent/context.py` | `skill_names` wird an `build_messages()` übergeben, dort aber nie verwendet. Entweder nutzen oder entfernen. |

### 1.2 Testabdeckung erweitern

| Bereich | Aktueller Stand | Ziel |
|---------|----------------|------|
| **Channels** | 2/10 getestet (Email, Matrix) | Mindestens Telegram, Discord, Slack, WhatsApp unit-testen |
| **Provider** | Kein direkter Test | `LiteLLMProvider` Model-Resolution, `CustomProvider` Basis-Flow |
| **Config** | Keine Tests | `load_config`, `save_config`, `_migrate_config`, Schema-Validierung |
| **ChannelManager** | Kein Test | `_init_channels`, `_dispatch_outbound`, `_validate_allow_from` |
| **BaseChannel** | Kein Test | `is_allowed()` für `"*"`, leere Liste, Pipe-getrennte IDs |

### 1.3 Code-Konsistenz

- [ ] Slack `_is_allowed()` an `BaseChannel.is_allowed()` angleichen oder dokumentierte Abweichung
- [ ] Tool-Schema-Definitionen vereinheitlichen (manche nutzen Class-Attribute, andere `@property`)
- [ ] Channel-Plugin-Registry statt hartcodierter Imports in `ChannelManager`

---

## Phase 2 – Architektur-Verbesserungen (Mittelfristig)

Fokus: Skalierbarkeit, Robustheit und Erweiterbarkeit verbessern.

### 2.1 Session-Parallelität

**Problem:** Ein einziger globaler `_processing_lock` verhindert parallele Nachrichtenverarbeitung über Sessions hinweg.

**Lösung:** Pro-Session-Locks einführen, damit verschiedene Chats gleichzeitig verarbeitet werden können.

```
Vorher:  _processing_lock (global) → alle Sessions blockiert
Nachher: _session_locks[session_id] → nur gleiche Session blockiert
```

### 2.2 Token-Budgetierung im Kontext

**Problem:** Kein Token-Budgeting oder Truncation für langen Verlauf. Bei langen Konversationen können Token-Limits überschritten werden.

**Lösung:**
- Token-Schätzung für History, System-Prompt und User-Nachricht
- Automatische Truncation älterer Nachrichten wenn Budget erschöpft
- Konfigurierbare Limits pro Provider/Modell

### 2.3 Shared Tool Factory

**Problem:** Tool-Registrierung wird in `loop.py` und `subagent.py` dupliziert.

**Lösung:** Gemeinsame Factory-Funktion die Tool-Sets basierend auf Kontext (Hauptschleife vs. Subagent) erstellt.

### 2.4 Konfigurationsverbesserungen

- [ ] Config-Versionsfeld und robustere Migrationsstrategie
- [ ] Optionale YAML/TOML-Unterstützung neben JSON
- [ ] Provider-spezifische Validierung im Schema
- [ ] Kanal-spezifische Prompt-Überschreibungen ermöglichen

### 2.5 Skill-System härten

- [ ] PyYAML statt manueller YAML-Frontmatter-Parsing (bricht bei verschachtelten Strukturen)
- [ ] Skill-Metadata cachen statt bei jedem Aufruf neu einlesen
- [ ] Skill-Abhängigkeiten zwischen Skills unterstützen
- [ ] Per-Channel Skill-Konfiguration (bestimmte Skills nur für bestimmte Kanäle)

---

## Phase 3 – Neue Features (Mittelfristig)

Fokus: Aus der bestehenden Roadmap im README die wichtigsten Features umsetzen.

### 3.1 Streaming-Support

**Beschreibung:** Aktuell werden Antworten erst nach vollständiger Generierung gesendet. Streaming würde die gefühlte Latenz deutlich reduzieren.

**Umfang:**
- LLM-Response-Streaming in `loop.py`
- Progressive Ausgabe über Channels die es unterstützen (Telegram Edit, Discord Edit, Slack Update)
- Fallback auf Batch-Modus für Channels ohne Streaming-Support

### 3.2 Erweiterte Suchwerkzeuge

**Beschreibung:** Kein `grep_file` oder `search_in_files`-Tool vorhanden. Bei großen Workspaces fehlt die Möglichkeit, effizient nach Inhalten zu suchen.

**Neue Tools:**
- `grep_file` – Regex-Suche in einzelnen Dateien
- `search_files` – Rekursive Suche über Verzeichnisse
- `find_files` – Dateinamen-basierte Suche mit Glob-Patterns

### 3.3 Multimodal-Erweiterung

**Beschreibung:** Bilder werden bereits im Kontext unterstützt (base64), aber die Verarbeitung ist rudimentär.

**Erweiterungen:**
- Bild-Analyse und -Beschreibung als eingebaute Fähigkeit
- Audio-Transkription über weitere Provider (nicht nur Groq)
- Video-Zusammenfassungen (Frame-Extraktion + Beschreibung)
- Datei-Uploads über alle Channels vereinheitlichen

### 3.4 Verbessertes Memory-System

**Beschreibung:** Das aktuelle zwei-Schichten-System (MEMORY.md + HISTORY.md) hat keine semantische Suche und keine Größenlimits.

**Verbesserungen:**
- Embedding-basierte Vektorsuche für relevante Erinnerungen
- Automatische Zusammenfassung/Verdichtung von MEMORY.md ab einer Größenschwelle
- Memory-Eviction-Policy für alte/irrelevante Einträge
- Retry-Logik für Konsolidierungsfehler

### 3.5 Subagent-Verbesserungen

- [ ] Progress-Updates während Subagent-Ausführung
- [ ] Konfigurierbare Iterations-Limits (aktuell fest auf 15)
- [ ] Optionaler MCP-Zugriff für Subagenten
- [ ] Subagent-Memory für langfristige Hintergrundaufgaben

---

## Phase 4 – Fortgeschrittene Features (Langfristig)

Fokus: Differenzierende Fähigkeiten für fortgeschrittene Nutzung.

### 4.1 Multi-Step Reasoning & Planung

**Beschreibung:** Der Agent handelt aktuell reaktiv (eine Nachricht → eine Verarbeitung). Für komplexe Aufgaben fehlt vorausschauende Planung.

**Ansätze:**
- Chain-of-Thought-Prompting mit Planungsschritt vor Ausführung
- Aufgabenzerlegung: Komplexe Anfragen in Teilschritte aufteilen
- Reflexionsschritt: Nach Ausführung prüfen ob das Ziel erreicht wurde
- Plan-Persistenz über Sitzungen hinweg

### 4.2 Selbst-Verbesserung durch Feedback

**Beschreibung:** Der Agent lernt nicht aus Fehlern oder Nutzerfeedback.

**Ansätze:**
- Feedback-Mechanismus: Nutzer kann Antworten bewerten
- Fehleranalyse: Gescheiterte Tool-Aufrufe analysieren und Muster erkennen
- Prompt-Anpassung: Basierend auf Feedback den System-Prompt personalisieren
- Skill-Vorschläge: Basierend auf Nutzungsmustern neue Skills empfehlen

### 4.3 Erweiterte Integrationen

| Integration | Beschreibung |
|-------------|--------------|
| **Kalender** | Google Calendar / CalDAV-Integration für Terminverwaltung |
| **Dateisystem-Watch** | Automatische Reaktion auf Dateiänderungen im Workspace |
| **Git-Integration** | Nativer Git-Workflow (Commits, PRs, Code-Review) |
| **Datenbank** | SQLite/PostgreSQL für strukturierte Datenspeicherung |
| **Notifications** | Push-Benachrichtigungen über verschiedene Kanäle |

### 4.4 Plugin-Architektur

**Beschreibung:** Channels, Provider und Tools sind aktuell hartcodiert. Eine Plugin-Architektur würde die Erweiterbarkeit für die Community massiv verbessern.

**Design:**
- Registry-Pattern für Channels (analog zu Provider-Registry)
- Plugin-Entdeckung über Entry-Points (`pyproject.toml`)
- Standardisierte Schnittstellen mit Validierung
- Plugin-Marketplace-Integration (ClawHub)

---

## Priorisierungsmatrix

| Aufgabe | Aufwand | Auswirkung | Priorität |
|---------|---------|------------|-----------|
| Schema-Fixes (1.1) | Niedrig | Mittel | **P0** |
| Testabdeckung (1.2) | Mittel | Hoch | **P0** |
| Code-Konsistenz (1.3) | Niedrig | Mittel | **P1** |
| Session-Parallelität (2.1) | Mittel | Hoch | **P1** |
| Token-Budgetierung (2.2) | Mittel | Hoch | **P1** |
| Shared Tool Factory (2.3) | Niedrig | Mittel | **P1** |
| Config-Verbesserungen (2.4) | Mittel | Mittel | **P2** |
| Skill-System härten (2.5) | Mittel | Mittel | **P2** |
| Streaming-Support (3.1) | Hoch | Hoch | **P1** |
| Suchwerkzeuge (3.2) | Niedrig | Hoch | **P1** |
| Multimodal (3.3) | Hoch | Hoch | **P2** |
| Memory-Verbesserungen (3.4) | Hoch | Hoch | **P2** |
| Subagent-Verbesserungen (3.5) | Mittel | Mittel | **P2** |
| Multi-Step Reasoning (4.1) | Hoch | Hoch | **P3** |
| Selbst-Verbesserung (4.2) | Hoch | Mittel | **P3** |
| Erweiterte Integrationen (4.3) | Hoch | Mittel | **P3** |
| Plugin-Architektur (4.4) | Hoch | Hoch | **P3** |

**Legende:** P0 = Sofort, P1 = Nächste Iteration, P2 = Mittelfristig, P3 = Langfristig

---

## Technische Schulden

Folgende technische Schulden sollten parallel adressiert werden:

1. **Doppelte `MatrixConfig`** in `config/schema.py` – Zeile 67–80 ist redundant
2. **`tools_used` nie befüllt** – `memory.py` bekommt immer leere Tool-Listen bei der Konsolidierung
3. **Skill-YAML-Parsing fragil** – Manuelles Parsing bricht bei verschachtelten Strukturen oder Doppelpunkten in Werten
4. **Tool-Ergebnis-Trunkierung fest auf 500 Zeichen** – Sollte konfigurierbar sein
5. **Subagent Tool-Registrierung dupliziert** – Gemeinsame Factory fehlt
6. **Kein `process_direct()` mit `channels_config`** – Direkte Verarbeitung ignoriert Kanal-Konfiguration
7. **Channel-Manager hartcodiert** – Kein Registry/Plugin-Pattern für Channels

---

## Metriken & Erfolgskriterien

| Metrik | Aktuell | Ziel (Phase 1) | Ziel (Phase 2) |
|--------|---------|-----------------|-----------------|
| Testabdeckung (Channels) | 2/10 | 6/10 | 10/10 |
| Testabdeckung (Provider) | 0 | 2 | Alle |
| Config-Tests | 0 | Basis-Suite | Vollständig |
| Kerncode-Zeilen | ~3.935 | <4.200 | <4.500 |
| Offene Bugs (bekannt) | ~7 | 0 | 0 |
| Antwort-Latenz (gefühlt) | Batch | Batch | Streaming |

---

## Nächste Schritte

1. **Sofort:** Schema-Fixes aus 1.1 umsetzen (niedrigster Aufwand, sofortiger Nutzen)
2. **Diese Woche:** Test-Infrastruktur für Channels und Config aufbauen
3. **Nächste Iteration:** Session-Parallelität und Token-Budgetierung angehen
4. **Fortlaufend:** Technische Schulden bei jeder Feature-Arbeit mit adressieren
