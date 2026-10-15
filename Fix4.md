Perfekt. Damit haben wir einen wichtigen Befund: `libLeapC.so` lässt sich laden, wenn du den absoluten Pfad explizit angibst. Das Problem liegt also wahrscheinlich in der Art, wie die native Bibliothek von den Ultraleap-Python-Bindings geladen wird.

Allerdings ist `ctypes.CDLL()` zunächst nur ein zusätzlicher Ladevorgang. Wir müssen noch sicherstellen, dass das CFFI-Modul dieselbe Bibliothek verwendet und anschließend tatsächlich Trackingdaten empfängt.

## 1. Prüfe, wie die Bindings die Bibliothek laden

Wechsle in das Repository:

```
cd ~/projects/ultraleap-python/leapc-python-bindings
```

Suche nach der Import- und Ladeimplementierung:

```
grep -RInE \
    'leapc_cffi|libLeapC|LEAPSDK_INSTALL_LOCATION|dlopen|ffi\.dlopen' \
    leapc-python-api leapc-cffi
```

Besonders interessant sind Stellen, an denen `leapc_cffi` importiert oder `libLeapC.so` geladen wird.

## 2. Den expliziten Ladepfad vorerst sauber kapseln

Wenn du den funktionierenden Workaround im Beispiel behalten möchtest, würde ich ihn nicht dauerhaft auf einen einzelnen absoluten Pfad festlegen. Verwende stattdessen den Pfad des aktiven Python-Environments:

```
import ctypes
import os
import sys

VENV_LIBRARY_PATH = os.path.join(
    sys.prefix,
    "lib",
    "python3.14",
    "site-packages",
    "leapc_cffi",
    "libLeapC.so",
)

leapc_library = ctypes.CDLL(VENV_LIBRARY_PATH)
```

Das funktioniert in deiner aktuellen Umgebung, solange die Bibliothek dort installiert bleibt.

Wichtig: Dieser Code allein ist noch keine endgültige Reparatur der Bindings. Falls das CFFI-Modul die Bibliothek selbst nochmals lädt, müssen wir den dortigen Ladepfad korrigieren oder sicherstellen, dass es die bereits geladene Bibliothek wiederverwendet.

## 3. Als Nächstes das eigentliche Tracking testen

Starte jetzt das Beispiel mit deinem Workaround und prüfe, ob Hand-Tracking-Ereignisse eintreffen. Dass die Bibliothek geladen wird, bedeutet noch nicht, dass der Tracking Service läuft oder eine Kamera erkannt wurde.

Wenn du mir die Ausgabe des `grep`-Befehls aus Abschnitt 1 schickst, kann ich dir zeigen, an welcher Stelle wir den Ladepfad in den offiziellen Bindings dauerhaft korrigieren sollten, anstatt den Workaround in jedes Beispiel einzubauen.
