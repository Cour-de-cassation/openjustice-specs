# Specifications fonctionnelles et techniques de l'API JUDILIBRE

`Version 1.0.2`

**Les spécifications de l'API publique JUDILIBRE sont dorénavant maintenues au sein du projet [JUDILIBRE-search](https://github.com/Cour-de-cassation/judilibre-search)** :
* Au format Swagger 2.0 : [JUDILIBRE-public-swagger.json](https://github.com/Cour-de-cassation/judilibre-search/blob/dev/public/JUDILIBRE-public-swagger.json)
* Au format OpenAPI 3.0.2 : [JUDILIBRE-public.json](https://github.com/Cour-de-cassation/judilibre-search/blob/dev/public/JUDILIBRE-public.json)

**Une [documentation statique au format HTML](https://github.com/Cour-de-cassation/judilibre-search/tree/dev/public/static) est automatiquement générée dans le même projet.**

## Introduction

### Contexte

La Cour de cassation, dans le cadre de la refonte de son site Web, a initié le projet JUDILIBRE visant à la conception et au développement en interne d'un moteur de recherche dans le corpus jurisprudentiel, mettant celui-ci à disposition du public dans l'esprit du décret sur l'Open Data des décisions de justice.

### Périmètre fonctionnel

Les présentes spécifications se focalisent sur l'API publique du moteur de recherche JUDILIBRE.

Il s'agit de l'interface programmatique accessible via le Web et permettant d'effectuer des requêtes sur la base de données où sont indexées les décisions de justice.

Le périmètre fonctionnel de l'API JUDILIBRE se déploie suivant les six points d'accès (ou "endpoints") suivants :

* La recherche libre : `GET /search`
* La récupération d'une décision complète : `GET /decision`
* La taxonomie : `GET /taxonomy`
* L'export par lots : `GET /export`
* Les statistiques d'utilisation du système : `GET /stats`
* L'état de fonctionnement du système : `GET /healthcheck`

### Choix d'implémentation

L'API JUDILIBRE est implémentée en JavaScript et repose sur Node.js.

Elle interagit avec un serveur de base de données Elasticsearch dont l'interface n'est pas accessible au public (la base de données ne sera donc jamais interrogée directement, mais uniquement par l'intermédiaire de l'API Node.js publique).

Son exposition repose sur la plateforme [PISTE](https://developer.aife.economie.gouv.fr/).

### Racine

Le point d'accès racine de l'API sera de la forme `https://api.piste.gouv.fr/minju/judilibre/v1/` (l'URL finale reste à définir).

Dans la suite de cette documentation, il y sera fait référence par `$API`.

### Types de requêtes

Cette API de type [REST](https://fr.wikipedia.org/wiki/Representational_state_transfer) est accessible via le Web par l'intermédiaire de requêtes HTTPS.

Les requêtes publiques acceptent uniquement des paramètres suivant le format **"query string"** classique, par exemple : `GET $API/search?query=exemple&sort=date`.

En complément, les réutilisateurs pourront s'authentifier via un "jeton" au format [**JSON Web Token**](https://fr.wikipedia.org/wiki/JSON_Web_Token) (JWT), afin d'accéder aux fonctionnalités avancées. JSON Web Token est un standard ouvert (RFC 75191) permettant l'échange sécurisé de données structurées entre plusieurs parties identifiées (données dont on vérifie l'intégrité à l'aide d'un mécanisme de signature numérique). Dans le cas des points d'accès `/search`, `/decision` et `/export`, l'utilisation de JSON Web Token (via une identité connue et valide) permettra de récupérer certaines informations absentes de l'accès de base par "query string", ou bien encore de bénéficier d'un plus grand nombre de résultats (au total ou par page).

### Formats de données

#### Type de contenu

Les différents points d'accès de l'API attendent une entrée au format "query string" correctement encodé (par exemple : `GET $API/search?query=petit%20exemple&field%5B%5D=expose&field%5B%5D=moyens&sort=date&order=asc`) et renvoient du JSON en sortie, encodé en `UTF-8`.

#### Listes simples

Les listes simples sont renvoyées sous forme d'une liste JSON.

Par exemple, la liste des niveaux de publication (`GET $API/taxonomy?id=publication`) :

```
{
  "id": "publication",
  "result": {
    "b": "Publié au Bulletin",
    "l": "Publié aux Lettres de chambre",
    "r": "Publié au Rapport",
    "c": "Communiqué de presse",
  }
}
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
    "next_page": "$API/search?page=1&query=...",
    "previous_page": null
}
```

#### Gestion d'erreurs

La gestion d'erreur de l'API utilise les codes d'erreur HTTP standards :

* **400** : Requête invalide
* **401** : Authentification requise
* **403** : Permissions insuffisantes
* **404** : Ressource demandée introuvable
* **423** : Accès bloqué suite à une activité suspecte
* **500** : Erreur indéfinie côté serveur
* **502** : Le serveur ne répond pas

Lorsque c'est possible, l'API répondra en JSON avec le format suivant :

```
{
  "errors": [
    {
      "msg": "Un message d'erreur"
    },
    {
      "msg": "Un autre message d'erreur"
    }
  ]
}
```

#### Support

Si vous n'arrivez pas à comprendre une erreur, ou si vous avez besoin d'aide et souhaitez contacter l'équipe du projet JUDILIBRE, pensez à fournir les éléments suivants :

* La requête effectuée (avec les en-têtes associés)
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

Certaines des informations ne sont retournées que sous forme de clé ou d'identifiant numérique (juridiction, chambre, niveau de publication, etc.). Il convient dès lors d'utiliser le point d'accès `GET /taxonomy` pour en récupérer l'intitulé complet, ou d'effectuer la requête en utilisant le paramètre `resolve_references=true`.

La récupération d'une décision complète repose sur le point d'accès `GET /decision`.

## Récupération d'une décision complète : `GET /decision`

### Description

Connaissant l'identifiant unique d'une décision, ce point d'accès permet d'en récupérer le contenu intégral (structuré, mais sans mise en forme), à savoir :

* L'identifiant de sa juridiction
* L'identifiant de sa chambre
* Sa formation
* Son numéro de pourvoi
* Son ECLI
* Son code NAC (si applicable)
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

Certaines des informations ne sont retournées que sous forme de clé ou d'identifiant numérique (juridiction, chambre, niveau de publication, etc.). Il convient dès lors d'utiliser le point d'accès `GET /taxonomy` pour en récupérer l'intitulé complet, ou d'effectuer la requête en utilisant le paramètre `resolve_references=true`.

## Taxonomie : `GET /taxonomy`

### Description

En complément, l'API publique propose la récupération des listes des termes (sous la forme d'un couple clé/valeur) constituants les différents critères et filtres pris en compte par le processus de recherche et les données qu'il restitue :

* La liste des types de décision (`type`)
* La liste des juridictions dont le système intègre les décisions (`jurisdiction`)
* La liste des chambres (`chamber`)
* La liste des formations (`formation`)
* La liste des commissions (`committee`)
* La liste des niveaux de publication (`publication`)
* La liste des matières (`theme`)
* La liste des solutions (`solution`)
* La liste des champs et des zones de contenu des décisions pouvant être ciblés par la recherche (`field`)
* La liste des opérateurs admis par le moteur de recherche (`operator`)
* La liste des modalités de tri admises par le moteur de recherche (`order` et `sort`)

La publication de cette taxonomie permettra principalement au prestataire chargé de l'implémentation du frontend (ainsi qu'à certains réutilisateurs avancés) d'automatiser la constitution du formulaire de recherche et l'enrichissement des résultats retournés.

## Statistiques : `GET /stats`

### Description

L'API JUDILIBRE publie notamment les statistiques suivantes, mises à jour quotidiennement :

* Nombre de décisions indexées (au total, par année, par juridiction : `index`)
* Nombre de requêtes (par jour, par semaine, etc. : `request`)
* Date de la décision la plus ancienne, date de la décision la plus récente (`date`)

## Export par lots : `GET /export`

### Description

Destiné aux utilisateurs désirant procéder à leur propre indexation et mise à disposition du contenu, ce point d'accès leur permet de récupérer des lots de décisions complètes suivant des paramètres et critères simples :

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