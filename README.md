# Ultraleap-Python-Ubuntu-26.04-Installation-Guide

Für deine Installation unter Ubuntu 26.04 brauchst du im Wesentlichen drei Komponenten:

1. Ultraleap Tracking Service – verarbeitet die Kameradaten und berechnet die Handpositionen und Fingerbewegungen.
2. LeapC Python Bindings – ermöglichen deinem Python-Programm, die Trackingdaten abzurufen.
3. USB-Zugriff und Geräteberechtigungen – damit die Ultraleap-Kamera korrekt erkannt wird.

Zunächst installiert man die `Tracking Software`:

```
wget https://s3.eu-west-1.amazonaws.com/downloads.ultraleap.com/software/tracking-software/6.2.0/tracking-software-linux-x64-6.2.0.deb
sudo apt install -y tracking-software-linux-x64-6.2.0.deb
```

Wichtig ist, dass die Installation der Datei `tracking-software-linux-x64-6.2.0.deb` allein noch nicht bedeutet, dass Python bereits auf die Trackingdaten zugreifen kann. Ultraleap verwendet dafür die LeapC-API, die mit dem Tracking Service kommuniziert.

Außerdem gibt es einen wichtigen Kompatibilitätspunkt: Ultraleaps Linux-Dokumentation nennt Ubuntu 22.04 LTS als unterstützte Plattform. Ubuntu 26.04 liegt außerhalb dieser dokumentierten Unterstützung. Wir sollten daher zuerst prüfen, ob die installierte Software und ihre Bibliotheken unter deiner Ubuntu-Version korrekt laufen, bevor wir weitere Pakete oder eigene udev-Regeln hinzufügen.&#x20;

## 1. Zuerst die vorhandene Installation prüfen

Bitte führe die folgenden Befehle im Terminal aus. Damit stellen wir fest, welche Dateien das `.deb` installiert hat, ob der Dienst läuft und welche Python-Version verfügbar ist.

```
cat /etc/os-release
uname -m
python3 --version

dpkg -l | grep -i ultraleap
dpkg -L tracking-software-linux-x64 2>/dev/null
```

Falls der Paketname anders lautet, ermitteln wir ihn mit:

```
dpkg -l | grep -Ei 'leap|ultraleap'
```

Danach prüfen wir den Dienst und die Bibliotheken:

```
systemctl list-units --type=service --all | grep -Ei 'leap|ultraleap'

ldconfig -p | grep -Ei 'LeapC|Leap'
```

Und schließlich die USB-Geräte:

```
lsusb
```

Wichtig: Erstelle noch keine eigenen udev-Regeln. Die benötigten Berechtigungen hängen davon ab, wie das Paket die Kamera einbindet. Zunächst müssen wir feststellen, ob bereits passende Regeln installiert wurden.

## 2. Python vorbereiten

Ultraleap stellt offizielle Python-Bindings für die LeapC-API bereit: [ultraleap/leapc-python-bindings auf GitHub](https://github.com/ultraleap/leapc-python-bindings). Sie benötigen die installierte Tracking-Software und eine kompatible Python-Umgebung. Die Bibliothek verwendet außerdem eine native CFFI-Komponente.

Installiere zunächst die üblichen Build-Werkzeuge und Python-Komponenten:

```
sudo apt update
sudo apt install -y \
    python3-venv \
    python3-pip \
    python3-dev \
    build-essential \
    git \
    pkg-config
```

Lege anschließend eine isolierte Python-Umgebung an:

```
mkdir -p ~/projects/ultraleap-python
cd ~/projects/ultraleap-python

python3 -m venv .venv
source .venv/bin/activate

python -m pip install --upgrade pip setuptools wheel build cffi
```

Die virtuelle Umgebung verhindert, dass wir System-Python-Pakete verändern. Ob sich die native Komponente auf Ubuntu 26.04 ohne Anpassungen kompilieren lässt, müssen wir noch testen.

## 3. Python-Bindings herunterladen und installieren

Wenn die Prüfungen oben erfolgreich waren, kannst du die offiziellen Bindings installieren:

```
cd ~/projects/ultraleap-python

git clone https://github.com/ultraleap/leapc-python-bindings.git
cd leapc-python-bindings

source ~/projects/ultraleap-python/.venv/bin/activate

python -m pip install -r requirements.txt
python -m pip install -e leapc-python-api
```

Die Requirements enthalten unter anderem `build`, `cffi`, `opencv-python` und `numpy`.

Teste anschließend den Import:

```
python -c "import leap; print('Leap Python API imported successfully')"
```

Falls dabei ein Fehler wie `ModuleNotFoundError: No module named 'leapc_cffi._leapc_cffi'` auftritt, fehlt wahrscheinlich das passende native Python-Modul für deine Python-Version. Ultraleap dokumentiert dafür einen Build aus dem Quellcode.

Führe dann gegebenenfalls Folgendes aus:

```
cd ~/projects/ultraleap-python/leapc-python-bindings

source ~/projects/ultraleap-python/.venv/bin/activate

python -m build leapc-cffi
python -m pip install leapc-cffi/dist/leapc_cffi-0.0.1.tar.gz
python -m pip install -e leapc-python-api
```

Falls dieser Build fehlschlägt, sollten wir die konkrete Fehlermeldung untersuchen, statt weitere Bibliotheken auf Verdacht zu installieren.

## 4. Tracking Service und Lizenz überprüfen

Prüfe zunächst, welche Verwaltungsprogramme tatsächlich installiert sind:

```
command -v leapctl
leapctl --help
```

Falls `leapctl` vorhanden ist, zeigt dir das Hilfsprogramm die unterstützten Verwaltungsfunktionen. Bei Installationen, die diesen Befehl bereitstellen, kannst du damit unter anderem den Lizenzstatus prüfen bzw. die EULA akzeptieren:

```
sudo leapctl eula
```

Führe den letzten Befehl nur aus, wenn die EULA noch nicht akzeptiert wurde und du den Lizenzbedingungen zustimmst. Die genaue Verfügbarkeit der Befehle hängt von der installierten Softwareversion ab.

Prüfe außerdem:

```
systemctl --no-pager --type=service --all | grep -Ei 'leap|ultraleap'
```

Wenn du den Namen des Dienstes kennst, kannst du seinen Status gezielt abfragen:

```
systemctl status DIENSTNAME
```

Ersetze `DIENSTNAME` durch den tatsächlich installierten Servicenamen.

## 5. USB und udev-Regeln

Eine eigene udev-Regel ist nicht automatisch erforderlich. Entscheidend ist, ob die installierte Software bereits die passende Geräteerkennung und Berechtigungen konfiguriert.

Prüfe zunächst die vorhandenen Regeln:

```
grep -RilE 'ultraleap|leap.?motion' \
    /etc/udev/rules.d \
    /usr/lib/udev/rules.d \
    /lib/udev/rules.d 2>/dev/null
```

Prüfe danach, ob die Kamera erkannt wird:

```
lsusb
```

Wenn sie dort erscheint, ermittle die USB-ID aus der Ausgabe, beispielsweise `xxxx:yyyy`. Mit dieser ID lässt sich feststellen, ob eine vorhandene Regel zur Kamera passt.

Bitte nicht pauschal `MODE="0666"` setzen. Das würde den Zugriff auf das betreffende Gerät unnötig weit öffnen. Falls Berechtigungen fehlen, sollten wir stattdessen eine gezielte Regel nach Hersteller- und Produkt-ID erstellen.

## 6. Erstes Python-Testprogramm

Lege in `~/projects/ultraleap-python/leapc-python-bindings` zunächst eine Datei `test_tracking.py` an:

```
import leapimport timeprint("Leap module:", leap.__file__)print("Python API loaded successfully.")print("Next: run the official tracking event example.")
```

Starte es in der virtuellen Umgebung:

```
source ~/projects/ultraleap-python/.venv/bin/activate
python test_tracking.py
```

Dieser Test prüft zunächst nur, ob die Python-API importiert werden kann. Er bestätigt noch nicht, dass die Kamera Trackingdaten liefert.

Für den tatsächlichen Trackingtest verwendest du anschließend das offizielle Beispiel:

```
python examples/tracking_event_example.py
```

Das Beispiel ist im offiziellen Repository enthalten.

## 7. Was du am Ende benötigst

| Komponente                        | Zweck                                      |
| --------------------------------- | ------------------------------------------ |
| Tracking Software 6.2.0           | Kameraverarbeitung und Tracking Service    |
| `libLeapC.so`                     | Native Tracking-API                        |
| Python 3 und virtuelle Umgebung   | Laufzeit für dein Programm                 |
| `cffi` und `leapc-cffi`           | Verbindung zwischen Python und LeapC       |
| USB-Erkennung und passende Rechte | Zugriff auf die Kamera                     |
| Offizielle Python-Bindings        | Zugriff auf Hände, Finger und Gelenke      |
| Optional: OpenCV und NumPy        | Bildverarbeitung und numerische Auswertung |

Die zentrale Dokumentation ist der [LeapC API Guide](https://docs.ultraleap.com/api-reference/tracking-api/leapc-guide.html). Er beschreibt, wie die Anwendung Tracking-, Status- und Bildereignisse vom Service erhält.&#x20;
