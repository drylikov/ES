# ES.

Package es provides an Elasticsearch query DSL.

## About

The use of Go "dot imports" is discouraged, however I'd recommend abstracting this logic into higher level query functions and packages if you'd like to utilize the expressiveness of dot imports in this scenario. I wouldn't recommend dot-importing it into other packages directly.

## Example

### Lispy

If you don't mind crazy nesting:

```go
query := Pretty(Query(
  Aggs(
    Agg("results",
      Filter(
        Term("user.login", "drylikov"),
        Range("now-7d", "now"),
      )(
        Aggs(
          Agg("repos",
            Terms("repository.name.keyword", 100),
            Aggs(
              Agg("labels",
                Terms("issue.labels.keyword", 100),
                Aggs(
                  Agg("duration_sum", Sum("duration"))))))))))))
```

### Less lispy

If you do mind crazy nesting:

```go
labels := Aggs(
  Agg("labels",
    Terms("issue.labels.keyword", 100),
    Aggs(
      Agg("duration_sum",
        Sum("duration")))))

repos := Aggs(
  Agg("repos",
    Terms("repository.name.keyword", 100),
    labels))

filter := Filter(
  Term("user.login", "drylikov"),
  Range("now-7d", "now"))

results := Aggs(
  Agg("results",
    filter(repos)))

query := Pretty(Query(results))
```

Both yielding:

```json
{
  "aggs": {
    "results": {
      "aggs": {
        "repos": {
          "aggs": {
            "labels": {
              "aggs": {
                "duration_sum": {
                  "sum": {
                    "field": "duration"
                  }
                }
              },
              "terms": {
                "field": "issue.labels.keyword",
                "size": 100
              }
            }
          },
          "terms": {
            "field": "repository.name.keyword",
            "size": 100
          }
        }
      },
      "filter": {
        "bool": {
          "filter": [
            {
              "term": {
                "user.login": "drylikov"
              }
            },
            {
              "range": {
                "timestamp": {
                  "gte": "now-7d",
                  "lte": "now"
                }
              }
            }
          ]
        }
      }
    }
  },
  "size": 0
}
```

### Reuse

This also makes reuse more trivial, for example note how `sum` is re-used in the following snippet to fetch global, daily, and label-level summation.

```go
sum := Agg("duration_sum", Sum("duration"))

labels := Agg("labels",
  Terms("issue.labels.keyword", 100),
  Aggs(sum))

days := Agg("days",
  DateHistogram("1d"),
  Aggs(sum, labels))

query := Query(Aggs(sum, labels, days))
```

Yielding:

```json
{
  "aggs": {
    "days": {
      "aggs": {
        "duration_sum": {
          "sum": {
            "field": "duration"
          }
        },
        "labels": {
          "aggs": {
            "duration_sum": {
              "sum": {
                "field": "duration"
              }
            }
          },
          "terms": {
            "field": "issue.labels.keyword",
            "size": 100
          }
        }
      },
      "date_histogram": {
        "field": "timestamp",
        "interval": "1d"
      }
    },
    "duration_sum": {
      "sum": {
        "field": "duration"
      }
    },
    "labels": {
      "aggs": {
        "duration_sum": {
          "sum": {
            "field": "duration"
          }
        }
      },
      "terms": {
        "field": "issue.labels.keyword",
        "size": 100
      }
    }
  },
  "size": 0
}
```

---























































































































