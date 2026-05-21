# ABAP Unit Test Doubles Reference

## CL_ABAP_TESTDOUBLE — Interface Mocking

```abap
" Prerequisite: class under test depends on an INTERFACE (not a class directly)
" Pattern: inject the interface → test provides a mock implementation

" Interface to mock:
INTERFACE zif_delivery_repository.
  METHODS:
    get_delivery
      IMPORTING iv_delivery_no TYPE /scwm/de_lgnum
      RETURNING VALUE(rs_delivery) TYPE zewm_delivery,
    save_delivery
      IMPORTING is_delivery TYPE zewm_delivery,
    get_open_deliveries
      RETURNING VALUE(rt_deliveries) TYPE zewm_delivery_t.
ENDINTERFACE.

" Class under test (uses interface, not concrete class):
CLASS zcl_delivery_processor DEFINITION.
  PUBLIC SECTION.
    METHODS process
      IMPORTING io_repo       TYPE REF TO zif_delivery_repository
                iv_delivery   TYPE /scwm/de_lgnum
      RETURNING VALUE(rv_status) TYPE char1.
ENDCLASS.
```

```abap
" Test class — create and configure double:
CLASS ltc_processor DEFINITION FOR TESTING RISK LEVEL HARMLESS DURATION SHORT.
  PRIVATE SECTION.
    DATA: go_cut  TYPE REF TO zcl_delivery_processor,
          go_repo TYPE REF TO zif_delivery_repository.  " holds the mock

    METHODS: setup,
             test_open_delivery_completes FOR TESTING,
             test_not_found_raises_error  FOR TESTING.
ENDCLASS.

CLASS ltc_processor IMPLEMENTATION.

  METHOD setup.
    go_cut  = NEW zcl_delivery_processor( ).
    " Create test double for the interface:
    go_repo = CAST zif_delivery_repository(
      cl_abap_testdouble=>create( 'ZIF_DELIVERY_REPOSITORY' )
    ).
  ENDMETHOD.

  METHOD test_open_delivery_completes.
    " ─── Stub: configure what get_delivery returns ────────────────────
    DATA(ls_delivery) = VALUE zewm_delivery(
      delivery_no = '001'  status = 'O'  ship_to_name = 'ACME'
    ).
    cl_abap_testdouble=>configure_call( go_repo )
      ->returning( ls_delivery ).
    go_repo->get_delivery( '001' ).   " register the stub

    " ─── Act ──────────────────────────────────────────────────────────
    DATA(lv_result) = go_cut->process(
      io_repo     = go_repo
      iv_delivery = '001'
    ).

    " ─── Assert ───────────────────────────────────────────────────────
    cl_abap_unit_assert=>assert_equals( act = lv_result  exp = 'C' ).

    " Verify save_delivery was called (interaction testing):
    cl_abap_testdouble=>verify_expectations( go_repo ).
  ENDMETHOD.

  METHOD test_not_found_raises_error.
    " Stub: return empty structure (not found)
    cl_abap_testdouble=>configure_call( go_repo )
      ->returning( VALUE zewm_delivery( ) ).
    go_repo->get_delivery( '999' ).

    TRY.
        go_cut->process( io_repo = go_repo  iv_delivery = '999' ).
        cl_abap_unit_assert=>fail( 'Expected exception' ).
      CATCH zcx_delivery_error.
        " correct
    ENDTRY.
  ENDMETHOD.

ENDCLASS.
```

## Configuring Stubs — All Options

```abap
" Return value
cl_abap_testdouble=>configure_call( go_mock )
  ->returning( ls_data ).
go_mock->get_delivery( '001' ).

" Raise exception
cl_abap_testdouble=>configure_call( go_mock )
  ->raising( NEW zcx_delivery_error( ) ).
go_mock->get_delivery( '999' ).

" Ignore (don't care about call — stub returns initial)
cl_abap_testdouble=>configure_call( go_mock )
  ->ignore( ).
go_mock->save_delivery( VALUE #( ) ).

" Expect exactly N calls
cl_abap_testdouble=>configure_call( go_mock )
  ->times( 2 ).
go_mock->save_delivery( VALUE #( ) ).

" Expect at least once
cl_abap_testdouble=>configure_call( go_mock )
  ->at_least_once( ).
go_mock->save_delivery( VALUE #( ) ).

" Ignore parameter values (stub matches regardless of input)
cl_abap_testdouble=>configure_call( go_mock )
  ->ignore_all_parameters( )
  ->returning( ls_data ).
go_mock->get_delivery( '' ).   " matches any input
```

## Verifying Interactions

```abap
" After act — verify all expected calls occurred:
cl_abap_testdouble=>verify_expectations( go_mock ).

" This fails the test if:
" - An expected call was never made
" - A call was made more times than configured with ->times(N)
" - A call was made that was not configured (strict mode)
```

## CL_CDS_TEST_ENVIRONMENT — CDS View Doubles

```abap
" Use when the class under test reads from a CDS view (SELECT from CDS view entity)

CLASS ltc_delivery_reader DEFINITION FOR TESTING RISK LEVEL HARMLESS DURATION SHORT.
  PRIVATE SECTION.
    CLASS-DATA go_env TYPE REF TO if_cds_test_environment.
    CLASS-METHODS: class_setup, class_teardown.
    METHODS: setup, test_read_open_deliveries FOR TESTING.
ENDCLASS.

CLASS ltc_delivery_reader IMPLEMENTATION.

  METHOD class_setup.
    " Create test double for one or more CDS view entities
    go_env = cl_cds_test_environment=>create(
      i_for_entity = 'ZI_DELIVERY'    " CDS view entity name (uppercase)
    ).
  ENDMETHOD.

  METHOD class_teardown.
    go_env->destroy( ).
  ENDMETHOD.

  METHOD setup.
    go_env->clear_doubles( ).   " reset injected test data before each test
  ENDMETHOD.

  METHOD test_read_open_deliveries.
    " Inject test data into the CDS double:
    go_env->insert_test_data(
      i_data = VALUE zewm_delivery_t(
        ( delivery_no = '001'  status = 'O'  ship_to_name = 'ACME' )
        ( delivery_no = '002'  status = 'O'  ship_to_name = 'Beta' )
        ( delivery_no = '003'  status = 'C'  ship_to_name = 'Gamma' )
      )
    ).

    " Act: run the class that reads from ZI_DELIVERY
    DATA(lo_reader) = NEW zcl_delivery_reader( ).
    DATA(lt_open) = lo_reader->get_open_deliveries( ).

    " Assert: only open ones returned
    cl_abap_unit_assert=>assert_equals(
      act = lines( lt_open )
      exp = 2
      msg = 'Only 2 open deliveries expected'
    ).
  ENDMETHOD.

ENDCLASS.
```

## Multiple CDS Entities in One Environment

```abap
" Create environment covering multiple view entities:
go_env = cl_cds_test_environment=>create_for_multiple_cds(
  i_for_entities = VALUE #(
    ( i_for_entity = 'ZI_DELIVERY' )
    ( i_for_entity = 'ZI_DELIVERY_ITEM' )
    ( i_for_entity = 'ZI_CARRIER' )
  )
).

" Insert into specific entity:
go_env->insert_test_data(
  i_data        = lt_items
  i_entity_name = 'ZI_DELIVERY_ITEM'
).
```

## Manual Test Double Class (No CL_ABAP_TESTDOUBLE)

```abap
" For complex stubs — implement the interface directly in the test include:
CLASS lcl_mock_repo DEFINITION FINAL IMPLEMENTING zif_delivery_repository.
  PUBLIC SECTION.
    DATA mt_deliveries TYPE zewm_delivery_t.   " test data to return
    DATA mv_save_called TYPE abap_bool.

    METHODS:
      get_delivery REDEFINITION,
      save_delivery REDEFINITION,
      get_open_deliveries REDEFINITION.
ENDCLASS.

CLASS lcl_mock_repo IMPLEMENTATION.
  METHOD get_delivery.
    rs_delivery = VALUE #(
      mt_deliveries[ delivery_no = iv_delivery_no ] OPTIONAL
    ).
  ENDMETHOD.

  METHOD save_delivery.
    mv_save_called = abap_true.
  ENDMETHOD.

  METHOD get_open_deliveries.
    rt_deliveries = FILTER #( mt_deliveries USING KEY primary_key
                               WHERE status = 'O' ).
  ENDMETHOD.
ENDCLASS.

" Usage in test:
METHOD test_save_called_on_process.
  DATA(lo_mock) = NEW lcl_mock_repo( ).
  lo_mock->mt_deliveries = VALUE #( ( delivery_no = '001' status = 'O' ) ).

  go_cut->process( io_repo = lo_mock  iv_delivery = '001' ).

  cl_abap_unit_assert=>assert_true(
    act = lo_mock->mv_save_called
    msg = 'save_delivery must be called after processing'
  ).
ENDMETHOD.
```
