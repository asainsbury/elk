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

Install Elasticsearch:

Noticed tcp6 was binding to 9200. So downloaded the role to disable all IPv6, but that probably wasn't necessary, as I actually had to initialise the host to listen on a dedicated port, and put in a seed device, then all the service started, and nmap showed the listening port.