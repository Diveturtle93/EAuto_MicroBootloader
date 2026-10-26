# Runbook EAuto MicroBootloader

Handgriffe am Steuergerät (BMS oder Motorsteuergerät): Bootloader erstmalig aufspielen, Firmware über
CAN aktualisieren und ein Steuergerät mit defekter Application wiederherstellen. Wie der Bootloader
arbeitet, steht in der [Architektur](ARCHITECTURE.md), wie er gebaut wird, in der
[README](README.md#bauen-und-flashen).

Das Aktualisieren über CAN steht im
[Runbook von STM32F7_MicroBoot](https://github.com/Diveturtle93/STM32F7_MicroBoot/blob/develop/RUNBOOK.md).
Die Bibliothek hat noch keinen Tag, der Link zeigt bis dahin auf `develop`. Für diese Steuergeräte gilt:

| Platzhalter | Wert |
|-------------|------|
| `‹CAN-Adapter›` | PEAK PCAN-USB |
| `‹CAN-Stecker›` | Antriebs-CAN des Steuergeräts (CAN3, PA8 = RX, PA15 = TX) |
| `‹Reset›` | Versorgung des Steuergeräts aus- und wieder einschalten |
| `‹Taster›` | nicht vorhanden, `PC13` ist auf beiden Platinen `POWER_ON`. Die Prozedur „Gerät mit defekter Application wiederherstellen“ der Bibliothek gilt hier nicht, sie steht unten |
| `‹LED›` | blaue LED an `PB7`. Blinkt im derzeitigen Stand nicht, siehe [README, Hinweise](README.md#hinweise) |
| `‹Firmware›` | `.srec` der Application aus EAuto_BMS bzw. EAuto_Motor, gebaut nach [README](README.md#application-für-den-bootloader-bauen) |

Feste Werte: CAN 500 kBit/s, Host → Gerät `0x7E0`, Gerät → Host `0x7E1`, Standard-IDs (11 Bit).

## Bootloader erstmalig programmieren

### Voraussetzungen
- ST-Link an SWD des Steuergeräts
- STM32CubeProgrammer oder STM32CubeIDE
- `Release/EAuto_MicroBootloader.elf`
- erste Application des Steuergeräts als `.elf` oder `.srec`, gebaut für `0x08008000` mit Info-Tabelle

### Schritte
1. ST-Link anschließen, Steuergerät versorgen.
2. Gesamten Flash löschen:
   `STM32_Programmer_CLI -c port=SWD mode=UR -e all`
3. Bootloader schreiben:
   `STM32_Programmer_CLI -c port=SWD mode=UR -w Release/EAuto_MicroBootloader.elf -v`
4. Erste Application schreiben, ohne vorher zu löschen:
   `STM32_Programmer_CLI -c port=SWD mode=UR -w ‹Application›.elf -v -rst`
5. ST-Link abziehen.

### Prüfung
- Das Steuergerät startet nach etwa 500 ms die Application und verhält sich wie gewohnt.
- Ein Update nach [Firmware mit MicroBoot aktualisieren](https://github.com/Diveturtle93/STM32F7_MicroBoot/blob/develop/RUNBOOK.md#firmware-mit-microboot-aktualisieren)
  mit derselben `‹Firmware›` läuft durch.

### Fehlerbilder

| Symptom | Ursache | Abhilfe |
|---------|---------|---------|
| `Error: No STM32 target found` | SWD nicht verbunden oder Steuergerät ohne Versorgung | Verkabelung und Versorgung prüfen |
| Application startet nicht, Update über CAN wird vor dem Löschen abgelehnt | Schritt 4 ausgelassen, ohne Application nimmt der Bootloader keine Firmware über CAN an | Schritt 4 nachholen |
| Schritt 4 überschreibt den Bootloader | Application für `0x08000000` gelinkt | Application nach README mit `ORIGIN = 0x8008000` bauen, ab Schritt 2 wiederholen |
| Steuergerät schaltet sich ab oder startet die Application nie | Pegel an `PC13` (`POWER_ON`) passt nicht zum Bootloader | Nicht am Gerät lösbar, Entwickler informieren (siehe [README, Hinweise](README.md#hinweise)) |

## Firmware über CAN aktualisieren

Prozedur [Firmware mit MicroBoot aktualisieren](https://github.com/Diveturtle93/STM32F7_MicroBoot/blob/develop/RUNBOOK.md#firmware-mit-microboot-aktualisieren)
oder [Firmware mit BootCommander aktualisieren](https://github.com/Diveturtle93/STM32F7_MicroBoot/blob/develop/RUNBOOK.md#firmware-mit-bootcommander-aktualisieren)
mit den Werten aus der Tabelle oben. Da `‹LED›` derzeit nicht blinkt, gilt als Prüfung nur die
Meldung von MicroBoot bzw. `[OK]` in jeder Zeile von BootCommander.

## Gerät mit defekter Application wiederherstellen

Die Application startet nicht, hängt sofort oder das Update wurde abgebrochen. Der Bootloader öffnet
nach jedem Reset sein 500-ms-Fenster, auch wenn die Application defekt ist. Ein Taster ist nicht nötig.

### Voraussetzungen
- wie bei [Firmware mit MicroBoot aktualisieren](https://github.com/Diveturtle93/STM32F7_MicroBoot/blob/develop/RUNBOOK.md#firmware-mit-microboot-aktualisieren)
- `‹Firmware›` für genau dieses Steuergerät (BMS oder Motor)

### Schritte
1. MicroBoot wie in der Aktualisierung, Schritte 1–5, vorbereiten, bis MicroBoot
   „Connecting to target“ anzeigt.
2. Steuergerät neu starten: `‹Reset›`.
3. Warten, bis Löschen und Programmieren abgeschlossen sind. Steuergerät dabei nicht ausschalten.

### Prüfung
- MicroBoot meldet den Abschluss und schließt das Fenster.
- Die Application läuft.

### Fehlerbilder

| Symptom | Ursache | Abhilfe |
|---------|---------|---------|
| MicroBoot verbindet sich nicht | Steuergerät war beim Start von MicroBoot schon an, das Fenster ist vorbei | Schritt 2 wiederholen, solange MicroBoot „Connecting“ anzeigt |
| Update wird abgelehnt, bevor gelöscht wird | Firmware des anderen Steuergeräts gewählt, oder Info-Tabelle im Flash gelöscht | Richtige `‹Firmware›` wählen. Hilft das nicht, [Bootloader erstmalig programmieren](#bootloader-erstmalig-programmieren) |
| MicroBoot verbindet sich auch nach mehreren Resets nicht | Bootloader im Flash beschädigt oder überschrieben | [Bootloader erstmalig programmieren](#bootloader-erstmalig-programmieren) |
