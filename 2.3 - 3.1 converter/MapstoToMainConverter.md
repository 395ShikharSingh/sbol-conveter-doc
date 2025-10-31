MapstoToMainConverter

In SBOL2, a MapsTo object defines how two parts in different modules relate — e.g., how one module’s input connects to another module’s component.
In SBOL3, there is no direct MapsTo entity anymore.
Instead, this relationship is represented using:
    ComponentReference → to point across components
    Constraint → to specify how they’re related (replace, merge, verify identical, etc.)
So, this converter’s job is to:
    Take SBOL2’s “MapsTo” relationships
    and build the equivalent ComponentReference + Constraint pair in SBOL3.


🧩 2. Structure overview
The class has two static methods, because MapsTo relationships can exist in two different contexts in SBOL2:
    | Context                       | SBOL2 Entity                         | Conversion Method                   |
    | ----------------------------- | ------------------------------------ | ----------------------------------- |
    | Inside a ComponentDefinition  | ComponentDefinition → Component      | convertFromComponentDefinition()    |
    | Inside a ModuleDefinition     | ModuleDefinition → Module → Component| convertFromModuleDefinition()       |


⚙️ 3. Let’s break down convertFromComponentDefinition()
```java
public static void convertFromComponentDefinition(SBOLDocument document, ComponentDefinition sbol2ComponentDefinition, Parameters parameters)
			throws SBOLGraphException, SBOLValidationException 
```
document → The SBOL3 document being built.
sbol2ComponentDefinition → The SBOL2 ComponentDefinition we’re converting.
parameters → Metadata and URI mapping helper.


Step 1️⃣ — Iterate through SBOL2 Components
```java
for(org.sbolstandard.core2.Component sbol2Component: sbol2ComponentDefinition.getComponents()) 
```
Each ComponentDefinition can contain multiple components (subparts).
You loop through them because MapsTo relationships live inside these components.

Step 2️⃣ — Iterate through their MapsTo relations
```java
for(MapsTo mapsTo: sbol2Component.getMapsTos()) {
```
Now you’re at the heart: every MapsTo object describes a mapping between a local and a remote entity.

Step 3️⃣ — Get the SBOL3 parent component
```java
URI sbol3ParentComponentURI = Util.createSBOL3Uri(sbol2ComponentDefinition);
Component sbol3ParentComponent = document.getIdentified(sbol3ParentComponentURI, Component.class);
```
This line fetches the SBOL3 version of the parent (the converted ComponentDefinition).
Remember, earlier in conversion, this ComponentDefinition was turned into an SBOL3 Component, so we now fetch it from the document.

Step 4️⃣ — Find the SBOL3 SubComponent corresponding to the SBOL2 Component
```java
SubComponent sbol3SubComponentForModule = Util.getSBOL3Entity(sbol3ParentComponent.getSubComponents(), sbol2Component, parameters);
```
Util.getSBOL3Entity(...) searches through the SBOL3 parent’s subcomponents and finds the SBOL3 equivalent of the SBOL2 component.
Basically: “What’s the SBOL3 version of this specific SBOL2 subcomponent?”

Step 5️⃣ — Create ComponentReference
```java
ComponentReference sbol3CompRef = MapstoToComponentReferenceConverter.convertForComponent(
    document,
    sbol2Component,
    sbol3ParentComponent,
    mapsTo,
    sbol3SubComponentForModule,
    parameters
);
```
Now the converter:
    builds a ComponentReference object in SBOL3,
    links it to the remote subcomponent described by the mapsTo.