# Chauffage anticipé VT (inertie + météo + calendrier + absence)

Ce document décrit le blueprint Home Assistant `Chauffage anticipé VT (inertie + météo + calendrier + absence)` situé dans `blueprints/chauffage-predictif.yaml`.

## Objectif

Ce blueprint pilote un thermostat (typiquement _Versatile Thermostat_) via des `preset_mode` en fonction :

- d’un **calendrier de présence** (présence active/inactive),
- d’un **calendrier d’absence** (vacances/déplacement) servant de **verrou**,
- d’une **inertie thermique apprise** (en minutes) pour anticiper le préchauffage.

## Prérequis

- Un thermostat `climate` supportant le service `climate.set_preset_mode`.
- Un calendrier `calendar` représentant la **présence**.
- Un calendrier `calendar` représentant l’**absence**.
- Un capteur `sensor` de température intérieure.
- Un capteur `sensor` fournissant l’**inertie apprise** en **°C/min** (taux de chauffe moyen).

## Paramètres (inputs)

- **Thermostat Versatile** (`thermostat`)

  - Entité `climate` ciblée.

- **Capteur de température intérieure** (`temperature_sensor`)

  - Entité `sensor` (device_class `temperature`).

- **Inertie apprise (°C/min)** (`inertia_sensor`)

  - Entité `sensor` contenant un taux (ex: `0.01`).
  - Les valeurs avec virgule sont acceptées (ex: `0,01`).

- **Température extérieure** (`outside_temperature`)

  - Entité `sensor` (device_class `temperature`).
  - Remarque: ce paramètre est présent dans le blueprint mais **n’est pas utilisé** dans la logique actuelle.

- **Calendrier de présence** (`calendar_entity`)

  - Entité `calendar`.

- **État requis du calendrier de présence** (`calendar_entity_state`)

  - Valeur `on`/`off`.
  - Remarque: ce paramètre est présent dans le blueprint mais **n’est pas utilisé** dans la logique actuelle.

- **Calendrier d'absence** (`absence_calendar`)

  - Entité `calendar`.

- **État bloquant du calendrier d'absence** (`absence_calendar_state`)

  - Valeur `on`/`off`.
  - L’automatisation est exécutée uniquement si le calendrier d’absence est dans cet état.

- **Preset si présence active** (`presence_preset_on`)

  - Exemple: `comfort`, `eco`, `boost`.

- **Preset si présence inactive** (`presence_preset_off`)

  - Exemple: `comfort`, `eco`, `boost`.

- **Température preset comfort (number)** (`comfort_temperature_number`)

  - Entité `number` (ex: `number.central_configuration_preset_comfort_temp`).
  - Sert à déterminer la consigne lorsque `presence_preset_on = comfort`.

- **Température preset eco (number)** (`eco_temperature_number`)

  - Entité `number` exposée par Versatile Thermostat.
  - Sert à déterminer la consigne lorsque `presence_preset_on = eco`.

- **Température preset boost (number)** (`boost_temperature_number`)

  - Entité `number` exposée par Versatile Thermostat.
  - Sert à déterminer la consigne lorsque `presence_preset_on = boost`.

- **Heure minimale de démarrage** (`earliest_start`)

  - Heure après laquelle le préchauffage est autorisé (défaut `05:00:00`).

- **Inertie maximale (minutes)** (`max_inertia`)
  - Plafond (défaut `180`).
  - L’inertie réelle utilisée = `min(inertie_apprise, inertie_max)`.

## Fonctionnement

### Déclencheurs (triggers)

- **`presence_on`**: quand le calendrier de présence passe à `on`.
- **`presence_off`**: quand le calendrier de présence passe à `off`.
- **`preheat`** (_template trigger_):
  - Récupère `start_time` du calendrier de présence.
  - Calcule une durée d’anticipation en minutes à partir du taux de chauffe:
    - `rate = sensor(inertia_sensor)` en °C/min
    - `consigne` est récupérée depuis les entités `number` des presets (comfort/eco/boost) en fonction de `presence_preset_on`
    - `delta = consigne - temp_int`
    - `minutes_needed = ceil(delta / rate)`
    - `minutes = min(minutes_needed, max_inertia)`
  - Déclenche lorsque `now() >= start_time - minutes`.
  - Si le calcul n’est pas possible (ex: `rate <= 0` ou `delta <= 0`), déclenche lorsque `now() >= start_time`.

### Condition globale

- **Blocage absence**: le calendrier d’absence doit être dans l’état `absence_calendar_state`.

Attention: dans la version actuelle, cette condition n’est pas un « blocage si absence active » au sens générique, mais une condition stricte: _l’état doit correspondre exactement à la valeur configurée_. Si vous laissez `absence_calendar_state = on`, l’automatisation ne s’exécutera que lorsque le calendrier d’absence est `on`.

### Actions

Le blueprint utilise un `choose` selon l’identifiant du trigger.

- **Si `presence_off`**:

  - Applique `climate.set_preset_mode` avec `presence_preset_off`.

- **Si `presence_on`**:

  - Applique `climate.set_preset_mode` avec `presence_preset_on`.

- **Si `preheat`** (pré-chauffage anticipé):
  - Conditions supplémentaires:
    - le calendrier de présence doit être `on`,
    - l’heure actuelle doit être après `earliest_start`,
    - la température intérieure doit être inférieure à `consigne - 0.2`.
      - `consigne` est lu depuis l’attribut `temperature` du thermostat.
  - Action:
    - applique `climate.set_preset_mode` avec `presence_preset_on`.

## Exemple de configuration

- **Calendrier de présence**: un événement « Présence » démarre à `07:30`.
- **Inertie apprise**: `0.01` °C/min.
- **Inertie max**: `180`.

Le trigger `preheat` devient vrai à `06:00` (`07:30 - minutes`). Si après `earliest_start` et si la température intérieure est < consigne - 0.2, le blueprint passe le thermostat sur le preset de présence (ex: `comfort`).

## Points d’attention / dépannage

- **Calendrier d’absence (condition)**

  - Vérifiez la valeur de `absence_calendar_state`.
  - Si votre intention est “bloquer quand absence = on”, alors il faut que la condition soit l’inverse (non implémenté dans la version actuelle).

- **`outside_temperature` et `calendar_entity_state` non utilisés**

  - Ces inputs existent mais ne modifient pas le comportement actuel.

- **Attribut `start_time`**

  - Le trigger `preheat` dépend de `state_attr(calendar_entity, 'start_time')`.
  - Si votre intégration calendrier ne fournit pas `start_time`, le préchauffage ne se déclenchera pas.

- **Inertie apprise**

  - Si le capteur n’est pas numérique (ou si le taux est `<= 0`), le blueprint bascule sur un déclenchement à l’heure du calendrier (`start_time`) au lieu d’anticiper.

- **Hystérésis**
  - Le seuil `consigne - 0.2` évite de relancer un préchauffage si la température est déjà proche de la consigne.
