CollectionConverter
🧠 High-level idea
The goal of this class is to take an SBOL2 Collection (a group of biological elements)
and convert it into an SBOL3 Collection object.

```java
 public Collection convert(SBOLDocument doc, org.sbolstandard.core2.Collection input, Parameters parameters) throws SBOLGraphException {  
        Collection col = doc.createCollection(Util.createSBOL3Uri(input),Util.getNamespace(input));
        Util.copyIdentified(input, col, parameters);
        for (URI memberUri : input.getMemberURIs()) {
        	col.addMember(Util.createSBOL3Uri(memberUri, parameters) );
        }
        return col;
    }
```


1️⃣ The main conversion method
```java
@Override
public Collection convert(SBOLDocument doc, org.sbolstandard.core2.Collection input, Parameters parameters)
        throws SBOLGraphException {
```
This is the main function that performs the conversion.
It takes:
    the new SBOL3 document you’re filling,
    the old SBOL2 collection you’re converting,
    and some conversion parameters (like how URIs should be generated).


2️⃣ Create the new SBOL3 collection
```java
Collection col = doc.createCollection(
    Util.createSBOL3Uri(input),
    Util.getNamespace(input)
);
```
This line creates a new Collection in the SBOL3 document.
Let’s unpack the helpers:
    Util.createSBOL3Uri(input) → generates a new URI for the converted collection, ensuring it follows SBOL3 URI conventions.
    Util.getNamespace(input) → determines the correct namespace to put the new entity in.


3️⃣ Copy common fields
```java
Util.copyIdentified(input, col, parameters);
```
This line copies all metadata that both SBOL2 and SBOL3 share:
    display ID
    name
    description
    annotations
    version
    etc.


4️⃣ Add members
```java
for (URI memberUri : input.getMemberURIs()) {
    col.addMember(Util.createSBOL3Uri(memberUri, parameters));
}
```
This is the core of what a Collection does — it links to other entities.
So here’s what happens:
    Take each member URI from the old SBOL2 Collection.
    Convert that old URI into a new SBOL3 URI (using the Util helper).
    Add that new URI as a member in the SBOL3 Collection.

