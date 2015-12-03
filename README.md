# SIREn Join Plugin for Elasticsearch

This plugin extends Elasticsearch with new search actions and a filter query parser that enables to perform
a "filter join" between two set of documents (in the same index or in different indexes).

## Installing the Plugin

you can use the following command to download the plugin from the online repository:

    $ bin/plugin -i solutions.siren/siren-join/1.0

you can now start Elasticsearch and see that our plugin gets loaded:

    $ bin/elasticsearch
    ...
    [2013-09-04 17:33:27,443][INFO ][node    ] [Andrew Chord] initializing ...
    [2013-09-04 17:33:27,455][INFO ][plugins ] [Andrew Chord] loaded [FilterJoinPlugin], sites []
    ...

alternatively, you can assemple it via Maven:

```
$ mvn package
```

this creates a Zip file that can be installed using the Elasticsearch plugin command:

    $ bin/plugin --url file:///PATH-TO-FILTERJOIN-PLUGIN/target/releases/siren-join-1.0.zip --install FilterJoinPlugin

Note that we use the `--url` option for the plugin command in order to inform it to get the file locally 
instead of trying to download it from an online repository.
Alternatively, 

To uninstall plugin

    $ bin/plugin --remove FilterJoinPlugin

## Usage

### Coordinate Search API

This plugin introduces two new search actions, `_coordinate_search` that replaces the `_search` action, 
and `_coordinate_msearch` that replaces the `_msearch` action. Both actions are wrappers around the original
elasticsearch actions and therefore supports the same API. One must use these actions with the `filterjoin` filter,
as the `filterjoin` filter is not supported by the original elaticsearch actions.
 
### Parameters

* `filterjoin`: the filter name
* `indices`:  the index names to lookup the terms from (optional, default to all indices).
* `types`: the index types to lookup the terms from (optional, default to all types).
* `path`: the path within the document to lookup the terms from.
* `query`: the query used to lookup terms with.
* `orderBy`: the ordering to use to lookup the maximum number of terms: default, doc_score (optional, default to default ordering).
* `maxTermsPerShard`: the maximum number of terms per shard to lookup (optional, default to all terms).
* `routing`: the node routing used to control the shards the lookup request is executed on.

### Example

In this example, we will join all the documents from `index1` with the documents of `index2`. 
The query first filters documents from `index2` and of type `type` with the query 
`{ "terms" : { "tag" : [ "aaa" ] } }`. It then retrieves the ids of the documents from the field `id`
 specified by the parameter `path`. The list of ids is then used as filter and applied on the field 
 `foreign_key` of the documents from `index1`.

```json
    {
      "filtered" : {
        "query" : {
          "match_all" : { }
        },
        "filter" : {
          "filterjoin" : {
            "foreign_key" : {
              "indices" : ["index2"],
              "types" : ["type"],
              "path" : "id",
              "query" : {
                "terms" : {
                  "tag" : [ "aaa" ]
                }
              }
            }
          }
        }
      }
    }
```

### Response Format

The response returned by the coordinate search API is identical to the response
returned by Elasticsearch's search API, but augmented with additional information
about the execution of the relational query planning. This additional information
is stored within the field named `coordinate_search` at the root of the response,
see example below. The object contains the following parameters:

* `actions`: a list of actions that has been executed - an action represents the execution of one single join.
* `relations`: the definition of the relations of the join - it contains two nested objects, `from` and `to`, one for each relation.
* `size`: the size of the filter used to compute the join, i.e., the number of terms across all shards used by the filterjoin.
* `size_in_bytes`: the size in bytes of the filter used to compute the join.
* `is_pruned`: a flag to indicate if the join computation has been pruned based on the `maxTermsPerShard` limit.
* `cache_hit`: a flag to indicate if the join was already computed and cached.
* `took`: the time it took to construct the filter.

```json
    {
      "coordinate_search": {
        "actions": [
          {
            "relations": {
              "from": {
                "indices": ["index2"],
                "types": ["type"],
                "field": "id"
              },
              "to": {
                "indices": null,
                "types": null,
                "field": "foreign_key"
              }
            },
            "size": 2,
            "size_in_bytes": 20,
            "is_pruned": false,
            "cache_hit": false,
            "took": 313
          }
        ]
      },
    ...
    }
```

## Acknowledgement

Part of this plugin is inspired and based on the pull request
[3278](https://github.com/elastic/elasticsearch/pull/3278) submitted by [Matt Wever](https://github.com/mattweber)
to the [Elasticsearch](https://github.com/elastic/elasticsearch) project.

- - -

Copyright (c) 2015, SIREn Solutions. All Rights Reserved.
