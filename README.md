# Report on XML XPath Mapping Issue

During the migration and mapping process of LSU content, we noticed something interesting about how the `title` element is mapped from MODS XML files. Initially, I thought the issue was about the **namespace version** causing inconsistencies, but after further analysis, I found that it’s not about the **namespace version** itself—it’s about the **structure** and **attributes** included in the `mods` element.

## Etymologies Used in This Report
To ensure clarity, the following terms are used consistently throughout this report:

- **Namespace**: A declaration (`xmlns` or `xmlns:prefix`) that defines the scope of element and attribute names, used to avoid name conflicts in XML.
- **Attribute**: Key-value pairs directly associated with an XML element (e.g., `version="3.7"` or `xsi:schemaLocation="...")`.
- **Schema Location**: A specific attribute (`xsi:schemaLocation`) that provides hints about the schema used for validation.
- **ElementTree .attrib**: The Python `xml.etree.ElementTree` module’s method to extract attributes from an XML element.

---

# What I Found

### 1. lasc-earlylaw Content (Incorrected mapping from xml namespace):

- The XPath `title/info/title` **doesn’t work** for the `title` element here.

- This XML includes the `xsi:schemaLocation` attribute:

- **Schema location definition:**
  - `xsi:schemaLocation="http://www.loc.gov/mods/v3 http://www.loc.gov/standards/mods/v3/mods-3-7.xsd"`

- **`mods` attribute extracted from `lasc-earlylaw` MODS content that maps correct title information using xml.etree.ElementTree:**
  - `mods [@schemaLocation='http://www.loc.gov/mods/v3 http://www.loc.gov/standards/mods/v3/mods-3-7.xsd', @version='3.7']`

- **Actual Namespaces defined in the XML in `mods` element:**
  - `<mods xmlns="http://www.loc.gov/mods/v3" xmlns:mods="http://www.loc.gov/mods/v3" xmlns:xlink="http://www.w3.org/1999/xlink" xmlns:xs="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" version="3.7" xsi:schemaLocation="http://www.loc.gov/mods/v3 http://www.loc.gov/standards/mods/v3/mods-3-7.xsd">`


### 2. LSU Content (Corrected mapping from xml namespace):

- The XPath `title/info/title` works **perfectly** for mapping the `title` element.

- This XML doesn’t include the `xsi:schemaLocation` attribute.

- The parser interpreted the structure **hierarchically**, recognizing **info** as an **intermediate element before title**.

- **Schema location definition:**
  - `schemaLocation = 'http://www.loc.gov/mods/v3 http://www.loc.gov/standards/mods/v3/mods-3-7.xsd']/titleInfo/title`

- **`mods` attribute extracted from `LSU` MODS content that maps correct title information using xml.etree.ElementTree:** 
  - `mods [@version = '3.7', @{http://www.w3.org/2001/XMLSchema-instance}schemaLocation = 'http://www.loc.gov/mods/v3 http://www.loc.gov/standards/mods/v3/mods-3-7.xsd']/titleInfo/title`

- **Actual Namespaces defined in the XML in `mods` element:**
  - `<mods xmlns="http://www.loc.gov/mods/v3" xmlns:mods="http://www.loc.gov/mods/v3" xmlns:xlink="http://www.w3.org/1999/xlink" xmlns:xs="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" version="3.7">`

---

# What’s Causing This
The difference comes from the `xsi:schemaLocation` attribute in the second XML file. It’s providing schema validation hints, which makes the parser:
- Follow the schema-defined layout instead of a looser interpretation.
- Skip intermediate elements like `titleInfo` to get text from this element and directly maps the text to `mods` instead of `mods/titleInfo/title`.

Even though both files share the same namespace (`http://www.loc.gov/mods/v3`), the schema hints make a big difference in how the structure is processed. The version (`version="3.7"`) and schema rigidity also seem to influence this behavior.

### Additional Insights on `xml.etree.ElementTree` Behavior
1. Namespace declarations (`xmlns`, `xmlns:prefix`) are not treated as attributes. These are metadata used to qualify element and attribute names but are not part of `.attrib`.
2. Attributes like `version` and `xsi:schemaLocation` are considered standard attributes and will appear in `.attrib`.
3. Namespaced attributes (e.g., `xsi:schemaLocation`) will have their keys qualified with the full namespace URI, such as:
   ```python
   {'{http://www.w3.org/2001/XMLSchema-instance}schemaLocation': '...'}
   ```

---

# Suggestions

To avoid this kind of inconsistency in the future we should:

### 1. Standardize XML Files:
- Either remove or align the `xsi:schemaLocation` attribute across all files to avoid these structural differences.

### 2. Use Explicit Namespace Queries:
- Updating XPath queries to directly handle namespaces, like this:
```python
namespace = {'mods': 'http://www.loc.gov/mods/v3'}
title = tree.find('.//mods:title', namespaces=namespace) #Example for `.find` in `ElementTree`
```
---

# Summery of issue Conclusion
This issue isn’t about namespace differences like I initially thought. It’s more about the extra attributes and schema-driven structure in the XML.

