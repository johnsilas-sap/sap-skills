---
name: sap-abap-units
description: |
  ABAP Unit Testing for SAP ABAP development. Covers test class anatomy (FOR TESTING, RISK LEVEL, DURATION, setup/teardown), CL_ABAP_UNIT_ASSERT assertion methods, test doubles for CDS views and OO interfaces, mocking with CL_ABAP_TESTDOUBLE, test seams for function module injection, RAP EML testing with CDS test environments, ATC (ABAP Test Cockpit) configuration and code coverage, and logistics-specific patterns for testing EWM/TM BAdI implementations and function module calls.

  Key patterns:
  - CLASS-SETUP / setup / teardown lifecycle
  - CL_ABAP_UNIT_ASSERT for all assertion types
  - CL_ABAP_TESTDOUBLE for interface mocking
  - CL_CDS_TEST_ENVIRONMENT for CDS view doubles
  - Test seam + injection for function module mocking
  - Factory pattern for dependency injection
  - EML READ/MODIFY/COMMIT in RAP unit tests
  - ATC variant configuration and coverage targets
license: GPL-3.0
metadata:
  version: "1.0.0"
  last_verified: "2026-05-20"
  abap_version: "S/4HANA 2022+ / ABAP 7.58+"
---

# SAP ABAP Unit Testing

## Test Class Skeleton

```abap
CLASS zcl_test_delivery_processor DEFINITION FINAL
  FOR TESTING
  RISK LEVEL HARMLESS    " HARMLESS / DANGEROUS / CRITICAL
  DURATION   SHORT.      " SHORT (<1s) / MEDIUM (<60s) / LONG

  PRIVATE SECTION.
    " Shared fixture — created once per class
    CLASS-DATA go_cut TYPE REF TO zcl_delivery_processor.  " class under test

    CLASS-METHODS:
      class_setup,      " once before all tests
      class_teardown.   " once after all tests

    METHODS:
      setup,            " before each test method
      teardown,         " after each test method
      test_open_delivery_is_processed  FOR TESTING,
      test_closed_delivery_is_skipped  FOR TESTING,
      test_invalid_delivery_raises_err FOR TESTING.
ENDCLASS.

CLASS zcl_test_delivery_processor IMPLEMENTATION.

  METHOD class_setup.
    go_cut = NEW zcl_delivery_processor( ).
  ENDMETHOD.

  METHOD class_teardown.
    CLEAR go_cut.
  ENDMETHOD.

  METHOD setup.
    " Reset state before each test
  ENDMETHOD.

  METHOD teardown.
    ROLLBACK WORK.
  ENDMETHOD.

  METHOD test_open_delivery_is_processed.
    DATA(lv_result) = go_cut->process( iv_delivery_no = '0180000001' ).
    cl_abap_unit_assert=>assert_equals(
      act = lv_result  exp = 'C'
      msg = 'Open delivery should be set to Completed'
    ).
  ENDMETHOD.

ENDCLASS.
```

## Key Assertion Methods

| Method | Checks |
|---|---|
| `assert_equals( act exp )` | act = exp (deep equal for structures/tables) |
| `assert_not_equals( act exp )` | act ≠ exp |
| `assert_initial( act )` | IS INITIAL |
| `assert_not_initial( act )` | NOT IS INITIAL |
| `assert_true( act )` | = abap_true |
| `assert_false( act )` | = abap_false |
| `assert_bound( act )` | IS BOUND |
| `assert_not_bound( act )` | IS NOT BOUND |
| `assert_char_cp( act exp )` | matches pattern (CP) |
| `assert_differs( act exp )` | act ≠ exp (alias) |
| `fail( msg )` | unconditional failure |
| `assert_subrc( exp )` | sy-subrc = exp (default 0) |

## Run Tests

```
ADT: Right-click class → Run As → ABAP Unit Test
     Or: Ctrl+Shift+F10

SE80: Utilities → ABAP Unit Tests
ATC: Transaction ATC → schedule variant
```
