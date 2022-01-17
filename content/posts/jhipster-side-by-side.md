---
title: "JHipster Side by Side"
date: 2022-01-14T16:59:26+01:00
draft: false
---

J'ai eu l'occasion par le passé d'utiliser l'excellente plateforme de développement [JHipster](https://www.jhipster.tech/) dans la cadre de [betterave](https://github.com/amischler/betterave), une application web de gestion des inscriptions pour les ateliers et distributions de l'[Amap la Fée des Champs](http://amap-lafeedeschamps.org/) dont je fais partie.

J'avais été épaté par la simplicité et la rapidité et la qualité avec laquelle JHipster permet de mettre en place une application dont le périmètre fonctionnel est bien défini. JHipster permet en effet de décrire son modèle métier grâce à [JDL](https://www.jhipster.tech/jdl/intro), un **domain language** spécifique, et de générer ensuite automatiquement l'ensemble des composants nécessaires pour l'application aussi bien côté serveur que côté client avec une interface en Angular dans mon cas et l'ensemble des fonctionnalités CRUD pour les entités décrites.

Le résultat est une application développée en un temps restreint, qui _juste marche_ depuis maintenant quelques années avec quelques 200 utilisateurs inscrits et un besoin de maintenance quasi nul.

![Aperçu betterave](/images/apercu-betterave.png)

Toutefois, le principal inconvénient qui m'est apparu au fil du temps et des évolutions de l'application vient de la complexité croissante à utiliser le générateur de code de JHipster au fur et à mesure que les fichiers générés ont été modifiés. L'utilisation de Git et des branches, en particulier lors des mises à jour de version de JHipster, permet de gérer cela, mais les merges deviennent de plus en plus complexes et consommateurs de temps...bref on perd un peu l'intérêt initial de JHispter.

A ce stade, rien n'empêche de continuer à faire évoluer son application "à la main" en se passant de JHispter mais c'est tout de même un peu dommage.

Heureusement, un tip [récemment ajouté aux Tips'n'tricks](https://github.com/jhipster/jhipster.github.io/pull/1171): [Combining generation and custom code](https://www.jhipster.tech/tips/035_tip_combine_generation_and_custom_code.html#combining-generation-and-custom-code) de JHipster propose plusieurs approches pour résoudre cette difficulté.
- On peut rapidement éliminer le Pattern #1 qui ne propose rien d'autre que de ne plus utiliser JHipster. 
- Le Pattern #2, qui consiste à séparer complètement le code généré et le code personnalisé, est simple et sûrement efficace mais il mène à produire beaucoup de code inutile/dupliqué et surtout ne permet pas de bénéficier au mieux des éléments générés par JHipster. 
- Le [Pattern #3](https://www.jhipster.tech/tips/035_tip_combine_generation_and_custom_code.html#pattern-3---side-by-side) est quant à lui nettement plus prometteur puisqu'il doit permettre de tirer profit au mieux du code généré par JHipster sans toutefois le modifier en utilisant au maximum l'héritage de classe et les principes de _beans precedence_ de Spring.

La documentation de ce pattern reste cependant assez sommaire pour bien comprendre de quoi il s'agit. En cherchant un peu on trouve rapidement [une présentation de Antonio Goncalves](https://www.youtube.com/watch?v=9WVpwIUEty0) ([slides](https://www.slideshare.net/agoncal/custom-and-generated-code-side-by-side-with-jhipster) et [exemples]()) ainsi  q'[une présentation de David Steinman](https://www.youtube.com/watch?v=Gg5CYoBdpVo) à la JHispter Conf 2019 qui détaillent un peu plus précisément cette approche.

{{< youtube 9WVpwIUEty0 >}}
{{< youtube Gg5CYoBdpVo >}}


Voyons donc ce que cela peut donner en pratique. Pour cet exemple nous allons concevoir une application qui permet à des utilisateurs d'organiser et de s'inscrire à des petit-déjeuners (rien à voir avec la bonne habitude que nous avions avant les temps de pandémie de prendre le traditionnel petit-déjeuner du vendredi matin ensemble avec quelques sociétés voisines du Port de Lille...). 

Le fichier JDL pour notre exemple [est disponible ici](https://github.com/amischler/PetitDej/blob/master/jhipster-jdl.jdl).
 
Commençons par générer l'application avec ce fichier :

     jhipster import-jdl jhipster-jdl.jdl

A ce stade on dispose d'une application qui peut être exécutée et qui permet d'hors et déjà de gérer la création des petit-déjeuners.

Pour démarrer l'application on peut utiliser :

     ./mnvw
     
ou encore mieux, lancer la main class `PetitDejApp` avec son IDE préféré pour profiter des fonctionnalités de rechargement à chaud.
> **Atttention** avec IntellijIDEA le rechargement à chaud ne marche _out of box_ et ça demande un tout petit peu de configuration [comme expliqué dans la documentation de JHispter](https://www.jhipster.tech/configuring-ide-idea/#application-hot-restart-with-spring-boot-devtools)  

Ensuite on démarre le serveur de développement npm :
    
     nmp start
     
S'ouvre alors le navigateur avec l'interface de l'application. Il est déjà possible de visualiser les petit déjeuners, d'en de créer de nouveau

![Liste des petit-déjeuners](/images/PetitDej-liste.png)

ainsi que d'en éditer les paramètres :

![Liste des petit-déjeuners](/images/PetitDej-edition.png)

Convenons toutefois qu'il n'est pas très pratique de devoir cliquer sur "Modifier" puis de sélectionner la liste des participants pour s'inscrire à un petit dej, nous allons donc ajouter un bouton "Participer" sur la liste des petit dejeuners.

Commençons par ajouter la route nécessaire coté serveur, sans modifier le controleur [`PetitDejResource`](https://github.com/amischler/PetitDej/blob/master/src/main/java/com/dooapp/petitdej/web/rest/PetitDejResource.java) généré par JHipster.

Pour ce faire, rien de plus simple, on ajoute simplement un nouveau controleur [`PetitDejExtendedResource`](https://github.com/amischler/PetitDej/blob/master/src/main/java/com/dooapp/petitdej/web/rest/PetitDejExtendedResource.java) qui étend le `PetitDejResource` généré par JHipster et de définir notre nouvelle route permettant de s'inscrire/désinscrire à un petit déjeuner. On modifie également le `RequestMapping` de ce controleur afin de ne pas entrer en conflit avec les routes du contrôleur généré automatiquement :

````java
@RestController
@RequestMapping("/api/v1")
@Transactional
public class PetitDejExtendedResource extends PetitDejResource {

    private final Logger log = LoggerFactory.getLogger(PetitDejResource.class);

    private static final String ENTITY_NAME = "petitDej";

    @Value("${jhipster.clientApp.name}")
    private String applicationName;

    private final PetitDejRepository petitDejRepository;

    private final UserService userService;

    public PetitDejExtendedResource(PetitDejRepository petitDejRepository, UserService userService) {
        super(petitDejRepository);
        this.petitDejRepository = petitDejRepository;
        this.userService = userService;
    }

    /**
     * {@code PUT  /petit-dejs/participate/:id} : Switch participation in an existing petitDej.
     *
     * @param id the id of the petitDej to participate in.
     * @return the {@link ResponseEntity} with status {@code 200 (OK)} and with body the updated petitDej,
     * or with status {@code 400 (Bad Request)} if the petitDej is not valid,
     * or with status {@code 500 (Internal Server Error)} if the petitDej couldn't be updated.
     * @throws URISyntaxException if the Location URI syntax is incorrect.
     */
    @PutMapping("/petit-dejs/participate/{id}")
    public ResponseEntity<PetitDej> switchParticipationInPetitDej(
        @PathVariable(value = "id", required = false) final Long id) {
        final User user = userService.getUserWithAuthorities().get();
        log.debug("REST request of {} to participate in PetitDej : {}", user.getLogin(), id);

        if (!petitDejRepository.existsById(id)) {
            throw new BadRequestAlertException("Entity not found", ENTITY_NAME, "idnotfound");
        }
        PetitDej petitDej = petitDejRepository.getById(id);
        if (!petitDej.getParticipants().contains(user)) {
            petitDej.getParticipants().add(user);
        } else {
            petitDej.getParticipants().remove(user);
        }
        PetitDej result = petitDejRepository.save(petitDej);
        return ResponseEntity
            .ok()
            .headers(HeaderUtil.createEntityUpdateAlert(applicationName, true, ENTITY_NAME, petitDej.getId().toString()))
            .body(result);
    }

}
````


On peut alors vérifier que notre contrôleur étendu et la nouvelle route sont bien disponible dans l'API via l'interface Administration > API générée automatiquement par JHispter grâce à Swagger. Il est également possible de tester immédiatement cette route depuis l'interface Swagger.

![API Swagger](/images/PetitDej-swagger.png)

> Dans sa présentation, David Steinman va encore plus loin puisqu'il étend également les services et repositories générés par JHipster, mais dans notre exemple ça n'est pas nécessaire. C'est toutefois très utile pour des cas plus avancés.

Attaquons nous à présent au côté front qui s'annonce un peu plus complexe que du côté Spring. Pour ce faire il est nécessaire d'overrider le composant [petit-dej.component](https://github.com/amischler/PetitDej/tree/master/src/main/webapp/app/entities/petit-dej/list) génré automatiquement par JHipster.

Commençons par ajouter un nouveau module Angular [`petit-dej-extended.module`](https://github.com/amischler/PetitDej/tree/master/src/main/webapp/app/entities/petit-dej-extended/petit-dej-extended.module.ts)

````
import { NgModule } from '@angular/core';
import { SharedModule } from 'app/shared/shared.module';
import { PetitDejComponent } from '../petit-dej/list/petit-dej.component';
import { PetitDejExtendedComponent } from './petit-dej-extended.component';
import { PetitDejDetailComponent } from '../petit-dej/detail/petit-dej-detail.component';
import { PetitDejUpdateComponent } from '../petit-dej/update/petit-dej-update.component';
import { PetitDejDeleteDialogComponent } from '../petit-dej/delete/petit-dej-delete-dialog.component';
import { PetitDejExtendedRoutingModule } from './petit-dej-extended-routing.module';

@NgModule({
  imports: [SharedModule, PetitDejExtendedRoutingModule],
  declarations: [PetitDejComponent, PetitDejExtendedComponent, PetitDejDetailComponent, PetitDejUpdateComponent, PetitDejDeleteDialogComponent],
  entryComponents: [PetitDejDeleteDialogComponent],
})
export class PetitDejExtendedModule {}
````

Notez que l'on déclare dans ce module un composant `PetitDejExtendedComponent` en plus des composants déjà générés par JHipster et que nous importons un module de routing personnalisé `PetitDejExtendedRoutingModule`.

Nous pouvons à présent ajouter notre module de routing personnalisé qui modifie uniquement le composant à utiliser pour le path `''`. Pour le reste, il faut hélas faire ici un copier-coller du code du module de routing initial.

```
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';

import { UserRouteAccessService } from 'app/core/auth/user-route-access.service';
import { PetitDejExtendedComponent } from './petit-dej-extended.component';
import { PetitDejDetailComponent } from '../petit-dej/detail/petit-dej-detail.component';
import { PetitDejUpdateComponent } from '../petit-dej/update/petit-dej-update.component';
import { PetitDejRoutingResolveService } from '../petit-dej/route/petit-dej-routing-resolve.service';

const petitDejRoute: Routes = [
  {
    path: '',
    component: PetitDejExtendedComponent,
    canActivate: [UserRouteAccessService],
  },
  {
    path: ':id/view',
    component: PetitDejDetailComponent,
    resolve: {
      petitDej: PetitDejRoutingResolveService,
    },
    canActivate: [UserRouteAccessService],
  },
  {
    path: 'new',
    component: PetitDejUpdateComponent,
    resolve: {
      petitDej: PetitDejRoutingResolveService,
    },
    canActivate: [UserRouteAccessService],
  },
  {
    path: ':id/edit',
    component: PetitDejUpdateComponent,
    resolve: {
      petitDej: PetitDejRoutingResolveService,
    },
    canActivate: [UserRouteAccessService],
  },
];

@NgModule({
  imports: [RouterModule.forChild(petitDejRoute)],
  exports: [RouterModule],
})
export class PetitDejExtendedRoutingModule {}
````

On peut à présent ajouter le code de notre composant [`petit-dej-extended.component.ts`](https://github.com/amischler/PetitDej/tree/master/src/main/webapp/app/entities/petit-dej-extended/petit-dej-extended.component.ts). Cette fois, on peut utiliser de l'héritage et étendre `PetitDejComponent`.

> Attention, comme le précise David Steiman, il est nécessaire de déclarer que notre composant implémente `OnInit` et d'implémenter la méthode correspondante pour qu'elle soit appelé par Angular - un mystère de l'héritage Typescript/Angular que je n'ai pas creusé ici

````
import { Component, OnInit } from '@angular/core';
import { NgbModal } from '@ng-bootstrap/ng-bootstrap';

import { IPetitDej } from '../petit-dej/petit-dej.model';
import { PetitDejExtendedService } from './petit-dej-extended.service';
import { PetitDejComponent } from '../petit-dej/list/petit-dej.component';

@Component({
  selector: 'jhi-petit-dej',
  templateUrl: './petit-dej-extended.component.html',
})
export class PetitDejExtendedComponent extends PetitDejComponent implements OnInit {

  constructor(protected petitDejService: PetitDejExtendedService, protected modalService: NgbModal) {
    super(petitDejService, modalService);
  }

  ngOnInit(): void {
      super.ngOnInit();
  }

  switchParticipation(petitDej: IPetitDej): void {
    this.petitDejService.switchParticipation(petitDej).subscribe(() => this.loadAll());
  }

}

````

Remarquez que nous utilisons ici une version étendue de `PetitDejService` afin de pouvoir implémenter la méthode nous permettant de modifier la participation de l'utilisateur connecté à un petit déjeuner. Ici aussi on peut utiliser de l'héritage pour définir ce `PetitDejExtendedService` :

````
import { Injectable } from '@angular/core';
import { HttpClient, HttpResponse } from '@angular/common/http';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';
import dayjs from 'dayjs/esm';

import { isPresent } from 'app/core/util/operators';
import { DATE_FORMAT } from 'app/config/input.constants';
import { ApplicationConfigService } from 'app/core/config/application-config.service';
import { createRequestOption } from 'app/core/request/request-util';
import { IPetitDej, getPetitDejIdentifier } from '../petit-dej/petit-dej.model';
import { PetitDejService } from '../petit-dej/service/petit-dej.service';

export type EntityResponseType = HttpResponse<IPetitDej>;
export type EntityArrayResponseType = HttpResponse<IPetitDej[]>;

@Injectable({ providedIn: 'root' })
export class PetitDejExtendedService extends PetitDejService {

  protected resourceUrl = this.applicationConfigService.getEndpointFor('api/v1/petit-dejs');

  constructor(protected http: HttpClient, protected applicationConfigService: ApplicationConfigService) {
    super(http, applicationConfigService);
  }

  switchParticipation(petitDej: IPetitDej): Observable<EntityResponseType> {
    return this.http
          .get<IPetitDej>(`${this.resourceUrl}/participate/${getPetitDejIdentifier(petitDej) as number}`, { observe: 'response' })
          .pipe(map((res: EntityResponseType) => this.convertDateFromServer(res)));
  }

}
````

Enfin nous pouvons ajouter le template html [`petit-dej-extended.component.html`](https://github.com/amischler/PetitDej/tree/master/src/main/webapp/app/entities/petit-dej-extended/petit-dej-extended.component.html) de notre composant personnalisé pour ajouter le bouton de participation à la vue, dont voici l'extrait pertinent. Pour le reste il s'agit malheureusement d'un copier-coller du template initial.

````html
[...]
<button type="submit" 
  (click)="switchParticipation(petitDej)" 
  class="btn btn-primary btn-sm" 
  data-cy="switchParticipationButton">
    <fa-icon icon="plus"></fa-icon>
    <span 
      class="d-none d-md-inline" 
      jhiTranslate="petitDejApp.petitDejExtended.switchParticipation">Participer</span>
</button>
[...]
````

Ce template contient une clé personnalisé qu'il est nécessaire d'ajouter dans le dossier i18n des locales attendues. Dans notre cas, nous ajoutons un fichier [`PetitDejExtended.json`](https://github.com/amischler/PetitDej/tree/master/src/main/webapp/i18n/fr/PetitDejExtended.json) avec notre nouvelle clé :

````json
{
  "petitDejApp": {
    "petitDejExtended": {
      "switchParticipation": "Participer"
    }
  }
}
````

A ce stade nous avons créé tous les fichiers nécessaires à notre composant petit-dej-extended. Notez que nous n'avons jusque là modifié aucun des fichiers générés par JHispter.

![Module petit-dej-extended](/images/PetitDej-files.png)

Il nous reste à présent à "brancher" ce nouveau composant en lieu et place du composant d'origine dans notre application.

Pour ce faire il est hélas nécessaire cette fois de modifier l'un des fichiers générés par JHipster [`entity-routing.module.ts`](https://github.com/amischler/PetitDej/tree/master/src/main/webapp/app/entities/entity-routing.module.ts) :

````
import { NgModule } from '@angular/core';
import { RouterModule } from '@angular/router';

@NgModule({
  imports: [
    RouterModule.forChild([
      {
        path: 'petit-dej',
        data: { pageTitle: 'petitDejApp.petitDej.home.title' },
        loadChildren: () => import('./petit-dej-extended/petit-dej-extended.module').then(m => m.PetitDejExtendedModule),
      },
      {
        path: 'lieu',
        data: { pageTitle: 'petitDejApp.lieu.home.title' },
        loadChildren: () => import('./lieu/lieu.module').then(m => m.LieuModule),
      },
      /* jhipster-needle-add-entity-route - JHipster will add entity modules routes here */
    ]),
  ],
})
export class EntityRoutingModule {}
````

Nous indiquons ici à Angular de charger notre module personnalisé plutôt que le module généré par JHipster.

A ce stade, nous pouvons vérifier que notre composant s'affiche correctement dans l'interface et que le bouton est bien fonctionnel :

![Vue avec bouton](/images/PetitDej-extended.png)

> En option - mais cela n'était pas nécessaire dans notre exemple - il est également possible d'overrider globalement le service injecté `PetitDejService`par `PetitDejExtendedService`. De cette manière il est possible de modifier globalement le comportement des routes de bases du service dans les composants d'origine sans devoir étendre ces composants. Dans ce cas, il faut ajouter une directive `providers`dans le fichier app.module.ts :

````
providers: [
    Title,
    { provide: LOCALE_ID, useValue: 'fr' },
    { provide: NgbDateAdapter, useClass: NgbDateDayjsAdapter },
    { provide: PetitDejService, useClass: PetitDejExtendedService },
    httpInterceptorProviders,
  ]
````


En conclusion :

- le côté back est particulièrement simple à étendre en approche "side-by-side" et n'a demandé aucune modification des fichiers générés par JHipster
- le côté front a été plus laborieux, j'ai perdu pas mal de temps avec des messages d'erreurs parfois peu compréhensibles de Typescript/Angular. La documentation de JHispter mériterait peut-être un peu plus de précisions à ce sujet, même s'il s'agit de thématiques plus spécifiques à Angular
- il est nécessaire de modifier quelques fichiers générés côté front, qui seront écrasés lors de la re-génération. Mais il est peut-être possible d'éviter cela [en configurant bien le .yo-resolve](https://github.com/jhipster/generator-jhipster/issues/12497#issuecomment-864456369) ?

Enfin il est à noter qu'une [Pull request]( https://github.com/jhipster/generator-jhipster/issues/12497) est en discussion pour inclure des outils facilitant la mise en place d'une approche Side by side dans JHipster, ce qui serait plutôt chouette.
