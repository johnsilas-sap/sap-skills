# ABAP Inheritance Reference

## INHERITING FROM

```abap
" Base class — can be abstract or concrete
CLASS zcl_document_processor DEFINITION PUBLIC ABSTRACT CREATE PUBLIC.
  PUBLIC SECTION.
    METHODS:
      process_document
        IMPORTING iv_doc_no TYPE char20
        RAISING   zcx_processing_error,
      get_doc_type ABSTRACT    " subclass MUST implement
        RETURNING VALUE(rv_type) TYPE char4,
      validate                 " subclass CAN override
        IMPORTING iv_doc_no TYPE char20
        RAISING   zcx_processing_error.
  PROTECTED SECTION.
    DATA mv_doc_no TYPE char20.
    METHODS log_event
      IMPORTING iv_msg TYPE string.
ENDCLASS.

CLASS zcl_document_processor IMPLEMENTATION.
  METHOD process_document.
    mv_doc_no = iv_doc_no.
    validate( iv_doc_no ).         " calls overridden version if any
    " ... common processing ...
  ENDMETHOD.

  METHOD validate.
    IF iv_doc_no IS INITIAL.
      RAISE EXCEPTION TYPE zcx_processing_error
        EXPORTING iv_msg = 'Document number required'.
    ENDIF.
  ENDMETHOD.

  METHOD log_event.
    " Write to application log
  ENDMETHOD.
ENDCLASS.
```

## Subclass — REDEFINITION and SUPER

```abap
CLASS zcl_delivery_processor DEFINITION PUBLIC FINAL
  INHERITING FROM zcl_document_processor
  CREATE PUBLIC.

  PUBLIC SECTION.
    " Implement mandatory abstract method:
    METHODS get_doc_type REDEFINITION.
    " Override optional method:
    METHODS validate     REDEFINITION.
    " Add new methods:
    METHODS post_goods_issue
      RAISING zcx_delivery_error.
  PRIVATE SECTION.
    DATA mv_lgnum TYPE /scwm/lgnum.
ENDCLASS.

CLASS zcl_delivery_processor IMPLEMENTATION.

  METHOD get_doc_type.
    rv_type = 'DELI'.
  ENDMETHOD.

  METHOD validate.
    " Call parent implementation first:
    super->validate( iv_doc_no ).

    " Add delivery-specific validation:
    IF strlen( iv_doc_no ) > 10.
      RAISE EXCEPTION TYPE zcx_processing_error
        EXPORTING iv_msg = 'Delivery number too long'.
    ENDIF.

    " Access inherited protected data:
    log_event( |Validated delivery { iv_doc_no }| ).
  ENDMETHOD.

  METHOD post_goods_issue.
    " ... EWM GI logic ...
  ENDMETHOD.

ENDCLASS.
```

## Polymorphism — Calling via Base Reference

```abap
" Table of base references — holds subclass instances:
DATA lt_processors TYPE TABLE OF REF TO zcl_document_processor.

APPEND NEW zcl_delivery_processor( ) TO lt_processors.
APPEND NEW zcl_order_processor( )    TO lt_processors.
APPEND NEW zcl_invoice_processor( )  TO lt_processors.

" Call via polymorphic dispatch:
LOOP AT lt_processors ASSIGNING FIELD-SYMBOL(<proc>).
  " Correct implementation called based on actual type:
  DATA(lv_type) = <proc>->get_doc_type( ).
  <proc>->process_document( '0180000001' ).
ENDLOOP.
```

## Casting — CAST, IS INSTANCE OF, ?=

```abap
DATA lo_base   TYPE REF TO zcl_document_processor.
DATA lo_deliv  TYPE REF TO zcl_delivery_processor.
DATA lo_order  TYPE REF TO zcl_order_processor.

lo_base = NEW zcl_delivery_processor( ).

" IS INSTANCE OF — check type before downcast:
IF lo_base IS INSTANCE OF zcl_delivery_processor.
  lo_deliv = CAST zcl_delivery_processor( lo_base ).
  lo_deliv->post_goods_issue( ).
ENDIF.

" CAST with # (type inferred from DATA declaration):
lo_deliv = CAST #( lo_base ).   " raises cx_sy_move_cast_error if wrong type

" Inline safe downcast pattern:
TRY.
    lo_deliv = CAST zcl_delivery_processor( lo_base ).
  CATCH cx_sy_move_cast_error INTO DATA(lo_ex).
    " handle wrong type
ENDTRY.

" Old-style move casting (still valid):
lo_deliv ?= lo_base.   " equivalent to CAST — raises cx_sy_move_cast_error

" Upcast (always safe — no CAST needed):
lo_base = lo_deliv.    " implicit upcast
```

## Abstract Class

```abap
" Cannot be instantiated — only subclasses can:
CLASS zcl_abstract_validator DEFINITION PUBLIC ABSTRACT CREATE PUBLIC.
  PUBLIC SECTION.
    " Concrete method — shared logic:
    METHODS validate_all
      IMPORTING it_items TYPE zewm_item_t
      RAISING   zcx_validation_error.
    " Abstract — each subclass provides its own implementation:
    METHODS validate_item ABSTRACT
      IMPORTING is_item TYPE zewm_item
      RAISING   zcx_validation_error.
ENDCLASS.

CLASS zcl_abstract_validator IMPLEMENTATION.
  METHOD validate_all.
    LOOP AT it_items ASSIGNING FIELD-SYMBOL(<item>).
      validate_item( <item> ).   " polymorphic — calls subclass method
    ENDLOOP.
  ENDMETHOD.
ENDCLASS.

" Concrete subclass:
CLASS zcl_weight_validator DEFINITION PUBLIC FINAL
  INHERITING FROM zcl_abstract_validator CREATE PUBLIC.
  PUBLIC SECTION.
    METHODS validate_item REDEFINITION.
ENDCLASS.

CLASS zcl_weight_validator IMPLEMENTATION.
  METHOD validate_item.
    IF is_item-weight > 1000.
      RAISE EXCEPTION TYPE zcx_validation_error
        EXPORTING iv_msg = |Item { is_item-material_no } exceeds weight limit|.
    ENDIF.
  ENDMETHOD.
ENDCLASS.
```

## Interface Inheritance

```abap
" Interfaces can inherit from other interfaces:
INTERFACE zif_readable.
  METHODS read RETURNING VALUE(rv_data) TYPE string.
ENDINTERFACE.

INTERFACE zif_writable.
  METHODS write IMPORTING iv_data TYPE string.
ENDINTERFACE.

" Combined interface — inherits both:
INTERFACE zif_read_write.
  INTERFACES:
    zif_readable,
    zif_writable.
  " Can add own methods:
  METHODS flush.
ENDINTERFACE.

" Class implementing combined interface must implement ALL:
CLASS zcl_rw_store DEFINITION PUBLIC IMPLEMENTING zif_read_write.
  PUBLIC SECTION.
    METHODS:
      zif_readable~read  REDEFINITION,
      zif_writable~write REDEFINITION,
      zif_read_write~flush REDEFINITION.
ENDCLASS.
```

## Calling Overridden Method from Same Instance

```abap
" 'me' = self reference; super->method( ) calls parent version
CLASS zcl_child DEFINITION INHERITING FROM zcl_parent.
  PUBLIC SECTION.
    METHODS calculate REDEFINITION.
ENDCLASS.

CLASS zcl_child IMPLEMENTATION.
  METHOD calculate.
    " Get parent result:
    DATA(lv_parent_result) = super->calculate( iv_input ).
    " Extend it:
    rv_result = lv_parent_result * 2.
    " Call another own method:
    me->log_calculation( rv_result ).
  ENDMETHOD.
ENDCLASS.
```
