# Poc of FILEBEAT

We will use :  
- a spring boot application to generate spécific logs
- Elasticsearch and Kibana to store log datas and create dashboard
- filebeat to ship log datas and push them to Elasticseatch

## Run Spring boot Maven project to generate logs
project path : /home/stephane/projets/projets_gitlabCI/gitlabci_01  
on **GITLAB** : [repository ](https://gitlab.com/skeres/gitlabci_01.git)  



## Run Elasticsearch and Kibana using Docker
project path : /home/stephane/projets/projets_elk/filebeat01
on github : here !

**Create docker volume if not already exists
```
docker volume create --name elasticsearch-filebeat
```

**Create docker network if not already exists
```
docker network create --driver=bridge --ip-range=172.28.1.0/24 --subnet=172.28.0.0/16 --gateway=172.28.1.254 my_custom_network
```

**Run Elasticsearch and Kibana
```
docker-compose -f docker-compose.yaml up -d
```

**Test Elasticsearch health and access
```
curl -XGET "http://localhost:9200/_cluster/health?pretty"
```   

or in a browser :  
```
http://localhost:9200/
```

**Test Kibana access
```
localhost:5601/
```

**Stop container without removing them
```
docker-compose -f docker-compose.yaml stop
```

** Restart container if they already exist
```
docker start elastic-filebeat && docker start kibana-filebeat
```

**Stop and remove caontainers
```
docker-compose down
```

## Install Filebeat on your host
source  :https://www.elastic.co/guide/en/beats/filebeat/7.17/filebeat-installation-configuration.html  

project path : /home/stephane/filebeat/filebeat-7.17.9-linux-x86_64
on github : none  

$ mkdir filebeat && cd filebeat  
$ curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.17.9-linux-x86_64.tar.gz  
$ tar xzvf filebeat-7.17.9-linux-x86_64.tar.gz  
$ cd filebeat-7.17.9-linux-x86_64  
$ cp filebeat.yml filebeat.yml.backup_initial_20230207_08h30  
$ cat filebeat.yml  

## How to use Filebeat
sources :   
filebeat : https://www.youtube.com/watch?v=ykuw1piMGa4&t=2123s  
kibane : https://www.youtube.com/watch?v=e1299MWyr98  


**To see a list of available modules**  
$ ./filebeat modules list  

**Test file filebeat.yml**  
$ ./filebeat test config  

**Test output and Elastcisearch access**  
$ /filebeat test output

** Set up : **  
Set up should ony be run once as changes are made in filebeat.yml   
filebeat will contact the output and start creating all resources needed to ship datas in elasticsearch,   
like indexes, indexe paterne and so on. It can take up to 1 minute to proceed  
$ vi filebeat.yml and change input configuration to true   
  enabled: true   

$ ./filebeat setup  
result :  
*Overwriting ILM policy is disabled. Set `setup.ilm.overwrite: true` for enabling.  
*Index setup finished*   
*Loading dashboards (Kibana must be running and reachable)  
Loaded dashboards  
Setting up ML using setup --machine-learning is going to be removed in 8.0.0. Please use the ML app instead.  
See more: https://www.elastic.co/guide/en/machine-learning/current/index.html  
It is not possble to load ML jobs into an Elasticsearch 8.0.0 or newer using the Beat.  
Loaded machine learning job configurations  
Loaded Ingest pipelines  

**Execute filebeat, and ctrl+c to cancel the process
$ ./filebeat -e



## Customize log parsing for data
** Data format log generated with spring boot looks like this  
023-02-07 16:10:44 [http-nio-8080-exec-8] DEBUG : project gitlabci|controller|date|2023-02-07 16:10:44   
2023-02-07 16:10:45 [http-nio-8080-exec-10] DEBUG : project gitlabci|controller|date|2023-02-07 16:10:45  
2023-02-07 16:10:54 [http-nio-8080-exec-1] DEBUG : project gitlabci|controller|hello|2023-02-07 16:10:54  

*In Kibana, using wizard "upload file" with machine learnig/data visualizer we got :*  
%{TIMESTAMP_ISO8601:timestamp} \[.*?\] %{LOGLEVEL:loglevel} : .*? .*?\|.*?\|.*?\|%{TIMESTAMP_ISO8601:extra_timestamp}

*We override it and customize Grock like this*  
%{TIMESTAMP_ISO8601:timestamp} \[%{GREEDYDATA:protocol}\] %{LOGLEVEL:loglevel} : %{GREEDYDATA:project}\|%{GREEDYDATA:classeName}\|%{GREEDYDATA:controllerName}\|%{TIMESTAMP_ISO8601:extra_timestamp}  

*Continue wizard using "Import" button*  
set an index name : springboot  
clicking "advanced" tab, you can see index mapping and Ingest pipeline that elasticsearch will use to import  datas :   

**Mappings :**  
```
{
  "properties": {
    "@timestamp": {
      "type": "date"
    },
    "classeName": {
      "type": "keyword"
    },
    "controllerName": {
      "type": "keyword"
    },
    "extra_timestamp": {
      "type": "date",
      "format": "yyyy-MM-dd HH:mm:ss"
    },
    "loglevel": {
      "type": "keyword"
    },
    "message": {
      "type": "text"
    },
    "project": {
      "type": "keyword"
    },
    "protocol": {
      "type": "keyword"
    }
  }
}
```  

**Ingest Pipeline :**   
```
  "description": "Ingest pipeline created by text structure finder",
  "processors": [
    {
      "grok": {
        "field": "message",
        "patterns": [
          "%{TIMESTAMP_ISO8601:timestamp} \\[%{GREEDYDATA:protocol}\\] %{LOGLEVEL:loglevel} : %{GREEDYDATA:project}\\|%{GREEDYDATA:classeName}\\|%{GREEDYDATA:controllerName}\\|%{TIMESTAMP_ISO8601:extra_timestamp}"
        ]
      }
    },
    {
      "date": {
        "field": "timestamp",
        "timezone": "{{ event.timezone }}",
        "formats": [
          "yyyy-MM-dd HH:mm:ss"
        ]
      }
    },
    {
      "remove": {
        "field": "timestamp"
      }
    }
  ]
}

```  


*A lot of thing then happened :*  
file processed  
Index created  
Ingest pipeline created  
Data uploaded  
Index pattern created  


Go to "ingest pipeline" to see springboot-pipeline : 
```
[
  {
    "grok": {
      "field": "message",
      "patterns": [
        "%{TIMESTAMP_ISO8601:timestamp} \\[%{GREEDYDATA:protocol}\\] %{LOGLEVEL:loglevel} : %{GREEDYDATA:project}\\|%{GREEDYDATA:classeName}\\|%{GREEDYDATA:controllerName}\\|%{TIMESTAMP_ISO8601:extra_timestamp}"
      ]
    }
  },
  {
    "date": {
      "field": "timestamp",
      "timezone": "Europe/Paris",
      "formats": [
        "yyyy-MM-dd HH:mm:ss"
      ]
    }
  },
  {
    "remove": {
      "field": "timestamp"
    }
  }
]
```

## Create Filebeat configuration file
Click on button "Create Filebeat configuration file" and copy/paste content in a new file called : 
specific-filebeat-conf.yaml

change content
- '<add path to your files here>'   by  - /home/stephane/projets/projets_gitlabCI/gitlabci_01/logs/*.log
 hosts: ["<es_url>"]  by  hosts: ["localhost:9200"]


## Run filebeat again

$ ./filebeat -c specific-filebeat-conf.yaml test config
$ ./filebeat -c specific-filebeat-conf.yaml test output
$ ./filebeat -c specific-filebeat-conf.yaml setup

** Tip ** : rm -rf data directory before launching filebeat if you work on existing datas  
Execute :  
$ ./filebeat -c specific-filebeat-conf.yaml -e  
** Tip** : -e is optional and sends output to standard error instead of the configured log output )  
  
Go to Menu analytic/discover and select the good range of date to see your datas; ALL DONe !!!  
** Tip ** : to stop filebeat, type ctrl+c  

