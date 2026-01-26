# Chauffage – Anticipation intelligente VT (auto-apprentissage + météo)

Blueprint: `blueprints/chauffage-anticipe.yaml`

## Objectif

Cette automatisation calcule une **heure optimale de démarrage de chauffe** (anticipation) afin d’atteindre une consigne Confort à l’heure de présence, en se basant sur :

- un **calendrier de présence** (prochain événement)
- un **slope** (°C/h) fourni par un capteur (Versatile Thermostat)
- la **température actuelle** du thermostat
- une **consigne Confort** (helper `number.*`)
- un **facteur de correction auto-apprenant** (helper `input_number.*`)
- une **correction météo** (température extérieure vs une référence)

Le résultat est stocké dans un `input_datetime` (ex: `input_datetime.prochaine_chauffe`).

## Principe de fonctionnement

1. Récupère les événements du calendrier sur 14 jours via `calendar.get_events`.
2. Extrait le **début du prochain événement** de présence.
3. Calcule le delta de température :
   - `delta = consigne_confort - température_actuelle`
4. Convertit le slope en °C/min :
   - `slope_min = slope_h / 60`
5. Calcule un **facteur global** :
   - `facteur_base` : valeur de l’`input_number` (auto-apprentissage)
   - `facteur_ext` : correction extérieure si `temp_ext` est plus froide que `temp_ext_reference`
     - `facteur_ext = 1 + (temp_ext_reference - temp_ext) * coeff_ext / 100`
     - plafonné à `1.3`
   - `facteur = facteur_base * facteur_ext`
6. Calcule les minutes d’anticipation nécessaires :
   - si `delta > 0` et `slope_min > 0` :
     - `minutes = ceil(delta / slope_min * facteur)`
   - sinon :
     - `minutes = 0`
7. Applique les bornes :
   - `minutes = max(minutes, inertie_min_minutes)`
   - `minutes = min(minutes, inertie_max_minutes)`
8. Calcule l’heure finale :
   - `ts_final = debut_presence - minutes`
9. Écrit `ts_final` dans l’`input_datetime` de sortie **uniquement si la valeur a changé**.

## Déclencheurs

- changement d’état du calendrier de présence (`on` -> `off`)
- toutes les 15 minutes (`time_pattern minutes: /15`)

## Entrées (inputs)

- **Thermostat Versatile** (`climate.*`) : utilisé pour lire `current_temperature`.
- **Slope VT utilisable (°C/h)** (`sensor.*`) : capteur renvoyant un nombre (ex: `1.2`) ; les virgules sont acceptées (ex: `1,2`).
- **Consigne confort** (`number.*`) : valeur cible Confort.
- **Facteur de correction (auto-apprenant)** (`input_number.*`) : multiplicateur appliqué au temps calculé.
- **Température extérieure** (`sensor.*`) : utilisée pour moduler le temps d’anticipation.
- **Température extérieure de référence (°C)** : base de comparaison (défaut: `10`).
- **Sensibilité température extérieure (% / °C)** : intensité de la correction (défaut: `5`).
- **Anticipation maximale (minutes)** : plafond de l’anticipation (défaut: `180`).
- **Anticipation minimale (minutes)** : plancher de l’anticipation (défaut: `15`).
- **Calendrier Présence** (`calendar.*`) : source des événements à venir.
- **Stockage prochaine chauffe** (`input_datetime.*`) : reçoit le timestamp calculé.

## Pré-requis

- Une entité “slope” en °C/h.
- Une entité `number.*` pour la consigne Confort.
- Une entité `input_number.*` pour le facteur auto-apprenant.
- Un `input_datetime` pour stocker la prochaine chauffe.
- Le thermostat doit exposer l’attribut `current_temperature`.

## Exemple de mise en place

1. Créer un `input_datetime.prochaine_chauffe`.
2. Choisir le capteur `sensor.vt_slope`.
3. Choisir une consigne `number.consigne_confort`.
4. Choisir un facteur `input_number.facteur_chauffe_piece`.
5. Choisir un capteur extérieur `sensor.temperature_exterieure`.
6. Choisir le calendrier de présence.
7. Créer une automatisation à partir du blueprint.

## Dépannage

- **La date calculée reste vide**

  - Vérifier qu’il existe au moins un événement à venir dans le calendrier (le blueprint prend le premier événement retourné par `calendar.get_events`).

- **Valeurs aberrantes**

  - Vérifier que le slope est > 0.
  - Vérifier que la consigne Confort est supérieure à la température actuelle (sinon le calcul produit `0`, puis une valeur bornée par `inertie_min_minutes`).
  - Vérifier la valeur du facteur auto-apprenant (ex: `1.0` à `1.5` selon la pièce).

- **La correction météo ne semble pas agir**

  - Si la température extérieure est supérieure à la référence, le facteur extérieur vaut `1` (pas de réduction du temps).
  - La correction extérieure est plafonnée à `1.3`.

- **Le calcul ne se met pas à jour**

  - L’automatisation n’écrit dans l’`input_datetime` que si le timestamp calculé est différent du précédent.
