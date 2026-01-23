# Chauffage – Calcul prochaine chauffe (slope VT + calendrier de présence)

Blueprint: `blueprints/predicat-chauffe.yaml`

## Objectif

Cette automatisation calcule une **heure optimale de démarrage de chauffe** (anticipation) afin d’atteindre une consigne Confort à l’heure de présence, en se basant sur :

- un **calendrier de présence** (prochain événement)
- un **slope** (°C/h) fourni par un capteur
- la **température actuelle** du thermostat
- une **consigne Confort** (helper `number.*`)

Le résultat est stocké dans un `input_datetime` (ex: `input_datetime.prochaine_chauffe`).

## Principe de fonctionnement

1. Récupère les événements du calendrier sur 7 jours via `calendar.get_events`.
2. Extrait le **début du prochain événement**.
3. Calcule le delta de température :
   - `delta = consigne_confort - température_actuelle`
4. Convertit le slope en °C/min et calcule la durée d’anticipation nécessaire.
5. Applique un plafond `inertie_max_minutes`.
6. Calcule `ts_final = debut_presence - anticipation`.
7. Écrit `ts_final` dans l’`input_datetime` de sortie si la valeur a changé.

## Déclencheurs

- `homeassistant.start`
- toutes les 5 minutes (`time_pattern minutes: /5`)
- changement d’état du calendrier de présence

## Entrées (inputs)

- **Thermostat** (`climate.*`)
- **Slope VT (°C/h)** (`sensor.*`) : capteur renvoyant un nombre (ex: `1.2`) ; les virgules sont acceptées (ex: `1,2`).
- **Consigne confort** (`number.*`) : valeur cible Confort.
- **Inertie max (minutes)** : borne max de l’anticipation.
- **Calendrier Présence** (`calendar.*`)
- **Stockage prochaine chauffe** (`input_datetime.*`) : reçoit le timestamp calculé.

## Pré-requis

- Une entité capteur “slope” en °C/h.
- Une entité `number.*` pour la consigne Confort.
- Un `input_datetime` pour stocker la prochaine chauffe.

## Exemple de mise en place

1. Créer un `input_datetime.prochaine_chauffe`.
2. Choisir le capteur `sensor.vt_slope`.
3. Choisir une consigne `number.consigne_confort`.
4. Choisir le calendrier de présence.
5. Créer une automatisation à partir du blueprint.

## Dépannage

- **La date calculée reste vide**

  - Vérifier qu’il existe au moins un événement à venir dans le calendrier.

- **Valeurs aberrantes**

  - Vérifier que le slope est > 0.
  - Vérifier que la consigne Confort est supérieure à la température actuelle (sinon l’anticipation devient 0).

- **Le calcul ne se met pas à jour**
  - L’automatisation n’écrit dans l’`input_datetime` que si le timestamp calculé est différent du précédent.
