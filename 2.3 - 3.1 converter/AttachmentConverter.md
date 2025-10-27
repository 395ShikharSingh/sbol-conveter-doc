AttachmentConverter


Step 1: Create the new SBOL 3 Attachment

```java
Attachment att = doc.createAttachment(
    Util.createSBOL3Uri(input),
    Util.getNamespace(input),
    Util.createSBOL3Uri(input.getSource(), parameters)
);
```


| Argument                                             | What it represents                              | Whyneeded         |

| ---------------------------------------------------- | --------------------------------------------|   ------------------- |
| `Util.createSBOL3Uri(input)`                         | New unique SBOL 3 URI for this attachment       | Identifies the attachment |
| `Util.getNamespace(input)`                           | The namespace                                   | Proper grouping           |
| `Util.createSBOL3Uri(input.getSource(), parameters)` | Converts source URI (where the file is located) | Retains link to file      |





Step 2: Copy general metadata

```java
Util.copyIdentified(input, att, parameters);
```

Copies common properties like:
    displayId
    name
    description
    version




Step 3: Copy hash (optional)

```java
att.setHash(input.getHash());
```

✅ The hash acts as a file integrity check (like an MD5 checksum).
If the file changes, the hash will change, so this is important.




Step 4: Copy file size (only if present)

```java
if (input.getSize() != null) {
    att.setSize(OptionalLong.of(input.getSize()));
}
```

✅ Converts old size value (Long) to a new format (OptionalLong), because SBOL 3 requires it wrapped safely.





Step 5: Copy file format (like MIME type)

```java
if (input.getFormat() != null) {
    att.setFormat(Util.createSBOL3Uri(input.getFormat(), parameters));
}
```

✅ Some attachments specify their format (e.g., "application/pdf" or "image/png"). This converts that value into an SBOL 3-compliant URI form.



Step 6: Return final converted attachment

```java
return att;
```







## Complete Implementation

```java
package org.sbolstandard.converter.sbol23_31;

import java.util.OptionalLong;

import org.sbolstandard.converter.Util;
import org.sbolstandard.core3.entity.Attachment;
import org.sbolstandard.core3.entity.SBOLDocument;
import org.sbolstandard.core3.util.SBOLGraphException;

public class AttachmentConverter implements EntityConverter<org.sbolstandard.core2.Attachment, Attachment> {

	@Override
	public Attachment convert(SBOLDocument doc, org.sbolstandard.core2.Attachment input, Parameters parameters) throws SBOLGraphException {
		Attachment att = doc.createAttachment(Util.createSBOL3Uri(input), Util.getNamespace(input),	Util.createSBOL3Uri(input.getSource(), parameters));
		Util.copyIdentified(input, att, parameters);
		att.setHash(input.getHash());
		if (input.getSize() != null) {
			att.setSize(OptionalLong.of(input.getSize()));
		}
		if (input.getFormat() != null) {
			att.setFormat(Util.createSBOL3Uri(input.getFormat(), parameters));
		}
		return att;
	}
}
```
