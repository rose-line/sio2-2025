# AP3 - All4Sport - Description des besoins

ALL4SPORT souhaite la mise en production d'une application mobile interne pour les caristes qui travaillent dans ses entrepôts.

## Les fonctionnalités souhaitées

Un ensemble de fonctionnalités ont été préalablement identifiées avec le client ALL4SPORT. Elles ont été regroupées en modules pour faciliter le partitionnement du développement de l'application mobile. Une priorisation globale des modules a également été définie en accord avec le client.

Les spécifications des modules et leur priorité seront cependant sujets à de potentielles modifications en fonction des retours du client au cours de l'avancement du projet. Le client n'exclue pas non plus la possibilité de découvrir et d'ajouter de nouveaux modules en fonction des retours des responsables-entrepôt et des caristes qui vont tester les _beta-releases_ de l'application. Une méthodologie agile sera donc de mise pour gérer ces évolutions potentielles.

NB : les priorités sont ordonnées de 1 à 3, 1 étant la priorité la plus élevée.

### Module 1 - Connexion des caristes

_Priorité : 2_

Plusieurs rôles pourront être spécifiés pour les utilisateurs des différentes applications accédant aux données centralisées de ALL4SPORT, avec des droits d'accès différents.

En l'occurrence, pour cette application mobile, un écran de connexion doit permettre uniquement à des comptes « *cariste* » de l'utiliser.

### Module 2 - Flash des QR code

_Priorité : 3_

On souhaite pouvoir scanner un QR Code sur une palette de produits, afin d'éviter la saisie manuelle des caristes, la lenteur et les erreurs de saisie qui en résultent.

Le QR Code représente la référence-produit du fournisseur. Celle-ci est différente de la référence-produit utilisée en interne chez ALL4SPORT. Il faudra donc effectuer un rapprochement entre la référence fournisseur et la référence interne pour correctement identifier le produit scanné.

### Module 3 - Géolocalisation

_Priorité : 3_

On considère que le smartphone des caristes est géolocalisé et possède une connexion réseau. Il n’existe qu’un seul entrepôt par ville, et celui-ci sera identifié par le nom de la ville elle-même.

L'objectif de ce module est de localiser la ville actuelle, et donc l'entrepôt de travail du cariste.

Cette information devra être obtenue automatiquement par l’application mobile, dès son lancement. Elle sera utilisée par la suite dans les différents modules qui en ont besoin. Dans le cas où le service de géolocalisation ne serait pas disponible (ou si celle-ci échoue), l’utilisateur sera contraint de saisir manuellement son entrepôt.

### Module 4 - Arrivage de stock

_Priorité : 1_

L’application mobile doit permettre au cariste de rentrer la quantité de stock en arrivage pour un produit donné. À cette fin, un écran devra permettre de saisir :

- la référence ALL4SPORT du produit concerné (récupérée grâce au QR code scanné si module 2 fonctionnel) ;
- l’entrepôt (récupéré automatiquement par géolocalisation si module 3 fonctionnel) ;
- la quantité effective.

### Module 5 - Synchronisation des stocks

_Priorité : 1_

L'application doit permettre d'envoyer les informations d'arrivage locales (module 4) sur le serveur web de ALL4SPORT afin de mettre à jour les stocks.

### Module 6 - Liste des produits

_Priorité : 1_

On souhaite pouvoir afficher dans l'application la liste de tous les produits de l'entrepôt actuel, avec les informations suivantes :

- référence produit ;
- désignation produit ;
- quantité en stock.

### Module 7 - Mode déconnecté

_Priorité : 3_

L'application doit être capable de fonctionner en mode déconnecté, c'est-à-dire sans connexion réseau. Les données saisies par les caristes en mode déconnecté devront être synchronisées avec le serveur web dès que la connexion sera rétablie. *Pour ce module*, vous devrez proposer une solution de persistance des données en local.

## Contraintes fonctionnelles

- L'application doit être ergonomique et intuitive pour les caristes, qui ne sont pas nécessairement des utilisateurs experts en informatique.
- L'application doit être réactive et rapide, pour permettre aux caristes de travailler efficacement.

- L'application doit être sécurisée, pour garantir la confidentialité des données des clients de ALL4SPORT.
- L'application doit être maintenable, pour permettre d'ajouter de nouveaux modules ou de corriger des bugs facilement.
- L'application doit être testable, pour garantir la qualité du logiciel avant sa mise en production.

## Contraintes techniques

- La technologie de développement mobile est libre
- L'utilisation d'un SGBD en local n'est pas souhaitée (exception éventuelle : module 7)
- La persistence des données est assurée par un serveur web qui sera l'objet d'un autre projet (exception : module 7)
- Les échanges de données entre l'app mobile et le serveur se font grâce à une API de type REST
- Le format JSON doit être utilisé pour l'échange des données, que ce soit :
  - les données récupérées depuis l'API REST par l'app mobile
  - les données envoyées par l'app mobile vers le webservice
- Le code source doit être versionné sur un dépôt Git
