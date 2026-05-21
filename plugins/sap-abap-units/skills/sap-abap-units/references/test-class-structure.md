# ABAP Unit Test Class Structure Reference

## Risk Level and Duration

```abap
" RISK LEVEL controls what the test is allowed to do:
RISK LEVEL HARMLESS    " No DB changes, no RFC calls — safe in all environments
RISK LEVEL DANGEROUS   " May change data but rolls back — use ROLLBACK WORK in teardown
RISK LEVEL CRITICAL    " May change data permanently — production run at own risk

" DURATION sets expected execution time (framework warns if exceeded):
DURATION SHORT         " < 1 second
DURATION MEDIUM        " < 60 seconds
DURATION LONG          " no limit — mark deliberately (slow tests block CI)
```

## Full Class Lifecycle

```abap
CLASS zcl_test_delivery DEFINITION FINAL
  FOR TESTING RISK LEVEL HARMLESS DURATION SHORT.

  PRIVATE SECTION.
    " ── Shared (class-level) state — created once ────────────────────
    CLASS-DATA:
      go_cut        TYPE REF TO zcl_delivery_processor,   " class under test
      go_mock_db    TYPE REF TO zif_delivery_repository.  " shared mock

    " ── Per-test state ────────────────────────────────────────────────
    DATA:
      go_test_double TYPE REF TO cl_abap_testdouble.

    " ── Lifecycle ─────────────────────────────────────────────────────
    CLASS-METHODS:
      class_setup,      " runs ONCE before first test in class
      class_teardown.   " runs ONCE after last test in class

    METHODS:
      setup,            " runs before EACH test method
      teardown,         " runs after EACH test method

      " ── Test methods — alphabetical, named for scenario ────────────
      test_open_delivery_processes    FOR TESTING,
      test_completed_delivery_skips   FOR TESTING,
      test_null_delivery_raises_error FOR TESTING,
      test_weight_calculation_correct FOR TESTING.
ENDCLASS.

CLASS zcl_test_delivery IMPLEMENTATION.

  " ── Class-level setup: expensive fixtures created once ─────────────
  METHOD class_setup.
    " Create class under test
    go_cut = NEW zcl_delivery_processor( ).

    " Prepare a shared mock (stateless — safe to reuse)
    go_mock_db = CAST zif_delivery_repository(
      cl_abap_testdouble=>create( 'ZIF_DELIVERY_REPOSITORY' )
    ).
  ENDMETHOD.

  METHOD class_teardown.
    CLEAR: go_cut, go_mock_db.
  ENDMETHOD.

  " ── Per-test setup: reset mutable state ────────────────────────────
  METHOD setup.
    " Reset the double's recorded calls and configured answers
    cl_abap_testdouble=>configure_call( go_mock_db ).
    " (Fresh stub config done inside each test)
  ENDMETHOD.

  METHOD teardown.
    ROLLBACK WORK.   " undo any DB changes (even if HARMLESS, good practice)
  ENDMETHOD.

  " ── Test methods ───────────────────────────────────────────────────
  METHOD test_open_delivery_processes.
    " Arrange
    DATA(ls_delivery) = VALUE zewm_delivery( delivery_no = '001' status = 'O' ).
    cl_abap_testdouble=>configure_call( go_mock_db )
      ->returning( ls_delivery ).
    go_mock_db->get_delivery( '001' ).   " stub the call

    " Act
    DATA(lv_new_status) = go_cut->process_delivery(
      io_repo       = go_mock_db
      iv_delivery   = '001'
    ).

    " Assert
    cl_abap_unit_assert=>assert_equals(
      act = lv_new_status
      exp = 'C'
      msg = 'Open delivery must reach status C after processing'
    ).
  ENDMETHOD.

  METHOD test_null_delivery_raises_error.
    " Assert exception is raised
    TRY.
        go_cut->process_delivery( io_repo = go_mock_db iv_delivery = '' ).
        cl_abap_unit_assert=>fail( 'Expected exception was not raised' ).
      CATCH zcx_delivery_error INTO DATA(lo_ex).
        cl_abap_unit_assert=>assert_char_cp(
          act = lo_ex->get_text( )
          exp = '*Delivery number*'
        ).
    ENDTRY.
  ENDMETHOD.

ENDCLASS.
```

## One Class, Multiple Test Classes (Local Test Include)

```abap
" In ABAP, test classes live in a dedicated include: CCIMP or the test include.
" In ADT: class → Test Classes tab → write test classes there.
" Multiple independent test classes can coexist in one test include:

" ─── Test class 1: validation logic ──────────────────────────────────
CLASS ltc_validation DEFINITION FINAL FOR TESTING
  RISK LEVEL HARMLESS DURATION SHORT.
  PRIVATE SECTION.
    METHODS: test_valid_status FOR TESTING,
             test_invalid_status FOR TESTING.
ENDCLASS.
CLASS ltc_validation IMPLEMENTATION.
  METHOD test_valid_status.
    " ...
  ENDMETHOD.
  METHOD test_invalid_status.
    " ...
  ENDMETHOD.
ENDCLASS.

" ─── Test class 2: weight calculation ────────────────────────────────
CLASS ltc_weight DEFINITION FINAL FOR TESTING
  RISK LEVEL HARMLESS DURATION SHORT.
  PRIVATE SECTION.
    METHODS: test_single_item FOR TESTING,
             test_multi_item  FOR TESTING.
ENDCLASS.
CLASS ltc_weight IMPLEMENTATION.
  METHOD test_single_item.
    " ...
  ENDMETHOD.
  METHOD test_multi_item.
    " ...
  ENDMETHOD.
ENDCLASS.
```

## Testing Private Methods via Friend Declaration

```abap
" In class under test (ZCL_DELIVERY_PROCESSOR):
CLASS zcl_delivery_processor DEFINITION PUBLIC.
  PUBLIC SECTION.
    METHODS process_delivery IMPORTING iv_delivery TYPE /scwm/de_lgnum.
  PRIVATE SECTION.
    FRIENDS zcl_test_delivery.   " grants test class access to private members
    METHODS calculate_weight RETURNING VALUE(rv_weight) TYPE p.
ENDCLASS.

" Now zcl_test_delivery can call go_cut->calculate_weight( ) in tests.
```

## Test Fixture Patterns

```abap
" ─── Method to build test data (avoid duplicating in every test) ─────
CLASS zcl_test_delivery DEFINITION ...
  PRIVATE SECTION.
    METHODS:
      make_delivery
        IMPORTING iv_status TYPE char1 DEFAULT 'O'
        RETURNING VALUE(rs_delivery) TYPE zewm_delivery.

METHOD make_delivery.
  rs_delivery = VALUE #(
    delivery_no = '0180000001'
    status      = iv_status
    ship_to_name = 'ACME Corp'
    ship_date    = '20260601'
  ).
ENDMETHOD.

" Usage in test:
METHOD test_open_delivery_processes.
  DATA(ls_delivery) = make_delivery( iv_status = 'O' ).
  " ...
ENDMETHOD.
```

## Naming Conventions

```
Test class name:   ZCL_TEST_<ClassName>    (global)
                   LTC_<Topic>             (local in test include)
Test method name:  Describe the scenario — what and expected outcome:
  test_open_delivery_is_processed
  test_empty_key_raises_cx_delivery_error
  test_weight_with_multiple_items_sums_correctly
  test_gi_post_fails_when_status_completed

" Avoid:
  test_1, test_a, test_method   " meaningless
  test_delivery                 " too vague
```
