# learning


#elastic-search
#kiban dev-tool
#creating index 
PUT /ecommerce
{}

#retrieving index 
GET /_cat/indices?v

#deleting index
DELETE /ecommerce

#Mapping
- to define the data types or format of the fields
- similar to db schema
    
    ##meta fields
    - _id, _type, _uid, _index
    
    ##dynamic mapping
    - automatic detection of types & fields
    
    ##explicit mapping
    - such as date format 
    
    ## mapping gotchas
    - existing type and field mappings cannot be updated, need to create new index and reindex your data.
    - fields are shared across mapping types
        - if a title fields exists in both an employee and article mapping type, the fields must have exactly the same mapping in each type.
        - employee_title, article_title to resolve.


# data types
 - core, complex, geo, specialized
 - core 
    -  string, numeric, data, boolean, binary
 - complex
    - object, array, nested
 - geo
    - geo-point, geo-shape
 - specialized
    - ipv4, completion, token count, attachment

    ## string - full text
        - for relevance search (product description)
        - analyzed by analyzer, string is converted into a list of individual terms before indexing
        - not used for sorting or aggregations
        - usage: find a product by terms in product descriptions
        
    ## string - keywords
        - exact values
        - tags, status, email address
        - not analyzed
        - for filtering
        - for sorting & aggregations
        - usage: find all products for which status='sale'
        
    ## date
        - can be string containing formatted dates
        - long - milliseconds
        - integer - seconds
        - internally, it is long
        
        
        

# meta fields - meta for the document
    - prefixed with underscore
    
    ## identity
        ### _index, _type
        ### _id : id of the document, not indexed, automatically derived from _uid field
        ### _uid : {_type}#{_id}, indexed
    
    ## document source
        ### _source: json we pass to elasticsearch, not indexed, not searchable, can be disabled to save storage space
        ### _size: size of _source field in bytes. Need to install mapper-size plugin to see the size.
                 sudo bin/plugin install mapper-size
                 
    ## indexing meta fields
        ### _all 
                - concatenated value of all other fields using space
                - analyzed, indexed, but not stored
                - can be searched, but not retrieved
                - usage: search value in document without knowing which field contains that value
                
        ### _field_names
                - indexes the field names
                - for exists and missing queries
                
    ## routing meta fields
            ### _PARENT
                    - for parent-child relationship between documents in an index
                    - one mapping type will be parent of another mapping type
                    - similar to foreign key in relational db
                    "mapping":{
                        "employee":{
                            "_parent":{
                                "type":"person"
                            }
                        }
                    }
                    
                    
            ### _routing
                - used to route a document to a particular shard within an index
                
    #_meta:
        - application specific mapping type
        
        
  
  # add mappings
  PUT /ecommerce
  {
    "mapping":{
        "product":{
            "properties":{
                "name":{
                    "type":"string"
                },
                "price":{
                    "type":"double"
                },
                
                "categories":{
                    "type":"nested",
                    "properties":{
                        "name":{
                            "type":"string"
                        }
                    }
                },
                "tags":{
                    "type":"string"
                }
            }
        }
    }
  }
       
# bulk loading for json data
curl -XPOST http://localhost:9200/ecommerce/product/_bulk --data-binary "@test-data.json"


# query DSL for product with name="pasta"
GET http://localhost:9200/ecommerce/product/_search
{
    "query":{
        "match":{
            "name": "pasta"
        }
    }
}

/ecommerce/product where name="pasta"


#type of queries
    ## Leaf
        - particular value for particular field

    ## compound
        - combine multiple queries using boolean logic
        
    ## full text queries
        - on full text fields like product description
        - values are analyzed when adding documents
            - like removing stop words, tokenizing, lowercasing
    
    ## term level
        - exact matching of values
        - for structured data like dates, numbers
            ex: persons born between year 1990 and 1995
            

# joining queries
    ## nested query
        -   
    
    ## has_child and has_parent queries
    
# geo queries
    ## geo_point
    ## geo_shape
    
    


# searching using query string
GET /ecommerce/product/_search?q=*
GET /ecommerce/product/_search?q=pasta //this will search pasta in all fields of the type product
GET /ecommerce/product/_search?q=name:pasta

    ## boolean query
    GET /ecommerce/product/_search?q=name:(pasta AND spaghetti)
    GET /ecommerce/product/_search?q=name:(+pasta AND -spaghetti)
    GET /ecommerce/product/_search?q=name:+pasta -spaghetti

    ## phrase query
    defaut boolean operator is OR
    GET /ecommerce/product/_search?q=name:pasta spaghetti
    GET /ecommerce/product/_search?q=name:"pasta spaghetti"
    GET /ecommerce/product/_search?q=name:"spaghetti pasta"

GET /_analyze?analyzer=standard&text=Pasta - Spaghetti


# query DSL - 
GET /ecommerce/product/_search
{
    "query":{
        "match_all":{}
    }
}

    ##match query
    GET /ecommerce/product/_search
    {
        "query":{
            "match":{
                "name":"pasta"
            }
        }
    }

    ##multi_match query
    GET /ecommerce/product/_search
    {
        "query":{
            "multi_match":{
                "query":"pasta",
                "fields":["name","description"]
            }
        }
    }

    ##match_phrase query
    GET /ecommerce/product/_search
    {
        "query":{
            "match_phrase":{
                "name":"pasta spaghetti"
            }
        }
    }
    
    
#term level queries: for exact matching, not analyzed
GET /ecommerce/product/_search
{
        "query":{
            "term":{
                "status":"active"
            }
        }
    }
    
GET /ecommerce/product/_search
{
        "query":{
            "terms":{
                "status":["active", "inactive"]
            }
        }
    }    

    ## range queries
    GET /ecommerce/product/_search
    {
        "query":{
            "range":{
                "quantity":{
                    "gte":1,
                    "lte":10
                }
            }
        }
    }

#prefix query : "past" will match "pasta"

#wild card queries: using wild cards *, ?

#regex query: using regular expression

#exists query: non-null value present for the field

#missing query: field is missing or null in the document


#compound queries:
    ## must query: like AND
    GET /ecommerce/product/_search
    {
        "query":{
            "bool":{
                "must":[
                    {"match":{"name":"pasta"}},
                    {"match":{"name":"spaghetti"}}
                ]
            }
        }
    }

    ##must_not query
    GET /ecommerce/product/_search
    {
        "query":{
            "bool":{
                "must":[
                    {"match":{"name":"pasta"}}
                ],
                "must_not":[
                    
                    {"match":{"name":"spaghetti"}}
                ]
            }
        }
    }
    
    
    ##should query : if matches, document's relevancy score's increases. behaves like logical OR.
    
    GET /ecommerce/product/_search
    {
        "query":{
            "bool":{
                "must":[
                    {"match":{"name":"pasta"}}
                ],
                "should":[
                    
                    {"match":{"name":"spaghetti"}}
                ]
            }
        }
    }
    
    
    ##function_score query:you can provide a function which will modify score of the document
    
    ##boosting query:reduce the score of document
    
    
#seaching across index & types
PUT /myfoodblog/recipe/1
{
    "name":"Past Quattro Formaggi",
    "description":"First you boil the pasta, then you add the chesse.",
    "ingredients":[
    {"name":"Pasta", "amount":"500g"},
    {"name":"Fontina cheese", "amount":"100g"},
    {"name":"Parmesan cheese", "amount":"100g"},
    {"name":"Romano cheese", "amount":"100g"},
    {"name":"Gorgonzola cheese", "amount":"100g"}
    ]
}


GET /ecommerce,myfoodblog/product,recipe/_search?q=pasta&size=15

#exclude ecommerce index, include myfoodblog index
GET /-ecommerce,%2Bmyfoodblog/product,recipe/_search?q=pasta&size=15

#how to search all types
GET /ecommerce/_search?q=pasta

#how to search all indexes
GET /_all/product/_search?q=pasta

#across search
GET /_search?q=pasta

#fuzzy searches
GET /ecommerce/product/_search?q=past~1
GET /ecommerce/product/_search
{
 "query": {
   "match": {
     "name": {
       "query": "past",
       "fuzziness": 1
     }
   }
 }   
}

GET /ecommerce/product/_search
{
 "query": {
   "match": {
     "name": {
       "query": "past",
       "fuzziness": "AUTO"
     }
   }
 }   
}


#proximity search


#boosting
GET /ecommerce/product/_search?q=name:pasta spaghetti^2.0
GET /ecommerce/product/_search?q=name:"pasta spaghetti"^2.0


#filter context: to exclude documents
GET /ecommerce/product/_search
{
 "query": {
   "bool": {
     "must": [
       {
         "match": {
           "name":"pasta"
         }
       }
     ],
     "filter":[
        {
          "range":{
            "quantity":{
              "gte":10,
              "lte":15
            }
          }
        }  
      ]
   }
 }   
}


#size:
GET /ecommerce/product/_search?q=name:pasta&size=2
GET /ecommerce/product/_search
{
 "query": {
       "match": {
           "name":"pasta"
         }
       },
       "size": 2
   
}


#pagination: from index parameter, starts with 0, default value is 0
GET /ecommerce/product/_search?q=name:pasta&size=5&from=5
GET /ecommerce/product/_search
{
 "query": {
       "match": {
           "name":"pasta"
         }
       },
       "size": 5,
       "from": 10
   
}

#sorting: 
GET /ecommerce/product/_search
{
 "query": {
       "match": {
           "name":"pasta"
         }
       },
       "sort": [{
         "quantity":{
           "order": "desc"
         }
       }]
}


#aggregation: group by, sum
metric, bucket, pipeline

##single value aggregation
GET /ecommerce/product/_search
{
 "query": {
       "match_all": {}
  }, 
       "size": 0,
       "aggs": {
         "quantity_sum": {
           "sum": {
             "field": "quantity"
           }
         }
       }
}


GET /ecommerce/product/_search
{
 "query": {
       "match": {
         "name": "pasta"
       }
  }, 
       "size": 0,
       "aggs": {
         "quantity_sum": {
           "sum": {
             "field": "quantity"
           }
         }
       }
}


GET /ecommerce/product/_search
{
 "query": {
       "match": {
         "name": "pasta"
       }
  }, 
       "size": 0,
       "aggs": {
         "avg_quantity": {
           "avg": {
             "field": "quantity"
           }
         }
       }
}


GET /ecommerce/product/_search
{
 "query": {
       "match": {
         "name": "pasta"
       }
  }, 
       "size": 0,
       "aggs": {
         "min_quantity": {
           "min": {
             "field": "quantity"
           }
         }
       }
}


GET /ecommerce/product/_search
{
 "query": {
       "match": {
         "name": "pasta"
       }
  }, 
       "size": 0,
       "aggs": {
         "max_quantity": {
           "max": {
             "field": "quantity"
           }
         }
       }
}

##multi value aggregation
GET /ecommerce/product/_search
{
 "query": {
       "match": {
         "name": "pasta"
       }
  }, 
       "size": 0,
       "aggs": {
         "stats_quantity": {
           "stats": {
             "field": "quantity"
           }
         }
       }
}


# bucket aggregation
GET /ecommerce/product/_search
{
 "query": {
       "match_all": {
         
       }
  }, 
       "size": 0,
       "aggs": {
         "quantity_ranges": {
           "range": {
             "field": "quantity",
             "ranges": [
               {
                 "from": 1,
                 "to": 50
               },
               {
                 "from": 50,
                 "to": 100
               }
             ]
           }
         }
       }
}


#bucket + metric for each bucket
GET /ecommerce/product/_search
{
 "query": {
       "match_all": {
         
       }
  }, 
       "size": 0,
       "aggs": {
         "quantity_ranges": {
           "range": {
             "field": "quantity",
             "ranges": [
               {
                 "from": 1,
                 "to": 50
               },
               {
                 "from": 50,
                 "to": 100
               }
             ]
           },
           "aggs":{
             "quantity_stats":{
               "stats": {
                 "field": "quantity"
               }
             }
           }
         }
       }
}
