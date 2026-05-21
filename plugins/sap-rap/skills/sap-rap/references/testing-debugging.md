# RAP Testing and Debugging Reference

## ABAP Unit Test for RAP (CDS Test Doubles)

```abap
" Test class for ZBP_Delivery behavior
CLASS zcl_test_delivery DEFINITION FINAL
  FOR TESTING RISK LEVEL HARMLESS DURATION SHORT.

  PRIVATE SECTION.
    CLASS-DATA environment TYPE REF TO if_cds_test_environment.

    CLASS-METHODS:
      class_setup,      " create test double environment once
      class_teardown.   " destroy it

    METHODS:
      setup,            " before each test
      teardown,         " after each test
      test_create_delivery FOR TESTING,
      test_post_gi_success FOR TESTING,
      test_post_gi_blocked FOR TESTING,
      test_validate_status FOR TESTING.
ENDCLASS.

CLASS zcl_test_delivery IMPLEMENTATION.

  METHOD class_setup.
    " Create CDS test double for the interface CDS view
    environment = cl_cds_test_environment=>create( i_for_entity = 'ZI_DELIVERY' ).
  ENDMETHOD.

  METHOD class_teardown.
    environment->destroy( ).
  ENDMETHOD.

  METHOD setup.
    " Clear all test doubles before each test
    environment->clear_doubles( ).
  ENDMETHOD.

  METHOD teardown.
    ROLLBACK ENTITIES.   " revert any EML changes
  ENDMETHOD.

  METHOD test_create_delivery.
    " Arrange: nothing to prepare — managed handles insert

    " Act: create via EML
    MODIFY ENTITIES OF zi_delivery
      ENTITY Delivery
        CREATE FIELDS ( ShipToName ShipDate )
        WITH VALUE #( ( %cid       = 'cid1'
                        ShipToName = 'ACME Corp'
                        ShipDate   = '20260601' ) )
      MAPPED   DATA(ls_mapped)
      FAILED   DATA(ls_failed)
      REPORTED DATA(ls_reported).

    " Assert: no errors
    cl_abap_unit_assert=>assert_initial( ls_failed-delivery ).
    cl_abap_unit_assert=>assert_not_initial( ls_mapped-delivery ).

    COMMIT ENTITIES
      RESPONSE OF zi_delivery
      FAILED   DATA(ls_commit_failed)
      REPORTED DATA(ls_commit_reported).

    cl_abap_unit_assert=>assert_initial( ls_commit_failed-delivery ).
  ENDMETHOD.

  METHOD test_post_gi_success.
    " Arrange: inject test double data (skip real DB)
    environment->insert_test_data(
      i_data = VALUE zewm_delivery_t(
        ( delivery_no = '0180000001'  status = 'O'  ship_to_name = 'ACME' )
      )
    ).

    " Act: execute bound action
    MODIFY ENTITIES OF zi_delivery
      ENTITY Delivery
        EXECUTE PostGoodsIssue
        FROM VALUE #( ( DeliveryNo = '0180000001' ) )
      RESULT   DATA(lt_result)
      FAILED   DATA(ls_failed)
      REPORTED DATA(ls_reported).

    " Assert
    cl_abap_unit_assert=>assert_initial(
      act = ls_failed-delivery
      msg = 'PostGoodsIssue should succeed for open delivery'
    ).
    cl_abap_unit_assert=>assert_equals(
      act = lt_result[ 1 ]-%param-Status
      exp = 'C'
    ).
  ENDMETHOD.

  METHOD test_post_gi_blocked.
    " Arrange: delivery already completed
    environment->insert_test_data(
      i_data = VALUE zewm_delivery_t(
        ( delivery_no = '0180000002'  status = 'C'  ship_to_name = 'Beta' )
      )
    ).

    " Act
    MODIFY ENTITIES OF zi_delivery
      ENTITY Delivery
        EXECUTE PostGoodsIssue
        FROM VALUE #( ( DeliveryNo = '0180000002' ) )
      FAILED   DATA(ls_failed)
      REPORTED DATA(ls_reported).

    " Assert: should fail
    cl_abap_unit_assert=>assert_not_initial(
      act = ls_failed-delivery
      msg = 'PostGoodsIssue should fail for completed delivery'
    ).
    cl_abap_unit_assert=>assert_not_initial( ls_reported-delivery ).
  ENDMETHOD.

  METHOD test_validate_status.
    " Arrange: create delivery with invalid status
    MODIFY ENTITIES OF zi_delivery
      ENTITY Delivery
        CREATE FIELDS ( ShipToName Status )
        WITH VALUE #( ( %cid       = 'bad'
                        ShipToName = 'Test'
                        Status     = 'X' ) )   " invalid
      MAPPED DATA(ls_mapped) FAILED DATA(ls_failed) REPORTED DATA(ls_reported).

    COMMIT ENTITIES
      RESPONSE OF zi_delivery
      FAILED DATA(ls_cf) REPORTED DATA(ls_cr).

    " Validation should have fired and reported error
    cl_abap_unit_assert=>assert_not_initial(
      act = ls_cr-delivery
      msg = 'Invalid status should generate error message'
    ).
  ENDMETHOD.

ENDCLASS.
```

## Debugging RAP at Runtime

```
" ADT Debugger — set breakpoints in handler class methods:
1. Set breakpoint in lhc_delivery=>post_goods_issue
2. Run action from Fiori app or ADT Preview
3. Debugger stops — inspect: keys[], failed[], reported[]

" Inspect EML result tables:
DATA(lt_failed)   = ls_failed-delivery.    " table of %tky that errored
DATA(lt_reported) = ls_reported-delivery.  " table with %tky + %msg

" Check %tky (typed key struct):
DATA(lv_key) = ls_failed-delivery[ 1 ]-%tky.
" lv_key-DeliveryNo = key field value

" Check message:
DATA(lo_msg) = ls_reported-delivery[ 1 ]-%msg.
DATA(lv_text) = lo_msg->if_message~get_text( ).
```

## Runtime Analysis Tools

```
" ADT → Window → ABAP Runtime Analysis (SAT equivalent)
" Or: SAT transaction on S/4HANA on-premise
" Trace RAP calls including EML READ/MODIFY/COMMIT

" HTTP Trace for OData V4:
/n/iwfnd/v4_admin → Tools → Gateway Client → enter URL → send request
" Shows: HTTP status, response body, error details

" ABAP Runtime Error (short dump) — also in ADT: Feed → ABAP Runtime Errors

" Read active entity from SQL to verify commit:
SELECT SINGLE * FROM zewm_delivery WHERE delivery_no = '0180000001' INTO @DATA(ls_db).
" Useful in test class to assert DB state after COMMIT ENTITIES
```

## Common RAP Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| `BDEF not found` | CDS view entity name mismatch in BDEF | BDEF `for <ViewName>` must match CDS object name exactly |
| `Draft table missing fields` | Draft table incomplete | Add all draft admin fields + active table key+fields |
| `%cid_ref` key not found | Referencing wrong %cid in same EML block | Match %cid in CREATE to %cid_ref in child CREATE |
| Action button not showing | Feature control returns `fc-o-disabled` | Check `get_instance_features` logic and Status value |
| `COMMIT ENTITIES` rollback | Validation failed | Check `ls_reported` for error messages |
| No data in READ result | CDS view entity inactive or wrong path | Activate CDS, check `IN LOCAL MODE` vs without |
| Lock error on Edit | Another session has draft open | Discard other session's draft or wait for timeout |
| 401 on OData V4 call | Service binding not activated | Click 'Activate' in service binding ADT object |

## EML `IN LOCAL MODE` vs Without

```abap
" IN LOCAL MODE: bypasses authorization checks, uses current transaction buffer
" Use in: handler methods, determinations, validations (you are already inside the BO)
READ ENTITIES OF zi_delivery IN LOCAL MODE
  ENTITY Delivery FIELDS ( Status ) WITH CORRESPONDING #( keys )
RESULT DATA(lt_result).

" Without IN LOCAL MODE: goes through full authorization + independent transaction
" Use in: external callers (reports, batch jobs, other BOs)
READ ENTITIES OF zi_delivery
  ENTITY Delivery FIELDS ( Status ) WITH CORRESPONDING #( keys )
RESULT DATA(lt_result).
```

## Checking Validation and Determination Trigger

```abap
" Validation: triggered automatically on save (COMMIT ENTITIES)
" To verify which validations fire: enable trace in handler class

" Add temporary logging at start of each handler method:
METHOD validate_status.
  cl_demo_output=>display( |validate_status called for { lines( keys ) } keys| ).
  " ... rest of method
ENDMETHOD.

" Check keys table — contains only entities that changed in the relevant fields:
LOOP AT keys ASSIGNING FIELD-SYMBOL(<key>).
  DATA(lv_delivery_no) = <key>-%key-DeliveryNo.
  DATA(lv_create)      = <key>-%is_draft.           " draft or active?
  DATA(lv_op)          = <key>-%op.                 " create/update/delete
ENDLOOP.
```

## Testing Service Binding (ADT OData Client)

```
ADT → Window → Other → OData V4 Client
  Service URL: https://my-s4.ondemand.com/sap/opu/odata4/sap/zsd_delivery/srvd/sap/zsd_delivery/0001/
  → Authenticate (basic or OAuth)
  → Request: GET $metadata
  → Request: GET Deliveries?$top=10
  → Request: POST Deliveries (with JSON body)
  → Request: POST Deliveries('0180000001')/ZI_Delivery.PostGoodsIssue
```
