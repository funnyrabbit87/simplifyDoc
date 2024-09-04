# Index modules
### Index Shard Allocation
#### Index-level shard allocation filtering
elasticsearch.yml
```
 node.attr.size: medium
 `./bin/elasticsearch -Enode.attr.size=medium
 PUT test/_settings
{
  "index.routing.allocation.include.size": "big,medium"
}
index.routing.allocation.include.{attribute}  
Assign the index to a node whose {attribute} has at least one of the comma-separated values.  
index.routing.allocation.require.{attribute}  
Assign the index to a node whose {attribute} has all of the comma-separated values.  
index.routing.allocation.exclude.{attribute}  
Assign the index to a node whose {attribute} has none of the comma-separated values.  
 ```
#### Delaying allocation when a node leaves
If a node is not going to return and you would like Elasticsearch to allocate the missing shards immediately, just update the timeout to 0:
```
PUT _all/_settings
{
  "settings": {
    "index.unassigned.node_left.delayed_timeout": "5m"
  }
}
```
#### Index recovery prioritization
```
PUT index_3
{
  "settings": {
    "index.priority": 10
  }
}

```

#### Index-level data tier allocation filtering
index.routing.allocation.include._tier_preference  For example, if you set index.routing.allocation.include._tier_preference to data_warm,data_hot, the index is allocated to the warm tier if there are nodes with the data_warm role. If there are no nodes in the warm tier, but there are nodes with the data_hot role, the index is allocated to the hot tier.

### Preloading data into the file system cache
elasticsearch.yml
```
index.store.preload: ["nvd", "dvd"]
PUT /my-index-000001
{
  "settings": {
    "index.store.preload": ["nvd", "dvd"]
  }
}
```
### Index Sorting
By default in Elasticsearch a search request must visit every document that matches a query to retrieve the top documents sorted by a specified sort. Though when the index sort and the search sort are the same it is possible to limit the number of documents that should be visited per segment to retrieve the N top ranked documents globally.
减少搜索量，加快搜索
```
PUT my-index-000001
{
  "settings": {
    "index": {
      "sort.field": "date", 
      "sort.order": "desc"  
    }
  },
  ....
}
```

# Mapping
### Dynamic mapping
不定义index，直接添加文档数据。
#### Dynamic field mapping
| json type    | 	"dynamic":"true"  | 	"dynamic":"runtime"    |
|------|-----|-----|
| null     | 	No field added |	No field added   |
| true or false     | boolean | boolean   |
| double   | float | double    |
| long   | long | long  |
| object | object    | No field added     |
| array | Depends on the first non-null value in the array    | Depends on the first non-null value in the array   |
| string that passes date detection | date    | date     |
| string that passes numeric detection | float or long    | double or long    
| string that passes date or numeric NOT detection | text with a .keyword sub-field    |  keyword    |

 "date_detection": false  The default value for dynamic_date_formats is:
[ "strict_date_optional_time","yyyy/MM/dd HH:mm:ss Z||yyyy/MM/dd Z"]
"numeric_detection": true

### Field data types
Arrays Binary Boolean Date Geopoint Geoshape IP Keyword Numeric(long integer short double...) Point Text

### Metadata fields
**_doc_count** 的字段，显示每个桶中聚合和分区的文档数量  
**_field_names** field used to index the names of every field in a document that contains any value other than null.   
 **_ignored** field indexes and stores the names of every field in a document that has been ignored when the document was indexed. This can, for example, be the case when the field was malformed and ignore_malformed was turned on, or when a keyword fields value exceeds its optional ignore_above setting. 有field 因为 ignore_above ignore_malformed 导致不能被  index  
**_id** 文档id  
**_index** 文档的索引名称  
**_meta** custom meta data associated with it   
**_routing**  A document is routed to a particular shard in an index 通过自定义 _routing 值，你可以直接影响文档的存储位置，这对于优化性能、减少数据倾斜（skew）或实现多租户（multi-tenancy）等场景非常有用。
```
routing_factor = num_routing_shards / num_primary_shards
shard_num = (hash(_routing) % num_routing_shards) / routing_factor
PUT /my_index/_doc/1?routing=user123
{
  "name": "John Doe",
  "age": 30
}

GET my-index-000001/_search?routing=user1,user2 
{
  "query": {
    "match": {
      "title": "document"
    }
  }
}

routing_value = hash(_routing) + hash(_id) % routing_partition_size
shard_num = (routing_value % num_routing_shards) / routing_factor
```
num_routing_shards is the value of the index.number_of_routing_shards index setting. num_primary_shards is the value of the index.number_of_shards index setting.  

**_source** The _source field contains the original JSON document body that was passed at index time. The _source field itself is not indexed (and thus is not searchable), but it is stored so that it can be returned when executing fetch requests, like get or search.

**_tier** 
```
GET index_1,index_2/_search
{
  "query": {
    "terms": {
      "_tier": ["data_hot", "data_warm"] 
    }
  }
}
```

#### Mapping parameters
**analyzer** 
Only text fields support the analyzer mapping parameter.

**boost** count more towards the relevance score .影响算分  
**coerce** 数据类型纠错  
**copy_to** copy_to parameter allows you to copy the values of multiple fields into a group field
```
PUT my-index-000001
{
  "mappings": {
    "properties": {
      "first_name": {
        "type": "text",
        "copy_to": "full_name" 
      },
      "last_name": {
        "type": "text",
        "copy_to": "full_name" 
      },
      "full_name": {
        "type": "text"
      }
    }
  }
}
```
**doc_values**  列式存储方便统计。Doc values are supported on almost all field types, with the notable exception of text and annotated_text fields.  

**dynamic** 
| 参数    | 描述  | 默认值    |
|------|-----|-----|
| **true** | New fields are added to the mapping (default).   |     |
| **runtime** | New fields are added to the mapping as runtime fields. These **fields are not indexed, and are loaded from _source at query time**.   |     |
| **false** | New fields are ignored. These fields will not be indexed or searchable, but will still appear in the _source field of returned hits. These fields will not be added to the mapping, and new fields must be added explicitly.   |     |
| **strict** | If new fields are detected, an exception is thrown and the document is rejected. New fields must be explicitly added to the mapping. |     |

**enabled** object类型的 field是否被索引

**format** 格式化字段的值
**ignore_above** 超多的部分不索引 Strings longer than the ignore_above setting will not be indexed or stored.   
**ignore_malformed**  The malformed field is not indexed  
**index** 字段是否被索引 accepts true or false and defaults to true  
**index_options** 
| 参数    | 描述  | 默认值    |
|------|-----|-----|
| **docs** | Only the doc number is indexed. Can answer the question Does this term exist in this field?  |     |
| **freqs** | Doc number and term frequencies are indexed. Term frequencies are used to score repeated terms higher than single terms.    |     |
| **positions** | Doc number, term frequencies, and term positions (or order) are indexed. Positions can be used for proximity or phrase queries. (default)   |     |
| **offsets** |Doc number, term frequencies, positions, and start and end character offsets (which map the term back to the original string) are indexed. Offsets are used by the unified highlighter to speed up highlighting. |     | 

**index_phrases** 参考phrases是什么  
**index_prefixes**  min_chars max_chars  
**meta**  Metadata attached to the field  
**fields** 
```
PUT my-index-000001
{
  "mappings": {
    "properties": {
      "city": {
        "type": "text",
        "fields": {
          "raw": { 
            "type":  "keyword"
          }
        }
      }
    }
  }
}
```
**normalizer** The normalizer property of keyword fields is similar to analyzer except that it guarantees that the analysis chain produces a single token. applied prior to indexing the keyword as well as at search-time when the keyword field is searched via a query parser such as the match query or via a term-level query such as the term query.  

**norms** used at query time in order to compute the score of a document relatively to a query. 耗磁盘空间 可禁用
```
PUT my-index-000001/_mapping
{
  "properties": {
    "title": {
      "type": "text",
      "norms": false
    }
  }
}
```
**null_value** A null value cannot be indexed or searched. 能通过这个把null 转成其他的value  
**position_increment_gap**  defaults to 100. 和slot类似
**properties** 定义index mapping的时候常用  
**search_analyzer**  use a different analyzer at search time, such as when using the edge_ngram tokenizer for autocomplete or when using search-time synonyms. By default, queries will use the analyzer defined in the field mapping, but this can be overridden with the search_analyzer setting
**store** 对字段值存储，而不是从_source中取。 相当于对这些字段多存一份  
#### Mapping limit settings
index.mapping.total_fields.limit default value is 1000.  
index.mapping.field_name_length.limit  Default is Long.MAX_VALUE (no limit).
# Text analysis
### analyzer
 contains three lower-level building blocks: **character filters**, 
 **tokenizers**, and **token filters**.  
**Character filters** receives the original text as a stream of characters and can transform the stream by adding, removing, or changing characters.  zero or more  
**tokenizers**   receives a stream of characters, breaks it up into individual tokens (usually individual words), and outputs a stream of tokens. exactly one  
**token filters** A token filter receives the token stream and may add, remove, or change tokens. For example, a lowercase token filter converts all tokens to lowercase. zero or more  
| custom analyzer  | configuration   |  |
| --- | --- | --- |
| type | Analyzer type. Accepts built-in analyzer types. For custom analyzers, use custom or omit this parameter. | custom |
| tokenizer | A built-in or customised tokenizer. (Required)  | standard |
| char_filter | An optional array of built-in or customised character filters.  |
| filter | An optional array of built-in or customised token filters.  | html_strip |
| position_increment_gap | When indexing an array of text values, Elasticsearch inserts a fake "gap" between the last term of one value and the first term of the next value to ensure that a phrase query doesn’t match two terms from different array elements. Defaults to 100. See position_increment_gap for more.  | lowercase  asciifolding |
|   |   |  |

##### Specify the analyzer for a field
```
PUT my-index-000001
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "whitespace"
      }
    }
  }
}
```
##### Specify the default analyzer for an index
```
PUT my-index-000001
{
  "settings": {
    "analysis": {
      "analyzer": {
        "default": {
          "type": "simple"
        }
      }
    }
  }
}
```

#### Built-in analyzer reference
**Fingerprint**  
**Keyword** The keyword analyzer is a “noop” analyzer which returns the entire input string as a single token.  
**Language** A set of analyzers aimed at analyzing specific language text.  
**Pattern** The pattern analyzer uses a regular expression to split the text into terms. The regular expression should match the token separators not the tokens themselves. The regular expression defaults to \W+ (or all non-word characters).  
| 参数    | 描述  | 默认值    |
|------|-----|-----|
| **pattern**     | A Java regular expression, defaults to `\W+`.                                                                                                         | `\W+`     |
| **flags**       | Java regular expression flags. Flags should be pipe-separated, e.g., `"CASE_INSENSITIVE|COMMENTS"`.                                                  | _None_    |
| **lowercase**   | Should terms be lowercased or not. Defaults to `true`.                                                                                                | `true`    |
| **stopwords**   | A pre-defined stop words list like `_english_` or an array containing a list of stop words. Defaults to `_none_`.                                     | `_none_`  |
| **stopwords_path** | The path to a file containing stop words.    | _None_    |

The pattern analyzer consists of:   
Tokenizer
 - Pattern Tokenizer  
  
Token Filters
- Lower Case Token Filter  
- Stop Token Filter (disabled by default)

**Simple** The simple analyzer breaks text into tokens at any non-letter character, such as numbers, spaces, hyphens and apostrophes, discards non-letter characters, and changes uppercase to lowercase.  
**Standard** The standard analyzer is the default analyzer which is used if none is specified. It provides grammar based tokenization (based on the Unicode Text Segmentation algorithm, as specified in Unicode Standard Annex #29) and works well for most languages.   

| 参数    | 描述  | 默认值    |
|------|-----|-----|
| **max_token_length**     | The maximum token length. If a token is seen that exceeds this length then it is split at max_token_length intervals. | 255    |
| **stopwords**       | A pre-defined stop words list like _english_ or an array containing a list of stop words.    |  \_none_    |
| **stopwords_path**   |The path to a file containing stop words.    |      |

**Stop** The stop analyzer is the same as the simple analyzer but adds support for removing stop words. It defaults to using the _english_ stop words.  
| 参数    | 描述  | 默认值    |
|------|-----|-----|
| **stopwords**       | A pre-defined stop words list like _english_ or an array containing a list of stop words.    |  \_none_    |
| **stopwords_path**   |The path to a file containing stop words.    |      |  

It consists of:  
Tokenizer
 - Pattern Tokenizer  
  
Token Filters
- Lower Case Token Filter  

**Whitespace**  The whitespace analyzer breaks text into terms whenever it encounters a whitespace character.  
It consists of:  
Tokenizer
 - Whitespace Tokenizer  

#### Built-in Tokenizer  reference  
**Character group**  The char_group tokenizer breaks text into terms whenever it encounters a character which is in a defined set. It is mostly useful for cases where a simple custom tokenization is desired, and the overhead of use of the pattern tokenizer is not acceptable.  
| 参数    | 描述  |
|------|-----|
| **tokenize_on_chars**       | A list containing a list of characters to tokenize the string on. Whenever a character from this list is encountered, a new token is started. This accepts either single characters like e.g. -, or character groups: whitespace, letter, digit, punctuation, symbol.    | 
| **max_token_length**   |The maximum token length. If a token is seen that exceeds this length then it is split at max_token_length intervals. Defaults to 255.  |

```
POST _analyze
{
  "tokenizer": {
    "type": "char_group",
    "tokenize_on_chars": [
      "whitespace",
      "-",
      "\n"
    ]
  },
  "text": "The QUICK brown-fox"
}
```  

**Classic**  a grammar based tokenizer that is good for English language documents. This tokenizer has heuristics for special treatment of acronyms, company names, email addresses, and internet host names.  
| 参数    | 描述  |
|------|-----|
| **max_token_length**   | The maximum token length. If a token is seen that exceeds this length then it is split at max_token_length intervals. Defaults to 255.  |

**Edge n-gram**  | first breaks text down into words whenever it encounters one of a list of specified characters, then it emits N-grams of each word where the start of the N-gram is anchored to the beginning of the word. 从**单词的开头**开始截取，而n-gram会从单词每个位置截取
| 参数    | 描述  | 默认值    |
|------|-----|-----|
| **min_gram**       | Minimum length of characters in a gram    |   1    |
| **max_gram**   |Maximum length of characters in a gram.    |  2    |  
| **token_chars**   |Character classes that should be included in a token. Elasticsearch will split on characters that don’t belong to the classes specified. Defaults to [] (keep all characters).Character classes may be any of the following: <br>  letter —  for example a, b, ï or 京  <br>  digit —  for example 3 or 7  <br>    whitespace —  for example " " or "\n"  <br>  punctuation — for example ! or " <br>   symbol —  for example $ or √<br>  custom —  custom characters which need to be set using the - custom_token_chars setting. |   [] keep all characters   |  

**Keyword**  The keyword tokenizer is a “noop” tokenizer that accepts whatever text it is given and outputs the exact same text as a single term. It can be combined with token filters to normalise output, e.g. lower-casing email addresses. buffer_size The number of characters read into the term buffer in a single pass. Defaults to 256  

**Letter**  The letter tokenizer breaks text into terms whenever it encounters a character which is not a letter. It does a reasonable job for most European languages, but does a terrible job for some Asian languages, where words are not separated by spaces.
**Lowercase** 
```
POST _analyze
{
  "tokenizer": "letter",//Lowercase
  "text": "The 2 QUICK Brown-Foxes jumped over the lazy dog's bone."
}
[ The, QUICK, Brown, Foxes, jumped, over, the, lazy, dog, s, bone ]
//Lowercase
[ the, quick, brown, foxes, jumped, over, the, lazy, dog, s, bone ]
```
**N-gram**
```
POST _analyze
{
  "tokenizer": "edge_ngram",
  "text": "Quick Fox"
}
[ Q, Qu ]
// ngram
[ Q, Qu, u, ui, i, ic, c, ck, k, "k ", " ", " F", F, Fo, o, ox, x ]
```

**Path hierarchy**  takes a hierarchical value like a filesystem path, splits on the path separator, and emits a term for each component in the tree.
```
POST _analyze
{
  "tokenizer": "path_hierarchy",
  "text": "/one/two/three"
}
[ /one, /one/two, /one/two/three ]
```
| 参数    | 描述  | 默认值    |
|------|-----|-----|
| **delimiter**       | The character to use as the path separator. Defaults to /. |    /   |
| **replacement**   |An optional replacement character to use for the delimiter. Defaults to the delimiter.   |  2    |  
| **buffer_size**   |The number of characters read into the term buffer in a single pass. Defaults to 1024.. | 1024   |  
| **reverse**   |If set to true, emits the tokens in reverse order.. | false   | 
| **skip**   |The number of initial tokens to skip. Defaults to 0. | 0   | 

**Pattern**  uses a regular expression to either split text into terms whenever it matches a word separator, or to capture matching text as terms. Default to \W+
| 参数    | 描述  | 默认值    |
|------|-----|-----|
| **pattern**       | The character to use as the path separator. Defaults to /. |    \W+   |
| **flags**   |Flags should be pipe-separated, eg "CASE_INSENSITIVE|COMMENTS". |     |  
| **buffer_size**   |The number of characters read into the term buffer in a single pass. Defaults to 1024.. | 1024   |  
| **group**   |Which capture group to extract as tokens. Defaults to -1 (split).. | -1   | 

**Simple pattern** parameter pattern  Default to empty string

**Simple pattern split**  uses a regular expression to split the input into terms at pattern matches. The set of regular expression features it supports is more limited than the pattern tokenizer, but the tokenization is generally faster.

**Standard** provides grammar based tokenization (based on the Unicode Text Segmentation algorithm, as specified in Unicode Standard Annex #29) and works well for most languages. 
max_token_length. Defaults to 255.

**UAX URL email** is like the standard tokenizer except that it recognises URLs and email addresses as single tokens. **能把避免拆分email url等** max_token_length. Defaults to 255.

**Whitespace**  breaks text into terms whenever it encounters a whitespace character.

#### Token filter
**ASCII folding** Converts alphabetic, numeric, and symbolic characters that are not in the Basic Latin Unicode block (first 127 ASCII characters) to their ASCII equivalent, if one exists. For example, the filter changes à to a.  
**Edge n-gram** 同上
**Keep types** Keeps or removes tokens of a specific type
```
GET _analyze
{
  "tokenizer": "standard",
  "filter": [
    {
      "type": "keep_types",
      "types": [ "<NUM>" ]
    }
  ],
  "text": "1 quick fox 2 lazy dogs"
}
[ 1, 2 ]
```
Token types are set by the tokenizer when converting characters to tokens. Token types can vary between tokenizers.

For example, the standard tokenizer can produce a variety of token types, including <ALPHANUM>, <HANGUL>, and <NUM>. Simpler analyzers, like the lowercase tokenizer, only produce the word token type.

Certain token filters can also add token types. For example, the synonym filter can add the <SYNONYM> token type.

| 参数    | 描述  | 默认值    |
|------|-----|-----|
| **types**       | (Required, array of strings) List of token types to keep or remove. |      |
| **mode**   |(Optional, string) Indicates whether to keep or remove the specified token types. Valid values are: <br> include (Default) Keep only the specified token types.  <br> exclude remove specified token types|     |  

**Length**  Removes tokens shorter or longer than specified character lengths. For example, you can use the length filter to exclude tokens shorter than 2 characters and tokens longer than 5 characters. min Default to 0, max Default to Integer.MAX_VALUE

**Lowercase**   text to lowercase.   
**N-gram**  同上  
**Remove duplicates**  去重
**Reverse**   反转
**Stop** 去除the a and等stop词  
**Trim**  Removes leading and trailing whitespace from each token in a stream
**Truncate** exceed a specified character limit. This limit defaults to 10 but can be customized using the length parameter.  
**Unique** Removes duplicate tokens from a stream
**Uppercase**   text to uppercase.    

#### Character filters reference
**HTML strip** html_strip  Strips HTML elements from a text and replaces HTML entities with their decoded value  
**Mapping**  把一个字符转成另一个  
**Pattern replace character** 用正则替换

### Normalizers
similar to analyzers except that they may only emit a single token. only accept a subset of the available char filters and token filters
```
"normalizer": {
        "my_normalizer": {
          "type": "custom",
          "char_filter": ["quote"],
          "filter": ["lowercase", "asciifolding"]
        }
      }
```

# Index templates
An index template is a way to tell Elasticsearch how to configure an index when it is created.  Templates are configured prior to index creation  includ:  
Index Templates 适用于为特定类型的索引配置全局设置。  
Component Templates 适用于定义通用的、可复用的配置片段，以供多个索引模板使用。  

可组合模板优先于传统模板, 创建索引时的显式设置优先于索引模板中的设置,索引模板中的设置优先于组件模板中的设置,当新数据流或索引匹配多个索引模板时，优先使用优先级最高的索引模板

# Aliases
An alias is a secondary name for a group of data streams or indices. Most Elasticsearch APIs accept an alias in place of a data stream or index name.

You can change the data streams or indices of an alias at any time. If you use aliases in your application’s Elasticsearch requests, you can **reindex data with no downtime or changes to your app’s code**.

# Search data
#### Collapse search

根据某个field聚合查询结果，只返回指定条数  
#### Filter search results
Use a boolean query with a filter clause. Search requests apply boolean filters to both search hits and aggregations.
```
GET /shirts/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "color": "red"   }},
        { "term": { "brand": "gucci" }}
      ]
    }
  }
}
```
Use the search API’s post_filter parameter. Search requests apply post filters only to search hits, not aggregations. 
#### Highlighting
 Elasticsearch supports three highlighters:unified, plain, and fvh (fast vector highlighter)
#### Paginate search results

```
GET /_search
{
  "from": 5,
  "size": 20,
  "query": {
    "match": {
      "user.id": "kimchy"
    }
  }
}
```
Search after 使用PIT确保查询一致性，相当于snapshot，然后用search_after做翻页。  
 point in time (PIT)  
 ```
 GET /_search
{
  "size": 10000,
  "query": {
    "match" : {
      "user.id" : "elkbee"
    }
  },
  "pit": {
    "id":  "46ToAwMDaWR5BXV1aWQyKwZub2RlXzMAAAAAAAAAACoBYwADaWR4BXV1aWQxAgZub2RlXzEAAAAAAAAAAAEBYQADaWR5BXV1aWQyKgZub2RlXzIAAAAAAAAAAAwBYgACBXV1aWQyAAAFdXVpZDEAAQltYXRjaF9hbGw_gAAAAA==", 
    "keep_alive": "1m"
  },
  "sort": [
    {"@timestamp": {"order": "asc", "format": "strict_date_optional_time_nanos"}}
  ],
  "search_after": [                                
    "2021-05-20T05:30:04.832Z",
    4294967298
  ],
  "track_total_hits": false                        
}
 ```

#### Scroll search results
scroll parameter in the query string _search?scroll=1m size parameter allows you to configure the maximum number of hits to be returned with each batch of results  
#### Retrieve selected fields from a search  
Use the fields option to extract the values of fields present in the index mapping  
Use the _source option if you need to access the original data that was passed at index time  

Doc Value Fields Doc Value是为排序、聚合等操作而优化的数据结构。它将字段的值以列式存储的方式保存在磁盘上
Stored Fields 来存储原始的文档数据。这些字段的数据存储在Elasticsearch的原始倒排索引（Inverted Index）中，可以通过_source或_stored_fields来直接获取文档的字段值。


Rescore filtered search results  The query rescorer executes a second query only on the Top-K results returned by the query and post_filter phases. 让结果更加精确

# Query DSL
#### Query and filter context
**Query** context In the query context, a query clause answers the question “How well does this document match this query clause?” Besides deciding whether or not the document matches, the query clause also calculates a relevance score in the _score metadata field.  
**Filter** context In a filter context, a query clause answers the question “Does this document match this query clause?” The answer is a simple Yes or No — no scores are calculated  

#### Compound queries
**Boolean** 
| 参数    | 描述  | 默认值    |
|------|-----|-----|
| **must**       | The clause (query) must appear in matching documents and will contribute to the score. |     |
| **filter**   |The clause (query) must appear in matching documents. However unlike must the score of the query will be ignored. Filter clauses are executed in filter context, meaning that scoring is ignored and clauses are considered for caching.  |     |  
| **should**   |The clause (query) should appear in the matching document. |    |  
| **must_not**   |The clause (query) must not appear in the matching documents. Clauses are executed in filter context meaning that scoring is ignored and clauses are considered for caching. Because scoring is ignored, a score of 0 for all documents is returned. |   | 

**Boosting** 
| 参数    | 描述  | 默认值    |
|------|-----|-----|
| **positive**       | (Required, query object) Query you wish to run. Any returned documents must match this query. |     |
| **negative**   |(Required, query object) Query used to decrease the relevance score of matching documents.  |     |  
| **negative_boost**   |(Required, float) Floating point number between 0 and 1.0 used to decrease the relevance scores of documents matching the negative query. |    |  

**Constant score query**  
Wraps a filter query and returns every matching document with a relevance score equal to the boost parameter value.
对filter封装 ，score是个固定值,因为filter不会打分.
| 参数    | 描述  | 默认值    |
|------|-----|-----|
| **filter**       | (Required, query object) Query you wish to run. Any returned documents must match this query. |     |
| **negative**   |(Optional, float) Floating point number used as the constant relevance score for every document matching the filter query. Defaults to 1.0. |   1.0.  |   

**Disjunction max query**
use the dis_max to search. 组合多个query
```
GET /_search
{
  "query": {
    "dis_max": {
      "queries": [
        { "term": { "title": "Quick pets" } },
        { "term": { "body": "Quick pets" } }
      ],
      "tie_breaker": 0.7
    }
  }
}
```
| 参数    | 描述  | 默认值    |
|------|-----|-----|
| **queries**       | (Required, array of query objects) Contains one or more query clauses. Returned documents must match one or more of these queries |     |
| **tie_breaker**   |(Optional, float) Floating point number between 0 and 1.0 used to increase the relevance scores of documents matching multiple query clauses. |   0.0.  |

#### Full text queries
**intervals** 复杂的组合查询，包含match，prefix，wildcard，fuzzy，all_of，any_of
**match** match The match query is of type boolean. It means that the text provided is analyzed and the analysis process constructs a boolean query from the provided text.常用参数
 | 参数    | 描述  | 默认值    |
|------|-----|-----|
| **query**       |(Required) Text, number, boolean value or date you wish to find in the provided |     |
| **analyzer**   |(Optional, string) Analyzer used to convert the text in the query value into tokens. Defaults to the index-time analyzer mapped for the <field>. If no analyzer is mapped, the index’s default analyzer is used. |   |
| **operator**   |(Optional, string) Boolean logic used to interpret text in the query value. AND OR  | OR   |

**match_bool_prefix**   它结合了 match  可以用于解决输入不完整的情况，允许搜索包含多个词的短语，其中部分词是前缀匹配的。

**match_phrase**  所谓的phrase就是完整的一句，slot 0 表示里面所有单词顺序必须一致，1 就是允许一个单词顺序和查询的不一样
**match_phrase_prefix**  Returns documents that contain the words of a provided text, in the same order as provided. 和上面的prefix一样  
**combined_fields**    查询支持搜索多个文本字段，就好像它们的内容已被索引到一个组合字段中一样  
combined_fields 查询提供了一种在多个文本字段之间进行匹配和评分的原则性方法。为了支持这一点，**它要求所有字段都具有相同的搜索分析器**。如果您想要一个处理不同类型字段（如关键字或数字）的单个查询，那么 multi_match 查询可能更合适。它支持文本和非文本字段，并接受不共享相同分析器的文本字段。**multi_match的多字段可以使用不同的搜索分析器**

**multi_match** builds on the match query to allow multi-field queries: If no fields are provided, the multi_match query **defaults to the index.query.default_field index settings**, which in turn defaults to *. * extracts all fields in the mapping that are eligible to term queries and filters the metadata fields. 参数常用type，best_fields 多个字段中选择单个匹配最佳的字段，文档的分数将基于这个字段的最佳匹配来计算 most_fields模式会将所有匹配字段的分数加在一起
**query_string**  When running the following search, the query_string query splits (new york
city) OR (big apple) into two parts: new york city and big apple. The content field’s analyzer then independently converts each part into tokens before returning matching documents. Because the query syntax does not use whitespace as an operator, new york city is passed as-is to the analyzer.
```
GET /_search
{
  "query": {
    "query_string": {
      "query": "(new york city) OR (big apple)",
      "default_field": "content"
    }
  }
}
```
**simple_query_string**   uses a simple syntax to parse and split the provided query string into terms based on special operators.the simple_query_string query does not return errors for invalid syntax. Instead, it ignores any invalid parts of the query string.

#### match_all
most simple query, which matches all documents, giving them all  

#### Term-level queries
**exists**  Returns documents that contain an indexed value for a field.
The field in the source JSON is null or []  
The field has "index" : false set in the mapping  
The length of the field value exceeded an ignore_above setting in the mapping  
The field value was malformed and ignore_malformed was defined in the mapping   

**fuzzy** Returns documents that contain terms similar to the search term  
Changing a character (box → fox)  
Removing a character (black → lack)  
Inserting a character (sic → sick)  
Transposing two adjacent characters (act → cat)  

**ids** 通过文档id直接查询  
**prefix** returns documents that contain a specific prefix in a provided field. speed up prefix queries using the index_prefixes mapping parameter    
**range** 常用数值范围查询  
**regexp** 
**term** documents that contain an **exact** term in a provided field  **Avoid using the term query for text fields** To better search text fields, the match query also analyzes your provided search term before performing a search. This means the match query can search text fields for analyzed tokens rather than an exact term.

**terms**  多个值
**terms_set** contain a minimum number of exact terms in a provided field.给的多个值中至少符合N个  
**wildcard** Returns documents that contain terms matching a wildcard pattern

# ILM: Manage the index lifecycle
Index lifecycle policies can trigger actions such as:  
**Rollover**: Creates a new write index when the current one reaches a certain size, number of docs, or age.  
**Shrink**: Reduces the number of primary shards in an index.  
**Force merge**: Triggers a force merge to reduce the number of segments in an index’s shards.  
**Freeze**: Freezes an index and makes it read-only.  
**Delete**: Permanently remove an index, including all of its data and metadata.  

ILM defines five index lifecycle phases:  
**Hot**: The index is actively being updated and queried.  
Set Priority->Unfollow->Rollover->Read-Only->Shrink->Force Merge->Searchable Snapshot  
**Warm**: The index is no longer being updated but is still being queried.
Set Priority->Unfollow->Read-Only->Allocate->Migrate->Shrink->Force Merge  
**Cold**: The index is no longer being updated and is queried infrequently. The information still needs to be searchable, but it’s okay if those queries are slower.  
Set Priority->Unfollow->Read-Only->Searchable Snapshot->Allocate->Migrate->Freeze  
**Frozen**: The index is no longer being updated and is queried rarely. The information still needs to be searchable, but it’s okay if those queries are extremely slow.  
Searchable Snapshot
**Delete**: The index is no longer needed and can safely be removed.  
Wait For Snapshot->Delete  
#### Index lifecycle actions
**Allocate** Phases allowed: warm, cold. change which nodes are allowed to host the index shards and change the number of replicas.  

| 参数    | 描述  | 默认值    |
|------|-----|-----|
| **number_of_replicas**       |(Optional, integer) Number of replicas to assign to the index. |      |
| **total_shards_per_node**   |(Optional, integer) The maximum number of shards for the index on a single Elasticsearch node. A value of -1 is interpreted as unlimited. See total shards. types|     |  
| **include**   |(Optional, object) Assigns an index to nodes that have at least one of the specified custom attributes.|     |  
| **exclude**   |(Optional, object) Assigns an index to nodes that have none of the specified custom attributes.|     |  
| **require**   |(Optional, object) Assigns an index to nodes that have all of the specified custom attributes.|     |  

**Delete** Phases allowed: delete. delete_searchable_snapshot (Optional, Boolean) Deletes the searchable snapshot created in a previous phase. Defaults to true. This option is applicable when the searchable snapshot action is used in any previous phase.
**Force merge**  Phases allowed: hot, warm. merge the index into the specified maximum number of segments. This action makes the index read-only. To use the forcemerge action in the hot phase, the rollover action must be present. If no rollover action is configured, ILM will reject the policy.  
**Freeze** Phases allowed: cold. 
Freezing an index closes the index and reopens it within the same API call. This means that for a short time no primaries are allocated.   
**Migrate** Phases allowed: warm, cold. enabled default to true  
**Read only** Phases allowed: hot, warm, cold. To use the readonly action in the hot phase, the rollover action must be present. If no rollover action is configured, ILM will reject the policy.  
**Rollover** Phases allowed: hot. Rolls over a target to a new index when the existing index meets one or more of the rollover conditions.   
The index name must match the pattern ^.*-\d+$, for example (my-index-000001).  
The index.lifecycle.rollover_alias must be configured as the alias to roll over.  
The index must be the write index for the alias.  

| 参数    | 描述  | 默认值    |
|------|-----|-----|
| **max_age**       |(Optional, time units) Triggers rollover after the maximum elapsed time from index creation is reached. The elapsed time is always calculated since the index creation time, even if the index origination date is configured to a custom date |      |
| **max_docs**   |(Optional, integer) Triggers rollover after the specified maximum number of documents is reached. Documents added since the last refresh are not included in the document count. The document count does not include documents in replica shards.|     |  
| **max_size**   |(Optional, byte units) Triggers rollover when the index reaches a certain size. This is the total size of all primary shards in the index. Replicas are not counted toward the maximum index size..|     |  
| **max_primary_shard_size**   |(Optional, byte units) Triggers rollover when the largest primary shard in the index reaches a certain size. This is the maximum size of the primary shards in the index. As with max_size, replicas are ignored.|     |   

**Searchable snapshot** Phases allowed: hot, cold, frozen.  Takes a snapshot of the managed index in the configured repository and mounts it as a searchable snapshot. If the index is part of a data stream, the mounted index replaces the original index in the stream. 
Don’t include the searchable_snapshot action in both the hot and cold phases. This can result in indices failing to automatically migrate to the cold tier during the cold phase. If the searchable_snapshot action is used in the hot phase the subsequent phases cannot include the shrink, forcemerge, or freeze actions.  
简单的说就是把能通过ES搜索外部存储，比如S3. 
| 参数    | 描述  | 默认值    |
|------|-----|-----|
| **snapshot_repository**       |(Required, string) Repository used to store the snapshot.|      |
| **force_merge_index**   |(Optional, Boolean) Force merges the managed index to one segment. Defaults to true. If the managed index was already force merged using the force merge action in a previous action the searchable snapshot action force merge step will be a no-op..|  true   |  
**Set priority** Phases allowed: hot, warm, cold. Sets the priority of the index as soon as the policy enters the hot, warm, or cold phase. Higher priority indices are recovered before indices with lower priorities following a node restart.**priority** (Required, integer) The priority for the index. Must be 0 or greater. Set to null to remove the priority.  
**Shrink** Phases allowed: hot, warm. Sets a source index to read-only and shrinks it into a new index with fewer primary shards. The name of the resulting index is shrink-<random-uuid>-<original-index-name>.  After the shrink action, any aliases that pointed to the source index point to the new shrunken index.  To use the shrink action in the hot phase, the rollover action must be present. If no rollover action is configured, ILM will reject the policy.  
| 参数    | 描述  | 默认值    |
|------|-----|-----|
| **number_of_shards**       |(Optional, integer) Number of shards to shrink to. Must be a factor of the number of shards in the source index. This parameter conflicts with max_primary_shard_size, only one of them may be set .|      |
| **max_primary_shard_size**   |(Optional, Boolean) Force merges the managed index to one segment. Defaults to true. If the managed index was already force merged using the force merge action in a previous action the searchable snapshot action force merge step will be a no-op..|  true   |  

**Unfollow** Phases allowed: hot, warm, cold, frozen. Converts a CCR follower index into a regular index. This enables the shrink, rollover, and searchable snapshot actions to be performed safely on follower indices. You can also use unfollow directly when moving follower indices through the lifecycle. Has no effect on indices that are not followers, phase execution just moves to the next action.  
当一个索引在 CCR 模式下作为跟随者索引时，它会持续从另一个集群中的领导者索引获取并同步数据。如果你希望这个索引停止跟随（即停止同步数据），并且希望将其转换为一个独立的、可写的索引，你需要执行 unfollow 操作。  
具体步骤
停止索引跟随： 执行 unfollow 操作之前，你必须先暂停复制任务（使用 pause_follow API）。这确保了在你转换索引状态时不会有数据变化。  
执行 unfollow 操作： 使用 unfollow API 将索引从跟随者索引状态转换为普通索引。  

**Wait for snapshot** Phases allowed: delete.
Waits for the specified SLM policy to be executed before removing the index. This ensures that a snapshot of the deleted index is available. policy (Required, string) Name of the SLM policy that the delete action should wait for.

#### Tune for indexing speed
Use bulk requests.  
Use multiple workers/threads to send data to Elasticsearch.  
Unset or increase the refresh interval.  index.refresh_interval  
Disable replicas for initial loads. index.number_of_replicas. Once the initial load is finished, you can set index.number_of_replicas back to its original value.

#### Tune for search speed
Give memory to the filesystem cache 
Avoid page cache thrashing by using modest readahead values on Linux  
Readahead 是操作系统为了提高磁盘 I/O 性能的一种优化技术。它会在实际需要读取数据之前，从磁盘读取一大块连续的数据并将其缓存到内存中。这样，当程序需要读取数据时，系统可以直接从内存中提供数据，而无需等待磁盘的 I/O 操作完成。  
当 Elasticsearch 执行搜索操作时，会引发大量的随机读 I/O 操作。这个过程涉及到从磁盘读取数据块。当 readahead 值设置得过高时，系统可能会在实际需要的数据之外，预读更多的数据块到内存中，尤其是在使用 memory mapping（内存映射）时。这种行为可能会导致大量不必要的 I/O 操作，从而浪费了内存和 I/O 带宽。  

特别是在使用软件 RAID、LVM（逻辑卷管理）或 dm-crypt（加密设备）等技术时，基础的块设备可能会拥有非常大的 readahead 值（几个 MiB）。这通常会导致页面缓存（filesystem cache）被频繁替换，严重影响 Elasticsearch 的搜索和更新性能。  
lsblk -o NAME,RA,MOUNTPOINT,TYPE,SIZE  
临时调整 sudo blockdev --setra 128 /dev/sda 永久 ACTION=="add|change", KERNEL=="sda", ATTR{bdi/read_ahead_kb}="128"   

Use faster hardware  
Document modeling  合理设计文档结构，理解查询模式和数据访问模式
Search as few fields as possible 多field搜索慢，可以用copy-to把多个合并到一个field  
Pre-index data  
Consider mapping identifiers as keyword  
Avoid scripts  
Search rounded dates  
Force-merge read-only indices   Indices that are read-only may benefit from being merged down to a single segment.  
Warm up global ordinals Global ordinals are a data structure that is used to optimize the performance of aggregations
Warm up the filesystem cache  index.store.preload 
Use index sorting to speed up conjunctions  
Use preference to optimize cache utilization There are multiple caches that can help with search performance, such as the filesystem cache, the request cache or the query cache.  
Replicas might help with throughput, but not always max(max_failures, ceil(num_nodes / num_primaries) - 1) num_nodes 个节点、num_primaries 个主分片，最多同时应对 max_failures 个节点故障  
Faster phrase queries with index_phrases.  For example, in the phrase "quick brown fox," 2-shingles (bigrams) would be "quick brown" and "brown fox." These are essentially pairs of consecutive words.   index_phrases 设置为true，就能直接从bigrams中获得结果，避免先查询是否有brown和fox，在判断brown是否在fox前
Phrase Queries:  

A** phrase query** in Elasticsearch is a search query where you look for documents containing a specific sequence of words. For example, searching for the phrase "quick brown fox" means you want to find documents where these words appear together in this exact order.  
**Slop** in the context of phrase queries allows for some flexibility in the order and proximity of the words in the phrase. A slop of 0 means the words must appear in the exact order, with no other words in between. A slop of 1 allows for one word to be out of order or for one extra word to appear between them, and so on.   
**Shingles** are a sequence of adjacent tokens (words) in a document. For example, in the phrase "quick brown fox," 2-shingles (bigrams) would be "quick brown" and "brown fox." These are essentially pairs of consecutive words.    


Faster prefix queries with index_prefixes. 通过min_chars和max_chars去分词

Use constant_keyword to speed up filtering specified property value to a fix word in index mapping. then split index to 2 or more.  
