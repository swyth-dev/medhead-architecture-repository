---
id: architecture-report

title: Rapport des résultats de la PoC
---

# Rapport des résultats de la PoC

## Sommaire

- [Rapport des résultats de la PoC](#rapport-des-résultats-de-la-poc)
  - [Sommaire](#sommaire)
  - [Objet de document](#objet-de-document)
  - [Historique de révision](#historique-de-révision)
  - [Introduction](#introduction)
    - [Contexte](#contexte)
    - [Objectif de la PoC](#objectif-de-la-poc)
  - [Description du projet](#description-du-projet)
    - [Architecture logicielle](#architecture-logicielle)
    - [Stack technologique](#stack-technologique)
    - [Composants de la PoC](#composants-de-la-poc)
    - [Gestion des migrations de données](#gestion-des-migrations-de-données)
    - [Gestion des branches](#gestion-des-branches)
    - [Sécurité](#sécurité)
  - [Intégration Continue](#intégration-continue)
    - [Tests unitaires](#tests-unitaires)
    - [Tests d'intégration (Interconnexion, client d'API)](#tests-dintégration-interconnexion-client-dapi)
    - [Tests de charge avec JMeter](#tests-de-charge-avec-jmeter)
    - [Pipeline d'intégration continue Github Actions](#pipeline-dintégration-continue-github-actions)
    - [Analyse statique avec SonarQube (SAST)](#analyse-statique-avec-sonarqube-sast)
  - [Résultats et enseignements de la PoC](#résultats-et-enseignements-de-la-poc)
    - [Conclusions](#conclusions)
    - [Enseignements](#enseignements)
    - [Recommandations pour la mise en production](#recommandations-pour-la-mise-en-production)

## Objet de document

Ce document détaille l'architecture logicielle retenue, les décisions architecturales, les choix technologiques et détails d'implémentation pour la preuve de concept. Ce document reprend le contexte dans le quel ce PoC a été mené et amène ses conclusions quand à la faisabilité d'un système de réservation d'urgence.

## Historique de révision

| Numéro de version | Auteur   | Description        | Date de modification |
| ----------------- | -------- | ------------------ | -------------------- |
| 1.0.0             | Y. Talon | Rédaction initiale | 08/04/2025           |

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

- **Langages** : Java, Typescript, HTML, CSS
- **Frameworks** : Spring Boot, Spring Cloud, Angular 19
- **Tests** : Junit, JMeter
- **Build** : Maven, Vite
- **Intégration continue** : Git, Github Actions,
- **Base de données** : PostgreSQL
- **Event Driven** : Kafka

### Composants de la PoC

- **Netflix Eureka Discovery Server** : Composant clé dans l'architecture micro-service, il permet de :

  - Permet aux services de se trouver et de communiquer entre eux sans avoir à coder en dur le nom d'hôte et le port.

  - Assurer le load balancing entre plusieurs instances d'un même service dans le cadre d'une mise à l'échelle.

- **Spring Cloud Gateway** : Centralise les requêtes HTTP entrantes et sortantes de sorte à offrir 1 seul point d'entrée pour les clients d'API de la PoC (App frontend, client d'API, tierce partie) - Dispose d'une **configuration de route** pour fournir aux clients d'API une interface homogène, sans besoin de spécifier le nom du micro service responsable des endpoints à consommer.

Example : `https://mydomain.com/v1/hospitals` est routé en interne vers `https://mydomain.com/hospital-service/v1/hospitals` 

Les services sont contactés grâce à leur nom et non leur URL. C'est grâce au Discovery Server que les URL des micro services sont connus. De cette manière, ils peuvent être mis à l'échelle en routant dynamiquement les requêtes pour répartir la charge.

- **Hospital Service** : Regroupe le domaine et les sous-domaine liés aux hopitaux : détails sur spécialisations médicales, hopitaux, disponibilité d'un lit, prise en compte

  - Agit en tant que **Consumer** : S'abonne à un topic d'événement "Bed-reservation-booked" afin de décrémenter le nombre de lit disponible en fonction de l'hopital et la spécialisation médicale.

- **Emergency Service** : Responsable de la prise de rendez-vous. Ce service communique via FeignClient à Hospital Service pour connaître les informations les hopitau et spécialisations, mais également pour connaître la dispnobilité d'un lit avant un réservation

  - Agit en tant que **Producer** : A chaque réservation complétée, le service génère un évènement envoyé aux broker d'évènement (kafka dans notre cas) afin qu'il soit consommés par d'autres services (Hospital service, outils de data analyse par exemple).

- **Message Broker Kafka** : Permet aux services d'échanger des informations via des événements. De cette manière plusieurs services interdépendants peuvent échanger en temps réels des informations sans que ces derniers soit développés dans le même langages ou hébergés sur la même plateforme.

- **Base de donnée relationnelle PostgreSQL** : Responsable de la persistance des données liées à chaque domaine ou sous-domaine métier. En Production, chaque microservice dispose de sa propre instance de base de donnée. Pour l'environnement de développement, une seule base de donné est démarrée, mais chaque service initialise et utilise son propre schéma, isolé des données des autres services.

### Gestion des migrations de données

La gestion des migration de la couche de persistance est déléguée à un outil dédié, Liquibase. Avec cet outil de **Database Change Management**, nous facilitons ainsi l'intégration continue et la refactorisation de la base de données et ses tables.

Notre composant de persistance n'est plus qu'un couche abstraite qui **lie nos données persistées avec les entités de notre domaine métier.**

### Gestion des branches

Dans le cadre de notre démarche d’intégration continue et de livraison continue, nous avons retenu la stratégie de gestion de branche basée sur le **trunk-based development**. Cette approche s’aligne étroitement avec les exigences de livraisons fréquentes, fiables et de haute qualité. Nos principales motivations sont les suivantes :

- **Facilitation de l’intégration continue** : en centralisant les développements sur une branche unique (`main`), les contributions sont intégrées de manière régulière, réduisant ainsi significativement les risques de conflits complexes lors des fusions.

- **Accélération du cycle de livraison** : Un seul code constamment à jour, stable et déployable permet des déploiements plus fréquents, tout en minimisant les retards et les incertitudes en fin de cycle.

- **Renforcement de la qualité par les tests automatisés** : Chaque modification (pull request, merge sur la branche principale) est soumise à une batterie de tests, évaluant rapidement les non-régressions et la stabilité de l'état du système.

- **Amélioration de la collaboration au sein de l’équipe** : le travail sur une base commune favorise une meilleure coordination, une plus grande transparence et une compréhension partagée de l’état du projet.

- **Simplification de la gestion des branches** : En évitant la multiplication des branches longues, les incréments sont rapidement ajoutés à la branche principale, favorisant un historique et un suivi plus claire, et une seule voie pour la mise en production.

### Sécurité

- **La gestion des utilisateurs** n'a pas été implémenté car elle n'a pas été jugée nécessaire afin de démontrer la faisabilité d'un système de réservation d'urgence. Néanmoins, dans un contexte de production, les patients seront amenés à s’authentifier avant d'être en mesure de réserver un lit d’hôpital.

- **Les données médicales,** soumises à d'importantes exigences de conformité (HIPAA), ne sont pas stockées dans les systèmes mis en oeuvre par la PoC. Ces données peuvent être fournis par d'autres systèmes nationaux sans que pour autant elles ne soient stockées dans les systèmes impliqués dans le périmètre de la preuve de concept.

- **Les données personnelles liées à la réservation d'un lit** sont stockées dans les systèmes liés au périmètre de la PoC. Dans des conditions réelles de production, les données des patients ou utilisateurs du service de Santé nationale sont fournies par d'autres systèmes (Identity Provider par exemple).

- **Les données personnelles** telles que le nom, prénom, adresse mail, numéro de téléphone sont traités et conservés dans les systèmes liés à la PoC, mais seulement dans des environnement éphémère comme l'environnement local. Les risques d'impact sur la confidentialité ou l'intégrité des données sont alors faibles durant le traitement et le stockage, conformément **RGPD**.

## Intégration Continue

Dans une démarche Agile, DevOps, de qualité, nous intégrons en continu nos changements grâce à une stratégie de test **Shift Left**. La prise en compte des tests en faite le plus en amont des développement ou durant, de sorte à toujours garder un état déployable en production de notre système.

### Tests unitaires

Les tests unitaires s'assurent que **les méthodes principales de notre code soient validées**. Si ces tests échouent, les mises à jours de notre code principal sont bloquées jusqu'à ce que les test passent.

L'objectif de ces test est de vérifier que nos **_Features_** répondent aux besoins exprimés, ainsi que de tester les **_edge cases_** comme des ressources introuvables.

- Gestion des hopitaux et des disponibilités
  ![Tests liés à HospitalService](https://raw.githubusercontent.com/swyth-dev/medhead-architecture-repository/refs/heads/main/docs/_assets/hospital-tests-screen.png)
  ![Tests liés à MedicalSpecializationService](https://raw.githubusercontent.com/swyth-dev/medhead-architecture-repository/refs/heads/main/docs/_assets/medical-spe-test-screen.png)

- Gestion des réservation
  ![enter image description here](https://raw.githubusercontent.com/swyth-dev/medhead-architecture-repository/refs/heads/main/docs/_assets/bed-reservation-test-screen.png)

### Tests d'intégration (Interconnexion, client d'API)

Les tests d'intégration couvrent le bon fonctionnement de chaque service applicatif ainsi que les points de terminaison REST de notre API. Ces tests sont à la fois automatisés et manuels via un client d'API.

Lorsque nous lançons nos services, nous nous assurons que

- La **base de donnée relationnelle** est accessible est initialisée avec le schéma de données.
- Le **_Binding_ au Message broker** est opérationnelle, pour produire les messages comme pour s'abonner au topic du message broker.

Nous contrôlons manuellement les différents endpoints Rest de notre API grâce à un **client d'API** comme **Postman** ou **Bruno**. De cette manière nous pouvons contrôler que l'APi renvoie correctmeent les données de test.

Cette étape d'assurance quaité peut également être automatisé en prenant en compte les scénarios d'utilisation de nos endpoints que les clients d'API réalisent.

### Tests de charge avec JMeter

![Test de charges d'une réservation de lit avec JMeter](https://raw.githubusercontent.com/swyth-dev/medhead-architecture-repository/refs/heads/main/docs/_assets/load-test-reservation-screen.png)

### Pipeline d'intégration continue Github Actions

![Pipeline d'intégration continue avec Github Actions](https://raw.githubusercontent.com/swyth-dev/medhead-architecture-repository/refs/heads/main/docs/_assets/github-actions-screen.png)

### Analyse statique avec SonarQube (SAST)

![Suivi des métriques de qualité du code avec SonarQube et Cloud](https://raw.githubusercontent.com/swyth-dev/medhead-architecture-repository/refs/heads/main/docs/_assets/sonarcloud-screen.png)

## Résultats et enseignements de la PoC

### Conclusions

A travers cette preuve de concept, nous avons pu prouvé au consortium MedHead qu'une meilleure gestion de la réservation d'urgence est possible.

Cette PoC s'inscrit dans une vision d'intégrer facilement un nouveau système communiquant avec le reste de l'architecture d'entreprise.

A travers cette implémentation de bout en bout, Nous pourrons proposer un nouveau service réduisant considérablement le temps de réservation, tout en augmentant la qualité de service proposée aux usagers.

### Enseignements

Cette preuve de concept a également servi à rendre compte de l'importance des choix de conception dans le développement de systèmes robustes et fiables.

En posant très tôt dans le projet des exigences et des bonnes pratiques, nous nous évitons une maintenance chronophage et couteuse pour faire évoluer nos systèmes et répondre rapidement aux nouveaux besoins exprimés.

La qualité entraîne la rigueur et l'organisation, et ce projet a été l'occasion de mettre en pratique les savoirs faire enseignés pour délivrer une solution répondant à des exigences d'entreprise.

### Recommandations pour la mise en production

- Sécuriser les services et les communications entre les différents composants :

  - Certificats TLS pour chaque composants afin de communiquer via HTTPS
  - Chiffrement des évènements envoyés et reçu par Kafka pour sécuriser des données critiques
  - Authentification des utilisateurs sur service de résevration par un fournisseur d'identité centralisé

- Compléter notre démarche DevOps en automatisant les phase de déploiement

  - Gérer les variables d'environnement pour déployer le système dans différents environnement (test, production)
  - Récupérer les artifacts (builds) de nos services pour les déployer sur un Cloud Provider

- Améliorer la disponibilité et la résilience de nos systèmes
  - Intégrer un orchestrateur de conteneur pour mettre à l'échelle de manière horizontale les services les plus sollicités
  - Implémenter une stratégie de cache pour accélerer considérabement les temps de réponses pour des résultats déjà réquêtés
  - Monitorer les composants à l'aide d'outil de supervision déjà intégrés dans l'architecture d'entreprise de MedHead
