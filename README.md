# Klipper — Artillery Sidewinder X1 + BLTouch (Waggster mod, Z max)

Configuration Klipper et stack Docker Compose (Unraid) pour une Sidewinder X1 :

- carte MKS Gen L (ATmega2560), drivers TMC2100 ;
- sonde 3D Touch / BLTouch en montage **Waggster mod** : signal sur **Z max** (D19 / `PD2`), commande sur la broche servo (D11 / `PB5`) ;
- écran TFT en mode Marlin (émulation ST7920), avec le brochage remappé par Digant ;
- Klipper + Moonraker + Mainsail + webcam en Docker, images de [prind](https://github.com/mkuf/prind).

Valeurs reprises de [rveniel/marlin-swx1](https://github.com/rveniel/marlin-swx1) (branche `aswx1_2.1.x_bltouch_waggster_mod_(z_max_used)`) et de l'EEPROM (M503).

## Contenu

| Fichier | Rôle |
|---|---|
| `docker-compose.yaml` | La stack : Klipper, Moonraker, Mainsail, webcam, Traefik |
| `.env.example` | Dossier, port, port série et webcam à adapter |
| `config/printer.cfg` | La machine : moteurs, chauffe, sonde, maillage, écran |
| `config/macros.cfg` | `PRINT_START`, `PRINT_END`, filament, pause/reprise/annulation |
| `config/moonraker.conf` | Moonraker (sans systemd, réseaux privés uniquement, webcam) |
| `firmware/build.config` | Compilation du firmware : atmega2560, 16 MHz, UART0, 250000 bauds |

Sur le serveur, **seuls `docker-compose.yaml` et `.env` sont nécessaires**.
Au premier démarrage, le conteneur `init` télécharge les fichiers de `config/` et
`firmware/` dans `STACK_DIR`. Il n'écrase jamais un fichier existant : ensuite,
les configs vivent dans appdata et se modifient depuis Mainsail
(onglet *Machine*).

## Installation sur Unraid (plugin Docker Compose Manager)

1. **Arrêter le conteneur OctoPrint** : il occupe le port série.
2. Repérer la carte et la webcam (terminal Unraid) :
   ```bash
   ls /dev/serial/by-id/
   ls /dev/v4l/by-id/
   ```
3. Créer une stack, coller `docker-compose.yaml`, puis renseigner le `.env`
   à partir de `.env.example` (`MCU_SERIAL`, `WEBCAM_DEVICE`, `HTTP_PORT`, `STACK_DIR`).
4. Flasher le firmware Klipper sur la carte (section suivante).
5. *Compose Up*, activer l'autostart, ouvrir `http://<ip-du-serveur>:8090`.

`MCU_SERIAL` n'est appliqué qu'à la création de `printer.cfg`. Pour le changer
ensuite, modifier la section `[mcu]` dans Mainsail.

## Firmware de la carte

Depuis le terminal Unraid, carte libre (stack arrêtée ou conteneur Klipper stoppé).
Lancer d'abord la stack une fois pour que `firmware/build.config` soit téléchargé.

```bash
docker run --rm -it --privileged -v /dev:/dev -v /mnt/user/appdata/klipper-swx1/firmware:/fw -w /opt/klipper --entrypoint bash mkuf/klipper:latest-tools -c "cp /fw/build.config .config && make olddefconfig && make && cp .config /fw/build.config && make flash FLASH_DEVICE=/dev/serial/by-id/<ta-carte>"
```

(adapter `/mnt/user/appdata/klipper-swx1` à ton `STACK_DIR`). `make olddefconfig`
met `build.config` à niveau quand Klipper ajoute de nouvelles options, puis la
version à jour est réenregistrée dans `firmware/`. Un flash réussi se termine par
`avrdude done. Thank you.`

Pour modifier la configuration de compilation, remplacer `make olddefconfig`
par `make menuconfig`.

Retour à Marlin à tout moment : reflasher le firmware Digant 2.1.2 (variante
`bltouch_waggster_mod_z_max_used`), le câblage est identique.

## Premier démarrage

Le guide détaillé, avec le résultat attendu et la correction pour chaque étape,
est dans **[docs/premier-demarrage.md](docs/premier-demarrage.md)**. En résumé :

1. Températures au repos.
2. Chauffe de la buse puis du plateau.
3. Fins de course X et Y (`QUERY_ENDSTOPS`).
4. Sens des moteurs (`STEPPER_BUZZ`).
5. Sonde (`QUERY_PROBE`, `BLTOUCH_DEBUG`).
6. Homing X, Y, puis Z avec test de sécurité.
7. `PROBE_CALIBRATE` et `PROBE_ACCURACY`.
8. `PID_CALIBRATE` buse et plateau.
9. `BED_MESH_CALIBRATE`.
10. `rotation_distance` de l'extrudeur.
11. Première impression.
12. Pressure advance et input shaping.

## OrcaSlicer

Profil d'imprimante :

- *G-code flavor* : **Klipper** ; *Machine limits usage* : **Use for time estimate**.
- G-code de démarrage :
  ```
  SET_PRINT_STATS_INFO TOTAL_LAYER_COUNT=[total_layer_count]
  PRINT_START BED_TEMP=[bed_temperature_initial_layer_single] EXTRUDER_TEMP=[nozzle_temperature_initial_layer]
  ```
- G-code de fin : `PRINT_END`
- G-code avant changement de couche : `SET_PRINT_STATS_INFO CURRENT_LAYER={layer_num + 1}`
- G-code après changement de couche : vide.
- G-code de pause : `PAUSE` (le `M0` de Marlin n'existe pas dans Klipper).
- Connexion : *Octo/Klipper*, `http://<ip-du-serveur>:8090`, sans clé API.

Profil d'impression, onglet *Others* : activer **Label objects** et **Exclude objects**
(maillage adaptatif et annulation d'un objet en cours d'impression).

## Sécurité

Moonraker accepte sans authentification tout client des réseaux privés.
Ne pas exposer le port de la stack sur Internet (ni via un reverse proxy public)
sans authentification devant.
