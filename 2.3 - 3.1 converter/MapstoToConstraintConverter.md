MapstoToConstraintConverter

⚙️ File Purpose
    Converts one MapsTo object (SBOL 2) into a Constraint object (SBOL 3).
    Ensures logical relationships between parts are preserved.
    Handles all 4 possible refinement types that existed in SBOL 2.


1️⃣ Method Signature

```java
public static Constraint convert(
    Component sbol3ParentComponent,
    MapsTo mapsTo,
    ComponentReference sbol3CompRef,
    Parameters parameters)
```
So it takes:
    sbol3ParentComponent: the parent SBOL 3 component where the constraint will live
    mapsTo: the SBOL 2 mapping rule (what maps to what)
    sbol3CompRef: a reference to a component (used for target)
    parameters: carries ID mappings between SBOL2 and SBOL3 entities
It returns → a Constraint in SBOL 3.


2️⃣ Get the Local SubComponent
```java
SubComponent sbol3LocalSubComponent =
    Util.getSBOL3Entity(sbol3ParentComponent.getSubComponents(),
                        mapsTo.getLocal(), parameters);
```

This line finds the SBOL 3 equivalent of the local component (the thing being mapped in SBOL 2).
It uses a helper from Util to look up the new entity by its mapped URI.


3️⃣ RefinementType — the “meaning” of the mapping

SBOL 2 used something called RefinementType to say how two parts relate.
These are like relationship labels:

| SBOL 2 Type       | Meaning                | SBOL 3 Equivalent                                      |
| ----------------- | ---------------------- | ------------------------------------------------------ |
| `USELOCAL`        | Prefer the local copy  | `IdentityRestriction.replaces` (local replaces remote) |
| `USEREMOTE`       | Prefer the remote copy | `IdentityRestriction.replaces` (remote replaces local) |
| `VERIFYIDENTICAL` | They must be identical | `IdentityRestriction.verifyIdentical`                  |
| `MERGE`           | Merge them             | Removed in SBOL 3 (treated as USEREMOTE)               |



4️⃣ The Conversion Logic

🔹 Case 1: USELOCAL
```java
sbol3Constraint = sbol3ParentComponent.createConstraint(
    RestrictionType.IdentityRestriction.replaces,
    sbol3LocalSubComponent, sbol3CompRef);
```
→ Meaning: “The local subcomponent replaces the remote one.”


🔹 Case 2: USEREMOTE
```java
sbol3Constraint = sbol3ParentComponent.createConstraint(
    RestrictionType.IdentityRestriction.replaces,
    sbol3CompRef, sbol3LocalSubComponent);
```
→ Meaning: “The remote one replaces the local.”


🔹 Case 3: VERIFYIDENTICAL
```java
sbol3Constraint = sbol3ParentComponent.createConstraint(
    RestrictionType.IdentityRestriction.verifyIdentical,
    sbol3CompRef, sbol3LocalSubComponent);
```
→ Meaning: “They must be the same; verify identical identity.”


🔹 Case 4: MERGE

// REMOVED IN SBOL3. HANDLED AS USEREMOTE
→ SBOL 3 doesn’t have “merge” anymore, so it treats it like “remote replaces local.”