# Enable logging all events of keyclock:

- Add a directory named `startup-scripts` and create a file named `logging.cli` into it.

- Add the required config lines:

```
embed-server --server-config=standalone.xml  --std-out=echo

/subsystem=logging/logger=org.keycloak.events/:add(category=org.keycloak.events,level=DEBUG)

/subsystem=logging/root-logger=ROOT:write-attribute(name=handlers,value=[file])

stop-embedded-server
```

- Modify the docker-compose file to include all new created directories and files to apply them during starting the container.

# Write logs into a file on keycloak:

- Add the follwoing config line in the existing `logging.cli` file:

```
/subsystem=logging/file-handler=file:add(autoflush=true, file={relative-to=jboss.server.log.dir, path="keycloak.log"})
```
- create `logs` directory and mount it to the container to save all logs in a persistent disk.

# Send the logs by filebeat to ELK:

- Create a file named `filebeat.yml` or whatever name you like to send the logs to logstash:

```
filebeat.inputs:
- type: log
  paths:
    - /usr/share/filebeat/logs/*.log

output.logstash:
  hosts: ["logstash:5044"]
```

- Mount the keycloak's logs to filebeat container in docker-compose file.
- Create a file named `logstash.conf` into `pipeline` directory and add the following config lines to send the received logs from filebeat to elasticsearch:

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
# Configs on Kibana:

- Login to kibana and go to Management => Stack management => Index patterns. If all configs are OK and filebeat can reach logstash, there must be a pattern named `filebeat-$CURRENT_DATE`. To be able to get all logs of any day, instead of writting `filebeat-$CURRENT_DATE` you must write only `filebeat-*` and then select timestamp and create the index. Now, all logs will be shown at `Discovery` section.


Note: If your ELK stack is designed to connected to each other by credentials you need to add "filebeat-*" indices into "logstash._writer.json" file to have a sufficient privilage to write filebeat received logs.