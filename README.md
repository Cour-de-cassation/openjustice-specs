# Specifications fonctionnelles et techniques de l'API OpenJustice

`Version 0.3`

## 1. Contexte

La Cour de cassation, dans le cadre de la refonte de son site Web, a initié le projet OpenJustice visant à la conception et au développement en interne d’un moteur de recherche dans le corpus jurisprudentiel, mettant celui-ci à disposition du public dans l’esprit du décret sur l'Open Data des décisions de justice.

## 2. Spécifications fonctionnelles

Les présentes spécifications se focalisent sur l’API publique du moteur de recherche OpenJustice. Il s’agit de l’interface programmatique accessible via le Web et permettant d’effectuer des requêtes sur la base de données où sont indexées les décisions de justice.

Le périmètre fonctionnel de l’API OpenJustice se déploie suivant sept axes (ou points d’entrée) distincts : la recherche libre, la récupération d’une décision complète, la taxonomie, les statistiques, l’export par lots, l’import par lots et l’administration.

### 2.1. Recherche libre : `GET /search`

L’API publique permet en premier lieu d’effectuer une recherche sur la base de données des décisions de justice, suivant les paramètres, filtres et critères suivants :

* Texte en saisie libre, lequel sera mis en correspondance avec tout ou partie du contenu des décisions ;
* Le mode de mise en rapport des termes de la recherche (*ou*, *et*, expression exacte) ;
* Contenu ciblé par la recherche : décision intégrale, zones spécifiques de la décision (exposé du litige, moyens, motivations, dispositif), sommaire, titrages, numéro de pourvoi, visa, etc. ; 
* Nature de décision (filtre)  ;
* Matière (filtre) ;
* Chambre et formation (filtre) ;
* Juridiction et commission (filtre) ;
* Niveau de publication (filtre) ;
* Type de solution (filtre) ;
* Intervalle de dates (filtre) ;
* Pertinence et date (tri) ;
* Nombre de résultats par page et index de la page de résultats affichée (navigation).

La pertinence de la recherche équivaut à un score calculé par Elasticsearch à partir de la correspondance entre le texte en saisie libre et le contenu recherché. Par défaut, le moteur de recherche retourne les résultats classés par pertinence décroissante. 

Les filtres sélectionnés ne modifient pas le score, mais permettent de retirer des résultats les décisions dont le contenu ne coïncide pas avec eux.

Le résultat de la recherche est nécessairement paginé (avec un maximum de 50 résultats par page, pour un maximum de 10 000 résultats au total) et ne contient qu’un aperçu des décisions trouvées (chacune possédant un identifiant unique).

La récupération d’une décision complète repose sur le point d’entrée suivant (`GET /decision`).

### 2.2. Récupération d’une décision complète : `GET /decision`

Connaissant l’identifiant unique d’une décision, ce point d’entrée permet d’en récupérer le contenu intégral (structuré, mais sans mise en forme), à savoir :

* L’identifiant de sa juridiction ;
* L’identifiant de sa chambre ;
* Sa formation ;
* Son numéro de pourvoi ;
* Son ECLI ;
* Son code NAC ;
* Son niveau de publication ;
* Son numéro de publication au bulletin ;
* Sa solution ;
* Sa date ;
* Son texte intégral ;
* Les délimitations des principales zones d’intérêt de son texte intégral (introduction, exposé du litige, moyens, motivations, dispositif et moyens annexés) ;
* Ses éléments de titrage ;
* Son sommaire ;
* Ses visas ;
* Ses documents associés (communiqué, note explicative, traduction, rapport, avis de l'avocat général) ;
* Les textes appliqués ;
* Les rapprochements de jurisprudence.

Certaines de ces informations ne sont retournées que sous forme de clé ou d’identifiant numérique (juridiction, chambre, niveau de publication, etc.). Il convient dès lors d’utiliser le point d’entrée suivant (`GET /taxonomy`) pour en récupérer la valeur "humainement lisible".

### 2.3. Taxonomie : `GET /taxonomy`

En complément, l’API publique propose la récupération des listes des termes (sous la forme d’un couple clé/valeur) constituants les différents critères et filtres pris en compte par le processus de recherche et les données qu’il restitue, notamment :

* La liste des natures de décision ;
* La liste des juridictions dont le système intègre les décisions ;
* La liste des chambres ;
* La liste des formations ;
* La liste des commissions ;
* La liste des niveaux de publication ;
* La liste des matières ;
* La liste des solutions ;
* La liste des contenus des décisions pouvant être ciblés par la recherche.

La publication de cette taxonomie permettra principalement au prestataire chargé de l’implémentation du frontend (ainsi qu’à certains réutilisateurs avancés) d’automatiser la constitution du formulaire de recherche et l’enrichissement des résultats retournés. 

### 2.4. Statistiques : `GET /stats`

L’API publiera notamment les statistiques suivantes, mises à jour quotidiennement :

* Nombre de décisions indexées (au total, par année, par juridiction) ;
* Nombre de requêtes (par jour, par semaine, etc.) ;
* Date de la décision la plus ancienne, date de la décision la plus récente.

### 2.5. Export par lots : `GET /export`

Destiné aux utilisateurs désirant procéder à leur propre indexation et mise à disposition du contenu, ce point d’entrée leur permet de récupérer des lots de décisions complètes suivant des paramètres et critères simples :

* Nature de décision (filtre) ;
* Matière (filtre) ;
* Chambre et formation (filtre) ;
* Juridiction et commission (filtre) ;
* Niveau de publication (filtre) ;
* Type de solution (filtre) ;
* Intervalle de dates (date de création ou de mise à jour) (filtre) ;
* Date (tri) ;
* Nombre de décisions par lot, index du lot (navigation).

L’export par lots est limité par défaut à 100 résultats par lot, pour un maximum de 10 000 résultats au total.

### 2.6. Import par lots : `POST/PUT/DELETE /import`

L’API OpenJustice possède en outre une interface privée et sécurisée destinée à alimenter le contenu de la base Open Data à partir des données pseudonymisées issues de la base documentaire interne de la Cour de cassation.

Cette interface, à l’usage exclusif de la Cour de cassation, permet l’indexation par lots dans Elasticsearch des décisions de justice pseudonymisées (nouvelles ou mises à jour) en vue de leur publication via le moteur de recherche.

### 2.7. Administration : `GET/POST/PUT/DELETE /admin`

Enfin l’API OpenJustice possède une seconde interface privée et sécurisée, elle aussi à l’usage exclusif de la Cour de cassation, destinée à l’administration et à la maintenance du dispositif (concernant essentiellement la récupération de son état et de son historique de fonctionnement).

## 3. Spécifications techniques

### 3.1. Choix d’implémentation

L’API OpenJustice est implémentée en JavaScript et repose sur Node.js (avec le module Express et un serveur Nginx en frontal).

Elle interagit avec un serveur de base de données Elasticsearch dont l’interface n’est pas accessible au public (la base de données ne sera donc jamais interrogée directement, mais uniquement par l’intermédiaire de l’API Node.js publique).

### 3.2. Types de requêtes

Cette API de type [REST](https://fr.wikipedia.org/wiki/Representational_state_transfer) sera accessible via le Web par l’intermédiaire de requêtes HTTP.

Un accès sécurisé via HTTPS n’est a priori pas nécessaire (comme la base de données ne contient que des informations pseudonymisées) mais demeure envisageable : sa mise en place dépendra des conclusions d’une étude en cours portant sur la sécurité du dispositif.

Ces requêtes accepteront uniquement des paramètres suivant deux formats : 

* Au format **"query string"** classique pour les fonctionnalités publiques de base reposant sur la méthode HTTP GET (`/search`, `/decision`, `/taxonomy`, `/stats` et `/export` dans une certaine mesure), par exemple : `GET /search?query=exemple&sort=date`
* Au format [**JSON Web Token**](https://fr.wikipedia.org/wiki/JSON_Web_Token) (JWT) pour les fonctionnalités avancées ou privées (`/export`, `/import` et `/admin`), reposant notamment sur les méthodes HTTP `POST`, `PUT` et `DELETE`, et nécessitant l’identification de l’émetteur. JSON Web Token est un standard ouvert (RFC 75191) permettant l'échange sécurisé de données structurées entre plusieurs parties identifiées (données dont on vérifie l’intégrité à l’aide d’un mécanisme de signature numérique). Dans le cas des interfaces `/search`, `/decision` et `/export`, l’utilisation de JSON Web Token (via une identité connue et valide) permettra de récupérer certaines informations absentes de l’accès de base par "query string", comme par exemple l’identifiant de chaque décision dans la base de données intègre d’où elle provient (Jurinet, Jurica, etc.), ou bien encore de bénéficier d’un plus grand nombre de résultats (au total ou par page).

Outre la sécurisation de l’API, l’utilisation du format JWT permettra ainsi d’intégrer aisément des extensions fonctionnelles à destination de réutilisateurs dûment identifiés (ajout d’un mécanisme d’authentification, spécification de l’origine de la requête, définition d’une date d’expiration pour celle-ci dans le cadre de traitements par lots, modification de la pondération du contenu pour le calcul de la pertinence des résultats de recherche, etc.).

Les spécifications de la version JWT de l’API, découlant des spécification du format "query string" de base, sont encore en cours de finalisation et ne seront donc pas détaillées dans la présente version du document.

Les sections suivantes décrivent les paramètres disponibles pour les différentes interfaces de l’API, lesquels sont attendus au format "query string" correctement encodé (par exemple : `GET /search?query=petit%20exemple&fields%5B%5D=expose&fields%5B%5D=moyens&sort=date&order=asc`).

#### 3.2.1 `GET /search`

* query (string) : la chaîne de caractères correspondant à la recherche (le support du format « simple query » tel qu’il est implémenté par Elasticsearch est envisagé). Une recherche avec un paramètre query vide est ignorée ;
* fields (array) : la liste des champs ou zones de contenu ciblés par la recherche (parmi les valeurs : 'expose', 'moyens', 'motivations', 'dispositif', 'annexes', etc.). Une recherche avec un paramètre fields vide est appliquée à l’intégralité de la décision (chapeau et moyens annexés compris) mais va exclure les autres champs (sommaire, titrage, etc.) ;
* operator (string) : l’opérateur logique reliant les multiples termes que le paramètre query peut contenir ('or' par défaut, 'and' ou 'exact' – dans ce dernier cas le moteur recherchera exactement le contenu du paramètre query) ;
* type (array) : filtre les résultats suivant la natures des décisions (parmi les valeurs : 'arret', 'qpc', 'qpj', 'ordonnance', 'saisie'). Une recherche avec un paramètre type vide retourne des décisions de toutes natures ;
* theme (array) : filtre les résultats suivant la matière (nomenclature de la Cour de cassation) relative aux décisions (les valeurs disponibles seront accessibles via GET /taxon?id=theme). Une recherche avec un paramètre theme vide retourne des décisions relatives à toutes les matières ;
* chamber (array) : filtre les résultats suivant la chambre relative aux décisions (les valeurs disponibles seront accessibles via GET /taxon?id=chamber). Une recherche avec un paramètre chamber vide retourne des décisions relatives à toutes les chambres ;
* formation (array) : filtre les résultats suivant la formation relative aux décisions (les valeurs disponibles seront accessibles via GET /taxon?id=formation). Une recherche avec un paramètre formation vide retourne des décisions relatives à toutes les formations ;
* jurisdiction (array) : filtre les résultats suivant la juridiction relative aux décisions (les valeurs disponibles seront accessibles via GET /taxon?id=jurisdiction). Une recherche avec un paramètre jurisdiction vide retourne des décisions relatives à toutes les juridictions ;
* committee (array) : filtre les résultats suivant la commission relative aux décisions (les valeurs disponibles seront accessibles via GET /taxon?id=committee). Une recherche avec un paramètre committee vide retourne des décisions relatives à toutes les commissions ;
* publication (array) : filtre les résultats suivant le niveau de publication des décisions (parmi les valeurs : 'nonpub', 'bulletin', 'lettre', 'rapport', 'c'). Une recherche avec un paramètre publication vide retourne des décisions de n’importe quel niveau de publication ;
* solution (array) : filtre les résultats suivant le type de solution des décisions (parmi les valeurs : 'annulation', 'avis', 'cassation', 'decheance', 'designation', 'irrecevabilite', 'nonlieu', 'qpc', 'rabat'). Une recherche avec un paramètre solution vide retourne des décisions ayant n’importe quel type de solution ;
* date_start et date_end (date au format 'dd/mm/yyy') : permet de restreindre les résultats à l’intervalle de dates fourni ;
* sort (string) : permet de choisir la valeur suivant laquelle les résultats sont triés ('score' pour un tri par pertinence ou 'date' pour un tri par date, vaut 'score' par défaut) ;
* order (string) : permet de choisir l’ordre du tri ('asc' pour un tri ascendant ou 'desc' pour un tri descendant, vaut 'desc' par défaut) ;
* size (integer) : permet de déterminer le nombre de résultats retournés par page (50 maximum, vaut 10 par défaut) ;
* offset (integer) : permet de déterminer le numéro de la page de résultats à retourner (la première page ayant un offset valant 0).


