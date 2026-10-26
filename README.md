# EAuto MicroBootloader

Bootloader für das Batteriemanagement-System (EAuto_BMS) und das Motorsteuergerät (EAuto_Motor).
Er basiert auf [OpenBLT](https://www.feaser.com/openblt/) und nimmt eine neue Application über den
Antriebs-CAN mit dem XCP-Protokoll an, gesteuert vom PC-Tool MicroBoot oder BootCommander von
Feaser. So lassen sich die Steuergeräte im Fahrzeug programmieren, ohne einen Debugger anzuschließen.

## Beschreibung

Das Projekt ist ein STM32CubeIDE-Projekt für den STM32F767ZIT. Den Bootloader selbst liefert die
Bibliothek [STM32F7_MicroBoot](https://github.com/Diveturtle93/STM32F7_MicroBoot) als Submodul in
`Lib/STM32F7_MicroBoot`. Dieses Projekt steuert die CubeMX-Initialisierung bei (Takt, GPIO, Pins und
Takt von CAN3) und ruft den Bootloader aus `main()` auf. Wie die Teile zusammenspielen, steht in der
[Architektur](ARCHITECTURE.md).

Der Flash ist in zwei Bereiche aufgeteilt:

| Flash | Startadresse | Endadresse | Speicher | Sektoren |
|:----- |:------------ |:---------- |:-------- |:-------- |
| Bootloader | `0x08000000` | `0x08007FFF` | 32 kB | 0 |
| Application | `0x08008000` | `0x081FFFFF` | 2016 kB | 1–11 |

Nach jedem Reset bleibt der Bootloader 500 ms aktiv. Verbindet sich in dieser Zeit MicroBoot über
CAN, bleibt er im Bootloader und nimmt die neue Firmware an. Sonst springt er in die Application ab
`0x08008000`. Vor dem Löschen vergleicht er die Info-Tabelle der neuen Firmware mit der im Gerät und
lässt das Update nur bei gleicher `tableId` und `productId` zu. So landet keine Motor-Firmware im BMS
und umgekehrt.

## Hardware

| Eigenschaft | Wert |
|:----------- |:---- |
| Mikrocontroller | STM32F767ZIT6, LQFP144, 2 MB Flash, 512 kB RAM |
| Takt | HSE 25 MHz, SYSCLK 216 MHz, APB1 54 MHz, APB2 108 MHz |
| Platinen | Motorsteuergerät Rev 1.0, Batteriemanagement-System Rev 1.0 |

Beide Platinen haben dieselbe Pinbelegung für den Bootloader:

| Funktion | Pin | Konfiguration in diesem Projekt | Verwendung |
|:-------- |:--- |:------------------------------- |:---------- |
| CAN3 RX | PA8 | Alternate Function 11 | Antriebs-CAN |
| CAN3 TX | PA15 | Alternate Function 11 | Antriebs-CAN |
| LED blau | PB7 | Analog (freier Pin) | Blinkt im Bootloader, siehe [Hinweise](#hinweise) |
| POWER_ON | PC13 | Analog (freier Pin) | Die Bibliothek liest hier einen Taster, siehe [Hinweise](#hinweise) |
| HSE | PH0, PH1 | Quarz 25 MHz | Systemtakt |

Alle übrigen Pins setzt CubeMX auf Analog (`FreePins`).

## Bauen und Flashen

Das Projekt wird mit STM32CubeIDE gebaut, die HAL stammt aus dem Paket STM32Cube FW_F7 V1.17.4
(CubeMX 6.18.1). Die Bibliothek liegt als Submodul in `Lib/` und wird beim Klonen mitgeholt:

```sh
git clone --recursive https://github.com/Diveturtle93/EAuto_MicroBootloader.git
git submodule update --init          # in einem bereits geklonten Repository
```

### Bootloader bauen

| Build-Konfiguration | Optimierung | Symbole | Ergebnis |
|:------------------- |:----------- |:------- |:-------- |
| Debug | keine | `DEBUG`, `USE_HAL_DRIVER`, `USE_FULL_LL_DRIVER`, `STM32F767xx` | `Debug/EAuto_MicroBootloader.elf` |
| Release | `-Os` | `USE_HAL_DRIVER`, `USE_FULL_LL_DRIVER`, `STM32F767xx` | `Release/EAuto_MicroBootloader.elf` |

`Lib/STM32F7_MicroBoot` ist als Include- und Quellpfad eingetragen. Der Bootloader muss in Sektor 0
(32 kB) passen. Das Linker-Skript prüft das derzeit nicht, siehe [Hinweise](#hinweise).

Wird der Code mit CubeMX neu erzeugt, müssen `AppInit()` und `AppTask()` in den `USER CODE`-Blöcken
von `main.c` stehen bleiben. `MX_CAN3_Init()` wird bewusst nicht aufgerufen, CAN3 initialisiert die
Bibliothek selbst. `HAL_CAN_MspInit()` in `Core/Src/can.c` wird aber gebraucht, weil es Pins und Takt
von CAN3 einschaltet.

### Application für den Bootloader bauen

Eine Application (EAuto_BMS, EAuto_Motor), die über den Bootloader aufgespielt wird, braucht:

1. **Linker-Skript:** `FLASH (rx) : ORIGIN = 0x8008000, LENGTH = 2016K`.
2. **Vektortabelle:** In `system_stm32f7xx.c` `USER_VECT_TAB_ADDRESS` aktivieren und
   `VECT_TAB_OFFSET` auf `0x00008000U` setzen.
3. **Platz für die Checksumme:** Am Ende von `g_pfnVectors` in der Startup-Datei ein zusätzliches
   Wort eintragen (`.word 0x55AA11EE`), damit an Offset `0x1F8` kein Code liegt.
4. **Info-Tabelle** mit 12 Byte (`tableId`, `productId`, `firmwareVersion`) an `0x08008200`, direkt
   hinter dem Boot-Block. BMS und Motor brauchen unterschiedliche `productId`s.
5. **Ausgabedatei:** Unter *MCU Post build outputs* „Convert to Motorola S-record file (-O srec)“
   aktivieren. Übertragen wird die `.srec`.

Codebeispiele für Info-Tabelle und Linker-Skript stehen in der README der Bibliothek unter
[Application vorbereiten](https://github.com/Diveturtle93/STM32F7_MicroBoot/blob/develop/README.md#4-application-vorbereiten).

### Flashen

Der Bootloader wird einmalig mit einem ST-Link über SWD aufgespielt, zusammen mit der ersten
Application. Danach werden Applications über CAN übertragen. Die Handgriffe am Steuergerät stehen im
[Runbook](RUNBOOK.md).

## Schnittstellen

| Schnittstelle | Einstellung | Verwendung |
|:------------- |:----------- |:---------- |
| CAN3 (Antriebs-CAN) | 500 kBit/s, Standard-IDs (11 Bit), Host → Gerät `0x7E0`, Gerät → Host `0x7E1` | XCP: Verbinden, Löschen, Programmieren, Start der Application |
| SWD | ST-Link | Aufspielen von Bootloader und erster Application, Debuggen |

Die Einstellungen stehen in `Lib/STM32F7_MicroBoot/blt_conf.h`. Die IDs `0x7E0` und `0x7E1` dürfen auf
dem Antriebs-CAN von keinem anderen Teilnehmer genutzt werden, siehe
[CAN_Matrix](https://github.com/Diveturtle93/CAN_Matrix).

## Hinweise

- **LED und POWER_ON:** Die Bibliothek lässt die LED an `PB7` blinken und liest an `PC13` einen
  Taster, der den Bootloader aktiv hält. In diesem Projekt sind beide Pins als freie Pins auf Analog
  gesetzt. Die LED blinkt deshalb nicht, und die Taster-Abfrage liefert immer „nicht gedrückt“. Auf
  beiden Platinen ist `PC13` zudem kein Taster, sondern der Ausgang `POWER_ON`. Wird `PC13` als
  Ausgang auf High gesetzt, liest die Abfrage „gedrückt“, und der Bootloader startet die Application
  nie. `PB7` in CubeMX als GPIO Output einstellen. Für `PC13` muss die Abfrage in `hooks.c` der
  Bibliothek angepasst werden, und es ist zu klären, welchen Pegel `POWER_ON` im Bootloader braucht.
- **Größe des Bootloaders:** `STM32F767ZITX_FLASH.ld` gibt `FLASH` mit `LENGTH = 2048K` an. Wird der
  Bootloader größer als 32 kB, meldet der Linker keinen Fehler, und der Bootloader ragt in den
  Bereich der Application. `LENGTH = 32K` eintragen.
- **Shared Parameters:** Die Bibliothek legt `sharedParamsBuffer` in die Sektion `.shared`. Das
  Linker-Skript kennt diese Sektion nicht, der Linker legt sie selbst ab. Für den Austausch mit der
  Application muss `.shared` in beiden Linker-Skripten an derselben Adresse als `NOLOAD` stehen.
- **Leeres Gerät:** Die Checksum-Hooks der Bibliothek prüfen nichts, und die Info-Tabelle wird auch
  bei leerem Flash verglichen. Ein Gerät ohne Application nimmt deshalb keine Firmware über CAN an.
  Die erste Application wird per ST-Link aufgespielt.
- **Submodul:** Eingebunden ist der Stand `f648bce` von `main`. In `.gitmodules` fehlt
  `branch = main`.
- Fehlen beim Bauen Header aus `Lib/`, ist das Submodul nicht ausgecheckt.

## Quellen

- [OpenBLT-Handbuch](https://www.feaser.com/openblt/doku.php?id=manual:start)
- [MicroBoot](https://www.feaser.com/openblt/doku.php?id=manual:microboot),
  [BootCommander](https://www.feaser.com/openblt/doku.php?id=manual:bootcommander)
- [STM32F767ZI – Datenblatt und Referenzhandbuch](https://www.st.com/en/microcontrollers-microprocessors/stm32f767zi.html)

## Abhängigkeiten

| Bibliothek | Tag | Zweck |
|---|---|---|
| [STM32F7_MicroBoot](https://github.com/Diveturtle93/STM32F7_MicroBoot) | – | OpenBLT-Bootloader mit CAN, XCP, Flash-Treiber und Hook-Funktionen |

Die Bibliothek hat noch keinen Tag. Eingebunden ist der Stand des Submoduls, Links auf ihre
Dokumentation zeigen bis zum ersten Tag auf `develop`.

Außerdem: STM32F7 HAL, LL-Treiber und CMSIS von ST (`Drivers/`).

## Lizenz

Dieses Projekt steht unter der [GPL-3.0 Lizenz](LICENSE).
