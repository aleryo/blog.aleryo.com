---
title: Ils ne savaient pas que c’était impossible...
author: Aleryo
date: 2017-11-13
---

# Ils ne savaient pas que c’était impossible...

## Introduction

Maintenir une _application patrimoniale_ est souvent considérée comme une
activité moins noble que concevoir un nouveau système. Une telle application
utilise des technologies en voie d'obsolescence, peu susceptibles d'être
valorisable sur un CV ; sa structure est souvent compliquée, étant le résultat
de multiples décisions prises par des personnes différentes dans des contextes
différents, parfois sous la pression de contraintes économiques ou
organisationnelles fortes ; tout changement nécessite de prudentes manœuvres au
travers d'un code contenant beaucoup de duplication, de code mort, de
connaissance implicite difficile à découvrir. Et pourtant de telles applications
ont beaucoup de valeurs pour les organisations les utilisant.

Cet article est le récit de notre expérience récente sur une application de ce
type, comment nous avons appliqué le _développement dirigé par les tests_ et
d'autres pratiques et principes de _l'eXtreme Programming_ ; comment nous avons
pu, pas à pas, mettre en place des harnais de test de plus en plus rapides et
précis nous permettant d'accroître notre compréhension du système et notre
capacité à le faire évoluer ; comment nous avons échoué à appliquer cette
stratégie sur une partie de code.

## L'herbe n'est pas toujours verte

### Créer une équipe

Notre point de départ est un appel d'offres pour la reprise en _tierce
maintenance applicative_ d'une application d'un éditeur dans le domaine de la
finance : le client cherche une équipe avec un _Product Owner_, un testeur, une
développeuse séniore et deux juniors. L'application a une dizaine d'années, deux
versions sont en production chez des clients et une prochaine est en cours de
développement pour livraison fin d'année sur laquelle une pression importante de
la direction s'exerce.

L'éditeur dispose en interne d'une équipe d'une dizaine de personnes mais cela
ne semble pas suffire pour gérer la maintenance corrective et évolutive et
développer la nouvelle version, d'où le besoin de "renforts". Le client est prêt
à travailler avec une équipe située en dehors de ses locaux mais cela n'a rien
de très surprenant dans un contrat de TMA : combien sont délocalisées en
province ou dans des pays à bas salaires ?

Bref, rien de très excitant en apparence ! Mais nous avons un bon contact avec
le client, le budget est adapté aux enjeux et nous avons envie de répondre
positivement. Nous faisons donc une contre-proposition : pour le même budget, au
lieu d'une équipe pyramidale de 5-6 personnes organisée comme un mini centre de
services, nous proposons une équipe répartie auto-organisée de 4 personnes
expérimentées:

* tous les membres de l'équipe seront des développeurs ou développeuses
  **chevronnées**, compétentes techniquement mais aussi capables de prendre en
  main l'organisation et de parler le langage du client ;
* il n'y aura **pas de chef** identifié dans l'équipe ;
* les membres de l'équipe travailleront depuis le lieu de leur choix, **à
  distance**, ce qui facilitera la constitution de l'équipe en supprimant la
  barrière de la co-localisation ;
* l'équipe mettra en oeuvre les pratiques et principes de
  l'[**eXtreme Programming**](http://extremeprogramming.org), développement
  dirigé par les tests, intégration continue, boucles de rétroactions courtes,
  priorisation par la valeur métier…

Après discussion rapide, le client est séduit par la proposition, accepte
l'offre et organise un premier rendez-vous afin de nous présenter l'application
et l'équipe.

### Découvrir le paysage

L'application est structurée en deux composants disjoints : un _backend_
organisé selon le schéma des [pipes and filters](http://www.enterpriseintegrationpatterns.com/patterns/messaging/PipesAndFilters.html),
un frontend de type application web. Entre les deux se trouve une base de
données relationnelle classique qui sert de mémoire partagée
persistante.

Première surprise : le backend est codé dans un langage propriétaire
multi-paradigmes, à la fois orienté-objet, fonctionnel et même logique,
compilé vers du C++. Au dessus de ce langage socle, l'équipe a développé un
autre langage propriétaire, syntaxiquement structuré comme un langage à balises,
dans lequel sont codés les processus métiers.

Deuxième surprise : le frontend est certes codé en Java/Javascript/HTML dans une
classique _webapp_ JEE, mais il est structuré par un _framework_ maison dont le
but est de décrire le flux d'interaction des utilisateurs sous la forme de
processus, des graphes dirigés reliant des pages, des requêtes SQL, des
traitements...

Troisième surprise (qui n'en est pas vraiment une) : il n'y a aucun test
unitaire et très peu de tests automatisés, ceux qui existent couvrant un
périmètre de fonctionnalités restreint et uniquement dans le backend. Quelques
efforts d'automatisation de tests sur le front ont été entrepris mais ils sont
encore balbutiants et ne font pas partie du processus standard de développement.

Nous nous retrouvons donc devant une base de code relativement conséquente,
séparée en deux parties développées de manière indépendantes mais fortement
couplées par une base de données dont le schéma se complexifie, sans tests, et
avec une qualité d'écriture de code plutôt médiocre voire par endroits
calamiteuse : copier-coller à outrance, beaucoup de code mort, règles de nommages incohérentes,
couplage fort entre composants, fonctions et expressions complexes sur plusieurs
dizaines de lignes, commentaires obsolètes voire contradictoires...

L'équipe en place suit un processus de développement à la Scrum : stand-up
meeting tous les matins, tableaux d'avancement des _user stories_ dans Jira,
sprints de deux semaines, scrummaster, niko-niko... Malgré le contexte difficile
d'une application vieillissante et rétive à toute modification, le moral et
l'ambiance dans l'équipe semblent bons, et le processus de développement
apparaît plutôt bien suivi.

## Explorer

### Prise en main

Nos premiers contacts avec le code du _backend_ sont laborieux, le cycle de
développement est long et pénible :

* il faut tout d'abord écrire le code dans un langage que nous ne maîtrisons pas
  encore ;
* le compilateur ne supportant que le 32 bits, il nécessite l'installation d'une
  _toolchain_ et de bibliothèques spécifiques, ainsi que le paramétrage du
  compilateur C ;
* ce code doit ensuite être compilé et "packagé" à l'aide scripts et de
  commandes spécifiques et inhabituelles ;
* pour exécuter ce code, il faut le déployer selon une structure de répertoire
  précise qui fait partie des sources ;
* les scripts et le code de configuration contiennent beaucoup de chemins
  relatifs de type `../something` ce qui ne facilite pas la compréhension de
  leur structure ni de leurs effets ;
* il y a plusieurs étapes manuelles nécessaires pour parvenir à obtenir un
  système fonctionnel ;
* la bonne exécution de tout ce processus dépend de variables d'environnements
  dont la sémantique est floue.

Chaque opération prend beaucoup de temps - parfois quelques minutes - et est peu
répetable - une installation peut modifier l'environnement de sorte que la
prochaine installation échoue. Pour nous qui sommes habitués à un cycle trés
rapide (2-3 secondes) pour écrire le code, le compiler, observer un résultat,
c'est inacceptable.

Notre premiere étape pour prendre en main le système va donc consister à
_automatiser_ ce processus au travers de simples scripts afin de compiler le
code, le _packager_ pour produire un paquet déployable, le déployer dans un
répertoire, lancer le processus serveur. Lors de l'écriture de ces scripts, nous
prenons bien garde à gérer correctement l'arborescence des répertoires : le
script devra pouvoir être lancer de n'importe quel endroit et gérer les chemins
relatifs existant, ainsi que positionner les variables d'environnement
nécessaires. Enfin, nous commencons par travailler sur un nouveau composant ce
qui nous permet de mettre au point ce cycle de compilation-déploiement-exécution
sans avoir à gérer le passif ni modifier de code existant.

Assez rapidement nous atteignons une _cadence_ qui nous permet de mieux
comprendre ce qui se passe et partant de commencer à mouvoir notre squelette de
composant. Il devient possible de copier/coller du code existant en élaguant les
parties non pertinentes et d'obtenir un module fonctionnel, capable de
simplement copier ses entrées en sorties mais déployable, compilable,
utilisable. L'étape suivante va consister à rajouter de la chair à ce squelette,
ce qui nécessite de mettre en places des _tests_ automatisés.

### Sous toute les coutures

Après quelques heures de travail et d'automatisation du cycle de développement,
nous comprenons mieux l'architecture du système:

* chaque composant est un _micro-service_ isolé, déployable indépendamment,
  construit à partir de modules - bibliothèques pré-compilées ou directement
  importées sous forme de sources. Il faudra faire un travail sur les modules
  contenant beaucoup de code redondant issu de vagues successives de
  copier-coller à la va-vite, mais au moins nous pouvons travailler sur un
  sous-système isolé, plus simple ;
* chaque composant fonctionne comme un *processeur* relié à d'autres processeurs
  au moyen de *files* qui peuvent être en entrée ou en sortie ;
* ces files sont de diverses natures mais pour l'essentiel sont constituées de
  *répertoires* contenant des fichiers aux formats variés et d'une *base de
  données* ;
* chaque composant produit donc un résultat dans des files de sorties, mais
  aussi des *logs* ;
* le traitement de la base de données comme une *file* en entrée - requêtes
  `SELECT` - ou en sorties - requêtes `INSERT` ou `UPDATE` - produit un certaine
  dissonance cognitive d'autant plus qu'il ne s'agit pas ici
  de [requêtes SQL en continu](https://en.wikipedia.org/wiki/StreamSQL). Cette
  dissonance se constate dans le code où le traitement des requêtes SQL est
  significativement plus compliqué. Mais cette uniformisation a le mérite de
  rendre la conception plus claire.

Ne maîtrisant pas - encore - le langage source, nous choisissons donc de
commencer l'automatisation des tests par le _haut_, c'est-à-dire d'écrire des
tests _systèmes_ qui vont tester le comportement global d'un composant. Cette
stratégie nécessite de construire un environnement d'exécution _propre_ pour
chaque test, donc d'isoler les données :

* l'arborescence des files d'entrées-sorties est créée automatiquement et de
  manière unique pour chaque test ;
* la base de données est exécutée dans un
  container [docker](https://github.com/wnameless/docker-oracle-xe-11g) ce qui
  nous garantit que chaque exécution est isolée.

### Ceintures et bretelles

Avoir des tests automatiques, repétables, isolés est une étape essentielle dans
notre prise en main du système permettant de :

* réduire le temps de cycle de chaque développement et fluidifier le processus
  en évitant d'inévitables erreurs et oublis qu'introduiraient des étapes
  manuelles ;
* raccourcir la longueur des boucles de _feedback_ et donc mieux guider notre
  développement ;
* créer des points de stabilité dans notre travail et pouvoir _commiter_ sur des
  _barres vertes_.

Ce harnais de test va se révéler fondamental pour nous aider à comprendre le
système et surtout le langage et son écosystème.

Se pose alors la question du langage dans lequel écrire ces tests systèmes.
Aprés avoir hésité avec Haskell que nous maîtrisons plutôt bien, nous nous
tournons vers Python qui présente des avantages qui nous paraissent pertinents pour ce projet:

* il est disponible facilement et fonctionne sans difficulté sur toutes les plate-formes ;
* il est facile à apprendre et à utiliser ce qui devrait lisser la courbe
  d'apprentissage pour nous même et pour d'autres personnes qui seraient amenées
  à rejoindre l'équipe ou travailler sur ces tests ;
* enfin python est très souple et dispose d'un outillage et d'un éco-système
  très riche pour gérer les aspects _systèmes_, de manière plus structurée que
  ne le permettrait le _shell_.

Après quelques itérations nous sommes parvenus à nos fins :

* nous avons une boucle de rétroaction fiable et relativement rapide ;
* nous pouvons développer notre code en le dirigeant par des tests plutôt fonctionnels ;
* nous maîtrisons mieux le système et sommes capables de le découper en unités plus petites.

## Évolution des tests

### Réduire l'empreinte carbone

Une fois passée l'euphorie des premières heures et lorsqu'il devient nécessaire
de coder réellement des comportements plus fins, nous nous heurtons à une limite
attendue de cette stratégie. Les tests python prennent de plus en plus de temps
pour gérer la mise en place des données: base, fichiers, nettoyage après
exécution, manipulation des processus qui nécessitent d'utiliser des mécanismes
de temporisations, tout cela est fragile et lent. Nos itérations de codage sont
aussi plus longues car chaque test décrit une fonctionnalité de haut niveau qui
nécessite plus de travail, donc introduit le risque de faire une erreur qui sera
détectée plus tard.

Il est nécessaire de raffiner nos tests, de descendre à un niveau de granularité
plus fin et donc d'écrire des tests unitaires _dans le langage
d'implémentation_. Les bénéfices attendus sont nombreux :

* nous permettre de mieux comprendre le langage que nous manipulons ;
* s'habituer à écrire du code dans ce langage donc à faire évoluer notre
  environnement de développement pour l'y adapter ;
* réduire la durée d'exécution.

Notre première étape et d'écrire les tests comme des fonctions dans un
exécutable spécifique qui sera _linké_ avec le code fonctionnel : chaque test
prend la forme d'une fonction `testXXX` se terminant par un appel à la fonction
interne `assert`, et le `main` exécute l'ensemble des tests de manière
séquentielle. Cette première approche nous permet rapidement d'écrire quelques
dizaines de tests et de raccourcir encore la boucle de rétroaction.

Dans une deuxième étape, nous sollicitons l'aide d'un architecte du client
maîtrisant parfaitement le langage pour nous écrire un framework _XUnit_. Le
passage à ce _framework_ de test va réduire la duplication et le travail de
"gestion" des tests en découvrant automatiquement les fonctions testables, et
clarifier l'écriture des tests grâce à des fonctions _d'assertion_ spécialisées.

Après environ un mois de travail, nous sommes donc dans une situation
relativement confortable pour développer sur cette partie _backend_ : nous avons
une batterie de tests fonctionnels automatisés qui nous aide dans la réalisation
des fonctionnalités attendues du système ; et une batterie de tests unitaires
qui nous aide dans les détails de l'écriture du code.

### Refactor Mercilessly

Trois mois plus tard, nous constatons que les tests python sont devenus un
_passif_ en tant que tel :

* il y a beaucoup de duplication générée par du copier-coller ;
* la taille des fichiers de tests qui peut atteindre plus de 1000 lignes les
  rend ingérables et incompréhensibles ;
* les tests contiennent beaucoup de _technique_ automatisant les tâches
  nécessaires à la mise en place du système ;
* l’intention fonctionnelle de chaque test est noyée dans ce bruit technique.

Une conséquence immédiate de cette complication est que les nouveaux arrivant
ont des difficultés à "habiter" le code des tests et à le prendre en main.

Nous profitons donc de l’arrivée dans l'équipe d'un nouveau membre pour qu’il
nous guide vers des tests qui lui _parlent davantage_. Ce travail prend la forme
de séances de binômage avec une personne connaissant le système existant et
permet de réduire la taille des fonctions de tests en en extrayant des fonctions
utilitaires initialisant les états de la base, par exemple
`Db_should_contains_a_basket_with_a_discounted_product`.

#### Passage à l'échelle

Malgré ces _refactorings_ une douleur importante persiste : il est difficile de
créer un test _from scratch_ pour une nouvelle fonctionnalité, cela oblige à
copier/coller beaucoup de code, donc introduit de la duplication et accroît le
degré d'entropie du code des tests.

Nous allons donc regrouper toutes nos fonctions utilitaires dans 3 “univers”:

* la base de données :
    * l'insertion de données métier,
    * les assertions sur le contenu de la base ;
* les processus systèmes :
  * démarrage des processus et gestion des états,
  * assertions sur les logs ;
* les fichiers :
  * création des répertoires et manipulation des fichiers,
  * assertions sur les répertoires et fichiers (taille, contenu, nom, structure...).

Ces univers sont plutôt de l'ordre de la technique et non du _métier_. Il s’agit
plus de traduire dans notre code et de représenter sous un forme exécutable le
_langage_ qui est utilisé au quotidien dans l’équipe que d'adopter une approche
_orienté métier_ ou _domaine_ qui ne nous aide pas à améliorer le système et à
rendre les tests plus lisibles. Le _bon_ langage est ici celui de l’application
et de son architecture et non pas celui de l’utilisateur.

### Quid du frontend ?

Parallèlement au travail sur le backend que nous avons détaillé ci-dessus, nous
avons été amené à travailler aussi sur le _frontend|_ et, bien évidemment, nous
avons voulu mettre en œuvre la même approche. Mais cette stratégie n'a pas aussi
bien fonctionné, malheureusement, bien que l'approche _outside-in_ apparaisse
tout aussi pertinente du fait de l'existence d'un important passif.

À cela, nous pouvons identifier plusieurs raisons:

* la nature technique de l’appli: elle est uniquement testable dans un
  navigateur ce qui implique une complexité de mise en oeuvre bien supérieure,
  et une fragilité plus importante :
    * c’est - beaucoup - plus compliqué que de poser un fichier texte dans un répertoire et de lancer une commande,
    * il faut lancer un navigateur, éventuellement dans des containers, avec des résultats non-prédictibles,
    * la fragilité vient du fait de n'être pas certain d’avoir toujours le même résultat à chaque lancement ;
* l’investissement initial qui est beaucoup plus élevé avec des risques plus
  importants. Sur le _backend_, nous avons pu le faire sans avoir besoin de
  l’autorisation du sponsor mais sur le _frontend_, le temps de retour sur
  investissement n’était pas acceptable par le sponsor ;
* nous avons eu au moins un succés mineur : construire un modèle réduit
  d'application myBatis pour ne pas avoir à lancer tout le frontend lorsque l'on
  souhaite travailler uniquement sur les données et requêtes SQL.

## Conclusion

Par manque de recul et sous la pression du quotidien, l'équipe en
place avait estimée qu'il était impossible de faire des tests
automatisés à l'interieur de ce code, la seule stratégie envisagée
étant de délocaliser l'exécution de tests non régression manuels vers
des pays à bas salaires.

Nous avons pris le parti d'aborder cette application avec les mêmes
outils et principes qui nous avaient réussis dans d'autres contextes :
rester concentré sur la valeur métier produite, éviter de longs
tunnels de _refactoring_, ne pas chercher à utiliser ou créer des
frameworks compliqués, s'appuyer sur des fonctionnalités disponibles
nativement dans les langages utilisés, enfin et surtout automatiser et
raccourcir la boucle de rétroaction.

Cette stratégie, facilitée par la conception modulaire du _backend_,
s'est avérée payante et ce harnais de tests pourrait servir de graine
pour aller vers des _refactoring_ plus larges. Et, nous l'espérons,
vers une migration technique attendue par toute l'équipe.

Même si le _front-end_ s'est avéré plus rétif à nos efforts, nous n'avons pas
baissé les bras. Un _refactoring_ progressif est probablement trop risqué et
coûteux et nous envisageons une autre stratégie : faire croître un deuxieme
_front-end_ à coté de l'existant.