---

excalidraw-plugin: parsed
tags: [excalidraw]

---
==⚠  Switch to EXCALIDRAW VIEW in the MORE OPTIONS menu of this document. ⚠== You can decompress Drawing data with the command palette: 'Decompress current Excalidraw file'. For more info check in plugin settings under 'Saving'


# Excalidraw Data

## Text Elements
Books

 ^iNnCTWaF

Document 1
- Title :
- Price : ^YHZ4ianC

Document 2
- Title :
-  Price : ^d1A3zase

Title : Text
 

 ^8yNZvKxR

Price : Float ^4rT5UkWu

Document ^8J67Psac

Index ^6MZO5aqH

Mapping ^N9atlgPC

Field ^5rL9JjY9

Deep dive on ElasticSearch ^95T0bEEj

DOCUMENT : Individual units of data that you search OVER.
(JSON object)
Like books in bookstore

{
  "id": "XYZ123",
  "title": "The Great Gatsby",
  "author": "F. Scott Fitzgerald",
  "price": 10.99,
  "createdAt": "2024-01-01T00:00:00.000Z"
} ^TxsllE8G

INDEX : refers to collection of documents. Think of an index as a database table
Searches happens against these indices and returns document results.  ^WrHgt0H1

MAPPING : Its the schema of the index. It defines the fileds that index will have
You can put whatever data you want in the document, but the mapping determines 
which fields are searchable and what type of data they contain

{
  "properties": {
    "id": { "type": "keyword" },
    "title": { "type": "text" },
    "author": { "type": "text" },
    "price": { "type": "float" },
    "createdAt": { "type": "date" }
  }
}

* Always keep in mind to map only required fields. if you include a lot of fields in your mapping that aren't
actually used in search, 
this increases the memory overhead of each index. This could cost us a performance issue

* Diff bw text and keyword:

- The text Field (Analyzed)
When you index the string "New York" into a text field, Elasticsearch uses an analyzer to break it down. It
creates an "inverted index" that looks like this:

        - new
        - york

Result: If you search for just "York", you will find this document because the token exists.

- The keyword Field (Not Analyzed)
When you index "New York" into a keyword field, Elasticsearch stores it exactly as is:
    - New York

Result: If you search for "York", you will not find the document. You must search for the exact string "New
York"


Use keyword when we need to filter out results (EX: status == "PUBLISHED"), need to sort results
Use text when we need to find a sentence containing that word ^S4vkIXbb

    OPERATIONS:

1. CREATE AN INDEX
 
// PUT /books
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  }
}


2. Set a mapping

This lets Elasticsearch know that certain fields should be treated as searchable 
and what types to expect in those fields




 3. Add documents to index 

- If I want to add another i can make another request to _doc endpoint  

- Response:
    







4. Update a document

- Updating a document is similar to creating a document but with a ' id ' field

// PUT /books/_doc/kLEHMYkBq7V9x4qGJOnh
{
  "title": "To Kill a Mockingbird",
  "author": "Harper Lee",
  "description": "A novel about racial injustice in the American South",
  "price": 13.99,
  "publish_date": "1960-07-11",
  "categories": ["Classic", "Fiction"],
  "reviews": [
    {
      "user": "reader3",
      "rating": 5,
      "comment": "Powerful and moving."
    }
  ]
}

    - And this might be appropriate in some instances, but it can be risky! If another process is updating the same document concurrently, you could
overwrite their changes.

    - If we want to guard against this we can use the _version field from above to specify that we only want to update the document if the version
matches. The following request will only update the document if the version is 1. Otherwise it will throw an error

// PUT /books/_doc/kLEHMYkBq7V9x4qGJOnh?version=1

Finally, the _update endpoint (note POST) allows you to update some fields of a document without having to fetch the entire document.

// POST /books/_update/kLEHMYkBq7V9x4qGJOnh
{
  "doc": {
    "price": 14.99
  }
}

    ^8zlXJSt8

    Operations continued :-
 
6. Sort
    
// GET /books/_search
{
  "sort": [
    { "price": "asc" }
  ],
  "query": {
    "match_all": {}
  }
}

- We can also sort by multiple fields
    
// GET /books/_search
{
  "sort": [
    { "price": "asc" },
    { "publish_date": "desc" }
  ],
  "query": {
    "match_all": {}
  }
}

- We can also sort in nested fields

** If we dont specify a order, ElasticSearch sorts by Relevance score.
- Scoring algo is Term frequency-Inverse document frequency ^Tk8Rzb0F

5. SEARCH A DOCUMENT

// GET /books/_search
{
  "query": {
    "match": {
      "title": "Great"
    }
  }
}
    ** Response might look like this
    
    {
  "took": 7,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 2,
      "relation": "eq"
    },
    "max_score": 2.1806526,
    "hits": [
      {
        "_index": "books",
        "_type": "_doc",
        "_id": "1",
        "_score": 2.1806526,
        "_source": {
          "title": "The Great Gatsby",
          "author": "F. Scott Fitzgerald",
          "price": 12.99
        }
      },
      {
        "_index": "books",
        "_type": "_doc",
        "_id": "2",
        "_score": 1.9876543,
        "_source": {
          "title": "Great Expectations",
          "author": "Charles Dickens",
          "price": 10.50
        }
      }
    ]
  }
}

## Explaining Response structure
- took is time taken to get the response
- timed_out is the request timed out
- shards.total is our index "books" is split into 5 nodes/shards
- shards.successfull is for how many nodes were queried sucessfully
- shards.failed is for how many nodes query failed, 
    if it is more than 0 then definitely there is a data miss in response
- hits.index tells the index of the related document
- hits.id is the id of document
- hits.score is the relevance score of the document
- source is for basic details of the document like title, author ^q1ZBMeFB

NOTE :

1.  ElasticSearch request can be GET/POST 

GET /my-index/_search
{
  "query": { "match": { "status": "active" } }
}

- HTTP method GET should not have request body. 
HTTP clients, proxies, firewalls, and caching layers will strip out the body of 
a GET request before it reaches the server.

- If you send a complex JSON query in a GET body and a proxy strips it, 
Elasticsearch will receive an empty search request and return all documents (match_all), 
which is a dangerous bug to debug.

2. Understanding the terms in search query

- match is used when we are trying for only one condition
- must is You are checking multiple conditions that must all be true at the same time 
(Logic: AND), bool is basically the container or wrapper there

// GET /books/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "Great" } },
        { "range": { "price": { "lte": 15 } } }
      ]
    }
  }
}

- path is for telling that we are searching in this particular field of json object 
Ex: path: "reviews" whicih means we are looking into nexted json having field name 
as reviews
{
  "query": {
    "nested": {
      "path": "reviews",
      "query": {
        "bool": {
          "must": [
            { "match": { "reviews.comment": "excellent" } },
            { "range": { "reviews.rating": { "gte": 4 } } }
          ]
        }
      }
    }
  }
} ^fFy4K46T

CLUSTER ARCHITECTURE

ElasticSearch is a distributed search engine. When you spin up ES cluster you are actually spiining up multple ndoes
Nodes can be of 5 types when the instance is started

1. Master Node 
    - it is responsible for coordinating the cluster. It is the only node that can perform cluster-level operations like adding
     or removing nodes and creating or deleting nodes. Think of it like a 'admin'.

2. Data node 
    - it is responsible for storing the data. It's where your data is actually stored. We will have lots of these in the cluster

3. Coordinating node
    - it is responsible for coordinating the search requests across the cluster. Its the node that recieves the search request from 
    the client and sends it to the appropriate nodes. This is the frontend of your cluster

4. Ingest node 
    - it is responsible for data ingestion. Its where your data is transformed and prepared for indexing

5. Machine learning node
    - it is responsible for machine learning tasks.













 ^KpJlS7sh

Q. Why cant we use Elastic search as a primary datasource?

1. Elastic search is not ACID compliant
    - meaning if any transaction is failed in between we cant roolback
    - Ex: If any transaction is failed in between, we cant process refund, that money is permanently debited and
    still not credited
    - This is basically Atomicity of ACID
2. Not consistent

* thats why it is used on top of a database, where any write operations happened on database are replicated to
ElasticSearch to make search queries faster.
* CDC (workers + queue) is used to replicate the data into ES
* In some cases we prefer Atlas search over ES, as with Atlas search its with the primary databse MONGO_DB
so no scenario of replication lag

  
 ^WG3n69gt

- Elasticsearch writes to immutable segment files rather than updating in place.
- Updates create a new document and mark the old one as deleted; deletes are soft until merges.
- Because segments are read-only, searches avoid locking and use memory-mapped I/O for fast, predictable reads.
- Background merges compact segments and clean up deleted docs, which keeps search fast but introduces write amplification ^tEXlntC2

## Embedded Files
b34b5b358a7968c94813e6891b89156ccfd97673: [[Pasted Image 20260615231208_942.png]]

6359675071bf0d49bad77d1e58c24756fa15d66d: [[Pasted Image 20260615231255_301.png]]

f158ae188c8e41d45f6cb7fbaa11271cc38cc954: [[Pasted Image 20260615231308_413.png]]

ae202a20abecba779ced4494cbae30d51ce4af21: [[Pasted Image 20260615231353_692.png]]

%%
## Drawing
```compressed-json
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=
```
%%