# Projet BTC — Prédiction de la volatilité

Projet de prédiction de la volatilité future du BTC (BTCUSDT 1h, données Binance Public Data).

## Objectif
Nettoyer le dataset, créer des features, explorer les données (EDA), puis entraîner un modèle scikit-learn.

## Répartition
- **Membre 1: Celia — Nettoyage** (`notebooks/01_cleaning.ipynb`) → produit `data/processed/btc_clean.csv`
- **Membre 2: Lina — Features & target** (`02_features.py`) → produit `data/processed/btc_features.csv`
- **Membre 3: Amandine — EDA & visualisations** (`03_eda.ipynb`)

## Structure
```
data/
├── raw/                      ← zips Binance (non versionnés)
└── processed/                ← CSV nettoyés (non versionnés)
notebooks/
└── 01_cleaning.ipynb         ← chargement + nettoyage
02_features.py                ← création des features et de la target
03_eda.ipynb                  ← exploration et graphiques
```
