# ABAP Unit Mocking Patterns Reference

## Test Seam — Function Module Injection

```abap
" TEST-SEAM allows replacing a code block in the CUT at test time.
" Wrap function module calls in a seam:

" In class under test (ZCL_DELIVERY_PROCESSOR):
METHOD post_goods_issue.
  TEST-SEAM call_ewm_gi.
    CALL FUNCTION 'ZEWM_POST_GOODS_ISSUE'
      EXPORTING iv_delivery = iv_delivery_no
      IMPORTING ev_status   = rv_new_status
      EXCEPTIONS not_found  = 1  OTHERS = 2.
    IF sy-subrc <> 0.
      RAISE EXCEPTION TYPE zcx_delivery_error.
    ENDIF.
  END-TEST-SEAM.
ENDMETHOD.

" In test class — inject replacement logic:
METHOD test_gi_posts_successfully.
  " Inject a stub that simulates successful GI:
  TEST-INJECTION call_ewm_gi.
    rv_new_status = 'C'.
    sy-subrc      = 0.
  END-TEST-INJECTION.

  DATA(lv_status) = go_cut->post_goods_issue( '0180000001' ).
  cl_abap_unit_assert=>assert_equals( act = lv_status  exp = 'C' ).
ENDMETHOD.

METHOD test_gi_raises_on_error.
  TEST-INJECTION call_ewm_gi.
    sy-subrc = 1.   " simulate function module error
  END-TEST-INJECTION.

  TRY.
      go_cut->post_goods_issue( '0180000001' ).
      cl_abap_unit_assert=>fail( 'Expected exception' ).
    CATCH zcx_delivery_error.
      " correct
  ENDTRY.
ENDMETHOD.
```

## Factory Pattern — Dependency Injection Without Test Seam

```abap
" Pattern: class under test gets dependencies via a factory/injector
" instead of creating them internally (new / call function)

" Factory interface:
INTERFACE zif_ewm_api_factory.
  METHODS create_gi_poster
    RETURNING VALUE(ro_poster) TYPE REF TO zif_gi_poster.
ENDINTERFACE.

" Class under test — receives factory:
CLASS zcl_delivery_processor DEFINITION.
  PUBLIC SECTION.
    METHODS constructor
      IMPORTING io_factory TYPE REF TO zif_ewm_api_factory.
    METHODS post_gi
      IMPORTING iv_delivery TYPE /scwm/de_lgnum.
  PRIVATE SECTION.
    DATA mo_factory TYPE REF TO zif_ewm_api_factory.
ENDCLASS.

CLASS zcl_delivery_processor IMPLEMENTATION.
  METHOD constructor.
    mo_factory = io_factory.
  ENDMETHOD.

  METHOD post_gi.
    DATA(lo_poster) = mo_factory->create_gi_poster( ).
    lo_poster->post( iv_delivery ).
  ENDMETHOD.
ENDCLASS.

" Test: inject mock factory:
METHOD setup.
  DATA(lo_mock_factory) = CAST zif_ewm_api_factory(
    cl_abap_testdouble=>create( 'ZIF_EWM_API_FACTORY' )
  ).
  DATA(lo_mock_poster) = CAST zif_gi_poster(
    cl_abap_testdouble=>create( 'ZIF_GI_POSTER' )
  ).

  cl_abap_testdouble=>configure_call( lo_mock_factory )
    ->returning( lo_mock_poster ).
  lo_mock_factory->create_gi_poster( ).   " stub factory call

  cl_abap_testdouble=>configure_call( lo_mock_poster )
    ->at_least_once( ).
  lo_mock_poster->post( '001' ).          " expect poster called

  go_cut = NEW zcl_delivery_processor( lo_mock_factory ).
ENDMETHOD.
```

## Constructor Injection Pattern

```abap
" Simplest injection — pass mock directly to constructor:
CLASS zcl_delivery_processor DEFINITION.
  PUBLIC SECTION.
    METHODS constructor
      IMPORTING io_repo TYPE REF TO zif_delivery_repository OPTIONAL.
  PRIVATE SECTION.
    DATA mo_repo TYPE REF TO zif_delivery_repository.
ENDCLASS.

CLASS zcl_delivery_processor IMPLEMENTATION.
  METHOD constructor.
    " Use injected or create real one (default)
    mo_repo = COALESCE( io_repo, NEW zcl_delivery_db_repo( ) ).
  ENDMETHOD.
ENDCLASS.

" In test: inject mock
go_cut = NEW zcl_delivery_processor( go_mock_repo ).

" In production: use default
DATA(lo_processor) = NEW zcl_delivery_processor( ).
```

## Setter Injection (For Classes You Can't Change Constructor)

```abap
" Add a setter method (friend of test class):
CLASS zcl_delivery_processor DEFINITION.
  PUBLIC SECTION.
    METHODS set_repository
      IMPORTING io_repo TYPE REF TO zif_delivery_repository.
  PRIVATE SECTION.
    FRIENDS ltc_delivery.
    DATA mo_repo TYPE REF TO zif_delivery_repository.
ENDCLASS.

" In test:
go_cut->set_repository( go_mock_repo ).
```

## Mocking Static Calls and Global State

```abap
" For calls to global/static classes that can't be injected,
" wrap them in an instance method on the class under test and use TEST-SEAM:

" In CUT:
METHOD get_current_timestamp.
  TEST-SEAM get_utc_time.
    rv_timestamp = utclong_current( ).
  END-TEST-SEAM.
ENDMETHOD.

" In test:
METHOD test_timestamp_is_used.
  DATA(lv_fixed_ts) = CONV utclong( '2026-06-01 12:00:00' ).
  TEST-INJECTION get_utc_time.
    rv_timestamp = lv_fixed_ts.
  END-TEST-INJECTION.

  DATA(ls_result) = go_cut->create_delivery_with_timestamp( ).
  cl_abap_unit_assert=>assert_equals(
    act = ls_result-created_at
    exp = lv_fixed_ts
  ).
ENDMETHOD.
```

## Spy Pattern — Capture Method Arguments

```abap
" Manual mock that records what it was called with:
CLASS lcl_spy_repo DEFINITION IMPLEMENTING zif_delivery_repository.
  PUBLIC SECTION.
    DATA mv_last_saved_delivery TYPE zewm_delivery.
    DATA mv_save_call_count     TYPE i.

    METHODS: get_delivery REDEFINITION,
             save_delivery REDEFINITION,
             get_open_deliveries REDEFINITION.
ENDCLASS.

CLASS lcl_spy_repo IMPLEMENTATION.
  METHOD save_delivery.
    mv_last_saved_delivery = is_delivery.
    ADD 1 TO mv_save_call_count.
  ENDMETHOD.

  METHOD get_delivery.   " return configured data
    rs_delivery = VALUE #( delivery_no = iv_delivery_no status = 'O' ).
  ENDMETHOD.

  METHOD get_open_deliveries.   " stub
  ENDMETHOD.
ENDCLASS.

" Usage: assert what was saved
METHOD test_correct_data_saved.
  DATA(lo_spy) = NEW lcl_spy_repo( ).
  go_cut->process( io_repo = lo_spy  iv_delivery = '001' ).

  cl_abap_unit_assert=>assert_equals(
    act = lo_spy->mv_last_saved_delivery-status
    exp = 'C'
    msg = 'Saved delivery must have status C'
  ).
  cl_abap_unit_assert=>assert_equals(
    act = lo_spy->mv_save_call_count
    exp = 1
    msg = 'save must be called exactly once'
  ).
ENDMETHOD.
```

## Mocking System Fields (sy-uname, sy-datum)

```abap
" Wrap system field reads in overrideable methods:
INTERFACE zif_system_context.
  METHODS:
    get_user    RETURNING VALUE(rv_user)    TYPE sy-uname,
    get_date    RETURNING VALUE(rv_date)    TYPE sy-datum,
    get_time    RETURNING VALUE(rv_time)    TYPE sy-uzeit.
ENDINTERFACE.

" Real implementation:
CLASS zcl_system_context DEFINITION IMPLEMENTING zif_system_context.
  PUBLIC SECTION.
    METHODS: get_user REDEFINITION, get_date REDEFINITION, get_time REDEFINITION.
ENDCLASS.
CLASS zcl_system_context IMPLEMENTATION.
  METHOD get_user.  rv_user = sy-uname.  ENDMETHOD.
  METHOD get_date.  rv_date = sy-datum.  ENDMETHOD.
  METHOD get_time.  rv_time = sy-uzeit.  ENDMETHOD.
ENDCLASS.

" Test stub returns fixed values:
CLASS lcl_mock_sys DEFINITION IMPLEMENTING zif_system_context.
  PUBLIC SECTION.
    METHODS: get_user REDEFINITION, get_date REDEFINITION, get_time REDEFINITION.
ENDCLASS.
CLASS lcl_mock_sys IMPLEMENTATION.
  METHOD get_user.  rv_user = 'TESTUSER'.        ENDMETHOD.
  METHOD get_date.  rv_date = '20260601'.         ENDMETHOD.
  METHOD get_time.  rv_time = '120000'.           ENDMETHOD.
ENDCLASS.
```
