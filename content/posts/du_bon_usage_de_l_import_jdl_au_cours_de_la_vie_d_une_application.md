---
title: "Du bon usage de l'import JDL au cours de la vie d'une application"
date: 2022-03-11T14:38:17+01:00
draft: false
---

Dans cet article nous allons poursuivre l'exploration du framework [JHipster](https://www.jhipster.tech/) et en particulier de la fonctionnalité d'import de fichiers JDL et de son usage au cours du cycle de vie d'une application.

JHipster permet en effet de définir la configuration de l'application et des entités dans un fichier JDL puis d'importer ce fichier pour générer automatiquement l'ensemble de l'application d'une simple commande. Pour notre exemple nous allons utiliser le fichier `petit_dej_v0.jdl` suivant :

````
application {
  config {
    baseName PetitDej,
    applicationType monolith,
    packageName com.dooapp.petitdej,
    authenticationType jwt,
    prodDatabaseType postgresql,
    clientFramework angular
    enableSwaggerCodegen true
    nativeLanguage fr
  }
  entities *
}

entity PetitDej {
  date LocalDate required
  commentaire String
}

relationship ManyToOne {
	PetitDej{organisateur(login)} to User
}

relationship ManyToMany {
    PetitDej{participants(login)} to User
}
````

L'application peut alors être générée grâce à la commande suivante :

     jhipster jdl petit_dej_v0.jdl

Jusque là, tout va bien. Seulement, notre nouvelle application ne fait que commencer son cycle de vie, et, on lui souhaite, va probablement faire l'objet de nombreuses évolutions et modifications à l'avenir. En particulier, notre modèle métier va probablement évoluer avec l'ajout, la suppression ou la modification d'entités et de relations.

JHipster permet d'effectuer ces modifications en utilisant l'[entity sub-generator](https://www.jhipster.tech/creating-an-entity/#updating-an-existing-entity) de JHispter. Mais le code source de notre application ne sera alors plus cohérent avec la description initiale dans le fichier JDL. Il est donc bien plus tentant de mettre à jour notre fichier JDL afin de conserver une source unique source de description de l'ensemble de l'application et de profiter par exemple du rendu du [JDL Studio](https://start.jhipster.tech/jdl-studio).

![JDL Studio](/images/JDL_Studio.png)

Il est tout à fait possible de modifier le fichier JDL et de le ré-importer en exécutant à nouveau la commande ``jhipster jdl``. Toutefois, il y a plusieurs subtilités auxquelles faire attention pour que cela fonctionne correctement sur la durée :
- la génération des changelogs liquibase
- la conservation des éventuelles modifications apportées au code source généré

## Gestion des changelogs Liquibase

Par défaut, la version actuelle de JHipster re-génère les changelogs initiaux du schéma de base de donnée. Cela ne pose pas de problème tant que l'application n'est pas en production, mais risque de le devenir rapidement.

Ajoutons par exemple un champ ``note`` à notre entité ``PetitDej`` dans le fichier ``petit_dej_v1.jdl``

````
entity PetitDej {
  date LocalDate required
  commentaire String
  note Integer
}
````

Et ré-importons le fichier JDL

    jhipster jdl petit_dej_v1.jdl


On constate que JHipster a modifié le changelog ``XXX_added_entity_PetitDej.xml`` plutôt que de créer un nouveau changelog. Cette migration va donc échouer sur une base sur laquelle elle a déjà été appliquée.

![Changelog modifié](/images/JDL_import_modified_changelog.png)

Pour résoudre ce problème, l'option ``--incremental-changelog`` a été introduite [depuis la version 7.0.0](https://github.com/jhipster/generator-jhipster/issues/11398) de JHipster. Quoique très peu documentée et malgré [quelques limitations](https://github.com/jhipster/generator-jhipster/issues/11398#issuecomment-803072142), cette option est très pratique car elle nous permet de ré-importer notre fichier JDL en générant cette fois ci un nouveau changelog :

    jhipster jdl --incremental-changelog petit_dej_v1.jdl

![Changelog ajouté](/images/JDL_import_added_changelog.png)

Il est à noter que cette option fonctionne même si l'application a été créée initialement sans cette option.

Dans le cas où le changelog n'est pas généré automatiquement en raison des limitations mentionnées ci-dessus, il est toujours possible [d'utiliser la commande liquibase:diff](https://www.jhipster.tech/development/#database-updates-with-the-maven-liquibasediff-goal) pour générer le changelog manquant :

    ./mvnw compile liquibase:diff

## Modification du code source généré

La seconde difficulté rencontrée avec les imports successifs de fichiers JDL concerne la modification du code source généré. En effet, lors de l'import d'un fichier JDL, [JHipster re-génère l'ensemble du code source de l'application](https://github.com/jhipster/generator-jhipster/issues/15967) et pas uniquement des entités/relations modifiées dans le fichier JDL. JHipster vérifie certes si le fichier local présente des différences et demande confirmation avant de l'écraser, mais en pratique cela devient rapidement impossible de vérifier tous les fichiers en conflit un par un à chaque import de fichier JDL et on risque donc de perdre par mégarde des changements appliqués sur le code généré.

Vous me direz que la solution à ce problème consiste à ne pas modifier le code source généré par JHipster, et vous avez tout à fait raison. Voir à ce sujet [ce précédent article]({{< ref "jhipster-side-by-side/" >}}) au sujet de l'approche *side-by-side*.

Hélas, il n'est pas toujours possible d'appliquer strictement l'approche *side-by-side*. Voici quelques cas auxquels on peut-être rapidement confrontés :
- utilisation d'une relation OneToMany undirectionnel [qui n'est pas supportée](https://github.com/jhipster/generator-jhipster/issues/1569) par JHipster
- utilisation de champs ``@Embedded`` ou ``@ElementCollection`` [non gérés actuellement ](https://github.com/jhipster/generator-jhipster/issues/6306)

Dans ces deux cas, il est nécessaire de modifier le code source de l'entité générée par JHipster, par exemple [en retirant l'attribut "mappedBy" de l'annotation ``OneToMany``](https://jhipster.ddocs.cn/managing-relationships/#3) ou en ajoutant manuellement le champ de type non supporté.

Cependant, il est possible de résoudre cette difficulté en utilisant une branche git distincte pour réaliser les imports de fichier JDL, un peu dans l'esprit de ce qui est fait [lors d'un upgrade JHipster](https://www.jhipster.tech/upgrading-an-application/#automatic_upgrade).

L'idée consiste à créer un branche dédiée aux imports de fichiers JDL distincte de notre branche de développement et de cherry-picker les modifications appliquées entre deux imports successifs.

Pour l'exemple, repartons de l'application générée avec ``petit_dej_v0.jdl`` et modifions notre entité PetitDej en ajoutant manuellement un champ de type @EmbededCollection :

````java
@ElementCollection
@Column(name = "values")
private Set<Integer> values = new HashSet<>();
````

On commit ensuite ce changement dans notre branche de développement, ici ``master``.

On va à présent importer le fichier jdl ``petit_dej_v1.jdl`` mentionné plus haut, mais cette fois ci dans une branche à part.

On commence donc par créer une branche nommée par exemple ``jdl_import`` à partir du commit initial réalisé par JHipster.

    git branch jdl_import initial_commit_id

Cette branche contient donc exactement le code source généré par JHipster lors de l'import de ``petit_dej_v0.jdl``.

On se place à présent dans cette branch, puis on importe la nouvelle version du fichier JDL :

````
git checkout jdl_import
jhipster jdl --incremental-changelog petit_dej_v1.jdl
````

On peut cette fois répondre sans craintes ``a`` à la question conflict/Overwrite de JHipster pour forcer la re-génération de tous les fichiers sans distinction.

Lorsque la génération est terminée, on commit les changements dans la branche ``jdl_import``. Ce commit contient alors exactement les changements nécessaires entre les deux versions de notre fichier JDL.

![Commit](/images/JDL_import_commit.png)

La dernière étape consiste à cherry-picker ce commit dans notre branche master pour intégrer ces évolutions.

````
git checkout master
git cherry-pick commit_id
````

Et le miracle opère :

![The miracle](/images/JDL_import_cherry-pick.png)

Et notre fichier PetitDej.java contient bien à la fois le champ ``note`` ajouté lors de l'import du fichier JDL ainsi que le champ ``values`` ajouté manuellement.

Il y a peut-être une commande git plus adaptée que le ``cherry-pick``pour intégrer les changements de la branche ``jdl_import`` dans la branche ``master``, je suis preneur de conseils à ce sujet !

L'utilisation de la branche ``jdl_import`` permet donc de concilier l'import des différentes versions de notre fichier JDL au fur et à mesure du développement de notre application et les éventuels modifications spécifiques à apporter au code source généré lorsque l'approche *side-by-side* n'a pas pu être appliquée.

Pour conclure un petit schéma pour résumer tout ça :

![Schéma du process](/images/JDL_import-process.png)
