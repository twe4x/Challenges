# Code-Review: Challenges-Plugin

**Stand:** Commit `6fccd14` („feat: v2.3.3“) · **Umfang:** ~470 Java-Dateien, ~41.500 Zeilen (`plugin/` + `mongo-connector/`), Build-/CI-Konfiguration, Sprachdateien
**Art:** Reine Analyse – es wurde kein Produktivcode geändert.

---

## Inhalt

1. [Kurzfassung](#1-kurzfassung)
2. [Vorgehen und Legende](#2-vorgehen-und-legende)
3. [Kritisch – sofort beheben](#3-kritisch--sofort-beheben)
4. [Sicherheit](#4-sicherheit)
5. [Datenverlust, Item-Verlust und Dupes](#5-datenverlust-item-verlust-und-dupes)
6. [Spiellogik-Bugs](#6-spiellogik-bugs)
7. [Performance](#7-performance)
8. [Threading (systemisches Problem)](#8-threading-systemisches-problem)
9. [Datenbank, Persistenz und Stats](#9-datenbank-persistenz-und-stats)
10. [Build, CI, Abhängigkeiten, Versionskompatibilität](#10-build-ci-abhängigkeiten-versionskompatibilität)
11. [Weitere Funde mit niedriger Priorität](#11-weitere-funde-mit-niedriger-priorität)
12. [Architektur und Code-Qualität](#12-architektur-und-code-qualität)
13. [Empfohlene Reihenfolge](#13-empfohlene-reihenfolge)

---

## 1. Kurzfassung

Das Plugin ist funktional sehr umfangreich und an vielen Stellen sorgfältig gebaut (Versions-Wrapper, Reflection-Fallbacks, Prepared Statements). Es gibt aber eine Reihe von Problemen, die **ohne Admin-Rechte ausnutzbar** sind oder **Spieler-Items bzw. Welten vernichten**. Die wichtigsten:

| # | Problem | Wirkung |
|---|---------|---------|
| 1 | **Der Build ist kaputt**: Das CloudNet-Maven-Repo ist umgezogen, `actions/upload-artifact@v3` ist von GitHub abgeschaltet | Weder lokal (frisch) noch in der CI baubar |
| 2 | **Custom-Challenge „Befehl ausführen“ läuft als Konsole**; die Whitelist prüft nur das erste Wort und nur bei der Eingabe, **nicht beim Laden aus Datei/DB** | Rechteausweitung bis zur Konsole |
| 3 | **Menüs blockieren weder Drag noch Doppelklick-Einsammeln** | Item-Dupes, z. B. aus dem MissingItems-Screen (ohne Permission) und aus `/invsee` |
| 4 | **Normale Spieler können den Server einfrieren**: HydraPlus spawnt 1024 Mobs pro Kill, Timber leakt pro einzelnem Log einen Endlos-Task, LoopChallenge-Loops wachsen unbegrenzt, AllBlocksDisappear/RepeatInChunk/Tsunami machen Zehntausende Blockänderungen pro Tick | DoS / Lag |
| 5 | **Item-Dupe in AllBlocksDisappear**: jeder Drop wird doppelt erzeugt | Beliebige Vervielfältigung |
| 6 | **Inventar-Vernichtung**: LoopChallenge löscht das komplette Inventar, CutClean löscht bei jedem Spielertod Verzauberungen und Shulker-Inhalte, Pausieren überschreibt Hotbar-Slots, Late-Joiner bei Extra-Welt-Challenges verlieren alles | Datenverlust |
| 7 | **World-Reset löscht jede fremde Welt namens `void`** (typischer Multiverse-Name); automatischer Reset nutzt immer Seed 0 | Datenverlust / immer gleiche Welt |
| 8 | **`@ScheduledTask`/`@TimerTask`/`TimedChallenge` laufen standardmäßig asynchron**; dadurch werden Inventare, Scoreboards, Blöcke und Entities off-thread verändert | Race Conditions, Dupes, Exceptions |
| 9 | **MySQL-Schema**: Die Tabelle überschreitet mit `utf8mb4` (MySQL-8-Standard) das 65.535-Byte-Zeilenlimit | Tabelle wird nicht angelegt |
| 10 | **Stats-DB-Zugriffe laufen synchron auf dem Main-Thread**, bei DB-Ausfall bei *jedem* Move-Event erneut | Massive Lags |
| 11 | Viele **Logikfehler in Challenges**: MaxBiome/MaxHeightTime töten nie, Force-Challenges starten nach Neustart im falschen Zustand, LastManStanding endet nie, JumpAndRun scheitert durch Spectators, Cheat-Erkennung greift nie … | Spiel funktioniert nicht wie beschrieben |

**Positiv:**
- SQL läuft über Prepared Statements. Eine SQL-Injection wurde nicht gefunden.
- `plugin.yml` und die Permission-Checks der Commands passen zusammen.
- Die Versionsabstraktion (`MinecraftNameWrapper`, `BukkitReflectionUtils`) ist größtenteils solide.
- Der Code kompiliert fehlerfrei gegen `--release 8`.

---

## 2. Vorgehen und Legende

- **Build:**
  - Der Build mit Maven 3.9 und JDK 21 schlägt mit der Original-Konfiguration fehl (siehe [B1](#b1-build-schlägt-fehl-cloudnet-repository-umgezogen)).
  - Mit temporär umgeleitetem Repository-Mirror baut das Projekt. Es gibt 96 Deprecation-Warnungen, aber keine Fehler.
  - Zusätzlich habe ich mit `javac --release 8` kompiliert. Dabei zeigte sich keine Nutzung von Java-9+-APIs.
- **Review:** Jede Datei wurde gelesen. Die Codebasis wurde dafür in sechs Bereiche aufgeteilt:
  - Kern-/Security-Pfade
  - Custom-Challenges und Menüs
  - Listener, Utils und Management
  - Goals, Settings und Typen
  - zwei Blöcke Challenge-Implementierungen

  Das Verhalten der Library `net.anweisen.utilities` 1.3.15 (Sources von JitPack) wurde mitgeprüft, wo das Plugin davon abhängt.
- **Automatisierte Checks:** Alle 549 statisch verwendeten `Message.forName(...)`-Keys wurden gegen `en.json` und `de.json` abgeglichen.

**Legende:**

| Symbol | Bedeutung |
|--------|-----------|
| ✅ | Selbst am Code nachvollzogen (Stelle gelesen, Logik durchgespielt) |
| 🔎 | Im Review gemeldet und begründet, Stelle plausibel, aber nicht einzeln von mir nachvollzogen |
| 🔴 / 🟠 / 🟡 / ⚪ | Kritisch / Hoch / Mittel / Niedrig |

**Pfad-Abkürzungen:** `P/` = `plugin/src/main/java/net/codingarea/challenges/plugin/`, `C/` = `P/challenges/implementation/challenge/`, `G/` = `P/challenges/implementation/goal/`, `T/` = `P/challenges/type/abstraction/`

---

## 3. Kritisch – sofort beheben

### K1 🔴 „Befehl ausführen“-Action läuft als Konsole, Whitelist ist umgehbar ✅
**Ort:** `P/challenges/custom/settings/action/impl/ExecuteCommandAction.java:26-58`, `P/management/challenges/CustomChallengesLoader.java:54-82`

- **Ausführung als Konsole:** `executeForPlayer` schickt den gespeicherten Befehl mit `Bukkit.getConsoleSender()` ab, also mit vollen Rechten. Mit Spieler-Ziel wird er als `execute as <name> at <name> run <cmd>` verpackt.
- **Whitelist nur bei der Eingabe:** `allowed-commands-to-execute` wird ausschließlich im Chat-Eingabe-Callback geprüft, und dort nur das **erste Wort**.
- **Keine Prüfung beim Ausführen oder Laden:**
  - Befehle, die später aus der Whitelist entfernt werden, laufen weiter.
  - Befehle aus `internal/custom_challenges.json` bzw. aus der DB-Spalte `custom_challenges` (`/db load customs`) werden **ungeprüft** ausgeführt, und geladene Challenges sind sofort aktiv.
  - Wer diese Daten verändern kann (gemeinsame MySQL, manipulierte Datei), führt damit beliebige Konsolenbefehle aus.
- **Gefährliche Argumente:** Auch die erlaubten Befehle sind mit Selektoren und NBT hochgefährlich:
  - `summon`/`setblock` können Command-Blöcke mit Auto-Ausführung erzeugen, sofern `enable-command-block` aktiv ist.
  - Außerdem möglich: `worldborder`, `kill @e`, `tp @a …`, großflächiges `fill`.
- **Rechte-Modell:** Es gibt keine eigene Permission (siehe [S1](#s1--custom-challenges-hängen-an-derselben-permission-wie-die-gui-)).

**Empfehlung:**
1. Befehle als der Spieler ausführen, damit dessen Permissions gelten. Alternativ den **kompletten** Befehl (inkl. Namespace, `execute`, NBT) beim **Ausführen und beim Laden** validieren.
2. Eine eigene, standardmäßig verweigerte Permission für Command-Actions einführen (z. B. `challenges.custom.command`).
3. Beim Laden `max-challenges`, `max-name-length` und `max-command-length` erzwingen.
4. Jede Ausführung loggen.

### K2 🔴 Menüs lassen Items herausziehen (Drag / Doppelklick) – Dupes ✅
**Ort:** Library `net.anweisen.utilities.bukkit.utils.menu.MenuPositionListener` (einziger Schutz aller Menüs)

- **Was abgefangen wird:** Der Listener cancelt nur Klicks im oberen Inventar und Shift-Klicks im unteren.
- **Was nicht abgefangen wird:**
  - `InventoryDragEvent`
  - `COLLECT_TO_CURSOR` (Doppelklick im eigenen Inventar sammelt gleiche Items **auch aus dem Menü** ein)
  - Klicks, während keine `MenuPosition` gesetzt ist
- **Konkrete Dupes:**
  - **MissingItems** (`C/inventory/MissingItemsChallenge.java:184-210`, `/openmissingitems` ohne Permission): Die Köder-Items sind echte, stapelbare `ItemStack`s mit bis zu 64 Stück, per Doppelklick einsammelbar. Auch das „echte“ fehlende Item lässt sich so doppeln. 🔎
  - **/invsee** (`P/spigot/command/InvseeCommand.java`): Die Ansicht ist eine Kopie des Zielinventars. Wer ein gleiches Item auf dem Cursor hat, sammelt die Kopien ein. ✅
- **Weitere Folgen:** Drag in Menüs vernichtet die gezogenen Items. `MenuPosition.Holder.positions` (`Map<Player, …>`) wird nie aufgeräumt, das ist ein Memory-Leak von Player-Objekten.

**Empfehlung:** Einen eigenen Listener mit hoher Priorität bauen, der für jede Inventaransicht mit `MenuPosition.HOLDER` oben **alle** Klicks (inkl. `COLLECT_TO_CURSOR`, `MOVE_TO_OTHER_INVENTORY`, Hotbar-Swap) und **alle** `InventoryDragEvent`s cancelt. `MenuPosition.remove(player)` beim Quit aufrufen. Köder-Items markieren (PDC/Lore), damit sie nie mit echten Items stapeln.

### K3 🔴 Normale Spieler können den Server einfrieren
Alle folgenden Punkte sind ohne Rechte auslösbar, sobald die jeweilige Challenge/Einstellung aktiv ist:

| Wo | Problem | Status |
|----|---------|--------|
| `C/entities/HydraPlusChallenge.java:35-44` | Speichert `min(count, 512)`, gibt aber das **unbegrenzte** `count` zurück. Ab dem Limit spawnt jeder Kill **1024 Mobs synchron** an einem Punkt; Sweep-Attacken multiplizieren das. | ✅ |
| `P/challenges/implementation/setting/TimberSetting.java:63-81` | Bei einem einzelnen Log (oder Stamm/Pilzstiel, `isLog` matcht „STEM“) ist `treeBlocks` leer, die Schleife läuft nie, `timer.cancel()` wird nie erreicht: **ein 1-Tick-Task pro Block läuft für immer**. Normales Farmen reicht. | ✅ |
| `C/world/LoopChallenge.java:67-186` | Jede `ProjectileLaunchEvent` (auch Skelette, Dispenser) und jeder Entity-Schaden erzeugt einen Loop, der sekündlich für immer wiederholt; `EntityDamageLoop` erzeugt sich selbst neu. Eine Dispenser-Clock ergibt quadratisches Wachstum. | 🔎 |
| `C/world/AllBlocksDisappearChallenge.java:81-147` | Pro Blockabbau wird der ganze Chunk (~98k Blöcke) gelesen; alle Blöcke gleichen Typs werden mit Physik auf AIR gesetzt. Bei Stein/Deepslate sind das Zehntausende Updates in einem Tick. | 🔎 |
| `C/world/RepeatInChunkChallenge.java:137-292` | Jede Blockänderung wird für jeden Spieler auf 289 Chunks repliziert, **mit Physik** (TNT ×289), und lädt/generiert Chunks synchron. `Chunk ==`-Vergleiche sind ab 1.19.4 immer false, deshalb läuft die Arbeit bei jedem Move. | 🔎 |
| `C/world/TsunamiChallenge.java:113-237` | Pro Move 121 `getChunkAt` + O(n²)-Listen; Fluten eines Chunks = 10⁴–10⁵ `setType` in einem Tick; Race mit asynchronem Timer. | 🔎 |
| `C/randomizer/RandomEventChallenge.java:151-172` | „ReplaceOres“ plant pro Spieler 40.401 Sync-Tasks im selben Tick (~5,8 Mio. Block-Reads). | 🔎 |
| `C/world/ChunkDeletionChallenge.java:126-142` | Pro gelöschtem Chunk ~98k `setType(AIR)` mit Physik in einem Tick, auch auf schon leeren Chunks. | ✅ |

**Empfehlung:**
- HydraPlus: das Limit auch im Rückgabewert anwenden und Spawns auf mehrere Ticks verteilen, mit globalem Entity-Cap.
- Timber: `if (treeBlocks.isEmpty()) return;` vor dem Scheduling.
- Große Blockoperationen generell mit `ChunkSnapshot` lesen, `setType(..., false)` verwenden und die Arbeit über mehrere Ticks verteilen.
- Loops und Spawns hart begrenzen.

---

## 4. Sicherheit

### S1 🟠 Custom-Challenges hängen an derselben Permission wie die GUI ✅
**Ort:** `P/management/menu/generator/ChallengeMenuGenerator.java:53-63`, `config.yml: manage-settings-permission: false`

- Mit dem Standardwert kehrt `playNoPermissionsEffect` sofort mit `false` zurück.
- Jeder mit `challenges.gui` darf deshalb alles: Challenges umschalten, Custom-Challenges bauen, Command-Actions anlegen (→ K1).
- Wer Freunden die GUI gibt, gibt ihnen damit faktisch Konsolen-Befehle.

**Empfehlung:** Getrennte Rechte (`challenges.custom.create`, `challenges.custom.command`), standardmäßig verweigert.

### S2 🟠 Pausen- und Vor-Start-Schutz hat Lücken ✅/🔎
**Ort:** `P/spigot/listener/RestrictionListener.java`

- **Nicht abgefangen:** `InventoryDragEvent` ✅, `PlayerPortalEvent`/`EntityPortalEvent`, `HangingBreakByEntityEvent` (Gemälde/Rahmen), `VehicleEnterEvent`. Außerdem ist Bewegung inkl. Elytra erlaubt 🔎. Spieler können also bei gestoppter Uhr erkunden und die Dimension wechseln.
- **`/top` ohne Permission** ignoriert Pause und Extra-Welt: Damit lässt sich JumpAndRun überspringen (`P/challenges/implementation/setting/TopCommandSetting.java:29-71`) 🔎.
- **`/bp` funktioniert während Extra-Welt-Challenges:** Items der Challenge (z. B. der WaterMLG-Eimer) lassen sich wiederholt einlagern, das ist ein Dupe (`P/challenges/implementation/setting/BackpackSetting.java:78-100`) 🔎.

### S3 🟠 World-Reset löscht Ordner anhand von Namen ohne Absicherung ✅/🔎
**Ort:** `P/management/server/GameWorldStorage.java:111-113`, `P/management/server/WorldManager.java:197-226`

- **Fremde `void`-Welt:** `getCustomGeneratedGameWorlds()` liefert fest `"void"`. **Jeder Reset löscht `<server>/void`**, auch wenn Challenges diesen Ordner nie angelegt hat, etwa bei einer Multiverse-Welt namens `void` ✅.
- **`level-name` ungeprüft:** Der Wert kommt aus `internal/session.json` und wird nicht validiert. `""` bzw. `..` würden auf den Server-Root bzw. aus ihm heraus zeigen 🔎.
- **Symlinks:** `FileUtils.deleteWorldFolder` (Library) folgt laut Bytecode-Analyse Symlinks 🔎.

**Empfehlung:**
- Einen Namespace-Namen verwenden (`challenges_void`) und nur löschen, wenn ein Gamestate-Flag sagt, dass das Plugin den Ordner angelegt hat.
- Den Pfad kanonisieren und prüfen, dass er ein direktes Kind von `getWorldContainer()` ist.
- `Files.walkFileTree` ohne `FOLLOW_LINKS` verwenden.

### S4 🟡 Weitere Sicherheitsaspekte

| Wo | Problem | Status |
|----|---------|--------|
| `P/content/loader/LanguageLoader.java:83-96` | Sprachnamen aus der heruntergeladenen `languages.json` werden unbereinigt zum Dateinamen (`getMessageFile(name)`). Ein kompromittiertes Repo könnte `.json`-Dateien außerhalb des Ordners schreiben. | ✅ |
| `P/content/loader/ContentLoader.java:25` | **Der Fork lädt Sprachdateien von `anweisen/Challenges`**, nicht aus `twe4x/Challenges`. Eigene Übersetzungen in diesem Repo kommen also nie beim Server an. | ✅ |
| `P/spigot/listener/HelpListener.java` | Überschreibt `/help`, `/h`, `/info`, `/hilfe`, reagiert auch auf `/minecraft:help` und auf von anderen Plugins gecancelte Befehle (MONITOR ohne `ignoreCancelled`) und gibt jedem Spieler die Plugin-Version aus. | ✅ |
| `P/challenges/implementation/setting/PositionSetting.java:90-172` | `/pos` (ohne Permission): Namen mit `.` werden vom `GsonDocument` als Pfad interpretiert. Damit lassen sich die gespeicherten Positionen korrumpieren (NPE für alle) oder fremde Einträge überschreiben. Es gibt kein Limit, und jede Position wird gebroadcastet und bekommt einen Partikel-Task (Spam). | 🔎 |
| `P/utils/misc/Utils.java:39-44` | `/stats <name>` (ohne Permission): Der Name geht unkodiert in die Mojang-API-URL, ohne Timeout. | ✅ |
| `C/entities/NewEntityOnJumpChallenge.java:46-50` | Der Spawn-Pool enthält Boote, Minecarts (mit Truhe/Hopper), TNT, End-Kristalle, Wither und Enderdrache. Durch Springen lassen sich Ressourcen farmen und Grief anrichten. | 🔎 |
| `P/utils/misc/InventoryUtils.java:17-20,230-234` | `getRandomItem` zieht aus einer ungefilterten Liste: Command-Blocks, Barrier, Bedrock, Spawner, Drachen-Spawn-Ei. Betrifft HotBarRandomizer, RandomItem und FiveHundredBlocks. | 🔎 |
| `C/entities/StoneSightChallenge.java:68` | Ersetzt den Block unter dem Mob durch Stein, auch Truhen und **Endportalrahmen**. Silverfish im Stronghold können das Spiel dadurch unschaffbar machen. | 🔎 |

---

## 5. Datenverlust, Item-Verlust und Dupes

| ID | Wo | Problem | Status |
|----|----|---------|--------|
| D1 🔴 | `C/world/AllBlocksDisappearChallenge.java:116-120` | `ChallengeHelper.dropItem(...)` **und** direkt danach `dropItemNaturally(...)`: **jeder Drop kommt doppelt**. Die Merge-Schleife (`:94-99`) hat außerdem kein `break`. Mit Diamantblöcken oder Shulkern ist das eine exponentielle Vervielfältigung. | ✅ |
| D2 🔴 | `C/world/LoopChallenge.java:300-331` | `isTool()` gibt `true` zurück, wenn `getAttributeModifiers(...)` `null` ist, also für fast alle Items. Danach wird jedes Item mit `damage >= maxDurability` gelöscht; für nicht beschädigbare Items gilt `0 >= 0`. **Das ganze Inventar wird gelöscht.** | ✅ |
| D3 🟠 | `P/challenges/implementation/setting/CutCleanSetting.java:223-234` | „Essen braten“ (standardmäßig an) ersetzt bei **jedem** `EntityDeathEvent` – auch beim Spielertod – alle Drops durch neue `ItemStack`s. Verzauberungen, Namen, Buchinhalte und Shulker-Inhalte gehen verloren, beschädigte Werkzeuge kommen repariert zurück. | ✅ |
| D4 🟠 | `P/management/inventory/PlayerInventoryManager.java:176-240` | Beim Pausieren akzeptiert `canGiveItems` Slots mit **gleichem Material**, und `giveItems` überschreibt sie. Wer z. B. 64 Bücher in Slot 4, Uhren in Slot 3 oder Limettenfarbstoff in Slot 5 hat, verliert den Stapel. Die Methode läuft zudem **absichtlich asynchron** (`:151`). | ✅ |
| D5 🟠 | `P/challenges/type/abstraction/WorldDependentChallenge.java:134-143`, `P/management/server/WorldManager.java:297-323` | Wer während einer Extra-Welt-Challenge (JumpAndRun, WaterMLG) joint, wird ohne `cachePlayerData` teleportiert und geleert. **Das Inventar ist endgültig weg**, der Spieler bleibt in der Extra-Welt. Die Backups liegen nur im RAM; WaterMLG stellt bei Disable/Stop nichts wieder her. | 🔎 |
| D6 🟠 | `P/spigot/listener/RestrictionListener.java:48-54` | Stirbt ein Spieler bei pausiertem Timer (Void wird bewusst nicht gecancelt, oder `/kill`), werden die Drops gelöscht, ohne das Inventar zu behalten. | 🔎 |
| D7 🟠 | `C/randomizer/MobRandomizerChallenge.java:87-102` | Bei **jeder Pause**, bei Disable und beim Shutdown wird jedes lebende Entity entfernt und neu gespawnt. Zahme Tiere, Namen, Villager-Trades und Truhen von Esel/Lama gehen verloren; natürliche Mobs werden mit durchgemischt. | ✅ |
| D8 🟠 | `C/world/LoopChallenge.java:211-370` | Loops halten das `Player`-Objekt und prüfen `isOnline()` nicht. Nach dem Logout nehmen sie weiter Items aus dem Offline-Objekt und droppen sie. Beim Rejoin lädt das volle Inventar erneut: ein Dupe. | 🔎 |
| D9 🟠 | `P/challenges/implementation/setting/SoupSetting.java:34-50` | Suppe in der Offhand: Der Spieler wird geheilt, das **Item in der Main-Hand durch eine Schüssel ersetzt**, und die Suppe bleibt. Das ergibt unendliche Heilung und Item-Verlust. | ✅ |
| D10 🟠 | `C/inventory/UncraftItemsChallenge.java:28-72,136-142` | Läuft asynchron (Standard von `TimedChallenge`). Ganzzahl-Division verliert Reste (7 Bretter → 1 Log), fast kaputte Diamantwerkzeuge werden zu vollen Diamanten, und auf 1.14/1.15 gibt es `NoClassDefFoundError` durch `SmithingRecipe` (`:110`). | ✅ (async) / 🔎 |
| D11 🟡 | `C/entities/BlockMobsChallenge.java:54-106` | Drops gehen verloren, wenn der Mob nicht durch einen Spieler stirbt oder despawnt. Explosionen löschen Container-Inhalte. | 🔎 |
| D12 🟡 | `C/inventory/MissingItemsChallenge.java:102-168` | Das Item liegt nur im RAM und am gespeicherten `Player`-Objekt. Nach Relog/Stop ist es verloren; der Timeout tötet auch nach Disable/Pause. | 🔎 |
| D13 🟡 | Gamestate allgemein | Timer und Gamestate (inkl. **Rucksäcke**, Goal-Fortschritt, `fresh`/`cheated`) werden **nur beim sauberen Shutdown** gespeichert. Nach einem Crash stammen Spielerinventare aus dem Vanilla-Autosave, der Rucksack aber vom letzten Stopp: Das dupliziert oder verliert Items. | ✅ |
| D14 🟡 | `G/forcebattle/ExtremeForceBattleGoal.java:84-107` | POSITION-Ziele werden als JSON-Objekt gespeichert, beim Laden gibt es eine `ClassCastException`. Der Teilzustand wird beim nächsten Shutdown zurückgeschrieben, **fast jede Extreme-Battle verliert ihren Fortschritt**. | 🔎 |
| D15 🟡 | `G/forcebattle/*`, `P/challenges/type/abstraction/ForceBattleGoal.java` | Umbenannte Enums (z. B. `GRASS→SHORT_GRASS`) führen zu `null`-Zielen, und **jeder weitere Save schlägt fehl**. Spieler ohne aktuelles Ziel verlieren ihren Fortschritt. | 🔎 |

---

## 6. Spiellogik-Bugs

### Challenges

| ID | Wo | Problem | Status |
|----|----|---------|--------|
| G1 🟠 | `C/time/MaxBiomeTimeChallenge.java:93-99`, `MaxHeightTimeChallenge.java:92-98` | Bei `time == limit` greift weder `>` noch `<`. Der Zähler bleibt bei „1 s übrig“, **niemand stirbt je**. Wird das Limit gesenkt, entsteht eine Todesschleife, weil der Zähler nie zurückgesetzt wird. | ✅ |
| G2 🟠 | `C/force/Force{Height,Block,Mob,Biome,Item}Challenge.java` | `writeGameState` schreibt immer `target`, `loadGameState` setzt dann `COUNTDOWN`. **Nach jedem Neustart** startet der Countdown mit altem Ziel; das tötet alle bzw. beendet die Challenge. | ✅ |
| G3 🟠 | `C/extraworld/JumpAndRunChallenge.java:157-168,215-234` | Die Fallback-Liste enthält ignorierte Spieler, und `onMove` prüft die Welt nicht. Ein Spectator in der Oberwelt lässt die Challenge **sofort scheitern**. | ✅ |
| G4 🟠 | `C/damage/WaterAllergyChallenge.java:33` | `return` statt `continue`: Ist der erste Spieler ignoriert (typisch: Spectator), bekommt **niemand** Schaden. | ✅ |
| G5 🟠 | `C/inventory/NoDupedItemsChallenge.java:77` | Die Inventare von Creative- und Spectator-Spielern werden mitgezählt, der Survival-Spieler stirbt. | ✅ |
| G6 🟠 | `C/miscellaneous/NoSharedAdvancementsChallenge.java:29-49` | `List<Advancement>.contains` greift auf 1.20.2+ nie, weil jedes Mal ein neues `CraftAdvancement` ohne `equals` entsteht. Die Challenge ist wirkungslos. | 🔎 |
| G7 🟠 | `C/effect/ChunkRandomEffectChallenge.java:104-145` | Der Effekt wird jede Sekunde entfernt und neu gesetzt: Absorption füllt sich ständig auf, Sofortschaden trifft jede Sekunde. `Chunk ==`-Vergleich pro Move. | 🔎 |
| G8 🟠 | `C/world/LevelBorderChallenge.java:127-158,316-320` | Die Border-Prüfung nutzt `abs(x)-abs(cx)` statt `abs(x-cx)` und vergleicht mit `size` statt `size/2`. Im Nether/End wird man ins Bedrock-Dach oder an das ×8-Zentrum teleportiert und erstickt in einer Schleife. | 🔎 |
| G9 🟡 | `C/world/ChunkDeletionChallenge.java:3,90` | Der Move-Handler ist mit **CloudNets** `@EventListener` annotiert statt mit `@EventHandler` und wird deshalb nie aufgerufen. Die Löschung läuft nur über den asynchronen Bossbar-Callback, mit HashMap-Race und `setProgress`-Exceptions. `getMinHeight()` direkt aufgerufen gibt auf 1.14–1.16 `NoSuchMethodError`. | ✅ |
| G10 🟡 | `C/damage/DeathOnFallChallenge.java:36-45` | `setDamage(health)` wird durch Rüstung und Absorption reduziert, der Spieler überlebt. `setCancelled(false)` ohne `ignoreCancelled` hebt Cancels anderer Plugins auf. | 🔎 |
| G11 🟡 | `C/world/LevelBorderChallenge.java:254-263` | `setCancelled(false)` auf Schaden hebelt Schutz-Plugins aus. | 🔎 |
| G12 🟡 | `C/world/SnakeChallenge.java`, `IceFloorChallenge.java`, `BedrockPath`, `FloorIsLava`, `BlockFlyInAir`, `LoopChallenge`, `SurfaceHole`, `AnvilRain`, `RandomEvent` | Portale, Endportalrahmen und Bedrock werden nicht geschützt. **Runs können unschaffbar werden.** | 🔎 |
| G13 🟡 | `P/spigot/listener/CustomEventListener.java:73` | Der „von Spieler getötet“-Event nutzt `getDamage()` statt `getFinalDamage()`. Hydra klont deshalb noch lebende Mobs, und MobsRespawnInEnd zählt Phantom-Kills. | ✅ |
| G14 🟡 | `P/challenges/type/helper/ChallengeHelper.java:218-220` | Absorption wird doppelt abgezogen (Freeze, DamageInventoryClear). | 🔎 |
| G15 🟡 | `C/entities/AllMobsToDeathPoint.java:42` | `getEntitiesByClasses` erfasst Unterklassen: Zombie → Husk/Drowned, **Guardian → Elder Guardian**. | 🔎 |
| G16 🟡 | `C/entities/MobSight…`, `StoneSight…` | `rayTraceEntities` ignoriert Blöcke, man „sieht“ durch Wände. | 🔎 |
| G17 🟡 | `C/inventory/PermanentItemChallenge.java:39-56`, `P/challenges/implementation/setting/SlotLimitSetting.java`, `…/setting/NoOffhandSetting.java`, `C/randomizer/HotBarRandomizerChallenge.java` | Zahlentasten-/F-Swap und Drags umgehen die Sperren. `NoOffhand` hat `return` statt `continue`. | 🔎 |
| G18 🟡 | `C/movement/*`, `C/damage/*` | Fahrzeuge umgehen alle Move-basierten Challenges, weil für Passagiere kein `PlayerMoveEvent` feuert. | 🔎 |
| G19 🟡 | `C/miscellaneous/OneDurabilityChallenge.java:62-70` | `setDamage(-1)` bei Items ohne Haltbarkeit wirft auf Paper bei jedem Klick eine Exception. | 🔎 |
| G20 🟡 | `C/miscellaneous/EnderGamesChallenge.java:73` | `nextInt(0)` ohne Entity in der Nähe; die übrigen Spieler werden nicht teleportiert. | 🔎 |
| G21 🟡 | `C/randomizer/RandomItemRemovingChallenge.java:36-49` | Anzeige `value*10` s, tatsächlich `value` s. | 🔎 |
| G22 🟡 | `C/randomizer/MobRandomizerChallenge.java:74-196` | Spawns werden *vor* dem Cap-Check gecancelt; der Welt-Zähler wird mit dem Pro-Spieler-Limit verglichen. Tier-, Wasser- und Ambient-Spawns verschwinden fast immer. | 🔎 |
| G23 🟡 | `T/RandomizerSetting.java` + Randomizer | Nur der Seed wird gespeichert, gemischt wird mit zustandsbehaftetem Random. Nach Toggle ergibt sich ein anderes Mapping als nach einem Neustart. | 🔎 |

### Goals

| ID | Wo | Problem | Status |
|----|----|---------|--------|
| G24 🟠 | `G/LastManStandingGoal.java:43-65` | Der Sterbende bzw. Quittende wird noch mitgezählt, weil RespawnSetting ihn erst bei MONITOR auf Spectator setzt. **Das Goal endet nie von selbst.** | ✅ |
| G25 🟠 | `G/forcebattle/ForceAdvancementBattleGoal.java:95` | Advancement-Vergleich per `==` (1.20.2+ neue Instanzen); NPEs ohne Ziel. | 🔎 |
| G26 🟡 | `G/forcebattle/ForceItemBattleGoal.java:72-94` | Ein Klick auf das Ziel-Item in **irgendeinem** Inventar (Truhe, Team-Rucksack) zählt. NPE ohne Ziel. | 🔎 |
| G27 🟡 | `G/forcebattle/ForceBlockBattleGoal.java:62-68` | „Block beim Skip geben“ gibt den **neuen** Block, der Joker zählt doppelt. | 🔎 |
| G28 🟡 | `G/forcebattle/targets/ForceTarget.java` | Kein `equals`/`hashCode`: „Keine doppelten Ziele“ wirkt nach jedem Neustart nicht mehr. | 🔎 |
| G29 🟡 | `P/challenges/type/abstraction/ForceBattleGoal.java:260-288` | `getWinnersOnEnd` fügt nie Gewinner hinzu: immer „kein Gewinner“. | 🔎 |
| G30 🟡 | `G/KillAllMobsGoal.java:28-36` | `ZOMBIE_HORSE` spawnt nie natürlich, das Goal ist unerfüllbar. | 🔎 |
| G31 🟡 | `P/utils/item/ItemUtils.java:28-86` | `"PATH_BLOCK"` existiert nicht, außerdem fehlen `SUSPICIOUS_*`, `TRIAL_SPAWNER` und `TEST_*`. Force-/Collect-Goals können unerreichbare Items wählen. | 🔎 |
| G32 🟡 | `G/EatCakeGoal.java:44-55` | Linksklick auf Kuchen gewinnt. | 🔎 |
| G33 🟡 | `P/challenges/type/abstraction/NetherPortalSpawnSetting.java` | `index` wird nie erhöht (nur eine Portal-Zuordnung gespeichert), `null` wird gespeichert, `locateNearestStructure(…, 10000)` läuft synchron im Teleport-Event. | 🔎 |

### Server-, Timer- und Kernlogik

| ID | Wo | Problem | Status |
|----|----|---------|--------|
| G34 🟠 | `P/management/server/WorldManager.java:74-76` | `prepareWorldReset(sender)` übergibt das primitive `customSeed`, das zu einem `Long ≠ null` wird. **Jeder automatische Reset (`reset-on-last-leave`) nutzt Seed 0** und generiert 3 Welten synchron im Quit-Event. | ✅ |
| G35 🟠 | `P/management/blocks/BlockDropManager.java:113` | `remove.forEach(drops::remove)` statt `chance::remove`: Das **Deaktivieren von LowDropRate löscht alle Randomizer-/CutClean-Drops**. | ✅ |
| G36 🟠 | `P/management/scheduler/timer/ChallengeTimer.java:119-122` | `resume()` setzt **alle Nicht-Creative-Spieler, auch Spectator**, auf Survival. Ausgeschiedene Spieler und zuschauende Admins werden „wiederbelebt“. | ✅ |
| G37 🟠 | `P/spigot/listener/CheatListener.java:76` | Vergleich `"give"` mit `"/give"`: **Die Cheat-Erkennung greift nie**, `no-stats-after-cheating` ist wirkungslos. | ✅ |
| G38 🟡 | `P/management/server/ServerManager.java:90` | `drop-items-on-end` bewirkt das Gegenteil der Config-Doku: Ohne Gewinner droppt niemand. | ✅ |
| G39 🟡 | `P/spigot/listener/BlockDropListener.java:34-67` | Ein Block mit Custom-Drop setzt `setYield(0)` für die **ganze** Explosion. | 🔎 |
| G40 🟡 | `P/management/scheduler/timer/ChallengeTimer.java:75-91` | Der Timer zählt Ticks statt Echtzeit: Bei 15 TPS läuft er 25 % zu langsam, und jede Pause/Fortsetzung verschiebt ihn um bis zu 1 s. | 🔎 |
| G41 🟡 | `P/spigot/listener/RestrictionListener.java:139-148` | Wetter-Events werden bei Pause gecancelt. `/weather rain` bzw. `/weather thunder` tun deshalb **bei pausiertem Timer nichts**, melden aber Erfolg, weil `setStorm` selbst ein `WeatherChangeEvent` auslöst. Ausnahme für `WeatherChangeEvent.Cause.PLUGIN/COMMAND` einbauen. | ✅ |
| G42 🟡 | `P/challenges/custom/settings/trigger/impl/IntervallTrigger.java:74,77` | `% 180` wird doppelt geprüft, `% 240` nie: **Das 4-Minuten-Intervall feuert nie.** | ✅ |
| G43 🟡 | `P/challenges/custom/settings/CustomSettingsLoader.java:39-151` | Alle Trigger und Actions werden erzeugt, *bevor* `@RequireVersion` geprüft wird, ohne try/catch. Ein fehlschlagender Konstruktor **deaktiviert das ganze Custom-Challenge-System**. | ✅ |
| G44 🟡 | `P/challenges/custom/settings/action/impl/CancelEventAction.java` | Ein statisches Boolean-Flag. Bei asynchronen Triggern läuft die Action erst im nächsten Tick, der Cancel geht verloren oder trifft das falsche Event. | 🔎 |
| G45 🟡 | `P/challenges/custom/settings/action/impl/PlaceStructureAction.java:59-62` | Konsolen-Befehl ohne `execute in <dimension>`: **Strukturen landen immer in der Oberwelt.** Der `(int)`-Cast rundet negative Koordinaten falsch (`getBlockX()` verwenden). | ✅ |
| G46 🟡 | `P/management/menu/generator/implementation/custom/InfoMenuGenerator.java:50` | Config-Key `save-player_challenges` statt `save-player-custom_challenges`. | ✅ |
| G47 ⚪ | `P/management/menu/generator/implementation/custom/MainCustomMenuGenerator.java:81` | Limit mit `>` statt `>=`: eine Challenge mehr als erlaubt. | ✅ |

---

## 7. Performance

### Hotspots

| ID | Wo | Problem | Status |
|----|----|---------|--------|
| P1 🟠 | `P/challenges/custom/settings/trigger/impl/StandsNotOnSpecificBlockTrigger.java:33-54` | Bei **jedem Blockwechsel jedes Spielers** entsteht eine Liste aller ~1.000 Materialien plus eine Namensliste; danach folgt `LinkedList.contains`. Das passiert auch, wenn keine Challenge den Trigger nutzt. | ✅ |
| P2 🟠 | `P/management/stats/StatsManager.java:67-112` | `onJoin`/`onLeave` sind **synchrone DB-Queries auf dem Main-Thread**. Bei `DatabaseException` wird nichts gecacht, danach löst **jedes** Move- und Block-Event eine neue synchrone Query aus. Mit einer MySQL, die in den Timeout läuft, steht der Server. | ✅ |
| P3 🟠 | `P/spigot/command/VillageCommand.java:28` | `locateNearestStructure(…, 5000, true)`: Radius 5000 **Chunks**, synchron, plus synchroner Chunk-Load. | ✅ |
| P4 🟡 | `P/management/challenges/CustomChallengesLoader.java:109-125` | Jede Trigger-Auslösung iteriert alle Custom-Challenges und allokiert eine neue `LinkedList`. | 🔎 |
| P5 🟡 | `P/challenges/type/abstraction/CollectionGoal.java:62-90` | Scoreboard-Kosten O(Spieler² × Items) mit String-Vergleichen. | 🔎 |
| P6 🟡 | `P/utils/misc/DatabaseHelper.java:53-83` | Ein `null`-Ergebnis wird gecacht, aber nie genutzt. Jeder Skull-Build für `MHF_ArrowLeft`/`Right` (Navigation) und für Offline-Spieler macht eine **synchrone DB-Query**. Der Cache ist eine `HashMap` mit Async-Zugriff. | ✅ |
| P7 🟡 | `P/utils/bukkit/nms/NMSUtils.java:31-43` | Bossbar-Titel (1.14–1.20.4): 3× `Class.forName` und 2× `getDeclaredMethod` pro Spieler und Update, ungecacht. | 🔎 |
| P8 🟡 | Library `CompatibilityUtils.getTopInventory` | Reflection-Lookup (`getMethod`) bei jedem Inventarklick, ungecacht. | ✅ |
| P9 🟡 | `P/spigot/command/SearchCommand.java:69-76` | Tab-Complete liefert alle ~1.300 Materialien ungefiltert, bei jedem Tastendruck. | ✅ |
| P10 🟡 | `C/effect/BlockEffectChallenge.java`, `ChunkRandomEffect…`, `IceFloor`, `OnlyDirt`, `MaxBiomeTime`, `ForceBiome` | Move-Handler ohne Early-Return bei gleichem Block, 6+ Block-Lookups pro Move. | 🔎 |
| P11 🟡 | `C/world/BedrockWallChallenge.java`, `SurfaceHoleChallenge.java` | ~380 Blockänderungen pro horizontalem Blockwechsel; mit Elytra >10k pro Sekunde und Spieler. | 🔎 |
| P12 🟡 | `C/inventory/NoDupedItemsChallenge.java` | O(Spieler²·Items²) mit `ArrayList.contains` bei jedem Klick und Pickup. | 🔎 |
| P13 ⚪ | 36 `PlayerMoveEvent`-Handler | Sind dauerhaft registriert, auch für deaktivierte Challenges. Jeder hat zwar einen Early-Return, trotzdem: Listener nur bei aktivierter Challenge registrieren. | ✅ |
| P14 ⚪ | `P/management/stats/StatsManager.java:126-173` | Die Rangliste lädt die **komplette Tabelle** und parst alle Stats-JSONs. Stats liegen als JSON-Blob in `varchar` und sind nicht per SQL sortierbar. `storeCached` schreibt alle 30 s *alle* Spieler, auch unveränderte. | ✅ |

### Allgemeine Empfehlungen
- **Chunks vergleichen:** Nicht mit `==`, sondern über Welt plus Chunk-Koordinaten (`x >> 4`). Ab 1.19.4 liefert `getChunk()` jedes Mal ein neues Objekt.
- **Große Blockoperationen:** `ChunkSnapshot` asynchron lesen, Änderungen gebündelt über mehrere Ticks schreiben, mit `applyPhysics=false`, und nur in geladenen Chunks (`isChunkLoaded`).
- **Material-Listen:** Einmal vorberechnen (`EnumSet`/`Set<String>`), statt `Material.values()` pro Event zu durchlaufen.
- **Datenbank:** Nie synchron auf dem Main-Thread (Join/Quit/Commands/Skull-Textures). Negative Treffer cachen und Stats nur als „dirty“ markiert speichern.

---

## 8. Threading (systemisches Problem)

**Ursache:**
- `@ScheduledTask.async()` und `@TimerTask.async()` stehen standardmäßig auf **`true`** (`P/management/scheduler/task/ScheduledTask.java:23`, `TimerTask.java:26`).
- `TimedChallenge` startet `onTimeActivation` standardmäßig **asynchron** (`P/challenges/type/abstraction/TimedChallenge.java:36-41`).
- 18 annotierte Tasks nutzen diesen Default. Mehrere Manager verlagern Arbeit sogar bewusst in Async-Threads.

**Betroffen (Auswahl):**

| Wo | Was off-thread passiert |
|----|------------------------|
| `PlayerInventoryManager.updateInventory` | Inventar-`setItem` (D4) |
| `ScoreboardManager.updateAll`, `handleQuit` | Scoreboards/Bossbars erstellen und ändern. `bossbars` ist eine `ArrayList`, die parallel iteriert und verändert wird (CME), und das Lambda liest ein mutables Feld (NPE) |
| `UncraftItems`, `HotBarRandomizer`, `RandomItem{Removing,Dropping,Swapping}` | Inventare, `dropItem` |
| `ChunkDeletion`, `ChunkDeconstruction`, `Tsunami` | `getChunk`, Block-Reads, Listen-Mutationen |
| `AlwaysRunning` (jeden Tick), `MoveMouseDamage`, `DontStopRunning` | `setVelocity`, HashMaps parallel zum Main-Thread |
| `BlockEffect`, `JumpAndRun`, `RaceGoal`, `ParticleUtils` | Potion-Effekte, Partikel, Bossbar |
| `ChatInputListener` | Statische `HashMap` + Callback auf dem Async-Chat-Thread |
| `LeaderboardCommand` / `StatsManager.getLeaderboard` | Sortiert die **geteilte** gecachte Liste parallel aus mehreren Async-Threads. Das führt zu CME oder falscher Rangliste |
| `DatabaseHelper.cachedTextures` | `HashMap` von Async- und Main-Thread |

Positiv: `ChallengeHelper.kill(...)` und `CustomChallenge.executeAction(...)` springen korrekt auf den Main-Thread zurück.

**Empfehlung:**
1. Den Default auf `async = false` bzw. `runAsync = false` umstellen. Async nur als bewusstes Opt-in für reine Berechnung oder IO.
2. Überall, wo async nötig ist, strikt „lesen/rechnen async → `runTask` → Bukkit-API sync“ umsetzen.
3. Geteilte Collections als `ConcurrentHashMap`/`CopyOnWriteArrayList` anlegen oder nur vom Main-Thread anfassen.
4. `AbstractTaskExecutor`: `Throwable` pro Funktion fangen, denn eine RuntimeException bricht sonst alle weiteren Funktionen dieses Ticks ab. `unregister` muss auch die Timer-Executor-Map bereinigen.

---

## 9. Datenbank, Persistenz und Stats

| ID | Wo | Problem | Status |
|----|----|---------|--------|
| DB1 🟠 | `P/management/database/DatabaseManager.java:91-98` | `CREATE TABLE` mit `varchar(36)+(16)+(500)+(1500)+(15000)`. Mit `utf8mb4` (Standard seit MySQL 8) sind das 17.052 × 4 = **68.208 Byte > 65.535 Byte Zeilenlimit**, also `ERROR 1118`: Die Tabelle wird nicht angelegt, und alle DB-Features sind still kaputt. Große Felder als `TEXT`/`MEDIUMTEXT` anlegen. *(Berechnet, nicht gegen eine echte MySQL-Instanz getestet.)* | ✅ |
| DB2 🟠 | Library `AbstractSQLDatabase` | `isConnected()` ruft `connection.isClosed()` auf, **ignoriert aber das Ergebnis**. Es gibt nur eine einzige JDBC-Verbindung ohne Reconnect oder Pool. Nach MySQLs `wait_timeout` (Standard 8 h, z. B. über Nacht bei leerem Server) funktioniert nichts mehr bis zum Neustart. Abhilfe: HikariCP oder Validierung plus Reconnect. | ✅ |
| DB3 🟠 | `P/management/stats/StatsManager.java`, `P/Challenges.java:197-221` | **Kein Flush beim Disable.** Plugins werden vor dem Kick der Spieler deaktiviert, der Quit-Handler läuft also nicht mehr. Bis zu 30 s Stats gehen bei jedem Stopp bzw. Reset verloren. | ✅ |
| DB4 🟡 | `StatsManager.getStats` | Ist `save-player-stats: true` gesetzt, aber `database.type: none` oder die Verbindung fehlgeschlagen, wirft `getDatabase()` ein `null` und damit eine **NPE in jedem Stats-Event** (nur `DatabaseException` wird gefangen). | ✅ |
| DB5 🟡 | `P/spigot/command/DatabaseCommand.java`, `ChallengeManager.saveSettings` | `/db save|load` machen synchrone DB-IO auf dem Main-Thread. Es gibt keine Größenprüfung vor `varchar(15000)`: Im Strict-Modus ist das ein Fehler, im Non-Strict-Modus **abgeschnittenes JSON**. | ✅ |
| DB6 🟡 | `P/management/files/ConfigManager.java:61-106` | Eine korrupte interne JSON-Datei führt zu `null` und dann zu einer NPE beim Laden, das Plugin startet nicht. `getDefaultConfig()` schreibt `defaultConfig.yml` ins **Server-Arbeitsverzeichnis** und schließt die Streams nicht. | 🔎 |
| DB7 ⚪ | `StatsListener.dragonDamager`, `ChallengeBossBar.bossbars`, `MenuPosition`, `BackCommand.lastLocations`, `StatsCommand.submitTimeByPlayer`, `CloudSupportManager`-Cache, `IceFloor`, `MoveMouseDamage` | Maps und Listen mit `Player` als Schlüssel werden nie aufgeräumt. Das sind Memory-Leaks; `dragonDamager` zählt nach einem Relog zudem doppelt. | ✅/🔎 |
| DB8 ⚪ | `StatsManager.store` | Setzt nur `stats`, nie `name`. Die Rangliste kann `null`-Namen zeigen, wenn der Datensatz nur über Stats angelegt wurde. | ✅ |

**Empfehlung Persistenz:**
- Periodisch speichern (z. B. alle 60 s): serialisieren auf dem Main-Thread, schreiben asynchron in eine Temp-Datei mit anschließendem atomarem Move.
- Zusätzlich beim Schließen des Rucksacks speichern.
- Stats in `handleDisable` synchron flushen, *bevor* die DB getrennt wird.

---

## 10. Build, CI, Abhängigkeiten, Versionskompatibilität

### B1 🔴 Build schlägt fehl: CloudNet-Repository umgezogen ✅
`plugin/pom.xml:110` zeigt auf `https://repo.cloudnetservice.eu/repository/releases/`, das jetzt 404 liefert. Die Artefakte liegen unter **`https://repo.cloudnetservice.eu/releases/`**; damit baut das Projekt vollständig.

### B2 🔴 CI-Workflow ist veraltet ✅
- **`maven-build.yml:25`:** `actions/upload-artifact@v3` ist seit dem 30.01.2025 von GitHub abgeschaltet → auf `@v4` umstellen.
- **`maven-build.yml:15-17`:** `actions/checkout@v2` und `actions/setup-java@v1` (ohne `distribution`) → `checkout@v4` und `setup-java@v4` mit `distribution: temurin` verwenden.
- **`qodana.yml`:** `JetBrains/qodana-action@v4.2.2` ist uralt, und mit `failThreshold: 999` schlägt der Check nie fehl.

### B3 🟡 Sonstiges Build/Repo ✅
- **`renovate.json`:** `baseBranches: ["development"]`, diesen Branch gibt es in diesem Fork nicht (nur `master`).
- **Shade ohne Relocation:** `net.anweisen.utilities` und im Mongo-Connector `com.mongodb` werden ohne `<relocations>` eingebettet. Andere Plugins mit derselben Library in anderer Version führen zu Klassenkonflikten.
- **Compiler-Version:** `source/target 1.8` sollte durch `<release>8</release>` ersetzt werden. Aktuell kompiliert es auch damit sauber (geprüft); `release` verhindert aber künftig versehentliche Java-9+-APIs.
- **Mongo-Treiber:** `mongodb-driver 3.12.8` ist EOL (2021).
- **`language/Language.iml`** ist eingecheckt, obwohl `*.iml` ignoriert wird.
- **Deprecations:** 96 Warnungen, u. a. `StructureType`, `PotionEffectType.getName()/values()`, `DamageModifier`, `getMaxHealth()`, `registerNewObjective(String,String,String)`, `new URL(String)`. Die meisten sind in Paper zum Entfernen markiert.

### B4 🟡 Versionskompatibilität (Plugin behauptet 1.14–1.21)
Kompiliert wird gegen 1.21. Fehlende Methoden fallen erst zur Laufzeit auf:

| Wo | Problem | Status |
|----|---------|--------|
| `C/world/ChunkDeletionChallenge.java` | `World#getMinHeight()` direkt aufgerufen → `NoSuchMethodError` auf 1.14–1.16 | 🔎 |
| `C/inventory/UncraftItemsChallenge.java:110` | `SmithingRecipe` (ab 1.16) hinter einer `V1_14`-Prüfung | 🔎 |
| `C/randomizer/RandomTeleportOnHitChallenge.java` | `LivingEntity#setInvisible` (ab 1.16.x) | 🔎 |
| `G/forcebattle/ExtremeForceBattleGoal.java:227` | `NamespacedKey.fromString` (ab 1.16.5) | 🔎 |
| `P/management/server/WorldManager.java` | `Player#getRespawnLocation()` (sehr neue API) | 🔎 |
| `P/utils/bukkit/nms/ReflectionUtil.java:64-87` | Versions-Parsing ohne Absicherung: `ArrayIndexOutOfBounds` → `ExceptionInInitializerError`; das neue Schema `26.1` wird als Major-Version 1 gelesen | 🔎 |
| `P/management/server/scoreboard/ChallengeScoreboard.java` | Sidebar-Einträge >40 Zeichen werfen auf 1.14–1.17 | 🔎 |
| `P/utils/item/ItemBuilder.java:330-341` | Skull-Texturen auf 1.20.5+ (`ResolvableProfile`) gehen mit DB verloren → `PlayerProfile`-API nutzen | 🔎 |
| `AbstractChallenge.java:155`, `BlockMaterialSetting.java:105` | `toLowerCase()` ohne `Locale`: Auf einer JVM mit türkischer Locale ändern sich die Settings-Keys | 🔎 |

Tipp: `@RequireVersion` **vor** der Instanziierung prüfen (Custom-Settings, G43), und ein CI-Job, der gegen 1.14/1.16/1.20.4/1.21 startet, würde diese Klasse von Fehlern früh finden.

---

## 11. Weitere Funde mit niedriger Priorität

### Commands

| Wo | Problem | Status |
|----|---------|--------|
| `BackCommand.java:63` | `/back 0` oder negativ → `list.get(-1)` → Exception. `lastLocations` hält `World`-Referenzen für immer. | ✅ |
| `DatabaseCommand.java:43,108` | `containsKey(args[1])` vor `toLowerCase()`. „settings save“ meldet „geladen“. `plugin.yml` nennt `config/custom_challenges` statt `settings/customs`. | ✅ |
| `InvseeCommand.java:75,135` | `inventories.remove(...)` innerhalb der `entrySet()`-Iteration → `ConcurrentModificationException`. Die `MenuPosition` des **Ziels** wird überschrieben. Die Ansicht aktualisiert sich nur bei Klick/Drop. | ✅ |
| `ResetCommand.java:74` | Eine ungültige Seed loggt einen Stacktrace, und **der Reset läuft trotzdem** mit zufälliger Seed. | ✅ |
| `TimeCommand.java:99,138` | `subtract` unter 0 → Syntaxfehler. „add“ fehlt im Tab-Complete. `plugin.yml` nennt `remove`. `getNearestTime` rechnet nicht zyklisch. `/time`, `/weather` sind nur für Spieler und überdecken die Vanilla-Befehle auch für die Konsole. | ✅ |
| `CommandHelper.java` | `@r` ohne Spieler → Exception. `Bukkit.getPlayer(name)` matcht per Präfix, das kann den falschen Spieler treffen (`getPlayerExact`). Completions sind ungefiltert. | ✅ |
| `StatsCommand.java` | Keine Timeouts beim Mojang-Lookup. `Map<Player,Long>` wird bei Fehlern nicht bereinigt. | ✅ |
| `WorldCommand.java` | `/world void` erzeugt die Welt synchron. | ✅ |
| `SkipTimerCommand.java` | Gibt keine Rückmeldung. | ✅ |
| `VillageCommand.java` | `setNoDamageTicks(10)` wirkt vor einer Verzögerung von 1 s, ist also wirkungslos. | ✅ |

### Sprachdateien (automatischer Abgleich) ✅

| Datei | Problem |
|-------|---------|
| `language/files/de.json:224` | Tippfehler im Key `"Chalheight-reached"` statt `"height-reached"`: Deutsche Server zeigen den rohen Key (`FirstPlayerAtHeightGoal`). |
| `en.json` | `item-mob-damage-teleport-challenge` fehlt (`RandomTeleportOnHitChallenge.java:47`). |
| `en.json` + `de.json` | `zero-hearts-failed` fehlt, betrifft aber nur die nie registrierte `ZeroHeartsChallenge`. |
| `P/content/impl/MessageImpl.java` | Im Small-Caps-Modus gehen Farben, Hover und Click der Komponenten verloren, und die Originalkomponente wird mutiert. |

### Diverses

| Wo | Problem | Status |
|----|---------|--------|
| `P/content/loader/*` | `IOUtils.toString(url)` öffnet ohne Connect/Read-Timeout (`url.openStream()`) und läuft synchron in `onLoad` (N+1 Downloads + Update-Check): Serverstart kann hängen. `readLanguage` schließt den Reader nicht. `LoaderRegistry` dekrementiert nicht im `finally`. | ✅ |
| `P/utils/misc/EntityUtils.java:21` | Die Z-Komponente wird aus `getX()` übernommen (FoodLaunch, PickupItemLaunch, BoostEntityInAir). | ✅ |
| `P/spigot/events/PlayerJumpEvent.java:36-39` | `setCancelled(false)` cancelt trotzdem. | ✅ |
| `P/spigot/listener/PlayerConnectionListener.java:97-100` | `start-on-first-join` setzt den Timer bei *jedem* Join fort und hebt damit manuelle Pausen auf. | 🔎 |
| `P/spigot/command/GodModeCommand.java:75` | Cancelt auch Void-Schaden: Man fällt endlos. | ✅ |
| `P/management/inventory/PlayerInventoryManager.java` | Der Slot aus `hotbar-items.yml` wird nicht validiert (AIOOBE). `getHand()` wird nicht geprüft. `removeItems` matcht nur Typ und Namen. | ✅/🔎 |
| `P/challenges/implementation/setting/PregameMovementSetting.java:51-54` | Unter y=0 (1.18+) wird der Spieler bei jedem Move einen Block hochgeschoben. | 🔎 |
| `DeathMessageSetting.java:73-76` | Im „Vanilla“-Modus wird die Nachricht vor dem Lesen genullt, sie erscheint also nicht. | 🔎 |
| `SplitHealthSetting.java:96` | `return` statt `continue`; tote und ignorierte Spieler senken die geteilte Gesundheit. | 🔎 |
| `GetFullHealthGoal.java:37-49` | `setHealth` über dem Maximum wirft; `SettingModifier.setEnabled` hat kein try/catch. | 🔎 |
| `KillMobsGoal.java:26` | `entitiesKilled` ist `static` und wird von allen Unterklassen geteilt. | 🔎 |
| `PickupItemLaunch`, `FoodLaunch` | `getValue() / 2` als Ganzzahl-Division. | 🔎 |
| `DamageTeleportChallenge`, `ZeroHeartsChallenge` | Toter Code (nie registriert), beide mit eigenen Bugs. | 🔎 |
| `P/utils/bukkit/misc/BukkitStringUtils.java:65-147` | NPE bei `§L`/`§x` bzw. einem einzelnen `§` (z. B. aus einer Custom-Sprachdatei). O(n²) String-Aufbau. | 🔎 |
| `P/management/cloud/*` | Unsynchronisierte `HashMap`; `hasNameFor` macht bei jedem `getName` einen API-Call. NPE bei Gruppe/Farbe `null`. | 🔎 |

---

## 12. Architektur und Code-Qualität

1. **Threading-Modell klar definieren** (siehe §8). Das ist der größte strukturelle Hebel: Sync als Standard, Async explizit.
2. **Zentrale Schutzfunktionen:**
   - ein `ProtectedBlocks.isProtected(block)` (Portale, Rahmen, Bedrock, Spawner, Tile-Entities) für alle blockverändernden Challenges
   - ein gemeinsamer „sicherer Menü“-Listener (K2)
   - `ChallengeHelper.getIngamePlayers()` konsequent statt `Bukkit.getOnlinePlayers()`
3. **`ignorePlayer`/`shouldExecuteEffect` fehlen häufig.** Das ist die häufigste Fehlerklasse (NoDupedItems, Force-Challenges, DamageInventoryClear, Block-Break/Place-Damage, RandomPotionEffect, MobsRespawnInEnd, JumpAndRun). Ein Basis-Handler oder Lint-Check wäre sinnvoll.
4. **`PlayerInventoryClickEvent` sollte `Cancellable` implementieren.** Aktuell ist `ignoreCancelled = true` dafür wirkungslos, und gecancelte Klicks (z. B. in geschützten Truhen) zählen trotzdem für Force-Challenges.
5. **Gamestate-Verträge vereinheitlichen:**
   - `loadGameState` sollte immer erst leeren (PointsGoal, Backpack, Snake und NetherPortal tun es nicht).
   - `writeGameState` sollte einen expliziten Zustand speichern (Force-Challenges).
   - `TimedChallenge` sollte `originalSecondsUntilActivation` mitspeichern (sonst `IllegalArgumentException` in `RandomChallengeChallenge`).
6. **Duplizierte Hierarchien:**
   - `Modifier`/`SettingModifier`, `ModifierCollectionGoal`/`SettingModifierCollectionGoal` und `NumberSubSetting`/`NumberAndBooleanSubSetting` wiederholen Validierung, Klicks und Laden/Speichern.
   - `Setting.setEnabled` fängt Exceptions, `SettingModifier` nicht.
   - `handleShutdown` ruft `onDisable` je nach Basisklasse bedingt oder unbedingt auf.
   - Virtuelle Aufrufe im Konstruktor (`BooleanSubSetting`, CutClean-Iron/Gold) führen zu nie registrierten Drops.
7. **Spieler-Referenzen:** Maps und Listen mit `UUID` statt `Player` als Schlüssel anlegen, beim Quit aufräumen, und in Tasks `isOnline()` prüfen.
8. **Gespeicherte Daten als nicht vertrauenswürdig behandeln:** Custom-Challenges, Positionen und Force-Ziele beim Laden validieren und Einträge einzeln in try/catch laden, damit ein kaputter Eintrag nicht den ganzen Zustand zerstört.
9. **Tests:**
   - Es gibt keine (`**/src/test` steht sogar in `.gitignore`).
   - Für Logik wie Seeds, Intervall-Trigger, Border-Mathematik, Gamestate-Round-Trips und Whitelist-Validierung lohnen sich Unit-Tests, z. B. mit MockBukkit.
   - Ein CI-Smoke-Test, der den Server in mehreren Versionen startet, würde Kompatibilitätsbrüche finden.
10. **Kleinkram:**
    - Tippfehler im Klassennamen `IntervallTrigger`.
    - Irreführende Methodennamen: `onBlockPlace(BlockBreakEvent)`, `getLowestBreakableBlock` liefert den höchsten.
    - Einrückung mit gemischten Tabs und Leerzeichen (z. B. `Challenges.java`, `DatabaseManager.java`).
    - Magic Numbers.
    - `Message.UNKNOWN_MESSAGES` ist eine nicht threadsichere `ArrayList`.

---

## 13. Empfohlene Reihenfolge

**Phase 1 – Sofort (klein, große Wirkung):**
1. CloudNet-Repo-URL und `upload-artifact@v4` korrigieren (B1, B2).
2. Command-Action: bei Ausführung und Laden validieren, eigene Permission, nicht als Konsole ausführen (K1, S1).
3. Eigener Menü-Schutz-Listener für Drag und Collect, `MenuPosition` beim Quit entfernen (K2).
4. Einzeiler-Fixes:
   - HydraPlus-Limit
   - Timber `isEmpty`
   - AllBlocksDisappear doppelter Drop
   - LoopChallenge `isTool`
   - `chance::remove`
   - Seed-Autoboxing
   - `"void"`-Löschung absichern
   - CheatListener
   - WaterAllergy `continue`
   - MaxBiome/MaxHeight `>=`
   - IntervallTrigger 240
   - SoupSetting Hand
   - CutClean nur Typ ändern und Spieler auslassen
   - `drop-items-on-end`
   - `resume()` ohne Spectator

**Phase 2 – Kurzfristig:**
5. Scheduler-Default auf sync umstellen, und alle Stellen aus §8 durchgehen.
6. Stats und DB: nichts synchron auf dem Main-Thread, Flush beim Disable, Negativ-Cache, Reconnect/Pool, Schema auf `TEXT` (DB1–DB5).
7. Persistenz: periodisches Speichern von Gamestate und Timer, Backups der Extra-Welt persistieren, Late-Joiner cachen (D5, D13).
8. Lag-Challenges entschärfen: RepeatInChunk, Tsunami, AllBlocksDisappear, ChunkDeletion, RandomEvent, LoopChallenge-Caps (K3, P-Tabelle).

**Phase 3 – Mittelfristig:**
9. Logikfehler der Force-/Goal-/Challenge-Tabellen (§6) abarbeiten.
10. Versionskompatibilität (B4) mit einer CI-Matrix absichern.
11. Refactoring der Basisklassen und des Gamestate-Vertrags (§12), plus erste Unit-Tests für die kritische Logik.

---

*Alle 🔎-Funde sind begründet und mit Code-Stelle belegt. Vor einem Fix sollte man die Stelle trotzdem kurz gegenlesen. Die ✅-Funde habe ich selbst am Code nachvollzogen.*
