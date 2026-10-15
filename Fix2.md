## 5. Wahrscheinlich müssen wir den Ladepfad in den Bindings korrigieren

Deine Vermutung mit dem expliziten Laden ist durchaus sinnvoll. Es gibt allerdings einen wichtigen Unterschied: Die Datei `libLeapC.so` ist die native LeapC-Bibliothek, während `_leapc_cffi` das kompilierte Python-Erweiterungsmodul ist. Beide müssen korrekt geladen werden.

Die offizielle Implementierung versucht, `leapc_cffi` zu importieren und greift bei Problemen auf den SDK-Pfad zurück. Außerdem ist die Standard-SDK-Position `/usr/lib/ultraleap-hand-tracking-service`.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub

+1



Bevor wir eine eigene Ladefunktion schreiben, sollten wir die konkrete Stelle ansehen, an der dein Build die Bibliothek referenziert. Denn wenn das kompilierte Modul bereits einen falschen absoluten Pfad enthält, wird `LD_LIBRARY_PATH` allein nicht unbedingt helfen.

## 6. Bitte diese Diagnosen ausführen

Wir müssen jetzt den tatsächlichen Importfehler sehen. Führe diese Befehle in deinem aktivierten virtuellen Environment aus:

```
cd ~/projects/ultraleap-python/leapc-python-bindings
source ~/projects/ultraleap-python/.venv/bin/activate

# 1. Welche Python-Dateien und nativen Module sind installiert?
find "$VIRTUAL_ENV/lib/python3.14/site-packages/leapc_cffi" \
    -maxdepth 2 -type f -printf '%f\n'

# 2. Wo liegt die native Bibliothek?
file "$VIRTUAL_ENV/lib/python3.14/site-packages/leapc_cffi/libLeapC.so"

# 3. Welche Abhängigkeiten hat sie?
ldd "$VIRTUAL_ENV/lib/python3.14/site-packages/leapc_cffi/libLeapC.so"

# 4. Woher importiert Python das Modul?
python -c "import leapc_cffi; print(leapc_cffi.__file__)"
```

Falls der letzte Befehl fehlschlägt, ist gerade der vollständige Traceback besonders wichtig.

### 7. Wenn du das Laden explizit in Python erzwingen möchtest

Für eine native Bibliothek kannst du grundsätzlich `ctypes.CDLL()` verwenden. Das ist allerdings zunächst ein Diagnosetest und ersetzt nicht das kompilierte CFFI-Modul, das die Ultraleap-Bindings benötigen.

```
import ctypesimport oslibrary_path = os.path.expanduser(    "~/projects/ultraleap-python/.venv/"    "lib/python3.14/site-packages/"    "leapc_cffi/libLeapC.so")leapc_library = ctypes.CDLL(library_path)print("Loaded:", leapc_library)
```

Wenn dieser Test erfolgreich ist, kann Linux die Bibliothek grundsätzlich laden. Wenn er scheitert, nennt der Fehler oft die fehlende Abhängigkeit oder ein anderes Ladeproblem.

Wichtig: `ctypes.CDLL()` allein macht die Funktionen noch nicht kompatibel mit dem Python-Modul `leap`. Dafür brauchen wir weiterhin die korrekte CFFI-Bindung.

Schicke mir bitte die Ausgaben aus Abschnitt 6. Dann können wir entscheiden, ob wir den Build korrigieren, den SDK-Pfad in `leapc-python-api` anpassen oder das native Modul gezielt laden müssen.
