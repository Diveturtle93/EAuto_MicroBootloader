# Changelog

Alle nennenswerten Änderungen an diesem Projekt werden hier dokumentiert.
Format nach [Keep a Changelog](https://keepachangelog.com/de/1.1.0/),
Versionierung nach [Semantic Versioning](https://semver.org/lang/de/).

## [Unveröffentlicht]

### Hinzugefügt
- Bootloader für den STM32F767ZIT auf BMS und Motorsteuergerät, der eine neue Application über CAN3 mit 500 kBit/s und XCP annimmt.
- Bibliothek STM32F7_MicroBoot als Submodul in `Lib/STM32F7_MicroBoot`.
- CubeMX-Projekt mit HSE 25 MHz, Systemtakt 216 MHz und CAN3 an PA8/PA15.
- Flash-Aufteilung: Bootloader in Sektor 0 (32 kB), Application ab `0x08008000`.
- Versionsnummer des Projekts in `version.h`.
- README mit Hardware, Bauen und Flashen, Einstellungen für die Application, Schnittstellen, Hinweisen und Abhängigkeiten.
- Architekturbeschreibung in `ARCHITECTURE.md` mit Komponenten, Startablauf, Speicher und Ressourcen.
- Runbook in `RUNBOOK.md` für Erstprogrammierung, Firmware-Update über CAN und Wiederherstellung.

[Unveröffentlicht]: https://github.com/Diveturtle93/EAuto_MicroBootloader/commits/develop
