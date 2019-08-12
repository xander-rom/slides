## A watch describes a single alert and can contain multiple notification actions.
## It is written in json and constructed from simple building blocks:

### [Schedule](https://www.elastic.co/guide/en/elastic-stack-overview/7.3/trigger-schedule.html) - how often your watcher will run and check the condition. 
```
{
  "trigger": {
    "schedule": {
      "interval": "your interval (10s, 15m, 2h, etc)"
    }
  }
}
```
### Input - what information your condition is based
```
 { "input": {
  "search": {
      "request": {
        "indices": [            # define the indice(s) for searching
          "filebeat-gke-*"
        ],
        "body": {
          "size": 0,            # how many buckets will shown additionally in the output of the watcher 
          "query": {
            "bool": {           # type of the compound query
              "must": [
                {
                  "match": {    # type of the full-text query
                    "message": "error"
                  }
                }
              ],
              "must_not": [
                {
                  "terms": {    # type of the term-level query
                    "message": [
                      "warn",
                      "info",
                      "debug"
                    ]
                  }
                } 
              ],
              "filter": [
                {
                  "term": {
                    "kubernetes.namespace": "fido"
                  }
                },
                {
                  "range": {
                    "@timestamp": {
                      "gte": "now-{{ctx.metadata.window_period}}"
                    }
                  }
                }
              ]
            }
          }
        }
      }
    }
  }
}
```
#### [Types of the input](https://www.elastic.co/guide/en/elastic-stack-overview/7.3/input.html)
#### [Types of a compound query](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/compound-queries.html)
#### [Types of full text query](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/full-text-queries.html)
#### [Types of term-level query](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/term-level-queries.html)

### Condition  - when your watch should trigger the action(s). Action(s) is triggered only when the condition is met.
```
{ "condition": {
    "compare": {    # type of the condition
      "ctx.payload.hits.total": {
        "gte": 5
      }
    }
  }
 }
```
#### [Types of condition](https://www.elastic.co/guide/en/elastic-stack-overview/7.3/condition.html)
### Actions - how watch will notify you. Each action can have its own condition, it's useful when you have different levels of notification (green, yellow, red)
```
{ "actions": {
    "my-email-action": {    # name of the action
      "condition": {        # this email will be sent only if watch's condition and action's condition will be met
        "compare": {
          "ctx.payload.hits.total": {
            "gte": 10
          }
        }   
      },    
      "email": {            # type of the action
        "profile": "{profile}",
        "priority": "{priority}",
        "to": [
          "your_email@company.com"
        ],
        "subject": "WatcherAlert - {{ctx.watch_id}} - {{ctx.metadata.danger}}",
        "body": {           # type of the email content
          "html": "<b>The number of errors is increased - {{ctx.payload.hits.total}} for last hour</b>"
        }
      }
    }
  }
}
```
### Metadata - you can store here additional info and use it with ctx.metadata.your_info
```
{  "metadata": {
    "window_period": "1h",
    "danger": "red"
    }
}
```
### [Throttling](https://www.elastic.co/guide/en/elastic-stack-overview/7.3/actions.html#actions-ack-throttle) - time between notifications. It can be applied at watcher or action levels
```
{
    "throttle_period": "1h"
}
```

### [Aggregations](https://www.elastic.co/guide/en/elasticsearch/client/net-api/7.x/reference-aggregations.html) aren't a building block of watcher, but it is useful and important part of input, which also helps us to understand the transform part.
Example: search gets information about average response time from the service by executing two aggregations - by service name and then by computing average of the response field.

```
{
  "input": {
    "search": {
      "request": {
        "indices": [
          "apm-*"
        ],
        "body": {
          "size": 0,
          "query": {
            "bool": {
              "must": {
                "regexp": {
                  "service.name": {
                    "value": "service.*"
                  }
                }
              },
              "filter": [
                {
                  "range": {
                    "@timestamp": {
                      "gte": "now-{{ctx.metadata.window_period}}"
                    }
                  }
                }
              ]
            }
          },
          "aggs": {
            "serv": {           # name of the aggregation
              "terms": {        # type of the aggregation
                "field": "service.name",
                "order": {      # ordering by field
                  "response": "desc"
                }
              },
              "aggs": {
                "response": {
                  "avg": {
                    "field": "transaction.duration.us"
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```
The result is a number of buckets. We can get the service name and response time with ctx.payload.aggregations.serv.buckets.{number of bucket}.key/response

### [Transform](https://www.elastic.co/guide/en/elastic-stack-overview/7.3/transform.html) - processes and changes the payload in the watch execution context to prepare it for the watch actions. It is optional. When none are defined, the actions have access to the payload as loaded by the watch input.
In the previous example we get the response time in microseconds which is not a human readable format. Lets change it.
```
{ "transform": {
    "script": {
      "source": "return ['resp': ctx.payload.aggregations.serv.buckets.stream().map(p -> [ 'app': p.key, 'time': Math.round(p.response.value/1000)]).collect(Collectors.toList()) ];",
      "lang": "painless"
    }
  }
}
```
After the transform execution we have in the ctx.payload.resp number of buckets with two fields (app and time). We can't get older payload from the input, the ctx.payload.aggregations.serv.buckets.#.response returns 0.
