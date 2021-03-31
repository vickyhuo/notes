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
- made it easier to add features to application



***

## Query Languages for Data
- 
## Graph-like Data Models

