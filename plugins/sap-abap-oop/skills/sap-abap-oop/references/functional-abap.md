# Modern and Functional ABAP Reference

## VALUE Constructor

```abap
" Build structure inline:
DATA(ls_item) = VALUE zewm_item(
  delivery_no = '0180000001'
  item_no     = '000010'
  material_no = 'MAT001'
  quantity    = 10
  unit        = 'PC'
).

" Build table inline:
DATA(lt_items) = VALUE zewm_item_t(
  ( delivery_no = '001'  item_no = '010'  material_no = 'M1'  quantity = 5  unit = 'PC' )
  ( delivery_no = '001'  item_no = '020'  material_no = 'M2'  quantity = 3  unit = 'KG' )
).

" Partial build with base ( ):
DATA ls_base TYPE zewm_item.
ls_base-delivery_no = '001'.
DATA(ls_derived) = VALUE zewm_item( BASE ls_base  item_no = '010'  material_no = 'M1' ).

" Empty initial value:
DATA(ls_empty) = VALUE zewm_item( ).
```

## CORRESPONDING

```abap
" Map fields with same name between structures:
DATA(ls_out) = CORRESPONDING zewm_output( ls_input ).

" Map with explicit field mapping (MAPPING clause):
DATA(ls_out2) = CORRESPONDING zewm_output( ls_input
  MAPPING output_qty = quantity
          output_uom = unit ).

" Exclude specific fields:
DATA(ls_out3) = CORRESPONDING zewm_output( ls_input
  EXCEPT delivery_no ship_date ).

" Table mapping:
DATA(lt_out) = CORRESPONDING zewm_output_t( lt_input ).

" CORRESPONDING with BASE (keep target fields not in source):
DATA ls_target TYPE zewm_output.
ls_target-extra_field = 'keep me'.
ls_target = CORRESPONDING #( BASE ( ls_target ) ls_input ).
```

## FOR — Table Comprehension

```abap
" Basic: copy all rows
DATA(lt_copy) = VALUE zewm_item_t( FOR ls IN lt_items ( ls ) ).

" Filter: only open items
DATA(lt_open) = VALUE zewm_item_t(
  FOR ls IN lt_items WHERE ( status = 'O' ) ( ls )
).

" Transform: project/map fields
DATA(lt_nos) = VALUE /scwm/de_lgnum_t(
  FOR ls IN lt_items ( ls-delivery_no )
).

" Transform to different structure:
TYPES: BEGIN OF ty_summary,
         material_no TYPE matnr,
         total_qty   TYPE menge_d,
       END OF ty_summary.
DATA(lt_summary) = VALUE TABLE OF ty_summary(
  FOR ls IN lt_items
  ( material_no = ls-material_no
    total_qty   = ls-quantity )
).

" Nested FOR (cross-join):
DATA(lt_pairs) = VALUE string_t(
  FOR la IN lt_a
  FOR lb IN lt_b
  ( |{ la }-{ lb }| )
).
```

## FILTER — Subset with Key Access

```abap
" FILTER requires a sorted table or hashed table with appropriate key
" (more efficient than FOR ... WHERE for large tables)

" By primary key:
DATA(lt_open) = FILTER zewm_item_t( lt_items USING KEY primary_key
                                     WHERE status = 'O' ).

" By secondary key (if defined in table type):
DATA(lt_by_mat) = FILTER zewm_item_t( lt_items USING KEY sk_material
                                        WHERE material_no = 'MAT001' ).

" Exclude matching rows (IN / NOT IN):
DATA(lt_not_open) = FILTER zewm_item_t( lt_items USING KEY primary_key
                                          EXCEPT WHERE status = 'O' ).
```

## COND and SWITCH

```abap
" COND — inline IF/ELSE:
DATA(lv_label) = COND string(
  WHEN ls_item-quantity > 1000 THEN 'Heavy'
  WHEN ls_item-quantity > 100  THEN 'Medium'
  ELSE                              'Light'
).

DATA(lv_icon) = COND string(
  WHEN ls_delivery-status = 'O' THEN 'sap-icon://status-positive'
  WHEN ls_delivery-status = 'C' THEN 'sap-icon://status-completed'
  ELSE 'sap-icon://status-inactive'
).

" SWITCH — inline CASE:
DATA(lv_state) = SWITCH string( ls_delivery-status
  WHEN 'O' THEN 'Information'
  WHEN 'P' THEN 'Warning'
  WHEN 'C' THEN 'Success'
  ELSE          'None'
).

" Combined COND + SWITCH in assignment:
DATA(lv_criticality) = SWITCH abap_int1( ls_delivery-status
  WHEN 'O' THEN 3
  WHEN 'P' THEN 1
  WHEN 'C' THEN 2
  ELSE          0
).
```

## REDUCE

```abap
" Aggregate a table to a single value:
DATA(lv_total_qty) = REDUCE menge_d(
  INIT qty = CONV menge_d( 0 )
  FOR  ls  IN lt_items
  NEXT qty = qty + ls-quantity
).

" Count matching rows (alternative to lines( FILTER(...) )):
DATA(lv_open_count) = REDUCE i(
  INIT cnt = 0
  FOR  ls  IN lt_items WHERE ( status = 'O' )
  NEXT cnt = cnt + 1
).

" Build a string by joining:
DATA(lv_delivery_list) = REDUCE string(
  INIT s = ``
  FOR  ls IN lt_deliveries
  NEXT s = COND #( WHEN s IS INITIAL THEN ls-delivery_no
                   ELSE s && `, ` && ls-delivery_no )
).
```

## String Templates

```abap
" Pipe-delimited interpolation:
DATA(lv_msg) = |Delivery { lv_delivery_no } posted at { lv_time }|.

" Format options:
DATA(lv_num)   = |Qty: { lv_quantity NUMBER = USER }|.     " locale number format
DATA(lv_date)  = |Ship: { lv_date DATE = USER }|.          " locale date format
DATA(lv_time2) = |Time: { lv_time TIME = ISO }|.           " ISO time
DATA(lv_padded)= |{ lv_no WIDTH = 10 ALIGN = RIGHT PAD = '0' }|.   " right-pad with zeros
DATA(lv_upper) = |{ lv_text CASE = UPPER }|.

" Multi-line (use && or CONCATENATION):
DATA(lv_body) = |Delivery: { lv_no }\n| &&
                |Status:   { lv_status }\n| &&
                |Ship To:  { lv_name }|.

" Nested expression:
DATA(lv_info) = |{ COND #( WHEN lv_status = 'O' THEN 'OPEN' ELSE 'CLOSED' ) }|.
```

## NEW Operator and Inline Data

```abap
" Create object inline:
DATA(lo_proc) = NEW zcl_delivery_processor( io_repo = NEW zcl_delivery_db_repo( ) ).

" Inline variable declaration:
DATA(lv_count) = lines( lt_items ).

" Inline CAST:
DATA(lo_specific) = CAST zcl_delivery_processor( lo_generic ).

" Chained method calls (builder/fluent pattern):
DATA(lv_status) =
  NEW zcl_delivery_processor( )
    ->set_warehouse( '1710' )
    ->set_options( is_opt )
    ->process( '0180000001' ).
```

## XSDBOOL and Inline Conditions

```abap
" Convert condition to abap_bool:
DATA(lv_is_open) = xsdbool( ls_delivery-status = 'O' ).

" Use in structure field:
ls_output-is_heavy = xsdbool( ls_item-quantity > 1000 ).

" Convert sy-subrc to boolean:
DATA(lv_found) = xsdbool( sy-subrc = 0 ).
```

## Table Expressions

```abap
" Access row by key — raises cx_sy_itab_line_not_found if not found:
DATA(ls_item) = lt_items[ delivery_no = '001'  item_no = '010' ].

" Safe access with OPTIONAL (returns initial if not found):
DATA(ls_opt) = VALUE #( lt_items[ delivery_no = '999' ] OPTIONAL ).

" Safe access with DEFAULT:
DATA(ls_def) = VALUE #( lt_items[ delivery_no = '999' ] DEFAULT VALUE zewm_item( status = 'X' ) ).

" Modify a row via field-symbol:
lt_items[ delivery_no = '001' item_no = '010' ]-quantity = 20.

" Check existence:
IF line_exists( lt_items[ delivery_no = '001' ] ).

" Get index of row:
DATA(lv_idx) = line_index( lt_items[ delivery_no = '001' ] ).
```

## CONV — Type Conversion

```abap
" Inline type conversion:
DATA(lv_qty) = CONV menge_d( '10.500' ).

DATA(lv_int) = CONV i( lv_quantity ).

DATA(lv_char) = CONV char10( lv_delivery_no ).

" In expressions:
lo_api->post( CONV /scwm/de_lgnum( lv_delivery_raw ) ).
```

## REF and Dereferencing

```abap
" Reference to existing variable:
DATA lv_qty TYPE menge_d VALUE 10.
DATA(lo_qty_ref) = REF #( lv_qty ).        " DATA-REF to lv_qty

" Dereference:
DATA(lv_copy) = lo_qty_ref->*.             " = 10

" Modify via reference:
lo_qty_ref->* = 20.                         " lv_qty is now 20

" Generic data reference:
DATA lo_data TYPE REF TO data.
lo_data = REF #( ls_delivery ).
ASSIGN lo_data->* TO FIELD-SYMBOL(<dlv>).  " access via FIELD-SYMBOL
```
