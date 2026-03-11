# Company in a Box – Entwicklungsplan

> Stand: 5. März 2026 · Branch: `cursor/neuer-plan-f015`

---

## Vision

**Eine komplette Firma, die auf KI-Agenten-Teams aufbaut und auf einem einzigen Rechner (Box) läuft.**

Jede Abteilung der Firma wird durch spezialisierte Agenten repräsentiert, die eigenständig arbeiten, miteinander kommunizieren und als Team Aufgaben erledigen. Die Firma operiert autonom, braucht aber einen menschlichen "CEO" der die strategische Richtung vorgibt und Ergebnisse abnimmt.

```
┌─────────────────────────────────────────────────────────┐
│                    COMPANY IN A BOX                      │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  │
│  │ Geschäfts-│  │Engineering│  │Marketing │  │Support │  │
│  │ führung   │  │          │  │ & Sales  │  │        │  │
│  │          │  │ ┌──────┐ │  │ ┌──────┐ │  │┌──────┐│  │
│  │  CEO-    │  │ │Back- │ │  │ │Content│ │  ││Kunden││  │
│  │  Agent   │  │ │end   │ │  │ │Writer │ │  ││betreu││  │
│  │          │  │ └──────┘ │  │ └──────┘ │  ││ung   ││  │
│  │  CFO-    │  │ ┌──────┐ │  │ ┌──────┐ │  │└──────┘│  │
│  │  Agent   │  │ │Front-│ │  │ │SEO    │ │  │┌──────┐│  │
│  │          │  │ │end   │ │  │ │Analyst│ │  ││FAQ   ││  │
│  └──────────┘  │ └──────┘ │  │ └──────┘ │  ││Bot   ││  │
│                │ ┌──────┐ │  └──────────┘  │└──────┘│  │
│                │ │DevOps│ │                 └────────┘  │
│                │ └──────┘ │                             │
│                └──────────┘                             │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │              Message Bus (Routing)               │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ Shared   │  │ Cron &   │  │ Externe  │              │
│  │ Memory   │  │ Heartbeat│  │ Channels │              │
│  └──────────┘  └──────────┘  └──────────┘              │
└─────────────────────────────────────────────────────────┘
```

---

## Fundament: Was nanobot heute kann

nanobot bringt bereits wichtige Bausteine mit:

| Baustein | Aktueller Stand | Rolle für Company-in-a-Box |
|----------|----------------|---------------------------|
| **AgentLoop** | Ein Agent, eine Schleife, Tool-Nutzung | Basis für jeden einzelnen Firmen-Agenten |
| **SubagentManager** | Hintergrund-Tasks, Ergebnis-Rückmeldung | Vorläufer der Team-Delegation |
| **MessageBus** | FIFO Inbound/Outbound-Queues | Erweiterbar zum internen Routing-Bus |
| **Sessions** | `channel:chat_id`-basiert | Erweiterbar um Agent- und Team-Kontext |
| **Skills** | Markdown-Dateien mit Fähigkeiten | Definieren Agenten-Rollen und Expertise |
| **Bootstrap-Dateien** | SOUL.md, IDENTITY.md, AGENTS.md | Grundlage für individuelle Agenten-Persönlichkeiten |
| **Cron/Heartbeat** | Zeitgesteuerte und proaktive Tasks | Autonomer Betrieb der Firma |
| **Provider-Registry** | Viele LLM-Provider, auto-detection | Verschiedene Modelle für verschiedene Rollen |
| **Channels** | Telegram, Discord, Slack, Email, etc. | Externe Schnittstelle der Firma |

**Was fehlt:**
- Kein Multi-Agenten-Routing (alles geht an einen Agent)
- Keine Agent-Identitäten oder Rollen
- Keine Agent-zu-Agent-Kommunikation
- Keine Team-Koordination (Delegation, Handoff, Eskalation)
- Keine per-Agent Konfiguration (Modell, Provider, Workspace)

---

## Phase 1 – Agenten-Identität & Multi-Agent-Grundlage

> Ziel: Mehrere benannte Agenten mit eigenen Rollen, Persönlichkeiten und Konfigurationen betreiben.

### 1.1 Config-Schema erweitern

**Aktuell:** Ein einziger `agents.defaults`-Block für alle.

**Neu:** Benannte Agenten mit eigener Konfiguration.

```json
{
  "company": {
    "name": "Meine Firma GmbH",
    "description": "KI-gestützte Softwareentwicklung"
  },
  "agents": {
    "defaults": {
      "model": "anthropic/claude-sonnet-4-20250514",
      "temperature": 0.1
    },
    "team": {
      "ceo": {
        "model": "anthropic/claude-opus-4-5",
        "role": "Geschäftsführer",
        "department": "management",
        "workspace": "~/.nanobot/company/management/ceo"
      },
      "backend-dev": {
        "model": "anthropic/claude-sonnet-4-20250514",
        "role": "Backend-Entwickler",
        "department": "engineering",
        "workspace": "~/.nanobot/company/engineering/backend"
      },
      "content-writer": {
        "model": "openrouter/google/gemini-2.5-pro",
        "role": "Content Writer",
        "department": "marketing",
        "workspace": "~/.nanobot/company/marketing/content"
      }
    }
  }
}
```

**Dateien:**
- `nanobot/config/schema.py` – `AgentConfig` mit `role`, `department`, `workspace` pro Agent
- `nanobot/config/loader.py` – Multi-Agent-Config laden

### 1.2 Workspace pro Agent

Jeder Agent bekommt eigene Bootstrap-Dateien:

```
~/.nanobot/company/
├── shared/                          # Geteiltes Firmenwissen
│   ├── COMPANY.md                   # Firmenbeschreibung, Werte, Ziele
│   ├── PROCESSES.md                 # Firmenweite Prozesse
│   ├── memory/
│   │   ├── MEMORY.md                # Geteiltes Langzeitgedächtnis
│   │   └── HISTORY.md               # Geteilte Historie
│   └── skills/                      # Geteilte Skills
│
├── management/
│   └── ceo/
│       ├── SOUL.md                  # "Ich bin der CEO. Ich delegiere..."
│       ├── IDENTITY.md              # Spezifische Identität
│       ├── AGENTS.md                # Anweisungen für diesen Agenten
│       ├── HEARTBEAT.md             # Proaktive CEO-Aufgaben
│       └── memory/                  # Eigenes Gedächtnis
│
├── engineering/
│   ├── backend/
│   │   ├── SOUL.md                  # "Ich bin Backend-Entwickler..."
│   │   ├── skills/                  # github, testing, deployment
│   │   └── memory/
│   └── frontend/
│       ├── SOUL.md
│       └── ...
│
├── marketing/
│   └── content/
│       ├── SOUL.md                  # "Ich schreibe überzeugende Texte..."
│       └── ...
│
└── support/
    └── customer/
        ├── SOUL.md
        └── ...
```

**Dateien:**
- `nanobot/agent/context.py` – `ContextBuilder` mit `agent_root` Parameter; lädt sowohl `shared/` als auch agent-spezifische Bootstrap-Dateien
- `nanobot/templates/` – Department-spezifische Templates

### 1.3 Mehrere AgentLoops

Statt eines einzigen `AgentLoop` laufen mehrere parallel:

```
CompanyRunner
  ├── AgentLoop("ceo", bus, provider_opus, workspace_ceo)
  ├── AgentLoop("backend-dev", bus, provider_sonnet, workspace_backend)
  ├── AgentLoop("content-writer", bus, provider_gemini, workspace_content)
  └── AgentLoop("support", bus, provider_sonnet, workspace_support)
```

**Dateien:**
- `nanobot/company/runner.py` – Neuer `CompanyRunner` der mehrere Agents startet
- `nanobot/agent/loop.py` – `AgentLoop` bekommt `agent_id` und `agent_role`
- `nanobot/cli/commands.py` – Neuer `nanobot company` CLI-Befehl

---

## Phase 2 – Agent-zu-Agent-Kommunikation

> Ziel: Agenten können sich gegenseitig Aufgaben delegieren, Ergebnisse austauschen und koordiniert arbeiten.

### 2.1 Message Bus erweitern

**Aktuell:** Zwei FIFO-Queues (Inbound/Outbound), ein Consumer.

**Neu:** Routing nach `target_agent_id` mit internem Agent-Channel.

```python
@dataclass
class InboundMessage:
    channel: str              # "telegram", "agent", "system"
    sender_id: str            # User-ID oder Agent-ID
    chat_id: str
    content: str
    target_agent_id: str = "" # Routing: welcher Agent soll antworten?
    message_type: str = ""    # "task", "result", "handoff", "broadcast"
    task_id: str = ""         # Für Tracking von delegierten Aufgaben
    priority: int = 0         # Höher = wichtiger
    media: list = field(default_factory=list)
    metadata: dict = field(default_factory=dict)
```

**Routing-Logik:**
1. Nachricht mit `target_agent_id` → direkt an diesen Agent
2. Nachricht ohne `target_agent_id` → an den Router-Agent (CEO) oder regelbasiert
3. `channel="agent"` → interne Agent-zu-Agent-Nachricht
4. `message_type="broadcast"` → an alle Agenten

**Dateien:**
- `nanobot/bus/events.py` – Erweiterte Message-Typen
- `nanobot/bus/router.py` – Neuer Message-Router mit Routing-Regeln
- `nanobot/bus/bus.py` – Multi-Consumer-Support

### 2.2 Delegate-Tool

Neues Tool mit dem Agenten Aufgaben an andere Agenten delegieren können:

```
delegate_to(
    agent_id: str,           # "backend-dev"
    task: str,               # "Implementiere die REST-API für User-Management"
    priority: str = "normal", # "low", "normal", "high", "urgent"
    wait_for_result: bool = false
)
```

**Workflow:**
```
CEO empfängt Anfrage: "Baue eine Landing Page"
  → CEO analysiert und delegiert:
    delegate_to("content-writer", "Schreibe Texte für Landing Page")
    delegate_to("backend-dev", "Erstelle HTML/CSS Template")
  → Content-Writer liefert Texte
  → Backend-Dev baut Template mit Texten
  → CEO prüft und liefert Ergebnis an den Menschen
```

**Dateien:**
- `nanobot/agent/tools/delegate.py` – Neues `DelegateTool`
- `nanobot/agent/loop.py` – Ergebnisse von delegierten Tasks verarbeiten

### 2.3 Koordinationsprotokolle

Strukturierte Kommunikation zwischen Agenten:

| Protokoll | Beschreibung | Beispiel |
|-----------|-------------|----------|
| **Delegation** | Agent A gibt Aufgabe an Agent B | CEO → Backend-Dev: "Implementiere Feature X" |
| **Ergebnis** | Agent B meldet Ergebnis an Agent A | Backend-Dev → CEO: "Feature X implementiert, PR #42" |
| **Handoff** | Agent A übergibt Kontext an Agent B | Support → Engineering: "Bug Report mit Details" |
| **Eskalation** | Agent B eskaliert an übergeordneten Agent | Backend-Dev → CEO: "Brauche Budget-Freigabe" |
| **Broadcast** | Agent A informiert alle | CEO → Alle: "Neue Firmenstrategie" |
| **Review** | Agent A bittet Agent B um Prüfung | Backend-Dev → QA: "Bitte Code-Review" |

---

## Phase 3 – Team-Koordination & Autonomer Betrieb

> Ziel: Die Firma operiert eigenständig mit Meetings, Workflows und proaktivem Handeln.

### 3.1 Team-Meetings (Cron-gesteuert)

Automatische Koordinations-Meetings über das Cron-System:

```json
{
  "company_cron": {
    "daily_standup": {
      "schedule": "0 9 * * *",
      "type": "team_meeting",
      "participants": ["ceo", "backend-dev", "content-writer"],
      "agenda": "Status-Update, Blocker, Tagesplanung"
    },
    "weekly_review": {
      "schedule": "0 17 * * 5",
      "type": "team_meeting",
      "participants": ["ceo", "cfo"],
      "agenda": "Wochenrückblick, KPIs, Budget"
    }
  }
}
```

**Ablauf eines Daily Standups:**
1. Cron triggert Meeting
2. Jeder Agent wird nach Status gefragt
3. CEO sammelt, priorisiert und verteilt Aufgaben
4. Ergebnis wird als Meeting-Protokoll gespeichert
5. Optional: Zusammenfassung an den Menschen via bevorzugtem Channel

**Dateien:**
- `nanobot/company/meetings.py` – Meeting-Koordination
- `nanobot/cron/service.py` – Erweitert um `team_meeting`-Typ

### 3.2 Workflow-Engine

Definierbare Geschäftsprozesse als Workflows:

```yaml
# workflows/feature-development.yml
name: Feature-Entwicklung
trigger: delegation_from_ceo
steps:
  - agent: backend-dev
    task: "Implementierung"
    output: code_branch
  - agent: qa-agent
    task: "Code Review & Tests"
    input: code_branch
    output: review_result
  - agent: backend-dev
    task: "Review-Feedback einarbeiten"
    condition: "review_result.approved == false"
  - agent: ceo
    task: "Finale Abnahme"
    output: approved
```

**Dateien:**
- `nanobot/company/workflows.py` – Workflow-Definition und -Ausführung
- `nanobot/company/templates/` – Vordefinierte Workflow-Templates

### 3.3 Per-Agent Heartbeat

Jeder Agent hat eigene proaktive Aufgaben:

| Agent | Heartbeat-Aufgaben |
|-------|-------------------|
| **CEO** | Offene Delegationen prüfen, Eskalationen bearbeiten, Tagesbericht |
| **Backend-Dev** | Build-Status prüfen, Dependencies updaten, Tech-Debt tracken |
| **Content-Writer** | SEO-Rankings prüfen, Content-Kalender aktualisieren |
| **Support** | Offene Tickets prüfen, FAQ aktualisieren |
| **CFO** | Kosten-Tracking (API-Kosten, Token-Verbrauch), Budget-Warnungen |

**Dateien:**
- `nanobot/heartbeat/service.py` – Per-Agent-Heartbeat statt globalem
- Jeder Agent-Workspace hat eigene `HEARTBEAT.md`

### 3.4 Geteiltes und privates Gedächtnis

```
Gedächtnis-Hierarchie:
├── Firmen-Gedächtnis (shared/)     → Alle Agenten lesen/schreiben
│   ├── COMPANY_MEMORY.md           → Firmenwissen, Entscheidungen
│   └── COMPANY_HISTORY.md          → Firmen-Historie
├── Team-Gedächtnis (department/)   → Nur Team-Mitglieder
│   └── TEAM_MEMORY.md              → Team-spezifisches Wissen
└── Agent-Gedächtnis (agent/)       → Nur dieser Agent
    ├── MEMORY.md                   → Persönliches Wissen
    └── HISTORY.md                  → Persönliche Historie
```

**Dateien:**
- `nanobot/agent/memory.py` – Drei-Ebenen-Gedächtnis
- `nanobot/agent/context.py` – Alle Ebenen in den Kontext laden

---

## Phase 4 – Externe Schnittstellen & Skalierung

> Ziel: Die Firma interagiert mit der Außenwelt und kann ihre Kapazität anpassen.

### 4.1 Intelligentes Channel-Routing

Externe Nachrichten werden zum richtigen Agenten geroutet:

| Quelle | Routing |
|--------|---------|
| Email an support@firma.de | → Support-Agent |
| Telegram-Nachricht vom CEO | → CEO-Agent |
| Discord #engineering Channel | → Engineering-Team |
| Slack DM | → CEO-Agent (Standardroute) |
| Email an bewerbung@firma.de | → HR-Agent |

**Regeln im Config:**
```json
{
  "routing": {
    "rules": [
      { "channel": "email", "match": "support@", "target": "support" },
      { "channel": "telegram", "from": "CEO_USER_ID", "target": "ceo" },
      { "channel": "discord", "room": "engineering", "target": "backend-dev" },
      { "default": "ceo" }
    ]
  }
}
```

**Dateien:**
- `nanobot/company/routing.py` – Regelbasiertes Routing
- `nanobot/bus/router.py` – Erweitert um externe Channel-Regeln

### 4.2 Firmen-Dashboard

Ein Web-Dashboard das den Firmenstatus zeigt:

- Agenten-Status (aktiv, idle, beschäftigt)
- Laufende Aufgaben und Delegationen
- Meeting-Protokolle
- Kosten-Übersicht (Token, API-Calls)
- Chat-Interface zum direkten Kontakt mit jedem Agenten
- Workflow-Fortschritt

**Dateien:**
- `nanobot/company/dashboard/` – Einfaches Web-UI (FastAPI + HTMX)

### 4.3 Kosten-Management

Der CFO-Agent trackt und optimiert Kosten:

```
Kosten-Tracking:
├── Token-Verbrauch pro Agent/Tag/Aufgabe
├── API-Kosten pro Provider
├── Kosten pro Workflow/Projekt
├── Budget-Limits und Warnungen
└── Modell-Empfehlungen (günstigeres Modell wenn möglich)
```

**Strategie:** Einfache Aufgaben → günstiges Modell (z.B. Haiku), komplexe Aufgaben → starkes Modell (z.B. Opus).

### 4.4 Dynamische Team-Skalierung

Agenten können bei Bedarf gespawnt oder pausiert werden:

- CEO erkennt hohe Last → spawnt temporären Agenten
- Kein Support-Bedarf nachts → Support-Agent pausiert
- Großes Projekt → temporäres Projekt-Team wird zusammengestellt
- Budget-Limit erreicht → nicht-kritische Agenten pausieren

---

## Phase 5 – Fortgeschrittene Fähigkeiten

> Ziel: Die Firma lernt, verbessert sich und handelt strategisch.

### 5.1 Firmenweites Lernen

- **Post-Mortems:** Nach gescheiterten Aufgaben analysiert das Team warum und aktualisiert Prozesse
- **Best Practices:** Erfolgreiche Muster werden in Skills extrahiert
- **Onboarding:** Neue Agenten lernen aus dem Firmen-Gedächtnis
- **Performance-Reviews:** CEO bewertet Agenten-Leistung und passt Prompts/Skills an

### 5.2 Strategische Planung

- CEO erstellt Quartals-/Monatspläne
- Ziele werden in Aufgaben zerlegt und auf Teams verteilt
- Fortschritt wird automatisch getrackt
- Abweichungen werden eskaliert

### 5.3 Externe Integrationen

| Integration | Agent | Zweck |
|-------------|-------|-------|
| GitHub/GitLab | Engineering | Code, PRs, Issues |
| Google Calendar | Alle | Termine, Deadlines |
| Notion/Wiki | Content | Dokumentation |
| Stripe/Billing | CFO | Finanzen |
| CRM (HubSpot etc.) | Sales/Support | Kundenverwaltung |
| Analytics | Marketing | Web-Analyse |

---

## Implementierungs-Roadmap

### Meilenstein 1: Multi-Agent-Grundlage (Phase 1)

```
Woche 1-2:
  ├── Config-Schema für benannte Agenten erweitern
  ├── Per-Agent Workspace-Struktur anlegen
  └── ContextBuilder für Agent-spezifische Bootstrap-Dateien

Woche 3-4:
  ├── CompanyRunner: Mehrere AgentLoops parallel starten
  ├── CLI: `nanobot company start`
  └── Erste zwei Agenten (CEO + ein Spezialist) laufen parallel
```

**Ergebnis:** Zwei unabhängige Agenten mit eigenen Persönlichkeiten laufen auf einer Maschine.

### Meilenstein 2: Kommunikation (Phase 2)

```
Woche 5-6:
  ├── Message Bus um Routing-Felder erweitern
  ├── DelegateTool implementieren
  └── Agent-zu-Agent Nachrichten funktionieren

Woche 7-8:
  ├── Koordinationsprotokolle (Delegation, Ergebnis, Handoff)
  ├── Task-Tracking (wer hat was an wen delegiert?)
  └── CEO kann Aufgaben verteilen und Ergebnisse sammeln
```

**Ergebnis:** CEO delegiert Aufgaben an Spezialisten, bekommt Ergebnisse zurück.

### Meilenstein 3: Autonomer Betrieb (Phase 3)

```
Woche 9-10:
  ├── Per-Agent Heartbeat
  ├── Team-Meetings via Cron
  └── Drei-Ebenen-Gedächtnis (Firma, Team, Agent)

Woche 11-12:
  ├── Workflow-Engine (einfache sequenzielle Workflows)
  ├── Eskalations-Mechanismus
  └── Die Firma läuft autonom mit regelmäßigen Check-ins
```

**Ergebnis:** Die Firma operiert eigenständig, hält Meetings und führt Workflows aus.

### Meilenstein 4: Externe Welt (Phase 4)

```
Woche 13-16:
  ├── Intelligentes Channel-Routing
  ├── Firmen-Dashboard (Web-UI)
  ├── Kosten-Tracking und -Management
  └── Dynamische Team-Skalierung
```

**Ergebnis:** Die Firma hat externe Schnittstellen, ein Dashboard und Kosten-Kontrolle.

### Meilenstein 5: Intelligenz (Phase 5)

```
Woche 17+:
  ├── Firmenweites Lernen
  ├── Strategische Planung
  └── Externe Integrationen
```

**Ergebnis:** Die Firma lernt, plant strategisch und ist in externe Systeme integriert.

---

## Technische Architektur

### Kern-Komponenten

```
nanobot/
├── company/                    # NEU: Company-in-a-Box Kern
│   ├── runner.py               # CompanyRunner: startet alle Agenten
│   ├── routing.py              # Nachrichtenrouting zwischen Agenten
│   ├── meetings.py             # Team-Meetings und Standups
│   ├── workflows.py            # Workflow-Engine
│   ├── dashboard/              # Web-Dashboard
│   └── templates/              # Firmen-Templates
│
├── agent/                      # Erweitert
│   ├── loop.py                 # + agent_id, agent_role
│   ├── context.py              # + agent_root, shared_root
│   ├── memory.py               # + Drei-Ebenen-Gedächtnis
│   └── tools/
│       ├── delegate.py         # NEU: DelegateTool
│       └── team.py             # NEU: TeamTool (Status, Mitglieder)
│
├── bus/                        # Erweitert
│   ├── events.py               # + target_agent_id, message_type, task_id
│   ├── bus.py                  # + Multi-Consumer-Support
│   └── router.py               # NEU: Message-Router
│
├── config/                     # Erweitert
│   └── schema.py               # + company, agents.team, routing
│
└── cli/                        # Erweitert
    └── commands.py             # + nanobot company [start|status|meeting]
```

### Datenfluss

```
Externe Nachricht (z.B. Telegram)
    │
    ▼
ChannelManager → MessageBus (Inbound)
    │
    ▼
Router (routing.py)
    ├── Regel: "Telegram CEO_ID" → CEO-Agent
    ├── Regel: "Email support@" → Support-Agent
    └── Default → CEO-Agent
    │
    ▼
AgentLoop (z.B. CEO)
    │
    ├── Direkt antworten → MessageBus (Outbound) → Channel
    │
    └── Delegieren → DelegateTool
         │
         ▼
    MessageBus (Inbound, channel="agent")
         │
         ▼
    Router → Ziel-Agent (z.B. Backend-Dev)
         │
         ▼
    AgentLoop (Backend-Dev)
         │
         └── Ergebnis → MessageBus → Router → CEO
```

---

## Beispiel-Szenario: "Baue mir eine Landing Page"

```
1. Mensch (via Telegram) → CEO-Agent:
   "Wir brauchen eine Landing Page für unser neues Produkt"

2. CEO-Agent analysiert:
   → delegate_to("content-writer", "Schreibe überzeugende Texte für
      eine Produktseite. Produkt: [Details]. Zielgruppe: [Details]")
   → delegate_to("backend-dev", "Erstelle HTML/CSS-Template für
      eine Landing Page. Warte auf Texte vom Content-Writer.")

3. Content-Writer liefert Ergebnis:
   → result_to("ceo", "Texte fertig: [Headline, Subtext, CTA, Features]")

4. CEO leitet Texte weiter:
   → delegate_to("backend-dev", "Hier sind die Texte: [...]
      Bitte ins Template einbauen.")

5. Backend-Dev liefert:
   → result_to("ceo", "Landing Page fertig. Dateien unter /output/landing/")

6. CEO prüft und antwortet dem Menschen:
   "Die Landing Page ist fertig! Dateien liegen unter /output/landing/.
    - Responsive Design ✓
    - SEO-optimierte Texte ✓
    - Call-to-Action ✓"
```

---

## Beispiel-Agenten-Profile

### CEO-Agent (SOUL.md)

```markdown
# Soul

Ich bin der CEO der Firma. Ich delegiere, koordiniere und stelle sicher,
dass Aufgaben erledigt werden.

## Verhalten
- Ich löse Aufgaben NICHT selbst, sondern delegiere an Spezialisten
- Ich zerlege komplexe Anfragen in klare Teilaufgaben
- Ich prüfe Ergebnisse bevor ich sie an den Menschen weiterleite
- Ich eskaliere wenn Agenten nicht liefern können

## Mein Team
- backend-dev: Programmierung, APIs, Infrastruktur
- content-writer: Texte, Marketing-Material, SEO
- support: Kundenfragen, FAQ, Problemlösung
```

### Backend-Entwickler-Agent (SOUL.md)

```markdown
# Soul

Ich bin ein erfahrener Backend-Entwickler. Ich schreibe sauberen,
getesteten Code und baue robuste Systeme.

## Verhalten
- Ich schreibe Code mit Tests
- Ich dokumentiere APIs und Architektur-Entscheidungen
- Ich frage nach wenn Anforderungen unklar sind (Eskalation an CEO)
- Ich bevorzuge einfache, wartbare Lösungen
```

---

## Kostenabschätzung (Token-Verbrauch)

| Szenario | Geschätzte Kosten/Tag |
|----------|----------------------|
| Idle (nur Heartbeats, 4 Agenten) | ~$0.50 |
| Leichter Betrieb (5-10 Aufgaben) | ~$2-5 |
| Normaler Betrieb (20-30 Aufgaben) | ~$10-20 |
| Intensiver Betrieb (50+ Aufgaben) | ~$30-50 |

**Optimierung:** CFO-Agent überwacht Kosten und empfiehlt günstigere Modelle für einfache Tasks.

---

## Erste Schritte (Sofort umsetzbar)

1. **Config-Schema erweitern** – `AgentConfig` mit `role`, `department`, `workspace`
2. **Zwei Agent-Workspaces anlegen** – CEO + ein Spezialist mit eigenen SOUL.md
3. **CompanyRunner schreiben** – Startet zwei `AgentLoop`-Instanzen parallel
4. **DelegateTool implementieren** – CEO kann an Spezialist delegieren
5. **CLI-Befehl** – `nanobot company start` startet die Firma

---

## Offene Fragen

- **Modell-Mix:** Soll jeder Agent sein eigenes Modell/Provider haben, oder alle das gleiche?
- **Kosten-Limit:** Gibt es ein tägliches Budget-Limit?
- **Channels:** Welche externen Channels soll die Firma nutzen?
- **Abteilungen:** Welche Abteilungen braucht die Firma initial?
- **Autonomie-Level:** Wie viel soll die Firma eigenständig entscheiden vs. nachfragen?
