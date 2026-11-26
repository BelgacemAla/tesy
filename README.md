# TinyInsta passe-t-elle à l'échelle ? — Benchmark 2026

Question du projet : qu'est-ce qui passe à l'échelle, jusqu'où, et pourquoi ?

## 1. Montage

* Projet `monprojet-515813` en `europe-west1`, sur App Engine standard (`python312`) et Datastore
* Lancement avec `gunicorn -b :$PORT main:app` (sans `--workers` je suppose 1 seul worker par instance, sans l'avoir vérifié dans les logs)
* Version `default` (`app.yaml`) : classe `F1` autoscaling jusqu'à 10 instances (`20261125t141537` et `20261126t140850`)
* Versions `fixedN` (`deploy_fixed.sh`) : classe `B1`, manual scaling à `N` instances (`fixed1`, `fixed2`, `fixed4`), déployées en `--no-promote` et arrêtées après les tests
* Client Locust lancé depuis Cloud Shell, runs d'environ 45 s, mesures faites le 26 novembre 2026 avec les jeux `base`, `u100`, `u10000`, `p10`, `p100`, `p1000` et `f05` à `f100` (`bench/seed_exact.py`)
* Dépôt : 67 CSV dans `raw/` (26 en P1, 21 en P2, 20 en P3), graphiques/CSV dans `figures/`, scripts dans `scripts/`, `patches/main_original.py` et `patches/main_corrige.py` (`main.py` contient le code corrigé)
* Commande pour refaire toutes les figures : `bash scripts/make_figures.sh`

## 2. Résultats

### Partie 1 : coût d'une requête

* **Hypothèse** : `get_timeline` lit l'utilisateur par clé puis fait une requête `Post` avec `IN` triée par date et limitée à 20 posts. Donc le temps devrait rester constant pour `U` et `P`, et monter doucement avec `F`
* **Protocole** : `fixed1` (1 instance), 1 utilisateur Locust (`-u 1`), un seul facteur varié à la fois

![Fig. 1a](figures/fig1a.png)

![Fig. 1b](figures/fig1b.png)

* **Résultats** :
  * `U` (`100`, `1000`, `10000`) : `27,2 ms`, `24,5 ms`, `25,2 ms` (constant)
  * `P` (`10`, `100`, `1000`) : `27,5 ms`, `26,4 ms`, `26,2 ms` (constant)
  * `F` de `5` à `25` : `23,8 ms` à `26,6 ms` (presque constant)
  * `F = 30` : saut à `484,4 ms`, puis montée jusqu'à `1611,4 ms` à `F = 100`
* **Interprétation** : hypothèse ok pour `U` et `P`, mais pas pour `F` à cause du saut à `F = 30`  Le filtre `IN` accepte 30 valeurs max : avec `F + 1` auteurs (les suivis + l'utilisateur), `F = 25` fait 26 valeurs et `F = 30` fait 31 valeurs. Le code passe alors sur une requête par auteur l'une après l'autre (`~15,6 ms` par auteur à `F = 30`, `~16 ms` à `F = 100`)
* **Réseau** (`client - serveur`) : environ `11` à `17 ms` (jusqu'à `21,75 ms` à `F = 100`), soit ~1/3 quand c'est rapide et ~1,3 % à `F = 100`
* **Entités lues et coût pour 1 million de timelines** (calcul depuis le code et ma facture à `1,99 $` pour `3 415 144` lectures, soit `~0,0583 $` / 100k lectures ; grille officielle à vérifier) :
  * `U`, `P` quelconques, `F <= 25` : `21` entités (`1` User + `20` posts max) → `~12 $`
  * `F = 30` (code d'origine) : `621` entités (`1` User + `31 × 20`) → `~362 $`
  * `F = 100` (code d'origine) : `2021` entités (`1` User + `101 × 20`) → `~1 178 $`

### Partie 2 : capacité

* **Hypothèse** : avec 1 worker Gunicorn par instance et ~40 ms par requête, une instance devrait faire ~25 req/s, et `N` instances ~`N` fois plus, avec saturation dès que `C` dépasse `N`
* **Protocole** : `fixed1`, `fixed2`, `fixed4`, jeu `base`, `C` de `1` à `64`

![Fig. 2a](figures/fig2a.png)

![Fig. 2b](figures/fig2b.png)

* **Résultats** :
  * Débit max : `25,90 req/s` (`N = 1`, déjà `22,83` à `C = 1`), `52,54 req/s` (`N = 2`), `84,08 req/s` (`N = 4`)
  * Loi de Little (`latence ≈ C / débit`) : proche à faible/moyenne charge (`N = 1, C = 1` : `40,0 ms` vs `43,8 ms` ; `C = 16` : `596,7 ms` vs `617,9 ms`), mais loin au-delà (`N = 2, C = 64` : `641,0 ms` vs `1308,1 ms`)
* **Interprétation** : une instance plafonne vers `25–26 req/s` (saturation vers `C = 1–2` pour `N = 1`, `C = 4–16` pour `N = 2` et `N = 4`). De `1` à `2` instances on fait `~2,03×`, mais de `1` à `4` on fait seulement `~3,25×` (sous-linéaire ; je n'ai pas mesuré pourquoi). Pour l'écart à la loi de Little à forte charge, c'est peut-être parce qu'on compare la médiane à la moyenne (tirée vers le haut par des requêtes lentes), sans l'avoir vérifié

### Partie 3 : corriger et re-mesurer

* **Hypothèse** : découper `follows` en lots de 30 auteurs (`range(0, len(follows), 30)`) évite une requête par auteur et fait des paliers vers `F = 30`, `60` et `90`
* **Correction et protocole** : `patches/main_corrige.py` (dans `main.py`) fait une requête `IN` (limitée à 20 posts) par lot de 30 auteurs, puis garde les 20 posts les plus récents. Déployé sur `fixed1`, même protocole que la partie 1 (`f05` à `f100`)

![Fig. 3](figures/fig3.png)

* **Résultats (temps serveur médian avant / après)** :
  * `F = 25` : `26,6 ms` / `32,25 ms`
  * `F = 30` : `484,4 ms` / `50,35 ms`
  * `F = 60` : `984,7 ms` / `74,6 ms`
  * `F = 100` : `1611,4 ms` / `97,5 ms` (~16,5× plus rapide)
* **Interprétation et coûts** : hypothèse confirmée, avec des marches vers `~50 ms` (`F = 30..55`), `~68–75 ms` (`F = 60..85`) et `~86,5–97,5 ms` (`F = 90..100`) ; je n'ai pas creusé le petit écart à `F <= 25`. Côté coûts : plus de complexité (boucle + tri en mémoire), mais à `F = 100` on passe théoriquement à ~`81` entités lues (`1` User + `4 × 20`) soit `~47 $` pour 1 million de timelines au lieu de `1 178 $`, et le résultat reste le même (top 20 global des plus récents)

## 3. Tableau des verdicts

| Dimension | Passe à l'échelle ? | Jusqu'où ? | Pourquoi (mécanisme) | Preuve |
| :--- | :--- | :--- | :--- | :--- |
| Utilisateurs (`U`) | Oui | Testé jusqu'à `10 000` | Lecture par clé + limite à 20 posts | Fig. 1a |
| Posts par utilisateur (`P`) | Oui | Testé jusqu'à `1 000` | Limite à 20 posts | Fig. 1a |
| Followees (`F`), code d'origine | Non | Jusqu'à `F = 25` | Filtre `IN` limité à 30 valeurs, puis 1 requête par auteur | Fig. 1b |
| Followees (`F`), code corrigé | Mieux, mais pas constant | Testé jusqu'à `F = 100` | Requêtes par lots de 30 auteurs | Fig. 3 |
| Instances (`N`) | En partie | `1 -> 2` : `~2,0×` ; `1 -> 4` : `~3,25×` | 1 worker Gunicorn déduit par instance (~25–26 req/s) ; cause de l'écart à `4×` non mesurée | Fig. 2a |

## 4. Validité

1. **Instances** : oui, c'est bien ce qu'on attendait (`1` instance en P1 et P3, et `1`, `2` ou `4` selon `N` en P2), vérifié avec `X-Instance` plutôt qu'à partir de ce que dit `gcloud`
2. **Ordre des runs** : les runs sont mélangés avec `shuf` dans `run_p1.sh`, `run_p2.sh` et `run_p3.sh` (gardé dans `logs/*_ordre.txt`), avec des pauses (`3 s` ou `5 s`) et un échauffement. Par contre, `N` a été fait dans l'ordre `1, 2, 4` et les parties se sont suivies le 26 novembre 2026, donc un effet d'ordre entre parties n'est pas exclu
3. **Réseau** : environ `11` à `17 ms` (jusqu'à `21,75 ms` à `F = 100`), calculé avec `client - serveur`
4. **Échecs** : tout statut différent de `200` est compté en échec ; on a vu `200` et `0` (erreur/délai côté client Locust). `0` échec en P1 et P3 ; `1` à `15` échecs en P2 pour `C = 16, 32, 64` (< 1,3 %), ce qui ne change pas les tendances
5. **Chemin de code** : oui, vérifié : le `grep -c` (`1`), `py_compile` et le saut à `F = 30` confirment le code, et la première passe P3 en `404` (due à la suppression de `fixed1`) a été supprimée et refaite sur `fixed1` redéployée avec `0` échec (`200`, 1 instance).
6. **Coût** : voir section 5 (`5,54 $` brut, `0,00 $` net)

## 5. Coût de la campagne

Total brut : **5,54 $**, ramené à **0,00 $** net grâce à *Other savings*

| Ligne (SKU) | Quantité | Coût brut |
| :--- | :---: | :---: |
| Lectures Datastore (`Cloud Firestore Read Ops`) | `3 415 144` | `1,99 $` |
| Écritures Datastore (`Cloud Firestore Entity Writes`) | `1 876 835` | `3,27 $` |
| Suppressions Datastore (`Cloud Firestore Entity Deletes`) | `1 451 121` | `0,28 $` |
| Instances frontend (`Frontend Instances`) | `4,96 h` | `0,00 $` |
| Instances backend (`Backend Instances`) | `7,5 h` | `0,00 $` |

