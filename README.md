# Klipper — Artillery Sidewinder X1 + BLTouch (Waggster mod, Z max)

Configuration Klipper et stack Docker (Unraid) pour une Sidewinder X1 :

- carte MKS Gen L (ATmega2560), drivers TMC2100 ;
- sonde 3D Touch / BLTouch en montage **Waggster mod** : signal sur **Z max** (D19 / `PD2`), commande sur la broche servo (D11 / `PB5`) ;
- écran TFT en mode Marlin (émulation ST7920), avec le brochage remappé par Digant ;
- Klipper + Moonraker + Mainsail + webcam en Docker, images de [prind](https://github.com/mkuf/prind).

Valeurs reprises de [rveniel/marlin-swx1](https://github.com/rveniel/marlin-swx1) (branche `aswx1_2.1.x_bltouch_waggster_mod_(z_max_used)`) et de l'EEPROM (M503).

## Contenu

| Fichier | Rôle |
|---|---|
| `docker-compose.yaml` | La stack : Klipper, Moonraker, Mainsail, webcam, Traefik, outils firmware |
| `.env.example` | Chemins et port à adapter (à copier en `.env`) |
| `config/printer.cfg` | La machine : moteurs, chauffe, sonde, maillage, écran |
| `config/macros.cfg` | `PRINT_START`, `PRINT_END`, filament, pause/reprise/annulation |
| `config/moonraker.conf` | Moonraker (sans systemd, réseaux privés uniquement, webcam) |
| `firmware/build.config` | Configuration de compilation : atmega2560, 16 MHz, UART0, 250000 bauds |

## Installation sur Unraid

1. Cloner le repo dans appdata :
   ```bash
   cd /mnt/user/appdata
   git clone https://github.com/rveniel/klipper-swx1-bltouch-z-max
   cd klipper-swx1-bltouch-z-max
   cp .env.example .env
   ```
2. Adapter `.env` : `STACK_DIR` (chemin absolu du clone), `HTTP_PORT`, `WEBCAM_DEVICE`.
3. Renseigner le port série de la carte dans `config/printer.cfg`, section `[mcu]` :
   ```bash
   ls /dev/serial/by-id/
   ```
4. **Arrêter le conteneur OctoPrint** : il occupe le port série.
5. Flasher le firmware Klipper sur la carte (section suivante).
6. Lancer la stack :
   ```bash
   docker compose up -d
   ```
   Avec le plugin *Docker Compose Manager*, créer une stack pointant sur ce dossier
   (ou coller le compose et le `.env`), et activer l'autostart.
7. Ouvrir `http://<ip-du-serveur>:8090`.

## Firmware de la carte

La carte doit être libre : arrêter Klipper avant de flasher.

```bash
docker compose stop klipper
docker compose run --rm tools "make"
docker compose run --rm tools "make flash FLASH_DEVICE=/dev/serial/by-id/<ta-carte>"
docker compose start klipper
```

`firmware/build.config` est déjà réglé pour la carte. Pour le modifier :
`docker compose run --rm tools "make menuconfig"`.

Retour à Marlin à tout moment : reflasher le firmware Digant 2.1.2 (variante
`bltouch_waggster_mod_z_max_used`), le câblage est identique.

## Premier démarrage

1. `QUERY_ENDSTOPS` : X et Y au repos puis appuyés.
2. `QUERY_PROBE`, puis `BLTOUCH_DEBUG COMMAND=pin_down` / `pin_up`.
3. Homing X, puis Y, puis Z, la main près de l'interrupteur.
4. `PROBE_CALIBRATE`, puis `SAVE_CONFIG`.
5. `PID_CALIBRATE HEATER=extruder TARGET=210` et `PID_CALIBRATE HEATER=heater_bed TARGET=60`, puis `SAVE_CONFIG`.
6. Calibration de l'extrudeur, pressure advance, puis input shaping
   (ADXL345 USB, section commentée dans `printer.cfg`).

Les CSV de résonance s'analysent avec le conteneur d'outils :
```bash
docker compose cp klipper:/tmp/resonances_x_<date>.csv ./resonances/
docker compose run --rm tools "scripts/calibrate_shaper.py resonances/resonances_x_<date>.csv -o resonances/shaper_x.png"
```

## OrcaSlicer

- Hôte d'impression : type **Klipper**, adresse `http://<ip-du-serveur>:8090`.
- G-code de démarrage : `PRINT_START BED_TEMP=[bed_temperature_initial_layer_single] EXTRUDER_TEMP=[nozzle_temperature_initial_layer]`
- G-code de fin : `PRINT_END`
- Activer « Label objects » pour le maillage adaptatif.

## Sécurité

Moonraker accepte sans authentification tout client des réseaux privés.
Ne pas exposer le port de la stack sur Internet (ni via un reverse proxy public)
sans authentification devant.
