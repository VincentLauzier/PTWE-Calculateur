# dataModel WebA PremierTech

>Menu : 
>- [Contact](#contact)
>- [Objectif du document](#objectif-du-document)
>- [Fonctionnement et techniques de marquage](#fonctionnement-et-techniques-de-marquage)
>	* [dataLayer](#datalayer)
>	* [Attribut data-trk](#attribut-data-trk)
>- [dataModel](#datamodel)
	* [Clés et valeurs](#clés-et-valeurs-datamodel)
	* [Exemple](#exemple-datamodel)
>- [Exploitation du dataModel pour dataLayer](#exploitation-du-datamodel-pour-datalayer)
	* [Events et variables associées](#events-et-variables-associées)
	* [Exemple](#exemple-datalayer.ready)
>- [Exploitation du dataModel pour data-trk](#exploitation-du-datamodel-pour-data-trk)
	* [Fonctions et variables associées](#fonctions-et-variables-associées)
	* [Exemple](#exemple-data-trk)

--------------------
--------------------

## Contact
Analytics et inté technique : 
>Vincent Lauzier
>lauv@premiertech.com
>(819)-919-7308

## Objectif du document

## Tracking page calculateur premiertechaqua.
L’équipe de PTWE et de Digital Strategy ont mandaté l’équipe d’Analyses Digitales afin de préparer un plan de mesure dans le cadre du projet de suivi des données de l’outil Calculateur pour la récupération des eaux de pluie. 
Les détails concernant ce projet d’amélioration sont disponible sur Miro: https://miro.com/app/board/uXjVOraDsTQ=/

## Demande du marketing
Eléments à tracker :
- Formulaires
- Boutons CTA (Buy Now, Get a free quote)
- Sessions
- Pages vues
- Temps de session moyen / Taux d'abandon
- Clics sur le bouton 'Get Results' et 'Submit Again'
- Régions et Types d'utilisations

--------------------
--------------------

## Fonctionnement et techniques de marquage
Le marquage s'appuie sur 2 techniques principales :
- le dataLayer.push
Cette technique est utilisée au chargement de la page, avant le script GTM, ou sur une interaction de l'internaute soumise à validation.
Ex : un ajout panier n'est effectif qu'après validation du stock et vérification du contenu du panier  
Ex : une création de compte n'est effective qu'après validation du formulaire et enregistrement en base

- le data-trk
Il s'agit d'un attribut html qu'on posera sur l'élément html visé uniquement si on cherche à mesurer l'affichage ou le clic sur cet élément.
Ex : pour mesurer un clic sur un menu, on pourra ajouter le data-trk correspondant sur le \<a> des liens du menu


### dataLayer

#### Anatomie
Vous trouverez la documentation Google officielle sur  [ce lien](https://developers.google.com/tag-manager/devguide).  
Le dataLayer est tout d’abord déclaré en haut de \<head> avant le script GTM.

```javascript
<script>
window.dataLayer = window.dataLayer || [];
window.dataLayer.push({
  'event':'datalayer.ready',
  'variable1':'variable1',
  'variable2':'variable2',
  'variable3':{
	  'variable3.variableA':'variable3.variableA',
	  'variable3.variableB':'variable3.variableB'
	  },
  'variable4':[{
	  'variable4[0].variableA':'variable4.variableA',
	  'variable3[0].variableB':'variable4.variableB'
	  },{
	  'variable4[1].variableA':'variable4.variableA',
	  'variable3[1].variableB':'variable4.variableB'
	  }] 
});
</script>
```
*A noter : aucune valeur de paramètre ne peut être vide. Si un paramètre vaut undefined, null ou “”, alors ce paramètre ne sera pas déclaré.*
Attention, s'il semble plus simple de splitter les variables en plusieurs pushs, ce n'est pas recommandé.
Ainsi, il faudra préférer grouper les variables au sein des mêmes pushs.

>**Si plusieurs push sont effectués sur un chargement de page, alors il faudra un ultime push contenant un event. Cet event ciblera un listener GTM qui permettra de savoir quand toutes les variables sont chargées. Ex : un 'event': 'dataLayer.allready' précisera au listener que toutes les variables sont censées être chargées.**

#### Computed state

Si certaines clés doivent attendre d'autres requêtes pour être poussées, elles seront push au moment le plus tôt et idéalement dans un dataLayer.push existant. Par exemple, si un appel Ajax en fiche produit doit alimenter la variable 'price', on préférera envoyer l'intégralité des informations produit après le retour de l'appel Ajax. 

Le dataLayer est un array mais GTM peut l'interpréter comme un simple object, en ne prenant en compte que la dernière occurrence d'une variable.
Par exemple, si le dataLayer est ainsi fait : 
```javascript
<script>
window.dataLayer = window.dataLayer || [];
window.dataLayer.push({
  'event':'dataLayer.ready',
  'page': 'page'
})
</script>
<script>
window.dataLayer = window.dataLayer || [];
window.dataLayer.push({
  'event':'dataLayerUserShopReady',
  'page': 'page2',
  'user': 'user'
})
</script>
<script>
window.dataLayer = window.dataLayer || [];
window.dataLayer.push({
  'event':'dataLayer.allready'
 })
</script>
```
On devrait obtenir un array composé de ces différents pushs.

Néanmoins, GTM peut mettre à plat ces différentes entrées en ne comptabilisant que la dernière valeur d'une clé s'il y a conflit. On l'appellera ce traitement 'computed state'.
Au final, ces différents pushs reviennent à un seul object : 
```javascript
window.dataLayer = {
  'event':'dataLayer.allready',
  'page': 'page2',
  'user': 'user'
}
```
Cet élément est primordial pour la compréhension de l'attendu développé dans ce document.
Prenons un exemple : nous souhaitons tracker le nombre d'enregistrement de carte de fidélité en y ajoutant une propriété donnant l'id de carte de l'internaute.

Si la validation de l'enregistrement d'une carte de fidélité est effective au rechargement de la page, la situation sera simple : 
- le dataLayer de chargement de page est déclaré. Il comprend le numéro de carte de fidélité de l'internaute
- sur le dataLayer.push de l'evenement 'enregistrement-carte-fidélité', nous n'avons pas besoin d'ajouter une variable donnant l'id de carte fid : cette variable est déjà chargée dans l'event dataLayer initial.

Si en revanche, la validation de cet enregistrement de carte intervient au clic sur le bouton, dans ce cas, il faudrait ajouter l'id de carte de fid dans l'event 'enregistrement-carte-fidélité' car cette valeur ne sera pas déjà disponible dans le computed state.

Au final, on retiendra que si la variable est déjà chargée dans la page, nous n'avons pas besoin de la recharger dans un dataLayer.push d'une interaction.
Autre exemple : nous souhaitons obtenir le template d'une page sur tous les éléments de tracking au sein de cette page. Etant donné que la variable est chargée au chargement de la page, nous n'avons pas besoin de la rappeler au clic sur un lien du menu.

>**Les Single Page Apps augmentent largement la difficulté de pertinence du computed state car il faut sans cesse écraser une partie du dataLayer existant pour obtenir des valeurs de variable fiables. Exemple : au 1er chargement de page, une variable template est renseignée. Mais lorsque l'internaute changera d'écran, il faudra overrider cette variable. On pourrait alors penser que la mécanique la plus simple serait un drop total du dataLayer à chaque changement d'écran mais ce n'est pas le cas puisque oneTrust ne republiera pas les variables de statut de consentement si on ne recharge pas son script.**


### Attribut data-trk

#### Anatomie
L'attribut data-trk est posé sur l'élément que l'on souhaite suivre et non sur un élément parent.
S'il est possible de parcourir le DOM pour aller chercher un élément parent, la tâche est rendue plus complexe à maintenir pour l'équipe tracking interne Premier Tech. On gardera donc ce scénario uniquement en cas de non faisabilité technique.

Le data-trk est un json stringifié embarquant une liste de variables.
Voici son modèle :
```html
data-trk="{
	"variable1":"variable1",
	"variable2":"variable2",
	"variable3": {
	"variable3.variableA":"variable3.variableA",
	"variable3.variableB":"variable3.variableB"
		},
	"variable4":[{
	  "variable4[0].variableA":"variable4.variableA",
	  "variable3[0].variableB":"variable4.variableB"
	  },{
	  "variable4[1].variableA":"variable4.variableA",
	  "variable3[1].variableB":"variable4.variableB"
	  }] 
	}"
```
Attention, il est possible que les " à l'intérieur du JSON doivent être encodés via '\&quot;'

Tout data-trk disposera d'une variable nommée 'fonction' qui donnera son type (ex : 'erreur' ou 'cta'...). Les autres variables associées dépendront du dataModel décrit ci-après.


### Normalisation dataLayer et data-trk
Le dataModel dictera le nom, graphie, format, typage et imbrication des variables au sein des dataLayer.push et dataModel.
Exemple : si le dataModel de la variable object 'product' embarque 2 sous variables, le même schéma sera adopté sur tous les dataLayer.push et data-trk qui contiendront la variable produit.

L'idée ici est d'adopter un modèle persistant au sein d'un même site en industrialisant les parsers GTM.
L'objectif actuel est également de déployer le même schéma sur l'ensemble des sites.
Avec une telle logique, il deviendrait alors possible de créer un container GTM tronc commun unique sur l'ensemble des sites passés sur la stack Drupal Premier Tech.
Chaque site de cette stack disposerait alors d'un GTM tronc commun générique chargé d'assurer le tracking de chaque vendor commun (ex : GA4, Facebook, Google Ads...) et d'un autre container chargé d'assurer le tracking spécifique à chaque site.
Dans ce container GTM tronc commun, la variable window.location.hostname deviendrait alors une variable d'environnement. Le tracking serait identique partout. Chaque évol ne demanderait qu'une intervention mineure. Et une table de correspondance basée sur le hostname donnerait l'id de compte GA4 par exemple.

Bien entendu, cette volonté d'industrialiser sera confrontée à de multiples problématiques de spécificité relative à chaque environnement. Nous en avons conscience.

--------------------
--------------------

## dataModel

Le dataModel décrit ci-dessous est, en quelque sorte, le rendu attendu à terme, d'une API de tracking. L'attendu est complexe car il cherche à obtenir, de manière normalisée et intelligible, tous les détails de chaque contenu affiché sur la page.
Les données reprises ici proviennent certainement de plusieurs blocs et origines.
Certains de mes clients choisissent d'ajouter un bloc caché sur la page et ce bloc joue le rôle de passe plat entre les différentes API et bases de données utilisée par le template. Au chargement de la page, chaque bloc est appelé et renvoi des infos à la page parente qui poussera ces infos de manière centralisée, dans un seul dataLayer.push.
S'il est tentant de laisser chaque bloc faire son propre push, n'oublions pas qu'il nous faudra ensuite un dernier push avec un event dédié précisant que toutes les variables dataLayer sont chargées.

Encore une fois, ce qui est décrit ci-après n'est qu'une ébauche de proposition.
Ce draft doit être mis en relation avec l'attendu côté métier et la faisabilité / coût technique. Ces arbitrages engendreront des suppressions / ajouts / modifications de clé/valeur.

###  Clés et valeurs dataModel

Ci-dessous, le dataModel cible afin de renseigner correctement les dataLayer.push et data-trk.
A noter : tous les retours de l'équipe tech seront accueillis avec grand plaisir. Nous ne pouvons pas prévoir tous les use cases et surtout, nous ne connaissons pas le schéma de la base ni l'archi du SI. Dans le même ordre d'idée, si un tech dispose d'une variable non mentionnée ci-dessous mais  qui semble pertinente, qu'il nous en informe et nous mettrons à jour ce document. Plus qu'une simple liste de variables attendues, l'objectif de ce document est de traduire notre vision pour qu'elle soit portée et interprétée par les dévs.

> **A noter : le dataModel suit une méthodologie logique mais également arbitraire. L'idée suivie est ici de partir de 2 objects, 'page' et 'clic' puisque le chargement d'une page et les clics sont les éléments les plus génériques de la navigation web. A partir du moment où l'un de ces 2 éléments devient important, alors il est sorti de l'object générique pour créer son propre modèle.**
> 
Exemple : la demande de devis (dont le suivi est extrêmement important est à la base un clic qui devient un formulaire, qui devient un contact, qui devient un quote.
Dans le même esprit, une modification de mot de passe dans un compte client est à la base un clic qui devient un form, qui devient un account.

Avant de décrire précisément l'attendu, je préfère que nos discussions portent dans un 1er temps sur la 2ème partie de ce chapitre : [l'exemple ci-dessous](#exemple-datamodel).

|Clé|Type|Origine|Valeur|Exemple|
|---|---|---|---|---|
|`user.id`|string|dataLayer ou cookie ou widening vendor ou widening serverside ou widening BI|identifiant du compte, idéalement CRM|'WIL-6821247a6a8'
|`A COMPLETER`|string|work in progress|work in progress|'work in progress'

### Exemple dataModel

Dans l'exemple ci-dessous, product.specifics est un object reprenant 5 valeurs.
Certains corps de métier auront des besoin très spécifiques. Ex : "je veux tracker le poids de la machine que je vends".
Cette mécanique pourra être reprise dans n'importe quelle entrée de ce json en ajoutant un object 'specifics' à la variable concernée. Ex : ajouter un 'specifics' à la variable user ou à la variable page.


```javascript
{
  "page": {
    "environment": {
      "language": "EN",
      "country": "FR",
      "currency": "EUR",
      "entity": "Premier Tech Aqua",
      "env": "prod"
    },
    "template": {
      "tech": "checkout_cart_index",
      "name": "panier"
    },
    "tree": {
      "level1": "checkout",
      "level2": "checkout",
      "level3": "checkout",
      "level4": "checkout",
      "level5": "checkout"
    },
    "category": {
      "level1": "checkout",
      "level2": "tunnel",
      "level3": "panier",
      "level4": "panier",
      "level5": "panier"
    },
    "step": 1
  },
  "user": {
    "logged": true,
    "id": "1140684",
    "type": "client",
    "segmentation": "vip",
    "company": "takumi data",
    "category": "b2b",
    "firstname": {
      "firstname": "vincent",
      "hash": "YWxleGFuZHJlQHRha3VtaS1kYXRhLmZy"
    },
    "lastname": {
      "firstname": "lauzier",
      "hash": "YWxleGFuZHJlQHRha3VtaS1kYXRhLmZy"
    },
    "address": {
      "address": "1540 Bonsecours",
      "zipcode": "J1K2C2",
      "city": "sherbrooke",
      "region": "quebec",
      "country": "canada"
    },
    "email": {
      "adress": "lauv@premiertech.com",
      "hash": "YWxleGFuZHJlQHRha3VtaS1kYXRhLmZy"
    },
    "order": {
      "date": {
        "latest": 1660141442,
        "first": 1660141442
      },
      "totalAmount": 16542
    }
  },
  "store": {
    "id": "S004",
    "city": "sherbrooke",
    "department": "none",
    "region": "estrie",
    "name": "Hillcrest Home Hardware",
    "zipcode": "J1K2C2",
    "type": "partner",
    "action": "select"
  },
  "product": {
    "id": "3448610219493",
    "name": "ecoflo",
    "priceDisplayedWithTax": 14.99,
    "priceWoDiscountWithTax": 29.99,
    "priceDisplayedWoTax": 10.99,
    "priceWoDiscountWoTax": 25.99,
    "quantity": 1,
    "brand": "premiertech-aqua",
    "range": "eaux usees",
    "category": {
      "level1": "products",
      "level2": "eaux usees",
      "level3": "residences",
      "level4": "ecoflo",
      "level5": "biofiltre ecoflo"
    },
    "taxonomy": {
      "level1": "insect control",
      "level2": "insecticide",
      "level3": "ants",
      "level4": "aerosol",
      "level5": "ant & roach killer"
    },
    "specifics": {
      "level1": "option: loading palletizer",
      "level2": "carac: low level",
      "level3": "availability: on quote",
      "level4": "sector: industry",
      "level5": "weight: 3.5t"
    },
    "reviewNb": 4,
    "reviewScore": 4.25,
    "picturesNb": 4,
    "availability": {
      "status": true,
      "detail": "on quote"
    },
    "position": 1
  },
  "article": {
    "id": "7987",
    "name": "why do we need to treat wastewater",
    "author": "laurent D",
    "tree": {
      "level1": "insect control",
      "level2": "insecticide",
      "level3": "ants",
      "level4": "aerosol",
      "level5": "ant & roach killer"
    },
    "taxonomy": {
      "level1": "insect control",
      "level2": "insecticide",
      "level3": "ants",
      "level4": "aerosol",
      "level5": "ant & roach killer"
    },
    "charactersNb": 798,
    "picturesNb": 4,
    "date": {
      "created": 1660141442,
      "updated": 1660141442
    },
    "position": 1
  },
  "promotion": {
    "id": "7987",
    "name": "insect control",
    "creative": "ants and wasps",
    "block": "home slider",
    "position": 1
  },
  "list": {
    "id": "354",
    "name": "ant & roach killer",
    "tree": {
      "level1": "insect control",
      "level2": "insecticide",
      "level3": "ants",
      "level4": "aerosol",
      "level5": "ant & roach killer"
    },
    "taxonomy": {
      "level1": "insect control",
      "level2": "insecticide",
      "level3": "ants",
      "level4": "aerosol",
      "level5": "ant & roach killer"
    },
    "itemNb": 4,
    "products": [
      product
    ],
    "articles": [
      article
    ],
    "promotions": [
      promotion
    ],
    "filter": [{
      "name": "insect control",
      "type": "insecticide",
      "value": true
    }]
  },
  "cart": {
    "offers": [{
      "id": "6786798",
      "name": "15€ offert",
      "description": "15€ offert sur toutes les armoires",
      "amount": 15.00
    }],
    "products": [
      product
    ],
    "cartId": "7583f711-efc6-421b-b6e6-8e87a097193e",
    "orderId": "MB2569934",
    "shipping": {
      "name": "colissimo",
      "feeDisplayed": 0.00,
      "feeWoDiscount": 4.99,
      "time": "retrait en magasin - 2h"
    },
    "payment": {
      "name": "atos_standard"
    },
    "amount": {
      "totalDisplayedWithTax": 14.99,
      "totalDisplayedWoTax": 10.99,
      "totalWoDiscountWithtax": 34.99,
      "totalWoDiscountWotax": 30.99
    }
  },
  "error": {
    "name": "inéligibilité avantage fidélité",
    "category": "avantage fidélité",
    "type": "fonctionnel",
    "message": "cet avantage n’est pas compatible avec votre panier"
  },
  "cta": {
    "name": "j’en profite",
    "category": "fidélité",
    "type": "cta",
    "action": "ajout avantage fidélité",
    "from": "vos avantages fidélité",
    "destination": "https://www.wilsoncontrol.com/insect-control/bed-bugs"
  },
  "click": {
    "name": "bed bugs",
    "category": "top menu",
    "type": "menu",
    "action": "lien",
    "from": "insects+",
    "destination": "https://www.wilsoncontrol.com/insect-control/bed-bugs"
  },
  "header": {
    "name": "bed bugs",
    "category": "top menu",
    "type": "menu",
    "action": "lien",
    "from": "insects+",
    "destination": "https://www.wilsoncontrol.com/insect-control/bed-bugs",
    "level1": "insecticids",
    "level2": "bed bugs range",
    "level3": "bed bugs spray"
  },
  "footer": {
    "name": "bed bugs",
    "category": "top menu",
    "type": "menu",
    "action": "lien",
    "from": "insects+",
    "destination": "https://www.wilsoncontrol.com/insect-control/bed-bugs",
    "level1": "insecticids",
    "level2": "bed bugs range",
    "level3": "bed bugs spray"
  },
  "quote": {
    "name": "wastewater expert",
    "id": "form-7987",
    "tool": "pardot",
    "option": {
      "level1": "project type:residential",
      "level2": "installation location:canada",
      "level3": "already customer:true",
      "level4": "project amount:50000",
      "level5": "installation category:wastewater"
    }
  },
  "contact": {
    "name": "formulaire générique",
    "id": "form-7987",
    "tool": "pardot",
    "contactType": "form",
    "subject": "product malfunction",
    "option": {
      "level1": "project type:residential",
      "level2": "installation location:canada",
      "level3": "already customer:true",
      "level4": "project amount:50000",
      "level5": "installation category:wastewater"
    }
  },
  "calculator": {
    "name": "eaux usées",
    "id": "form-7987",
    "tool": "drupal",
    "option": {
      "level1": "project type:residential",
      "level2": "installation location:canada",
      "level3": "already customer:true",
      "level4": "project amount:50000",
      "level5": "installation category:wastewater"
    }
  },
  "form": {
    "name": "captcha",
    "id": "form-7987",
    "tool": "GCaptcha"
  },
  "video": {
    "name": "low level palletizer presentation",
    "id": "46878",
    "platform": "on premise",
    "type": "product presentation"
  },
  "search": {
    "query": "ant out",
    "category": "product"
  },
  "account": {
    "action": "modification",
    "field": "address",
    "status": "error"
  },
  "action": {
    "name": "picture browse",
    "step": 2
  },
  "checkout": {
    "name": "shipping",
    "step": 2
  },
  "social": {
    "name": "facebook"
  }
}
```

--------------------
--------------------


## Implémentation

### dataLayer page
Les variables dataLayer retrouvables sur toutes les pages du site seront implémentées :
- siteName	
- primaryCategory	
- subCategory1	
- page_name	
- site_language	
- login_user_type	
- page_category	
- page_country	

## dataLayer.push à l'affichage du calculateur

### Code d'implémentation
```javascript
window.dataLayer = window.dataLayer || [];
  window.dataLayer.push({
  "event": 'calculator.display',
  "calculator": {
    "name": "Wastewater recuperation",
    "id": formId,		
    "tool": calculatorTool,	
  }
})
```

### Variables
|Clé|Type|Valeur|Exemple|
|---|---|---|---|
|`formId`|string|identifiant du calculateur si existant|'65468a7a'|
|`calculatorTool`|string|nom de l'outil utilisé pour le calcul|'Drupal'|



## dataLayer.push à la validation du formulaire
Si la page est rechargée, ce push arrivera sur la page suivante.
Si la page n'est pas rechargée, le push arrive à la soumission du formulaire.
Si l'utilisateur clic sur 'Submit Again', le push est relancé. Conditionnel aux 2 situations plus hautes. 

### Code d'implémentation

```javascript
	window.dataLayer = window.dataLayer || [];
	window.dataLayer.push({
	  "event": "calculator.submit",
	  "calculator": {
	    "name": "Wastewater recuperation",
	    "id": formId, 
	    "tool": calculatorTool, 
	    "option": {
	      "level1": administrativeRegion,
	      "level2": roofArea,
	      "level3": numberOfPeople, 
	      "level4": numberOfVehicle, 
	      "level5": wateringArea 
	    }
	  }
	})
```

### Variables
|Clé|Type|Valeur|Exemple|
|---|---|---|---|
|`formId`|string|identifiant du calculateur si existant|'65468a7a'|
|`calculatorTool`|string|nom de l'outil utilisé pour le calcul|'Drupal'|
|`administrativeRegion`|string|nom de la région de provenance de l'utilisateur|'Riviere-du-Loup'|
|`roofArea`|array|Étendue de surface à couvrir en m²|'65m²'|
|`numberOfPeople`|string|Nombre de personnes utilisant le recupérateur d'eaux usées, si non sélectionné ; undefined|'4 adults and children'|
|`numberOfVehicule`|string|'nombre de véhicules qui seront utilisés, si non sélectionné ; undefined|'2 cars or other vehicles'|
|`wateringArea`|array|étendue de surface à couvrir en m², si non sélectionné ; undefined|'100m²|



## dataLayer.push sur chaque erreur de formulaire

### Code d'implémentation
A la soumission du formulaire, si une erreur de champs est rencontrée, alors on envoie le push suivant autant de fois qu'il y a d'erreurs.
```javascript
	window.dataLayer = window.dataLayer || [];
	window.dataLayer.push({
	  "event": "error",
	  "error": {
	    "name": "calculator error",
	    "category": "wastewater calculator",
	    "type": "form",
	    "message": "required fields not valid"
	  }
	})
```

### Variables
|Clé|Type|Valeur|Exemple|
|---|---|---|---|
|`formId`|string|identifiant du calculateur si existant|'65468a7a'|
|`calculatorTool`|string|nom de l'outil utilisé pour le calcul|'Drupal'|
|`event`|string|type d'erreur, ici générique, donc toujours la même valeur|'error'|
|`name`|string|nom de l'erreur,ici générique, donc toujours la même valeur|'wastewater calculator'|
|`type`|string|type d'erreur dans le formulaire,ici générique, donc toujours la même valeur|'form'|
|`message`|string|message envoyé sur la page lorsque des conditions ne sont pas remplies adéquatements ou que des infos sont manquantes|'Required fields not valid/or incomplete'|





## data-trk sur les CTA

Sur chaque CTA, on ajoutera les data-trk suivants :
### Code d'implémentation
```html
data-trk="{
	"role": "cta",
	"cta": {
		"name": ctaName,  
    		"category": "calculateur",
    		"type": "contact",
    		"action": "link",
    		"from": "Recommended product",
    		"destination": clickUrl  
		}
	}"
```

### Variables
|Clé|Type|Valeur|Exemple|
|---|---|---|---|
|`formId`|string|identifiant du calculateur si existant|'65468a7a'|
|`calculatorTool`|string|nom de l'outil utilisé pour le calcul|'Drupal'|
|`role`|string|Identification de la donnée qui sera trackée par le data-trk|'CTA'|
|`name`|string|nom de la variable qui est trackée|'ctaName' 'BUY NOW'|
|`category`|string|la donnée trackée est issue de quelle type de page|'Wastewater calculator'|
|`action`|string|l'action qui est déclanchée par l'interaction avec le cta|'link'|
|`from`|string|nom du bouton qui déclanche l'interaction avec le cta|'Recommended product'|
|`destination`|string|url de la page en lien avec l'interaction avec le cta|'pageUrl'|


#
## Merci !
