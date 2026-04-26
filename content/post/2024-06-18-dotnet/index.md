+++
author = "yuhao"
title = "Working with Elasticsearch in .NET Core"
date = "2024-06-18"
description = "Elasticsearch is an open-source search engine built on top of Apache Lucene™, the most advanced, high-performance, full-featured search engine library available today. While powerful, Lucene remains..."
tags = [
    ".NET",
    ".NET Core",
    "ElasticSearch",
]
categories = [
    "Programming/Development",
]
+++
# Elasticsearch Quick Start Guide

## Introduction to Elasticsearch

[Elasticsearch](https://www.elastic.co/) is an open-source search engine built on top of Apache Lucene™, the most advanced, high-performance, full-featured search engine library available today. While powerful, Lucene remains just a library. To leverage its capabilities fully, developers need Java expertise and direct integration into applications. Elasticsearch simplifies this process by:

- Providing a distributed real-time document store with every field indexed and searchable
- Offering a distributed analytics engine
- Scaling to hundreds of servers and handling petabytes of structured/unstructured data

With official clients available for Java, .NET, PHP, Python, Ruby, Node.js, and more, Elasticsearch leads the enterprise search market per DB-Engines rankings, followed by Apache Solr (also Lucene-based).

## Development Resources

### Documentation

- Chinese: [Elasticsearch: The Definitive Guide](https://www.elastic.co/guide/cn/elasticsearch/guide/cn/index.html)
- English: [Elasticsearch Reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)

### Downloads

[https://www.elastic.co/cn/downloads/](https://www.elastic.co/cn/downloads/)

## Core APIs

| API Category | Documentation Link |
|-------------|--------------------|
| API Conventions | [Guide](https://www.elastic.co/guide/en/elasticsearch/reference/current/api-conventions.html) |
| Document APIs | [Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs.html) |
| Search APIs | [Reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/search.html) |
| Indices APIs | [Guide](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices.html) |
| cat APIs | [Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat.html) |
| Cluster APIs | [Reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster.html) |
| JavaScript API | [Client Docs](https://www.elastic.co/guide/en/elasticsearch/client/javascript-api/current/index.html) |

## Logstash Integration

| Component | Documentation |
|-----------|---------------|
| Logstash Reference | [Guide](https://www.elastic.co/guide/en/logstash/current/index.html) |
| Configuration | [Config Docs](https://www.elastic.co/guide/en/logstash/current/configuration.html) |
| Input Plugins | [List](https://www.elastic.co/guide/en/logstash/current/input-plugins.html) |
| Output Plugins | [Reference](https://www.elastic.co/guide/en/logstash/current/output-plugins.html) |
| Filter Plugins | [Docs](https://www.elastic.co/guide/en/logstash/current/filter-plugins.html) |

## Kibana DevTools Essentials

### Keyboard Shortcuts

- `Ctrl+i`: Auto-indent
- `Ctrl+Enter`: Execute query
- `Down Arrow`: Open auto-complete menu
- `Enter/Tab`: Select auto-complete suggestion
- `Esc`: Close suggestion menu

Append `?pretty=true` to any query string for formatted JSON responses.

### Common Commands

```bash
# Check disk allocation
GET _cat/allocation?v

# List all indices
GET _cat/indices

# Sort indices by document count
GET _cat/indices?s=docs.count:desc
GET _cat/indices?v&s=index

# List cluster nodes
GET _cat/nodes

# Cluster health status
GET _cluster/health?pretty=true
GET _cat/indices/*?v&s=index

# Get shard information
GET logs/_search_shards
```

### Cluster Health Check

```bash
curl -XGET 'http://<host>:9200/_cluster/health?pretty'
```

Sample Response:

```json
{
  "cluster_name": "es-cluster",
  "status": "green",
  "timed_out": false,
  "number_of_nodes": 3,
  "number_of_data_nodes": 3,
  "active_primary_shards": 1,
  "active_shards": 2,
  "relocating_shards": 0,
  "initializing_shards": 0,
  "unassigned_shards": 0,
  "delayed_unassigned_shards": 0,
  "number_of_pending_tasks": 0,
  "number_of_in_flight_fetch": 0,
  "task_max_waiting_in_queue_millis": 0,
  "active_shards_percent_as_number": 100.0
}
```

## CRUD Operations

### Search Queries

```json
POST logs/_search
{
  "query": {
    "range": {
      "createdAt": {
        "gt": "2020-04-25",
        "lt": "2020-04-27",
        "format": "yyyy-MM-dd"
      }
    }
  },
  "size": 0,
  "aggs": {
    "url_type_stats": {
      "terms": {
        "field": "urlType.keyword",
        "size": 2
      }
    }
  }
}

POST logs/_search
{
  "size": 0,
  "aggs": {
    "date_total_ClientIp": {
      "date_histogram": {
        "field": "createdAt",
        "interval": "quarter",
        "format": "yyyy-MM-dd",
        "extended_bounds": {
          "min": "2020-04-26 13:00:00",
          "max": "2020-04-26 14:00:00"
        }
      },
      "aggs": {
        "url_type_api": {
          "terms": {
            "field": "urlType.keyword",
            "size": 10
          }
        }
      }
    }
  }
}
```

### Document Deletion

```bash
# Delete by query
POST logs/_delete_by_query {"query":{"match_all": {}}}

# Delete index
DELETE logs
```

## Index Management

### Creating Indices

Data migration essentially involves index re-creation. Note that reindexing doesn't copy source index settings - configure mapping, shards, and replicas explicitly.

### Migration Strategies

#### Cross-Cluster Reindexing

```json
POST _reindex
{
  "source": {
    "remote": {
      "host": "http://source-host:9200",
      "username": "user",
      "password": "pass"
    },
    "index": "source_index",
    "query": {"match": {"test": "data"}}
  },
  "dest": {"index": "dest_index"}
}
```

Configure `reindex.remote.whitelist` in elasticsearch.yml for security.

#### Elasticsearch-Dump

Node.js-based migration tool:

```bash
npm install elasticdump -g

# Transfer index components
elasticdump \
  --input=http://source:9200/my_index \
  --output=http://dest:9200/my_index \
  --type=analyzer

elasticdump \
  --input=http://source:9200/my_index \
  --output=http://dest:9200/my_index \
  --type=mapping

elasticdump \
  --input=http://source:9200/my_index \
  --output=http://dest:9200/my_index \
  --type=data
```

## Deep Pagination

Overcome 10,000 document limit using `search_after`:

```json
GET logs/_search
{
  "search_after": [1588042597000, "V363vnEBz1D1HVfYBb0V"],
  "size": 10,
  "sort": [
    {"createdAt": {"order": "desc"}},
    {"_id": {"order": "desc"}}
  ]
}
```

## Installation

### Docker Setup

```bash
# Elasticsearch
docker run -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:7.8.1

# Kibana
docker run -p 5601:5601 --link elasticsearch -e "elasticsearch.hosts=http://elasticsearch:9200" docker.elastic.co/kibana/kibana:7.8.1
```

## .NET Integration

### NuGet Packages

```powershell
Install-Package NEST
Install-Package Swashbuckle.AspNetCore
```

### Sample Entity

```csharp
public class VisitLog
{
    public string Id { get; set; }
    public string UserAgent { get; set; }
    public string Method { get; set; }
    public string Url { get; set; }
    public string Referrer { get; set; }
    public string IpAddress { get; set; }
    public int Milliseconds { get; set; }
    public DateTimeOffset CreatedAt { get; set; } = DateTimeOffset.UtcNow;
}
```

### Elasticsearch Provider

```csharp
public class ElasticsearchProvider : IElasticsearchProvider
{
    public IElasticClient GetClient() => new ElasticClient(
        new ConnectionSettings(new Uri("http://localhost:9200")));
}
```

### Repository Implementation

```csharp
public class VisitLogRepository : ElasticsearchRepositoryBase, IVisitLogRepository
{
    protected override string IndexName => "visitlogs";
    
    public async Task InsertAsync(VisitLog log) => 
        await Client.IndexAsync(log, x => x.Index(IndexName));
    
    public async Task<Tuple<int, IList<VisitLog>>> QueryAsync(int page, int limit)
    {
        var response = await Client.SearchAsync<VisitLog>(s => s
            .Index(IndexName)
            .From((page - 1) * limit)
            .Size(limit)
            .Sort(x => x.Descending(v => v.CreatedAt)));
        
        return Tuple.Create((int)response.Total, response.Documents.ToList());
    }
}
```

### API Controller

```csharp
[ApiController]
[Route("api/[controller]")]
public class VisitLogController : ControllerBase
{
    private readonly IVisitLogRepository _repo;

    [HttpGet]
    public async Task<IActionResult> Get(int page = 1, int limit = 10)
    {
        var result = await _repo.QueryAsync(page, limit);
        return Ok(new { total = result.Item1, items = result.Item2 });
    }

    [HttpPost]
    public async Task<IActionResult> Post([FromBody] VisitLog log)
    {
        await _repo.InsertAsync(log);
        return Ok("Created successfully");
    }
}
```

## Verification

After implementation, use Kibana to validate data:

```bash
GET _cat/indices
GET visitlogs/_search
```

This guide provides foundational knowledge for working with Elasticsearch in .NET environments. For advanced query patterns and optimization techniques, refer to the official [NEST Examples](https://github.com/elastic/elasticsearch-net/tree/master/examples).

