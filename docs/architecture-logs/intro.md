# Rapport des résultats de la PoC
## Sommaire



## Objet de document

Ce document détaille l'architecture logicielle retenue, les décisions architecturales, les choix technologiques et détails d'implémentation pour la preuve de concept. Ce document reprend le contexte dans le quel ce PoC a été mené et amène ses conclusions quand à la faisabilité d'un système de réservation d'urgence.

## Historique de révision 

| Numéro de version | Auteur | Description | Date de modification |  
|--|--|--|--|
| 1.0.0 | Y. Talon | Rédaction initiale | 08/04/2025 |

## Introduction

### Contexte

Un consortium composé de quatre entreprises leaders s’est formé pour mutualiser leurs efforts, données, applications et feuilles de route, afin de développer une plateforme de nouvelle génération, centrée sur le patient. Celle-ci vise à améliorer les soins de base tout en étant réactive, opérationnelle en temps réel et capable de prendre des décisions critiques en situation d’urgence, en tenant compte de toutes les données disponibles.

### Objectif de la PoC

La mise en œuvre d'une preuve de concept pour le système d'intervention d'urgence en temps réel par l'équipe d'architecture métier du consortium MedHead permettra :
- D’améliorer la qualité des traitements d'urgence et de sauver plus de vies.
- De gagner la confiance des utilisateurs quant à la simplicité d'un tel système.

## Description du projet

### Architecture logicielle

![Diagramme D'architecture logiciel de la PoC](https://raw.githubusercontent.com/swyth-dev/realtime-emergency-response-system/3aadd8408b7008bf03535022f80a301feecd5ee2/doc/diagrams/Medhead_Architecture.png)

### Stack technologique



### Composants de la PoC

- **Netflix Eureka Discovery Server** : Composant clé dans l'architecture micro-service, il permet de :
	- Permet aux services de se trouver et de communiquer entre eux sans avoir à coder en dur le nom d'hôte et le port. 
	- Assurer le load balancing entre plusieurs instances d'un même service dans le cadre d'une mise à l'échelle.
	

- **Spring Cloud Gateway** : Centralise les requêtes HTTP entrantes et sortantes de sorte à offrir 1 seul point d'entrée pour les clients d'API de la PoC (App frontend, client d'API, tierce partie)
	- Dispose d'une **configuration de route** pour fournir aux clients d'API une interface homogène, sans besoin de spécifier le nom du micro service responsable des endpoints à consommer. 
Example : `https://mydomain.com/v1/hospitals` est routé en interne vers `https://mydomain.com/hospital-service/v1/hospitals`
	- Les services sont contactés grâce à leur nom et non leur URL. C'est grâce au Discovery Server que les URL des micro services sont connus. De cette manière, les microservcies peuvent être mis à l'échelle sans se soucier de router dynamiquement les requêtes pour répartir la charge.

- **Hospital Service** : Regroupe le domaine et les sous-domaine liés aux hopitaux : détails sur spécialisations médicales, hopitaux, disponibilité d'un lit, prise en compte
	- Agit en tant que **Consumer** : S'abonne à un topic d'événement "Bed-reservation-booked" afin de décrémenter le nombre de lit disponible en fonction de l'hopital et la spécialisation médicale.

- **Emergency Service** : Responsable de la prise de rendez-vous. Ce service communique via FeignClient à Hospital Service pour connaître les informations les hopitau et spécialisations, mais également pour connaître la dispnobilité d'un lit avant un réservation
	- Agit en tant que **Producer** : A chaque réservation complétée, le service génère un évènement envoyé aux broker d'évènement (kafka dans notre cas) afin qu'il soit consommés par d'autres services (Hospital service, outils de data analyse par exemple).

- **Message Broker Kafka** : Permet aux services d'échanger des informations via des événements. De cette manière plusieurs services interdépendants peuvent échanger en temps réels des informations sans que ces derniers soit développés dans le même langages ou hébergés sur la même plateforme. 

- **Base de donnée relationnelle PostgreSQL** : Responsable de la persistance des données liées à chaque domaine ou sous-domaine métier. En Production, chaque microservice dispose de sa propre instance de base de donnée. Pour l'environnement de développement, une seule base de donné est démarrée, mais chaque service initialise et utilise son propre schéma, isolé des données des autres services.

### Gestion des migrations de données

### Gestion des branches
Dans le cadre de notre démarche d’intégration continue et de livraison continue, nous avons retenu la stratégie de gestion de branche basée sur le **trunk-based development**. Cette approche s’aligne étroitement avec les exigences de livraisons fréquentes, fiables et de haute qualité. Nos principales motivations sont les suivantes :

-   **Facilitation de l’intégration continue** : en centralisant les développements sur une branche unique (`main`), les contributions sont intégrées de manière régulière, réduisant ainsi significativement les risques de conflits complexes lors des fusions.
    
-   **Accélération du cycle de livraison** : Un seul code constamment à jour, stable et déployable permet des déploiements plus fréquents, tout en minimisant les retards et les incertitudes en fin de cycle.
    
-   **Renforcement de la qualité par les tests automatisés** : Chaque modification (pull request, merge sur la branche principale) est soumise à une batterie de tests, évaluant rapidement les non-régressions et la stabilité de l'état du système.
    
-   **Amélioration de la collaboration au sein de l’équipe** : le travail sur une base commune favorise une meilleure coordination, une plus grande transparence et une compréhension partagée de l’état du projet.
    
-   **Simplification de la gestion des branches** : En évitant la multiplication des branches longues, les incréments sont rapidement ajoutés à la branche principale, favorisant un historique et un suivi plus claire, et une seule voie pour la mise en production.

### Sécurité

- La gestion des utilisateurs n'a pas été implémenté car elle n'a pas été jugée nécessaire afin de démontrer la faisabilité d'un système de réservation d'urgence. Néanmoins, dans un contexte de production, les patients seront amenés à s’authentifier avant d'être en mesure de réserver un lit d’hôpital.

- Les données médicales, soumises à d'importantes exigences de conformité (HIPAA), ne sont pas stockées dans les systèmes mis en oeuvre par la PoC. Ces données peuvent être fournis par d'autres systèmes nationaux sans que pour autant elles ne soient stockées dans les systèmes impliqués dans le périmètre de la preuve de concept.

- Les données personnelles liées à la personne qui réserve un lit sont stockées dans les systèmes liés au périmètre de la PoC. Dans des conditions réelles de production, les données des patients ou utilisateurs du service de Santé nationale sont fournies par d'autres systèmes (Identity Provider par exemple).
	- Les données personnelles telles que le nom, prénom, adresse mail, numéro de téléphone sont traités et conservés dans les systèmes liés à la PoC, mais seulement dans des environnement éphémère comme l'environnement local. Les risques d'impact sur la confidentialité ou l'intégrité des données est alors faible.


## Intégration Continue 
### Tests unitaires
### Tests d'intégration 
### Tests de charge
### Workflow Github Actions
### Analyse statique avec SonarQube (SAST)

## Démonstrations
### Parcours utilisateur
### Tests unitaires
### Tests de charge
### Commentaires de Pull Requests (Unit tests, SonarCloud)

## Résultats et enseignements de la PoC
### Conclusions

### Enseignements

### Recommandations pour la mise en production
