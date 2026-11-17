# Premier démarrage — vérifications et calibration

À faire **dans l'ordre**, une fois que Klipper affiche « Ready » dans Mainsail,
et **avant toute impression**. Chaque étape protège la suivante : un fin de course
lu à l'envers ou un moteur qui tourne dans le mauvais sens fait taper un axe en butée
au premier homing.

## Avant de commencer

- **Arrêt d'urgence** : le bouton rouge (icône « power ») en haut de Mainsail, ou
  la commande `M112`. Il coupe tout immédiatement. Garde aussi l'interrupteur de
  l'imprimante à portée de main pour les tests de mouvement.
- **Console** : les commandes se tapent dans la console de Mainsail (tableau de bord).
- **Modifier la config** : onglet *Machine* → `printer.cfg` → modifier →
  *Save & Restart*. Klipper redémarre en quelques secondes.
- **`SAVE_CONFIG`** : enregistre les résultats de calibration en bas de
  `printer.cfg` (bloc `#*# <---------------------- SAVE_CONFIG`) et redémarre Klipper.
  Ce bloc a priorité sur les valeurs écrites plus haut dans le fichier.

---

## 1. Températures

**Faire** : regarder le panneau *Températures* de Mainsail, imprimante froide.

**Attendu** : buse et plateau affichent la température ambiante (15 à 25 °C) et
la valeur est stable.

**Sinon** :
- Valeur aberrante (-200 °C, 500 °C…) ou erreur `ADC out of range` : thermistance
  débranchée ou mauvaise broche. Vérifier les connecteurs.
- Valeur plausible mais fausse de beaucoup : mauvais `sensor_type`.

## 2. Chauffes

**Faire** :
```
SET_HEATER_TEMPERATURE HEATER=extruder TARGET=50
```
Attendre que la courbe monte, puis :
```
SET_HEATER_TEMPERATURE HEATER=extruder TARGET=0
SET_HEATER_TEMPERATURE HEATER=heater_bed TARGET=40
```
Puis remettre le plateau à 0 de la même façon.

**Attendu** : la température de l'élément demandé monte (et pas celle de l'autre).
Le ventilateur du hotend démarre au-dessus de 50 °C.

**Sinon** :
- C'est l'autre élément qui chauffe : broches `heater_pin` inversées entre
  `[extruder]` et `[heater_bed]`.
- Rien ne chauffe et erreur `Heater not heating at expected rate` : vérifier
  l'alimentation et le câblage. Ne pas insister.

## 3. Fins de course X et Y

**Faire** : axes loin des butées, taper
```
QUERY_ENDSTOPS
```
Puis **maintenir le fin de course X enfoncé** à la main et relancer la commande
sans le lâcher. Recommencer avec le fin de course Y.
(Équivalent : onglet *Machine* → panneau *Endstops* → rafraîchir.)

**Attendu** :

| Situation | Résultat |
|---|---|
| Au repos | `x:open y:open` |
| X enfoncé | `x:TRIGGERED y:open` |
| Y enfoncé | `x:open y:TRIGGERED` |

Z n'est pas concerné ici : c'est la sonde qui sert de fin de course (étape 6).

**Sinon** :
- Résultat inversé (`TRIGGERED` au repos, `open` appuyé) : dans `printer.cfg`,
  enlever le `!` de `endstop_pin` pour l'axe concerné (`^!PE5` → `^PE5` pour X,
  `^!PJ1` → `^PJ1` pour Y), *Save & Restart*, refaire le test.
- Rien ne change quand on appuie : câble débranché ou mauvais connecteur.

## 4. Sens des moteurs

**Faire** : placer chaque axe au milieu de sa course (moteurs non alimentés,
à la main), puis :
```
STEPPER_BUZZ STEPPER=stepper_x
```
Le moteur fait 10 allers-retours de 1 mm, en partant dans le **sens positif**.
Observer le **premier** mouvement, puis faire de même avec `stepper_y` et `stepper_z`.

**Attendu** (premier mouvement) :

| Moteur | Sens positif attendu |
|---|---|
| `stepper_x` | la tête s'éloigne du fin de course X (vers la droite) |
| `stepper_y` | le plateau s'éloigne du fin de course Y |
| `stepper_z` | le portique **monte** (la buse s'éloigne du plateau) |

**Sinon** :
- Mauvais sens : ajouter ou enlever le `!` devant `dir_pin` de l'axe concerné
  (`PF1` ↔ `!PF1`, `PF7` ↔ `!PF7`, `!PL1` ↔ `PL1`), *Save & Restart*, refaire.
- Le moteur ne bouge pas ou vibre sans avancer : vérifier les broches `step_pin` /
  `enable_pin` et le câble.
- Z : les deux moteurs doivent bouger ensemble et dans le même sens.

## 5. Sonde 3D Touch

**Faire**, en vérifiant à chaque fois le résultat :
```
QUERY_PROBE
BLTOUCH_DEBUG COMMAND=pin_down
BLTOUCH_DEBUG COMMAND=pin_up
BLTOUCH_DEBUG COMMAND=pin_down
BLTOUCH_DEBUG COMMAND=touch_mode
QUERY_PROBE
```
Puis **pousser doucement la tige vers le haut** avec l'ongle et relancer
`QUERY_PROBE` en la maintenant. Terminer par :
```
BLTOUCH_DEBUG COMMAND=pin_up
QUERY_PROBE
```

**Attendu** :

| Commande | Résultat |
|---|---|
| premier `QUERY_PROBE` (tige rentrée) | `probe: open` |
| `pin_down` / `pin_up` | la tige descend / remonte |
| `QUERY_PROBE` en `touch_mode`, tige libre | `probe: open` |
| `QUERY_PROBE` en `touch_mode`, tige poussée | `probe: TRIGGERED` |
| `QUERY_PROBE` final (tige rentrée) | `probe: open` |

**Sinon** :
- La tige ne bouge pas : broche de commande (`control_pin`) ou câble du servo.
- La LED de la sonde clignote (alarme) : `BLTOUCH_DEBUG COMMAND=reset`.
- `TRIGGERED` alors que la tige est libre, ou jamais `TRIGGERED` : broche de signal
  (`sensor_pin`) ou câble Z max.
- Erreur `BLTouch failed to verify sensor state` plus tard : décommenter
  `pin_up_touch_mode_reports_triggered: False` dans `[bltouch]`.

## 6. Homing

Garder la main sur l'arrêt d'urgence pendant toute cette étape.

### X puis Y

**Faire** : placer la tête et le plateau au milieu, puis
```
G28 X
```
puis
```
G28 Y
```

**Attendu** : l'axe part vers son fin de course, le touche, recule un peu,
revient plus lentement, s'arrête.

**Sinon** : s'il part dans le mauvais sens ou ne s'arrête pas → **arrêt d'urgence**,
puis revoir les étapes 3 et 4.

### Z (avec test de sécurité)

**Faire** : vérifier qu'il y a au moins 2 cm entre la buse et le plateau, puis
```
G28
```
Pendant la descente de Z, **toucher la tige de la sonde avec le doigt**.

**Attendu** :
1. X et Y font leur homing.
2. Z monte de 10 mm, la tête se place pour que la sonde soit au centre du plateau.
3. La tige descend, Z descend.
4. **Dès que tu touches la tige, Z s'arrête** (et Klipper peut terminer le homing
   ou signaler une erreur : c'est normal pour ce test).

Après ce test, Klipper croit que Z=0 est en l'air : ne faire **aucun mouvement**
avant le `G28` suivant.

Relancer ensuite `G28` sans toucher : Z doit s'arrêter quand la sonde touche le plateau,
remonter, toucher une seconde fois, puis la tige remonte.

**Sinon** : si Z ne s'arrête pas quand tu touches la tige → **arrêt d'urgence**
et revoir l'étape 5. Ne pas aller plus loin tant que ce test échoue.

## 7. Hauteur de la buse (z_offset)

**Faire** : plateau à température d'impression (ex. 60 °C), buse propre, puis
```
G28
PROBE_CALIBRATE
```
Glisser une feuille de papier sous la buse et ajuster la hauteur avec les boutons
de l'interface ou avec `TESTZ Z=-0.1` (descendre), `TESTZ Z=+0.05` (monter),
jusqu'à sentir un léger frottement sur la feuille. Puis :
```
ACCEPT
SAVE_CONFIG
```

**Attendu** : une valeur autour de **1,7** (ton Marlin avait -1,74 : le signe est
inversé dans Klipper).

**Sinon** : une valeur très différente (0,2 ou 4) indique un souci de montage de la
sonde ou un mauvais `z_offset` de départ. Refaire la mesure.

Vérifier ensuite la répétabilité de la sonde :
```
PROBE_ACCURACY
```
**Attendu** : `standard deviation` sous 0,01 mm (très bien), jusqu'à 0,025 mm
(acceptable pour un clone). Au-delà : vis du support, câble qui tire, tige sale.

## 8. PID

**Faire**, l'un après l'autre (plusieurs minutes chacun, ne pas toucher) :
```
PID_CALIBRATE HEATER=extruder TARGET=210
SAVE_CONFIG
PID_CALIBRATE HEATER=heater_bed TARGET=60
SAVE_CONFIG
```

**Attendu** : la température oscille autour de la cible quelques fois, puis la
console affiche `pid_Kp=… pid_Ki=… pid_Kd=…`. `SAVE_CONFIG` les enregistre.

## 9. Maillage du plateau

**Faire** : plateau à 60 °C, puis
```
G28
BED_MESH_CALIBRATE
```

**Attendu** : la sonde fait 7 × 7 points. Dans Mainsail, onglet *Heightmap*,
le maillage s'affiche. Un écart total (« range ») sous 0,3 mm est bon pour une SWX1.

Pas besoin de le sauvegarder : `PRINT_START` refait un maillage adapté à chaque impression.

## 10. Extrudeur (rotation_distance)

**Faire** :
1. Chauffer la buse à 200 °C (PLA).
2. Faire une marque sur le filament à **120 mm** au-dessus de l'entrée de l'extrudeur.
3. Extruder 100 mm lentement :
   ```
   M83
   G1 E100 F60
   ```
4. Mesurer la distance restante entre l'entrée de l'extrudeur et la marque.

**Calcul** :
```
extrudé réel   = 120 - distance restante
nouvelle valeur = 7.191 × extrudé réel / 100
```
Exemple : il reste 22 mm → 98 mm extrudés → `7.191 × 98 / 100 = 7.047`.

Mettre la nouvelle valeur dans `rotation_distance` de `[extruder]`, *Save & Restart*,
refaire le test jusqu'à être à ±1 mm.

**Attendu** : le filament entre dans l'extrudeur (s'il recule, ajouter/enlever le `!`
de `dir_pin` dans `[extruder]`).

## 11. Premier test d'impression

Imprimer une pièce simple (cube de calibration) avec OrcaSlicer, en profil Klipper,
en surveillant la première couche. Ajuster finement la hauteur pendant l'impression
avec le *Z offset* de Mainsail (babystep), puis l'enregistrer avec
`Z_OFFSET_APPLY_PROBE` et `SAVE_CONFIG`.

## 12. Ensuite : pressure advance et input shaping

- **Pressure advance** : tour de calibration (outil intégré d'OrcaSlicer :
  *Calibration* → *Pressure advance*), puis reporter la valeur dans `pressure_advance`.
- **Input shaping** : avec l'ADXL345 USB (section commentée en bas de `printer.cfg`),
  `SHAPER_CALIBRATE` puis `SAVE_CONFIG`. Pour analyser les CSV de
  `TEST_RESONANCES` (dans `STACK_DIR/resonances`) :
  ```bash
  docker run --rm -v /mnt/user/appdata/klipper-swx1/resonances:/opt/klipper/resonances \
    -w /opt/klipper --entrypoint bash mkuf/klipper:latest-tools \
    -c "scripts/calibrate_shaper.py resonances/resonances_x_*.csv -o resonances/shaper_x.png"
  ```
- Une fois l'input shaping réglé, remonter progressivement `max_accel` dans `[printer]`.
