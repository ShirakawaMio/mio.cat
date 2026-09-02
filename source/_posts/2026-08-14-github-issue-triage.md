---
title: GitHub Issues automatisch triagieren, ohne ein weiteres nutzloses KI Ranking zu bauen
date: 2026-08-14 09:00:00
updated: 2026-08-14 09:00:00
tags: [GitHub, automation, triage, machine-learning]
categories: [Engineering, Automation]
excerpt: Eine brauchbare automatische Issue-Triage braucht Zustandsrekonstruktion, Evidenz und Unsicherheitsangaben statt nur Labelfilter und Scores.
---

## Deutsch

Große Open Source Repositories können gleichzeitig hunderte oder tausende offene Issues enthalten. Für Maintainer und externe Contributor entsteht daraus ein Informationsproblem. Nicht jedes offene Issue ist tatsächlich bearbeitbar, nicht jedes als `good first issue` markierte Ticket ist einfach, und ein kleiner Code Fix kann mehr Aufwand für Reproduktion und Tests benötigen als eine größere, aber klar spezifizierte Änderung.

Eine automatische Suche nach geeigneten Contribution Aufgaben sollte deshalb nicht einfach Issues nach Labels filtern. Das eigentliche Problem ist eine Kombination aus Zustandsrekonstruktion, Aufwandsschätzung und Unsicherheitsbewertung.

Ein sinnvoller Ablauf sieht ungefähr so aus:

```text
Repository events
↓
candidate extraction
↓
hard filters
↓
evidence extraction
↓
feasibility scoring
↓
uncertainty estimation
↓
human review
```

Die entscheidende Eigenschaft ist, dass das System nicht nur einen Score liefert. Es muss erklären können, welche Evidenz zu diesem Score geführt hat.

## 1. Offen bedeutet nicht bearbeitbar

Ein offenes Issue kann verschiedene Zustände repräsentieren:

```text
reproduzierter Bug
unbestätigter Bug
Feature Request
Diskussion
bereits implementierte Änderung
wartet auf Maintainer Entscheidung
Duplikat
blockiert durch andere Änderung
Issue mit bereits offenem Pull Request
```

Eine Abfrage nach `state = open` liefert deshalb nur syntaktisch offene Tickets. Für Contribution Discovery interessiert dagegen ein semantischer Zustand:

\[
Actionable(issue)
\]

Dieser Zustand muss aus mehreren Signalen rekonstruiert werden.

## 2. Labels sind Hinweise und keine Wahrheit

Labels wie `good first issue`, `help wanted`, `bug` und `documentation` sind nützlich. Ihre Bedeutung ist aber nicht repositoryübergreifend standardisiert.

In einem Projekt kann `good first issue` eine kleine isolierte Änderung bedeuten. In einem anderen Projekt bedeutet es vielleicht nur, dass jemand das Issue irgendwann als möglicherweise geeignet betrachtet hat.

Ein Ranking System sollte Labels daher als Features behandeln:

\[
P(Easy \mid label)
\]

und nicht als deterministische Regeln:

\[
label = good\ first\ issue
\Rightarrow Easy
\]

Diese Unterscheidung reduziert viele falsche Treffer.

## 3. Aufwand hat mehrere Dimensionen

Issue Schwierigkeit lässt sich nicht gut durch eine einzige Größe wie erwartete Zeilenänderung beschreiben.

Nützlicher ist eine Zerlegung:

```text
Reproduktionsaufwand
Code Lokalisierung
Änderungsumfang
Testaufwand
Abstimmungsbedarf
Release Risiko
Domänenwissen
```

Ein Issue kann in einer Dimension klein und in einer anderen groß sein. Ein One-line Fix mit großem Reproduktionsaufwand ist kein einfaches Issue, nur weil der Patch klein sein kann.

## 4. Harte Filter sollten vor dem Ranking greifen

Vor jedem Score sollten Kandidaten entfernt werden, die offensichtlich ungeeignet sind:

```text
locked conversation
stale unanswered question
duplicate marker
needs maintainer decision
security-sensitive topic
active linked pull request
requires private infrastructure
```

Solche Fälle mit einem niedrigen Score zu versehen ist weniger hilfreich als sie aus dem normalen Vorschlagsfluss zu entfernen.

Die Pipeline wird dadurch interpretierbarer:

```text
not eligible
eligible but hard
eligible and feasible
```

## 5. Evidenz muss extrahiert und angezeigt werden

Ein System, das nur `score: 0.82` sagt, ist für Maintainer kaum überprüfbar.

Besser ist eine strukturierte Begründung:

```text
positive evidence:
- reproducible steps are present
- affected file path is mentioned
- maintainers accepted the bug

negative evidence:
- no failing test exists
- reproduction depends on external service
- last maintainer reply asks for clarification
```

Der Score ist dann nur eine kompakte Zusammenfassung der Evidenz, nicht das eigentliche Produkt.

## 6. Kommentare sind zeitliche Signale

Der letzte Kommentar kann wichtiger sein als der ursprüngliche Issue Text.

Ein Issue kann mit klarer Beschreibung beginnen und später blockiert werden. Umgekehrt kann eine unklare Meldung nach mehreren Kommentaren gut spezifiziert sein.

Deshalb sollte die Analyse eine Timeline bilden:

```text
issue opened
↓
maintainer asks question
↓
user provides reproduction
↓
maintainer confirms
↓
related PR opened
↓
PR merged or abandoned
```

Ohne diese Rekonstruktion werden veraltete Informationen überbewertet.

## 7. Unsicherheit ist ein Feature

Automatische Triage sollte nicht so tun, als sei jedes Ergebnis eindeutig.

Ein gutes System kann sagen:

```text
high confidence: documentation typo
medium confidence: likely reproducible bug
low confidence: architecture decision hidden in discussion
```

Unsicherheit ist besonders wichtig, wenn das System externen Contributorinnen und Contributoren Aufgaben vorschlägt. Ein unsicherer Vorschlag kostet reale Zeit.

## Fazit

GitHub Issue Triage ist kein reines Ranking Problem. Labels, Scores und Sprachmodelle sind hilfreich, aber nur als Bestandteile einer erklärbaren Pipeline.

Die robuste Lösung trennt harte Ausschlüsse, semantischen Zustand, Aufwandsschätzung, Evidenz und Unsicherheit. Dadurch entsteht kein magischer Priorisierer, sondern ein Werkzeug, das Maintainer und Contributor schneller zu guten Entscheidungen bringt.

## English

Large open source repositories can contain hundreds or thousands of open issues at the same time. For maintainers and external contributors, this creates an information problem. Not every open issue is actionable, not every ticket labeled `good first issue` is actually easy, and a tiny code change may require more work in reproduction and testing than a larger but precisely specified feature.

Automated contribution discovery should therefore do more than filter issues by labels. The actual problem combines state reconstruction, effort estimation, and uncertainty assessment.

A useful pipeline looks roughly like this:

```text
Repository events
↓
candidate extraction
↓
hard filters
↓
evidence extraction
↓
feasibility scoring
↓
uncertainty estimation
↓
human review
```

The important property is that the system should not merely produce a score. It should explain which evidence produced that score.

An open issue may represent a reproduced bug, an unconfirmed report, a feature request, a discussion, a duplicate, a blocked change, or a ticket that already has an active pull request. `state = open` is only a syntactic filter. Contribution discovery needs a semantic predicate:

\[
Actionable(issue)
\]

Labels are useful evidence, but they are not ground truth. `good first issue`, `help wanted`, `bug`, and `documentation` mean different things across repositories. A better system treats labels as probabilistic features rather than deterministic rules.

Difficulty also has several dimensions:

```text
reproduction effort
code localization
change size
testing effort
coordination cost
release risk
domain knowledge
```

A one-line fix with a difficult reproduction setup is not necessarily easy.

Hard filters should run before ranking: locked conversations, duplicates, security-sensitive reports, stale unanswered questions, issues waiting for maintainer decisions, or items with active linked pull requests should not be mixed into ordinary recommendations.

The output should show evidence:

```text
positive evidence:
- reproducible steps are present
- affected path is mentioned
- maintainer confirmed the behavior

negative evidence:
- no failing test exists
- reproduction depends on external service
- latest comment asks for clarification
```

Comments must be interpreted as a timeline. The latest maintainer comment can invalidate the original issue text, and later reproduction details can make an initially vague report actionable.

Uncertainty is not a weakness. A responsible system distinguishes high-confidence documentation fixes from low-confidence architecture discussions. The goal is to reduce review load while keeping humans in the decision loop.

Good issue triage is therefore not another opaque AI ranking. It is an evidence pipeline that reconstructs state, estimates effort, exposes uncertainty, and lets maintainers make faster decisions.
