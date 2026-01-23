# Chauffage – Confort selon heure calculée (événementiel)

Blueprint: `blueprints/chauffage-auto.yaml`

## Objectif

Cette automatisation applique automatiquement un mode **Confort** ou **Éco** sur un thermostat (preset + consigne), en fonction :

- d’un **calendrier de présence**
- d’un **calendrier d’absence** (inhibiteur)
- d’une **heure de démarrage Confort** (stockée dans un `input_datetime`)

Elle ré-applique également le preset/consigne après un redémarrage de Home Assistant.

## Principe de fonctionnement

- Si le **calendrier d’absence** est `on`, l’automatisation est bloquée (condition globale).
- Sinon, l’automatisation calcule une fenêtre `comfort_window` :
  - présence active (`calendar_presence` == `on`)
  - et l’heure courante est comprise entre `heure_prochaine_chauffe` et `end_time` de l’événement de présence
- Si `comfort_window` est vraie :
  - `desired_preset = preset_confort`
  - `desired_temperature = consigne_confort`
- Sinon :
  - `desired_preset = preset_eco`
  - `desired_temperature = consigne_eco`

L’automatisation :

- applique `climate.set_preset_mode`
- applique `climate.set_temperature`
- force ensuite `climate.set_hvac_mode: heat`

## Déclencheurs

- `homeassistant.start` (avec un délai de 1 min 30 avant action)
- changement d’état du calendrier de présence
- changement d’état du calendrier d’absence
- changement de `heure_prochaine_chauffe`

## Entrées (inputs)

- **Thermostat** (`climate.*`)
- **Calendrier Présence** (`calendar.*`)
- **Calendrier Absence** (`calendar.*`) : si `on`, l’automatisation ne fait rien
- **Heure de démarrage Confort** (`input_datetime.*`) : heure à partir de laquelle on peut passer en Confort
- **Preset Confort** (valeur `preset_mode`, ex: `comfort`)
- **Preset Éco** (valeur `preset_mode`, ex: `eco`)
- **Consigne confort** (`number.*`) : entité “Number” contenant une valeur (°C) à appliquer via `climate.set_temperature`
- **Consigne éco** (`number.*`) : entité “Number” contenant une valeur (°C)

## Pré-requis

- Un thermostat supportant :
  - `climate.set_preset_mode`
  - `climate.set_temperature`
  - `climate.set_hvac_mode`
- Un calendrier de présence qui expose `end_time` quand il est `on`.

## Exemple de mise en place

1. Créer/choisir :
   - un calendrier `calendar.presence_maison`
   - un calendrier `calendar.absence_maison`
2. Créer un `input_datetime.heure_prochaine_chauffe` (sans date, uniquement l’heure).
3. Créer deux helpers `number.consigne_confort` et `number.consigne_eco`.
4. Créer une automatisation à partir du blueprint et renseigner les inputs.

## Dépannage

- **Erreur `expected float for dictionary value @ data['temperature']`**

  - Vérifier que `consigne_confort` / `consigne_eco` pointent bien vers des entités `number.*` ayant un état numérique.

- **Le Confort ne s’active pas**

  - Vérifier que `calendar_presence` est bien `on`.
  - Vérifier que `heure_prochaine_chauffe` est défini.
  - Vérifier que `end_time` est présent dans les attributs du calendrier quand il est `on`.

- **L’automatisation ne fait rien**

  - Vérifier que `calendar_absence` est `off`.

- **À propos du redémarrage**
  - Au trigger `homeassistant.start`, l’automatisation attend 1 min 30 puis ré-applique preset/consigne.
