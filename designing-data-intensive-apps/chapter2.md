## **Data Models and Query Languages**
Most applications are built by layering one data model on top of another. Each layer hides the complexity of the layers below it by providing a clean model. These abstractions allow different grouops of people to work together effectively

### Relational Model vs Document Model
The roots of relational databases lie in *business data processing, transaction processing, and batch processing*. The goal was to hide implementation detail behind a cleaner interface.

Driving forces behind adoption of NoSQL or *Not Only SQL* databases:
- greater scalability involving very large datasets or very high throughput
- open source software >> commercial database products
- specialized query operations
- desire for dynamic and expressive data model

#### Object-Relational Mismatch
- if data is stored in relational tables, translation layer required between application code and database model -- _impedance mismatch_
- JSON model reduces impedance mismatch due to lack of schema
- JSON representation has better locality than multi-table schema

#### Many-to-Many & Many-to-One Relationships
- In relational databases, it's normal to refer to rows in other tables by ID because joins are easy -- not so much in in document databases.
- If database doesn't support joins, can emulate it in application by making multiple database queries
- IBM's Information Management System (IMS) used a hierarchical model (similar to JSON model for document databases). Relational model and network model were proposed to solve IMS' problem with many-to-many relationships as joins weren't supported

#### Network model
- generalization of hierarchical model
  - for tree structure, every record has exactly one parent. In network model, a record could have multiple parents
- links between records were like pointers. The only way to access record was to follow _access path_ from root 
- manual access path selection made use of limited hardware capabilities in 1980s, but querying and updating the database was complicated and inflexible
 - changing access paths was difficult and required changing DB query code to handle new access paths. Also difficult to change application's data model

#### Relational model
- A relation (table) is a collection of tuples (rows)
- query optimiser decides which parts of query to execute in which order and which indexes to use. 
- general purpose optimizer made it easier to add features to application

### Relational vs Document Databases Today
- document data model: schema flexibility, better performance due to locality, sometimes closer to data structures used by application
- relational model: better support for joins, many-to-one and many-to-many relationships
- if application data has document-like structure, use document model. _Shredding_ (splitting document-like structure into multiple tables) can cause complicated application code
- if application data has many-to-many relationships, and you are using the document model, can emulate joins by making multiple requests or denormalizing the data. This moves complexity into application code leading to worse performance.

#### Schema-on-write vs. Schema-on-read
- Document databases and JSON structures don't enforce schemas - arbitrary keys and values can be added to a document and clients have no guarantee for what fields are present
- Document databases are sometimes called _schemaless_, but more accurately _schema-on-read_ in contrast to schema-on-write. 
- Schema-on-read is similar to dynamic type checking and schema-on-write is similar to static (compile-time) type checking
- schema-on-read approach advantageous if data is heterogeneous (many different object types or data structure determined by external systems)

#### Data locality for queries
If application needs access to entire document, there is performance advantage to storage locality.
- database loads the entire document even if you access part of it
- on updates, entire document needs to be rewritten. In-place updates only possible if encoding size is not changed
- need to keep document sizes small and avoid writes that increase document size

Relational and document databases are converging over time. 
- PostgreSQL supports JSON documents, RethinkDB supports joins, MongoDB drivers auto resolve document references (client-side join) 

***

## Query Languages for Data
- SQL is a declarative language - describe pattern of data you want, but not _how_ to acheive that goal.
- declarative language hides implementation details, making it possible to introduce performance improvements without changes to queries
- declarative languages are faster in parallel execution because they only specify pattern of results, not the algorithm to determine the results. Imperative code is harder to parallelize because it specifies instructions that must be performed in a particular order

In a web browser, using declarative CSS styling is much better than manipulating styles imperatively in JavaScript. 

### MapReduce Querying
- MapReduce is a programming model for processing large amounts of data popularized by Google.
```js
db.observations.mapReduce(
    function map() { 2
        var year  = this.observationTimestamp.getFullYear();
        var month = this.observationTimestamp.getMonth() + 1;
        emit(year + "-" + month, this.numAnimals); 3
    },
    function reduce(key, values) { 4
        return Array.sum(values); 5
    },
    {
        query: { family: "Sharks" }, 1
        out: "monthlySharkReport" 6
    }
);
```
- map and reduce must be pure functions. This allows db to run functions in any order
- usability problem with MapReduce is that you have to write 2 coordinated functions. MongoDB added support for declarative query language called _aggregation pipeline_
```js
db.observations.aggregate([
    { $match: { family: "Sharks" } },
    { $group: {
        _id: {
            year:  { $year:  "$observationTimestamp" },
            month: { $month: "$observationTimestamp" }
        },
        totalAnimals: { $sum: "$numAnimals" }
    } }
]);
```

***

## Graph-Like Data Models
- Model data as graph if have many many-to-many relationships
- A graph has _vertices_ and _edges_
- There are different ways of structuring and querying data - the _property graph_ model (implemented by Neo4j, Titan, InfiniteGraph) and _triple-store_ model (Datomic, AllgroGraph).

### Property Graphs
- each vertex consists of:
 - unique identifier
 - set of outgoing + incoming edges
 - collection of properties (key-value pairs)
- each edge consists of:
 - unique identifier
 - tail (start) and end (tail) vertices
 - label to describe relationship between two vertices
 - collection of properties (key-value pairs)
Graphs are flexible for data modelling and are good for evolvability.

### Cyper Query Language
- _Cypher_ is a declarative query language for property graphs created for Neo4j
- Graph queries in SQL: In relational DB, you know in advice which/ how many joins you need. You need to traverse a variable number of edges in a graph.
 - E.g. in Cypher, :WITHIN*0.. means "follow a WITHIN edge, zero or more times" like the regex * operator
 - variable-length traversal paths in query can be expressed using _recursive common table expressions_ (WITH RECURSIVE syntax)

### Triple-Stores and SPARQL
- In triple-store model, all info is stored in three-part statements: (_subject_, _predicate_, _object_)
- if object is vertex, predicate is edge (e.g. lucy bornIn usa). If object is string, predicate is property (e.g. lucy age 33)

_SPARQL_ is a query language for triple-stores using RDF data model.

### The Foundation: Datalog
_Datalog_ provides the foundation that later query languages build on. The data model is similar to triple-store. Instead of (_subject_, _predicate_, _object_), it is _predicate_(_subject_, _object_)
- we define rules that tell DB about new predicates. Rules can refer to other rules
- rules can be combined and reused in different queries
