# Documentation de l'entraînement du modèle

Ce fichier explique, étape par étape et simplement, tout ce qu'on a fait pour entraîner le modèle qui prédit la volatilité future du Bitcoin. Le travail est dans `notebooks/04_model.ipynb`.

> **Modèle** : un programme qui apprend à deviner une valeur à partir d'autres valeurs.
> **Entraîner** : la phase où le modèle regarde des exemples déjà corrigés et ajuste ses règles internes.
> **Régression** : quand on prédit un **nombre** (ici, la valeur de la volatilité).
> **Classification** : quand on prédit une **catégorie** (ex. « monte » ou « baisse »).

Notre projet est un problème de **régression**.

---

## Étape 1 — Charger les données et séparer X et y

On charge `data/processed/btc_features.csv`, puis on sépare :
- **X** = les features (les indices donnés au modèle).
- **y** = la target `volatility_24h_future` (la réponse à deviner).

On enlève de X les colonnes de dates et **surtout la target** (sinon le modèle triche).

---

## Étape 2 — Découper en entraînement et test (dans l'ordre du temps)

On coupe les données en deux :
- **jeu d'entraînement** (80 % du début) : le modèle apprend dessus.
- **jeu de test** (20 % de la fin) : on vérifie dessus s'il devine bien.

> On garde l'**ordre du temps** : on entraîne sur le passé, on teste sur le futur. On ne mélange **pas** les lignes au hasard, sinon le modèle verrait des informations du futur pendant l'entraînement (**fuite de données**).

---

## Étape 3 — Entraîner un premier modèle

On utilise une **forêt aléatoire** (`RandomForestRegressor`), un modèle courant et simple à utiliser. L'apprentissage se fait avec la commande `.fit(...)`.

> **Forêt aléatoire** : un modèle qui construit beaucoup d'« arbres de décision » et fait la moyenne de leurs réponses.

---

## Étape 4 — Évaluer le modèle

On demande au modèle de prédire sur le jeu de test, puis on compare aux vraies valeurs avec trois mesures :

> **MAE** (erreur absolue moyenne) : de combien on se trompe en moyenne. Plus c'est petit, mieux c'est.
> **RMSE** (racine de l'erreur quadratique moyenne) : comme la MAE, mais pénalise davantage les grosses erreurs.
> **R²** (coefficient de détermination) : note globale entre le modèle et une prédiction « bête ». 1 = parfait ; 0 = aussi bon que prédire toujours la moyenne ; négatif = pire que prédire la moyenne.

On compare aussi à une **baseline** (modèle de référence très simple servant de point de comparaison) : ici, « la volatilité future sera égale à la volatilité passée ». Le but est de faire mieux que cette baseline.

---

## Étape 5 — Regarder l'importance des features

Le modèle indique quelles features il a le plus utilisées. Résultat : `volatility_24h_past` et `high_low_range` sont les plus importantes, ce qui confirme l'analyse visuelle (EDA).

---

## Étape 6 — Validation croisée temporelle

Tester sur une **seule** période peut tromper (cette période peut être particulière). On utilise donc `TimeSeriesSplit`.

> **Validation croisée** : tester le modèle sur plusieurs périodes différentes, puis faire la moyenne des scores, pour une évaluation plus fiable.
> **TimeSeriesSplit** : une validation croisée qui respecte l'ordre du temps (on entraîne toujours sur le passé, on teste sur la suite).

Résultat : le R² est **négatif sur les 5 périodes** testées. Le problème est donc difficile de façon systématique, pas juste un mauvais découpage.

---

## Étape 7 — Comparer plusieurs modèles

On ne peut pas savoir à l'avance quel modèle sera le meilleur : il faut les comparer sur les mêmes données. Scores en validation croisée (R² moyen) :

| Modèle | R² |
|---|---|
| RandomForest | −0.460 |
| GradientBoosting | −0.319 |
| **Ridge** | **−0.215** (le meilleur) |

> **GradientBoosting** : un modèle qui construit des arbres l'un après l'autre, chacun corrigeant les erreurs du précédent.
> **Ridge** : une régression linéaire (qui trace la « meilleure droite ») avec une **régularisation** (réglage qui empêche le modèle de donner des poids trop extrêmes, pour rester stable).

On garde **Ridge** : c'est le meilleur, et c'est un modèle linéaire qui sait **extrapoler** (deviner des valeurs en dehors de celles déjà vues), ce que les forêts et arbres ne savent pas faire.

> **Attention** : sur le seul découpage 80/20, Ridge semblait le pire. C'est justement pourquoi on se fie à la validation croisée (plusieurs périodes) et pas à un seul test.

---

## Étape 8 — Changer la cible : prédire la variation

Au lieu de prédire la volatilité directement, on a essayé de prédire sa **variation** (volatilité future − volatilité passée), puis de reconstruire la volatilité. Ça améliore un peu le résultat, car la variation est plus stable dans le temps.

---

## Étape 9 — La grosse amélioration : nouvelles features + cible log

C'est l'étape qui a le plus aidé. Deux changements :
1. **Ajouter les features multi-horizons** (`rv_6h`, `rv_72h`, `rv_168h`, etc. — voir `features.md`).
2. **Prédire le logarithme de la volatilité** au lieu de la volatilité brute.

> **Logarithme (log)** : une transformation mathématique qui « écrase » les grandes valeurs. La volatilité étant souvent très étirée vers le haut, prédire son log stabilise l'échelle et aide face aux changements de marché.

**Résultat** : le R² passe de **−0.215 à −0.062**. On a réduit d'environ 70 % l'écart au-dessus de la moyenne : on est passé tout près de zéro.

**À noter sur le graphique** : la courbe prédite du modèle amélioré paraît plus « plate » que l'ancienne, mais elle est **plus précise** (MAE plus faible) et mieux calibrée. Un modèle qui « suit visuellement » la courbe n'est pas forcément plus juste ; on juge sur les chiffres, pas sur l'allure.

---

## Étape 10 — Bonus : prédire la direction (classification)

On a aussi essayé de prédire non pas la valeur, mais la **direction** : la volatilité va-t-elle monter ou baisser ? C'est une **classification** (deux catégories).

> **AUC** : mesure de qualité pour une classification. 0.5 = hasard ; 1 = parfait.

Résultat : **AUC ≈ 0.78** et **67 % de bonnes réponses**. Il y a donc un vrai signal : prédire le **sens** est plus facile que prédire la **valeur exacte**.

> **À ne pas surinterpréter** : un bon AUC ne veut pas dire un profit garanti (il y a les frais, un seul actif testé, 6 mois de données seulement).

---

## Conclusion

- Prédire la **valeur exacte** de la volatilité horaire du BTC est difficile (R² proche de zéro) : c'est le cas le plus dur de la littérature scientifique.
- Le vrai levier n'était **pas le modèle**, mais **les features et la cible**.
- Un R² négatif ne prouve pas que le BTC est imprévisible : la **direction** garde un vrai signal.
- La démarche est rigoureuse : découpage dans l'ordre du temps, validation croisée, comparaison de modèles, puis amélioration par les features et la transformation de la cible.

**Modèle retenu** : Ridge, avec les features multi-horizons et la cible en log.
