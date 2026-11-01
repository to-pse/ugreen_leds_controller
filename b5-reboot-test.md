# Hardwarevalidierung und Reboot-Test B5

**Ergebnis:** Der Reboot-Test B5 wurde am **1. November 2026** auf einem UGREEN DXP4800 Pro mit Debian 13 erfolgreich abgeschlossen. Dieser Bericht dokumentiert tatsächlich berichtete Tests und ergänzt sie um eine nach dem Boot vorgenommene lesende Zustandsprüfung. Er behauptet keine vollständige Neuinstallation oder Prüfung aller Hardware-/Kernelvarianten.

## 1. Ausgangslage und Sicherung

Vor der **B2-Umstellung auf den erweiterten Originalmonitor** waren der lokale Referenzmonitor `ugreen-diskiomon-hotplug.service` (`active`, `enabled`) und die lokale `ugreen-led-init.service` (`active`, `enabled`) in Betrieb. Der Referenzmonitor wurde bei B2 abgelöst und deaktiviert; vor der späteren Migration der LED-Initialisierung war bereits `ugreen-diskiomon.service` aktiv und der Referenzmonitor inaktiv und disabled. Der bisherige Monitor, seine Unit und die lokale LED-Initialisierung wurden gesichert; die älteren Sicherungen und Hashlisten werden unverändert aufbewahrt. Ein separates Sicherungsarchiv enthält Originaleigentümer und Dateimodi; bei einer Wiederherstellung müssen diese erhalten bleiben.

## 2. Validierung vor dem Reboot

Folgende Ergebnisse wurden vom Testdurchführenden berichtet und bereits in früheren Projektberichten dokumentiert:

1. Installation des erweiterten Disk-Monitors und seiner udev-/Notify-Komponenten; Unit- und Hash-Prüfungen erfolgreich.
2. Zuordnung der Testplatte `ata1 → sda → disk1`; bei einem Lesetest blinkte die LED während I/O und leuchtete anschließend dauerhaft.
3. SIGUSR1-Refresh, Notify-Helper und ein Burst mit zehn Benachrichtigungen erfolgreich; Prozessbereinigung ohne verwaiste Monitorprozesse und ohne Cleanup-Fehlermarkierung.
4. Der SIGTERM-Fix wurde separat installiert und durch einen Stopptest mit `Result=success` und `ExecMainStatus=0` bestätigt.
5. Physischer Hotplug **vor dem späteren LED-Refresh-Fix**: am 1. November um 15:13:24 WD60EFRX aus Einschub 1 entfernt; der Monitor blieb aktiv, erkannte keine ATA-Laufwerke mehr und LED 1 ging aus. Um 15:15:17 wurde dieselbe Platte wieder eingesetzt und automatisch `ata1 → sda → disk1` zugeordnet; genau ein Blink-Prozess lief danach, ohne Cleanup-Fehler.
6. Nach dem physischen Test wurde das kurzzeitige Aufleuchten leerer Einschübe im Refresh-Pfad korrigiert. Der Fix bestand 18 isolierte Tests der Schreibreihenfolge. Nach Installation zeigte ein **Notify-Refresh ohne erneutes physisches Hotplug** nur an LED 1 ein kurzes Flackern; LEDs 2–4 blieben dunkel.
7. Das überarbeitete `ugreen-probe-leds` wurde an einem **bereits registrierten und korrekt gebundenen Controller** erfolgreich auf realer Hardware getestet. Anschließend erfolgte die Migration auf die versionierten Units; systemd-Validierung sowie manueller Dienststart waren erfolgreich. Beide versionierten Dienste wurden für Autostart aktiviert und die lokale `ugreen-led-init.service` deaktiviert. Der Referenzmonitor war bereits seit B2 deaktiviert.

Die vor dem Reboot festgestellten PIDs, Hashes und Unit-Zustände sind zeitgebundene Momentaufnahmen.

## 3. B5: erfolgreicher Reboot-Test

### Plattform und Modul

| Merkmal | Ergebnis |
|---|---|
| System | Debian 13, Kernel `6.12.113+deb13-amd64` |
| DKMS | `led-ugreen/0.3` für den laufenden Kernel als `installed` ausgewiesen |
| Geladene Module | `led_ugreen`, `i2c_dev`, `ledtrig_oneshot`, `i2c_i801` |
| Adapter | Ein I801-Adapter als `i2c-0`, `SMBus I801 adapter at efa0` |
| Controller | `0-003a`, Name `led-ugreen`; Treiberlink auf `/sys/bus/i2c/drivers/led-ugreen` |

Die Adapterzuordnung `i2c-0` ist ein Befund dieses Geräts, keine allgemeine Voraussetzung für andere UGREEN-Systeme.

### systemd und Reihenfolge

- `ugreen-probe-leds.service`: **enabled**, automatisch gestartet, `active (exited)`, `Result=success`, `ExecMainStatus=0`.
- `ugreen-diskiomon.service`: **enabled**, automatisch gestartet, `active (running)`, `Result=success`, `NRestarts=0`.
- Die Disk-Unit verlangt den Probe-Dienst und startet nach ihm. Die lesend abgefragten systemd-Zeitstempel zeigen den Abschluss des Probe-Hauptprozesses um **15:57:25 CET** und den Start des Disk-Hauptprozesses anschließend ebenfalls um **15:57:25 CET**; die monotonen Zeitstempel unterscheiden die Reihenfolge (Probe-Ende `7127950`, Disk-Start `7169746` Mikrosekunden nach Boot).
- Die früheren lokalen Units `ugreen-led-init.service` und `ugreen-diskiomon-hotplug.service` waren nach dem Boot inaktiv und disabled.
- Bei der Nachkontrolle lief genau ein Hauptmonitor (PID `863`) mit einem Blink-Prozess (PID `880`) für `sda disk1`; zusätzlich lief der vorgesehene untergeordnete Bash-Monitorprozess. PIDs sind Momentaufnahmen.
- Eine `/run/ugreen-diskiomon.cleanup-failed`-Markierung war nicht vorhanden. Die Disk-Unit hat `ExecCondition`, `Restart=on-failure` und `RestartSec=3`.

### Laufwerk und LEDs

- Testplatte: `WDC WD60EFRX-68MYMN1`, nach dem Boot `/dev/sda`, ATA-Pfad `ata1`; Zuordnung `sda → disk1`.
- Der neue udev-Gerätestand meldete `DISKSEQ=3`. Vor dem Reboot war `DISKSEQ=5`; die Nummer ist boot-/geräteinstanzbezogen und keine dauerhafte Identität.
- `disk1`: Helligkeit **128**, Trigger **`oneshot`**.
- `disk2`, `disk3`, `disk4`: Helligkeit jeweils **0**, Trigger jeweils **`none`**.
- `/dev/sda2` trägt eine alte `linux_raid_member`-Signatur. Für diesen Bericht wurde sie nicht verändert.

## 4. Einordnung und Grenzen

- Der separat durchgeführte Probe-Hardwaretest betrifft den **bereits registrierten, korrekt gebundenen** Controller.
- B5 bestätigt den erfolgreichen **Boot-Initialisierungspfad als Ganzes**: Probe-Service erfolgreich, Controller gebunden und LEDs vorhanden, danach Disk-Monitor erfolgreich gestartet. Der alleinige erfolgreiche Endzustand und die Unit-Zeitstempel beweisen nicht unabhängig, ob der Controller unmittelbar vor dem Probe-Aufruf noch unregistriert war oder welche Komponente ihn registrierte. Der Neuregistrierungszweig wurde isoliert simuliert, aber nicht durch gezieltes Löschen einer I²C-Registrierung auf Hardware erzwungen.
- Der physische Remove-/Add-Test wurde vor dem LED-Fix durchgeführt. Nach dem Fix ist ein Notify-Refresh auf Hardware bestätigt; ein erneuter physischer Remove-/Add-Test nach dem Fix wurde nicht durchgeführt.
- Eine vollständige frische Debian-13-Neuinstallation nach [`debian13-installation.md`](debian13-installation.md), die Debian-Paketinstallation aus diesem Checkout, ein Kernel-Update und der Secure-Boot-/MOK-Ablauf wurden nicht separat validiert.
- Andere UGREEN-Modelle, alle drei Mapping-Verfahren und alle SMART-/ZFS-/Standby-Kombinationen wurden durch B5 nicht auf Hardware validiert. Die B5-Konfiguration hatte SMART, ZFS und Standby deaktiviert.
- Die durchgeführten Reboot-Ergebnisse und die erfolgreiche systemd-Validierung wurden vom Testdurchführenden gemeldet; die aktuellen Unit-Zustände, Zeitstempel, Geräte, Module und LED-Werte wurden für diesen Bericht lesend geprüft. Ein vollständiges privilegiertes Systemjournal war aus der nichtprivilegierten Sitzung nicht unabhängig einsehbar.

## 5. Aktueller reproduzierbarer Ansatz

Die dokumentierte Zielarchitektur verwendet die versionierten `ugreen-probe-leds.service` und `ugreen-diskiomon.service`, nicht die lokale `ugreen-led-init.service` oder den lokalen Referenzmonitor. Probe initialisiert den I801-/I²C-Controller; der Disk-Monitor startet nach erfolgreicher Probe und verarbeitet passende udev-Ereignisse über den pidfd-basierten Notify-Helper. Die ausführbare Schrittfolge für eine geplante Neuinstallation steht in [`debian13-installation.md`](debian13-installation.md); sie ist eine Anleitung, keine Behauptung eines bereits absolvierten Frischinstallationstests.
