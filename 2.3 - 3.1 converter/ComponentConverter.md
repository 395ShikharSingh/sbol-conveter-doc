# ComponentConverter

## Overview

`ComponentConverter` is a converter class that transforms SBOL2 `Component` objects into SBOL3 `Feature` objects (specifically `SubComponent` instances). This converter is part of the SBOL 2.3 to SBOL 3.1 conversion pipeline.

**Purpose**: Convert SBOL2 Components (which represent instances of ComponentDefinitions within a parent Component) into SBOL3 SubComponents (which represent instances of Components within a parent Component).

## Util.java functions used (read this first)

This converter relies on these shared helpers:

### 1. `Util.createSBOL3Uri(URI inputUri, Parameters parameters)`

- **Purpose**: Convert an SBOL2 URI to SBOL3 format.
- **Process**:
  1) Resolve persistent identity to the latest version via RDF model; 2) extract URI prefix, version, displayId; 3) rebuild as `namespace/version/displayId`; 4) handle URN or malformed cases.
- **Example**:
  - SBOL2: `http://example.org/CD1/1.0` → SBOL3: `http://example.org/1.0/CD1`

### 2. `Util.toList(Set<T> set)`

- **Purpose**: Convert a Set to a List.
- **Why**: SBOL3 APIs commonly expect Lists where SBOL2 uses Sets (e.g., roles).

### 3. `Util.copyIdentified(org.sbolstandard.core2.Identified input, Identified output, Parameters parameters)`

- **Purpose**: Copy common Identified fields and annotations from SBOL2 to SBOL3.
- **Copies**: `name`, `description`, `wasDerivedFrom` (Set→List), attachments (TopLevel), and converts SBOL2 `Annotation` → SBOL3 `Metadata` (including nested).

## Class Structure

### Interface Implementation

```java
public class ComponentConverter implements ChildEntityConverter<org.sbolstandard.core2.Component, Feature>
```

The converter implements the `ChildEntityConverter` interface, which defines the contract for converting child entities from SBOL2 to SBOL3.

### Interface Definition

```java
interface ChildEntityConverter<InputEntity, OutputEntity> {
    OutputEntity convert(
        SBOLDocument document,
        Identified parent,
        org.sbolstandard.core2.Identified inputParent,
        InputEntity input,
        Parameters parameters
    ) throws SBOLGraphException, SBOLValidationException;
}
```

## Main Conversion Method

### `convert()` Method

**Signature:**
```java
public Feature convert(
    SBOLDocument doc,
    Identified parent,
    org.sbolstandard.core2.Identified inputParent,
    org.sbolstandard.core2.Component input,
    Parameters parameters
) throws SBOLGraphException, SBOLValidationException
```

**Parameters:**
- `doc`: The target SBOL3 document where the converted entity will be added
- `parent`: The SBOL3 parent entity (Component) that will contain the converted SubComponent
- `inputParent`: The SBOL2 parent entity (ComponentDefinition) of the input Component
- `input`: The SBOL2 Component to be converted
- `parameters`: Conversion parameters containing mappings, RDF model, and other shared state

**Returns:**
- `Feature`: The converted SBOL3 SubComponent (which implements Feature interface)

**Throws:**
- `SBOLGraphException`: If there's an error in the SBOL graph structure
- `SBOLValidationException`: If the conversion results in invalid SBOL data

## Conversion Process

### Step-by-Step Breakdown

#### 1. **Parent Type Casting**
```java
Component sbol3ParentComponent = (Component) parent;
```
Cast the parent to an SBOL3 Component type.

#### 2. **URI Conversion (Definition URI)**
```java
URI sbol3InstanceOfUri = Util.createSBOL3Uri(input.getDefinitionURI(), parameters);
```

**What this does:**
- Converts the SBOL2 Component's `definitionURI` (which points to a ComponentDefinition) to an SBOL3 URI format
- The `definitionURI` specifies what ComponentDefinition this Component is an instance of

**Util Function Used:**
- `Util.createSBOL3Uri(URI inputUri, Parameters parameters)`: Converts SBOL2 URI format to SBOL3 URI format
  - Handles version resolution from RDF model
  - Transforms URI structure from `namespace/displayId/version` to `namespace/version/displayId`
  - Falls back to default namespace if URI parsing fails

#### 3. **SubComponent Creation**
```java
SubComponent resultSC = null;
if (input.getDisplayId() != null) {
    resultSC = sbol3ParentComponent.createSubComponent(input.getDisplayId(), sbol3InstanceOfUri);
} else {
    resultSC = sbol3ParentComponent.createSubComponent(sbol3InstanceOfUri);
}
```

**What this does:**
- Creates a new SubComponent in the parent Component
- If the SBOL2 Component has a displayId, it's used; otherwise, one is auto-generated

#### 4. **Identity Mapping**
```java
parameters.addMapping(input.getIdentity(), resultSC.getUri());
```

**What this does:**
- Records the mapping between the SBOL2 Component's identity URI and the new SBOL3 SubComponent's URI
- This mapping is used for cross-references throughout the conversion process

#### 5. **Roles Conversion**
```java
resultSC.setRoles(Util.toList(input.getRoles()));
```

**What this does:**
- Converts the Set of role URIs from SBOL2 to a List for SBOL3
- Roles define the biological function (e.g., promoter, terminator, coding sequence)

**Util Function Used:**
- `Util.toList(Set<T> set)`: Simple utility to convert a Set to a List
  - Creates a new ArrayList containing all elements from the input Set

#### 6. **Role Integration Conversion**
```java
RoleIntegrationType ri = input.getRoleIntegration();
if (ri != null) {
    RoleIntegration risbol3 = RoleIntegration.mergeRoles;
    
    if (ri.equals(RoleIntegrationType.MERGEROLES)) {
        risbol3 = RoleIntegration.mergeRoles;
    } else if (ri.equals(RoleIntegrationType.OVERRIDEROLES)) {
        risbol3 = RoleIntegration.overrideRoles;
    }
    resultSC.setRoleIntegration(risbol3);
}
```

**What this does:**
- Converts SBOL2 role integration type to SBOL3 equivalent
- Role integration determines how roles from the definition are combined with roles from the instance
  - `MERGEROLES`: Combine roles from both definition and instance
  - `OVERRIDEROLES`: Use only instance roles, ignoring definition roles

#### 7. **Measurements Conversion**
```java
if (input.getMeasures() != null) {
    MeasurementConverter measurementConverter = new MeasurementConverter();
    for (Measure sbol2Measure : input.getMeasures()) {
        measurementConverter.convert(doc, result, input, sbol2Measure, parameters);
    }
}
```

**What this does:**
- Converts any measurements associated with the Component
- Measurements specify quantitative properties (e.g., concentration, length)

#### 8. **Common Fields Copying**
```java
Util.copyIdentified(input, result, parameters);
```

**What this does:**
- Copies common Identified entity fields from SBOL2 Component to SBOL3 SubComponent:
  - `name`: Human-readable name
  - `description`: Textual description
  - `wasDerivedFrom`: Provenance links
  - `annotations`: Custom metadata (converted from SBOL2 Annotation format to SBOL3 Metadata format)
  - `attachments`: Files or other resources attached to the entity

**Util Function Used:**
- `Util.copyIdentified(org.sbolstandard.core2.Identified input, Identified output, Parameters parameters)`
  - Copies `name` and `description` directly
  - Converts `wasDerivedFrom` Set to List
  - Handles TopLevel-specific properties (attachments)
  - Converts SBOL2 Annotations to SBOL3 Metadata recursively (handles nested annotations)

#### 9. **Access Type Handling (Public Components)**
```java
if (input.getAccess() == AccessType.PUBLIC) {
    Interface sbol3Interface = sbol3ParentComponent.getInterface();
    if (sbol3Interface == null) {
        sbol3Interface = sbol3ParentComponent.createInterface();
    }
    
    List<Feature> features = sbol3Interface.getNonDirectionals();
    if (features == null) {
        features = new ArrayList<>();
    }
    features.add(result);
    sbol3Interface.setNonDirectionals(features);
}
```

**What this does:**
- If the SBOL2 Component has PUBLIC access, it's added to the parent Component's Interface
- In SBOL3, Interfaces define interaction points between Components
- PUBLIC components are added as non-directional features (accessible from outside the parent Component)

**Conceptual Mapping:**
- SBOL2 `AccessType.PUBLIC` → SBOL3 Interface's `nonDirectionals` list

#### 10. **Source Locations Conversion**
```java
if (input.getSourceLocations() != null) {
    SourceLocationConverter locationConverter = new SourceLocationConverter();
    for (Location loc : input.getSourceLocations()) {
        locationConverter.convert(doc, result, input, loc, parameters);
    }
}
```

**What this does:**
- Converts source locations (Range, Cut, GenericLocation) from SBOL2 format to SBOL3
- Source locations specify where in a sequence the Component is located
- Each location type is handled differently:
  - **Range**: Start and end positions
  - **Cut**: Single position
  - **GenericLocation**: Entire sequence

#### 11. **Return Result**
```java
return result;
```
Returns the converted SubComponent.

## SBOL2 to SBOL3 Concept Mapping

| SBOL2 Concept | SBOL3 Concept | Notes |
|--------------|--------------|-------|
| `Component` | `SubComponent` | Instance of a ComponentDefinition within a Component |
| `Component.getDefinitionURI()` | `SubComponent.getInstanceOf()` | Reference to the type definition |
| `Component.getRoles()` (Set) | `SubComponent.getRoles()` (List) | Biological roles |
| `Component.getRoleIntegration()` | `SubComponent.getRoleIntegration()` | How roles are combined |
| `Component.getAccess()` (PUBLIC) | `Interface.getNonDirectionals()` | Public access mapping |
| `Component.getMeasures()` | `Measurement` entities | Quantitative properties |
| `Component.getSourceLocations()` | `SubComponent.getLocations()` | Sequence locations |
| `Component` annotations | `SubComponent` metadata | Custom properties |

## Example Conversion Flow

```
SBOL2 Input:
  Component {
    identity: http://example.org/CD1/comp1/1.0
    displayId: comp1
    definitionURI: http://example.org/CD1/1.0
    roles: [http://identifiers.org/so/SO:0000167]  // promoter
    roleIntegration: MERGEROLES
    access: PUBLIC
    name: "Promoter Component"
    description: "A promoter sequence"
  }

Conversion Process:
  1. Convert definitionURI to SBOL3 format
  2. Create SubComponent in parent Component
  3. Set roles (convert Set to List)
  4. Set roleIntegration
  5. Copy name, description, annotations
  6. Add to Interface.nonDirectionals (if PUBLIC)

SBOL3 Output:
  SubComponent {
    uri: http://example.org/1.0/comp1
    displayId: comp1
    instanceOf: http://example.org/1.0/CD1
    roles: [http://identifiers.org/so/SO:0000167]  // List
    roleIntegration: mergeRoles
    name: "Promoter Component"
    description: "A promoter sequence"
    // Added to parent.interface.nonDirectionals
  }
```

## Dependencies

### External Converters Used:
- **MeasurementConverter**: Converts SBOL2 Measures to SBOL3 Measurements
- **SourceLocationConverter**: Converts SBOL2 Locations (Range, Cut, GenericLocation) to SBOL3 Locations

### SBOL Library Classes:
- SBOL2: `org.sbolstandard.core2.Component`, `AccessType`, `RoleIntegrationType`, `Location`, `Measure`
- SBOL3: `org.sbolstandard.core3.entity.Component`, `SubComponent`, `Feature`, `Interface`, `SBOLDocument`

## Error Handling

The converter can throw:
- **SBOLGraphException**: If there's an issue with the SBOL graph structure (e.g., invalid URI, missing parent)
- **SBOLValidationException**: If the resulting SBOL3 data is invalid according to SBOL3 schema

## Notes for Implementation in Other Languages

When implementing this converter in Python or other languages:

1. **URI Format Conversion**: Implement SBOL2 → SBOL3 URI transformation (namespace/displayId/version → namespace/version/displayId)

2. **Set to List Conversion**: SBOL2 uses Sets, SBOL3 uses Lists for roles and wasDerivedFrom

3. **Annotation System**: SBOL2 uses QName-based annotations, SBOL3 uses URI-based Metadata. The conversion is recursive for nested structures.

4. **Role Integration**: Map enum values:
   - `MERGEROLES` → `mergeRoles`
   - `OVERRIDEROLES` → `overrideRoles`

5. **Interface Handling**: In SBOL3, PUBLIC access maps to Interface's nonDirectionals list

6. **Identity Mapping**: Maintain a mapping dictionary/table to track SBOL2 identity → SBOL3 URI conversions

7. **Type Checking**: Ensure proper type casting (Component, SubComponent, Feature)

## Related Converters

- **ComponentDefinitionConverter**: Converts the ComponentDefinition that Components reference
- **SequenceAnnotationToFeatureConverter**: Converts SequenceAnnotations to Features
- **FunctionalComponentToSubComponentConverter**: Converts FunctionalComponents (from ModuleDefinitions) to SubComponents

