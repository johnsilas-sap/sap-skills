# SAP Adobe Forms Development Skill

ABAP development skill for SAP Adobe Forms (ADS) — form building in SFP, print programs, layout design, scripting, PPF integration, and PDF output.

## Skill Overview

- **Form Structure**: Interface (ABAP imports/globals/init) + Layout (Adobe LiveCycle Designer)
- **Print Programs**: FP_JOB_OPEN → FP_FUNCTION_MODULE_NAME → generated FM → FP_JOB_CLOSE
- **Output Parameters**: Printer, PDF retrieval, archiving, preview, copies, language/country
- **Form Interface**: Import parameters, global data, initialization ABAP code
- **Layout Design**: Master pages, subforms, repeating rows, barcode fields, images
- **XFA JavaScript**: Field scripts, conditional visibility, running totals, page breaks
- **Barcodes**: Code128, QR Code, PDF417, EAN-13 — field binding and data formatting
- **PPF Integration**: IF_FPF_ACTION implementation for event-driven form printing
- **Email Output**: PDF as attachment via SO_NEW_DOCUMENT_ATT_SEND_API1
- **Interactive Forms**: Fillable PDF, signature fields, data extraction from submitted PDF
- **ADS Administration**: SFPADM, SM59 connection, SICF activation
- **EWM/TM Forms**: Delivery notes, warehouse labels, TO slips, DG declarations, CMR

## Auto-Trigger Keywords

### Core Adobe Forms
- Adobe Forms, ADS, Adobe Document Services
- SFP, Form Builder, form interface, form layout
- Adobe LiveCycle Designer, XFA, XML Forms Architecture
- FP_JOB_OPEN, FP_JOB_CLOSE, FP_FUNCTION_MODULE_NAME
- SFPOUTPUTPARAMS, SFPDOCPARAMS, print form

### Print Programs
- print program, ABAP print, form call
- FP_JOB_OPEN, FP_JOB_CLOSE, FP_CHECK_DESTINATION
- generated function module, form FM
- spool, SP01, output request, print job

### Form Interface and Context
- form interface, import parameter, global definition
- initialization code, form context, data binding
- ABAP interface, form data, context mapping
- form node, data node, schema node

### Layout and Design
- master page, subform, body page
- repeating subform, table rows, flowed subform
- barcode field, Code128, QR code, PDF417
- text field, numeric field, image field
- header, footer, page number, watermark

### JavaScript / XFA Scripting
- XFA script, JavaScript form, field script
- Initialize event, Calculate event, Validate event
- presence, visible, hidden, conditional field
- rawValue, formattedValue, xfa.resolveNodes
- page break, relayout, dynamic form

### Output and Delivery
- PDF output, print output, email PDF
- SFPOUTPUTPARAMS, nodialog, getpdf, dest
- email attachment, SO_NEW_DOCUMENT_ATT_SEND_API1
- archiving, ArchiveLink, PDF archive
- preview, PREVIEW flag, form preview

### PPF Integration
- PPF, Post Processing Framework, PPF action
- IF_FPF_ACTION, PPF processing class
- output condition, NAST, output type
- automatic printing, event-driven print

### ADS Administration
- SFPADM, ADS administration, ADS connection
- SM59 ADS destination, SICF fp service
- ADS server, ADS Java, FP_CHECK_DESTINATION

### Interactive Forms
- interactive PDF, fillable form, form fields
- signature field, digital signature
- submit button, data extraction from PDF
- Web Dynpro, interactive scenario

### Form Types and Documents (EWM/TM/SD/MM)
- delivery note, warehouse label, picking list
- transfer order slip, TO confirmation
- goods receipt slip, GR document
- dangerous goods declaration, DG form, CMR
- shipping label, packing list, bill of lading
- invoice form, purchase order form

## Directory Structure

```
sap-adobe-forms/
├── SKILL.md                              # Main skill — architecture, print program pattern
├── README.md                             # This file
└── references/
    ├── form-interface.md                 # Interface design, imports, globals, init code
    ├── print-program.md                  # Full ABAP print program patterns
    ├── layout-design.md                  # Subforms, barcodes, tables, master pages
    ├── xfa-scripting.md                  # JavaScript events, conditional logic, calculations
    ├── ppf-integration.md                # PPF action class, output conditions, triggers
    ├── interactive-forms.md              # Fillable PDF, signature, data submission
    └── ads-administration.md             # SFPADM, SM59, SICF, troubleshooting
```

## Source Documentation

- SAP Adobe Forms: https://help.sap.com/docs/ABAP_PLATFORM_NEW/a996ed769f474ef78e6b35e03f4bffe0
- SAP NetWeaver ADS: https://help.sap.com/docs/SAP_NETWEAVER

## Version

- **Skill Version**: 1.0.0
- **Last Updated**: 2026-05-19
- **Reference Files**: 7

---

## License

GPL-3.0 License — see LICENSE file in repository root.
