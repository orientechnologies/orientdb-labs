## OrientDB SQL v3

OEP discussion: https://github.com/orientechnologies/orientdb-labs/issues/4

References:

https://github.com/orientechnologies/orientdb-docs/blob/newexecutor/SQL-Syntax.md
https://github.com/orientechnologies/orientdb-docs/blob/newexecutor/SQL-Projections.md
https://github.com/orientechnologies/orientdb-docs/blob/newexecutor/SQL-Update.md



**Summary:**

Rationalize current SQL syntax to allow easy, complete and consistent manipulation of multi-model data structures

**Goals:**

- Remove ambiguities of current SQL syntax
- Enrich the grammar with new operators, to support all the possible data manipulations on a Multi-Model domain (Eg. array split and concatenation, bitwise operations)
- Make SQL syntax easier to understand, use and to document
- Use the OEP as a starting point for the formal grammar documentation

**Non-Goals:**

- change the parsing technology (current JavaCC parser will remain)
- discuss the query planning and execution (it will be managed in another OEP)

**Motivation:**

OrientDB SQL query syntax/engine is probably the first component that was developed in OrientDB.

In the years, the syntax evolved by addition of new keywords and features, sometimes in a way that is not completely consistent.

The result is that current OrientDB SQL is sometimes ambiguous or incomplete (see https://github.com/orientechnologies/orientdb/issues/5950). 


**Description:**

Redefine the SQL syntax and semantics, validate its consistency.
Some work is already in progress on a parallel docs branch

https://github.com/orientechnologies/orientdb-docs/blob/newexecutor/SQL-Syntax.md https://github.com/orientechnologies/orientdb-docs/blob/newexecutor/SQL-Projections.md https://github.com/orientechnologies/orientdb-docs/blob/newexecutor/SQL-Update.md


**Alternatives:**

- leave it as is and only implement the query execution part

**Risks and assumptions:**

- SQL v.3 won't be 100% backward compatible with the previous versions

**Impact matrix**

- [ ] Storage engine
- [x] SQL
- [ ] Protocols
- [ ] Indexes
- [ ] Console
- [ ] Java API
- [ ] Geospatial
- [ ] Lucene
- [ ] Security
- [ ] Hooks
- [ ] EE

