# Storage and Retrieval
## Data Structures That Power Your Database
- Many databases internally use a log(append-only data file). Real databases need to deal with concurrency control, reclaiming disk space to limit log size, handling errors and partially written records.
- Need _index_ to efficiently find value for particular key. An index is an additional structure derived from primary data.
- Well-chosen indexes speed up read queries but every index slows down writes. Databases require you to choose indexes manually using your knowledge and typical query patterns
 
### Hash Indexes
- Key-value stores are similar to dictionary type (hash map or hash table)
- Simplest index is to keep an in-memory hash map where every key is mapped to byte offset in log data file.
- Bitcask (default storage engine in Riak) offers high-performance reads and writes by requiring that all keys fit in available RAM. Values can be loaded from disk, thus using more space than available memory. Suited for when value for each key is updated frequently.
- To avoid running out of disk space, break log into segments and close segment file when capacity reached. Perform _compaction_ by throwing away duplicate keys in log and keeping only most recent update for each key.
 - segments never modified after writem so merged segment is written to new file. Merging and compaction of segments done in background thread. Use old segment files for read requests, and latest segment file for write requests. After finished merging, delete old segment files
- each segment has own in-memory hash mapping keys to offsets in file. To find value for key, check most recent segment's hash table, then second most recent segment and so on.

Issues that are important in real implementation:
- file format: simpler to use binary format
- deleting records: set tombstone on deleted record to tell merging process to discard any previous values for deleted key
- crash recovery: in-memory hash maps lost if DB restarted. Bitcask speeds up recovery by storing snapshot of each segment's hash map on disk enabling faster load to memory
- partially written records: DB may crash at any time. Bitcask includes checksums to allow corrupted parts of log to be detected and ignored
- concurrency control: writes appended in strictly sequential order - have only one writer thread, can support multiple read threads

Append-only design good for several reasons:
- sequential write operations faster than random writes, especially on magnetic spinning-disks, and solid state drives to some extent
- concurrency and crash recovery much simpler if segment files are immutable or append-only
- merging old segments avoids file fragmentation

Hash table limitations:
- hash table must fit in memory. Hard to make on-disk hash map performant
- range queries not efficient

### SSTables and LSM-Trees
_Sorted string table_ (SSTable) is where segment files are sorted by key
Advantages over log segments with hash indexes:
- merging segments is simple and efficient - can use algorithms like mergesort
- in order to find particular key in file, don't need to keep index of all keys in memory. e.g. Keep index for a and d. In order to find c, just need to start at a and keep scanning until you reach c
- can compress group of records before writing to disk since read requests need to scan over group

#### Constructing and maintaining SSTables
With red-black trees or AVL trees, can insert keys in any order and read them back in sorted order
- When write comes in, add to in-memory balanced tree data structure (aka _memtable_)
- When memtable exceeds threshold (megabytes), write to disk as SSTable file. Writes can continue to new memtable instance
- For read requests, try to find key in memtable, then most recent on-disk segment, then next-older segment etc.
- From time to time, run merging and compaction  process in background to discard overwritten or deleted values

If database crashes, most recent writes not written to disk are lost. To precent this, can keep separate log on disk where every write is appended. Everytime memtable written to SSTable, can discard log

#### Making an LSM-tree out of SSTables
- LSM-trees are a set of SSTables merged in background
- Storage engines that merge and compact sorted files are called LSM (Log-Structured Merge-Tree) storage engines
- Lucene, an indexing engine for full-text search used by Elasticsearch and Solr uses similar method to store its _term dictionary_

#### Performance optimizations
- LSM-tree algo must check memtable and all segments before determining key doesn't exist. _Bloom filters_ (memory-efficient data structure for approximating contents of set) often used by storage engines for optimization
- size-tiered compaction: newer and smaller SSTables successively merged into older and larger SSTables
- leveled compaction: key range split into smaller SSTables and older data moved into separate levels, allowing incremental compaction + uses less disk space

### B-Trees
- most widely used indexing structure
- keep key-value pairs sorted by key
- log-structured indexes break DB down into variable-size megabyte segments. B-trees break DB down into fixed-size _blocks_ or _pages_, traditionally 4KB
- one page is designated as the _root_ of the B-tree. The page contains keys and references to child pages

- if you want to update value of existing key in B-tree, search for leaf page containing that key, change the value and write page back to disk
- if want to add new key, find page whose range includes new key and add to page. If there isn't enough space in the page for new key, split into 2 half-full pages, and parent page updated to account for new key ranges
- tree is _balanced_ - B-tree with n keys always has depth of O(log n)

#### Making B-trees reliable
- The write operation of B-tree overwrites page on disk with new data without changing the page location. This is different from log-structured indexes like LSM-trees which only append to files and doesn't modify files in place
- splitting page because insertion caused it to overfill requires writes to the two pages that were split, and overwrites to parent page to update references to the two child pages. If database crashes after only some pages have been written, can have corrupted index.
- _Write-ahead log_ (WAL, also known as _redo log_) is an additional data structure on disk to help restore DB to consistent state after crash
- concurrency control required to ensure threads see tree in a consistent state, done through protecting tree's data structures with latches (lightweight locks)

#### B-tree optimizations
- Use copy-on-write scheme to write modified page to different location and update all parent pages to point to new location - useful for concurrency control
- Save space by abbreviating key and only providing enough information to act as boundaries between key range. More keys in page = higher branch factor = fewer tree levels
- Many B-tree implementations try to lay out the tree so that leaf pages appear in sequential order on disk, but this is difficult to maintain as tree grows. LSM-trees rewrite large segments when merging, so easier for them to keep sequential keys close to each other
- Additional pointers. Leaf page may contain references to sibling pages on the left/right so keys can be sequentially scanned without going back to parent page
- B-tree variants like fractal trees borrow log-structured ideas to reduce disk seeks

### Comparing B-Trees and LSM-Trees
- LSM-trees typically faster for writes, B-trees faster for reads. Reads are slower on LSM-trees because they have to check several different data structures and SSTables at different stages of compaction
- benchmarks inconclusive and sensitive to details of workload

#### Advantages of LSM-trees
- LSM-trees may sustain higher write throughput than B-trees because they have lower write amplification (one write to DB resulting in multiple writes to disk over DB lifetime), and because they sequentially write compact SSTable files rather than having to overwrite several pages in tree. In magnetic hard drives, sequential writes are much faster than random writes
- LSM-trees have better compression and thus produces smaller files on disk - B-trees have unused disk space due to fragmentation

#### Downsides of LSM-trees
- compaction process can sometimes interfere with performance of ongoing reads and writes. B-trees are more predictable. The bigger the database, the more disk bandwith required for compaction. 
- for B-trees, each key exists in exactly one place in the index. This offers strong transactional semantics. Transaction isolation implemented using locks on ranges of keys, and in B-tree index, locks can be directly attached to tree.

### Other Indexing Structures
- Key-value indexes is like the _primary key_ index in the relational model. There are also secondary indexes
- Secondary indexes can be constructed from key-value index. The main difference is that secondary index indexed values are not necessarily unique. Can be implemented in two ways: make each value in index a list of matching row identifiers or by making each entry unique by appending row identifier to it

_Clustered index_ is when row is stored within index. A compromise between clustered and nonclustered index is a _covering index_ where some of the table's columns is stored within the index. Clustered and covering indexes can speed up reads but need additional storage and can add overhead on writes. DB has more work to enforce transactional guarantees in order to remove inconsistencies from duplication

Most common type of multi-dimensional index is _concatenated index_, where key is made by appending one column to another

#### Full-text search and fuzzy indexes
- Indexes don't allow you to search for _similar_ keys. Such _fuzzy_ querying requires different techniques.
- Full-text search engines allow grammatical variations, synonyms and occurrences of words near each other

#### Keeping everything in memory
- Disk data needs to be laid out carefully for good performance on reads and writes, but disks are durable and have lower cost per gigabyte
- Datasets can be kept entirely in memory across several machines, leading to _in-memory databases_.
- Some in-memory key-value stores like Memcached are intended for cachine use, and data loss is acceptable if machine is restarted
- In-memory databases aimed for durability can be achieved through special hardware (e.g. battery-powered RAM), writing change log or periodic snapshots to disk, or replicating in-memory state to other machines
- When in-memory DB is restarted, it needs to reload its state from disk or over the network from a replica. The disk is used as an append-only log for durability.
- In-memory DB avoid overheads of encoding in-memory data structures in a form that can be written to disk
- In-memory DB provides data models difficult to implement with disk-based indexes, like priority queues and sets
- _Anti-cachine_ approach - evict least recently used data from memory to disk, similar to virtual memory and swap files in OS but with more granularity
- Further changes to storage engine design needed when non-volatile memory (NVM) technologies become more popular

*** 

## Transaction Processing or Analytics?
- A _transaction_ is a group of reads and writes that form a logical unit. This access pattern became known as online transaction processing (OLTP)
- Data analytics have different access patterns - query needs to scan over huge number of records, only reading a few columns per record and calculating aggregate statistics. This pattern is called online analytic processing (OLAP)

### Data Warehousing
- A separate database that analysts can query without affecting the performance of concurrently executing transactions
- Contains read-only data that is extracted from OLTP databases, transformed into analysis-friendly schema, cleaned up and loaded into data warehouse (_Extract-Transform-Load_ or _ETL_ process)
- Data warehouse can be optimized for analytic access patterns

- data model of data warehouse is most commonly relational. Most database vendors focus on supporting either transaction processing or analytic workloads.

### Stars and Snowflakes: Schemas for Analytics
- Data warehouses used in formulaic style known as _star schema_ or _dimensional modeling_
- Facts are captured as individual events as it allows maximum flexibility of analysis. The table can become extremely large
- Dimensions represent the _who_, _what_, _where_, _when_, _how_ and _why_ of the event
- "Star schema" -> when table relationships are visualized, fact table is in the middle, surrounded by its dimension tables, like the rays of a star
- _Snowflake schema_: dimensions further broken down into subdimensions. They are more normalized than star schemas, but star schemas are easier to work with

***

## Column-Oriented Storage
- In row-oriented storage engines (e.g. most OLTP databases), all the values from one row of a table are stored next to each other. The engine will load all these rows into memory, parse them, filter based on conditions.
- In column-oriented storage, each column is stored in separate file. The query only needs to read and parse columns used in query

### Column Compression
- Column storage can be compressed as values are often repetitive. Different techniques like bitmap encoding can be used.
- Column storage are also good for efficiently using CPU cycles. By fitting in more rows through column compression, and using vectorized processing (bitwise OR and AND operations directly on compressed data)

### Sort Order in Column Storage
- Data needs to be sorted an entire row at a time.
- Sorting data helps with column compression. Compression effect is strongest on first sort key. Columns further down sorting priority will not compress well.
- Having multiple sort orders is similar to multiple secondary indexes

### Writing to Column-Oriented Storage
- Update-in-place approach, like B-trees is not possible. If inserting a row in the middle of a sorted table, need to rewrite all the column files
- Writes can be executed using LSM-trees

### Aggregation: Data Cubes and Materialized Views
- Materialized aggregates are cached aggregates that queries use most often
- Cache can be created through _materialized view_ - a table-like object that contains the results of some query
- When underlying data changes, view needs to be updated. The database can do it automatically, but writes would be more expensive

