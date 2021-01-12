---
layout: default
title: KNN
nav_order: 50
has_children: true
has_toc: false
has_math: true
---

# k-NN

Short for *k-nearest neighbors*, the k-NN plugin enables users to search for the k-nearest neighbors to a query point across an index of points. To determine the neighbors, a user can specify the space (the distance function) they want to use to measure the distance between points. Currently, the k-NN plugin supports Euclidean, cosine similarity, and Hamming bit spaces.

Use cases include recommendations (for example, an "other songs you might like" feature in a music application), image recognition, and fraud detection. For background information on the algorithm, see [Wikipedia](https://en.wikipedia.org/wiki/K-nearest_neighbors_algorithm).

The k-NN plugin supports two methods of k-NN search. The first method uses the HNSW algorithm to return the approximate k-nearest neighbors. This algorithm sacrifices indexing speed and search accuracy in return for lower latency and more scalable search. To learn more about the algorithm, please refer to [nmslib's documentation](https://github.com/nmslib/nmslib/) or [the paper introducing the algorithm](https://arxiv.org/abs/1603.09320). Currently, only the Euclidean and cosine similarity spaces are available for this method.

The second method extends Elasticsearch's custom scoring functionality to execute a brute force, exact k-NN search. With the custom scoring approach, users are able to run k-NN search on a subset of vectors in their index (sometimes referred to as a pre-filter search). The Euclidean, cosine similarity, and Hamming bit spaces are available in this method.

For larger data sets, users should generally choose the approximate nearest neighbor method, because it scales much better. For smaller data sets, where a user may want to apply a filter, they should choose the custom scoring approach.


## Get started

To use the k-NN plugin's search functionality, you must first create a k-NN index. If you want to use the approximate k-nearest neighbors method, you will need to set the index setting, `index.knn` to `true`. This setting tells the plugin to create HNSW graphs for the index. If you only want to use the custom scoring method, you can use `false`.

Additionally, if you are using the approximate k-nearest neighbor method, you should specify `knn.space_type` to the space that you are interested in. This is not necessary for the custom scoring approach, because space is set for each query. Currently, we only support two spaces in the approximate nearest neighbor method: `l2` to use Euclidean distance or `cosinesimil` to use cosine similarity. By default, `index.knn.space_type` is `l2`.

For both methods, you must add one or more fields of the `knn_vector` data type. However, if you are using Hamming distance with the custom scoring method, you should use the long or binary field type. Here is an example that creates an index with two `knn_vector` fields and uses cosine similarity:

```json
PUT my-knn-index-1
{
  "settings": {
    "index": {
      "knn": true,
      "knn.space_type": "cosinesimil"
    }
  },
  "mappings": {
    "properties": {
      "my_vector1": {
        "type": "knn_vector",
        "dimension": 2
      },
      "my_vector2": {
        "type": "knn_vector",
        "dimension": 4
      }
    }
  }
}
```

The `knn_vector` data type supports a single list of up to 10,000 floats, with the number of floats defined by the required dimension parameter.

In Elasticsearch, codecs handle the storage and retrieval of indices. The k-NN plugin uses a custom codec to write vector data to a graph so that the underlying KNN search library can read it.
{: .tip }

After you create the index, add some data to it:

```json
POST _bulk
{ "index": { "_index": "my-knn-index-1", "_id": "1" } }
{ "my_vector1": [1.5, 2.5], "price": 12.2 }
{ "index": { "_index": "my-knn-index-1", "_id": "2" } }
{ "my_vector1": [2.5, 3.5], "price": 7.1 }
{ "index": { "_index": "my-knn-index-1", "_id": "3" } }
{ "my_vector1": [3.5, 4.5], "price": 12.9 }
{ "index": { "_index": "my-knn-index-1", "_id": "4" } }
{ "my_vector1": [5.5, 6.5], "price": 1.2 }
{ "index": { "_index": "my-knn-index-1", "_id": "5" } }
{ "my_vector1": [4.5, 5.5], "price": 3.7 }
{ "index": { "_index": "my-knn-index-1", "_id": "6" } }
{ "my_vector2": [1.5, 5.5, 4.5, 6.4], "price": 10.3 }
{ "index": { "_index": "my-knn-index-1", "_id": "7" } }
{ "my_vector2": [2.5, 3.5, 5.6, 6.7], "price": 5.5 }
{ "index": { "_index": "my-knn-index-1", "_id": "8" } }
{ "my_vector2": [4.5, 5.5, 6.7, 3.7], "price": 4.4 }
{ "index": { "_index": "my-knn-index-1", "_id": "9" } }
{ "my_vector2": [1.5, 5.5, 4.5, 6.4], "price": 8.9 }

```

Then you can execute an approximate nearest neighbor search on the data using the `knn` query type:

```json
GET my-knn-index-1/_search
{
  "size": 2,
  "query": {
    "knn": {
      "my_vector2": {
        "vector": [2, 3, 5, 6],
        "k": 2
      }
    }
  }
}
```

In this case, `k` is the number of neighbors you want the query to return, but you must also include the `size` option. Otherwise, you get `k` results for each shard (and each segment) rather than `k` results for the entire query. The plugin supports a maximum `k` value of 10,000.

Additionally, you can execute an exact nearest neighbor search on the data using the `knn` script:
```json
GET my-knn-index-1/_search
{
 "size": 4,
 "query": {
   "script_score": {
     "query": {
       "match_all": {}
     },
     "script": {
       "source": "knn_score",
       "lang": "knn",
       "params": {
         "field": "my_vector2",
         "query_value": [2, 3, 5, 6],
         "space_type": "cosinesimil"
       }
     }
   }
 }
}
```


## Compound queries with KNN

If you use the `knn` query alongside filters or other clauses (e.g. `bool`, `must`, `match`), you might receive fewer than `k` results. In this example, `post_filter` reduces the number of results from 2 to 1:

```json
GET my-knn-index-1/_search
{
  "size": 2,
  "query": {
    "knn": {
      "my_vector2": {
        "vector": [2, 3, 5, 6],
        "k": 2
      }
    }
  },
  "post_filter": {
    "range": {
      "price": {
        "gte": 5,
        "lte": 10
      }
    }
  }
}
```


## Pre-filtering with custom scoring

The [previous example](#mixing-queries) shows a search that returns fewer than `k` results. If you want to avoid this situation, the custom scoring method lets you essentially invert the order of events. In other words, you can filter down the set of documents you want to execute the k-nearest neighbor search over.

If you *only* want to use custom scoring, you can omit `"index.knn": true`. The benefit of this approach is faster indexing speed and lower memory usage, but you lose the ability to perform standard k-NN queries on the index.
{: .tip}

This example shows a pre-filter approach to k-NN search with custom scoring. First, create the index:

```json
PUT my-knn-index-2
{
  "mappings": {
    "properties": {
      "my_vector": {
        "type": "knn_vector",
        "dimension": 2
      },
      "color": {
        "type": "keyword"
      }
    }
  }
}
```

Then add some documents:

```json
POST _bulk
{ "index": { "_index": "my-knn-index-2", "_id": "1" } }
{ "my_vector": [1, 1], "color" : "RED" }
{ "index": { "_index": "my-knn-index-2", "_id": "2" } }
{ "my_vector": [2, 2], "color" : "RED" }
{ "index": { "_index": "my-knn-index-2", "_id": "3" } }
{ "my_vector": [3, 3], "color" : "RED" }
{ "index": { "_index": "my-knn-index-2", "_id": "4" } }
{ "my_vector": [10, 10], "color" : "BLUE" }
{ "index": { "_index": "my-knn-index-2", "_id": "5" } }
{ "my_vector": [20, 20], "color" : "BLUE" }
{ "index": { "_index": "my-knn-index-2", "_id": "6" } }
{ "my_vector": [30, 30], "color" : "BLUE" }

```

Finally, use the `script_score` query to pre-filter your documents before identifying nearest neighbors:

```json
GET my-knn-index-2/_search
{
  "size": 2,
  "query": {
    "script_score": {
      "query": {
        "bool": {
          "filter": {
            "term": {
              "color": "BLUE"
            }
          }
        }
      },
      "script": {
        "lang": "knn",
        "source": "knn_score",
        "params": {
          "field": "my_vector",
          "query_value": [9.9, 9.9],
          "space_type": "l2"
        }
      }
    }
  }
}
```

All parameters are required.

- `lang` is the script type. This value is usually `painless`, but here you must specify `knn`.
- `source` is the name of the script, `knn_score`.

  This script is part of the KNN plugin and isn't available at the standard `_scripts` path. A GET request to  `_cluster/state/metadata` doesn't return it, either.

- `field` is the field that contains your vector data.
- `query_value` is the point you want to find the nearest neighbors for. For the Euclidean and cosine similarity spaces, the value must be an array of floats that matches the dimension set in the field's mapping. For Hamming bit distance, this value can be either of type signed long or a base64-encoded string (for the long and binary field types, respectively).
- `space_type` is either `l2`, `cosinesimil`, or `hammingbit`.


## Performance considerations

The standard KNN query and custom scoring option perform differently. Test using a representative set of documents to see if the search results and latencies match your expectations.

Custom scoring works best if the initial filter reduces the number of documents to no more than 20,000. Increasing shard count can improve latencies, but be sure to keep shard size within [the recommended guidelines](../elasticsearch/#primary-and-replica-shards).


## Use Similarity methods in painless scripting

Sometimes users would like to go beyond Elasticsearch’s built-in features for scoring and might want to customize the search scores in more complex ways.
Elasticsearch provides script_score, the ability to provide custom scores for desired documents.
Elasticsearch Painless, a simple, secure scripting language designed specifically for this purpose.

A Painless script is evaluated within a context. Painless has a strict list of allowed methods and classes per context to ensure all Painless scripts are secure.
Now, opendistro users can use following specialized methods within Score context to customize their scores.

### Cosine Similarity
This function calculates the measure of cosine similarity between a given query vector and document vectors.
Optionally accepts normQueryVector, to avoid repeated calculation of normalization for query vector for every filtered documents.  
#### Definition
```
float cosineSimilarity (float[] queryVector, doc['vector field']) 
float cosineSimilarity (float[] queryVector, doc['vector field'], float normQueryVector) 
```

#### Usage
```
GET my-knn-index-2/_search
{
  "size": 2,
  "query": {
    "script_score": {
      "query": {
        "bool": {
          "filter": {
            "term": {
              "color": "BLUE"
            }
          }
        }
      },
      "script": {
        "source": "1.0 + cosineSimilarity(params.query_value, doc[params.field])", 
        "params": {
          "field": "my_vector",
          "query_value": [9.9, 9.9],
        }
      }
    }
  }
}
```
The above script adds 1.0 to the cosine similarity to keep score positive.
### L2 Squared
This function calculates L2 distance (Euclidean distance) between a given query vector and document vectors.

#### Definition
```
float l2Squared (float[] queryVector, doc['vector field'])
```
#### Usage
```
GET my-knn-index-2/_search
{
  "size": 2,
  "query": {
    "script_score": {
      "query": {
        "bool": {
          "filter": {
            "term": {
              "color": "BLUE"
            }
          }
        }
      },
      "script": {
        "source": "1 / (1 + l2Squared(params.query_value, doc[params.field]))", 
        "params": {
          "field": "my_vector",
          "query_value": [9.9, 9.9],
        }
      }
    }
  }
}

```
The lesser the distance the more the relevance of the document to the query vector. In order to bring the lesser 
distances documents to the top of the scores, we invert the distance from l2Squared function.
Also, when a document vector matches the query vector, we needed to add 1 in the denominator to avoid divide by zero error.

####Constraints
1. If a document’s knn vector field has different dimensions from the query, an error(IllegalArgumentException) will be thrown.
2. If similarity function is executed on vector field which doesn't have a value, an error(IllegalStateException) will be thrown.
This can be avoided by checking if a document has a value for the field my_vector by doc['my_vector'].size() == 0. You can re-write the source as below:
```
 "source": "doc[params.field].size() == 0 ? 0 : 1 / (1 + l2Squared(params.query_value, doc[params.field]))",
```
Since the score can only be positive, the above script will rank non-vector field documents lower than others
