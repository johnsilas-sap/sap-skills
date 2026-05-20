# SAP BAdI Development Skill

ABAP development skill for SAP BAdIs (Business Add-Ins) — classic and kernel BAdI definitions, implementation classes, filter-dependent BAdIs, enhancement spots, and full EWM/TM BAdI catalogs.

## Skill Overview

- **BAdI Generations**: Classic (CL_EXITHANDLER), Kernel (GET BADI/CALL BADI), Enhancement Spots
- **Definition**: SE18 — interface, methods, filter type, multiple-use flag
- **Implementation**: SE19 — implementation class, filter values, activation
- **Kernel Pattern**: GET BADI / CALL BADI / LOOP AT for multiple implementations
- **Filter BADIs**: Warehouse number, document type, plant, movement type filters
- **EWM BAdIs**: Delivery processing, TO/WT creation/confirmation, RF mobile, packing, resource
- **TM BAdIs**: Carrier selection, charge calculation, freight order processing, track & trace
- **Testing**: Breakpoints in impl. class, SE19 test function, switch framework activation

## Auto-Trigger Keywords

### Core BAdI
- BAdI, Business Add-In, BADI, BAdi
- SE18, SE19, BAdI Builder, BAdI implementation
- enhancement spot, SXSB_SPOT, enhancement framework
- kernel BAdI, new BAdI, classic BAdI

### BAdI Patterns
- GET BADI, CALL BADI, LOOP AT BADI
- CL_EXITHANDLER, GET_INSTANCE, IF_EX
- implementation class, BAdI interface
- filter-dependent, filter value, BAdI filter

### Classic BADIs
- CL_EXITHANDLER=>GET_INSTANCE
- IF_EX_ interface, classic BAdI
- adapter class, single-use BAdI

### Enhancement Framework
- enhancement spot, BAdI spot
- SFW5, switch framework, business function
- SPRO BAdI, IMG BAdI, customizing BAdI
- active implementation, BAdI activation

### EWM BADIs
- /SCWM/ES_, EWM enhancement spot
- warehouse order, warehouse task, picking BAdI
- TO creation BAdI, WT confirmation BAdI
- EWM delivery BAdI, /SCDL/ BAdI
- RF BAdI, RF mobile enhancement, ITSmobile BAdI
- packing BAdI, HU BAdI, handling unit BAdI
- resource BAdI, /SCWM/RSRC BAdI

### TM BADIs
- /SCMTMS/ES_, TM enhancement spot
- carrier selection BAdI, tendering BAdI
- charge calculation BAdI, settlement BAdI
- freight order BAdI, FO processing BAdI
- track and trace BAdI, /SAPTRX/ BAdI

### ABAP Enhancement
- exit, user exit, enhancement, modification
- implicit enhancement, explicit enhancement
- ABAP OO BAdI, interface method, FINAL class
- where-used enhancement, find BAdI, search BAdI
