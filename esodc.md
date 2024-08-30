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

Use constant_keyword to speed up filtering