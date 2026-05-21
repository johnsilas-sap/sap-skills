# RAP Unit Testing Reference

## CDS Test Environment for RAP

```abap
" RAP tests use EML (Entity Manipulation Language) + CDS test doubles.
" The test environment intercepts SELECT from CDS view entities
" and substitutes injected test data — no real DB read.

CLASS ltc_delivery_rap DEFINITION FINAL
  FOR TESTING RISK LEVEL HARMLESS DURATION SHORT.

  PRIVATE SECTION.
    CLASS-DATA go_env TYPE REF TO if_cds_test_environment.

    CLASS-METHODS:
      class_setup,
      class_teardown.

    METHODS:
      setup,
      teardown,
      test_create_delivery                FOR TESTING,
      test_post_gi_on_open_delivery       FOR TESTING,
      test_post_gi_blocked_on_completed   FOR TESTING,
      test_validate_status_rejects_bad    FOR TESTING.
ENDCLASS.

CLASS ltc_delivery_rap IMPLEMENTATION.

  METHOD class_setup.
    go_env = cl_cds_test_environment=>create( i_for_entity = 'ZI_DELIVERY' ).
  ENDMETHOD.

  METHOD class_teardown.
    go_env->destroy( ).
  ENDMETHOD.

  METHOD setup.
    go_env->clear_doubles( ).
  ENDMETHOD.

  METHOD teardown.
    ROLLBACK ENTITIES.
  ENDMETHOD.

  " ── Test: EML CREATE ──────────────────────────────────────────────────
  METHOD test_create_delivery.
    MODIFY ENTITIES OF zi_delivery
      ENTITY Delivery
        CREATE FIELDS ( ShipToName ShipDate )
        WITH VALUE #( ( %cid       = 'cid1'
                        ShipToName = 'ACME Corp'
                        ShipDate   = '20260601' ) )
      MAPPED   DATA(ls_mapped)
      FAILED   DATA(ls_failed)
      REPORTED DATA(ls_reported).

    " No failures
    cl_abap_unit_assert=>assert_initial(
      act = ls_failed-delivery
      msg = 'Create should not fail'
    ).
    " Mapped contains the new entity reference
    cl_abap_unit_assert=>assert_not_initial( act = ls_mapped-delivery ).

    COMMIT ENTITIES
      RESPONSE OF zi_delivery
      FAILED   DATA(ls_cf)
      REPORTED DATA(ls_cr).

    cl_abap_unit_assert=>assert_initial(
      act = ls_cf-delivery
      msg = 'Commit must succeed'
    ).
  ENDMETHOD.

  " ── Test: Bound Action ────────────────────────────────────────────────
  METHOD test_post_gi_on_open_delivery.
    " Inject test data (action reads it via READ ENTITIES IN LOCAL MODE)
    go_env->insert_test_data(
      i_data = VALUE zewm_delivery_t(
        ( delivery_no = '0180000001'  status = 'O'  ship_to_name = 'ACME' )
      )
    ).

    MODIFY ENTITIES OF zi_delivery
      ENTITY Delivery
        EXECUTE PostGoodsIssue
        FROM VALUE #( ( DeliveryNo = '0180000001' ) )
      RESULT   DATA(lt_result)
      FAILED   DATA(ls_failed)
      REPORTED DATA(ls_reported).

    cl_abap_unit_assert=>assert_initial(
      act = ls_failed-delivery
      msg = 'PostGoodsIssue should succeed for open delivery'
    ).
    cl_abap_unit_assert=>assert_equals(
      act = lt_result[ 1 ]-%param-Status
      exp = 'C'
      msg = 'Status should be C after GI'
    ).
  ENDMETHOD.

  METHOD test_post_gi_blocked_on_completed.
    go_env->insert_test_data(
      i_data = VALUE zewm_delivery_t(
        ( delivery_no = '0180000002'  status = 'C'  ship_to_name = 'Beta' )
      )
    ).

    MODIFY ENTITIES OF zi_delivery
      ENTITY Delivery
        EXECUTE PostGoodsIssue
        FROM VALUE #( ( DeliveryNo = '0180000002' ) )
      FAILED   DATA(ls_failed)
      REPORTED DATA(ls_reported).

    cl_abap_unit_assert=>assert_not_initial(
      act = ls_failed-delivery
      msg = 'PostGoodsIssue must fail for completed delivery'
    ).
    cl_abap_unit_assert=>assert_not_initial(
      act = ls_reported-delivery
      msg = 'Error message must be reported'
    ).
  ENDMETHOD.

  " ── Test: Validation on Save ──────────────────────────────────────────
  METHOD test_validate_status_rejects_bad.
    MODIFY ENTITIES OF zi_delivery
      ENTITY Delivery
        CREATE FIELDS ( ShipToName Status )
        WITH VALUE #( ( %cid = 'bad'  ShipToName = 'Test'  Status = 'X' ) )
      MAPPED DATA(ls_mapped) FAILED DATA(ls_failed) REPORTED DATA(ls_reported).

    COMMIT ENTITIES
      RESPONSE OF zi_delivery
      FAILED   DATA(ls_cf)
      REPORTED DATA(ls_cr).

    " Validation should have fired during Activate/Commit
    cl_abap_unit_assert=>assert_not_initial(
      act = ls_cr-delivery
      msg = 'Validation must report error for invalid status X'
    ).
  ENDMETHOD.

ENDCLASS.
```

## Testing Deep Operations (Header + Items)

```abap
METHOD test_deep_create_with_items.
  MODIFY ENTITIES OF zi_delivery
    ENTITY Delivery
      CREATE FIELDS ( ShipToName ShipDate )
      WITH VALUE #( ( %cid = 'hdr1'  ShipToName = 'ACME'  ShipDate = '20260601' ) )
    ENTITY DeliveryItem
      CREATE BY \_Items
      FIELDS ( MaterialNo Quantity Unit )
      WITH VALUE #(
        ( %cid_ref = 'hdr1'  %cid = 'itm1'
          MaterialNo = 'MAT001'  Quantity = 10  Unit = 'PC' )
        ( %cid_ref = 'hdr1'  %cid = 'itm2'
          MaterialNo = 'MAT002'  Quantity = 5   Unit = 'KG' )
      )
  MAPPED   DATA(ls_mapped)
  FAILED   DATA(ls_failed)
  REPORTED DATA(ls_reported).

  cl_abap_unit_assert=>assert_initial( act = ls_failed-delivery ).
  cl_abap_unit_assert=>assert_initial( act = ls_failed-deliveryitem ).
  cl_abap_unit_assert=>assert_equals(
    act = lines( ls_mapped-deliveryitem )
    exp = 2
  ).
ENDMETHOD.
```

## Testing READ Entities

```abap
METHOD test_read_injected_delivery.
  go_env->insert_test_data(
    i_data = VALUE zewm_delivery_t(
      ( delivery_no = '001'  status = 'O'  ship_to_name = 'ACME'  ship_date = '20260601' )
    )
  ).

  READ ENTITIES OF zi_delivery
    ENTITY Delivery
      FIELDS ( DeliveryNo Status ShipToName )
      WITH VALUE #( ( DeliveryNo = '001' ) )
    RESULT DATA(lt_result)
    FAILED DATA(ls_failed).

  cl_abap_unit_assert=>assert_initial( act = ls_failed-delivery ).
  cl_abap_unit_assert=>assert_equals( act = lines( lt_result )  exp = 1 ).
  cl_abap_unit_assert=>assert_equals(
    act = lt_result[ 1 ]-Status
    exp = 'O'
  ).
ENDMETHOD.
```

## Testing Feature Control

```abap
METHOD test_post_gi_disabled_for_completed.
  go_env->insert_test_data(
    i_data = VALUE zewm_delivery_t(
      ( delivery_no = '001'  status = 'C' )
    )
  ).

  " Read instance features
  READ ENTITIES OF zi_delivery
    ENTITY Delivery
      FROM VALUE #( ( DeliveryNo = '001' ) )
    LINK TO zi_delivery~instance_features
    RESULT DATA(lt_features)
    FAILED DATA(ls_failed).

  cl_abap_unit_assert=>assert_equals(
    act = lt_features[ 1 ]-%action-PostGoodsIssue
    exp = if_abap_behv=>fc-o-disabled
    msg = 'PostGoodsIssue must be disabled for completed delivery'
  ).
ENDMETHOD.
```

## Testing Determination

```abap
METHOD test_set_defaults_on_create.
  MODIFY ENTITIES OF zi_delivery
    ENTITY Delivery
      CREATE FIELDS ( ShipToName )
      WITH VALUE #( ( %cid = 'c1'  ShipToName = 'Test' ) )   " Status intentionally omitted
    MAPPED DATA(ls_mapped) FAILED DATA(ls_failed) REPORTED DATA(ls_reported).

  COMMIT ENTITIES
    RESPONSE OF zi_delivery
    FAILED DATA(ls_cf).

  cl_abap_unit_assert=>assert_initial( act = ls_cf-delivery ).

  " Read back and check determination ran
  READ ENTITIES OF zi_delivery
    ENTITY Delivery
      FIELDS ( Status CreatedBy )
      WITH CORRESPONDING #( ls_mapped-delivery )
    RESULT DATA(lt_result).

  cl_abap_unit_assert=>assert_equals(
    act = lt_result[ 1 ]-Status
    exp = 'O'
    msg = 'Determination must set default status to O'
  ).
  cl_abap_unit_assert=>assert_not_initial(
    act = lt_result[ 1 ]-CreatedBy
    msg = 'CreatedBy must be set by determination'
  ).
ENDMETHOD.
```

## Common Pitfalls

```
" ROLLBACK ENTITIES vs ROLLBACK WORK:
" - ROLLBACK ENTITIES: rolls back EML changes in current txn (RAP-specific)
" - ROLLBACK WORK: rolls back entire DB LUW
" In teardown, use ROLLBACK ENTITIES first, then ROLLBACK WORK if DANGEROUS.

" IN LOCAL MODE in tests:
" - Handler methods use IN LOCAL MODE internally
" - Test EML (outside handler) should NOT use IN LOCAL MODE
" - This ensures authorization checks still run in tests

" CDS test double scope:
" - go_env->clear_doubles( ) resets data but NOT the environment
" - go_env->destroy( ) releases resources — call in class_teardown only
" - Don't call destroy( ) in teardown — it would break subsequent tests
```
