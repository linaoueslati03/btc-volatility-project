# Documentation des features

Ce fichier décrit **toutes les colonnes créées** pour le projet : celles de base, puis celles ajoutées à la fin pour améliorer le modèle.

> **Feature** : une colonne d'information donnée au modèle pour l'aider à prédire. Elle ne regarde que le présent et le passé.
> **Target (cible)** : la colonne que le modèle doit deviner. C'est la seule qui a le droit de regarder le futur.

Le fichier de départ est `data/processed/btc_clean.csv` (données horaires du Bitcoin). Le résultat final est `data/processed/btc_features.csv`.

---

## 1. La target (ce qu'on veut prédire)

| Colonne | Comment elle est calculée | Ce que c'est |
|---|---|---|
| `volatility_24h_future` | écart-type des rendements des **24 prochaines heures** | La cible : la volatilité future du BTC |

> **Volatilité** : à quel point le prix bouge sur une période. On la mesure par l'**écart-type** des rendements.
> **Écart-type** : un chiffre qui dit si les valeurs sont proches les unes des autres (petit) ou très dispersées (grand).
> **Rendement** : la variation en % du prix d'une heure à l'autre.

C'est la seule colonne qui regarde le **futur** (décalage de 24 heures vers l'avant). Toutes les autres colonnes ne regardent que le passé, sinon ce serait de la triche (**fuite de données** : donner au modèle une information qu'il n'aurait pas en vrai au moment de prédire).

---

## 2. Les features de base

Créées au début du projet, à partir des données brutes.

| Feature | Comment elle est calculée | Pourquoi on l'a mise |
|---|---|---|
| `return_1h` | (prix de clôture actuel − précédent) / précédent | La brique de base : la variation de prix chaque heure |
| `volatility_24h_past` | écart-type des rendements des **24 dernières heures** | La volatilité récente : le meilleur indice de la volatilité à venir |
| `high_low_range` | (plus haut − plus bas) / clôture | L'amplitude du prix dans l'heure (écart entre le max et le min) |
| `open_close_return` | (clôture − ouverture) / ouverture | Le sens du mouvement dans l'heure (monté ou descendu) |
| `volume_change` | variation du volume échangé | Si l'activité accélère ou ralentit |
| `quote_volume_change` | variation du volume en dollars (USDT) | Même idée, mais mesurée en valeur |
| `trade_intensity` | nombre de transactions / volume | Combien de transactions par unité échangée |
| `buy_pressure` | volume d'achats « agressifs » / volume total | La part des acheteurs pressés (pression acheteuse) |
| `hour` | heure de la journée (0 à 23) | Pour capter un effet selon l'heure |
| `day_of_week` | jour de la semaine (0 = lundi) | Pour capter un effet selon le jour |

> **Volume** : la quantité de Bitcoin échangée pendant l'heure.

**Correction importante** : `trade_intensity` et `buy_pressure` sont des divisions par le volume. Si le volume vaut 0, la division est impossible (résultat infini). On remplace donc les volumes égaux à 0 par une valeur vide, qui sera ensuite supprimée.

---

## 3. Les features ajoutées à la fin (pour améliorer le modèle)

Au début, le modèle prédisait mal (voir `modele.md`). Les recherches montrent que **le meilleur indice de la volatilité future, c'est la volatilité passée observée sur plusieurs durées** (structure appelée *HAR*). Or on n'avait qu'**une seule** durée (24h). On a donc ajouté :

| Feature | Comment elle est calculée | Pourquoi on l'a ajoutée |
|---|---|---|
| `rv_6h` | écart-type des rendements des 6 dernières heures | Volatilité passée à **court terme** |
| `rv_72h` | écart-type des rendements des 72 dernières heures (3 jours) | Volatilité passée à **moyen terme** |
| `rv_168h` | écart-type des rendements des 168 dernières heures (7 jours) | Volatilité passée à **long terme** |
| `parkinson` | estimateur basé sur le plus haut et le plus bas | Une mesure de volatilité **plus précise** que l'écart-type simple |
| `atr_14` | amplitude moyenne des 14 dernières heures / clôture | L'amplitude typique récente du prix |
| `vol_ma_ratio` | volume / moyenne du volume sur 24h | Repère les **pics d'activité** par rapport à la normale |

**Résultat** : ajouter ces features (et changer la façon de prédire, voir `modele.md`) a beaucoup amélioré le modèle. C'était le plus gros progrès du projet : le problème ne venait pas du choix du modèle, mais du **manque de bonnes features**.

---

## 4. Nettoyage final

Certaines colonnes créent des cases vides (**NaN** : case sans valeur) :
- au **début** du tableau : les calculs sur plusieurs heures ont besoin d'un historique qui n'existe pas encore (ex. `rv_168h` a besoin de 168 heures avant de donner un résultat) ;
- à la **fin** du tableau : la target regarde 24 heures dans le futur, qui n'existent pas pour les dernières lignes.

On **supprime ces lignes incomplètes** pour ne garder que les lignes où toutes les colonnes ont une valeur. Le fichier final contient 4176 lignes et 28 colonnes.
