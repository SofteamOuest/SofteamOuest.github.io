---
title: "Déploiement d'Elastic Stack"
date: 2018-12-28 00:00:00 +001
layout: post
author: Mehdi EL KOUHEN
description: Déploiement d'Elastic Stack
toc: false
---

Dans cet article, nous expliquons comment nous avons intégré la [Stack Elastic](https://www.elastic.co/) dans notre Usine pour gérer la centralisation des logs et monitoring des applications déployées. 

L'URL de notre kibana est [https://kibana.k8.wildwidewest.xyz](https://kibana.k8.wildwidewest.xyz/). L'accès est sécurisé via login/password. N'hésitez pas à me le demander.

## Composants Elastic Stack

Elastic Stack est composé de différents [composants](https://www.elastic.co/fr/products) qui répondent à des besoins variés.

Pour répondre à nos besoin de centralisation des logs et monitoring nous avons choisi d'installer les composants suivants :

* [filebeat](https://github.com/helm/charts/tree/master/stable/filebeat) : remontée de logs fichiers vers elasticsearch
* [metricbeat](https://github.com/helm/charts/tree/master/stable/metricbeat) : remontée de métriques applicatives et système vers elasticsearch
* [elasticsearch](https://github.com/helm/charts/tree/master/stable/elasticsearch) : base de données de la Suite Elastic
* [elasticsearch-curator](https://github.com/helm/charts/tree/master/stable/elasticsearch-curator) : suppression des index elasticsearch anciens (> 1 semaine)
* [kibana](https://github.com/helm/charts/tree/master/stable/kibana) : IHM d'analyse des données (log + monitoring)
 
## Chart elasticstack

Pour déployer la stack dans notre cluster Kubernetes, nous avons implémenté un chart [Helm](https://helm.sh/).

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

Pour trouver les charts, nous avons utilisé *helm search*.

```bash
# helm search | grep elastic
stable/elastic-stack                 	1.1.0        	6                           	A Helm chart for ELK                                        
stable/elasticsearch                 	1.15.1       	6.5.3                       	Flexible and powerful open source, distributed real-time ...
stable/elasticsearch-curator         	1.0.1        	5.5.4                       	A Helm chart for Elasticsearch Curator                      
stable/elasticsearch-exporter        	0.4.1        	1.0.2                       	Elasticsearch stats exporter for Prometheus                 
stable/fluentd-elasticsearch         	2.0.0        	2.3.2                       	A Fluentd Helm chart for Kubernetes with Elasticsearch ou...

# helm search | grep beat
stable/auditbeat                     	0.4.1        	6.5.3                       	A lightweight shipper to audit the activities of users an...
stable/filebeat                      	1.1.1        	6.5.3                       	A Helm chart to collect Kubernetes logs with filebeat       
stable/heartbeat                     	0.2.0        	6.5.1                       	A Helm chart to periodically check the status of your ser...
stable/metricbeat                    	0.4.2        	6.5.1                       	A Helm chart to collect Kubernetes logs with metricbeat   
```

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