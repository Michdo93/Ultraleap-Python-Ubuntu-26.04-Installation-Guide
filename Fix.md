Ja, das kann an einem Suchpfad liegen. `PATH` ist dafür aber nicht die richtige Variable. Unter Linux müssen wir zwei unterschiedliche Dinge auseinanderhalten:

- `PATH`: findet ausführbare Programme.
- `PYTHONPATH`: findet Python-Module.
- `LD_LIBRARY_PATH`: hilft dem dynamischen Linker, native `.so`-Bibliotheken und deren Abhängigkeiten zu finden.

Die offiziellen Ultraleap-Bindings verwenden das kompilierte Modul `leapc_cffi`. Außerdem erwarten sie standardmäßig das Leap SDK unter `/usr/lib/ultraleap-hand-tracking-service`.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub

+1



Da du `libLeapC.so` bereits in deiner virtuellen Umgebung gefunden hast, gehen wir am besten systematisch vor, statt gleich globale Umgebungsvariablen zu setzen.

## 1. Zunächst testen, ob die Bibliothek geladen werden kann

Aktiviere deine virtuelle Umgebung:

```
source ~/projects/ultraleap-python/.venv/bin/activate
```

Prüfe anschließend die Abhängigkeiten der Bibliothek:

```
ldd "$VIRTUAL_ENV/lib/python3.14/site-packages/leapc_cffi/libLeapC.so"
```

Achte besonders auf Zeilen mit `not found`. Falls dort eine Bibliothek fehlt, ist das zunächst ein Problem ihrer nativen Abhängigkeiten und nicht des Python-Modulpfads.

## 2. Den tatsächlichen Fehler unterscheiden

Führe bitte diesen Test aus:

```
python - <<'PY'
import os
import importlib.util

path = os.path.join(
    os.environ["VIRTUAL_ENV"],
    "lib/python3.14/site-packages/leapc_cffi"
)

print("Library directory:", path)
print("Contents:", os.listdir(path))
print("leapc_cffi module:", importlib.util.find_spec("leapc_cffi"))

try:
    import leapc_cffi
    print("leapc_cffi imported:", leapc_cffi.__file__)
    print("LeapC library:", leapc_cffi.libleapc)
except Exception:
    import traceback
    traceback.print_exc()
PY
```

Damit sehen wir, ob das Python-Paket selbst importiert werden kann oder ob erst beim Laden der nativen Bibliothek ein Fehler entsteht.

## 3. Falls tatsächlich `libLeapC.so` nicht gefunden wird

Teste vorübergehend `LD_LIBRARY_PATH`, ohne etwas dauerhaft zu verändern:

```
export LD_LIBRARY_PATH="$VIRTUAL_ENV/lib/python3.14/site-packages/leapc_cffi${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"

python -c "import leapc_cffi; print(leapc_cffi.__file__)"
```

Wenn das funktioniert, kennen wir zumindest einen relevanten Suchpfad. Falls nicht, brauchen wir die genaue Fehlermeldung.

Beachte: Das Python-Modul muss unter Umständen die Bibliothek explizit aus seinem eigenen Paketverzeichnis laden. Eine Umgebungsvariable ist daher nicht zwingend die endgültige Lösung.

## 4. Auch die Ultraleap-Installationsposition prüfen

Die offizielle Dokumentation sieht unter Linux standardmäßig dieses Verzeichnis vor:

```
ls -la /usr/lib/ultraleap-hand-tracking-service/
```

Prüfe außerdem, ob die Bibliothek dort liegt:

```
find /usr/lib /usr/share/doc \
    -name 'libLeapC.so*' 2>/dev/null
```

Wenn `libLeapC.so` sowohl im virtuellen Environment als auch im systemweiten Ultraleap-Verzeichnis vorhanden ist, sollten wir nicht vorschnell eine der Dateien verschieben oder kopieren. Die Python-Bindings unterstützen eine abweichende SDK-Position über `LEAPSDK_INSTALL_LOCATION`.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub

+1



Schicke mir bitte insbesondere die vollständige Fehlermeldung von `import leapc_cffi` und die Ausgabe von `ldd`. Dann können wir die korrekte Lösung für Ubuntu 26.04 und Python 3.14 festlegen, ohne die Installation unnötig zu verändern.
