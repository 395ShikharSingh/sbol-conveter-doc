# ComponentConverter

instead of having "Components inside Components", you now have Features inside Components, with a special subtype called SubComponent.


1️⃣ Class Declaration

```java
public class ComponentConverter implements ChildEntityConverter<org.sbolstandard.core2.Component, Feature>
```

This means it converts a child entity from SBOL 2 → SBOL 3.
Input: org.sbolstandard.core2.Component
Output: a Feature (in SBOL 3)



2️⃣ The Main convert() Method

```java
convert(SBOLDocument doc, Identified parent, Identified inputParent, Component input, Parameters parameters)
```

doc: SBOL 3 document being built
parent: the SBOL 3 parent entity
inputParent: the SBOL 2 parent entity
input: the SBOL 2 Component being converted
parameters: helper class for mapping old→new IDs and URIs


3️⃣ Create a new SBOL 3 SubComponent

```java
URI sbol3InstanceOfUri = Util.createSBOL3Uri(input.getDefinitionURI(), parameters);
SubComponent resultSC = sbol3ParentComponent.createSubComponent(...);
```


🔹 Think of this as:
“In SBOL 3, Components don’t directly contain Components anymore. Instead, they have SubComponents — smaller building blocks linked by a definition URI.”
So the converter builds that link and keeps track of it.
parameters.addMapping(...) keeps a record of how old IDs map to new URIs — useful later to resolve relationships.


4️⃣ Copy roles

```java
resultSC.setRoles(Util.toList(input.getRoles()));
```

Each SBOL 2 component can play certain biological “roles” (like a promoter, terminator, etc.), so these are simply transferred.


5️⃣ Handle Role Integration

SBOL 2 and SBOL 3 have slightly different enum systems for how roles merge.
This code translates between them:

```java
if (ri.equals(RoleIntegrationType.MERGEROLES)) → RoleIntegration.mergeRoles
if (ri.equals(RoleIntegrationType.OVERRIDEROLES)) → RoleIntegration.overrideRoles
```


6️⃣ Measurements

```java
if(input.getMeasures()!=null) {
   MeasurementConverter measurementConverter = new MeasurementConverter();
   ...
}
```

If a component has physical measurements (like length, time, concentration), it uses a dedicated MeasurementConverter to handle those.



7️⃣ Copy Generic Attributes

```java
Util.copyIdentified(input, result, parameters);
```
This copies identifiers, display IDs, names, descriptions — all the general metadata.


8️⃣ Handle AccessType (Public/Private)

```java
if (input.getAccess()==AccessType.PUBLIC) {
   Interface sbol3Interface = sbol3ParentComponent.getInterface();
   ...
   sbol3Interface.setNonDirectionals(features);
}
```

If a Component is marked public, it becomes part of the parent’s interface, meaning other parts can interact with it.



9️⃣ Handle Source Locations

```java
if (input.getSourceLocations()!=null){
   SourceLocationConverter locationConverter = new SourceLocationConverter();			
   for (Location loc : input.getSourceLocations()) {
      locationConverter.convert(...);
   }
}
```
This converts positional info — e.g., if part A starts at base 50 and ends at 100, this preserves that mapping.





SBOL 2 document loads → gives a Component.
ComponentConverter is called → creates a SubComponent in SBOL 3 doc.
Metadata and properties are mapped.
Extra converters (MeasurementConverter, SourceLocationConverter) handle nested info.
The mapping between SBOL2 IDs and SBOL3 URIs is stored in parameters.
Final result is added to the SBOL3 parent component.



SBOL2.Component
    ↓
ComponentConverter
    ↓
SBOL3.SubComponent (Feature)
       ├── roles
       ├── measurements
       ├── roleIntegration
       ├── access/interface
       └── locations
