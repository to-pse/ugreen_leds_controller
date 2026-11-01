# Debian 13: LED-Treiber und Disk-Monitor installieren

Diese Anleitung beschreibt einen **manuellen Installationsweg aus diesem Checkout** für Debian 13 auf amd64. Die einzelnen Komponenten und der Betrieb nach einem Reboot wurden auf einem UGREEN DXP4800 Pro getestet; eine vollständige Neuinstallation anhand dieser Anleitung wurde **nicht** separat durchgeführt. Die Hardwareergebnisse und ihre Grenzen stehen in [`b5-reboot-test.md`](b5-reboot-test.md).

**Vor Eingriffen:** Befehle mit `sudo` ändern das System. Nur auf einem geeigneten System und nach Prüfung der vorhandenen Installation ausführen. Keine RAID-Signaturen verändern, keine Datenträger entfernen und keine I²C-Geräte löschen oder unbinden. Bei Fehlermeldungen anhalten statt mit dem nächsten Schritt fortzufahren.

## 1. System und vorhandene Dienste prüfen

Vorausgesetzt werden Debian 13, ein passender I801-SMBus-Adapter, ein unterstützter LED-Controller, Internet-/Paketquellenzugriff und ein Checkout, der die in dieser Anleitung genannten Dateien enthält. Der Disk-Monitor benötigt Python 3.9 oder neuer und Kernel-pidfd-Unterstützung; Linux 5.3 oder neuer wird vorausgesetzt.

```bash
# Benutzer-Shell: Debian-Version, Architektur und aktiven Kernel feststellen.
cat /etc/os-release
dpkg --print-architecture
uname -r

# Benutzer-Shell: vorhandene UGREEN-Dienste und Monitorprozesse anzeigen.
systemctl list-units --all 'ugreen*' --no-pager
systemctl list-unit-files 'ugreen*' --no-pager
ps -eo pid,ppid,pgid,stat,args --forest | grep -E '[u]green-diskiomon|[u]green-blink-disk|[u]green-led-init|[u]green-probe-leds'
```

Die grep-Abfrage kann bei keinen Treffern Status 1 liefern. Auf einem **frischen** System dürfen keine konkurrierenden Disk-Monitore oder parallelen Initialisierungsdienste laufen. Vorhandene Installationen, Konfigurationsdateien und Units zuerst untersuchen und sichern; diese Anleitung überschreibt sie nicht ungeprüft. Insbesondere sind die frühere lokale `ugreen-led-init.service` und `ugreen-diskiomon-hotplug.service` **keine** Bestandteile des vorgesehenen Neuinstallationswegs.

```bash
# Benutzer-Shell: belegte Zielpfade anzeigen; bei einem Treffer zunächst anhalten.
for f in /usr/src/led-ugreen-0.3 /etc/modules-load.d/ugreen-led.conf \
  /usr/bin/ugreen-probe-leds /usr/bin/ugreen-diskiomon /usr/bin/ugreen-diskiomon-notify \
  /usr/bin/ugreen-blink-disk /usr/bin/ugreen-check-standby \
  /etc/systemd/system/ugreen-probe-leds.service /etc/systemd/system/ugreen-diskiomon.service \
  /usr/lib/udev/rules.d/99-ugreen-diskiomon.rules /etc/ugreen-leds.conf; do
  if [ -e "$f" ] || [ -L "$f" ]; then ls -ld "$f"; fi
done
```

## 2. Debian-Pakete und passende Kernel-Headers

Auf Debian 13 amd64 benötigen der DKMS-Build und die späteren Monitorfunktionen unter anderem folgende Pakete:

```bash
# Root-Rechte: Paketlisten und benötigte Werkzeuge/Headers installieren.
sudo apt update
sudo apt install build-essential dkms linux-headers-amd64 "linux-headers-$(uname -r)" \
  i2c-tools python3 util-linux coreutils smartmontools

# Benutzer-Shell: Build-Verzeichnis für den laufenden Kernel prüfen.
readlink -f "/lib/modules/$(uname -r)/build"
```

`linux-headers-amd64` ist das Debian-Metapaket für künftige Standard-amd64-Kernel; das zusätzliche versionsgenaue Paket betrifft den **jetzt laufenden** Kernel. Bei abweichendem oder eigenem Kernel müssen passende Headers aus dessen Quelle verfügbar sein. Wenn Paket oder Build-Verzeichnis fehlen, nicht fortfahren.

## 3. Modul `led-ugreen` über DKMS 0.3 installieren

[`kmod/dkms.conf`](kmod/dkms.conf) definiert `PACKAGE_VERSION="0.3"` und `AUTOINSTALL="yes"`. Die folgenden Befehle setzen voraus, dass die Shell im Repository-Root steht; `/pfad/zum/checkout` muss durch den tatsächlich gewählten Pfad ersetzt werden.

```bash
# Benutzer-Shell: den ausgewählten Checkout betreten und Versionsstand prüfen.
cd /pfad/zum/checkout
grep -E '^(PACKAGE_NAME|PACKAGE_VERSION|AUTOINSTALL)=' kmod/dkms.conf

# Nicht über einen bereits vorhandenen Quellbaum installieren.
test ! -e /usr/src/led-ugreen-0.3 || { echo 'DKMS source already exists; inspect it first' >&2; exit 1; }

# Root-Rechte: nur die benötigten DKMS-Quellen installieren.
sudo install -d -m 0755 /usr/src/led-ugreen-0.3
sudo install -m 0644 kmod/led-ugreen.c kmod/led-ugreen.h kmod/Makefile kmod/dkms.conf /usr/src/led-ugreen-0.3/

# Root-Rechte: Version 0.3 registrieren, für den aktiven Kernel bauen und installieren.
sudo /usr/sbin/dkms add -m led-ugreen -v 0.3
sudo /usr/sbin/dkms build -m led-ugreen -v 0.3 -k "$(uname -r)"
sudo /usr/sbin/dkms install -m led-ugreen -v 0.3 -k "$(uname -r)"

# Benutzer-Shell: DKMS- und Modulstand prüfen.
/usr/sbin/dkms status led-ugreen
/usr/sbin/modinfo -k "$(uname -r)" led-ugreen
```

Erwartet wird `installed` für Version 0.3 und den aktiven Kernel sowie ein auflösbares Modul. Ein erfolgreicher Build bedeutet nicht automatisch, dass der Kernel das Modul auch akzeptiert.

### Secure Boot

Bei aktiviertem Secure Boot kann der Kernel ein nicht vertrauenswürdig signiertes DKMS-Modul ablehnen. Den Signierungs-/MOK-Prozess der eigenen Debian-Installation prüfen und gegebenenfalls einen vertrauenswürdigen Schlüssel registrieren; eine Schlüsselregistrierung kann eine interaktive Bestätigung beim Boot benötigen. Signaturfehler nicht durch Deaktivieren von Sicherheitsfunktionen „reparieren“, ohne die Folgen zu verstehen. Nach erfolgreichem Laden `lsmod` und das Kerneljournal kontrollieren.

### Modul-Autoload einrichten und prüfen

Der manuelle DKMS-Weg erstellt nicht selbst die Autoload-Datei. Nicht über eine bereits vorhandene gleichnamige Datei schreiben:

```bash
# Benutzer-Shell: bei vorhandener Datei erst Inhalt und Herkunft prüfen.
test ! -e /etc/modules-load.d/ugreen-led.conf || { echo 'Autoload configuration already exists; inspect it first' >&2; exit 1; }

# Root-Rechte: benötigte Module für den Systemstart vormerken.
printf 'i2c-dev\nled-ugreen\nledtrig_oneshot\n' | sudo tee /etc/modules-load.d/ugreen-led.conf >/dev/null

# Root-Rechte: das DKMS-Modul für die sofortige Kontrolle laden.
sudo modprobe led-ugreen

# Benutzer-Shell: geladenes Modul und vorhandene Konfiguration kontrollieren.
lsmod | grep -E '^(led_ugreen|i2c_dev|ledtrig_oneshot)\b'
cat /etc/modules-load.d/ugreen-led.conf
```

Vor dem ersten Probe-Lauf müssen nicht bereits alle drei Module geladen sein: `ugreen-probe-leds` lädt `i2c-dev` und `ledtrig_oneshot` selbst. `i2c_i801` wird vom passenden System/PCI-Gerät bereitgestellt. `lsmod` zeigt keine fest in den Kernel eingebauten Treiber; bei abweichenden Kernelkonfigurationen auch die tatsächliche Modul-/Adapterverfügbarkeit prüfen.

### Kernel-Updates

`AUTOINSTALL=yes` ermöglicht den DKMS-Neubau, garantiert ihn aber nicht: Für den neuen Kernel müssen passende Headers verfügbar und Build sowie gegebenenfalls Signierung erfolgreich sein. Vor dem Boot in einen neuen Kernel dessen DKMS-Status prüfen; nach dem Boot erneut `uname -r`, `/usr/sbin/dkms status led-ugreen`, `/usr/sbin/modinfo -k "$(uname -r)" led-ugreen`, `lsmod` und die Dienstjournale kontrollieren. Bei fehlgeschlagenem Build oder fehlender Signatur nicht von funktionsfähiger LED-Initialisierung ausgehen.

## 4. Initialisierungsskript und Probe-Unit installieren

```bash
# Root-Rechte, aus dem Repository-Root: Skript und versionierte Unit installieren.
sudo install -m 0755 scripts/ugreen-probe-leds /usr/bin/ugreen-probe-leds
sudo install -m 0644 scripts/systemd/ugreen-probe-leds.service /etc/systemd/system/ugreen-probe-leds.service
```

Die Unit ist `Type=oneshot` und verwendet `RemainAfterExit=true`. Das Skript lädt `i2c-dev`, `led-ugreen` und `ledtrig_oneshot`, ermittelt genau einen Adapter anhand von `SMBus I801 adapter`, registriert `led-ugreen` an `0x3a` nur bei Bedarf und prüft Gerätenamen, Treiberbindung sowie mindestens eine zu diesem Controller gehörende Disk-LED. Bei keinem oder mehreren Adaptern, fremdem Gerät/Treiber oder fehlenden LEDs bricht es ab. Der im Testsystem beobachtete Bus `i2c-0` ist ein **Hardwarebefund**, keine allgemeine Voraussetzung.

Ein erfolgreicher Skriptaufruf kann bei fehlendem Controller tatsächlich eine I²C-Registrierung auslösen und ist daher kein rein lesender Test. Keine parallele lokale Initialisierungs-Unit betreiben. Die frühere NAS-spezifische `ugreen-led-init.service` nutzte `/usr/local/libexec/ugreen-led-init` und fest Bus 0; der hier vorgesehene versionierte Probe-Weg ersetzt sie, ohne eine zweite Initialisierung einzuführen.

## 5. Disk-Monitor und Helper installieren

```bash
# Root-Rechte, aus dem Repository-Root: Helper vor dem Monitor installieren.
sudo install -m 0755 scripts/ugreen-diskiomon-notify /usr/bin/ugreen-diskiomon-notify
sudo install -m 0755 scripts/ugreen-diskiomon /usr/bin/ugreen-diskiomon
sudo install -m 0644 scripts/systemd/ugreen-diskiomon.service /etc/systemd/system/ugreen-diskiomon.service

# Benutzer-Shell: optionale, im validierten NAS vorhandene C++-Programme bauen.
BUILD_DIR=$(mktemp -d)
g++ -std=c++17 -O2 scripts/blink-disk.cpp -o "$BUILD_DIR/ugreen-blink-disk"
g++ -std=c++17 -O2 scripts/check-standby.cpp -o "$BUILD_DIR/ugreen-check-standby"

# Root-Rechte: die gebauten Programme installieren.
sudo install -m 0755 "$BUILD_DIR/ugreen-blink-disk" /usr/bin/ugreen-blink-disk
sudo install -m 0755 "$BUILD_DIR/ugreen-check-standby" /usr/bin/ugreen-check-standby
```

Das temporäre `BUILD_DIR` wird bewusst nicht automatisch gelöscht; seinen Inhalt bei Bedarf nach eigener Prüfung entfernen. Der Monitor hat einen Bash-I/O-Fallback; der oben installierte C++-Monitor ist der auf dem Testsystem verwendete Weg.

Für eine frische Installation kann die folgende **minimale Testkonfiguration** angelegt werden. Sie entspricht den für B5 dokumentierten relevanten Werten; SMART, ZFS und Standby sind dabei absichtlich deaktiviert. Eine bereits vorhandene `/etc/ugreen-leds.conf` vorher prüfen und sichern, nicht überschreiben.

```bash
# Benutzer-Shell: nicht über eine bestehende Konfiguration schreiben.
test ! -e /etc/ugreen-leds.conf || { echo 'Configuration already exists; inspect it first' >&2; exit 1; }

# Root-Rechte: minimale Konfiguration für den ersten Disk-Monitor-Test anlegen.
sudo tee /etc/ugreen-leds.conf >/dev/null <<'EOF'
MAPPING_METHOD=ata
CHECK_SMART=false
CHECK_ZPOOL=false
STANDBY_MON_PATH=/nonexistent/ugreen-check-standby
BLINK_MON_PATH=/usr/bin/ugreen-blink-disk
LED_REFRESH_INTERVAL=0.1
LED_INVERT=1
BRIGHTNESS_DISK_LEDS=128
EOF
```

Die vollständigen Optionen und Alternativen für `ata`, `hctl`, `serial`, SMART, ZFS, Standby, Farben und Helligkeit stehen in [`scripts/ugreen-leds.conf`](scripts/ugreen-leds.conf). Vor Aktivierung weiterer Prüfungen deren Auswirkungen und Gerätezuordnung gesondert beurteilen. Die obige minimale Konfiguration ersetzt keine allgemeine SMART-Überwachung.

## 6. udev-Regel installieren und laden

```bash
# Root-Rechte, aus dem Repository-Root: Regel an den im Paket verwendeten Ort installieren.
sudo install -D -m 0644 scripts/99-ugreen-diskiomon.rules /usr/lib/udev/rules.d/99-ugreen-diskiomon.rules

# Root-Rechte: Regeln für zukünftige Events neu laden; keine Events künstlich auslösen.
sudo udevadm control --reload

# Benutzer-Shell: Regeldatei statisch prüfen.
udevadm verify /usr/lib/udev/rules.d/99-ugreen-diskiomon.rules
```

Die Regel benachrichtigt bei passenden `add`-/`remove`-Events ganzer SATA-Blockgeräte `/usr/bin/ugreen-diskiomon-notify`; Partitionen werden ausgeschlossen. Der Helper prüft PID, Startzeit und Instanz-Token und verwendet pidfds, statt eine möglicherweise wiederverwendete PID blind zu signalisieren. Der Hauptmonitor verarbeitet `SIGUSR1`, bereinigt seine bisherigen Überwachungsprozesse und enumeriert erneut. Er bleibt bei leeren Einschüben aktiv. Es wird kein zweiter Disk-Monitor und kein zusätzlicher systemd-Hotplug-Dienst benötigt.

## 7. systemd-Konfiguration neu einlesen

```bash
# Root-Rechte: neue Unit-Dateien einlesen.
sudo systemctl daemon-reload

# Benutzer-Shell: Unit-Syntax und vorgesehene Abhängigkeiten prüfen.
systemd-analyze verify /etc/systemd/system/ugreen-probe-leds.service /etc/systemd/system/ugreen-diskiomon.service
systemctl cat ugreen-probe-leds.service ugreen-diskiomon.service
systemctl show ugreen-diskiomon.service -p Requires -p After -p ExecCondition -p Restart -p RestartUSec
```

Die Disk-Unit verlangt mit `Requires=` den Probe-Dienst und startet mit `After=` erst nach ihm. Bei einem fehlschlagenden Probe-Dienst darf der Monitor nicht als erfolgreich initialisiert gelten. Die Disk-Unit verwendet `Restart=on-failure` und `RestartSec=3` sowie eine `ExecCondition` gegen `/run/ugreen-diskiomon.cleanup-failed`. Bei vorhandener Markierung wird der Start übersprungen: **Die Überwachung läuft dann nicht.** Zuerst Journal und verbliebene Prozesse prüfen; die Markierung nicht blind löschen. `/run` ist flüchtig und die Markierung überlebt einen vollständigen Reboot nicht.

## 8. Dienste aktivieren und starten

Nur wenn keine konkurrierenden alten Initialisierungsdienste oder Disk-Monitore laufen und die vorherigen Prüfungen erfolgreich waren:

```bash
# Root-Rechte: Probe-Dienst für den Boot aktivieren und einmalig ausführen.
sudo systemctl enable --now ugreen-probe-leds.service

# Benutzer-Shell: Probe-Erfolg kontrollieren, bevor der Monitor gestartet wird.
systemctl status ugreen-probe-leds.service --no-pager
sudo journalctl -u ugreen-probe-leds.service -b -n 80 --no-pager
```

Erwartet: `active (exited)`, erfolgreicher Exit, genau ein I801-Adapter, passende Bindung und Disk-LEDs. Bei Fehler nicht fortfahren.

```bash
# Root-Rechte: nur nach erfolgreicher Probe den Disk-Monitor aktivieren und starten.
sudo systemctl enable --now ugreen-diskiomon.service
```

Ein regulärer `systemctl stop` startet den Dienst nicht automatisch neu. Ein unerwarteter Fehler kann entsprechend `Restart=on-failure` einen Neustart auslösen. Diese Anweisungen aktivieren keine frühere lokale `ugreen-led-init.service` und keinen `ugreen-diskiomon-hotplug.service`.

## 9. Funktion und Zuordnung prüfen

```bash
# Benutzer-Shell: Unit-Zustand und automatische Neustarts prüfen.
systemctl show ugreen-probe-leds.service ugreen-diskiomon.service \
  -p Id -p ActiveState -p SubState -p UnitFileState -p Result -p MainPID -p NRestarts

# Root-Rechte: vollständiges Dienstjournal lesen.
sudo journalctl -b -u ugreen-probe-leds.service -u ugreen-diskiomon.service --no-pager

# Benutzer-Shell: Geräte, Prozesshierarchie und I801-Adapter ermitteln.
lsblk -o NAME,PATH,TYPE,TRAN,FSTYPE,MOUNTPOINTS,MODEL,SERIAL,HCTL
ps -eo pid,ppid,pgid,stat,args --forest | grep -E '[u]green-diskiomon|[u]green-blink-disk|[u]green-probe-leds'
for f in /sys/bus/i2c/devices/i2c-*/name; do printf '%s: ' "${f%/name}"; cat "$f"; done
ls -l /sys/class/leds/disk*
```

Erwartet sind ein erfolgreich beendeter Probe-Dienst und ein laufender Hauptmonitor. Bei mindestens einer gültig zugeordneten SATA-Platte läuft der Blink-Monitor mit der zugehörigen LED; bei leeren Einschüben bleibt der Hauptmonitor aktiv. Die physische Zuordnung muss am tatsächlichen Gerät überprüft werden.

Auf dem getesteten DXP4800 Pro lag der I801-Adapter auf `i2c-0`. Nur **wenn die obige Adapterprüfung auf dem eigenen System Bus 0 ergibt**, kann die Bindung wie folgt gelesen werden:

```bash
# Benutzer-Shell: nur für den zuvor bestätigten Bus 0.
cat /sys/bus/i2c/devices/0-003a/name
readlink -f /sys/bus/i2c/devices/0-003a/driver
```

Bei anderem Bus die tatsächliche Gerätekennung verwenden, nicht blind `0-003a` anlegen. Der Treiberlink muss auf `led-ugreen` zeigen. Keine I²C-Bus-Scans, I/O-Schreibtests oder physischen Hotplug-Vorgänge ohne separate Sicherheitsprüfung und Freigabe.

## 10. Reboot-Test und Nachkontrolle

Ein Reboot unterbricht den NAS-Betrieb und benötigt eine gesonderte Freigabe sowie einen Wiederherstellungszugang.

```bash
# Root-Rechte: nur nach ausdrücklicher Freigabe neu starten.
sudo systemctl reboot
```

Nach dem Boot die lesenden Prüfungen aus Schritt 9 wiederholen. Zusätzlich den Kernel-/DKMS-Stand aus Schritt 3 und die LED-Werte prüfen:

```bash
# Benutzer-Shell: aktiven Kernel, DKMS und Disk-LED-Zustände kontrollieren.
uname -r
/usr/sbin/dkms status led-ugreen
lsmod | grep -E '^(led_ugreen|i2c_dev|ledtrig_oneshot)\b'
for led in /sys/class/leds/disk[1-8]; do
  if [ -d "$led" ]; then
    printf '%s brightness=' "${led##*/}"
    cat "$led/brightness"
    printf 'trigger='
    cat "$led/trigger"
  fi
done
```

Erwartet: Probe-Dienst erfolgreich vor dem Disk-Monitor, ein aktiver Hauptmonitor ohne Neustartschleife, korrekte Geräte-/LED-Zuordnung und kein Cleanup-Fehler. Gerätenamen, PID und `DISKSEQ` können sich bei einem Boot ändern; die Hardware anhand stabiler Identität und physischem Einschub prüfen. Bei Fehlern zuerst Status und Journal untersuchen, keine fremden Prozesse blind beenden und keine Hardwarezustände automatisch verändern.

## Debian-Paketvariante und Grenzen

Ein `led-ugreen-dkms`-Paket baut das Modul mittels DKMS; `led-ugreen-utils` aus diesem Checkout enthält Probe-Skript, Disk-Monitor, Notify-Helper, udev-Regel und kompilierte Hilfsprogramme; `led-ugreen-systemd` installiert die versionierten Units. Die Build-Skripte liegen unter [`build-scripts/debian/`](build-scripts/debian/) und sind in dessen [README](build-scripts/debian/README.md) beschrieben. Der Systemd-Paket-Postinst kann Dienste aktivieren und starten; Paketinhalt und Maintainer-Skripte vor einer Installation auf einem aktiven System prüfen.

**Nicht als validiert behauptet:** Eine komplette Neuinstallation nach dieser Anleitung, eine Installation der Debian-Pakete aus diesem Checkout, Kernel-Update-/Secure-Boot-Durchlauf, alle UGREEN-Modelle und alle Mapping-/SMART-/ZFS-/Standby-Kombinationen wurden nicht separat auf Hardware getestet. Der konkrete B5-Erfolg ist in [`b5-reboot-test.md`](b5-reboot-test.md) abgegrenzt.
