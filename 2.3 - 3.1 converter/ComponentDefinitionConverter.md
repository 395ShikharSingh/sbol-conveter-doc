ComponentDefinitionConverter

Think of SBOL2 as the old language and SBOL3 as the new version.
This converter acts like a translator who:
    Reads an SBOL2 ComponentDefinition (like a DNA part definition, protein, etc.)
    Creates the equivalent SBOL3 Component object.
    Makes sure every small relationship (like sequences, subcomponents, features, constraints) is transferred correctly.

```java
public class ComponentDefinitionConverter implements EntityConverter<ComponentDefinition, Component>
```
It implements the EntityConverter interface — meaning it must provide:
```java
public Component convert(SBOLDocument doc, ComponentDefinition input, Parameters parameters)
```
This function does the real conversion.

1️⃣ Duplicate check
```java
if (doc.getComponents()!=null) {
    for (Component c : doc.getComponents()) {
        if (c.getUri().toString().equals(Util.createSBOL3Uri(input).toString())) {
            return null;
        }
    }
}
```
This ensures we don’t recreate the same Component if it already exists in the SBOL3 document.


2️⃣ Create a new Component in SBOL3
```java
Component comp = doc.createComponent(
    Util.createSBOL3Uri(input),
    Util.getNamespace(input),
    Util.toSBOL3ComponentDefinitionTypes(Util.toList(input.getTypes()))
);
```
    Util.createSBOL3Uri(input) → gives it a new URI in SBOL3 namespace.
    Util.getNamespace(input) → ensures it stays in the same logical group.
    Util.toSBOL3ComponentDefinitionTypes(...) → converts the SBOL2 type (DNA, RNA, protein, etc.) into the SBOL3 equivalent.
👉 This line creates the shell of the SBOL3 Component.


3️⃣ Copy metadata (name, description, annotations, etc.)
```java 
Util.copyIdentified(input, comp, parameters);
```
This is a utility method that copies “identifiable” info — like identity URI, displayId, name, description, etc.


4️⃣ Convert Roles
```java
comp.setRoles(Util.convertSORoles2_to_3(input.getRoles()));
```
SBOL2 and SBOL3 define biological roles differently (e.g., “promoter”, “CDS”).
This step translates SBOL2 role URIs to SBOL3-compliant URIs.


5️⃣ Convert Sequences
```java
if (input.getSequenceURIs()!= null && input.getSequenceURIs().size() > 0) {
    List<org.sbolstandard.core3.entity.Sequence> sequences= new ArrayList<>();
    for (URI uri:input.getSequenceURIs()){
        URI sbol3URI=Util.createSBOL3Uri(uri, parameters);
        org.sbolstandard.core3.entity.Sequence sbol3Seq=doc.getIdentified(sbol3URI, org.sbolstandard.core3.entity.Sequence.class);
        sequences.add(sbol3Seq);        		
    }
    comp.setSequences(sequences);        	 
}
```
SBOL2 ComponentDefinitions link to Sequences by URI.
This part:
    Converts each SBOL2 sequence URI into SBOL3 form.
    Finds the corresponding SBOL3 Sequence object in the document.
    Links it to the new Component.


6️⃣ Convert Subcomponents
```java
ComponentConverter converter = new ComponentConverter();        
for (org.sbolstandard.core2.Component c : input.getComponents()) {
    converter.convert(doc, comp, input, c, parameters);
}
```
Each SBOL2 ComponentDefinition can have “subcomponents.”
This creates SBOL3 SubComponents under the new Component.


7️⃣ Convert SequenceAnnotations
```java
for (SequenceAnnotation sa : input.getSequenceAnnotations()) {
    if (sa.isSetComponent()) {
        SequenceAnnotationToSubComponentConverter satoscConverter = new SequenceAnnotationToSubComponentConverter();
        satoscConverter.convert(doc, comp, input, sa, parameters);
    } 
    else {
        SequenceAnnotationToFeatureConverter satosfConverter = new SequenceAnnotationToFeatureConverter();
        satosfConverter.convert(doc, comp, input, sa, parameters);
    }
}
```
SBOL2 SequenceAnnotations can represent:
    SubComponents (if they refer to another component)
    Features (if they mark specific locations on a sequence)
So this logic says:
    If annotation refers to a component → use SequenceAnnotationToSubComponentConverter.
    Else → use SequenceAnnotationToFeatureConverter.


8️⃣ Convert SequenceConstraints
```java
SequenceConstraintConverter seqconstConverter = new SequenceConstraintConverter();
for (SequenceConstraint sc : input.getSequenceConstraints()) {
    seqconstConverter.convert(doc, comp, input, sc, parameters);
}
```
SBOL2 constraints describe how subcomponents are ordered or related (e.g., promoter before CDS).
This part converts those relationships into SBOL3’s constraint model.


🔚 Return the Converted Component
```java
return comp
```