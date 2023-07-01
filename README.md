# Enabling Logging of All Keycloak Events:

- Create a directory named `startup-scripts` and within it, create a file named `logging.cli`.

- Insert the necessary configuration lines:

```
embed-server --server-config=standalone.xml  --std-out=echo

/subsystem=logging/logger=org.keycloak.events/:add(category=org.keycloak.events,level=DEBUG)

/subsystem=logging/root-logger=ROOT:write-attribute(name=handlers,value=[file])

stop-embedded-server
```

- Modify the docker-compose file to include the newly created directories and files. This ensures they are applied when the container starts.

# Writing Logs into a File on Keycloak:

- Add the following configuration line to the existing `logging.cli` file:

```
/subsystem=logging/file-handler=file:add(autoflush=true, file={relative-to=jboss.server.log.dir, path="keycloak.log"})
```
- Create a `logs` directory and mount it to the container to persistently save all logs on a disk.

# Sending Logs via Filebeat to ELK:

- Create a file named `filebeat.yml` (or any name of your choice) to transmit the logs to Logstash:

```
filebeat.inputs:
- type: log
  paths:
    - /usr/share/filebeat/logs/*.log

output.logstash:
  hosts: ["logstash:5044"]
```

- Mount the Keycloak logs to the Filebeat container in the docker-compose file.
- Create a file named `logstash.conf` in the `pipeline` directory and add the following configuration lines. These will direct the logs received from Filebeat to Elasticsearch:

```
input {
  beats {
    port => 5044
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
  }
}
```
# Configurations on Kibana:

- Log into Kibana and navigate to Management => Stack Management => Index Patterns. If all configurations are correctly set and Filebeat can connect to Logstash, a pattern named `filebeat-$CURRENT_DATE` should be visible. However, to access all logs of any given day, replace `filebeat-$CURRENT_DATE` with `filebeat-*`, then select timestamp and create the index. Following this, all logs will appear in the `Discovery` section.


Note: If your ELK stack is designed to connect to each other using credentials, you need to add "filebeat-*" indices into the "logstash._writer.json" file to grant sufficient privileges to write the logs received by Filebeat.