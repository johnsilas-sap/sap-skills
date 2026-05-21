# ABAP RTTI and Dynamic Programming Reference

## Runtime Type Services (RTTI) Overview

```
CL_ABAP_TYPEDESCR           — base class for all type descriptors
  ├── CL_ABAP_ELEMDESCR     — elementary types (char, int, date...)
  ├── CL_ABAP_STRUCTDESCR   — structures
  ├── CL_ABAP_TABLEDESCR    — internal tables
  ├── CL_ABAP_REFDESCR      — reference types (REF TO ...)
  └── CL_ABAP_CLASSDESCR    — classes
      CL_ABAP_INTFDESCR     — interfaces
      CL_ABAP_OBJECTDESCR   — base for class+interface descriptors
```

## Describe Type of Variable

```abap
DATA ls_delivery TYPE zewm_delivery.
DATA lt_items    TYPE zewm_item_t.
DATA lo_proc     TYPE REF TO zcl_delivery_processor.

" Get type descriptor:
DATA(lo_type) = cl_abap_typedescr=>describe_by_data( ls_delivery ).
DATA(lv_kind) = lo_type->kind.   " 'S' = structure, 'T' = table, 'E' = elementary, 'r' = ref, 'C' = class

" Check kind constants:
IF lo_type->kind = cl_abap_typedescr=>kind_struct.
  DATA(lo_struct) = CAST cl_abap_structdescr( lo_type ).
ELSEIF lo_type->kind = cl_abap_typedescr=>kind_table.
  DATA(lo_table) = CAST cl_abap_tabledescr( lo_type ).
ELSEIF lo_type->kind = cl_abap_typedescr=>kind_elem.
  DATA(lo_elem) = CAST cl_abap_elemdescr( lo_type ).
ENDIF.

" Describe by name (compile-time unknown type):
DATA(lo_by_name) = cl_abap_typedescr=>describe_by_name( 'ZEWM_DELIVERY' ).
```

## Inspect Structure Components

```abap
DATA(lo_struct) = CAST cl_abap_structdescr(
  cl_abap_typedescr=>describe_by_data( ls_delivery )
).

" Get all fields:
DATA(lt_components) = lo_struct->get_components( ).

LOOP AT lt_components ASSIGNING FIELD-SYMBOL(<comp>).
  DATA(lv_field_name) = <comp>-name.        " e.g. 'DELIVERY_NO'
  DATA(lo_field_type) = <comp>-type.        " CL_ABAP_ELEMDESCR
  DATA(lv_type_kind)  = <comp>-type->kind.  " 'C' = char, 'N' = numc, 'D' = date...
  DATA(lv_length)     = CAST cl_abap_elemdescr( <comp>-type )->length.
ENDLOOP.
```

## Dynamic FIELD-SYMBOL Access

```abap
" Access field by runtime-determined name:
ASSIGN COMPONENT 'DELIVERY_NO' OF STRUCTURE ls_delivery TO FIELD-SYMBOL(<field>).
IF sy-subrc = 0.
  DATA(lv_value) = CAST string( <field> ).   " read value
  <field> = 'NEW_VALUE'.                      " set value
ENDIF.

" Iterate all fields of a structure dynamically:
DATA(lo_sdescr) = CAST cl_abap_structdescr(
  cl_abap_typedescr=>describe_by_data( ls_delivery )
).
LOOP AT lo_sdescr->get_components( ) ASSIGNING FIELD-SYMBOL(<c>).
  ASSIGN COMPONENT <c>-name OF STRUCTURE ls_delivery TO FIELD-SYMBOL(<val>).
  IF <val> IS ASSIGNED.
    " Process each field value generically
    lo_output->set_field( name = <c>-name  value = |{ <val> }| ).
    UNASSIGN <val>.
  ENDIF.
ENDLOOP.
```

## Dynamic Table Operations (REF TO DATA)

```abap
" Create a table dynamically from a type name:
DATA lo_table_ref TYPE REF TO data.
CREATE DATA lo_table_ref TYPE TABLE OF ('ZEWM_DELIVERY').  " type by name
ASSIGN lo_table_ref->* TO FIELD-SYMBOL(<lt_dynamic>).

" SELECT into dynamic table:
SELECT * FROM zewm_delivery INTO TABLE @<lt_dynamic>.

" Access rows:
LOOP AT <lt_dynamic> ASSIGNING FIELD-SYMBOL(<row>).
  ASSIGN COMPONENT 'DELIVERY_NO' OF STRUCTURE <row> TO FIELD-SYMBOL(<no>).
  IF <no> IS ASSIGNED.
    WRITE: / <no>.
  ENDIF.
ENDLOOP.

" Create a structure dynamically:
DATA lo_struct_ref TYPE REF TO data.
CREATE DATA lo_struct_ref TYPE ('ZEWM_DELIVERY').
ASSIGN lo_struct_ref->* TO FIELD-SYMBOL(<ls_dynamic>).
ASSIGN COMPONENT 'STATUS' OF STRUCTURE <ls_dynamic> TO FIELD-SYMBOL(<status>).
<status> = 'O'.
```

## Dynamic Method Call (CALL METHOD)

```abap
" Call method by name at runtime:
DATA lo_obj TYPE REF TO object.
lo_obj = NEW zcl_delivery_processor( ).

" Dynamic call — all parameters via PARAMETER-TABLE / EXCEPTION-TABLE:
DATA lt_params  TYPE abap_parmbind_tab.
DATA lt_excepts TYPE abap_excpbind_tab.

APPEND VALUE #( name = 'IV_DELIVERY_NO' kind = cl_abap_objectdescr=>exporting
                value = REF #( lv_delivery_no ) ) TO lt_params.
APPEND VALUE #( name = 'RV_NEW_STATUS'  kind = cl_abap_objectdescr=>returning
                value = REF #( lv_result ) )       TO lt_params.

CALL METHOD lo_obj->('PROCESS')         " method name as string
  PARAMETER-TABLE lt_params
  EXCEPTION-TABLE lt_excepts.

IF sy-subrc <> 0.
  " exception table contains raised exception name
ENDIF.
```

## CL_ABAP_CLASSDESCR — Class Inspection

```abap
" Describe a class:
DATA(lo_class) = CAST cl_abap_classdescr(
  cl_abap_typedescr=>describe_by_object_ref( lo_obj )
).

" Get class name:
DATA(lv_class_name) = lo_class->get_relative_name( ).   " e.g. 'ZCL_DELIVERY_PROCESSOR'

" Get all public methods:
DATA(lt_methods) = lo_class->methods.
LOOP AT lt_methods ASSIGNING FIELD-SYMBOL(<meth>).
  DATA(lv_meth_name) = <meth>-name.
  " Check method parameters:
  LOOP AT <meth>-parameters ASSIGNING FIELD-SYMBOL(<param>).
    DATA(lv_param_name) = <param>-name.
    DATA(lv_param_type) = <param>-type_kind.
  ENDLOOP.
ENDLOOP.

" Check if class implements interface:
DATA(lt_interfaces) = lo_class->interfaces.
IF line_exists( lt_interfaces[ name = 'ZIF_DELIVERY_PROCESSOR' ] ).
  " class implements the interface
ENDIF.
```

## Generic Programming with ANY

```abap
" Process any structure type generically:
METHODS set_field
  IMPORTING
    iv_structure_ref TYPE REF TO data    " pass REF #( ls_struct )
    iv_field_name    TYPE string
    iv_value         TYPE any.

METHOD set_field.
  ASSIGN iv_structure_ref->* TO FIELD-SYMBOL(<struct>).
  ASSIGN COMPONENT iv_field_name OF STRUCTURE <struct> TO FIELD-SYMBOL(<field>).
  IF <field> IS ASSIGNED.
    <field> = iv_value.
  ENDIF.
ENDMETHOD.

" Call:
set_field(
  iv_structure_ref = REF #( ls_delivery )
  iv_field_name    = 'STATUS'
  iv_value         = 'C'
).
```

## Object Reference Comparison

```abap
" Check if two references point to same instance:
IF lo_proc1 = lo_proc2.    " same object in memory
  " identical instance
ENDIF.

" Check type:
IF lo_proc IS INSTANCE OF zcl_delivery_processor.
ENDIF.

" Get runtime type name:
DATA(lv_type_name) = cl_abap_typedescr=>describe_by_object_ref( lo_proc )->get_relative_name( ).
```
