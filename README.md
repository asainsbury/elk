# ELK Install Notes

Decisions:

- Java 11
- SYSLOG-NG and Logstash on the same box.
- 3 Node elastic cluster.
- One Kibana node.
- Single core CPU.
- 2048 RAM.

Install information:

OpenJava 11
Went for 11 over 8, which is the newer LTS version.
All products are supported on 11 and 8, but though we should be looking at 11 as it has a longer shelf life with better support.

- https://www.elastic.co/support/matrix
- https://www.server-world.info/en/note?os=CentOS_7&p=jdk11&f=2


SSL Filebeat to Logstash

- https://www.elastic.co/guide/en/beats/filebeat/current/configuring-ssl-logstash.html

Install Filebeats

- https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-installation.html

```bash
[user@elk1 ~]$ systemctl status filebeat
● filebeat.service - Filebeat sends log files to Logstash or directly to Elasticsearch.
   Loaded: loaded (/usr/lib/systemd/system/filebeat.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2019-12-12 01:46:35 UTC; 47s ago
     Docs: https://www.elastic.co/products/beats/filebeat
 Main PID: 21836 (filebeat)
   CGroup: /system.slice/filebeat.service
           └─21836 /usr/share/filebeat/bin/filebeat -e -c /etc/filebeat/filebeat.yml -path.home /usr/share/filebeat -path.config /etc/filebeat -path.data /var/lib/filebeat -path.logs /var/log/filebeat
[user@elk1 ~]$
```

Install SYSLOG-NG:
Ansible playbooks to do this.
https://support.oneidentity.com/technical-documents/syslog-ng-premium-edition/7.0.9/performance-guideline-for-syslog-ng-premium-edition/


Install Logstash:
https://www.elastic.co/guide/en/logstash/7.5/pipeline.html

If the config is not good, jvm crashes and cycles states, this caused the CPU to spike.

```bash
[user@elk2 ~]$ systemctl status logstash
● logstash.service - logstash
   Loaded: loaded (/etc/systemd/system/logstash.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2019-12-13 04:37:34 UTC; 37s ago
 Main PID: 7122 (java)
   CGroup: /system.slice/logstash.service
           └─7122 /bin/java -Xms1g -Xmx1g -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -Djava.awt.headless=true -Dfile.encoding=UTF-8 -Djruby.compile...
[user@elk2 ~]$
```

```bash
(elastic) ➜  logstash git:(master) ✗ egrep -v "#|^$" tdc1dapp0005/logstash.yml
node.name: tdc1dapp0005
path.data: /app/logstash/data
pipeline.workers: 8
pipeline.batch.size: 200
pipeline.batch.delay: 50
queue.type: memory
config.test_and_exit: false
config.reload.automatic: true
config.reload.interval: 30s
http.host: "10.202.16.17"
http.port: 9600-9700
xpack.monitoring.enabled: true
xpack.monitoring.elasticsearch.url: [ "tdc1dapp0002", "tdc1dapp0003", "tdc2dapp0004" ]
xpack.monitoring.elasticsearch.sniffing: true
xpack.monitoring.collection.interval: 10s
xpack.management.pipeline.id: ["paloalto", "cisco"]
```

```bash
- pipeline.id: cisco
  path.config: "/etc/logstash/conf.d/10-cisco*.conf"
  pipeline.workers: 2
```

Install Elasticsearch:
To Do
vim /etc/sysconfig/elasticsearch
MAX_LOCKED_MEMORY=unlimited


Noticed tcp6 was binding to 9200. So downloaded the role to disable all IPv6, but that probably wasn't necessary, as I actually had to initialise the host to listen on a dedicated port, and put in a seed device, then all the service started, and nmap showed the listening port.

After running the initial setup, without configuration changes, you can check the status and health on each node:

```bash
[user@elk2 ~]$ curl -XGET 'http://localhost:9200'
{
  "name" : "elk2",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "Su_MoepAS-urO3esoa7VOg",
  "version" : {
    "number" : "7.5.0",
    "build_flavor" : "default",
    "build_type" : "rpm",
    "build_hash" : "e9ccaed468e2fac2275a3761849cbee64b39519f",
    "build_date" : "2019-11-26T01:06:52.518245Z",
    "build_snapshot" : false,
    "lucene_version" : "8.3.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
[user@elk2 ~]$ curl http://localhost:9200/_cluster/health?pretty
{
  "cluster_name" : "elasticsearch",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 0,
  "active_shards" : 0,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}

curl http://1.1.1.32:9200/_cat/nodes
1.1.1.33  9 88 1 0.35 0.42 0.26 dilm - elk4
1.1.1.31 10 90 4 0.26 0.32 0.23 dilm * elk2
1.1.1.32 11 90 4 0.61 0.38 0.25 dilm - elk3

curl -XGET http://elk2:9200/_cluster/health?pretty
{
  "cluster_name" : "gary",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 1,
  "active_shards" : 2,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```

Install Kibana

https://www.elastic.co/guide/en/kibana/current/rpm.html