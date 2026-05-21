# ABAP Unit Assertions Reference

## CL_ABAP_UNIT_ASSERT — Full Method Reference

### Equality

```abap
" Scalar and structure equality (deep comparison)
cl_abap_unit_assert=>assert_equals(
  act = lv_actual
  exp = lv_expected
  msg = 'Optional failure message shown in test results'
  level = cl_abap_unit_assert=>severity-critical   " default: severity-medium
).

" Inequality
cl_abap_unit_assert=>assert_not_equals(
  act = lv_actual
  exp = lv_not_expected
).

" Table equality (entire internal table, row by row)
cl_abap_unit_assert=>assert_equals(
  act = lt_actual_items
  exp = lt_expected_items
  msg = 'Item table mismatch'
).
```

### Emptiness / Initialness

```abap
" Initial = empty string, 0, empty table, null ref, etc.
cl_abap_unit_assert=>assert_initial(
  act = lt_failed
  msg = 'Failed table should be empty'
).

cl_abap_unit_assert=>assert_not_initial(
  act = lv_delivery_no
  msg = 'Delivery number must be set'
).
```

### Boolean

```abap
cl_abap_unit_assert=>assert_true(
  act = lv_is_valid      " must be abap_true ('X')
  msg = 'Delivery should be valid'
).

cl_abap_unit_assert=>assert_false(
  act = lv_is_blocked
  msg = 'Delivery must not be blocked'
).
```

### Reference Binding

```abap
cl_abap_unit_assert=>assert_bound(
  act = lo_processor
  msg = 'Processor object must be instantiated'
).

cl_abap_unit_assert=>assert_not_bound(
  act = lo_optional_service
  msg = 'Optional service should not be set'
).
```

### Pattern Matching

```abap
" CP (contains pattern) — wildcards: * (any chars), + (one char)
cl_abap_unit_assert=>assert_char_cp(
  act = lo_exception->get_text( )
  exp = '*Delivery 001 not found*'
  msg = 'Exception text must contain delivery number'
).

" NP (not matches pattern)
cl_abap_unit_assert=>assert_char_np(
  act = lv_output_text
  exp = '*error*'
  msg = 'Output should not contain error'
).
```

### Numeric Comparison

```abap
" No dedicated numeric comparisons — use assert_equals or assert_true:
cl_abap_unit_assert=>assert_true(
  act = xsdbool( lv_quantity > 0 )
  msg = 'Quantity must be positive'
).

cl_abap_unit_assert=>assert_true(
  act = xsdbool( lv_weight BETWEEN 0 AND 999 )
  msg = 'Weight out of expected range'
).
```

### sy-subrc

```abap
" After a statement that sets sy-subrc:
SELECT SINGLE * FROM zewm_delivery INTO @DATA(ls_db).
cl_abap_unit_assert=>assert_subrc(
  exp = 0
  msg = 'Delivery record must exist in DB'
).

" Or check non-zero:
cl_abap_unit_assert=>assert_subrc(
  exp = 4
  msg = 'Expected record NOT FOUND'
).
```

### Unconditional Failure

```abap
" Use in code paths that should never be reached:
METHOD test_exception_is_raised.
  TRY.
      go_cut->risky_method( ).
      cl_abap_unit_assert=>fail(
        msg = 'Expected ZCX_DELIVERY_ERROR was not raised'
      ).
    CATCH zcx_delivery_error.
      " Expected — test passes
  ENDTRY.
ENDMETHOD.
```

## Severity Levels

```abap
" After a CRITICAL failure, remaining assertions in the method are skipped.
" MEDIUM (default) — logs failure and continues.
" LOW — logs as informational, method continues.

cl_abap_unit_assert=>assert_bound(
  act   = lo_required_object
  msg   = 'Object must be set before proceeding'
  level = cl_abap_unit_assert=>severity-critical   " abort test if unbound
).

cl_abap_unit_assert=>assert_equals(
  act   = lv_optional_field
  exp   = 'expected'
  level = cl_abap_unit_assert=>severity-low        " soft check
).
```

## Asserting Structures Field-by-Field

```abap
" Instead of comparing entire structure (which gives unhelpful diff on failure),
" assert the fields that matter:
METHOD test_delivery_created_correctly.
  DATA(ls_result) = go_cut->create_delivery( VALUE #(
    ship_to_name = 'ACME'
    ship_date    = '20260601'
  ) ).

  cl_abap_unit_assert=>assert_equals( act = ls_result-status      exp = 'O' ).
  cl_abap_unit_assert=>assert_equals( act = ls_result-ship_to_name exp = 'ACME' ).
  cl_abap_unit_assert=>assert_not_initial( act = ls_result-delivery_no ).
ENDMETHOD.
```

## Asserting Tables

```abap
" Check count
cl_abap_unit_assert=>assert_equals(
  act = lines( lt_items )
  exp = 3
  msg = 'Expected 3 items'
).

" Check content of specific row
cl_abap_unit_assert=>assert_equals(
  act = lt_items[ material_no = 'MAT001' ]-quantity
  exp = 10
).

" Check table is not empty
cl_abap_unit_assert=>assert_not_initial( act = lt_items ).

" Check no error messages in reported table
cl_abap_unit_assert=>assert_initial(
  act = lt_reported
  msg = 'No error messages expected'
).
```

## Custom Assertion Helper

```abap
" Extract repeated assertion patterns into a helper method:
METHODS assert_delivery_open
  IMPORTING is_delivery TYPE zewm_delivery.

METHOD assert_delivery_open.
  cl_abap_unit_assert=>assert_equals(
    act = is_delivery-status
    exp = 'O'
    msg = |Delivery { is_delivery-delivery_no } should be Open|
  ).
  cl_abap_unit_assert=>assert_not_initial(
    act = is_delivery-delivery_no
    msg = 'Delivery number must be set'
  ).
ENDMETHOD.

" Usage:
METHOD test_new_delivery_is_open.
  DATA(ls_delivery) = go_cut->create_new_delivery( ).
  assert_delivery_open( ls_delivery ).
ENDMETHOD.
```
