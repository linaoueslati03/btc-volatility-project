# Présentation des features — partie de Lina (Membre 2)

Ce document sert de **support de présentation orale**. On explique chaque colonne une par une,
toujours avec le même schéma :

> **1. À quoi ça sert (1 phrase) → 2. La formule maths → 3. Le code Python utilisé → 4. Un exemple chiffré → 5. Pourquoi on l'a mise.**

Rappel des définitions clés :
- **Feature** = une colonne d'information donnée au modèle. Elle ne regarde que le **présent et le passé**.
- **Target** = la colonne à prédire. C'est la **seule** qui a le droit de regarder le futur.
- **Rendement** = variation en % du prix d'une heure à l'autre.
- **Volatilité** = à quel point le prix bouge, mesurée par l'**écart-type** des rendements.

On part de `data/processed/btc_clean.csv` (données horaires BTC de Celia) → on produit `data/processed/btc_features.csv`.

---

## 1. `return_1h` — le rendement horaire

**À quoi ça sert :** de combien le prix a bougé, en %, entre l'heure d'avant et maintenant.

**Formule (taux de variation relatif) :**
$$return\_1h = \frac{close_t - close_{t-1}}{close_{t-1}}$$

**Python :**
```python
df["return_1h"] = df["close"].pct_change()
```
> `.pct_change()` fait exactement `(valeur actuelle − précédente) / précédente`. La 1ʳᵉ ligne = `NaN` (pas d'heure avant).

**Exemple :** le BTC passe de 50 000 $ à 50 500 $ → `return_1h` = 500 / 50 000 = **+0,01 = +1 %**.

**Pourquoi :** c'est la **brique de base** de tout le projet. On ne travaille pas avec le prix brut mais avec sa variation en % (comparable quel que soit le niveau du prix). Et surtout, notre target — la volatilité — se calcule à partir de ces rendements.

---

## 2. `volatility_24h_past` — la volatilité récente

**À quoi ça sert :** à quel point le prix a bougé sur les **24 dernières heures**.

**Formule (écart-type des rendements sur 24h glissantes) :**
$$volatility\_24h\_past_t = \text{écart-type}\big(return\_1h_{\,t-23},\ \dots,\ return\_1h_{\,t}\big)$$

**Python :**
```python
df["volatility_24h_past"] = df["return_1h"].rolling(window=24).std()
```
> `.rolling(24).std()` = écart-type glissant sur les 24 dernières lignes (il regarde **en arrière**).

**Exemple :** si sur les dernières 24h les rendements sont très dispersés (−3 %, +4 %, −2 %…), l'écart-type est **grand** → marché agité. S'ils sont tous proches de 0, l'écart-type est **petit** → marché calme.

**Pourquoi :** la volatilité passée est **le meilleur indice** de la volatilité à venir (un marché agité a tendance à le rester quelques heures). C'est notre feature la plus importante.

---

## 3. `volatility_24h_future` — ⭐ LA TARGET (ce qu'on prédit)

**À quoi ça sert :** la volatilité des **24 prochaines heures** — c'est la valeur que le modèle doit deviner.

**Formule (même calcul mais tourné vers le futur) :**
$$volatility\_24h\_future_t = \text{écart-type}\big(return\_1h_{\,t+1},\ \dots,\ return\_1h_{\,t+24}\big)$$

**Python :**
```python
df["volatility_24h_future"] = df["return_1h"].rolling(window=24).std().shift(-24)
```
> On calcule l'écart-type glissant, puis `.shift(-24)` décale le résultat de 24 lignes vers le **haut** → la fenêtre pointe vers le futur.

**Exemple :** à l'instant `t`, cette colonne contient la volatilité qui sera observée entre `t+1` et `t+24`.

**Pourquoi / le piège :** c'est la **seule colonne autorisée à regarder le futur**. Les deux fenêtres (passé `[t-23…t]` et futur `[t+1…t+24]`) **ne se chevauchent pas** → pas de **fuite de données** (data leakage). Si une feature normale regardait le futur, ce serait de la triche et le modèle serait faussement bon.

---

## 4. `high_low_range` — l'amplitude dans l'heure

**À quoi ça sert :** l'écart entre le prix max et le prix min pendant l'heure.

**Formule :**
$$high\_low\_range = \frac{high - low}{close}$$

**Python :**
```python
df["high_low_range"] = (df["high"] - df["low"]) / df["close"]
```

**Exemple :** high = 51 000, low = 50 000, close = 50 500 → (51 000 − 50 000) / 50 500 ≈ **0,0198 ≈ 2 %**. Le prix a balayé ~2 % dans l'heure.

**Pourquoi :** une grande amplitude = une heure agitée. C'est une autre façon de mesurer l'agitation, complémentaire des rendements.

---

## 5. `open_close_return` — le sens du mouvement

**À quoi ça sert :** est-ce que le prix a monté ou baissé entre l'ouverture et la clôture de l'heure.

**Formule :**
$$open\_close\_return = \frac{close - open}{open}$$

**Python :**
```python
df["open_close_return"] = (df["close"] - df["open"]) / df["open"]
```

**Exemple :** open = 50 000, close = 50 250 → +0,5 % (l'heure s'est terminée plus haut qu'elle a commencé).

**Pourquoi :** donne la **direction** du mouvement horaire (positif = hausse, négatif = baisse), là où `high_low_range` ne donne que l'amplitude sans le sens.

---

## 6. `volume_change` — l'accélération de l'activité

**À quoi ça sert :** est-ce qu'on échange plus ou moins de BTC que l'heure d'avant.

**Formule :**
$$volume\_change = \frac{volume_t - volume_{t-1}}{volume_{t-1}}$$

**Python :**
```python
df["volume_change"] = df["volume"].pct_change()
```

**Exemple :** volume passe de 100 à 150 BTC → **+50 %** d'activité.

**Pourquoi :** les pics d'activité accompagnent souvent les gros mouvements de prix → utile pour anticiper la volatilité.

---

## 7. `quote_volume_change` — l'activité en dollars

**À quoi ça sert :** même idée que `volume_change`, mais mesurée en **valeur (USDT)** et non en quantité de BTC.

**Formule :**
$$quote\_volume\_change = \frac{quote\_asset\_volume_t - quote\_asset\_volume_{t-1}}{quote\_asset\_volume_{t-1}}$$

**Python :**
```python
df["quote_volume_change"] = df["quote_asset_volume"].pct_change()
```

**Pourquoi :** le volume en BTC et le volume en dollars ne bougent pas toujours pareil (car le prix change). Les deux ensemble donnent une image plus complète de l'activité.

---

## 8. `trade_intensity` — le nombre de trades par unité échangée

**À quoi ça sert :** est-ce que le volume vient de **beaucoup de petites transactions** ou de **quelques grosses**.

**Formule :**
$$trade\_intensity = \frac{number\_of\_trades}{volume}$$

**Python :**
```python
# volume == 0 remplacé par NaN pour éviter la division infinie (piège #2)
df["trade_intensity"] = df["number_of_trades"] / df["volume"].replace(0, np.nan)
```

**Exemple :** 1 000 trades pour 100 BTC → 10 trades par BTC.

**Pourquoi / le piège :** on **divise par le volume** → si `volume == 0`, la division donne l'infini. On remplace donc les 0 par `NaN` (case vide), qui sera supprimée à la fin. Une forte intensité peut signaler de la nervosité (beaucoup de petits ordres).

---

## 9. `buy_pressure` — la pression acheteuse

**À quoi ça sert :** la part des achats « agressifs » (ceux qui prennent le prix du marché immédiatement) dans le volume total.

**Formule :**
$$buy\_pressure = \frac{taker\_buy\_base\_volume}{volume}$$

**Python :**
```python
df["buy_pressure"] = df["taker_buy_base_volume"] / df["volume"].replace(0, np.nan)
```

**Exemple :** 70 BTC d'achats agressifs sur 100 BTC échangés → `buy_pressure` = **0,7** (70 % d'acheteurs pressés).

**Pourquoi :** un déséquilibre acheteurs/vendeurs précède souvent un mouvement de prix. Même piège que ci-dessus : division par le volume → gestion du 0.

---

## 10. `hour` — l'heure de la journée

**À quoi ça sert :** repérer un effet lié à l'heure (0 à 23).

**Python :**
```python
df["hour"] = df["open_time"].dt.hour
```

**Pourquoi :** les marchés sont plus ou moins actifs selon l'heure (ouverture des marchés US/Asie…). Le modèle peut capter ces motifs récurrents.

---

## 11. `day_of_week` — le jour de la semaine

**À quoi ça sert :** repérer un effet lié au jour (0 = lundi … 6 = dimanche).

**Python :**
```python
df["day_of_week"] = df["open_time"].dt.dayofweek
```

**Pourquoi :** l'activité du week-end diffère de celle de la semaine. Même logique que `hour`, à une autre échelle de temps.

---

# Features avancées (ajoutées pour améliorer le modèle)

> **Contexte à dire à l'oral :** au début le modèle prédisait mal. Les recherches montrent que le meilleur indice de la volatilité future, c'est la **volatilité passée sur plusieurs durées** (structure *HAR*, Corsi 2009). On n'avait qu'**une seule** durée (24h) → on en a ajouté d'autres. Ça a été **le plus gros progrès du projet** : le problème venait du manque de bonnes features, pas du choix du modèle.

## 12–14. `rv_6h`, `rv_72h`, `rv_168h` — la volatilité sur plusieurs horizons

**À quoi ça sert :** la même volatilité passée, mais mesurée à **court terme (6h)**, **moyen terme (72h = 3 jours)** et **long terme (168h = 7 jours)**.

**Formule :** écart-type des rendements sur des fenêtres de tailles différentes.
$$rv\_Nh_t = \text{écart-type}\big(return\_1h_{\,t-N+1},\ \dots,\ return\_1h_{\,t}\big)$$

**Python :**
```python
df["rv_6h"]   = df["return_1h"].rolling(window=6).std()
df["rv_72h"]  = df["return_1h"].rolling(window=72).std()
df["rv_168h"] = df["return_1h"].rolling(window=168).std()
```

**Pourquoi :** combiner court/moyen/long terme donne au modèle une vraie « photo » de la dynamique de la volatilité — bien plus riche qu'une seule fenêtre de 24h.

---

## 15. `parkinson` — un estimateur de volatilité plus précis

**À quoi ça sert :** mesurer la volatilité intra-heure à partir du plus haut et du plus bas (plus fin que l'écart-type simple).

**Formule (estimateur de Parkinson) :**
$$parkinson = \sqrt{\frac{1}{4\ln 2}\left(\ln\frac{high}{low}\right)^2}$$

**Python :**
```python
df["parkinson"] = np.sqrt((1 / (4 * np.log(2))) * (np.log(df["high"] / df["low"]) ** 2))
```

**Pourquoi :** utilise l'info high/low de **chaque heure** (pas juste la clôture) → une mesure de volatilité plus précise et réactive.

---

## 16. `atr_14` — l'amplitude moyenne récente

**À quoi ça sert :** l'amplitude typique du prix sur les 14 dernières heures.

**Formule :**
$$atr\_14 = \frac{\text{moyenne sur 14h de } (high - low)}{close}$$

**Python :**
```python
df["atr_14"] = (df["high"] - df["low"]).rolling(window=14).mean() / df["close"]
```

**Pourquoi :** version « lissée » de `high_low_range` (moyenne sur 14h) → capte l'amplitude de fond, moins bruitée qu'une seule heure.

---

## 17. `vol_ma_ratio` — le pic d'activité

**À quoi ça sert :** comparer le volume actuel à sa moyenne récente → repérer les **pics** par rapport à la normale.

**Formule :**
$$vol\_ma\_ratio = \frac{volume_t}{\text{moyenne sur 24h de } volume}$$

**Python :**
```python
df["vol_ma_ratio"] = df["volume"] / df["volume"].rolling(window=24).mean()
```

**Exemple :** ratio = 2 → on échange **2× plus** que d'habitude → activité anormalement forte.

**Pourquoi :** un volume anormalement haut annonce souvent une période agitée → bon signal pour la volatilité.

---

# Nettoyage final (à dire pour conclure)

Certaines features créent des cases vides (**NaN**) :
- au **début** du tableau : les calculs sur plusieurs heures (ex. `rv_168h`) ont besoin d'un historique qui n'existe pas encore ;
- à la **fin** : la target regarde 24h dans le futur, qui n'existent pas pour les dernières lignes.

On supprime ces lignes incomplètes :
```python
df_features = df.dropna().reset_index(drop=True)
df_features.to_csv("../data/processed/btc_features.csv", index=False)
```

**Résultat final : 4176 lignes et 28 colonnes** (les colonnes brutes conservées + les 16 features + la target).

---

## Résumé express (pour l'antisèche)

| # | Feature | Idée en 3 mots |
|---|---|---|
| 1 | `return_1h` | variation de prix / heure |
| 2 | `volatility_24h_past` | agitation des 24h passées |
| ⭐ | `volatility_24h_future` | **TARGET** — agitation des 24h futures |
| 4 | `high_low_range` | amplitude dans l'heure |
| 5 | `open_close_return` | sens du mouvement |
| 6 | `volume_change` | activité (quantité) |
| 7 | `quote_volume_change` | activité (dollars) |
| 8 | `trade_intensity` | trades par unité échangée |
| 9 | `buy_pressure` | part d'acheteurs agressifs |
| 10 | `hour` | effet heure |
| 11 | `day_of_week` | effet jour |
| 12–14 | `rv_6h / rv_72h / rv_168h` | volatilité court/moyen/long terme |
| 15 | `parkinson` | volatilité high/low (précise) |
| 16 | `atr_14` | amplitude moyenne sur 14h |
| 17 | `vol_ma_ratio` | pic d'activité vs normale |
