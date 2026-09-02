---
title: Verlorene Git Änderungen rekonstruieren, wenn Reflog und Working Tree nicht mehr helfen
date: 2026-08-15 09:00:00
updated: 2026-08-15 09:00:00
tags: [Git, recovery, debugging, version-control]
categories: [Engineering, Debugging]
excerpt: Git kann verlorene Änderungen nur dann zuverlässig retten, wenn sie irgendwann als Objekt persistiert wurden.
---

## Deutsch

Git wird häufig als Sicherheitsnetz für Quellcode verstanden. Diese Annahme ist nur teilweise korrekt.

Git kann Commits sehr zuverlässig rekonstruieren. Änderungen, die niemals Bestandteil eines Git Objekts wurden, besitzen dagegen wesentlich schwächere Wiederherstellungsgarantien. Besonders problematisch wird die Situation nach einer Kombination aus Branchwechseln, Resets, Stashes, Rebases oder anderen Operationen, wenn anschließend festgestellt wird, dass lokale Änderungen fehlen.

Die zentrale Frage bei einer Wiederherstellung lautet deshalb nicht:

```text
Wie mache ich den letzten Git Befehl rückgängig?
```

sondern:

```text
Hat Git den verlorenen Inhalt irgendwann als Objekt gespeichert?
```

Davon hängt ab, ob `reflog`, Objektanalyse oder nur noch externe Backups helfen können.

### 1. Git verwaltet mehrere unterschiedliche Zustände

Für die Diagnose müssen mindestens drei Zustände getrennt werden:

```text
HEAD
Index
Working Tree
```

`HEAD` bezeichnet normalerweise den aktuell ausgecheckten Commit. Der Index enthält den Zustand, der für den nächsten Commit vorbereitet wurde. Der Working Tree enthält die Dateien, mit denen tatsächlich gearbeitet wird.

Damit können für dieselbe Datei gleichzeitig drei verschiedene Versionen existieren:

```text
HEAD version
staged version
working tree version
```

Viele Recovery Fehler entstehen, weil diese Ebenen gedanklich zusammengefasst werden.

### 2. Git speichert Inhalte als Objekte

Das Git Objektmodell enthält hauptsächlich:

```text
blob
tree
commit
tag
```

Ein Blob enthält Dateiinhalt. Ein Tree beschreibt Verzeichnisstrukturen und verweist auf Blobs oder weitere Trees. Ein Commit verweist unter anderem auf einen Tree und auf vorherige Commits.

```text
commit
↓
tree
↓
blob
```

Für Recovery ist insbesondere der Blob relevant. Wenn eine verlorene Dateiversion irgendwann als Blob in die Objektdatenbank geschrieben wurde, kann sie unter Umständen wiedergefunden werden, auch wenn kein normaler Branch mehr auf sie verweist.

### 3. Uncommitted bedeutet nicht immer ungespeichert

Eine Änderung kann nicht committed sein und trotzdem bereits als Git Objekt existieren.

Das kann beispielsweise passieren, wenn sie gestaged wurde, Teil eines Stash war, Teil eines später verworfenen Commits war oder durch bestimmte interne Git Operationen als Objekt geschrieben wurde.

Umgekehrt gilt: Nur im Working Tree vorhanden bedeutet nicht automatisch in Git gespeichert. Diese Unterscheidung bestimmt die Recovery Möglichkeiten.

### 4. Reflog verfolgt Referenzbewegungen

`git reflog` ist eines der wichtigsten Recovery Werkzeuge. Es wird jedoch häufig falsch verstanden.

Reflog zeichnet Bewegungen bestimmter Referenzen auf, beispielsweise `checkout`, `commit`, `reset`, `rebase` und `merge`. Es ist keine kontinuierliche Historie des Working Trees.

Wenn eine Datei drei Stunden bearbeitet, niemals gestaged oder anderweitig in Git gespeichert und anschließend überschrieben wurde, kann Reflog diese Zwischenversion nicht rekonstruieren.

\[
ReflogHistory 
eq WorkingTreeHistory
\]

### 5. Reflog ist besonders nützlich bei verlorenen Commits

Angenommen, eine Branchreferenz zeigte zunächst auf:

```text
A → B → C
```

und wird anschließend auf `A` zurückgesetzt.

Der sichtbare Branch enthält nun:

```text
A
```

Commit `C` kann jedoch weiterhin existieren. Der Reflog kann die frühere Position der Referenz enthalten.

Dann kann zunächst geprüft werden, ob der verlorene Commit tatsächlich der gesuchte Zustand ist. Wichtig ist, nicht sofort weitere destruktive Operationen auszuführen.

### 6. Recovery sollte mit einem Freeze beginnen

Nach dem Verlust von Änderungen sollte die Anzahl weiterer Git Operationen minimiert werden.

Besonders ungünstig sind zunächst:

```text
git gc
git prune
weitere aggressive resets
Repository Cleanup
automatische Wartungsoperationen
```

Der Grund liegt in der Lebensdauer unerreichbarer Objekte. Ein Objekt ohne erreichbare Referenz kann noch vorhanden sein, aber später durch Garbage Collection entfernt werden.

Die erste Phase sollte deshalb ausschließlich beobachtend sein:

```text
stop mutation
↓
inspect state
↓
preserve repository
↓
attempt recovery
```

### 7. Eine Repository Kopie ist eine einfache Sicherheitsmaßnahme

Bevor tiefergehende Recovery Versuche beginnen, ist eine Kopie des Repositoryzustands sinnvoll.

Das gilt insbesondere für `.git`, denn dort befindet sich die Git Objektdatenbank. Recovery Experimente sollten möglichst nicht die einzige noch vorhandene Kopie verändern.

### 8. Zuerst sollte der aktuelle Zustand dokumentiert werden

Bevor Änderungen vorgenommen werden, sollte festgehalten werden:

```text
aktueller Branch
HEAD Commit
Working Tree Status
Index Status
vorhandene Stashes
Reflog
ungewöhnliche Referenzen
```

Die Zielsetzung ist eine Timeline:

```text
bekannter guter Zustand
↓
Operation 1
↓
Operation 2
↓
Verlust bemerkt
↓
aktueller Zustand
```

Eine solche Rekonstruktion ist wesentlich zuverlässiger als zufälliges Ausprobieren von Reset Befehlen.

### 9. `git fsck` untersucht die Objektstruktur

Wenn Reflog keinen direkten Weg liefert, wird die Objektdatenbank interessant.

`git fsck` kann unter anderem Objekte identifizieren, die nicht mehr über normale Referenzen erreichbar sind.

Konzeptionell existiert eine Menge aller vorhandenen Objekte:

\[
O
\]

und eine Menge der von bekannten Referenzen erreichbaren Objekte:

\[
R \subseteq O
\]

Dann sind:

\[
U = O \setminus R
\]

die unerreichbaren Objekte. Ein Teil von `U` kann für die Recovery interessant sein.

### 10. Blobs besitzen keine Dateinamen

Ein Blob speichert primär Inhalt. Er weiß nicht, dass diese Daten beispielsweise zu `src/example.ts` gehörten.

Die Zuordnung von Dateinamen entsteht über Trees. Ein isolierter unerreichbarer Blob besitzt deshalb möglicherweise keinen direkt sichtbaren ursprünglichen Pfad.

Recovery wird dann zu Content Discovery:

```text
enumerate candidate blobs
↓
extract contents
↓
search known fragments
↓
inspect matches
```

### 11. Verlorene Commits sind leichter als isolierte Blobs

Wenn ein verlorener Commit gefunden wird, enthält dessen Tree weiterhin die Dateistruktur.

Die Recovery Priorität sollte deshalb typischerweise lauten:

```text
reflog
↓
lost commits
↓
trees
↓
individual blobs
```

und nicht sofort `scan every blob`.

### 12. Stash besitzt ebenfalls eine Git Objektstruktur

Ein Stash sollte nicht als separates Backupformat betrachtet werden. Intern basiert er auf Git Objekten und Commits.

Gestashte Änderungen können mit normalen Git Mechanismen untersucht werden, und ein verlorener Stash kann unter bestimmten Bedingungen weiterhin als Objekt existieren, obwohl seine normale Referenz nicht mehr sichtbar ist.

Auch hier gilt:

```text
reference removal
≠
object deletion
```

### 13. Recovery Befehle sollten zunächst nur lesen

Ein schlechter Recovery Workflow besteht aus einer Reihe von Befehlen wie `reset`, `checkout`, `restore` und `reset again`. Jeder Schritt verändert zusätzlichen Zustand.

Ein besseres Verfahren trennt Diagnose und Reparatur:

```text
Phase 1: read only inspection
Phase 2: identify recovery object
Phase 3: create protective reference
Phase 4: restore selected content
```

Sobald ein relevanter Commit gefunden wurde, kann beispielsweise zunächst eine neue Referenz darauf erzeugt werden, bevor weitere Experimente stattfinden.

### 14. Externe Historien können wichtiger sein als Git

Wenn eine Änderung niemals in die Git Objektdatenbank gelangte, endet die Git spezifische Recovery möglicherweise.

Dann werden externe Zustandsquellen relevant:

```text
IDE local history
editor swap files
filesystem snapshots
backup system
cloud file history
temporary files
```

Diese Systeme besitzen andere Persistenzgrenzen als Git.

Deshalb sollte die Recovery Frage breiter formuliert werden:

```text
Welche Systeme haben diese Daten irgendwann beobachtet und persistiert?
```

### 15. Recovery Fähigkeit sollte vor dem Fehler entworfen werden

Die beste Datenrettung beginnt vor dem Datenverlust.

Für Entwicklungsworkflows sind mehrere Maßnahmen hilfreich:

```text
kleine häufige Commits
Work in Progress Commits
regelmäßige Pushes auf private Branches
IDE Local History
Dateisystem Backups
vorsichtiger Umgang mit destruktiven Git Befehlen
```

Ein Commit muss nicht releasefähig sein, um als Recovery Punkt nützlich zu sein. Lokale Commits sind billig und können später reorganisiert werden.

### Fazit

Git bietet sehr gute Wiederherstellungsmöglichkeiten, solange verlorene Daten irgendwann Bestandteil seiner Objektdatenbank waren.

\[
Git\ can\ recover\ stored\ objects
\]

aber nicht:

\[
Git\ can\ recover\ every\ text\ ever\ typed
\]

Ein belastbarer Recovery Workflow beginnt deshalb mit einem Freeze, dokumentiert den aktuellen Zustand und untersucht anschließend Reflog, verlorene Commits und erst danach einzelne unerreichbare Objekte. Gefundene Kandidaten werden gesichert, bevor der aktive Working Tree verändert wird.

Versionierung, Backups und lokale Historie lösen unterschiedliche Probleme. Ein robuster Entwicklungsworkflow verwendet deshalb mehrere voneinander unabhängige Recovery Ebenen.

## English

Git is often treated as a safety net for source code. That assumption is only partially correct.

Git can recover committed state remarkably well. Changes that never became Git objects have much weaker recovery guarantees. The distinction becomes important after branch switches, resets, stashes, rebases, or other operations when local changes unexpectedly disappear.

The central recovery question is therefore not:

```text
How do I undo the last Git command?
```

It is:

```text
Did Git ever store the lost content as an object?
```

The answer determines whether reflog, object database analysis, or only external backups can help.

Git recovery requires separating `HEAD`, the index, and the working tree. A file can have a committed version, a staged version, and a working-tree version at the same time. Many mistakes come from treating these as one state.

Git stores content as objects: blobs, trees, commits, and tags. A blob stores file content, a tree stores paths and references to blobs, and a commit points to a tree and parent commits. If a lost version was ever written as a blob, it may still be recoverable even if no branch points to it.

Uncommitted does not always mean unstored. A change may have been staged, included in a stash, part of a discarded commit, or written by another Git operation. Conversely, content that existed only in the working tree may never have entered Git's object database.

`git reflog` tracks reference movement, not every edit. It is excellent for recovering lost commits after reset, rebase, checkout, or merge operations, but it cannot reconstruct text that was typed, never staged, and then overwritten.

After discovering loss, begin with a freeze:

```text
stop mutation
↓
inspect state
↓
preserve repository
↓
attempt recovery
```

Avoid `git gc`, `git prune`, additional aggressive resets, and cleanup operations until the repository has been preserved. Unreachable objects can exist physically for some time, but garbage collection may remove them later.

Before recovery, document the current branch, `HEAD`, working tree status, index status, stashes, reflog, and unusual references. The goal is to reconstruct a timeline rather than randomly trying reset commands.

If reflog does not provide a direct answer, inspect the object database. `git fsck` can reveal dangling or unreachable commits and blobs. These objects are not automatically errors; they are candidates.

Blobs do not store file names. An isolated blob knows only its content. That makes known text fragments especially valuable:

```text
enumerate candidate blobs
↓
extract contents
↓
search known fragments
↓
inspect matches
```

Lost commits are easier than isolated blobs because commits still include trees and paths. A sensible recovery order is:

```text
reflog
↓
lost commits
↓
trees
↓
individual blobs
```

Stashes are also Git object structures. A lost stash may still exist as an object even if the normal stash reference is gone.

Recovery commands should be read-only until a candidate is identified. Then create a protective reference or restore into a separate file or branch, compare, and only then move selected content into the active working tree.

If the content never entered Git's object database, Git-specific recovery ends. IDE local history, editor swap files, filesystem snapshots, cloud history, and backups may become more important than Git.

The broader lesson is that recovery guarantees depend on what was actually persisted, not what a user once saw on screen. Version control, backups, and local history solve different problems; robust workflows use several independent recovery layers.
