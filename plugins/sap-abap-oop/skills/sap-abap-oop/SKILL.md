---
name: sap-abap-oop
description: |
  ABAP Object-Oriented Programming for modern SAP ABAP development. Covers class definition and implementation (visibility sections, static/instance members, constructor, FINAL/ABSTRACT), interfaces and multiple implementation, inheritance (INHERITING FROM, REDEFINITION, super calls, polymorphism, CAST/IS), exception handling (CX_ hierarchy, TRY/CATCH/CLEANUP, IF_T100_MESSAGE), design patterns (Factory/Singleton/Strategy/Observer/Template Method), runtime type services (RTTI), modern ABAP expressions (VALUE/CORRESPONDING/FILTER/FOR/COND/REDUCE/string templates), and OOP patterns for EWM/TM service and repository classes.

  Key patterns:
  - Interface-based dependency injection for testability
  - Factory class for object creation with environment switch
  - Exception class with T100 message and attribute-rich context
  - Strategy pattern for pluggable algorithm selection
  - Repository pattern: data access isolated behind interface
  - FOR ... IN ... WHERE table comprehension expressions
  - RTTI for dynamic type inspection and generic processing
  - CAST / IS / ?= for safe OO downcasting
license: GPL-3.0
metadata:
  version: "1.0.0"
  last_verified: "2026-05-20"
  abap_version: "ABAP 7.58+ / S/4HANA 2022+"
---

# SAP ABAP Object-Oriented Programming

## Class Skeleton

```abap
CLASS zcl_delivery_processor DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION.
    INTERFACES zif_delivery_processor.
    CLASS-METHODS get_instance RETURNING VALUE(ro_instance) TYPE REF TO zif_delivery_processor.
    METHODS constructor IMPORTING io_repo TYPE REF TO zif_delivery_repository.
  PROTECTED SECTION.
  PRIVATE SECTION.
    CLASS-DATA go_instance TYPE REF TO zcl_delivery_processor.
    DATA mo_repo TYPE REF TO zif_delivery_repository.
    METHODS validate IMPORTING is_delivery TYPE zewm_delivery
                     RAISING   zcx_delivery_error.
ENDCLASS.

CLASS zcl_delivery_processor IMPLEMENTATION.
  METHOD constructor.
    mo_repo = io_repo.
  ENDMETHOD.
  METHOD get_instance.
    IF go_instance IS NOT BOUND.
      go_instance = NEW #( NEW zcl_delivery_db_repo( ) ).
    ENDIF.
    ro_instance = go_instance.
  ENDMETHOD.
ENDCLASS.
```

## Interface

```abap
INTERFACE zif_delivery_processor.
  METHODS process
    IMPORTING iv_delivery_no TYPE /scwm/de_lgnum
    RETURNING VALUE(rv_new_status) TYPE char1
    RAISING   zcx_delivery_error.
ENDINTERFACE.
```

## Key Modern ABAP Expressions

```abap
" VALUE constructor
DATA(ls_item) = VALUE zewm_item( material_no = 'M01'  quantity = 10  unit = 'PC' ).

" CORRESPONDING field mapping
DATA(ls_out) = CORRESPONDING zewm_output( ls_input ).

" FOR table comprehension
DATA(lt_open) = VALUE zewm_delivery_t(
  FOR ls IN lt_all WHERE ( status = 'O' ) ( ls )
).

" COND inline condition
DATA(lv_label) = COND string( WHEN lv_qty > 0 THEN 'Positive' ELSE 'Zero' ).
```
