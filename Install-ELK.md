# Install ELK

## Step 1 Download ELK

```
cd /app
mkdir elk_install
cd elk_install
```

> Download Elasticsearch
```
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.2.0-x86_64.rpm
```

> Download Kibana
```
wget https://artifacts.elastic.co/downloads/kibana/kibana-8.2.0-x86_64.rpm
```

> Download Logstash
```
wget https://artifacts.elastic.co/downloads/logstash/logstash-8.2.0-x86_64.rpm
```

## Step 2 Install ELK

> Install Elasticsearch
```
rpm --install elasticsearch-8.2.0-x86_64.rpm
```
> After installation ElasticSearch will generate password for elastic built-in superuser 

> Install Kibana
```
rpm --install kibana-8.2.0-x86_64.rpm
```

> Install Logstash
```
rpm --install logstash-8.2.0-x86_64.rpm
```

## Step 3 Configuration
### ElasticSearch configuration

```
cd /app
mkdir Elasticsearch
chown elasticsearch:elasticsearch Elasticsearch
cd Elasticsearch
mkdir data
mkdir logs
```

> Change owner and group of Elasticsearch/data and Elasticsearch/logs from root root to elasticsearch elasticsearch
```
chown elasticsearch:elasticsearch data
chown elasticsearch:elasticsearch logs
```

> Change node.name data path and log path in /etc/elasticsearch/elasticsearch.yml 
```
uncomment and change node.name: node-1 to node.name: RAOT-TRT-LOG
■ change path.data: /var/lib/elasticsearch to path.data: /app/Elasticsearch/data
■ change path.logs: /var/log/elasticsearch to path.logs: /app/Elasticsearch/logs
```

### Kibana configuration

> Change port and host in /etc/kibana/kibana.yml 
```
■ uncomment server.port to use port 5601
■ uncomment and change server.host:  “localhost” to server.host: ”10.99.71.27”
``` 

> Allow firewall at port 5601/tcp with command
```
firewall-cmd --add-port=5601/tcp --permanent
systemctl restart firewalld
firewall-cmd --list-port
```

### Create pipeline 
```
vi /etc/logstash/conf.d/pipeline-filebeats.conf
```

```
input { 
  beats { 
    port => 9292 
}} 
filter {} 
output { 
    if ![kubernetes][namespace]{ 
      elasticsearch { 
        hosts => ["https://10.99.76.20:9200"] 
        index => "no_namespace-alias" 
         ssl => true
        ssl_certificate_verification => false 
        user => "elastic" 
        password => "Changethispassword" 
      } 
    }else{ 
      elasticsearch { 
        hosts => ["https://10.99.76.20:9200"]
        index => "%{[kubernetes][namespace]}-alias" 
         ssl => true
        ssl_certificate_verification => false 
        user => "elastic" 
        password => "Changethispassword" 
}}} 
```

> Allow firewall at port 9292/tcp with command
```
firewall-cmd --add-port=9292/tcp --permanent
systemctl restart firewalld
firewall-cmd --list-port
```

### start service ELK
```
systemctl enable elasticsearch
systemctl start elasticsearch
systemctl enable kibana
systemctl start kibana
systemctl enable logstash
systemctl start logstash
```

## Step 4 Access to Kibana dashborad

### Generate Token 
```
/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
```

### Create Index Lifecycle Management (ILM)

> Create ILM (Index Lifecycle Policy) on Web console Kibana > Menu > Devtools
>> Create Policy
```
PUT _ilm/policy/(Follow namespace)-policy
{ 
   "policy":{ 
      "phases":{ 
         "hot":{ 
            "min_age":"0ms", 
            "actions":{ 
               "rollover":{ 
                  "max_primary_shard_size":“40gb", 
                  "max_age":"30d" 
               }}}, 
         "delete":{ 
            "min_age":"90d", 
            "actions":{ 
               "delete":{ 
                  "delete_searchable_snapshot":true 
}}}}}} 
```

>> Create Template
```
PUT _template/(Follow namespace)-template 
{ 
  "order": 50, 
  "index_patterns": [ 
    "(Follow namespace)-*" 
  ], 
  "settings": { 
    "index": { 
      "lifecycle": { 
        "name": "(Follow namespace)-policy", 
        "rollover_alias": "(Follow namespace)-alias" 
      }, 
      "refresh_interval": "30s", 
      "number_of_shards": "1", 
      "number_of_replicas": "1" 
 }}} 
```

>> Create Alias
```
PUT %3C(Follow namespace)-%7Bnow%2Fd%7D-000001%3E 
{ 
  "aliases": { 
    "(Follow namespace)-alias": { 
      "is_write_index": true 
}}} 
```
