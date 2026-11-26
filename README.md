# TinyInsta passe-t-elle à l'échelle ? — Benchmark 2026

Question du projet : qu'est-ce qui passe à l'échelle, jusqu'où, et pourquoi ?

## 1. Montage

* **Projet et région** : projet GCP `monprojet-515813`, région `europe-west1`.
* **Application et lancement** : Google App Engine standard, runtime `python312`, lancée avec `gunicorn -b :$PORT main:app` (sans option `--workers`, d'où l'hypothèse d'un seul worker par instance, non vérifiée dans les logs Gunicorn).
* **Versions et instances** :
  * Version `default` (`app.yaml`) : classe `F1`, mise à l'échelle automatique (`automatic_scaling`, `max_instances: 10`), versions déployées `20261125t141537` et `20261126t140850`.
  * Versions `fixedN` (`deploy_fixed.sh`) : classe `B1` (même CPU et mémoire que `F1`), mise à l'échelle manuelle (`manual_scaling`, `instances: N`), déployées avec `--no-promote` (`fixed1`, `fixed2`, `fixed4`), puis arrêtées après usage.
* **Stockage et client de charge** : Datastore (Firestore en mode Datastore) ; client Locust lancé depuis Cloud Shell ; runs d'environ 45 secondes.
* **Dates et jeux de données** : toutes les mesures ont été réalisées le 26 novembre 2026 sur les jeux générés avec `bench/seed_exact.py` (`base`, `u100`, `u10000`, `p10`, `p100`, `p1000`, `f05` à `f100`).
* **Organisation du dépôt et reproductibilité** :
  * `raw/` : 67 journaux CSV bruts (26 pour la partie 1, 21 pour la partie 2, 20 pour la partie 3).
  * `figures/` : graphiques `.png` (`fig1a.png`, `fig1b.png`, `fig2a.png`, `fig2b.png`, `fig3.png`) et données `.csv` associées.
  * `scripts/` : `run_p1.sh`, `run_p2.sh`, `run_p3.sh`, `analyse_p1.py`, `analyse_p2.py`, `analyse_p3.py`, `common.py`, `make_figures.sh`.
  * `patches/` : `main_original.py` (code initial) et `main_corrige.py` (correctif de la partie 3).
  * `main.py` à la racine : code corrigé de la partie 3 (les parties 1 et 2 ont été mesurées avec `patches/main_original.py`).
  * Régénération de toutes les figures en une commande : `bash scripts/make_figures.sh`.

## 2. Résultats

### Partie 1 : coût d'une requête

* **Hypothèse** : dans `get_timeline` (`patches/main_original.py`), l'application lit l'entité `User` par sa clé, puis interroge `Post` avec un filtre `IN` sur les comptes suivis (plus l'utilisateur lui-même), trié par date décroissante avec une limite de 20 posts. On attend donc un temps serveur constant quand `U` ou `P` varie, et une hausse progressive quand `F` augmente.
* **Protocole** : version `fixed1` (1 instance), 1 utilisateur Locust (`-u 1`), un seul paramètre varié à la fois autour des valeurs de référence.

![Fig. 1a — Temps serveur médian en fonction de U et de P](figures/fig1a.png)

![Fig. 1b — Temps serveur médian et latence client médiane en fonction de F](figures/fig1b.png)

* **Résultats** (valeurs de `figures/fig1a.csv` et `figures/fig1b.csv`) :
  * `U` (`100`, `1000`, `10000`) : temps serveur médian de `27,2 ms`, `24,5 ms` et `25,2 ms` (quasi constant).
  * `P` (`10`, `100`, `1000`) : temps serveur médian de `27,5 ms`, `26,4 ms` et `26,2 ms` (quasi constant).
  * `F` de `5` à `25` : temps serveur médian de `23,8 ms` à `26,6 ms` (quasi constant).
  * `F = 30` : saut à `484,4 ms` (environ 18 fois plus élevé qu'à `F = 25`), puis montée régulière jusqu'à `1611,4 ms` à `F = 100`.
* **Interprétation** :
  * L'hypothèse est confirmée pour `U` et `P`, mais réfutée pour `F` à cause du saut à `F = 30`. Le filtre `IN` accepte au plus 30 valeurs ; avec `follows` augmenté de l'utilisateur (`F + 1` valeurs), `F = 25` fait 26 valeurs (ok) et `F = 30` fait 31 valeurs (dépassement). Le code bascule alors sur son plan de secours (une requête par auteur l'une après l'autre), d'où un coût qui croît avec le nombre d'auteurs (environ `15,6 ms` par auteur à `F = 30` et `~16 ms` à `F = 100`).
  * **Part du réseau** : la différence `client_med_ms - server_med_ms` vaut environ `11` à `17 ms` sur la plupart des points (et `21,75 ms` à `F = 100`), soit environ un tiers de la latence client quand `F` est petit (`13,7 ms` sur `40,9 ms` pour `U = 100`), mais une part très faible quand `F` est grand (`~1,3 %` à `F = 100`).
  * **Entités lues et coût estimé pour 1 million de timelines** (calcul théorique d'après le code et le ratio de facturation `1,99 $` pour `3 415 144` lectures, soit `~0,0583 $` pour `100 000` lectures, donc `~0,583 $` par entité lue pour 1 million de timelines ; tarif catalogue officiel à vérifier) :

| Cas | Entités lues par timeline (calcul code) | Coût estimé pour 1 million de timelines |
| :--- | :---: | :---: |
| `U`, `P` quelconques, `F <= 25` (filtre `IN`) | `21` (`1` User + `20` posts max) | `~12 $` |
| `F = 30` (code d'origine, plan de secours) | `621` (`1` User + `31` auteurs × `20` posts) | `~362 $` |
| `F = 100` (code d'origine, plan de secours) | `2021` (`1` User + `101` auteurs × `20` posts) | `~1 178 $` |

### Partie 2 : capacité

* **Hypothèse** : l'application étant lancée avec `gunicorn -b :$PORT main:app` sans option `--workers` (hypothèse d'un seul worker par instance), chaque instance traite les requêtes séquentiellement : avec une latence d'environ `40 ms`, une instance devrait plafonner autour de `25 req/s`, et `N` instances devraient multiplier le débit par environ `N`, avec saturation dès que `C` dépasse `N`.
* **Protocole** : versions `fixed1`, `fixed2` et `fixed4` (`N = 1, 2, 4`), jeu `base`, concurrence `C` dans `{1, 2, 4, 8, 16, 32, 64}`.

![Fig. 2a — Débit en fonction de C selon N](figures/fig2a.png)

![Fig. 2b — Latence médiane et loi de Little en fonction de C](figures/fig2b.png)

* **Résultats** (valeurs de `figures/fig2a.csv` et `figures/fig2b.csv`) :
  * Débit maximal observé : `25,90 req/s` (`N = 1`, déjà `22,83 req/s` à `C = 1`), `52,54 req/s` (`N = 2`), `84,08 req/s` (`N = 4`).
  * Loi de Little (`latence ≈ C / débit`) : proche de la latence médiane à faible/moyenne charge (ex. `N = 1, C = 1` : `40,0 ms` mesuré vs `43,8 ms` prédit ; `C = 16` : `596,7 ms` vs `617,9 ms`), mais s'en écarte nettement à forte saturation (ex. `N = 2, C = 64` : `641,0 ms` mesuré vs `1308,1 ms` prédit).
* **Interprétation** :
  * Une instance plafonne à environ `25–26 req/s` (cohérent avec un worker unique et une latence de l'ordre de `40 ms`), avec saturation dès `C = 1–2` pour `N = 1`, vers `C = 4–16` pour `N = 2`, et vers `C = 4–16` pour `N = 4`.
  * De `N = 1` à `N = 2`, le débit maximal est multiplié par `~2,03×` ; de `N = 1` à `N = 4`, par `~3,25×` (sous-linéaire). Je n'ai pas fait de mesure supplémentaire pour prouver la cause exacte de cet écart à `4×`.
  * Pour la loi de Little au-delà de la saturation, l'écart peut venir de l'usage de la latence médiane au lieu de la moyenne (sensible aux requêtes très lentes), sans vérification expérimentale complémentaire.

### Partie 3 : corriger et re-mesurer

* **Goulot choisi et hypothèse** : le dépassement de la limite de 30 valeurs du filtre `IN` sur `F`. En découpant `follows` en lots d'au plus 30 auteurs (`range(0, len(follows), 30)`), on exécute une requête par lot (`ceil((F + 1) / 30)` lots), ce qui doit réduire fortement le temps serveur pour `F >= 30` et faire apparaître des paliers autour de `F = 30`, `60` et `90`.
* **Correction et protocole** : `patches/main_corrige.py` (copié dans `main.py`) exécute une requête `IN` triée par date et limitée à 20 posts par lot de 30 auteurs, puis fusionne et trie les résultats pour garder les 20 posts les plus récents. Déployé sur `fixed1` (`--no-promote`) et re-mesuré sur les 20 valeurs de `F` (`f05` à `f100`).

![Fig. 3 — Comparaison avant et après correction en fonction de F](figures/fig3.png)

* **Résultats** (valeurs de `figures/fig3.csv`) :
  * `F = 25` : temps serveur médian `26,6 ms` (avant) vs `32,25 ms` (après).
  * `F = 30` : `484,4 ms` (avant) vs `50,35 ms` (après).
  * `F = 60` : `984,7 ms` (avant) vs `74,6 ms` (après).
  * `F = 100` : `1611,4 ms` (avant) vs `97,5 ms` (après), soit environ `16,5×` plus rapide sur le temps serveur médian.
* **Interprétation et coûts** :
  * L'hypothèse est confirmée : le temps serveur passe d'environ `50 ms` (`F = 30..55`) à `~68–75 ms` (`F = 60..85`) puis `~86,5–97,5 ms` (`F = 90..100`). L'écart de quelques millisecondes à `F <= 25` (zone non modifiée par le correctif) n'a pas été investigué davantage (possible effet de déploiement/contexte).
  * **Complexité** : ajout d'une boucle sur les lots de 30 auteurs et d'un tri en mémoire dans l'application.
  * **Coût financier** : à `F = 100` (4 lots), estimation théorique d'environ `81` entités lues par timeline (`1` User + `4 × 20` posts max) au lieu de `2021`, soit environ `47 $` pour 1 million de timelines au lieu de `1178 $` (même tarif de lecture qu'en partie 1, non mesuré directement).
  * **Cohérence des données** : chaque lot sélectionne au plus ses 20 posts les plus récents, et le tri global conserve les 20 plus récents de l'union : le résultat est identique à une requête globale triée par date décroissante.

## 3. Tableau des verdicts

| Dimension | Passe à l'échelle ? | Jusqu'où ? | Pourquoi (mécanisme) | Preuve |
| :--- | :--- | :--- | :--- | :--- |
| Utilisateurs (`U`) | Oui | Testé jusqu'à `10 000` | Lecture par clé (`User`) puis requête limitée à 20 posts | Fig. 1a |
| Posts par utilisateur (`P`) | Oui | Testé jusqu'à `1 000` | Requête limitée à 20 posts | Fig. 1a |
| Followees (`F`), code d'origine | Non | Jusqu'à `F = 25` (`26` valeurs avec l'utilisateur) | Filtre `IN` limité à 30 valeurs ; au-delà, plan de secours séquentiel par auteur | Fig. 1b |
| Followees (`F`), code corrigé | Mieux, mais pas constant | Testé jusqu'à `F = 100` | Requêtes par lots d'au plus 30 auteurs (`ceil((F + 1) / 30)` lots) | Fig. 3 |
| Instances (`N`) | En partie | `1 -> 2` : `~2,0×` ; `1 -> 4` : `~3,25×` | Un seul worker Gunicorn déduit par instance (`~25–26 req/s`) ; écart à `4×` non mesuré | Fig. 2a |

## 4. Validité

1. **Instances ayant servi les mesures** : vérifié via l'en-tête `X-Instance` dans les CSV : `1` instance en partie 1 (`fig1a.csv`, `fig1b.csv`), `1` instance en partie 3 (`fig3.csv`), et `1`, `2` ou `4` instances selon `N` en partie 2 (`fig2a.csv`). C'est conforme aux attentes.
2. **Indépendance des runs et ordre** : l'ordre des runs est mélangé avec `shuf` dans `scripts/run_p1.sh` (ligne 22), `scripts/run_p2.sh` (ligne 11) et `scripts/run_p3.sh` (ligne 11), et enregistré dans `logs/p1_ordre.txt`, `logs/p2_n${N}_ordre.txt` et `logs/p3_ordre.txt`, avec pauses (`3 s` en P1/P3, `5 s` en P2) et échauffement (`15` requêtes en P1/P3, `80` requêtes en parallèle en P2). Toutefois, la partie 2 suit l'ordre `N = 1, 2, 4` et les parties s'enchaînent le 26 novembre 2026 (`p1_f05.csv` à `14:54`, `p2_n1_c1.csv` à `15:10`, `p2_n2_c1.csv` à `15:21`, `p2_n4_c1.csv` à `15:32`, `p3_f05.csv` à `16:30`), donc un effet d'ordre entre parties n'est pas exclu.
3. **Part du réseau** : estimée par `client_med_ms - server_med_ms`, autour de `11` à `17 ms` (jusqu'à `21,75 ms` à `F = 100` dans `fig1b.csv`).
4. **Échecs** : tout statut différent de `200` est compté comme échec. Seuls les statuts `200` et `0` (erreur/délai côté client Locust) ont été observés : `0` échec en parties 1 et 3 ; `1` à `15` échecs par fichier en partie 2 uniquement pour `C = 16, 32, 64` (au plus `15 / 1195 ≈ 1,26 %`), sans changer les tendances.
5. **Chemin de code exercé** : la première campagne de la partie 3 a produit des `404` pendant la suppression de `fixed1` ; ces 20 fichiers ont été supprimés, `fixed1` a été redéployée, et les 20 nouveaux fichiers ont `0` échec (`1` instance, statut `200`). La présence du correctif dans `main.py` (`grep -c "range(0, len(follows), 30)" main.py` renvoie `1`), la compilation (`py_compile`) et le saut à `F = 30` en partie 1 confirment le chemin emprunté.
6. **Coût de la campagne** : détaillé dans la section 5 (`5,54 $` d'usage brut, `0,00 $` net après `Other savings`).

## 5. Coût de la campagne

Le coût d'usage brut total est de **5,54 $**, entièrement compensé dans la colonne *Other savings* de la page Facturation pour un sous-total net de **0,00 $**.

| Ligne de facturation (SKU) | Consommation | Coût d'usage brut |
| :--- | :---: | :---: |
| Lectures Datastore (`Cloud Firestore Read Ops`) | `3 415 144` | `1,99 $` |
| Écritures Datastore (`Cloud Firestore Entity Writes`) | `1 876 835` | `3,27 $` |
| Suppressions Datastore (`Cloud Firestore Entity Deletes`) | `1 451 121` | `0,28 $` |
| Instances frontend (`Frontend Instances`, App Engine) | `4,96 heures` | `0,00 $` |
| Instances backend (`Backend Instances`, App Engine) | `7,5 heures` | `0,00 $` |
| Autres lignes (`Cloud Firestore Small Ops`, `Log Storage cost`, `Out Bandwidth`, `Build time e2-standard-2`, `Artifact Registry Storage`) | Voir rapport GCP | `0,00 $` |

Total brut : `1,99 $ + 3,27 $ + 0,28 $ = 5,54 $` (`0,00 $` net).
