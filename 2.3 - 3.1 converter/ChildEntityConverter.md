ChildEntityConverter

Think of the whole SBOL document as a family tree:
    Parent Entity → like a parent node (e.g., a ComponentDefinition).
    Child Entity → things inside it (like a subcomponent, sequence annotation, feature, etc.).
Now, SBOL2 and SBOL3 represent this tree structure differently.
So the ChildEntityConverter interface is like a common job description for translators who deal with nested elements.

```java
public interface ChildEntityConverter<InputEntity, OutputEntity> {
	 OutputEntity convert(SBOLDocument document, Identified parent, org.sbolstandard.core2.Identified inputParent, InputEntity input,Parameters parameters) throws SBOLGraphException, SBOLValidationException;
}
```

This makes it a generic interface, meaning it can be reused for any kind of entity type.
    InputEntity: the SBOL2 entity (old version)
    OutputEntity: the SBOL3 entity (new version)

1️⃣ SBOLDocument document
This is the SBOL3 document you’re adding entities into.
It’s like the “destination container” — all converted objects go here.


2️⃣ Identified parent
This is the SBOL3 parent — the already converted object in SBOL3.


3️⃣ org.sbolstandard.core2.Identified inputParent
This is the SBOL2 parent, from the old model.


4️⃣ InputEntity input
This is the child object being converted, like:
    a SequenceConstraint
    a SequenceAnnotation
    a Component depending on which subclass implements this interface.


5️⃣ Parameters parameters
This holds conversion-specific options, such as:
    namespace handling
    ID mapping rules
    URI conversion preferences


6️⃣ Throws
These exceptions protect against:
    SBOLGraphException → if something breaks SBOL3’s graph structure (e.g., invalid link or circular reference)
    SBOLValidationException → if the converted data violates SBOL rules (like missing required properties)