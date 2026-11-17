# Klipper — Artillery Sidewinder X1 + BLTouch (Waggster mod, Z max)

Configuration Klipper pour une Sidewinder X1 :

- carte MKS Gen L (ATmega2560), drivers TMC2100 ;
- sonde 3D Touch / BLTouch en montage **Waggster mod** : signal sur **Z max** (D19 / `PD2`), commande sur la broche servo (D11 / `PB5`) ;
- écran TFT en mode Marlin (émulation ST7920), avec le brochage remappé par Digant ;
- hôte : stack [prind](https://github.com/mkuf/prind) (Klipper + Moonraker + Mainsail en Docker).

Valeurs reprises de [rveniel/marlin-swx1](https://github.com/rveniel/marlin-swx1) (branche `aswx1_2.1.x_bltouch_waggster_mod_(z_max_used)`) et de l'EEPROM (M503).

## Fichiers

- `printer.cfg` : la machine (moteurs, chauffe, sonde, maillage, écran).
- `macros.cfg` : `PRINT_START`, `PRINT_END`, chargement/déchargement du filament, pause/reprise/annulation.

## Firmware MCU

`make menuconfig` : Atmega AVR → atmega2560, 16 MHz, communication série UART0, 250000 bauds.
Flasher en USB via le chemin `/dev/serial/by-id/...`.

## Premier démarrage

1. Renseigner le bon `serial:` dans `[mcu]` (`ls /dev/serial/by-id/`).
2. `QUERY_ENDSTOPS` : X et Y au repos puis appuyés.
3. `QUERY_PROBE`, puis `BLTOUCH_DEBUG COMMAND=pin_down` / `pin_up`.
4. Homing X, puis Y, puis Z, la main près de l'interrupteur.
5. `PROBE_CALIBRATE`, `PID_CALIBRATE HEATER=extruder TARGET=210`, `PID_CALIBRATE HEATER=heater_bed TARGET=60`, calibration de l'extrudeur.
6. Pressure advance, puis input shaping (ADXL345 USB, section commentée dans `printer.cfg`).

## OrcaSlicer

- G-code de démarrage : `PRINT_START BED_TEMP=[bed_temperature_initial_layer_single] EXTRUDER_TEMP=[nozzle_temperature_initial_layer]`
- G-code de fin : `PRINT_END`
- Activer « Label objects » pour le maillage adaptatif.

## Retour à Marlin

Reflasher le firmware Digant 2.1.2 (variante `bltouch_waggster_mod_z_max_used`) : le câblage est identique.
