# TinyInsta passe-t-elle à l'échelle ? Benchmark 2026

Projet Données massives et Cloud

Application déployée : https://monprojet-515813.ew.r.appspot.com

## 1. Montage

* Projet GCP `monprojet-515813`, région `europe-west1`, App Engine standard (`python312`) et Datastore
* Le client Locust tourne depuis Cloud Shell, chaque run dure environ 45 s
* Toutes les mesures ont été faites le 26/11/2026
* L'application est lancée avec `gunicorn -b :$PORT main:app` sans option `--workers` donc il y a 1 seul worker par instance
* Versions déployées :

| Version | Configuration | Usage |
|---|---|---|
| `20261125t141537` et `20261126t140850` (défaut) | F1, autoscaling jusqu'à 10 instances | version rendue |
| `fixed1` | B1, 1 instance, code d'origine | Partie 1 et Partie 2 (N=1) |
| `fixed2` | B1, 2 instances, code d'origine | Partie 2 (N=2) |
| `fixed4` | B1, 4 instances, code d'origine | Partie 2 (N=4) |
| `fixed1` (redéployée) | B1, 1 instance, code corrigé | Partie 3 |

* Code d'origine : `patches/main_original.py`, mon correctif : `patches/main_corrige.py` (copié dans `main.py`)
* Les journaux bruts de Locust sont dans `raw/` et l'ordre des runs dans `logs/`
* Pour refaire toutes les figures avec une seule commande : `bash scripts/make_figures.sh`

## 2. Résultats

### Partie 1 : coût d'une requête

**Hypothèse** : dans `get_timeline` le code lit l'utilisateur par sa clé puis fait une requête sur les `Post` avec un filtre `IN` sur les auteurs, triée par date et limitée à 20 posts
* U : la lecture par clé ne dépend pas du nombre d'utilisateurs donc le temps reste constant
* P : la requête s'arrête à 20 posts (index trié par auteur et date) donc le temps reste constant
* F : plus il y a d'auteurs dans le `IN` plus il y a de choses à fusionner donc je pense que le temps monte doucement

**Protocole** : version `fixed1` de 1 instance, 1 seul utilisateur Locust, un seul facteur change à la fois, on compare le temps serveur (en-tête `Server-Timing`) et la latence client

![Fig. 1a](figures/fig1a.png)

*Fig. 1a : `fixed1`, 1 utilisateur Locust, pour U : P=20 et F=20, pour P : U=100 et F=20*

![Fig. 1b](figures/fig1b.png)

*Fig. 1b : `fixed1`, 1 utilisateur Locust, U=120 et P=20*

| Facteur | Valeurs | Temps serveur médian (ms) |
|---|---|---|
| U (P=20, F=20) | 100 / 1000 / 10 000 | 27,2 / 24,5 / 25,2 |
| P (U=100, F=20) | 10 / 100 / 1000 | 27,5 / 26,4 / 26,2 |
| F (U=120, P=20) | 5 à 25 | 23,8 à 26,6 |
| F | 30 | 484,4 |
| F | 100 | 1611,4 |

**Interprétation**
* Mes hypothèses sont bonnes pour U et P : le temps ne change pas quand on multiplie U par 100 ou P par 100
* Pour F je n'avais pas vu le saut à F=30
* Le filtre `IN` de Datastore accepte 30 valeurs au maximum et la liste contient les comptes suivis plus l'utilisateur lui-même donc F+1 valeurs
* F=25 fait 26 valeurs donc ça marche, F=30 fait 31 valeurs donc ça ne marche plus
* Dans ce cas le code utilise son plan B : une requête par auteur, l'une après l'autre (pas en parallèle)
* Le calcul colle avec les mesures : 31 requêtes × 15,6 ms = 484 ms à F=30 et 101 requêtes × 16 ms = 1616 ms à F=100
* Réseau : latence client moins temps serveur donne environ 11 à 17 ms (jusqu'à 21,75 ms à F=100), c'est environ un tiers de la latence quand tout est rapide et seulement 1,3 % à F=100
* Entités lues par timeline (calculé à partir du code) : 21 pour F ≤ 25 (1 User + 20 posts), 621 pour F=30 (1 + 31×20) et 2021 pour F=100 (1 + 101×20)
* Prix d'après ma facture : 3,43 $ pour 5 872 549 lectures, soit environ 0,0584 $ pour 100 000 lectures 
* Coût de 1 million de timelines : environ 12 $ pour F ≤ 25, 363 $ pour F=30 et 1 180 $ pour F=100

### Partie 2 : capacité

**Hypothèse** : gunicorn est lancé sans `--workers` donc il y a un seul worker et une instance traite une requête à la fois
* Avec environ 40 ms par requête une instance fait environ 1 / 0,040 = 25 req/s
* Avec N instances je m'attends à environ N fois plus
* La saturation doit arriver dès que C dépasse N

**Protocole** : versions `fixed1`, `fixed2`, `fixed4`, jeu `base`, C de 1 à 64, une version à la fois, j'ai vérifié avec l'en-tête `X-Instance` que le nombre d'instances était bien N

![Fig. 2a](figures/fig2a.png)

*Fig. 2a : jeu `base` (U=1000, P=20, F=20), une courbe par valeur de N*

![Fig. 2b](figures/fig2b.png)

*Fig. 2b : jeu `base`, latence médiane mesurée et prédiction de Little (C / débit)*

| N | Débit maximum (req/s) | Par rapport à N=1 |
|---|---|---|
| 1 | 25,90 (déjà 22,83 à C=1) | ×1 |
| 2 | 52,54 | ×2,03 |
| 4 | 84,08 | ×3,25 |

**Interprétation**
* Une instance sature vers 25 à 26 req/s dès C=1 ou 2 car elle traite une seule requête à la fois, c'est ce qui limite la capacité
* La saturation arrive vers C=4 à 16 pour N=2 et N=4
* Loi de Little : latence ≈ C / débit, elle marche bien à faible charge (N=1, C=1 : 40,0 ms mesuré et 43,8 ms prévu, C=16 : 596,7 ms et 617,9 ms)
* Elle marche moins bien après la saturation (N=1, C=64 : 2235,3 ms mesuré et 2572,1 ms prévu, N=2, C=64 : 641,0 ms et 1308,1 ms)
* Je pense que c'est parce que je compare une médiane avec une moyenne tirée vers le haut par quelques requêtes très lentes mais je ne l'ai pas vérifié
* Passer de 1 à 4 instances ne multiplie pas le débit par 4 mais par 3,25
* Avec N=4 et C=32 les 4 instances ont servi 1068, 921, 869 et 843 requêtes en 45 s : la plus chargée fait environ 23,7 req/s (le maximum d'une instance seule) et les autres vers 19 à 20 req/s
* Avec C=4 la répartition est égale (788, 773, 771, 748) mais chaque instance fait seulement environ 17 req/s car elle n'a pas toujours une requête à traiter
* Je pense que la mauvaise répartition de la charge entre les instances est le goulot mais je n'ai pas testé d'autre cause

### Partie 3 : corriger et re-mesurer

**Goulot choisi** : le filtre `IN` limité à 30 valeurs (Partie 1)

**Hypothèse** : si je coupe la liste des auteurs en lots de 30 on ne fait plus F+1 requêtes mais ⌈(F+1)/30⌉ requêtes
* Le temps doit donc monter par paliers : 2 lots à F=30, 3 lots à F=60 et 4 lots à F=90
* Chaque lot devrait coûter environ 15 à 20 ms

**Correction** : `patches/main_corrige.py` fait une requête `IN` (limitée à 20 posts) par lot de 30 auteurs (`range(0, len(follows), 30)`) puis garde les 20 posts les plus récents de tous les lots

**Protocole** : version `fixed1` avec le code corrigé, même protocole que la Partie 1 (jeux `f05` à `f100`, `-u 1`)

![Fig. 3](figures/fig3.png)

*Fig. 3 : avant et après, `fixed1` avec le code corrigé, 1 utilisateur Locust, U=120 et P=20*

| F | Temps serveur avant (ms) | Temps serveur après (ms) |
|---|---|---|
| 25 | 26,6 | 32,25 |
| 30 | 484,4 | 50,35 |
| 60 | 984,7 | 74,6 |
| 100 | 1611,4 | 97,5 |

**Interprétation**
* Hypothèse confirmée : on voit des paliers vers 50 ms (F=30 à 55), vers 68 à 75 ms (F=60 à 85) et vers 86,5 à 97,5 ms (F=90 à 100)
* Chaque lot en plus ajoute environ 20 ms
* À F=100 c'est environ 16,5 fois plus rapide
* Pour F ≤ 25 le temps est un peu plus grand après correction, je n'ai pas cherché pourquoi
* Coût en complexité : le code est un peu plus compliqué (une boucle et un tri en mémoire)
* Coût en argent : à F=100 on lit environ 81 entités (1 User + 4 lots × 20) donc environ 47 $ pour 1 million de timelines au lieu de 1 180 $ (calculé à partir du code, pas mesuré)
* Cohérence des données : pas de problème car on lit les mêmes données et on garde bien les 20 posts les plus récents

## 3. Tableau des verdicts

| Dimension | Passe à l'échelle ? | Jusqu'où ? | Pourquoi (mécanisme) | Preuve |
|---|---|---|---|---|
| Utilisateurs (U) | Oui | Testé jusqu'à 10 000 | Lecture par clé et limite à 20 posts | Fig. 1a |
| Posts par utilisateur (P) | Oui | Testé jusqu'à 1000 | Index trié et limite à 20 posts | Fig. 1a |
| Followees (F), code d'origine | Non | Constant jusqu'à F=25 | `IN` limité à 30 valeurs puis une requête par auteur, l'une après l'autre | Fig. 1b |
| Followees (F), code corrigé | Mieux mais le temps monte encore par paliers | Testé jusqu'à F=100 | Une requête par lot de 30 auteurs | Fig. 3 |
| Instances (N) | En partie | N=2 : ×2,0 et N=4 : ×3,25 | 1 worker gunicorn par instance (environ 25 req/s) et charge mal répartie entre les instances | Fig. 2a |

## 4. Validité

1. **Instances** : oui ce sont celles que j'attendais (1 en Partie 1 et 3, puis 1, 2 ou 4 en Partie 2), je l'ai vérifié avec l'en-tête `X-Instance` et pas avec ce que dit `gcloud`
2. **Indépendance des runs** : l'ordre des runs est mélangé avec `shuf` dans chaque partie et gardé dans `logs/`, avec des pauses de 3 à 5 s et un échauffement avant les runs, mais les versions de la Partie 2 ont été faites dans l'ordre N=1, 2, 4 et les trois parties se sont suivies le même jour donc je ne peux pas exclure un effet de l'ordre
3. **Réseau** : environ 11 à 17 ms (latence client moins temps serveur), jusqu'à 21,75 ms pour F=100
4. **Échecs** : j'ai compté comme échec tout statut différent de 200 et j'ai vu deux sortes de problèmes
   * Pendant les mesures : j'ai vu 200 et le statut 0 qui n'est pas un code HTTP mais une erreur réseau côté client avant toute réponse (`gaierror` ou `ConnectionRefusedError`), seulement pour C=16, 32 et 64 (de 1 à 15 par fichier soit moins de 1,3 %), aucune réponse de l'application n'est en erreur (pas de 5xx) et je n'ai pas vérifié la cause
   * Pendant la création des jeux de données : l'écriture avec `bench/seed_exact.py` (par exemple le jeu de U=1000) restait bloquée longtemps puis la connexion se coupait, ou j'avais le message de temps limite dépassé avec un nouvel essai au bout de 2 s qui se répétait jusqu'à l'erreur, dans ce cas le jeu était écrit à moitié donc je le purgeais avec `--purge` et je recommençais tout (ça explique en partie les écritures et suppressions de la facture)
5. **Chemin de code** : oui, le saut à F=30 en Partie 1 montre que le plan B est bien utilisé, et ma première campagne de la Partie 3 donnait des 404 car `fixed1` était en cours de suppression donc je mesurais la plateforme et pas mon code, je l'ai supprimée, j'ai redéployé `fixed1` et refait la Partie 3 (statut 200, 1 instance, 0 échec)
6. **Coût** : voir section 5

## 5. Coût de la campagne

D'après la page Facturation du 1er au 27 novembre 2026 : 7,96 $ brut

| Ligne (SKU) | Quantité | Coût brut |
|---|---|---|
| Lectures Datastore | 5 872 549 | 3,43 $ |
| Écritures Datastore | 2 426 438 | 4,22 $ |
| Suppressions Datastore | 1 605 821 | 0,31 $ |
| Petites opérations  | 2 870 680 | 0,00 $ |
| Instances frontend | 5,41 h | 0,00 $ |
| Instances backend | 11,19 h | 0,00 $ |
| Autres (logs, bande passante, build, Artifact Registry) | 0,23 Gio, 0,15 Gio, 11 min, 0,53 Gio/mois | 0,00 $ |
