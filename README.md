# TinyInsta passe-t-elle à l'échelle ? Benchmark 2026

Question du projet : qu'est-ce qui passe à l'échelle, jusqu'où, et pourquoi ?

## 1. Montage

* Appli déployée : https://monprojet-515813.ew.r.appspot.com
* Projet `monprojet-515813`, région `europe-west1`, App Engine standard (`python312`) et Datastore
* L'application est lancée avec `gunicorn -b :$PORT main:app`, sans option `--workers`, donc il y a 1 seul worker par instance (c'est la valeur par défaut de gunicorn)
* Version `default` (`app.yaml`) : classe `F1`, autoscaling jusqu'à 10 instances (versions `20261125t141537` et `20261126t140850`)
* Versions `fixedN` (`deploy_fixed.sh`) : classe `B1`, manual scaling avec `N` instances (`fixed1`, `fixed2`, `fixed4`), déployées avec `--no-promote` et arrêtées après les mesures
* Le client Locust tourne depuis Cloud Shell, chaque run dure environ 45 s
* Toutes les mesures ont été faites le 26 novembre 2026
* Jeux de données (créés avec `bench/seed_exact.py`) : `base`, `u100`, `u10000`, `p10`, `p100`, `p1000` et `f05` à `f100`
* Dans le dépôt : 67 fichiers CSV dans `raw/` (26 pour la partie 1, 21 pour la partie 2, 20 pour la partie 3), les figures et leurs CSV dans `figures/`, les scripts dans `scripts/`
* `patches/main_original.py` est le code du prof (parties 1 et 2), `patches/main_corrige.py` est mon correctif (partie 3), et `main.py` contient le code corrigé
* Pour refaire toutes les figures avec une seule commande : `bash scripts/make_figures.sh`

## 2. Résultats

### Partie 1 : coût d'une requête

* **Hypothèse** : en lisant `get_timeline`, je vois que le code lit l'utilisateur avec sa clé, puis fait une requête sur les `Post` avec un filtre `IN`, triée par date et limitée à 20 posts. Je pense donc que le temps reste constant quand `U` et `P` augmentent, et qu'il monte doucement quand `F` augmente (plus d'auteurs à chercher)
* **Protocole** : version `fixed1` (1 instance), 1 seul utilisateur Locust (`-u 1`), un seul facteur change à la fois

![Fig. 1a](figures/fig1a.png)

Paramètres fixés : `fixed1` (1 instance), 1 utilisateur Locust. Pour `U` : `P = 20` et `F = 20`. Pour `P` : `U = 100` et `F = 20`

![Fig. 1b](figures/fig1b.png)

Paramètres fixés : `fixed1` (1 instance), 1 utilisateur Locust, `U = 120`, `P = 20`

* **Résultats** :
  * `U` (`100`, `1000`, `10000`) : `27,2 ms`, `24,5 ms`, `25,2 ms`, donc constant
  * `P` (`10`, `100`, `1000`) : `27,5 ms`, `26,4 ms`, `26,2 ms`, donc constant
  * `F` de `5` à `25` : de `23,8 ms` à `26,6 ms`, presque constant
  * `F = 30` : le temps saute à `484,4 ms`, puis il monte jusqu'à `1611,4 ms` pour `F = 100`
* **Interprétation** : l'hypothèse est bonne pour `U` et `P`. Pour `F`, je ne m'attendais pas au saut à `F = 30`. Il vient du filtre `IN`, qui accepte 30 valeurs au maximum. La liste contient les comptes suivis et l'utilisateur lui-même, donc `F + 1` valeurs : `F = 25` fait 26 valeurs (ça marche), `F = 30` fait 31 valeurs (ça ne marche plus). Le code utilise alors son plan B : une requête par auteur, l'une après l'autre. Chaque requête prend environ `15,6 ms` à `F = 30` et `16 ms` à `F = 100`, ce qui est cohérent
* **Réseau** (latence client moins temps serveur) : environ `11` à `17 ms` (jusqu'à `21,75 ms` pour `F = 100`). C'est environ un tiers de la latence quand tout est rapide, et seulement `1,3 %` pour `F = 100`
* **Entités lues et coût pour 1 million de timelines** : c'est un calcul à partir du code et de ma facture (`1,99 $` pour `3 415 144` lectures, donc environ `0,0583 $` pour 100 000 lectures). Je n'ai pas vérifié la grille de prix officielle
  * `U` et `P` quelconques, `F <= 25` : `21` entités (`1` User + `20` posts) donc `~12 $`
  * `F = 30` (code d'origine) : `621` entités (`1` User + `31 × 20`) donc `~362 $`
  * `F = 100` (code d'origine) : `2021` entités (`1` User + `101 × 20`) donc `~1 178 $`

### Partie 2 : capacité

* **Hypothèse** : avec 1 worker gunicorn par instance et environ 40 ms par requête, une instance devrait faire environ 25 requêtes par seconde. Avec `N` instances, je m'attends à environ `N` fois plus, et la saturation devrait arriver quand `C` dépasse `N`
* **Protocole** : versions `fixed1`, `fixed2`, `fixed4`, jeu `base`, `C` de `1` à `64`

![Fig. 2a](figures/fig2a.png)

![Fig. 2b](figures/fig2b.png)

Paramètres fixés : jeu `base` (`U = 1000`, `P = 20`, `F = 20`), `N = 1`, `2` ou `4` instances

* **Résultats** :
  * Débit maximum : `25,90 req/s` pour `N = 1` (déjà `22,83` à `C = 1`), `52,54 req/s` pour `N = 2`, `84,08 req/s` pour `N = 4`
  * Loi de Little (latence ≈ `C` / débit) : elle marche bien quand la charge est faible ou moyenne (`N = 1, C = 1` : `40,0 ms` mesuré contre `43,8 ms` prévu ; `C = 16` : `596,7 ms` contre `617,9 ms`)
  * Au-delà de la saturation, elle marche moins bien : `N = 1, C = 64` donne `2235,3 ms` contre `2572,1 ms` prévu, et `N = 2, C = 64` donne `641,0 ms` contre `1308,1 ms` (l'écart est plus grand avec `N = 2`)
* **Interprétation** : une instance plafonne vers `25` à `26 req/s`. La saturation arrive vers `C = 1–2` pour `N = 1`, et vers `C = 4–16` pour `N = 2` et `N = 4`. De 1 à 2 instances, le débit est multiplié par environ `2,03`, mais de 1 à 4 instances seulement par environ `3,25`, donc ça ne passe pas tout à fait à l'échelle. Pour l'écart avec la loi de Little, je pense que c'est parce que je compare la médiane (qui ne bouge pas beaucoup) avec une moyenne tirée vers le haut par quelques requêtes très lentes, mais je ne l'ai pas vérifié
* **Ce qui limite une instance** : gunicorn n'a qu'un worker, donc l'instance traite une requête à la fois. Le débit maximum est environ `1 / latence` (`1 / 0,040 s = 25 req/s`)
* **Pourquoi pas `4×`** : avec `N = 4` et `C = 32`, les 4 instances ont servi `1068`, `921`, `869` et `843` requêtes en 45 s. La plus chargée fait environ `23,7 req/s`, proche du maximum d'une instance seule, les autres sont vers `19–20 req/s`. Avec `C = 4`, la répartition est égale (`788`, `773`, `771`, `748`) mais chaque instance fait seulement environ `17 req/s`. Je pense que la mauvaise répartition de la charge explique une partie de l'écart, mais je n'ai pas testé d'autre cause

### Partie 3 : corriger et re-mesurer

* **Hypothèse** : si je coupe la liste `follows` en lots de 30 auteurs (`range(0, len(follows), 30)`), on ne fait plus une requête par auteur. Le temps devrait augmenter par paliers, à `F = 30`, `60` et `90` (quand on passe à un lot de plus)
* **Correction** : `patches/main_corrige.py` (copié dans `main.py`) fait une requête `IN` (limitée à 20 posts) par lot de 30 auteurs, puis garde les 20 posts les plus récents de tous les lots
* **Protocole** : version `fixed1` avec le code corrigé, même protocole que la partie 1 (`f05` à `f100`)

![Fig. 3](figures/fig3.png)

Paramètres fixés : `fixed1` avec le code corrigé, 1 utilisateur Locust, `U = 120`, `P = 20`

* **Résultats** (temps serveur médian, avant et après) :
  * `F = 25` : `26,6 ms` et `32,25 ms`
  * `F = 30` : `484,4 ms` et `50,35 ms`
  * `F = 60` : `984,7 ms` et `74,6 ms`
  * `F = 100` : `1611,4 ms` et `97,5 ms`, soit environ `16,5` fois plus rapide
* **Interprétation** : l'hypothèse est confirmée. On voit des paliers vers `50 ms` (`F = 30` à `55`), vers `68–75 ms` (`F = 60` à `85`) et vers `86,5–97,5 ms` (`F = 90` à `100`). Je n'ai pas cherché pourquoi le temps est un peu plus grand pour `F <= 25`
* **Ce que ça coûte** : le code est un peu plus compliqué (une boucle et un tri en mémoire). Pour `F = 100`, on lit environ `81` entités (`1` User + `4 × 20`), soit `~47 $` pour 1 million de timelines au lieu de `1 178 $` (calcul à partir du code, pas mesuré). Les données restent cohérentes : on garde bien les 20 posts les plus récents

## 3. Tableau des verdicts

| Dimension | Passe à l'échelle ? | Jusqu'où ? | Pourquoi (mécanisme) | Preuve |
| :--- | :--- | :--- | :--- | :--- |
| Utilisateurs (`U`) | Oui | Testé jusqu'à `10 000` | Lecture par clé et limite à 20 posts | Fig. 1a |
| Posts par utilisateur (`P`) | Oui | Testé jusqu'à `1 000` | Limite à 20 posts | Fig. 1a |
| Followees (`F`), code d'origine | Non | Jusqu'à `F = 25` | Le filtre `IN` accepte 30 valeurs, ensuite une requête par auteur | Fig. 1b |
| Followees (`F`), code corrigé | Mieux, mais le temps augmente encore | Testé jusqu'à `F = 100` | Une requête par lot de 30 auteurs | Fig. 3 |
| Instances (`N`) | En partie | De 1 à 2 : `~2,0×`. De 1 à 4 : `~3,25×` | 1 worker gunicorn par instance (`~25–26 req/s`), charge mal répartie entre les instances | Fig. 2a |

## 4. Validité

1. **Instances** : oui, ce sont celles que j'attendais. Il y avait `1` instance en partie 1 et en partie 3, et `1`, `2` ou `4` selon `N` en partie 2. Je l'ai vérifié avec l'en-tête `X-Instance`, pas avec ce que dit `gcloud`
2. **Ordre des runs** : dans chaque partie, l'ordre des runs est mélangé avec `shuf` (`run_p1.sh`, `run_p2.sh`, `run_p3.sh`) et il est gardé dans `logs/*_ordre.txt`. Il y a des pauses (`3 s` ou `5 s`) et un échauffement avant les runs. Par contre, les versions de la partie 2 ont été faites dans l'ordre `N = 1, 2, 4`, et les trois parties se sont suivies le même jour. Je ne peux donc pas exclure un effet de l'ordre entre les parties
3. **Réseau** : environ `11` à `17 ms` (jusqu'à `21,75 ms` pour `F = 100`), calculé avec latence client moins temps serveur
4. **Échecs** : j'ai compté comme échec tout statut différent de `200`. J'ai vu `200` et le statut `0`, qui n'est pas un code HTTP : c'est une erreur réseau côté client, avant toute réponse. La colonne `ERROR` montre deux types : `gaierror` (nom de domaine non résolu) et `ConnectionRefusedError` (connexion refusée). Elles durent en général `10` à `15 s` avant d'échouer. Elles arrivent seulement pour `C = 16, 32, 64`, de `1` à `15` par fichier (moins de `1,3 %`). Aucune réponse de l'application n'est en erreur (pas de `5xx`). Je n'ai pas vérifié la cause (limite réseau de Cloud Shell ou de la plateforme ?). Il y a eu aussi des `404` dans ma première campagne de la partie 3, que j'ai supprimée et refaite : les 20 fichiers finaux ont `0` échec
5. **Chemin de code** : oui, je l'ai vérifié. `grep -c` renvoie `1`, `py_compile` ne montre pas d'erreur, et le saut à `F = 30` en partie 1 montre que le plan B est bien utilisé. La première campagne de la partie 3 donnait des `404` parce que `fixed1` était en cours de suppression : je mesurais la plateforme, pas mon code. J'ai supprimé ces fichiers, redéployé `fixed1` et refait la partie 3 (`200`, 1 instance, `0` échec)
6. **Coût** : voir la section 5 (`5,54 $` brut, `0,00 $` net)

## 5. Coût de la campagne

Total brut : **5,54 $**, ramené à **0,00 $** net grâce à *Other savings*

| Ligne (SKU) | Quantité | Coût brut |
| :--- | :---: | :---: |
| Lectures Datastore (`Cloud Firestore Read Ops`) | `3 415 144` | `1,99 $` |
| Écritures Datastore (`Cloud Firestore Entity Writes`) | `1 876 835` | `3,27 $` |
| Suppressions Datastore (`Cloud Firestore Entity Deletes`) | `1 451 121` | `0,28 $` |
| Instances frontend (`Frontend Instances`) | `4,96 h` | `0,00 $` |
| Instances backend (`Backend Instances`) | `7,5 h` | `0,00 $` |
