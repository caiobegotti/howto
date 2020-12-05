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

PUT newindex/_settings
{
  "index.mapping.depth.limit": 1
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
     #- drop_fields:
     #    fields: ['message']
```
