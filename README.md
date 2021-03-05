# Specifications fonctionnelles et techniques de l'API OpenJustice

`Version 0.3 (work in progress)`

## Introduction

### Contexte

La Cour de cassation, dans le cadre de la refonte de son site Web, a initié le projet OpenJustice visant à la conception et au développement en interne d'un moteur de recherche dans le corpus jurisprudentiel, mettant celui-ci à disposition du public dans l'esprit du décret sur l'Open Data des décisions de justice.

### Périmètre fonctionnel

Les présentes spécifications se focalisent sur l'API publique du moteur de recherche OpenJustice. 

Il s'agit de l'interface programmatique accessible via le Web et permettant d'effectuer des requêtes sur la base de données où sont indexées les décisions de justice.

Le périmètre fonctionnel de l'API OpenJustice se déploie suivant sept axes (ou points d'entrée) distincts : 

* [Recherche libre : `GET /search`](#recherche-libre--get-search)
* [Récupération d'une décision complète : `GET /decision`](#r%C3%A9cup%C3%A9ration-dune-d%C3%A9cision-compl%C3%A8te--get-decision)
* [Taxonomie : `GET /taxonomy`](https://github.com/Cour-de-cassation/openjustice-specs#taxonomie--get-taxonomy)
* [Statistiques : `GET /stats`](#statistiques--get-stats)
* [Export par lots : `GET /export`](#export-par-lots--get-export)
* [Import par lots : `POST/PUT/DELETE /import`](#import-par-lots--postputdelete-import)
* [Administration : `GET/POST/PUT/DELETE /admin`](#administration--getpostputdelete-admin)

### Choix d'implémentation

L'API OpenJustice est implémentée en JavaScript et repose sur Node.js (avec le module Express et un serveur Nginx en frontal).

Elle interagit avec un serveur de base de données Elasticsearch dont l'interface n'est pas accessible au public (la base de données ne sera donc jamais interrogée directement, mais uniquement par l'intermédiaire de l'API Node.js publique).

### Racine

Le point d'entrée racine de l'API est [http://dev.opj.intranet.justice.gouv.fr/openjustice/v1/](http://dev.opj.intranet.justice.gouv.fr/openjustice/v1/).

Dans la suite de cette documentation, il y sera fait référence par `$API`.

### Types de requêtes

Cette API de type [REST](https://fr.wikipedia.org/wiki/Representational_state_transfer) est accessible via le Web par l'intermédiaire de requêtes HTTP.

Un accès sécurisé via HTTPS n'est a priori pas nécessaire (comme la base de données ne contient que des informations pseudonymisées) mais demeure envisageable : sa mise en place dépendra des conclusions d'une étude en cours portant sur la sécurité du dispositif.

Les requêtes acceptent uniquement des paramètres suivant deux formats :

* Au format **"query string"** classique pour les fonctionnalités publiques de base reposant sur la méthode HTTP GET (`/search`, `/decision`, `/taxonomy`, `/stats` et `/export` dans une certaine mesure), par exemple : `GET $API/search?query=exemple&sort=date`
* Au format [**JSON Web Token**](https://fr.wikipedia.org/wiki/JSON_Web_Token) (JWT) pour les fonctionnalités avancées ou privées (`/export`, `/import` et `/admin`), reposant notamment sur les méthodes HTTP `POST`, `PUT` et `DELETE`, et nécessitant l'identification de l'émetteur. JSON Web Token est un standard ouvert (RFC 75191) permettant l'échange sécurisé de données structurées entre plusieurs parties identifiées (données dont on vérifie l'intégrité à l'aide d'un mécanisme de signature numérique). Dans le cas des points d'entrée `/search`, `/decision` et `/export`, l'utilisation de JSON Web Token (via une identité connue et valide) permettra de récupérer certaines informations absentes de l'accès de base par "query string", comme par exemple l'identifiant de chaque décision dans la base de données intègre d'où elle provient (Jurinet, Jurica, etc.), ou bien encore de bénéficier d'un plus grand nombre de résultats (au total ou par page)

Outre la sécurisation de l'API, l'utilisation du format JWT permettra ainsi d'intégrer aisément des extensions fonctionnelles à destination de réutilisateurs dûment identifiés (ajout d'un mécanisme d'authentification, spécification de l'origine de la requête, définition d'une date d'expiration pour celle-ci dans le cadre de traitements par lots, modification de la pondération du contenu pour le calcul de la pertinence des résultats de recherche, etc.).

Les spécifications de la version JWT de l'API, découlant des spécification du format "query string" de base, sont encore en cours de finalisation et ne seront donc pas détaillées dans la présente version du document.

Les sections suivantes décrivent les paramètres disponibles pour les différentes interfaces de l'API, lesquels sont attendus au format "query string" correctement encodé (par exemple : `GET $API/search?query=petit%20exemple&field%5B%5D=expose&field%5B%5D=moyens&sort=date&order=asc`).

### Formats de données

#### Type de contenu

Les différents points d'entrée de l'API attendent du JSON (`application/json`) en entrée et renvoient du JSON en sortie, encodé en `UTF-8`.

Les seules exceptions sont les points d'entrée qui gèrent l'upload de fichiers : ils acceptent du `multipart/form-data` et renvoient du JSON.

#### Listes simples

Les listes simples sont renvoyées sous forme d'une liste JSON.

Par exemple, la liste des niveaux de publication :

```
[
  { "key": "P", "value": "Publié" },
  { "key": "N", "value": "Non diffusé" },
  { "key": "D", "value": "Diffusé" }
]
```

#### Pagination

Certaines méthodes sont paginées suivant le même modèle de pagination.

Vous n'avez pas à construire vous-même les URLs des pages précédentes et suivantes puisque celles-ci sont disponibles dans la réponse via les attributs `previous_page` et `next_page`, lesquels valent `null` si il n'y a pas de page précédente ou suivante.

Par exemple:

```
{
    "result": [{...}, {...}],
    "page": 0,
    "page_size": 10,
    "total": 20,
    "next_page": "$API/search?page=1",
    "previous_page": null
}
```

#### Gestion d'erreurs

La gestion d'erreur de l'API utilise les codes d'erreur HTTP standards :

* **400** : Requête invalide
* **401** : Authentification requise
* **403** : Permissions insuffisantes
* **423** : Accès bloqué suite à une activité suspecte 
* **500** : Erreur indéfinie côté serveur
* **502** : Le serveur ne répond pas

Lorsque c'est possible, l'API répondra en JSON avec le format suivant :

```
{
  "message": "un message d'erreur"
}
```

#### Support

Si vous n'arrivez pas à comprendre une erreur, ou si vous avez besoin d'aide et souhaitez contacter l'équipe du projet OpenJustice, pensez à fournir les éléments suivants :

* La requête HTTP effectuée (avec les en-têtes HTTP)
* La réponse éventuelle du serveur (avec ses en-têtes)
* La date et l'heure de la requête
* Le contexte et la raison de cette requête

## Recherche libre : `GET /search`

### Description

L'API publique permet en premier lieu d'effectuer une recherche sur la base de données des décisions de justice, suivant les paramètres, filtres et critères suivants :

* Texte en saisie libre, lequel sera mis en correspondance avec tout ou partie du contenu des décisions
* Le mode de mise en rapport des termes de la recherche (*ou*, *et*, expression exacte)
* Contenu ciblé par la recherche : décision intégrale, zones spécifiques de la décision (exposé du litige, moyens, motivations, dispositif), sommaire, titrages, numéro de pourvoi, etc.
* Nature de décision (filtre)
* Matière (filtre)
* Chambre et formation (filtre)
* Juridiction et commission (filtre)
* Niveau de publication (filtre)
* Type de solution (filtre)
* Intervalle de dates (filtre)
* Pertinence et date (tri)
* Nombre de résultats par page et index de la page de résultats affichée (navigation)

La pertinence de la recherche équivaut à un score calculé par Elasticsearch à partir de la correspondance entre le texte en saisie libre et le contenu recherché. Par défaut, le moteur de recherche retourne les résultats classés par pertinence décroissante. 

Les filtres sélectionnés ne modifient pas le score, mais permettent de retirer des résultats les décisions dont le contenu ne coïncide pas avec eux.

Le résultat de la recherche est nécessairement paginé (avec un maximum de 50 résultats par page, pour un maximum de 10 000 résultats au total) et ne contient qu'un aperçu des décisions trouvées (chacune possédant un identifiant unique).

Certaines des informations ne sont retournées que sous forme de clé ou d'identifiant numérique (juridiction, chambre, niveau de publication, etc.). Il convient dès lors d'utiliser le point d'entrée `GET /taxonomy` pour en récupérer l'intitulé complet, ou d'effectuer la requête en utilisant le paramètre `resolve_references=true`.

La récupération d'une décision complète repose sur le point d'entrée `GET /decision`.

### Paramètres de la requête

* **query** (`string`) : la chaîne de caractères correspondant à la recherche (le support du format "simple query" tel qu'il est implémenté par Elasticsearch est envisagé). Une recherche avec un paramètre `query` vide est ignorée et retourne un résultat vide
* **field** (`array`) : la liste des champs, métadonnées ou zones de contenu ciblés par la recherche (parmi les valeurs : `expose`, `moyens`, `motivations`, `dispositif`, `annexes`, `sommaire`, `titrage`, etc. - les valeurs disponibles sont accessibles via `GET $API/taxonomy?id=field`). Une recherche avec un paramètre `field` vide est appliquée à l'intégralité de la décision (introduction et moyens annexés compris) mais va exclure les métadonnées (sommaire, titrage, etc.)
* **operator** (`string`) : l'opérateur logique reliant les multiples termes que le paramètre `query` peut contenir (`or` par défaut, `and` ou `exact` – dans ce dernier cas le moteur recherchera exactement le contenu du paramètre `query`)
* **type** (`array`) : filtre les résultats suivant la natures des décisions (parmi les valeurs : `arret`, `qpc`, `qpj`, `ordonnance`, `saisie`, etc. - les valeurs disponibles sont accessibles via `GET $API/taxonomy?id=type`). Une recherche avec un paramètre `type` vide retourne des décisions de toutes natures
* **theme** (`array`) : filtre les résultats suivant la matière (nomenclature de la Cour de cassation) relative aux décisions (les valeurs disponibles sont accessibles via `GET $API/taxonomy?id=theme`). Une recherche avec un paramètre `theme` vide retourne des décisions relatives à toutes les matières
* **chamber** (`array`) : filtre les résultats suivant la chambre relative aux décisions (les valeurs disponibles sont accessibles via `GET $API/taxonomy?id=chamber`). Une recherche avec un paramètre `chamber` vide retourne des décisions relatives à toutes les chambres
* **formation** (`array`) : filtre les résultats suivant la formation relative aux décisions (les valeurs disponibles sont accessibles via `GET $API/taxonomy?id=formation`). Une recherche avec un paramètre `formation` vide retourne des décisions relatives à toutes les formations
* **jurisdiction** (`array`) : filtre les résultats suivant la juridiction relative aux décisions (les valeurs disponibles sont accessibles via `GET $API/taxonomy?id=jurisdiction`). Une recherche avec un paramètre `jurisdiction` vide retourne des décisions relatives à toutes les juridictions
* **committee** (`array`) : filtre les résultats suivant la commission relative aux décisions (les valeurs disponibles sont accessibles via `GET $API/taxonomy?id=committee`). Une recherche avec un paramètre `committee` vide retourne des décisions relatives à toutes les commissions
* **publication** (`array`) : filtre les résultats suivant le niveau de publication des décisions (parmi les valeurs : `b`, `p`, `r`, `i`, `l`, `c`, etc. - les valeurs disponibles sont accessibles via `GET $API/taxonomy?id=publication`). Une recherche avec un paramètre `publication` vide retourne des décisions de n'importe quel niveau de publication
* **solution** (`array`) : filtre les résultats suivant le type de solution des décisions (parmi les valeurs : `annulation`, `avis`, `cassation`, `decheance`, `designation`, `irrecevabilite`, `nonlieu`, `qpc`, `rabat`, etc. - les valeurs disponibles sont accessibles via `GET $API/taxonomy?id=solution`). Une recherche avec un paramètre `solution` vide retourne des décisions ayant n'importe quel type de solution
* **date_start** et **date_end** (`dates au format dd/mm/yyy`) : permet de restreindre les résultats à l'intervalle de dates fourni
* **sort** (`string`) : permet de choisir la valeur suivant laquelle les résultats sont triés (`score` pour un tri par pertinence, `scorepub` pour un tri par pertinence et niveau de publication et `date` pour un tri par date, vaut `scorepub` par défaut)
* **order** (`string`) : permet de choisir l'ordre du tri (`asc` pour un tri ascendant ou `desc` pour un tri descendant, vaut `desc` par défaut)
* **page_size** (`integer`) : permet de déterminer le nombre de résultats retournés par page (50 maximum, vaut 10 par défaut)
* **page** (`integer`) : permet de déterminer le numéro de la page de résultats à retourner (la première page valant `0`)
* **resolve_references** (`boolean`) : lorsque ce paramètre vaut `true`, le résultat de la requête contiendra, pour chaque information retournée par défaut sous forme de clé, l'intitulé complet de celle-ci (vaut `false` par défaut)

### Format du résultat

Une requête réussie retourne un objet contenant une liste de résultats ainsi que les propriétés suivantes :

* **page** (`integer`) : indice de la page courante de résultat (la première page valant 0)
* **page_size** (`integer`) : nombre de résultats retournés par page (10 par défaut)
* **query** (`object`) : objet contenant les paramètres de la requête originelle
* **total** (`integer`) : nombre total de décisions retournées par la requête
* **next_page** (`string`) : URL de la page de résultats suivante (vaut `null` si la page courante est la dernière)
* **previous_page** (`string`) : URL de la page de résultats précédente (vaut `null` si la page courante est la première)
* **took** (`integer`) : temps d'exécution de la requête (en millisecondes)
* **max_score** (`float`) : score maximal obtenu sur l'ensemble des résultats
* **result** (`array`) : liste des résultats retournés, chaque résultat étant un objet contenant les propriétés suivantes :
  * **id** (`string`) : identifiant de la décision
  * **score** (`float`) : score de la décision
  * **highlight** (`array`) : liste des segments de la décision ayant des correspondances avec la requête saisie, les correspondances étant délimitées par des balises `<em>`. Chaque segment est un objet contenant les propriétés suivantes :
    * **field** (`string`) : nom du champ ou de la zone contenant le segment (par exemple : `expose`, `dispositif`, `summary`, etc.). Il peut y avoir plusieurs segments par zone
    * **fragment** (`string`) : segment de texte contenant une ou plusieurs correspondances
  * **jurisdiction** (`string`) : clé de la juridiction. Par défaut, utiliser `GET $API/taxonomy?id=jurisdiction&key=$jurisdiction` pour récupérer l'intitulé complet de celle-ci. Si la requête utilise `resolve_references=true`, alors cette propriété est retournée sous la forme d'un objet `{ key, value }`, où `key` contient la clé et `value` contient l'intitulé complet de celle-ci
  * **chamber** (`string`) : clé de la chambre. Par défaut, utiliser `GET $API/taxonomy?id=chamber&context_value=$jurisdiction&key=$chamber` pour récupérer l'intitulé complet de celle-ci. Si la requête utilise `resolve_references=true`, alors cette propriété est retournée sous la forme d'un objet `{ key, value }`, où `key` contient la clé et `value` contient l'intitulé complet de celle-ci
  * **number** (`string`) : numéro de pourvoi de la décision
  * **ecli** (`string`) : code ECLI de la décision
  * **formation** (`string`) : clé de la formation. Par défaut, utiliser `GET $API/taxonomy?id=formation&context_value=$jurisdiction&key=$formation` pour récupérer l'intitulé complet de celle-ci. Si la requête utilise `resolve_references=true`, alors cette propriété est retournée sous la forme d'un objet `{ key, value }`, où `key` contient la clé et `value` contient l'intitulé complet de celle-ci
  * **publication** (`string`) : clé du niveau de publication. Par défaut, utiliser `GET $API/taxonomy?id=publication&context_value=$jurisdiction&key=$publication` pour récupérer le nom de celui-ci. Si la requête utilise `resolve_references=true`, alors cette propriété est retournée sous la forme d'un objet `{ key, value }`, où `key` contient la clé et `value` contient l'intitulé complet de celle-ci
  * **creation_date** (`date au format dd/mm/yyy`) : date de création de la décision
  * **solution** (`string`) : clé de la solution. Par défaut, utiliser `GET $API/taxonomy?id=solution&context_value=$jurisdiction&key=$solution` pour récupérer l'intitulé complet de celle-ci. Si la requête utilise `resolve_references=true`, alors cette propriété est retournée sous la forme d'un objet `{ key, value }`, où `key` contient la clé et `value` contient l'intitulé complet de celle-ci
  * **solution_alt** (`string`) : intitulé complet de la solution (si celle-ci n'est pas normalisée et comprise dans la taxonomie)
  * **theme** (`array`) : liste des matières (ou éléments de titrage) par ordre de maillons (texte brut)
  * **summary** (`string`) : sommaire (texte brut)
  * **attachment** (`array`) : liste des documents associés à la décision, chaque document étant représenté par un objet `{ type, URL }` où `type` contient le type de document (communiqué, note explicative, traduction, rapport, avis de l'avocat général, etc.) et `URL` contient le lien vers celui-ci

Rappel : le texte intégral et les zones qu'il contient ne sont pas inclus dans les résultats de la recherche. La récupération d'une décision complète (incluant les zones) repose sur le point d'entrée `GET /decision`.

## Récupération d'une décision complète : `GET /decision`

### Description

Connaissant l'identifiant unique d'une décision, ce point d'entrée permet d'en récupérer le contenu intégral (structuré, mais sans mise en forme), à savoir :

* L'identifiant de sa juridiction
* L'identifiant de sa chambre
* Sa formation
* Son numéro de pourvoi
* Son ECLI
* Son code NAC
* Son niveau de publication
* Son numéro de publication au bulletin
* Sa solution
* Sa date
* Son texte intégral
* Les délimitations des principales zones d'intérêt de son texte intégral (introduction, exposé du litige, moyens, motivations, dispositif et moyens annexés)
* Ses éléments de titrage
* Son sommaire
* Ses documents associés (communiqué, note explicative, traduction, rapport, avis de l'avocat général, etc.)
* Les textes appliqués
* Les rapprochements de jurisprudence

Certaines des informations ne sont retournées que sous forme de clé ou d'identifiant numérique (juridiction, chambre, niveau de publication, etc.). Il convient dès lors d'utiliser le point d'entrée `GET /taxonomy` pour en récupérer l'intitulé complet, ou d'effectuer la requête en utilisant le paramètre `resolve_references=true`.

### Paramètres de la requête

* **id** (`string`) : identifiant de la décision à récupérer (**obligatoire**)
* **resolve_references** (`boolean`) : lorsque ce paramètre vaut `true`, le résultat de la requête contiendra, pour chaque information retournée par défaut sous forme de clé, l'intitulé complet de celle-ci (vaut `false` par défaut)
* **query** (`object`) : objet facultatif contenant les paramètres complets d'une requête de recherche, tel qu'il est retourné en résultat de `GET /search`. Ce paramètre est utilisé pour surligner en retour, dans le texte intégral de la décision, les termes correspondant avec la recherche (ces termes étant délimitées par des balises `<em>`)

### Format du résultat

Une requête réussie retourne un objet contenant les propriétés suivantes :

* **id** (`string`) : identifiant de la décision
* **jurisdiction** (`string`) : clé de la juridiction. Par défaut, utiliser `GET $API/taxonomy?id=jurisdiction&key=$jurisdiction` pour récupérer l'intitulé complet de celle-ci. Si la requête utilise `resolve_references=true`, alors cette propriété est retournée sous la forme d'un objet `{ key, value }`, où `key` contient la clé et `value` contient l'intitulé complet de celle-ci
* **chamber** (`string`) : clé de la chambre. Par défaut, utiliser `GET $API/taxonomy?id=chamber&context_value=$jurisdiction&key=$chamber` pour récupérer l'intitulé complet de celle-ci. Si la requête utilise `resolve_references=true`, alors cette propriété est retournée sous la forme d'un objet `{ key, value }`, où `key` contient la clé et `value` contient l'intitulé complet de celle-ci
* **number** (`string`) : numéro de pourvoi de la décision
* **ecli** (`string`) : code ECLI de la décision
* **nac** (`string`) : code NAC de la décision
* **formation** (`string`) : clé de la formation. Par défaut, utiliser `GET $API/taxonomy?id=formation&context_value=$jurisdiction&key=$formation` pour récupérer l'intitulé complet de celle-ci. Si la requête utilise `resolve_references=true`, alors cette propriété est retournée sous la forme d'un objet `{ key, value }`, où `key` contient la clé et `value` contient l'intitulé complet de celle-ci
* **publication** (`string`) : clé du niveau de publication. Par défaut, utiliser `GET $API/taxonomy?id=publication&context_value=$jurisdiction&key=$publication` pour récupérer le nom de celui-ci. Si la requête utilise `resolve_references=true`, alors cette propriété est retournée sous la forme d'un objet `{ key, value }`, où `key` contient la clé et `value` contient l'intitulé complet de celle-ci
* **creation_date** (`date au format dd/mm/yyy`) : date de création de la décision
* **update_date** (`date au format dd/mm/yyy`) : date de dernière mise à jour de la décision
* **solution** (`string`) : clé de la solution. Par défaut, utiliser `GET $API/taxonomy?id=solution&context_value=$jurisdiction&key=$solution` pour récupérer l'intitulé complet de celle-ci. Si la requête utilise `resolve_references=true`, alors cette propriété est retournée sous la forme d'un objet `{ key, value }`, où `key` contient la clé et `value` contient l'intitulé complet de celle-ci
* **solution_alt** (`string`) : intitulé complet de la solution (si celle-ci n'est pas normalisée et comprise dans la taxonomie)
* **text** (`string`): texte intégral et pseudonymisé de la décision (texte brut)
* **zones** (`object`) : objet définissant les différentes zones détectées dans le texte intégral de la décision. Chaque zone contient une liste d'objets `{ start, end }` indiquant respectivement l'indice de début et de fin des caractères (relativement au texte intégral) contenus dans chaque segment de la zone (une zone pouvant contenir plusieurs segments) :
  * `introduction` : introduction de la décision
  * `expose` : exposé du litige
  * `moyens` : moyens
  * `motivations` : motivations
  * `dispositif` : dispositifs
  * `annexes` : moyens annexés
* **theme** (`array`) : liste des matières (ou éléments de titrage) par ordre de maillons (texte brut)
* **summary** (`string`) : sommaire (texte brut)
* **attachment** (`array`) : liste des documents associés à la décision, chaque document étant représenté par un objet `{ type, description, URL }` où `type` contient le type de document (communiqué, note explicative, traduction, rapport, avis de l'avocat général, etc.), `description` la description spécifique du document et `URL` contient le lien vers celui-ci
* **bulletin** (`string`) : numéro de publication au bulletin
* **applied** (`array`) : liste des textes appliqués par la décision, chaque texte étant représenté par un objet `{ title, URL }` où `title` contient l'intitulé du texte et `URL` contient le lien vers celui-ci
* **linked** (`array`) : liste des rapprochements de jurisprudence, chaque rapprochement étant représenté par un objet décrivant une décision `{ number, description, theme, URL }` où `number` contient son numéro de pourvoi, `description` son court texte descriptif, `theme` la liste de ses matières (ou éléments de titrage) et `URL` le lien vers celle-ci

## Taxonomie : `GET /taxonomy`

### Description

En complément, l'API publique propose la récupération des listes des termes (sous la forme d'un couple clé/valeur) constituants les différents critères et filtres pris en compte par le processus de recherche et les données qu'il restitue, notamment :

* La liste des types de décision (`type`)
* La liste des juridictions dont le système intègre les décisions (`jurisdiction`)
* La liste des chambres (`chamber`)
* La liste des formations (`formation`)
* La liste des commissions (`committee`)
* La liste des niveaux de publication (`publication`)
* La liste des matières (`theme`)
* La liste des solutions (`solution`)
* La liste des champs et des zones de contenu des décisions pouvant être ciblés par la recherche (`field`)
* etc.

La publication de cette taxonomie permettra principalement au prestataire chargé de l'implémentation du frontend (ainsi qu'à certains réutilisateurs avancés) d'automatiser la constitution du formulaire de recherche et l'enrichissement des résultats retournés.

### Paramètres de la requête

* **id** (`string`) : identifiant de l'entrée de taxonomie à interroger (`type`, `jurisdiction`, `chamber`, etc. - les valeurs disponibles sont accessibles via `GET $API/taxonomy`)
* **key** (`string`) : clé dont on veut récupérer l'intitulé complet (le paramètre `id` est alors requis)
* **value** (`string`) : intitulé complet dont on veut récupérer la clé (le paramètre `id` est alors requis)

### Format du résultat

Une requête réussie retourne un objet contenant les propriétés suivantes :

* **id** (`string`) : identifiant de l'entrée de taxonomie interrogée
* **key** (`string`) : clé dont on veut récupérer l'intitulé complet (pour une requête avec `key`)
* **value** (`string`) : intitulé complet dont on veut récupérer la clé (pour une requête avec `value`)
* **result** (`array`) : liste des résultats retournés, chaque résultat étant un objet contenant un couple clé/valeur (appel par `id` seul), ou seulement une clé ou une valeur (appel par `key` ou `value`)

Par exemple :

`GET $API/taxonomy?id=publication` :

```
{
	"id": "publication",
	"result": {
		"nonpub": "Non publié",
		"bulletin": "Au Bulletin",
		"lettre": "Lettre de chambre",
		"rapport": "Au Rapport",
		"c": "Publié C",
	}
}
```

`GET $API/taxonomy?id=publication&key=c` :

```
{
	"id": "publication",
	"key": "c",
	"result": {
		"value": "Publié C",
	}
}
```

`GET $API/taxonomy?id=publication&value=Publi%C3%A9%20C` :

```
{
	"id": "publication",
	"value": "Publié C",
	"result": {
		"key": "c",
	}
}
```

## Statistiques : `GET /stats`

### Description

L'API publiera notamment les statistiques suivantes, mises à jour quotidiennement :

* Nombre de décisions indexées (au total, par année, par juridiction)
* Nombre de requêtes (par jour, par semaine, etc.)
* Date de la décision la plus ancienne, date de la décision la plus récente.

### Paramètres de la requête

`TODO`

### Format du résultat

`TODO`

## Export par lots : `GET /export`

### Description

Destiné aux utilisateurs désirant procéder à leur propre indexation et mise à disposition du contenu, ce point d'entrée leur permet de récupérer des lots de décisions complètes suivant des paramètres et critères simples :

* Nature de décision (filtre)
* Matière (filtre)
* Chambre et formation (filtre)
* Juridiction et commission (filtre)
* Niveau de publication (filtre)
* Type de solution (filtre)
* Intervalle de dates (date de création ou de mise à jour) (filtre)
* Date (tri)
* Nombre de décisions par lot, index du lot (navigation).

L'export par lots est limité par défaut à 100 résultats par lot, pour un maximum de 10 000 résultats au total.

### Paramètres de la requête

`TODO`

### Format du résultat

Une requête réussie retourne un objet contenant une liste de décisions, chaque décision étant un objet similaire à celui retourné par une requête `GET /decision` :

* **offset** (`integer`) : numéro du lot courant (le premier lot ayant un offset valant 0)
* **size** (`integer`) : nombre de résultats retournés par lot
* **query** (`object`) : objet contenant les paramètres de la requête originelle (voir 3.2.5)
* **total** (`integer`) : nombre total de décisions retournées par la requête
* **decisions** (`array`) : liste des décisions retournées.

## Import par lots : `POST/PUT/DELETE /import`

### Description

L'API OpenJustice possède en outre une interface privée et sécurisée destinée à alimenter le contenu de la base Open Data à partir des données pseudonymisées issues de la base documentaire interne de la Cour de cassation.

Cette interface, à l'usage exclusif de la Cour de cassation, permet l'indexation par lots dans Elasticsearch des décisions de justice pseudonymisées (nouvelles ou mises à jour) en vue de leur publication via le moteur de recherche.

### Paramètres de la requête

`TODO`

### Format du résultat

`TODO`

## Administration : `GET/POST/PUT/DELETE /admin`

### Description

Enfin l'API OpenJustice possède une seconde interface privée et sécurisée, elle aussi à l'usage exclusif de la Cour de cassation, destinée à l'administration et à la maintenance du dispositif (concernant essentiellement la récupération de son état et de son historique de fonctionnement).

### Paramètres de la requête

`TODO`

### Format du résultat

`TODO`