Ingest node pipeline (as long message fields don't start with dots):

```
PUT /_ingest/pipeline/mypipeline
{
  "description": "mypipeline",
  "processors": [
    {
      "json": {
        "field": "message",
        "target_field": "app",
        "if": """
            if(ctx.message != null){
              if (ctx.message.startsWith('{')) {
                  return true;
              }
            }
            return false;
        """,
        "ignore_failure": false
      }
    }
  ],
  "on_failure": [
    {
      "set": {
        "field": "_index",
        "value": "failed-{{ _index }}"
      }
    }
  ]
}

PUT /newindex
{
  "mappings": {
    "properties": {
      "app.status": {
        "type": "text"
      },
      "app.source": {
        "type": "text"
      },
      "app.msg": {
        "type": "text"
      },
      "app.level": {
        "type": "text"
      },
      "app.method": {
        "type": "text"
      },
      "app.params": {
        "type": "object"
      },
      "app.params.value": {
        "type": "text"
      },
      "app.module": {
        "type": "text"
      }
    }
  }
}

PUT newindex/_settings
{
  "index.mapping.depth.limit": 1
}

POST /_reindex
{
   "source": {
       "index": "oldindex"
   },
   "dest": {
       "index": "newindex",
       "pipeline": "mypipeline"
   }
}
```

Filebeat JSON processing for Kubernetes pods:

```
- type: container
  containers.ids:
    - "*"
  paths: 
    - '/var/lib/docker/containers/*/*.log'  
  multiline.pattern: '^{'
  multiline.negate: true
  multiline.match: after
  multiline.max_lines: 5000
  multiline.timeout: 10
  processors:
    - add_kubernetes_metadata:
        in_cluster: true
    - decode_json_fields:
        when.regexp.message: '^{'
        fields: ["message"]
        target: "app"
        max_depth: 1
        process_array: true
        overwrite_keys: true
        add_error_key: true
```

Fluentbit version for JSON logs and text logs with mapping:

```
[INPUT]
    Name              tail
    Tag               kube.*
    Path              /var/log/containers/*_mycluster_*.log
    Parser            docker
    DB                /var/log/flb_kube.db
    Mem_Buf_Limit     5MB
    Skip_Long_Lines   Off
    Refresh_Interval  10
    Ignore_Older      5d

[PARSER]
    Name   json
    Format json
    Time_Key time
    Time_Format %d/%b/%Y:%H:%M:%S %z

[PARSER]
    Name        docker
    Format      json
    Time_Key    time
    Time_Format %Y-%m-%dT%H:%M:%S.%L
    Time_Keep   On
    Decode_Field_As   escaped_utf8    log

[PARSER]
    Name    app
    Format  regex
    Regex   ^(?<app_timestamp>[^ ]+) (?<log_type>[^ ]+) (?<jaeger_span_id>[^ ]+)? (?<log_level>[^ ]+) (?<filename>[^ ]+) (?<line_number>[^ ]+) (?<request_type>[^ ]+) "(?<called_method>[^"]+)?" "(?<message>.+)?" "(?<details>.+)?".*$

[OUTPUT]
    Name            es
    Match           *
    Host            ${ELASTICSEARCH_HOST}
    Port            ${ELASTICSEARCH_PORT}
    HTTP_User       ${ELASTICSEARCH_USERNAME}
    HTTP_Passwd     ${ELASTICSEARCH_PASSWORD}
    Logstash_Format On
    Logstash_Prefix app
    Replace_Dots    On
    Retry_Limit     False
    tls             On
    tls.verify      Off
```

Logstash exploding the JSON message field into multiple ones with a prefix:

```
filter {
    if [message] =~ /^{.*}$/ {
        json {
            source => message
            target => "app"
        }
    }
}
```
