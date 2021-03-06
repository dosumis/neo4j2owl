# OWL 2 EL <-> Neo4J Mapping "Direct existentials"

This is a preliminary draft of our Neo4J to OWL 2 mapping. The goal is to be able to import a well defined subset of OWL 2 EL ontologies into and export them from Neo4J in such a way that entailments and annotations are preserved (not however the syntactic structure) in the ontology after the round-trip. The main differences of this mapping to other mappings (see References below) are
* its treatment of blank nodes in existential restrictions. Rather than creating blank nodes, we create direct edges between entities, labelled with the property of the existential restriction. This makes querying the graph more intuitive.
* its use of qualified safe labels for typing relations (which makes for easier querying)
* taking advantage of OWL implications (rather than relying on pure RDF syntax). Rather than merely mapping an asserted axiom into Neo4J, we are interested in the following **`implied`** relationships:
  * **Class-Class**. For two class names A and B, an annotation property P, and an object property name R, we consider three types of relationships
    * SubClassOf restrictions of the form A SubClassOf: B
    * Existential restrictions of the form A SubClassOf: R some B
    * Annotation assertions of the form A Annotations: P B
  * **Individual-Individual**. For two individuals i and j, an annotation property P and an object property R, we consider
    * Object property assertions of the form i Facts: R j
    * Annotation assertions of the form i Annotations: P j
  * **Class-Individual**. For a class A, an individual i, an annotation property P and an object property name R, we consider three types of relationships
    * Class assertions of the form i Types: A
    * Existential restrictions of the form i Types: R some B
    * Annotation assertions of the form C Annotations: P i
  * **Individual-Class**. For a class A, an individual i, an annotation property P and an object property name R, we consider
    * Existential restrictions of the form A SubClassOf: R value i
    * Annotation assertions of the form i Annotations: P A    

The most similar mapping to our is the one used by [Monarch Initiatives SciGraph](https://github.com/SciGraph/SciGraph/wiki/MappingToOWL). The main differences are:
* In SciGraph, IRIs are first class citizens everywhere, while we prioritise safe labels to make query construction easier. This is especially important for edge types: Instead of MATCH p=()-[r:`http://purl.obolibrary.org/obo/BFO_0000050`]->() RETURN p LIMIT 25, we prefer to say MATCH p=()-[r:part_of_obo]->() RETURN p LIMIT 25
* Anonymous class patterns are kept alongside so called "convenience" edges in SciGraph, the latter of which correspond to the way we treat edges in general.


Some **ideosyncracies** of our approach are:
* To be able to roundtrip, we create disconnected nodes in the Neo4J graph representing properties so that we can represent metadata (such as labels or other annotations) pertaining to them.
* We introduce a number of properties based on the notion of label for easier yet unambiguous querying, which are materialised on all nodes. **qualified safe labels** in particular are used to type relationships. Given an entity e (Example: http://purl.obolibrary.org/obo/BFO_0000050),
  * **ns** corresponds to the namespace the relationship in question in question (Example: http://purl.obolibrary.org/obo/).
  * **short_form** corresponds to the remainder (or fragment) of the IRI of e. (Example: "BFO_0000050")
  * **label** corresponds to either
    * the first rdfs:label annotation encountered or, if there are no rdfs:label annotation,
    * the short_form, or, if there is no short_form,
    * the whole IRI (Example: "part of").
  * **safe label** (sl) is the **label**, with all non-alphanumeric characters being replaced by **_**. Trailing and heading underscores are removed, any sequence of underscores is replaced by a single underscore (Example: "part_of").
  * **curie** is the valid curie for an entity, i.e. the namespace and the short_form, separated by ":" (Example: "obo:BFO_0000050").
  * **qualified safe label** (qsl) is the safe label of an entity and its namespace, separated by **_** (Example: "part_of_obo").
* An example use of an SL in Cypher is (:n)-[part_of_obo]-(:x)
* Individuals are currently only typed with their most direct type
* We only support datatypes that are supported by both neo4j and OWL (other OWL2 datatypes will be cast to Neo4j:String, which means their typing information is lost in a round-trip):

 Neo4J | OWL 2 | Comment |
|-----|-----|-----|
| Integer | xsd:integer |  |
| String | xsd:string |  |
| Boolean | xsd:boolean | |
| Float | xsd:float |  |

* For properties and axioms, all annotations are treated as if they were to literals (i.e. they wont be connected to other entities)

For readibility, we omit the neo4j2owl namespaces in the OWL 2 EL Axiom Column;

PREFIX: n2o: <http://neo4j2owl.org/mapping#>
Prefix: : <http://neo4j2owl.org/mapping#>

## Entities

All entities in the ontology, i.e. classes, individuals, object properties, data properties, and annotation properties are represented as nodes in the graph. Relationship nodes are only added to hold metadata about relations, and are disconnected from the rest of the graph. The iri, sl and short_form attributes on Neo4J nodes are the only three attributes that are not mapped into corresponding rdf statements.

| Concept | OWL 2 EL Axiom | Neo4J Graph Pattern | Comment |
|---------------|---------------|----------------|----------------|
| Class declaration | Class: A | (:Class {iri: 'http://neo4j2owl.org/mapping#A', short_form:'A', label:'L(A)'}) |  |
| Individual declaration | Individual: i | (:Individual {iri: 'http://neo4j2owl.org/mapping#i', label:'L(A)'}) |  |
| Annotation property declaration | AnnotationProperty: R | (:AnnotationProperty {iri: 'http://neo4j2owl.org/mapping#R', short_form:'R', sl:'SL(R)'}, label:'L(A)') |  |
| Object property declaration | ObjectProperty: R | (:ObjectProperty {iri: 'http://neo4j2owl.org/mapping#R', short_form:'R', sl:'SL(R)'}, label:'L(A)') |  |
| Data property declaration | DataProperty: R | (:DataProperty {iri: 'http://neo4j2owl.org/mapping#R', short_form:'R', sl:'SL(R)'}, label:'L(A)') |  |


## Class-Class relationships

| Concept | OWL 2 EL Axiom | Neo4J Graph Pattern | Comment |
|---------------|---------------|----------------|----------------|
| SubClassOf | Class: A SubClassOf: B | (:Class {.. short_form:'A'..})-[r:SubClassOf]-(:Class {.. short_form:'B'..}) |  |
| Annotations on classes to other classes | Class: A Annotations: R B | (:Class {.. short_form:'A'..})-[r:`SL(R)`]-(:Class {.. short_form:'B'..}) |  |
| Simple existential "class" restrictions on classes | Class: A SubClassOf: R some B | (:Class {.. short_form:'A'..})-[r:`SL(R)`]-(:Class {.. short_form:'B'..}) |  |

## Class-Individual relationships

| Concept | OWL 2 EL Axiom | Neo4J Graph Pattern | Comment |
|---------------|---------------|----------------|----------------|
| Annotations on classes to individuals | Class: A Annotations: R i | (:Class {.. short_form:'A'..})-[r:`SL(R)`]-(:Individual {.. short_form:'i'..}) |  |
| Simple existential "individual" restrictions on classes | Class: A SubClassOf: R value i | (:Class {.. short_form:'A'..})-[r:`SL(R)`]-(:Individual {.. short_form:'i'..}) |  |

## Individual-Individual relationships

| Concept | OWL 2 EL Axiom | Neo4J Graph Pattern | Comment |
|---------------|---------------|----------------|----------------|
| Object Property Assertion | Individual: i Facts: R j | (:Individual {.. short_form:'i'..})-[r:`SL(R)`]-(:Individual {.. short_form:'j'..}) |  |
| Annotations on individuals to other individuals | Individual: i Annotations: R j | (:Individual {.. short_form:'i'..})-[r:`SL(R)`]-(:Individual {.. short_form:'j'..}) |  |

## Individual-Class relationships

| Concept | OWL 2 EL Axiom | Neo4J Graph Pattern | Comment |
|---------------|---------------|----------------|----------------|
| Class Assertion | Individual: i Types: A | (:Individual {.. short_form:'i'..})-[r:Types]-(:Class {.. short_form:'A'..}) |  |
| Simple existential restriction on assertion | Individual: i Types: R some A | (:Individual {.. short_form:'i'..})-[r:`SL(R)`]-(:Class {.. short_form:'A'..}) |  |
| Annotations on individuals to classes | Individual: i Annotations: R A | (:Individual {.. short_form:'i'..})-[r:`SL(R)`]-(:Class {.. short_form:'A'..}) |  |

## Entity-literal relationships
For reasons of feasibility, data property assertions or restrictions, will be incomplete in almost any implementation. In our reference implementation, we only consider asserted data property assertions.

| Concept | OWL 2 EL Axiom | Neo4J Graph Pattern | Comment |
|---------------|---------------|----------------|----------------|
| Annotations on classes to literals | Class: A Annotations: P "A"@en | (:Class {..,SF(R):'"A"@en'}) |  |
| Annotations on individuals to literals | Individual: i Annotations: P "A"@en | (:Individual {..,SF(R):'"A"@en'}) |  |
| Annotations on object properties to literals | ObjectProperty: R Annotations: P "A"@en | (:ObjectProperty {..,SF(R):'"A"@en'}) |  |
| Annotations on data properties to literals | DataProperty: R Annotations: P "A"@en | (:DataProperty {..,SF(R):'"A"@en'}) |  |
| Annotations on annotation properties to literals | AnnotationProperty: R Annotations: P "A"@en | (:AnnotationProperty {..,SF(R):'"A"@en'}) |  |
| Data property assertion | Individual: A Facts: R 2 | (:Individual {..,SF(R):2}) |  |
| Data property restriction | Class: A SubClassOf: R value 2 | (:Class {..,SF(R):2}) | |

## Axiom annotations
| Concept | OWL 2 EL Axiom | Neo4J Graph Pattern | Comment |
|---------------|---------------|----------------|----------------|
| Axiom Annotations | Class: A SubclassOf: Annotations: P "A"@en | (:Class {..})-[r: {..SF(R):'"A"@en'}..]-() |  |

## Notes on mapping procedure
* If we use curies to indicated edge-type for OPs, we need labels as an attribute.

## Custom Neo4J properties
For edge types, node labels (in the Neo/Cypher sense of the term) and property keys, special characters and spaces can potentially be supported via the use of back-tick escaping, but avoiding them makes writing cypher much easier - especially if via script.

* iri, http://purl.obolibrary.org/obo/BFO_0000050
* short_form, BFO_0000050
* label, "part of"
* prefix, "http://purl.obolibrary.org/obo/BFO_"
* curie, BFO:0000050
* safe_label, part_of (replace non-alphanumeric by "_")
* qualified_safe_label (BFO:part_of) [[Maybe use alterntive to :, to avoid necessity of escape marke in Cypher]]

## Related user stories (internal use only)
* As a  developer writing OWL from the KB,  I want to be able to easily find the correct iri for all OWL entities from the database.
   => All nodes have an IRI.  All edges have an IRI or have a key  (e.g. short_form or Curie) that makes it easy to look up and IRI from the relevant node.  The key may be a edge type name or an edge attribute (neo4J property key).
* As a developer writing OWL from the KB, I want to be able to tell from all nodes and edges what type of OWL entity or axiom I should create.
  => There must be an unambiguous mapping from node:label (or some standard attribute) and edge types (or some standard attribute) to OWL entity and axiom types.
* As a developer writing to the database,  I need to know what identifiers it is safe to use to uniquely identify nodes for the purpose of merging in new content.
External content is loaded from ontologies that we don't have complete control over. Uniqueness of  rdfs:Label can not be relied upon (although it is typically a safe assumptions within the context of a single ontology).   short_form uniqueness is much safer, but Curies would be safer still.   
* As a developer writing to the database, I want to be able to easily and unambiguouysly refer  to existing entities.  =>
Using  full IRIs for this can be a big pain in the ass as, outside of OBO, these are hard to remember. Using curie's is somewhat easier, but can still be a pain.  Using short_forms is easiest, but to do this safely requires a commitment to short_form uniqueness.  While this cannot be absolutely guaranteed it is rare.
=> All nodes and edge-types have unique short_form or curie
* As a developer editing the DB I want to be able to easily write queries to check what information it contains.
I should be able to easily select major categories of content.  
I should be able to write queries using lexical identifiers (labels or something closely related to them), even if this is occasionally unreliable.
I should be able to use lexical identifiers (a readable neo4j:label or node attribute) to filter/select on major categories of entity (e.g. anatomical entity; expression pattern; genetic feature).  
* As a Geppetto developer, I should be able to find the display names to use for edges and property keys without having to mung text or run secondary lookups.
Edges should store rdfs:label; property keys should not be Qualified ?


## Future work
* Provenance should be added on edges only
* Add superproperties as additional edge types?
* Axiom annotation with an Individual as a value can not be directly supported as edges to edges, but we can (and should) model these by using an identifier for the Individual (iri or short_form).  The join required to pull axioms associated with the referenced individual can be easily performed in Cypher.
* Constraint: Edge types should correspond to some allowable set of object property and annotation property types. All of these are defined in source ontologies. This makes querying efficient and easy, and allows for queries specify disjunctive sets of edge type.

## References

| Reference | Explanation |
| ---------------|----------------|
| [And Now for Something Completely Different: Using OWL with Neo4j](https://neo4j.com/blog/using-owl-with-neo4j/) | 2013 Blogpost on how OWL could be loaded into Neo. It provides a motivation for the conversion, and some code snippets to get started. |
| [SciGraph OWL2Neo](https://github.com/SciGraph/SciGraph/wiki/Neo4jMapping) | Preliminary mapping |
| [SciGraph Neo2OWL](https://github.com/SciGraph/SciGraph/wiki/MappingToOWL) | Preliminary mapping |
| [Convert OWL to labeled property graph and import into Neo4J](https://github.com/flekschas/owl2neo4j) | Covers only [class hierarchy/and annotations](https://github.com/flekschas/owl2neo4j/wiki/How-is-OWL-translated-into-a-labeled-property-graph%3F).   |
| [Neo4J-Jena Wrapper](https://github.com/semr/neo4jena) | Provides the mapping of RDF to property graphs (Neo4J) using Jena API. Literals are represented as nodes. |
| [Sail Ouplementation](https://github.com/tinkerpop/blueprints/wiki/Sail-Ouplementation) | Interface to [access property graphs directly] as a triple store. No details on mappings in documentation. |
| [Using Neo4J to load and query OWL ontologies](http://sujitpal.blogspot.co.uk/2009/05/using-neo4j-to-load-and-query-owl.html) | 2009 blogpost with ad-hoc implementation, showing how to load the wine ontology with Jena and transform it into Neo4J. The mapping is intuitive and RDF-like, translating triples directly to nodes and edges. |
| [Importing RDF data into Neo4J](Using Neo4J to load and query OWL ontologies) | 2016 blogpost defining a mapping proposition. Annotations are added on nodes if they are literatels, else nodes are created (good). Blank nodes are created. [Neo4J stored procedure](https://github.com/jbarrasa/neosemantics) exists. |
| [Building a semantic graph in Neo4j](https://jesusbarrasa.wordpress.com/2016/04/06/building-a-semantic-graph-in-neo4j/) | 2016 blogpost on how to define an RDFS style ontology in Neo4J. The author has a keen interest in RDF/OWL2Neo4J mappings, his entire blog seems to be mainly about that topic. |
| [Neo4j is your RDF store (series)](https://jesusbarrasa.wordpress.com/2016/11/17/neo4j-is-your-rdf-store-part-1/) | Interesting series describing how to use Neo as a triple store. |
| [Storing and querying RDF in Neo4j](http://www.snee.com/bobdc.blog/2014/01/storing-and-querying-rdf-in-ne.html) | Description of a SPARQL plugin to query Neo4J based on Sail Ouplementation. |
| [Importing ttl (Turtle) ontologies in Neo4j](http://michaelbloggs.blogspot.co.uk/2013/05/importing-ttl-turtle-ontologies-in-neo4j.html) | 2013 blogpost with code on how to import ttl into neo using Sesame API |
| [OLS: OWL to Neo4j schema](https://www.slideshare.net/thesimonjupp/ontologies-neo4jgraphworkshopberlin) | Largely undocumented, but see slide 16 following of Berlin workshop. Implementing our mapping partially. |
