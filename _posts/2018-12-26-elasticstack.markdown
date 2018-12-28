---
title: "Déploiement d'Elastic Stack"
date: 2018-12-26 00:00:00 +001
layout: post
author: Mehdi EL KOUHEN
description: Déploiement d'Elastic Stack
toc: false
---

Nous avons intégré la [Stack Elastic](https://www.elastic.co/fr/products) pour gérer la centralisation des logs et monitoring des applications déployées dans notre Usine.

## Composants Installés

La Stack Elastic comporte un grand nombre de composants à installer; composants qui répondent à des besoins variés.

Pour répondre à nos besoin de centralisation des logs et monitoring nous avons choisi d'installer les composants suivants :

* filebeat : remontée de logs fichiers vers elasticsearch
* metricbeat  : remontée de métriques applicatives et système vers elasticsearch
* elasticsearch : base de données de la Suite Elastic
* kibana : IHM d'analyse des données (log + monitoring)
* elasticsearch-curator : suppression des vieux index elasticsearch (> 1 semaine) 

## Chart elasticstack

Pour déployer la stack dans notre cluster Kubernetes, nous utilisons le [packaging Helm](https://helm.sh/).

### Gestion des dépendances

Notre chart se limite en grande partie à référencer (via le fichier *requirements.yaml*) les charts à déployer.

```yaml
dependencies:
  - name: kube-state-metrics [1]
    version: "0.12.1"
    repository: https://kubernetes-charts.storage.googleapis.com
  - name: filebeat
    version: "1.1.1"
    repository: https://kubernetes-charts.storage.googleapis.com
  - name: metricbeat
    version: "0.4.2"
    repository: https://kubernetes-charts.storage.googleapis.com
  - name: elasticsearch
    version: "1.15.1"
    repository: https://kubernetes-charts.storage.googleapis.com
  - name: kibana
    version: "1.1.0"
    repository: https://kubernetes-charts.storage.googleapis.com
  - name: elasticsearch-curator
    version: "1.0.1"
    repository: https://kubernetes-charts.storage.googleapis.com
```

* [1] metricbeat remonte des métriques du cluster exposées par l'agent [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics).  

### Configuration des dépendances

Nous avons surchargé la configuration des charts installés dans le fichier values.yaml.

La surcharge des values d'un chart se fait via un arbre yaml qui porte le nom du composant.

Pour exemple, nous avons surchargé ci-dessous l'objet config défini dans le fichier values du chart filebeat.

```yaml
filebeat:
  config:
    output:
      file:
        enabled: false [1]
      elasticsearch:
        hosts: [ 'elasticstack-elasticsearch-client:9200' ] [2]
        pipeline: opus_log  [3]
    setup:
      kibana:
        host: elasticstack-kibana:80 [4]
```             

* [1] Désactivation de l'envoie des logs filebeat vers des fichiers
* [2] Activation de l'envoie des logs filebeat vers ElasticSearch
* [3] Activation du pipeline d'ingestion (nommé opus_log) sur les logs envoyés vers ElasticSearch
* [4] Configuration de Kibana pour Filebeat (création des Dashboards, ...)