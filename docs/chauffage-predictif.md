# Chauffage anticipé VT (inertie + météo + calendrier + absence)

Ce document décrit le blueprint Home Assistant `Chauffage anticipé VT (inertie + météo + calendrier + absence)` situé dans `blueprints/chauffage-predictif.yaml`.

## Objectif

Ce blueprint pilote un thermostat (typiquement _Versatile Thermostat_) en fixant une consigne via `climate.set_temperature` en fonction :

- d’un **calendrier de présence** (présence active/inactive),
- d’un **calendrier d’absence** (vacances/déplacement) servant de **verrou**,
- d’un **taux de chauffe** (slope) pour anticiper le préchauffage,
- d’une **pondération météo** (température extérieure),
- d’un **facteur d’apprentissage** pour ajuster le taux de chauffe dans le temps.

## Prérequis

- Un thermostat `climate` supportant le service `climate.set_temperature`.
- Un calendrier `calendar` représentant la **présence**.
- Un calendrier `calendar` représentant l’**absence**.
- Un capteur `sensor` de température intérieure.
- Un capteur `sensor` fournissant un **taux de chauffe** (slope) en **°C/min** ou **°C/h**.
- Un `input_number` servant de facteur d’apprentissage.
- Un `input_text` servant à stocker le timestamp de préchauffe calculé.

## Paramètres (inputs)

- **Thermostat Versatile** (`thermostat`)

  - Entité `climate` ciblée.

- **Capteur de température intérieure** (`temperature_sensor`)

  - Entité `sensor` (device_class `temperature`).

- **Slope VT** (`inertia_sensor`)

  - Entité `sensor` contenant un taux (ex: `0.01`).
  - Les valeurs avec virgule sont acceptées (ex: `0,01`).

- **Unité du slope** (`inertia_unit`)

  - `c_per_min` (°C/min) ou `c_per_hour` (°C/h).

- **Température extérieure** (`outside_temperature`)

  - Entité `sensor` (device_class `temperature`).
  - Utilisée pour pondérer le taux de chauffe:
    - si `ext < 0` alors `rate * 0.8`
    - si `ext > 10` alors `rate * 1.15`

- **Calendrier de présence** (`calendar_entity`)

  - Entité `calendar`.

- **Calendrier d'absence** (`absence_calendar`)

  - Entité `calendar`.

- **Température preset comfort (number)** (`comfort_temperature_number`)

  - Entité `number` (ex: `number.central_configuration_preset_comfort_temp`).
  - Sert de consigne `target_temp`.

- **Heure minimale de démarrage** (`earliest_start`)

  - Heure après laquelle le préchauffage est autorisé (défaut `05:00:00`).

- **Inertie maximale (minutes)** (`max_inertia`)

  - Plafond (défaut `180`).
  - L’inertie réelle utilisée = `min(inertie_apprise, inertie_max)`.

- **Facteur d’apprentissage** (`learning_factor`)

  - Entité `input_number`.
  - Multiplie le taux de chauffe estimé (`rate`).

- **Stockage timestamp préchauffe** (`preheat_start_time_text`)

  - Entité `input_text`.
  - Stocke le timestamp Unix calculé du début de préchauffe (`preheat_start_ts`) pour éviter les doublons et servir à l’apprentissage.

## Fonctionnement

### Déclencheurs (triggers)

- **`preheat`** (_template trigger_):

  - Récupère `start_time` du calendrier de présence.
  - Calcule un timestamp de début de préchauffe `preheat_start_ts` (en secondes Unix) avec:
    - `target_temp` depuis `comfort_temperature_number`
    - `delta = target_temp - temp_int`
    - `rate` depuis `inertia_sensor` (converti selon `inertia_unit`)
    - pondération météo via `outside_temperature`
    - multiplication par `learning_factor`
    - plafonnement via `max_inertia`
    - contrainte de démarrage minimum via `earliest_start`
  - Déclenche dans une fenêtre de 60s: `preheat_start_ts <= now() < preheat_start_ts + 60`.

- **`preheat_catchup`** (_time_pattern_):

  - Se déclenche toutes les minutes.
  - Sert de rattrapage si Home Assistant a raté la fenêtre de 60s.

- **`learning`** (_numeric_state_):
  - Se déclenche lorsque la température intérieure dépasse la consigne comfort (`temperature_sensor` au-dessus de `comfort_temperature_number`).

### Condition globale

- **Blocage absence**: l’automatisation ne s’exécute pas si `absence_calendar` est `on`.

### Actions

Le blueprint utilise un `choose` selon l’identifiant du trigger.

- **Si `preheat`** (pré-chauffage anticipé):

  - Conditions:
    - le calendrier de présence doit être `on`.
  - Actions:
    - stocke `preheat_start_ts` dans `preheat_start_time_text`.
    - applique `climate.set_temperature` avec `temperature = target_temp`.

- **Si `preheat_catchup`** (rattrapage):

  - Conditions:
    - le calendrier de présence doit être `on`.
    - `now()` est après `preheat_start_ts + 60`.
    - `now()` est avant le début de l’évènement calendrier.
    - anti-doublon via `preheat_start_time_text`.
  - Actions identiques à `preheat` (stockage + `climate.set_temperature`).

- **Si `learning`** (apprentissage):
  - Compare l’heure réelle (quand on dépasse la consigne) à l’heure planifiée stockée.
  - Ajuste `learning_factor` légèrement (borné) pour améliorer le calcul des prochaines préchauffes.
  - Garde-fous: si le timestamp planifié est absent/incohérent/trop ancien, le facteur n’est pas modifié.

## Exemple de configuration

- **Calendrier de présence**: un événement « Présence » démarre à `07:30`.
- **Slope**: `0.01` °C/min.
- **Inertie max**: `180`.

Le blueprint calcule `preheat_start_ts` (par ex. vers `06:00` si l’anticipation nécessaire est 90 minutes), puis déclenche à ce moment-là pour fixer la consigne comfort sur le thermostat.

## Points d’attention / dépannage

- **Calendrier d’absence (condition)**

  - Le blueprint ne s’exécute pas si `absence_calendar` est `on`.

- **Température extérieure**

  - `outside_temperature` influence le taux de chauffe (pondération) et donc l’heure de préchauffe.

- **Attribut `start_time`**

  - Le trigger `preheat` dépend de `state_attr(calendar_entity, 'start_time')`.
  - Si votre intégration calendrier ne fournit pas `start_time`, le préchauffage ne se déclenchera pas.

- **Inertie apprise**

  - Si le capteur n’est pas numérique (ou si le taux est `<= 0`), le blueprint bascule sur un déclenchement à l’heure du calendrier (`start_time`) au lieu d’anticiper.

- **Fenêtre de déclenchement et rattrapage**

  - Le trigger `preheat` a une fenêtre de 60 secondes.
  - Si Home Assistant rate cette fenêtre (redémarrage/charge), le rattrapage `preheat_catchup` tentera de démarrer la préchauffe jusqu’au début de l’évènement.
