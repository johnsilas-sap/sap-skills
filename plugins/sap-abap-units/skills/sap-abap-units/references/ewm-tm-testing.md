# EWM and TM Unit Testing Patterns

## Testing BAdI Implementations

```abap
" Pattern: test the BAdI implementation class directly
" — no need to trigger the BAdI runtime, just call the interface method

" BAdI implementation class under test: ZCL_EWM_BADI_WHO_CREATION
" Interface method to test: IF_EX_/SCWM/EX_WHO_CREATION~SET_WHO_ATTRIBUTES

CLASS ltc_who_badi DEFINITION FINAL
  FOR TESTING RISK LEVEL HARMLESS DURATION SHORT.

  PRIVATE SECTION.
    DATA go_badi_impl TYPE REF TO zcl_ewm_badi_who_creation.

    METHODS:
      setup,
      test_priority_set_for_express_delivery  FOR TESTING,
      test_resource_assigned_for_warehouse    FOR TESTING,
      test_no_change_for_normal_delivery      FOR TESTING.
ENDCLASS.

CLASS ltc_who_badi IMPLEMENTATION.

  METHOD setup.
    go_badi_impl = NEW zcl_ewm_badi_who_creation( ).
  ENDMETHOD.

  METHOD test_priority_set_for_express_delivery.
    " Arrange: simulate the parameters the framework would pass
    DATA ls_who  TYPE /scwm/s_who_creation_in.
    DATA cs_ltap TYPE /scwm/ltap_vb.    " changing parameter

    ls_who-lgnum   = '1710'.
    ls_who-who_type = 'WHT'.
    ls_who-refnum  = '0180000001'.      " express delivery number

    " Act: call BAdI method directly
    go_badi_impl->if_ex_/scwm/ex_who_creation~set_who_attributes(
      EXPORTING is_who  = ls_who
      CHANGING  cs_ltap = cs_ltap
    ).

    " Assert: priority was set
    cl_abap_unit_assert=>assert_equals(
      act = cs_ltap-priority
      exp = '1'
      msg = 'Express delivery must get priority 1'
    ).
  ENDMETHOD.

  METHOD test_resource_assigned_for_warehouse.
    DATA ls_who  TYPE /scwm/s_who_creation_in.
    DATA cs_ltap TYPE /scwm/ltap_vb.

    ls_who-lgnum  = '1710'.
    ls_who-lgtyp  = 'ZG01'.   " zone requiring forklift

    go_badi_impl->if_ex_/scwm/ex_who_creation~set_who_attributes(
      EXPORTING is_who  = ls_who
      CHANGING  cs_ltap = cs_ltap
    ).

    cl_abap_unit_assert=>assert_equals(
      act = cs_ltap-rsrc_grp
      exp = 'FORK'
      msg = 'Zone ZG01 must get forklift resource group'
    ).
  ENDMETHOD.

ENDCLASS.
```

## Testing EWM Function Module Wrappers

```abap
" Pattern: wrap EWM FMs in a class method, use TEST-SEAM to mock

" Class under test:
CLASS zcl_ewm_gi_poster DEFINITION.
  PUBLIC SECTION.
    METHODS post_goods_issue
      IMPORTING iv_lgnum      TYPE /scwm/lgnum
                iv_delivery   TYPE /scwm/de_lgnum
      RETURNING VALUE(rv_ok)  TYPE abap_bool.
ENDCLASS.

CLASS zcl_ewm_gi_poster IMPLEMENTATION.
  METHOD post_goods_issue.
    TEST-SEAM post_gi_call.
      CALL FUNCTION '/SCWM/GI_POST'
        EXPORTING
          iv_lgnum    = iv_lgnum
          iv_delivery = iv_delivery
        EXCEPTIONS
          not_found = 1  OTHERS = 2.
      rv_ok = xsdbool( sy-subrc = 0 ).
    END-TEST-SEAM.
  ENDMETHOD.
ENDCLASS.

" Test:
CLASS ltc_gi_poster DEFINITION FOR TESTING RISK LEVEL HARMLESS DURATION SHORT.
  PRIVATE SECTION.
    DATA go_cut TYPE REF TO zcl_ewm_gi_poster.
    METHODS: setup,
             test_success FOR TESTING,
             test_failure FOR TESTING.
ENDCLASS.

CLASS ltc_gi_poster IMPLEMENTATION.
  METHOD setup.
    go_cut = NEW zcl_ewm_gi_poster( ).
  ENDMETHOD.

  METHOD test_success.
    TEST-INJECTION post_gi_call.
      sy-subrc = 0.
      rv_ok    = abap_true.
    END-TEST-INJECTION.

    DATA(lv_result) = go_cut->post_goods_issue( iv_lgnum = '1710' iv_delivery = '001' ).
    cl_abap_unit_assert=>assert_true( act = lv_result ).
  ENDMETHOD.

  METHOD test_failure.
    TEST-INJECTION post_gi_call.
      sy-subrc = 1.
      rv_ok    = abap_false.
    END-TEST-INJECTION.

    DATA(lv_result) = go_cut->post_goods_issue( iv_lgnum = '1710' iv_delivery = '001' ).
    cl_abap_unit_assert=>assert_false( act = lv_result ).
  ENDMETHOD.

ENDCLASS.
```

## Testing Warehouse Task Confirmation Logic

```abap
" Test the confirmation decision logic (which tasks to confirm, in what order)
" without actually calling the EWM confirmation API

CLASS ltc_task_confirmer DEFINITION FINAL
  FOR TESTING RISK LEVEL HARMLESS DURATION SHORT.

  PRIVATE SECTION.
    DATA go_cut   TYPE REF TO zcl_wht_confirmer.
    DATA go_mock  TYPE REF TO zif_ewm_api.

    METHODS: setup,
             test_confirms_pick_before_put  FOR TESTING,
             test_skips_already_confirmed   FOR TESTING,
             test_partial_confirm_quantity  FOR TESTING.
ENDCLASS.

CLASS ltc_task_confirmer IMPLEMENTATION.

  METHOD setup.
    go_mock = CAST zif_ewm_api(
      cl_abap_testdouble=>create( 'ZIF_EWM_API' )
    ).
    go_cut = NEW zcl_wht_confirmer( io_api = go_mock ).
  ENDMETHOD.

  METHOD test_confirms_pick_before_put.
    DATA(lt_tasks) = VALUE /scwm/tt_ltap_vb(
      ( tanum = '000001' flgto = 'PICK' priority = '1' )
      ( tanum = '000002' flgto = 'PUT'  priority = '2' )
    ).

    " Expect confirm called for pick task first
    cl_abap_testdouble=>configure_call( go_mock )
      ->times( 1 ).
    go_mock->confirm_task( '000001' ).

    cl_abap_testdouble=>configure_call( go_mock )
      ->times( 1 ).
    go_mock->confirm_task( '000002' ).

    go_cut->confirm_all( lt_tasks ).

    cl_abap_testdouble=>verify_expectations( go_mock ).
  ENDMETHOD.

  METHOD test_partial_confirm_quantity.
    DATA ls_task TYPE /scwm/ltap_vb.
    ls_task-tanum  = '000003'.
    ls_task-vsola  = 10.     " planned quantity
    ls_task-vsolb  = 'PC'.

    " Confirm with 8 (partial)
    DATA(lv_actual_qty) = CONV /scwm/menge( 8 ).
    DATA(lv_result) = go_cut->confirm_partial(
      is_task    = ls_task
      iv_act_qty = lv_actual_qty
    ).

    cl_abap_unit_assert=>assert_true(
      act = lv_result-is_partial
      msg = 'Should be marked as partial confirmation'
    ).
    cl_abap_unit_assert=>assert_equals(
      act = lv_result-remaining_qty
      exp = 2
    ).
  ENDMETHOD.

ENDCLASS.
```

## Testing TM Carrier Selection Logic

```abap
" Test the score/filter logic without the TM optimizer

CLASS ltc_carrier_sel DEFINITION FINAL
  FOR TESTING RISK LEVEL HARMLESS DURATION SHORT.

  PRIVATE SECTION.
    DATA go_cut TYPE REF TO zcl_tm_carrier_scorer.

    METHODS: setup,
             test_preferred_carrier_gets_bonus  FOR TESTING,
             test_blacklisted_carrier_excluded  FOR TESTING,
             test_scores_sorted_descending      FOR TESTING.
ENDCLASS.

CLASS ltc_carrier_sel IMPLEMENTATION.

  METHOD setup.
    go_cut = NEW zcl_tm_carrier_scorer( ).
  ENDMETHOD.

  METHOD test_preferred_carrier_gets_bonus.
    DATA(lv_score) = go_cut->score_carrier(
      iv_carrier_id  = 'DHL'
      iv_route       = 'EU_MAIN'
      iv_is_preferred = abap_true
    ).

    cl_abap_unit_assert=>assert_true(
      act = xsdbool( lv_score > 100 )
      msg = 'Preferred carrier must score above 100'
    ).
  ENDMETHOD.

  METHOD test_blacklisted_carrier_excluded.
    DATA(lt_carriers) = VALUE /scmtms/t_carrier(
      ( carrier_id = 'BAD_CARRIER' )
      ( carrier_id = 'GOOD_CARRIER' )
    ).

    go_cut->filter_blacklisted( CHANGING ct_carriers = lt_carriers ).

    " BAD_CARRIER should be removed
    cl_abap_unit_assert=>assert_equals(
      act = lines( lt_carriers )
      exp = 1
    ).
    cl_abap_unit_assert=>assert_initial(
      act = VALUE #( lt_carriers[ carrier_id = 'BAD_CARRIER' ] OPTIONAL )
      msg = 'Blacklisted carrier must be removed'
    ).
  ENDMETHOD.

ENDCLASS.
```

## Testing PPF Action Classes (Adobe Forms / SmartForms)

```abap
" Test the IF_FPF_ACTION implementation directly

CLASS ltc_ppf_action DEFINITION FINAL
  FOR TESTING RISK LEVEL HARMLESS DURATION SHORT.

  PRIVATE SECTION.
    DATA go_action TYPE REF TO zcl_ewm_delivery_print.
    DATA go_mock_fp TYPE REF TO zif_form_printer.

    METHODS: setup,
             test_print_called_for_open_delivery  FOR TESTING,
             test_no_print_for_completed          FOR TESTING.
ENDCLASS.

CLASS ltc_ppf_action IMPLEMENTATION.

  METHOD setup.
    go_mock_fp = CAST zif_form_printer(
      cl_abap_testdouble=>create( 'ZIF_FORM_PRINTER' )
    ).
    go_action = NEW zcl_ewm_delivery_print( io_printer = go_mock_fp ).
  ENDMETHOD.

  METHOD test_print_called_for_open_delivery.
    " Expect print called exactly once
    cl_abap_testdouble=>configure_call( go_mock_fp )
      ->times( 1 ).
    go_mock_fp->print_document( VALUE #( ) ).

    " Simulate PPF execute call
    go_action->execute(
      ip_application_object = VALUE #( )
      ip_partner_object     = VALUE #( )
      ip_message_object     = VALUE #( )
    ).

    cl_abap_testdouble=>verify_expectations( go_mock_fp ).
  ENDMETHOD.

ENDCLASS.
```

## Testing ABAP Classes That Use EWM APIs (Full Integration Pattern)

```abap
" For RISK LEVEL DANGEROUS tests that need a real EWM system:
" Mark them explicitly and isolate from HARMLESS suite

CLASS ltc_ewm_integration DEFINITION FINAL
  FOR TESTING
  RISK LEVEL DANGEROUS    " May create real WHO / warehouse tasks
  DURATION MEDIUM.

  PRIVATE SECTION.
    METHODS:
      teardown,                                    " always rollback
      test_create_who_in_ewm FOR TESTING.
ENDCLASS.

CLASS ltc_ewm_integration IMPLEMENTATION.
  METHOD teardown.
    ROLLBACK WORK.   " undo any created WHO / tasks
  ENDMETHOD.

  METHOD test_create_who_in_ewm.
    " Only run in EWM test system (guard with environment check)
    IF sy-sysid <> 'EWT'.   " EWT = EWM test system SID
      RETURN.               " skip in other systems
    ENDIF.

    DATA(lo_creator) = NEW zcl_who_creator( ).
    DATA(lv_who_id) = lo_creator->create_warehouse_order(
      iv_lgnum      = '1710'
      iv_delivery   = '0180000001'
    ).

    cl_abap_unit_assert=>assert_not_initial(
      act = lv_who_id
      msg = 'Warehouse order ID must be returned'
    ).
  ENDMETHOD.
ENDCLASS.
```
