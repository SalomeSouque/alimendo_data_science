# Alimendo — data science

Notebooks d'exploration, de calibration et d'entraînement du projet **Alimendo**, site d'aide aux choix
alimentaires pour les personnes atteintes d'endométriose (projet étudiant — certification Développeur en
intelligence artificielle, RNCP37827).

Ce dépôt sert à **décider et justifier** : chaque choix de données ou de modèle y est mesuré.
Le code qui tourne dans l'application vit dans le dépôt applicatif
[`alimendo_certification_rncp`](https://github.com/SalomeSouque/alimendo_certification_rncp), qui reprend
les résultats produits ici.

> Projet étudiant. Le score d'inflammation est une **approche inspirée du Dietary Inflammatory Index**
> (Shivappa et al., 2014) appliquée à des aliments isolés ; il n'a aucune valeur de conseil médical.

---

## Contenu

| Notebook | Objet | Produit | Repris dans l'application |
|---|---|---|---|
| `01_calibrage_score_aliments.ipynb` | Calibration du score d'inflammation sur CIQUAL 2020 | `landmarks_v1.json` (repères de normalisation v1.0) | `backend/app/domain/score/landmarks_v1.json` |
| `02_classifieur.ipynb` | Classifieur d'intention du chatbot (TF-IDF + régression logistique) et filet de sécurité « détresse » | `intent_classifier.pkl` + `intent_classifier_meta.json` | Routage des questions du chatbot |
| `03_ciqual_cleaning.ipynb` | Audit et nettoyage de la table CIQUAL 2020 | Jeu d'aliments nettoyé, rejets, rapport | `scripts/ciqual_clean.py` (résultat identique, vérifié en § 6.2) |

Ordre de lecture conseillé : **03 → 01 → 02**. Le nettoyage (03) a été formalisé après la calibration (01),
mais il en reproduit exactement la conversion des valeurs (§ 5.2 : repères v1.0 retrouvés à l'identique).

---

## 01 · Calibration du score d'inflammation

- **23 paramètres nutritionnels** (7 pro-inflammatoires, 16 anti-inflammatoires), direction et intensité
  reprises du DII ; énergie retirée (double comptage), graisses trans absentes de CIQUAL.
- **Normalisation centrée [−1, +1]** sur la médiane, bornée par les 10ᵉ et 90ᵉ centiles. Une première
  normalisation min-max [0, 1] a été testée puis abandonnée : elle ne pénalisait jamais une carence et classait
  les corps gras comme anti-inflammatoires. Le z-score est écarté (distributions très asymétriques).
- **Seuils −2…+2** fixés sur les quintiles : `S1 = −0,7268 · S2 = 0,0307 · S3 = 0,7063 · S4 = 1,5368`.
- **Score indisponible** si complétude < 50 % ou glucides + protéines + lipides < 1 g/100 g :
  2 342 aliments scorés, 844 indisponibles.
- Les artefacts identifiés (épices séchées, aliments enrichis, charcuteries) sont documentés plutôt que masqués.

## 02 · Classifieur d'intention

- **5 classes** : `detresse_urgence`, `restriction_alimentaire`, `out_of_scope`, `food_specific`, `in_scope`.
- **Données synthétiques** générées par LLM : 375 questions d'entraînement (3 générateurs : Claude Opus 5,
  Claude Opus 4.8, GPT Luna), 125 de test produites par **un générateur jamais vu à l'entraînement**
  (Gemini 1.5 Pro), pour mesurer la généralisation à d'autres formulations. 0 fuite train/test.
- **Modèle** : TF-IDF (unigrammes + bigrammes, sans suppression des mots vides, qui portent le signal)
  + régression logistique.
- **Filet de sécurité déterministe** sur la détresse : un message de détresse mal routé est l'erreur la plus
  grave, le filet prime donc sur le modèle.

| Métrique (jeu de test) | Modèle seul | Modèle + filet |
|---|---|---|
| Accuracy | 0,664 | 0,712 |
| **Rappel `detresse_urgence`** | 0,72 | **1,00** (25/25) |

L'accuracy est une métrique de contrôle ; le critère de décision est le rappel sur la détresse.

## 03 · Nettoyage CIQUAL

Démarche : mesurer chaque problème sur tout le jeu, examiner les lignes, puis décider.

- **Formats non normalisés** sur 67 colonnes : virgule décimale (38,9 %), tiret « non mesuré » (29,8 %),
  « < X » (7,8 %), traces (0,9 %), cellule vide (0,7 %). Conversion selon le référentiel v1.0 :
  **non mesuré → NULL, jamais 0** ; traces et « < X » → 0.
- **Doublon** `alim_code` 9621 fusionné (0 contradiction) ; **44 noms de groupe** réparés depuis leur code.
- **5 règles de cohérence physique** (dont la cohérence énergie / macronutriments du Règlement UE 1169/2011,
  avec correction du double compte des polyols) : **1 entrée corrompue** rejetée.
- Bilan : **3 186 → 3 185 → 3 184 aliments**. Les aliments non scorables sont conservés : ils relèvent du calcul
  du score, pas du nettoyage.
- Contrôles avant import : schéma PostgreSQL (débordement du bêta-carotène, précision des oméga),
  non-régression des repères v1.0, équivalence avec le script du dépôt applicatif.

---

## Environnement

Les notebooks tournent dans un conteneur **Jupyter (Docker)**, sur CPU.

| Paquet | Utilisé par |
|---|---|
| `pandas` (testé en 3.0.5), `numpy`, `matplotlib` | tous |
| `requests` | 01, 03 (téléchargement CIQUAL) |
| `xlrd`, `openpyxl` | 01, 03 (lecture du fichier CIQUAL) |
| `seaborn`, `scikit-learn`, `joblib` | 02 |

Les chemins sont relatifs à la racine du dépôt : lancer Jupyter depuis ce dossier.

## Données

Le dossier `data/` n'est **pas versionné** (`.gitignore`), pas plus que les modèles `*.pkl`.

| Fichier attendu | Source | Obtention |
|---|---|---|
| `data/raw/ciqual_2020.xls` | Table CIQUAL 2020, ANSES — Licence Ouverte / Etalab 2.0 | Téléchargé automatiquement par les notebooks 01 et 03 depuis data.gouv.fr |
| `data/classifieur/dataset_train_*.csv` (×3), `dataset_test_GEMINI_15pro.csv` | Questions synthétiques générées par LLM | À placer manuellement (colonnes `id_cat`, `name_cat`, `question`, `generator`) |
| `data/script/aliments_ciqual_2020_clean.csv` | Sortie de `scripts/ciqual_clean.py` (dépôt applicatif) | Copie manuelle, uniquement pour la cellule 6.2 du notebook 03 |

Citation CIQUAL : *Anses. 2020. Table de composition nutritionnelle des aliments Ciqual.*

## Conventions

- Une cellule = un problème ; chaque tableau et chaque graphique est titré.
- Les décisions sont consignées dans des cellules « Lecture » avec les chiffres affichés juste au-dessus.
- Les colonnes source ne sont jamais écrasées ; les garde-fous (`assert`) précèdent les exports.
- Commits au format Conventional Commits (`feat(data)`, `test(data)`, `chore`…), une branche par travail,
  fusion par pull request.

## Limites et points ouverts

- **CIQUAL 2025** existe (publiée en novembre 2025) ; le projet reste sur 2020, base de calibration du
  référentiel v1.0. Changer de version imposerait une recalibration (v2.0).
- La convention « < X → 0 » pèse fortement sur quelques paramètres (sélénium, DHA, EPA, vitamine D) :
  limite documentée au notebook 03, § 3.3.
- La complétude du score est calculée sur 21 paramètres dans le notebook 01 et sur 23 dans le backend
  (17 aliments concernés, notebook 03 § 5.4) : définition à arrêter dans le référentiel.
- Le classifieur est entraîné sur 375 exemples synthétiques : ses performances hors filet de sécurité
  (accuracy 0,664) restent modestes.