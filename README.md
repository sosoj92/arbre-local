# Arbre local : retrouver la commune de naissance d’une personne décédée

**Tapez le nom d'une personne décédée en France depuis 1970 : l'outil vous donne sa commune de naissance,
donc les archives où chercher son acte.** La recherche se fait entièrement dans votre navigateur, rien de
ce que vous tapez n'est envoyé à un serveur.

- **Source** : le fichier public des 29 millions de décès de l'INSEE (1970 à aujourd'hui).
- **Résultat** : la commune de naissance, et un bouton pour demander l'acte au bon service d'archives.
- **Remonter les générations** : l'outil propose des parents *possibles* (même nom, même commune, 20 à 40 ans plus tôt), à vérifier dans les actes.
- **Confidentialité** : pas de compte, pas de base de données, pas de serveur applicatif.

**Ce qu'il ne fait pas** : il ne prouve aucun lien de parenté (seul un acte le prouve), il ne voit personne
décédé avant 1970, et il ne lit pas les registres à votre place. Pour la suite de la recherche, voir le
[guide de recherche en archives](docs/guide-recherche.md).

Ce document explique comment construire l'outil soi-même, en 7 étapes, sans savoir programmer.

---

## Sommaire

- [Comment ça marche](#comment-ça-marche)
- [Prérequis](#prérequis)
- [Étape 1 — Récupérer les données](#étape-1--récupérer-les-données)
- [Étape 2 — Nettoyer et trier](#étape-2--nettoyer-et-trier)
- [Étape 3 — Partitionner](#étape-3--partitionner)
- [Étape 4 — Héberger](#étape-4--héberger)
- [Étape 5 — Générer le site](#étape-5--générer-le-site)
- [Étape 6 — Le bouton « demander l'acte »](#étape-6--le-bouton--demander-lacte-)
- [Étape 7 — Remonter les générations](#étape-7--remonter-les-générations)
- [Limites à connaître](#limites-à-connaître)
- [Aller plus loin](#aller-plus-loin)
- [Licences et attribution](#licences-et-attribution)
- [Remerciements](#remerciements)

---

## Comment ça marche

L'approche classique d'un site de recherche :

```
Utilisateur → tape un nom → serveur → base de données → résultats
```

Le nom quitte la machine de l'utilisateur.

L'approche retenue ici :

```
Utilisateur → tape un nom → moteur SQL dans SON navigateur
                              ↓ demande « octets 4 000 000 à 4 200 000 »
                            fichiers Parquet sur un CDN
```

Le serveur ne reçoit qu'un numéro de plage d'octets. Il ne sait pas ce qui est cherché.

Trois briques rendent ça possible :

| Brique | Rôle |
|---|---|
| **Parquet** | Format en colonnes, découpé en blocs indexés. Permet de ne lire qu'une partie d'un fichier. |
| **HTTP Range Requests** | Standard web permettant de demander une plage d'octets précise d'un fichier distant. |
| **DuckDB-WASM** | Moteur SQL compilé pour le navigateur. Il sait combiner les deux ci-dessus. |

Résultat : quelques Mo téléchargés par recherche au lieu de 579 Mo.

---

## Prérequis

- **DuckDB** (CLI) — un exécutable unique, aucune installation : <https://duckdb.org/docs/installation>
- **Un compte d'hébergement objet** — Cloudflare R2 (10 Go gratuits) ou équivalent S3
- **~2 Go d'espace disque** temporaire
- **~8 Go de RAM** pour l'étape de tri
- **Un générateur de site par IA** — Runable, Lovable, Bolt, v0, Claude… ou vos mains

Aucune connaissance en programmation n'est nécessaire. Il faut savoir copier-coller dans un terminal.

---

## Étape 1 — Récupérer les données

### Le fichier des décès

Source : **data.gouv.fr → « Agrégation des fichiers des personnes décédées »** (publié par data.gouv.fr à partir des fichiers de l'INSEE).

Deux formats proposés. **Prenez le Parquet** (~630 Mo), pas le CSV (3,3 Go) : c'est déjà le bon format.

Cette version agrégée est préférable aux fichiers annuels de l'INSEE, car elle ajoute deux colonnes précieuses :

- `fichier_origine` — de quel fichier source vient la ligne
- `opposition` — booléen indiquant si les proches se sont opposés à la rediffusion

> **Obligation légale** : les proches d'une personne décédée peuvent s'opposer à la rediffusion de ses données par des tiers. L'INSEE centralise ces demandes. Si vous republiez ces données, vous devez exclure ces lignes. La colonne `opposition` rend ça trivial — ne sautez pas ce filtre.

Renommez le fichier en `deces.parquet` et placez-le dans un dossier de travail (`projet-arbre/`).

> ⚠️ Évitez un dossier synchronisé (OneDrive, Dropbox, iCloud) : la synchronisation de plusieurs Go ralentira tout et saturera votre quota.

### L'annuaire des services d'archives

Source : **data.gouv.fr → « Portail FranceArchives — Annuaire des services d'archives publiques en France »** (~400 Ko, Licence Ouverte 2.0).

Contient les ~1 200 services d'archives français avec adresses, courriels, sites web et code INSEE de la commune. Enregistrez-le sous `annuaire-archives.csv`.

### Structure des données

```
nom                   VARCHAR   -- MAJUSCULES
prenoms               VARCHAR   -- séparés par des virgules
sexe                  VARCHAR   -- M / F
date_naissance        VARCHAR   -- AAAAMMJJ
code_insee_naissance  VARCHAR   -- code officiel géographique
commune_naissance     VARCHAR   -- en clair
pays_naissance        VARCHAR   -- si né à l'étranger
date_deces            VARCHAR   -- AAAAMMJJ
code_insee_deces      VARCHAR   -- code uniquement, pas de libellé
numero_acte_deces     VARCHAR
fichier_origine       VARCHAR
opposition            BOOLEAN
```

---

## Étape 2 — Nettoyer et trier

Ouvrez un terminal dans votre dossier de travail, lancez `duckdb`, puis :

```sql
-- Inspection préalable
DESCRIBE SELECT * FROM 'deces.parquet';
SELECT count(*) FROM 'deces.parquet';
```

### Filtrage

```sql
CREATE TABLE base AS
SELECT nom, prenoms, sexe, date_naissance, code_insee_naissance,
       commune_naissance, pays_naissance, date_deces, code_insee_deces,
       numero_acte_deces
FROM 'deces.parquet'
WHERE opposition = false
  AND nom IS NOT NULL AND nom <> ''
  AND date_naissance ~ '^(18|19|20)[0-9]{6}$';
```

Ce filtre écarte : les oppositions à rediffusion, les lignes sans nom, et les dates de naissance impossibles (le fichier contient historiquement des fautes de frappe type `91xx` au lieu de `19xx`).

Sur les données de mi-2026 : 29 253 002 lignes en entrée → 29 249 413 conservées. Le fichier est nettement plus propre que sa réputation.

### Tri et export

```sql
COPY (SELECT * FROM base ORDER BY nom, prenoms)
TO 'deces_trie.parquet'
(FORMAT PARQUET, COMPRESSION ZSTD, ROW_GROUP_SIZE 100000);
```

**Le tri est l'étape la plus importante du projet.** Sans lui, les homonymes sont dispersés dans tout le fichier et le navigateur doit lire des dizaines de blocs par recherche. Avec lui, tous les « DUPONT » sont contigus : un ou deux blocs suffisent.

`ROW_GROUP_SIZE 100000` définit la granularité de lecture. Trop petit, les métadonnées gonflent ; trop grand, on télécharge inutilement. 100 000 lignes est un bon compromis.

Comptez plusieurs minutes et une consommation mémoire importante. Si vous obtenez une erreur de mémoire, triez par tranches de première lettre plutôt qu'en une passe.

Vérifiez la taille obtenue :

```sql
SELECT round(sum(total_compressed_size)/1024/1024, 1) AS taille_Mo
FROM parquet_metadata('deces_trie.parquet');
```

Attendu : environ **579 Mo**.

---

## Étape 3 — Partitionner

Deux raisons de découper le fichier : la plupart des interfaces d'upload refusent les fichiers de plus de 300 Mo, et surtout le navigateur n'aura à connaître qu'un seul fichier par recherche.

```sql
COPY (
  SELECT *,
    CASE WHEN regexp_matches(nom, '^[A-Z]')
         THEN substr(nom, 1, 1)
         ELSE 'AUTRE' END AS lettre
  FROM 'deces_trie.parquet'
)
TO 'parts'
(FORMAT PARQUET, PARTITION_BY (lettre), COMPRESSION ZSTD, ROW_GROUP_SIZE 100000);
```

Produit une arborescence :

```
parts/
├── lettre=A/data_0.parquet
├── lettre=B/data_0.parquet
├── ...
├── lettre=Z/data_0.parquet
└── lettre=AUTRE/data_0.parquet
```

La partition `AUTRE` récupère les noms commençant par un caractère non alphabétique — apostrophes des noms néerlandais (`'T LAM`), tirets, noms non latins. Sans elle, ces personnes seraient introuvables.

### Répartition constatée

| Lettre | Lignes | Taille approx. |
|---|---|---|
| B | 3 612 181 | 79 Mo |
| L | 2 848 548 | ~62 Mo |
| C | 2 711 450 | ~59 Mo |
| D | 2 643 637 | ~58 Mo |
| M | 2 614 621 | ~57 Mo |

Toutes les partitions passent sous les 300 Mo. Si ce n'était pas le cas, il faudrait partitionner sur deux caractères (`MA`, `MB`…).

### Renommage groupé

Les fichiers générés s'appellent `data_0.parquet`. Pour uniformiser les URL :

**Windows** (invite de commandes, pas DuckDB) :
```bat
for /d %d in (parts\lettre=*) do @ren "%d\data_0.parquet" data.parquet
```

**macOS / Linux** :
```bash
for d in parts/lettre=*/; do mv "$d"data_0.parquet "$d"data.parquet; done
```

---

## Étape 4 — Héberger

N'importe quel stockage objet supportant les **requêtes de plage** (`Range`) et le **CORS** convient : Cloudflare R2, AWS S3, Backblaze B2, Scaleway, Hugging Face Datasets.

Exemple avec **Cloudflare R2** (10 Go gratuits) :

1. **Créer un bucket** — nom en minuscules, sans espace
2. **Settings → Public Development URL → Enable** — vous obtenez une URL du type `https://pub-xxxxx.r2.dev`
3. **Settings → CORS Policy → Add** :

```json
[
  {
    "AllowedOrigins": ["*"],
    "AllowedMethods": ["GET", "HEAD"],
    "AllowedHeaders": ["*"],
    "ExposeHeaders": ["Content-Length", "Content-Range", "ETag", "Accept-Ranges"],
    "MaxAgeSeconds": 3600
  }
]
```

> **`ExposeHeaders` est la ligne critique.** Sans elle, le navigateur ne peut pas lire les en-têtes de plage, et DuckDB-WASM se rabat sur le téléchargement intégral du fichier. Tout l'intérêt de l'architecture disparaît sans que rien ne semble cassé.
>
> En production, remplacez `"*"` par votre domaine exact.

4. **Uploader** l'arborescence `parts/` en conservant la structure.

À la main : créer chaque dossier `lettre=X` et y glisser le fichier. Comptez 30 minutes pour 27 dossiers.

Avec **rclone** (recommandé, et réutilisable à chaque mise à jour mensuelle des données) :

```bash
rclone copy parts r2:mon-bucket/parts --progress
```

5. **Tester** dans un navigateur :

```
https://pub-xxxxx.r2.dev/parts/lettre=B/data.parquet
```

Si le téléchargement démarre, la chaîne est valide.

---

## Étape 5 — Générer le site

Ce prompt fonctionne avec n'importe quel générateur de site par IA (Runable, Lovable, Bolt, v0, Claude…). Remplacez l'URL par la vôtre.

<details>
<summary><b>Prompt — moteur de recherche</b></summary>

```
Crée un site web d'une seule page, en français, qui recherche dans une base
de personnes décédées.

CONTRAINTE ABSOLUE : tout s'exécute côté client. Aucun backend, aucun appel
serveur contenant le texte saisi par l'utilisateur. Utilise duckdb-wasm chargé
depuis un CDN.

DONNÉES : fichiers Parquet hébergés à la base d'URL
https://VOTRE-URL/parts/

Ils sont partitionnés par première lettre du nom de famille. Le chemin d'un
fichier est lettre=B/data.parquet pour les noms commençant par B. Les noms
commençant par un caractère non alphabétique sont dans lettre=AUTRE/data.parquet.

COLONNES : nom, prenoms, sexe, date_naissance, code_insee_naissance,
commune_naissance, pays_naissance, date_deces, code_insee_deces,
numero_acte_deces. Toutes de type texte. Dates au format AAAAMMJJ, à afficher
en JJ/MM/AAAA.

FONCTIONNEMENT : un champ « Nom » (obligatoire) et un champ « Prénom »
(optionnel). À la recherche, mets le nom en majuscules, prends sa première
lettre pour déterminer le fichier à interroger, et ne charge que celui-là.
Si la première lettre n'est pas de A à Z, interroge lettre=AUTRE.

FILTRES OPTIONNELS : plage d'années de naissance (deux champs « de » / « à »),
commune de naissance (recherche partielle, insensible à la casse), département
de naissance (deux premiers caractères de code_insee_naissance).

RÉSULTATS : tableau avec nom, prénoms, né(e) le, à (commune), décédé(e) le.
Tri par date de naissance croissante. Limite à 100 résultats affichés, mais
indique le nombre total trouvé. Indicateur de chargement pendant la requête,
message clair si aucun résultat.

MENTIONS : précise que la base couvre les décès survenus en France depuis 1970
et que la recherche s'effectue entièrement dans le navigateur.

Design sobre et lisible, responsive mobile.
```
</details>

### Points de vigilance

- **Le premier essai plante presque toujours.** DuckDB-WASM demande une initialisation particulière (worker, bundle). Ouvrez la console du navigateur (`F12`), copiez l'erreur, donnez-la à l'IA. Deux ou trois allers-retours sont normaux.
- **Casse** : les noms sont en majuscules dans les données. Si l'utilisateur tape `dupont`, la requête doit chercher `DUPONT`.
- **Initialisation anticipée** : demandez que DuckDB-WASM soit initialisé au chargement de la page, pas à la première recherche — sinon le premier utilisateur attend plusieurs secondes sans comprendre pourquoi.
- **Accents** : `MÜLLER` et `MULLER` sont deux entrées distinctes. Une normalisation Unicode côté client améliore nettement le taux de trouvaille.

### Vérifier la promesse

C'est le test qui donne toute sa valeur au projet, et il est reproductible par n'importe qui :

1. `F12` → onglet **Réseau**
2. Lancer une recherche
3. Examiner les requêtes : elles ciblent `data.parquet` avec un en-tête `Range: bytes=...`
4. **Aucune requête ne contient le nom saisi**

Si une requête contient le nom, l'architecture est cassée quelque part.

---

## Étape 6 — Le bouton « demander l'acte »

Le fichier des décès donne la **commune de naissance**. C'est le point d'entrée vers l'acte de naissance — et un acte de naissance ou de décès mentionne les parents du défunt. C'est ainsi qu'on remonte réellement d'une génération.

### Extraire l'annuaire

```sql
COPY (
  SELECT DISTINCT
    substr(Code_insee_commune, 1, 2) AS dept,
    Nom_du_service, Courriel, Site_internet
  FROM 'annuaire-archives.csv'
  WHERE Categorie_de_service = 'D'
    AND Courriel IS NOT NULL AND Courriel <> ''
) TO 'archives_dept.json' (FORMAT JSON, ARRAY true);
```

`Categorie_de_service = 'D'` isole les **Archives départementales** (~118 entrées, DOM inclus) — celles qui détiennent les registres anciens. Les autres codes : `C` municipales, `R` régionales, `N` nationales, `I` autres institutions publiques.

Le fichier obtenu fait quelques dizaines de Ko : intégrez-le en dur dans le code du site, inutile de l'héberger.

<details>
<summary><b>Prompt — bouton mailto</b></summary>

```
Ajoute un bouton « Demander l'acte » sur chaque résultat.

Intègre en dur dans le code la table JSON fournie ci-dessous, qui associe un
code de département aux coordonnées des Archives départementales.

Au clic :
1. Prendre les 2 premiers caractères de code_insee_naissance
2. Retrouver l'entrée correspondante
3. Ouvrir le client mail de l'utilisateur via un lien mailto: pré-rempli

Objet : Demande de copie d'acte de naissance

Corps :
Madame, Monsieur,

Je souhaite obtenir une copie de l'acte de naissance de :

Nom : [NOM]
Prénoms : [PRENOMS]
Né(e) le : [DATE_NAISSANCE en JJ/MM/AAAA]
À : [COMMUNE_NAISSANCE]

Je vous remercie par avance.

Cordialement,

Si aucun département ne correspond, afficher un lien vers le site des archives
si disponible, sinon désactiver le bouton.

Le lien mailto ouvre le logiciel de messagerie de l'utilisateur : aucun envoi
ne part du site.

Table : [COLLER LE JSON]
```
</details>

> **Le `mailto:` n'est pas un détail d'implémentation, c'est un choix d'architecture.** Envoyer les courriels depuis un serveur reviendrait à faire transiter les données de l'utilisateur — exactement ce que le projet évite — et constituerait un envoi automatisé non sollicité vers des services publics. Le `mailto:` laisse l'utilisateur relire et envoyer depuis sa propre messagerie : c'est sa démarche, adressée par lui.

### Le cas des actes récents

En France, la **règle des 75 ans** s'applique : pour un acte de naissance ou de mariage de moins de 75 ans, il faut s'adresser à la **mairie** et justifier de sa filiation. Les actes de décès, eux, sont communicables à toute personne qui en fait la demande depuis 2008.

Pour bien faire, le site devrait router vers la mairie plutôt que vers les Archives départementales quand `date_naissance` a moins de 75 ans. L'annuaire de l'administration (également en open data) fournit les coordonnées des mairies par code INSEE.

---

## Étape 7 — Remonter les générations

C'est ce qui distingue cet outil d'un simple moteur de recherche.

### Le principe

Avec un seul nom de famille, aucun outil ne peut reconstituer un arbre : il y a des milliers d'homonymes et rien ne dit lesquels sont apparentés. Toute prétention inverse est de la fabrication.

Le point de départ minimal est **une personne décédée que l'utilisateur connaît** : un grand-parent, une grand-tante. Nom, prénom, époque approximative.

À partir de là :

```
Personne trouvée
    ↓  on lit sa commune de naissance
Chercher : même patronyme, même commune, né 25 à 35 ans plus tôt
    ↓
Candidats parents
    ↓  recommencer
Génération précédente
```

Chaque itération remonte d'un cran, jusqu'à la limite de la base (décès antérieurs à 1970 absents).

<details>
<summary><b>Prompt — chaînage</b></summary>

```
Ajoute un bouton « Remonter d'une génération » sur chaque résultat.

Au clic, relance une recherche avec :
- le même nom de famille
- la même commune de naissance
- une plage d'années de naissance allant de (année de naissance - 40) à
  (année de naissance - 20)

Affiche les résultats comme « parents possibles » et non comme un fait établi :
il s'agit d'une hypothèse fondée sur le patronyme et le lieu, à vérifier par
l'acte d'état civil.

Conserve l'historique des personnes visitées dans un fil de navigation, pour
que l'utilisateur puisse revenir en arrière et voir l'arbre qu'il a parcouru.
```
</details>

### Sur la formulation

Affichez toujours ces résultats comme des **pistes**, jamais comme des filiations établies. Le patronyme et la commune donnent une forte présomption, pas une preuve — seul l'acte d'état civil fait foi. C'est précisément ce qui sépare cet outil des IA qui inventent des arbres, et c'est un argument de crédibilité plutôt qu'une faiblesse.

Un onboarding explicite aide beaucoup :

> *« Pour commencer, pensez à une personne de votre famille qui est décédée : un grand-parent, un arrière-grand-parent. Vous avez besoin de son nom, son prénom, et à peu près de son année de naissance. »*

---

## Limites à connaître

**Couverture temporelle : 1970 à aujourd'hui.** Le fichier ne contient que les décès enregistrés depuis 1970. Une personne décédée en 1955 est absente. Concrètement, on remonte typiquement 3 à 4 générations, pas jusqu'au Moyen Âge.

**Décès seulement.** Ni naissances, ni mariages, ni filiations. Les liens de parenté sont déduits, jamais donnés.

**Lieu de décès non résolu.** Le fichier fournit `code_insee_deces` sans libellé. Pour afficher un nom de commune, il faut le croiser avec le Code Officiel Géographique de l'INSEE (disponible en open data).

**Qualité variable.** Doublons connus, dates aberrantes, communes disparues ou fusionnées. Les filtres de l'étape 2 en écartent l'essentiel, mais pas tout.

**Homonymie.** Les patronymes fréquents produisent beaucoup de bruit. Les filtres commune et plage de dates sont indispensables pour rendre les résultats exploitables.

**Périmètre géographique.** Le fichier inclut les décès survenus en France de personnes nées à l'étranger, et les décès de Français à l'étranger. Les naissances hors de France sont fréquentes dans les résultats.

**Coût potentiel.** Sous Cloudflare R2, le stockage est gratuit jusqu'à 10 Go et l'egress est gratuit — mais les opérations de classe B (lectures) sont facturées au-delà d'un seuil. Un site qui devient viral peut générer beaucoup de requêtes de plage. Surveillez les métriques.

---

## Aller plus loin

### Autres sources ouvertes

| Source | Volume | Période | Accès |
|---|---|---|---|
| Décès INSEE | 29 M | 1970→ | ✅ Téléchargeable |
| Morts pour la France 14-18 | 1,3 M | 1914-1919 | ⚠️ Téléchargeable, lien difficile à localiser sur le site du ministère des Armées |
| Base Léonore | 381 k | 1802-1977 | ⚠️ Moteur de recherche, pas d'export public |
| Socface (recensements) | 291 M | 1836-1936 | ⚠️ Open Access annoncé, diffusion via FranceArchives |

**Socface est la source la plus prometteuse** : issue du projet ANR/INED, elle transcrit par reconnaissance d'écriture les listes nominatives de 20 recensements. Elle contient la **relation au chef de ménage** — autrement dit des foyers entiers, donc de vrais liens familiaux, ce que le fichier des décès ne fournit jamais. Le projet affiche un engagement de diffusion en Open Access. Contactez l'équipe pour connaître les modalités d'export.

### Ce qu'il ne faut pas faire

**Ne scrapez pas Geneanet, Filae, FamilySearch ou les bases associatives.** En droit français, une base de données constituée par un investissement substantiel est protégée par le droit *sui generis* du producteur (art. L341-1 et L342-1 du code de la propriété intellectuelle) : l'extraction massive est illicite même quand chaque fiche isolée est publique. Ces acteurs poursuivent effectivement.

Et surtout, cela détruit la proposition du projet : un site qui interroge des serveurs tiers fait sortir la saisie de l'utilisateur. La promesse tombe, et elle tombe publiquement dès que quelqu'un ouvre l'onglet Réseau.

**La bonne voie, c'est le courriel.** Les associations généalogiques départementales sont tenues par des bénévoles qui ont dépouillé des registres pendant des années. Un projet non commercial, respectueux de la vie privée, qui renvoie du trafic vers leur site, est un argument qui porte. C'est lent, mais c'est ainsi qu'on obtient des données que les autres n'ont pas.

### Améliorations utiles

- **Recherche phonétique** (Soundex, Metaphone) pour absorber les variantes orthographiques
- **Résolution des codes INSEE** via le Code Officiel Géographique, y compris les communes fusionnées
- **Export GEDCOM** pour réimporter l'arbre dans Gramps ou Ancestris
- **Liens directs vers les registres numérisés** des Archives départementales, ciblés sur la commune et l'année
- **Mise à jour automatisée** : l'INSEE publie mensuellement ; le pipeline complet se rejoue en une commande

---

## Licences et attribution

- **Fichier des personnes décédées** — INSEE / data.gouv.fr, Licence Ouverte 2.0
- **Annuaire des services d'archives** — Ministère de la Culture / FranceArchives, Licence Ouverte 2.0

La Licence Ouverte autorise la réutilisation, y compris commerciale, sous réserve de mentionner la source et la date de dernière mise à jour.

**Obligation spécifique** : excluez les lignes marquées `opposition = true`. Ce n'est pas optionnel.

---

## Remerciements

Ce document doit beaucoup aux généalogistes qui ont critiqué sa première version. Ils avaient raison sur l'essentiel : elle survendait l'outil et passait sous silence la vraie méthode de recherche. Un fichier de décès ne fait pas une généalogie ; il indique seulement où commencer à chercher. Le [guide de recherche en archives](docs/guide-recherche.md) leur est directement redevable.
