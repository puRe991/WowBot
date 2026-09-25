# WowBot

Ein experimenteller Bot für **World of Warcraft Classic (Client 1.12.x, 32-Bit, Windows)**, geschrieben in Python.

Der Bot liest den Speicher des laufenden `WoW.exe`-Prozesses aus und schleust Assembler-Code ein, um interne Spielfunktionen direkt aufzurufen (Ziel wählen, drehen, laufen, Lua ausführen, zaubern). Das Verhalten wird über Zustandsautomaten (State Machines) gesteuert – aktuell für einen **Krieger zum Grinden** und einen **Heiler, der dem Gruppenleiter folgt**.

> ⚠️ **Status:** Frühes Hobby-/Lernprojekt, nicht produktionsreif. Mehrere Versionen der State Machine existieren parallel, einige Skripte sind unvollständig.
>
> ⚠️ **Hinweis:** Bots verstoßen gegen die Nutzungsbedingungen von Blizzard und vieler privater Server und können zur Sperrung des Accounts führen. Nutzung auf eigene Verantwortung, z. B. auf einem lokalen Testserver.

---

## Funktionsweise

```
WoW.exe (1.12)                                   Python
┌──────────────────────┐   ReadProcessMemory   ┌──────────────────────┐
│ Objektliste, Spieler,│ ────────────────────► │ ObjectManager        │
│ Party, Descriptors   │                       │  Player / UnitObject │
│                      │                       └─────────┬────────────┘
│ EndScene (Hook) ─────┼─► Code-Cave            ┌─────────▼────────────┐
│   ruft SetTarget,    │ ◄──────────────────── │ State Machine        │
│   SetFacing, DoString│   WriteProcessMemory   │ (Healer / Warrior)   │
│   usw. auf           │   + pyfasm-Assembler   └──────────────────────┘
└──────────────────────┘
```

1. Der WoW-Prozess wird mit `PROCESS_ALL_ACCESS` geöffnet (benötigt `SeDebugPrivilege`).
2. Der `ObjectManager` durchläuft die verkettete Objektliste des Spiels und erzeugt `UnitObject`- bzw. `Player`-Wrapper. Deren Werte (Position, HP, Mana, Ziel …) sind Methoden und werden bei jedem Aufruf frisch aus dem Speicher gelesen.
3. Aktionen werden als Assembler-Text formuliert, mit `pyfasm` assembliert und über einen Hook auf die DirectX-Funktion **EndScene** im Haupt-Thread des Spiels ausgeführt (`Injector` in `memoryManips.py`).
4. Eine State Machine entscheidet periodisch, welche Aktion als Nächstes ausgeführt wird.

## Projektstruktur

| Datei | Inhalt |
|---|---|
| `memoryManips.py` | Windows-API über `ctypes`: Prozess öffnen, Speicher lesen/schreiben, EndScene-Hook, `Injector` für Code-Injektion |
| `constants.py` | Speicheradressen und Offsets für Client 1.12 (Objekt-Manager, Descriptors, Spielfunktionen, Opcodes, Movement-Flags, Klassen-IDs) |
| `GameObjects.py` | `Location`, `Object`, `UnitObject`, `Player` (Aktionen wie `SetTarget`, `turnCharacter`, `walkToTarget`, `doString`, `cast`) und `ObjectManager` (Objektliste, Party, nächstes Ziel) |
| `spellMachine.py` | `Timer` (Cooldowns/GCD), `Spell`, `SpellBook` sowie Zauberlisten für Krieger und Priester |
| `stateMachine.py` | Klassische State Machine (`Enter` / `Execute` / `Exit`) |
| `stateMachineV2.py` | Hierarchische State Machine (`HSM`), Zustände liefern in `next()` den Folgezustand |
| `stateMachineV3.py` | Neueste Version mit globalem Zustand und `ChangeState` / `RevertToPreviousState` |
| `States.py` | Gemeinsame Zustände (Folgen, Pfad ablaufen, Gebiet säubern) und `MovementData` |
| `healerStates.py` | Heiler-Logik: Leader folgen, Gruppe heilen (`Lesser Heal` unter 50 % bzw. 80 % HP), Schaden machen |
| `warriorGrindStates.py`, `warriorGrindStatesV2.py` | Krieger-Grind-Logik (V2 basiert auf `stateMachineV3`) |
| `command.py` | Ansatz für eine Konsole zur Verwaltung mehrerer WoW-Instanzen/Rollen (unvollständig) |
| `memReader.py`, `combat.py` | Hilfs- bzw. Platzhalter-Skripte |
| `test.py` | Testskript: verbindet sich mit WoW, baut den Objekt-Manager auf und gibt die Spielerposition aus |
| `source.cpp` | Früherer C++-Prototyp der Hook-/Speicherlogik |

### Grind-Ablauf (Krieger)

```
MovingToNextLoc ──angegriffen──► Combat ──Gegner tot──► Recovering
       ▲                                                    │
       └──────Gebiet leer──── ClearArea ◄────HP voll────────┘
```

## Voraussetzungen

- Windows
- World of Warcraft Client **1.12.x** (32-Bit) – alle Adressen in `constants.py` gelten nur für diese Version
- Python 3.7 (32-Bit empfohlen, passend zum Client)
- Python-Pakete:

```bash
pip install pywin32 pyfasm psutil numpy uptime
```

- Das Skript muss mit Administratorrechten laufen (wegen `SeDebugPrivilege`).

## Benutzung

1. WoW starten und einloggen.
2. Die Prozess-ID von `WoW.exe` ermitteln (z. B. im Task-Manager) und in `test.py` bei `GetProcessA(<PID>)` eintragen.
   Alternativ `GetProcess()` verwenden, das das Fenster „World of Warcraft“ sucht.
3. Ausführen:

```bash
python test.py
```

Zum Starten einer Rolle lässt sich im Anschluss eine State Machine erzeugen und in einer Schleife ausführen, z. B. für den Krieger (`stateMachineV3`):

```python
from warriorGrindStatesV2 import warriorSMFactory
import time

sm = warriorSMFactory(my_object_manager)
while True:
    sm.Update()
    time.sleep(0.5)
```

bzw. für den Heiler (`stateMachineV2`):

```python
from healerStates import healerSMFactory

sm = healerSMFactory(my_object_manager).createHealerGeneralSM()
while True:
    sm.next()
    time.sleep(0.5)
```

Welche Zauber verfügbar sind, wird in `SpellBook` (`spellMachine.py`) festgelegt – standardmäßig `getSpellsWarrior()`; für den Heiler auf `getSpellsPriest()` umstellen.

## Bekannte Einschränkungen / TODO

- PID ist in `test.py` fest eingetragen.
- `command.py` ist nicht lauffähig (z. B. fehlt `self` in `runSM`, `hprocess` ist undefiniert).
- Drei parallele State-Machine-Versionen; Migration auf `stateMachineV3` ist in Arbeit.
- Keine Wegfindung – Bewegung erfolgt geradlinig zu Wegpunkten.
- Offsets sind hart kodiert und funktionieren nur mit Client 1.12.
- `__pycache__` ist eingecheckt und sollte per `.gitignore` ausgeschlossen werden.
