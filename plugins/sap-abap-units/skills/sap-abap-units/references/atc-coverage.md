# ATC and Code Coverage Reference

## ATC — ABAP Test Cockpit

```
Transaction: ATC
Purpose: Run static checks + unit tests across packages / transport requests

ATC configuration hierarchy:
  System profile (SCI variant) → Package profile → Object override

Main workflow:
  ATC → Schedule Run → select package/transport → Configure variant → Execute
  → View results → fix findings → re-run
```

## ATC Variant Configuration

```
ATC → Manage → Check Variants
  Create variant: ZEWM_DEFAULT

  Check set contents:
    ✓ ABAP Unit Tests (runs all FOR TESTING classes)
    ✓ Code Inspector checks:
        - Extended program check (SLIN equivalent)
        - Security checks (SQL injection, authority-check)
        - Performance checks (SELECT * in loops)
        - Naming conventions
        - Dead code
    ✓ Custom checks (add via SCI)

  Priority thresholds:
    Errors   (prio 1-2): block transport
    Warnings (prio 3):   require justification
    Info     (prio 4):   informational only

" Assign variant to package:
ATC → Manage → Package Properties
  Package:  ZEWM_FIORI
  Variant:  ZEWM_DEFAULT
```

## Running Tests in ADT

```
" Single class:
Right-click ZCL_TEST_DELIVERY → Run As → ABAP Unit Test
Keyboard: Ctrl+Shift+F10

" Package (all test classes):
Right-click package ZEWM → Run As → ABAP Unit Test

" Results in ADT:
Window → Show View → ABAP Unit (unit test result tree)
  Green = pass
  Red   = fail (shows method, line, assertion message)
  Gray  = skipped

" Coverage view:
After running: Click 'Show Coverage' in ABAP Unit view
  → Opens coverage browser
  → Red lines: not covered
  → Green lines: covered
  → Percentage per method shown
```

## Code Coverage in SE80

```
SE80 → Tools → ABAP Unit → Execute with Coverage
  Select class or package
  → Runs all FOR TESTING classes
  → Opens coverage editor:
      Statement coverage: lines executed / total
      Branch coverage:    IF/CASE branches covered
      Procedure coverage: methods called / total
```

## Coverage Targets

```
" No enforced minimum in standard ABAP, but recommended:
Statement coverage:  ≥ 70% for business logic classes
                     ≥ 80% for utility/helper classes
                     100% for pure calculation functions

" Focus coverage on:
  - All IF/CASE branches
  - Error handling paths (CATCH blocks)
  - Edge cases (empty input, max values, special characters)
  - Each public method

" Don't chase 100% on:
  - Generated boilerplate (constructors that just set attributes)
  - One-liner getters/setters
  - Exception class constructors
```

## ATC in Transport Workflow

```
" Block transport on ATC findings:
SE09/SE10 → Release transport → triggers ATC check
  → If ATC findings with prio ≤ 2: release is blocked
  → Developer must fix or justify findings

" Configure block:
SPRO → SAP NetWeaver → ABAP Development → ABAP Test Cockpit
     → Transport Integration → Configure Check Run on Transport Release

" Exemptions (justified exceptions):
ATC → Manage → Exemptions
  Select finding → Create Exemption
  Reason: 'False positive — field is read-only by design'
  Approver: must be a named role (e.g., ZEWM_ATC_APPROVER)
  Validity: 180 days (must be renewed)
```

## Running ATC Programmatically (CI/CD)

```abap
" Via ABAP function module (for SAP CI/CD integration):
CALL FUNCTION 'ATC_RUN'
  EXPORTING
    iv_run_variant   = 'ZEWM_DEFAULT'
    iv_object_set    = lv_object_set    " transport or package
  IMPORTING
    ev_run_id        = lv_run_id
  EXCEPTIONS
    OTHERS           = 1.

" Check results:
SELECT * FROM satc_ac_result_db
  WHERE run_id = @lv_run_id
  ORDER BY priority
  INTO TABLE @DATA(lt_results).
```

## Code Inspector (SCI) — Custom Check

```
Transaction: SCI

Create inspection:
  SCI → Create Inspection
  Name: ZEWM_QUALITY_CHECK
  Check variant: (select or create)

Add check objects:
  Package: ZEWM_*
  Or: Transport: DE1K900123
  Or: Object list

Configure checks:
  Performance:
    ✓ SELECT inside loop (avoid nested selects)
    ✓ SELECT * (select specific fields)
    ✓ Missing WHERE clause

  Naming:
    ✓ Class names start with ZCL_
    ✓ Interface names start with ZIF_
    ✓ Test class names start with ZCL_TEST_ or LTC_

  Security:
    ✓ Dynamic OPEN SQL (injection risk)
    ✓ AUTHORITY-CHECK missing
    ✓ RFC enabled with no auth check

  Unit Tests:
    ✓ Classes without test include
    ✓ Methods without test coverage (if threshold configured)
```

## Useful ATC / Unit Test Reports

```
" List all unit test results for a package:
Program: SATC_ATC_UI_START (ATC cockpit entry)

" Display coverage analysis:
Program: RS_ABAP_UNIT_TEST_COVERAGE_RPT (if available)
Or: ADT → Coverage Browser after test run

" Schedule nightly ATC run:
ATC → Manage → Schedules
  Variant:   ZEWM_DEFAULT
  Object set: Package ZEWM
  Schedule:  Daily 02:00

" Email results:
ATC → Manage → Schedules → Notification
  Recipient: team@company.com
  Threshold: prio 1 findings only
```

## Common ATC Findings and Fixes

| Finding | Severity | Fix |
|---|---|---|
| SELECT * usage | Warning | List explicit fields |
| SELECT in LOOP | Error | Move SELECT outside loop, use FOR ALL ENTRIES |
| Dynamic OPEN SQL | Error | Use static SQL or parameterize safely |
| AUTHORITY-CHECK missing | Error | Add authority check before DB write |
| Method complexity > 30 | Warning | Refactor into smaller methods |
| Dead code (unreachable) | Warning | Remove dead branches |
| Missing unit tests | Info | Add FOR TESTING class |
| Hard-coded client | Error | Use SY-MANDT |
| COMMIT WORK in method | Warning | Move to caller or use BAPI transaction |
