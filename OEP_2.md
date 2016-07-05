##Unified Multi-Model (Document/Graph) API and class hierarchy 


Initial proposal (please do not comment here anymore):  https://github.com/orientechnologies/orientdb/issues/6052

OEP comments: https://github.com/orientechnologies/orientdb-labs/issues/2

**Summary:**

Review the class hierarchy of both Document and Graph APIs to create a unified hierarchy and a unified Multi-Model API.

**Goals:**
- Finally have a single, Multi-Model API
- Better interoperability between Document and Graph API
- Have a single way to use all the OrientDB functionalities with a single API
- Make it easier to choose the API (no choice at all)
- Make it easier to write docs for the API (no more duplicated docs for document and graph)

**Non-Goals:**

- deprecate or remove any of the existing data models (Document vs. Graph)


**Motivation:**

In OrientDB 2.2, Document and Graph APIs are distinct and very different (see doc.field() and vertex.getProperty()). 
This creates the following problems:
- New users have a hard time choosing the API to start with
- Query results sometimes return a mix of the two models, so it's hard to write consistent code
- We have to write an maintain duplicate docs, one of each API
- We lack the core requirement of a Multi-Model database: a unified Multi-Model API


**Description:**

Now we have two separate class hierarchies, one for ODocument and one for OrientElement, where the only super-interface is OIdentifiable, but it obviously lacks all the methods for the domain manipulation.

The proposal is following:
- Create a common interface T for both ODocument and OrientElement, with methods to 
  - get/set properties
  - access and manipulate metadata (OClass)
- Have two sub-interfaces of this interface, one for vertices (T1) and one for edges (T2), with specific methods for graph traversal (getEdges(), getVertices(), or maybe out()/in()?)
- Create a new implementation of O*Vertex and O*Edge in the Core module and review their implementation to extend what currently is ODocument. Let OrientVertex and OrientEdge extend this new implementation.
- Unify the implementation of all the hierarchy, to use the same methods for property and schema manipulation
- Refactor ODatabase interface to
  - let all the methods return T or Generics<T>
  - implement methods to instantiate and retrieve both T and T1/T2 explicitly
- See if it's possible to refactor OrientGraph to implement both ODatabase and TinkerPop Graph.

**Alternatives:**

- only create the interface hierarchy but avoid to have graph classes extend ODocument class

**Risks and assumptions:**

- collision with method signatures of other standards (eg. next TinkerPop versions)

**Impact matrix**

- [ ] Storage engine
- [ ] SQL
- [ ] Protocols
- [ ] Indexes
- [ ] Console
- [x] Java API
- [ ] Geospatial
- [ ] Lucene
- [ ] Security
- [x] Hooks
- [ ] EE


## Implementation proposal

### New interfaces and classes

```java
interface OElement {
  public void setProperty(String name, Object val);
  public Object getProperty(String name);

  public Optional<ODocument> asDocument();
  public Optional<OVertex> asVertex();
  public Optional<OEdge> asEdge();

  public boolean isDocument();
  public boolean isVertex();
  public boolean isEdge();

  public Optional<OClass> getSchemaType();
}
```

```java
interface OVertex extends OElement {
  public Collection<OEdge> getEdges(Direction direction);
  public Collection<OEdge> getEdges(Direction direction, String... type);
  public Collection<OEdge> getEdges(Direction direction, OClass... type);
  
  public Collection<OVertex> getVertices(Direction direction);
  public Collection<OVertex> getVertices(Direction direction, String... type);
  public Collection<OVertex> getVertices(Direction direction, OClass... type);
  
  public OEdge addEdge(OVertex to);
  public OEdge addEdge(OVertex to, String type);
  public OEdge addEdge(OVertex to, OClass type);
}
```

```java
interface OEdge extends OElement {
  public OVertex getFrom();
  public OVertex getTo();
}
```

```java
class ODocument extends OElement {
   ...
}
```

```java
class OVertexImpl extends OVertex {
   ...
}
```

```java
class OEdgeImpl extends OEdge {
   ...
}
```

Open questions:
- Should OElement extend OIdentifiable? What about result sets (see OEP_5)?
- In case we decide to have OElement that does not extend OIdentifiable, should we still have save() and delete() methods? Do we need another interface in the middle?
- Add asVertex() and asEdge() methods to OElement?
