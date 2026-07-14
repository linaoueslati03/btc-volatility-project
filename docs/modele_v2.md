# Modèle v2 — l'avancée après l'ajout de 6 mois de données

Ce document compare les résultats du modèle **avant** (6 mois de données, 4176 lignes) et **après** l'extension du dataset à **12 mois** (juil. 2025 → juin 2026, 8568 lignes). Il accompagne le notebook `notebooks/04_modelnv.ipynb`.

> L'ancien bilan détaillé (6 mois) est conservé dans `docs/modele.md` (archive).
> Rappel : **R²** = qualité globale (1 = parfait ; 0 = aussi nul que prédire la moyenne ; **négatif = pire que la moyenne**). **MAE** = erreur moyenne (plus petit = mieux).

---

## Le message en une phrase

> **Doubler les données (6 → 12 mois) a fait passer le modèle du négatif au positif.** Prédire la volatilité future du BTC, longtemps jugé impossible en horaire, devient réellement faisable — et ce sont désormais les **modèles d'arbres** (GradientBoosting, RandomForest) qui gagnent, pas Ridge.

---

## Tableau récapitulatif (R² en validation croisée, moyenne sur 5 périodes)

| Modèle | Avant (6 mois) | Après (12 mois) | Évolution |
|---|---|---|---|
| RandomForest (absolu) | −0.460 | **+0.172** | négatif → positif |
| GradientBoosting (absolu) | −0.319 | **+0.201** | négatif → positif |
| Ridge (absolu) | −0.215 | +0.135 | négatif → positif |
| **GradientBoosting (variation)** | — | **+0.208** | ⭐ meilleur |
| Ridge + features + cible log | −0.062 | +0.136 | négatif → positif |
| RandomForest — test 80/20 unique | négatif | **+0.331** | négatif → positif |

---

## Étape par étape (chaque calcul Python du notebook)

### Partie 5 — Évaluation du 1ᵉʳ modèle (RandomForest, test 80/20)
- **Calcul :** `r2_score(y_test, y_pred)` sur les 20 % de la fin.
- **Avant :** R² négatif, le modèle ne battait pas la moyenne.
- **Après :** **MAE = 0.00126, R² = 0.331**, et il bat la baseline naïve (R² = −0.039).
- **Lecture :** sur la période récente, le modèle explique déjà ~33 % de la variance de la volatilité.

### Partie 7 — Validation croisée temporelle (`TimeSeriesSplit`)
- **Calcul :** moyenne du R² sur 5 périodes successives.
- **Avant :** R² **négatif sur les 5 périodes** → problème « difficile de façon systématique ».
- **Après :** R² moyen **+0.172** ; 4 périodes sur 5 sont positives (seule la 1ʳᵉ, la plus courte, reste à −0.03).
- **Lecture :** le modèle **généralise** dans le temps, ce n'était pas un coup de chance sur un découpage.

### Partie 8 — RandomForest vs GradientBoosting
- **Calcul :** même validation croisée pour les deux algorithmes.
- **Avant :** RandomForest −0.460, GradientBoosting −0.319 (les deux négatifs).
- **Après :** RandomForest **+0.172**, GradientBoosting **+0.201** (les deux positifs).
- **Lecture :** GradientBoosting est un peu meilleur, mais le vrai déclencheur reste la **quantité de données**.

### Partie 9 — Prédire la *variation* de volatilité
- **Calcul :** cible = `volatility_24h_future − volatility_24h_past`, puis reconstruction.
- **Avant :** ≈ −0.25 (la « moins mauvaise » approche).
- **Après :** **R² = 0.208** avec GradientBoosting → **meilleur résultat du projet**.
- **Lecture :** la bonne cible + assez de données = un modèle franchement utile.

### Partie 10 — Bilan des modèles (le classement s'est inversé)
- **Avant :** Ridge était **le meilleur** (−0.215), car seul modèle capable d'**extrapoler** ; les arbres ne savent pas prédire des niveaux jamais vus.
- **Après :** classement inversé — **GradientBoosting (variation) 0.208 > GB absolu 0.201 > RandomForest 0.172 > Ridge 0.135**.
- **Lecture :** avec 12 mois, les arbres ont **assez d'exemples** pour couvrir les différents régimes de marché → ils dépassent Ridge.

### Partie 11 — Feature engineering + cible log
- **Calcul :** Ridge + features multi-horizons (`rv_6h`, `rv_72h`, `rv_168h`, `parkinson`, `atr_14`, `vol_ma_ratio`) + cible `log(volatilité)`.
- **Avant :** faisait bondir Ridge de **−0.215 à −0.06** → **le plus gros gain du projet**.
- **Après :** Ridge part déjà de +0.135 ; avec features + log, il atteint +0.136 → **gain marginal**.
- **Lecture :** ces techniques **compensaient surtout le manque de données**. Avec un dataset doublé, le modèle capte déjà l'essentiel du signal sans elles.

### Comparaison « courbe plate » (test 80/20, même période)
| | RandomForest | Ridge + features + log | Réel |
|---|---|---|---|
| MAE | **0.00126** ✅ | 0.00133 | — |
| R² | **0.331** ✅ | 0.154 | — |
| Moyenne prédite | 0.00414 (≈ réel) | 0.00369 | 0.00400 |

- **Avant :** le modèle « amélioré » (Ridge + log) était le plus précis et RandomForest surestimait.
- **Après :** **RandomForest repasse devant** (MAE plus basse, meilleur R², bien calibré).
- **Lecture :** on juge sur les chiffres, pas sur l'allure ; et le meilleur modèle **dépend de la quantité de données**.

---

## Importance des features (inchangée sur le fond)
`volatility_24h_past` (0.27) et `high_low_range` (0.21) restent les plus importantes, suivies de `day_of_week` (0.13) et `trade_intensity` (0.11). Cohérent avec l'EDA et avec l'ancienne version.

---

## Conclusion pour la présentation

1. **Le levier décisif était les données** : doubler la période (6 → 12 mois) a fait basculer tous les modèles du négatif au positif.
2. **Le classement des modèles a changé** : les arbres (GradientBoosting en tête) dépassent maintenant Ridge.
3. **Les techniques de la v1** (variation, features multi-horizons, cible log) restaient utiles mais leur gain est devenu marginal une fois les données suffisantes.
4. **Modèle recommandé désormais : GradientBoosting** (cible « variation », R² ≈ 0.21 en validation croisée).

> Démarche rigoureuse conservée : découpage temporel sans triche, validation croisée, comparaison de modèles, feature engineering.
>
> **Sources :** Corsi (2009), HAR-RV ; Andersen-Bollerslev-Diebold-Labys (2003), log-volatilité ; Bergsli et al. (2022), HAR vs GARCH sur Bitcoin.
