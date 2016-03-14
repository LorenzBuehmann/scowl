# Scowl

[![Build Status](https://secure.travis-ci.org/phenoscape/scowl.png)](http://travis-ci.org/phenoscape/scowl)

Scowl is a Scala library allowing a declarative approach to composing OWL expressions and axioms using the [OWL API](http://owlapi.sourceforge.net).

## Usage

Add the dependency to your `build.sbt`:

```scala
resolvers += "Phenoscape Maven repository" at "http://phenoscape.svn.sourceforge.net/svnroot/phenoscape/trunk/maven/repository"

libraryDependencies += "org.phenoscape" %% "scowl" % "1.0.1"
```

Import `org.phenoscape.scowl._`, and Scowl implicit conversions will add pseudo Manchester syntax methods to native OWL API objects. Additionally, functional syntax-style constructors and extractors will be in scope.

Scowl 1.0.1 is built with OWL API 3.5. Scowl 1.1 will support OWL API 4.

## Examples
Implementations of all the examples from the [OWL 2 Web Ontology Language 
Primer](https://www.w3.org/TR/owl2-primer/) are provided in [Manchester syntax](https://github.com/phenoscape/scowl/blob/master/src/main/scala/org/phenoscape/scowl/example/OWL2PrimerManchester.scala) and [Functional syntax](https://github.com/phenoscape/scowl/blob/master/src/main/scala/org/phenoscape/scowl/example/OWL2PrimerFunctional.scala). The examples below are also available in 
[code](https://github.com/phenoscape/scowl/blob/master/src/main/scala/org/phenoscape/scowl/example/ReadMeExamples.scala).

### Scowl expressions use and return native OWL API objects
```scala
import org.phenoscape.scowl._
// import org.phenoscape.scowl._

val hasParent = ObjectProperty("http://www.co-ode.org/roberts/family-tree.owl#hasParent")
// hasParent: org.semanticweb.owlapi.model.OWLObjectProperty = <http://www.co-ode.org/roberts/family-tree.owl#hasParent>

val isParentOf = ObjectProperty("http://www.co-ode.org/roberts/family-tree.owl#isParentOf")
// isParentOf: org.semanticweb.owlapi.model.OWLObjectProperty = <http://www.co-ode.org/roberts/family-tree.owl#isParentOf>

val isSiblingOf = ObjectProperty("http://www.co-ode.org/roberts/family-tree.owl#isSiblingOf")
// isSiblingOf: org.semanticweb.owlapi.model.OWLObjectProperty = <http://www.co-ode.org/roberts/family-tree.owl#isSiblingOf>

val Person = Class("http://www.co-ode.org/roberts/family-tree.owl#Person")
// Person: org.semanticweb.owlapi.model.OWLClass = <http://www.co-ode.org/roberts/family-tree.owl#Person>

val FirstCousin = Class("http://www.co-ode.org/roberts/family-tree.owl#FirstCousin")
// FirstCousin: org.semanticweb.owlapi.model.OWLClass = <http://www.co-ode.org/roberts/family-tree.owl#FirstCousin>

val axiom = FirstCousin EquivalentTo (Person and (hasParent some (Person and (isSiblingOf some (Person and (isParentOf some Person))))))
// axiom: org.semanticweb.owlapi.model.OWLEquivalentClassesAxiom = EquivalentClasses(<http://www.co-ode.org/roberts/family-tree.owl#FirstCousin> ObjectIntersectionOf(<http://www.co-ode.org/roberts/family-tree.owl#Person> ObjectSomeValuesFrom(<http://www.co-ode.org/roberts/family-tree.owl#hasParent> ObjectIntersectionOf(<http://www.co-ode.org/roberts/family-tree.owl#Person> ObjectSomeValuesFrom(<http://www.co-ode.org/roberts/family-tree.owl#isSiblingOf> ObjectIntersectionOf(<http://www.co-ode.org/roberts/family-tree.owl#Person> ObjectSomeValuesFrom(<http://www.co-ode.org/roberts/family-tree.owl#isParentOf> <http://www.co-ode.org/roberts/family-tree.owl#Person>)))))) )
```
### Add some axioms and programmatically generated GCIs to an ontology
```scala
val manager = OWLManager.createOWLOntologyManager()
val ontology = manager.createOntology()
val PartOf = ObjectProperty("http://example.org/part_of")
val HasPart = ObjectProperty("http://example.org/has_part")
val DevelopsFrom = ObjectProperty("http://example.org/develops_from")
val Eye = Class("http://example.org/eye")
val Head = Class("http://example.org/head")
val Tail = Class("http://example.org/tail")

manager.addAxiom(ontology, Eye SubClassOf (PartOf some Head))
manager.addAxiom(ontology, Eye SubClassOf (not(PartOf some Tail)))

val gcis = for {
  term <- ontology.getClassesInSignature(true)
} yield {
  (not(HasPart some term)) SubClassOf (not(HasPart some (DevelopsFrom some term)))
}
manager.addAxioms(ontology, gcis)
```

### Using pattern matching extractors to implement negation normal form
```scala
def nnf(given: OWLClassExpression): OWLClassExpression = given match {
    case Class(_) => given
    case ObjectComplementOf(Class(_)) => given
    case ObjectComplementOf(ObjectComplementOf(expression)) => nnf(expression)
    case ObjectUnionOf(operands) => ObjectUnionOf(operands.map(nnf))
    case ObjectIntersectionOf(operands) => ObjectIntersectionOf(operands.map(nnf))
    case ObjectComplementOf(ObjectUnionOf(operands)) => ObjectIntersectionOf(operands.map(c => nnf(ObjectComplementOf(c))))
    case ObjectComplementOf(ObjectIntersectionOf(operands)) => ObjectUnionOf(operands.map(c => nnf(ObjectComplementOf(c))))
    case ObjectAllValuesFrom(property, filler) => ObjectAllValuesFrom(property, nnf(filler))
    case ObjectSomeValuesFrom(property, filler) => ObjectSomeValuesFrom(property, nnf(filler))
    case ObjectMinCardinality(num, property, filler) => ObjectMinCardinality(num, property, nnf(filler))
    case ObjectMaxCardinality(num, property, filler) => ObjectMaxCardinality(num, property, nnf(filler))
    case ObjectExactCardinality(num, property, filler) => ObjectExactCardinality(num, property, nnf(filler))
    case ObjectComplementOf(ObjectAllValuesFrom(property, filler)) => ObjectSomeValuesFrom(property, nnf(ObjectComplementOf(filler)))
    case ObjectComplementOf(ObjectSomeValuesFrom(property, filler)) => ObjectAllValuesFrom(property, nnf(ObjectComplementOf(filler)))
    case ObjectComplementOf(ObjectMinCardinality(num, property, filler)) => ObjectMaxCardinality(math.max(num - 1, 0), property, filler)
    case ObjectComplementOf(ObjectMaxCardinality(num, property, filler)) => ObjectMinCardinality(num + 1, property, filler)
    case ObjectComplementOf(ObjectExactCardinality(num, property, filler)) => ObjectUnionOf(ObjectMinCardinality(num + 1, property, filler), ObjectMaxCardinality(math.max(num - 1, 0), property, filler))
  }
```

### Using pattern matching extractors in for comprehensions
```scala
// Print all properties and fillers used in existential restrictions in subclass axioms
for {
  SubClassOf(_, subclass, ObjectSomeValuesFrom(property, filler)) <- ontology.getAxioms
} yield {
  println(s"$property $filler")
}

// Make an index of language tags to label values
val langValuePairs = for {
  AnnotationAssertion(_, RDFSLabel, _, value @@ lang) <- ontology.getAxioms
} yield {
  lang -> value
}
val langToValues: Map[String, Set[String]] = langValuePairs.foldLeft(Map.empty[String, Set[String]]) {
  case (langIndex, (lang, value)) =>
    langIndex.updated(lang, langIndex.getOrElse(value, Set.empty) ++ Set(value))
}
```

## License

Scowl is open source under the [MIT License](http://opensource.org/licenses/MIT).  See [LICENSE](LICENSE) for more information.
