## Docker Centralized Logging with ELK:




### ElasticSearch:


```
mkdir -p /docker_elk/elasticSearch

cd /docker_elk/elasticSearch
```



```
vim elasticsearch.yml


cluster.name: "docker-cluster"

network.host: 0.0.0.0

xpack.license.self_generated.type: basic

#xpack.security.enabled: true
xpack.security.enabled: false

xpack.monitoring.collection.enabled: true
```



```
vim Dockerfile


FROM docker.elastic.co/elasticsearch/elasticsearch:7.15.1

COPY --chown=elasticsearch:elasticsearch ./elasticsearch.yml /usr/share/elasticsearch/config/
```




### Kibana:

```
mkdir -p /docker_elk/kibana

cd /docker_elk/kibana
```



```
vim kibana.yml


server.name: kibana

server.host: "0.0.0.0"

elasticsearch.hosts: [ "http://elasticsearch01:9200" ]

xpack.monitoring.ui.container.elasticsearch.enabled: true

#elasticsearch.username: "kibana_system"
elasticsearch.username: "elastic"
elasticsearch.password: "elastic"
```



```
vim Dockerfile 


FROM docker.elastic.co/kibana/kibana:7.15.1

COPY  ./kibana.yml /usr/share/kibana/config/
```



### Logstash:

```
mkdir -p /docker_elk/logstash

cd /docker_elk/logstash
```



```
vim logstash.yml


http.host: "0.0.0.0"
xpack.monitoring.elasticsearch.hosts: [ "http://elasticsearch01:9200" ]

xpack.monitoring.enabled: false

xpack.monitoring.elasticsearch.username: "elastic"
xpack.monitoring.elasticsearch.password: "elastic"
```



```
vim logstash.conf


input {
  beats {
    port => 5044
  }

  tcp {
     port => 5000
   }
}

filter {
  # add filters here if needed (grok, date, mutate)
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch01:9200"]
    index => "logstash-docker-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "elastic"
  }
}

```




```
vim Dockerfile 


FROM docker.elastic.co/logstash/logstash:7.15.1

COPY ./logstash.yml /usr/share/logstash/config/
COPY ./logstash.conf /usr/share/logstash/pipeline/
```





### Directory layout:

```
tree /docker_elk/

/docker_elk/
├── elasticSearch
│   ├── Dockerfile
│   └── elasticsearch.yml
├── kibana
│   ├── Dockerfile
│   └── kibana.yml
└── logstash
    ├── Dockerfile
    ├── logstash.conf
    └── logstash.yml
```



### Create the Docker Compose File:


```
docker network create elk
```


```
mkdir es01_data
chown -R 1000:0 es01_data
```


### Method-1: 

_Create file `docker-compose.yml`:_
```
services:
 es01:
   build:
     context: ./elasticSearch
     dockerfile: Dockerfile
   container_name: elasticsearch01

   volumes:
     - ./es01_data:/usr/share/elasticsearch/data
   ports:
     - "9200:9200"
     - "9300:9300"

   environment:
     discovery.type: single-node
     ES_JAVA_OPTS: "-Xmx256m -Xms256m"
     ELASTIC_PASSWORD: elastic
   networks:
     - elk

 logstash:
   build:
     context: ./logstash
     dockerfile: Dockerfile
   container_name: logstash01

   #volumes:

   ports:
     - "9600:9600"
     - "5000:5000"
     - "5044:5044"

   environment:
     LS_JAVA_OPTS: "-Xmx256m -Xms256m"

   networks:
     - elk
   depends_on:
     - es01

 kibana:
   build:
     context: ./kibana
     dockerfile: Dockerfile
   container_name: kibana01

   #volumes:

   ports:
     - "5601:5601"

   #environment:
     #SERVER_NAME: kibana

   networks:
     - elk
   depends_on:
     - es01

networks:
 elk:
   driver: bridge
   external: true

```


```
docker compose config
```


```
docker compose up -d
```



#### Verify: 

_For elasticsearch:_
```
curl localhost:9200

curl http://localhost:9200/_cluster/health?pretty
```


```
curl -X GET "http://localhost:9200/_cat/indices?v"
```


_For kibana UI:_
```
http://<host-ip>:5601
```


_For logstash to send a test event to Logstash (tcp json)::_
```
printf '{"message":"hello elk","@timestamp":"'"$(date --iso-8601=seconds)"'"}\n' | nc localhost 5000
```




---
---





### Method-2: (Recommended)


```
vim logstash.yml


http.host: "0.0.0.0"
#xpack.monitoring.elasticsearch.hosts: [ "http://elasticsearch01:9200" ]
xpack.monitoring.elasticsearch.hosts: [ "http://es01:9200" ]

xpack.monitoring.enabled: false

xpack.monitoring.elasticsearch.username: "elastic"
xpack.monitoring.elasticsearch.password: "elastic"
```


```
vim pipeline/logstash.conf


input {
  beats {
    port => 5044
  }

  tcp {
     port => 5000
   }
}

filter {
  # add filters here if needed (grok, date, mutate)
}

output {
  elasticsearch {
    hosts => ["http://es01:9200"]
    index => "logstash-docker-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "elastic"
  }
}
```


Or, To test without Filebeat, mount `audit.log` into Logstash and read via file input:
```
input {
  file {
    path => "/var/log/audit/audit.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
  }
}

output {
  elasticsearch {
    hosts => ["http://es01:9200"]
    index => "logstash-docker-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "elastic"
  }
}
```


```
ll logstash/

-rw-r--r-- 1 root root 230 Sep 23 11:15 logstash.yml
drwxr-xr-x 2 root root   6 Sep 23 13:41 pipeline
```


```
tree logstash/

logstash/
├── logstash.yml
└── pipeline
    └── logstash.conf
```


_Create file `docker-compose.yml`:_
```
services:
  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.15.1   # replace version if you want
    container_name: elasticsearch01
    environment:
      node.name: es01
      cluster.name: docker-cluster
      discovery.type: single-node
      bootstrap.memory_lock: "true"
      #xpack.security.enabled: "false"    # disable security for quick local testing
      xpack.security.enabled: "true"
      ELASTIC_PASSWORD: elastic          # If security is disabled, Elasticsearch will ignore ELASTIC_PASSWORD
      ES_JAVA_OPTS: "-Xms256m -Xmx256m"

    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - ./es01_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
      - "9300:9300"
    networks:
      - elk

  kibana:
    image: docker.elastic.co/kibana/kibana:7.15.1
    container_name: kibana01
    environment:
      SERVER_NAME: kibana
      ELASTICSEARCH_HOSTS: "http://es01:9200"
      ELASTICSEARCH_USERNAME: elastic
      ELASTICSEARCH_PASSWORD: elastic
    ports:
      - "5601:5601"
    depends_on:
      - es01
    networks:
      - elk

  logstash:
    image: docker.elastic.co/logstash/logstash:7.15.1
    container_name: logstash01
    environment:
      ELASTIC_PASSWORD: elastic
      LS_JAVA_OPTS: "-Xmx256m -Xms256m"
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
      - ./logstash/logstash.yml:/usr/share/logstash/config/logstash.yml:ro
      - /var/log/audit/audit.log:/var/log/audit/audit.log:ro

    command: logstash -f /usr/share/logstash/pipeline/logstash.conf
    #user: root
    ports:
      - "5044:5044"   # beats input
      - "5000:5000"   # tcp input (optional)
    depends_on:
      - es01
    networks:
      - elk

networks:
  elk:
    driver: bridge
    external: true

```



```
docker compose config
```


```
docker compose up -d
```





#### Verify: 

_For elasticsearch:_
```
curl localhost:9200

curl http://localhost:9200/_cluster/health?pretty
```


```
curl -u elastic:elastic localhost:9200

curl -u elastic:elastic http://localhost:9200/_cluster/health?pretty
```


```
curl -X GET "http://localhost:9200/_cat/indices?v"

curl -u elastic:elastic -X GET "http://localhost:9200/_cat/indices?v"
```



_For kibana UI:_
```
http://<host-ip>:5601
```



_Send test events for quick testing, you can send something manually:_
```
nc localhost 5000
```


_Type:_
```
hello from logstash
```


Or,

```
echo "hello from logstash test" | nc localhost 5000
```



_If connection fails → Logstash port 5044 is not reachable from Filebeat host. Check it:_
```
nc -zv 192.168.10.193 5044
```



### Configure Filebeat:

If you have multiple servers, use `Filebeat` → `Logstash` → `Elasticsearch`. Filebeat will send logs to Logstash (5044), which will parse and forward to Elasticsearch.



_Install Filebeat on the host and configure:_
```
### ==== Filebeat inputs =====
filebeat.inputs:

- type: log
  enabled: true
  paths:
    #- /var/log/*.log
    - /var/log/audit/audit.log

### ----- Logstash Output -------
output.logstash:
  #hosts: ["localhost:5044"]
  hosts: ["192.168.10.193:5044"]
```


```
chmod 644 /var/log/audit/audit.log
```



```
./filebeat modules enable system nginx mysql auditd

./filebeat modules list
```



_Start Filebeat:_
```
./filebeat -e
```


_Enable debug output:_
```
filebeat -e -d "*"
```



### License:
This project is licensed under the **MIT** License.



### Author:
- **Name:** Technbd   



### Ref:
- [Elasticsearch with Docker](https://www.elastic.co/guide/en/elasticsearch/reference/7.5/docker.html)
- [Kibana with Docker](https://www.elastic.co/docs/deploy-manage/deploy/self-managed/install-kibana-with-docker)
- [Logstash for Docker](https://www.elastic.co/docs/reference/logstash/docker-config)

