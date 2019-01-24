---
title: "Gestion de secrets & Déploiement Kubernetes"
date: 2019-01-24 00:00:00 +001
layout: post
author: Mehdi EL KOUHEN
description: Gestion de secrets & Déploiement Kubernetes
tags: [ "Helm", "Kubernetes" ]
toc: false
---

Pour déployer une application dans Kubernetes, il faut décrire les ressources de l'application (Deployment, Pod, Service, ...) puis demander à Kubernetes de déployer ces ressources. 

Kubernetes gère des ressources de type *Secret* qui sécurisent la gestion des secrets à l'intérieur du cluster. Mais il reste nécessaire de les gérer par application et par environnement (DEV, RE7, PROD) en dehors du cluster (i.e. dans un coffre fort). 

Dans ce Post, nous proposons une manière de gérer les secrets dans un contexte où :

* Les applications sont déployées dans Kubernetes
* Jenkins est déployé dans Kubernetes
* Les déploiements sont réalisés via Jenkins

## Méthodologie

Nous déployons nos applications avec [Helm](https://helm.sh/). Helm est une solution extensible (via des plugins) de packaging d'applications Kubernetes.
 
Un Chart (ou Package) Helm contient : 

* Un répertoire templates qui contient les templates des ressources Kubernetes à déployer
* Un fichier values.yaml qui contient les valeurs injectées dans les templates (pour la génération des ressources)

L'installation d'un Chart se fait via la ligne de commande *helm*. Dans l'exemple ci-dessous nous surchargeons le fichier values.yaml pour adapter le déploiement à l'environnement de RE7.

````bash
> Installation du chart books-api (trouvé dans le repository softeam-charts)
> helm install --values RE7/values.yaml softeam-charts/books-api 
````

Le plugin *secrets* de Helm gère les fichiers de secrets. Le plugin *secrets* est basé sur l'outil [SOPS](https://github.com/mozilla/sops) pour le chiffrement des fichiers de secret. L'outil *SOPS* chiffre les fichiers en se basant sur le standard PGP (Pretty Good Privacy) de chiffrement.

L'intérêt du chiffrement des fichiers de secrets est la possibilité de les stocker directement dans GIT avec les scripts de déploiement. Et donc les rendre facilement accessibles aux Jobs Jenkins de déploiement.

## Déploiement Poste de Dev

### Création de la clef PGP

La génération de la clef se fait via pgp (gpg sur ubuntu).

````bash
gpg --full-generate-key
````

### Création de la conf SOPS

SOPS permet de gérer sur un même projet différentes clefs PGP : une clef par environnement projet par exemple (RE7, PROD). Ceci permet par exemple de s'assurer que les clefs PGP de PROD ne sont déchiffrables que par l'équipe de PROD.

Pour faire simple, le fichier de configuration *.sops.yaml* ci-dessous ne gère qu'une unique règle de chiffrement : tous les fichiers sont chiffrés avec la clef *59D14BD4108D748C3E9FF048EB31F93C01C4EDDC* (créé à l'étape précédente).

````yaml
# Contenu du fichier SOPS
> cat .sops.yaml
creation_rules:
  - pgp: '59D14BD4108D748C3E9FF048EB31F93C01C4EDDC'
````

### Création d'un fichier de secrets

La création d'un fichier de secrets se faits via *sops*. Le fichier créé est un fichier comportant uniquement des métadonnées SOPS nécessaires pour le chiffrement/déchiffrement.

````bash
> sops -i -e secrets.yaml
````

Le fichier de secrets est ensuite modifiable soit directement via *sops* soit via helm secrets. Les deux outils fonctionnent comme des éditeurs "lignes de commande" (exemple : vi).

````bash
> sops secrets.yaml
> helm secrets secrets.yaml
````

### Déploiement du Chart Helm

Comme montré par le fragment de code ci-dessous, le déploiement d'un chart Helm avec secrets se fait via "helm secrets install" (et non helm install). En effet, "helm secrets" déchiffre le fichier de secrets avant déploiement, déploie le chart puis supprime le fichier de secrets après déploiement.

````bash
# Installation du chart books-api
> helm secrets install --values RE7/secrets.yaml softeam-charts/books-api 
````

## Déploiement Jenkins

### Job Jenkins de Déploiement

Nous implémentons nos Jobs Jenkins via le plugin [kubernetes-plugin](https://github.com/jenkinsci/kubernetes-plugin). Ainsi, l'exécution d'un Job est réalisé par un agent Jenkins déployé dans le cluster Kubernetes.

Dans notre cas, Jenkins est déployé dans le cluster. Il est exécuté via un compte de service kubernetes qui doit avoir le droit de réaliser des déploiements.

### Gestion de la clef PGP

Comme la clef PGP ne peut pas être utilisée sans mot de passe, il n'est pas risqué de la gérer en clair dans Jenkins. Nous stockons donc la clef dans un fichier de configuration géré par Jenkins et nous injectons (à l'exécution) la clef (via *configFileProvider*) dans le job Jenkins de déploiement du chart Helm

Le *configFileProvider* copie le fichier de config identifié *pgp_helm_key* dans le job Jenkins (chemin: *pgp.asc*).

````groovy
configFileProvider([configFile(fileId: 'pgp_helm_key', targetLocation: "pgp.asc")]) {

}
````

### Gestion du mot de passe de la clef

Le mot de passe de la clef doit être géré par Jenkins comme un secret. Nous stockons donc le mot de passe dans un Credential Jenkins et nous injectons le mot de passe (via *withCredentials*) dans le job Jenkins de déploiement du chart Helm.

La commande *withCredentials* génère une variable d'environnement *pgp_helm_pwd* accessible dans le bloc de code associé.

````groovy
withCredentials([string(credentialsId: 'pgp_helm_pwd', variable: 'pgp_helm_pwd')]) {

}
````

### Exemple Complet

L'exemple complet est disponible sur le dépôt [chart-run](https://github.com/SofteamOuest-SoftwareFactory/chart-run).