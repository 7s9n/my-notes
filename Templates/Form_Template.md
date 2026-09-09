
---

### 🧩 Specialized Template for `30_Forms` (Object-Centric)

Since you build and improve forms, use this template *inside* a form's subfolder to track its entire lifecycle:

```markdown
---
date: {{date}}
module: "MRP"
type: "Form"
status: "Active | Deprecated"
form_name: "{{form_name}}"
version: "1.0"
last_enhancement: ""
---

# Form: {{form_name}}

## 📂 File Locations
- **FMB Path**: `\\server\forms\mrp\{{form_name}}.fmb`
- **PLL Dependencies**: 

## 🧩 Core Functionality
_What does this form do in the MRP cycle?_

## 📜 Enhancement History
- **{{date}}**: [Initial Build / Fixed bug X / Added Y feature] -> [[Link to Issue Note]]

## 🔗 Related SQL / Packages
- [[QRY_GET_BOM]]
- [[PKG_VALIDATE_ROUTING]]