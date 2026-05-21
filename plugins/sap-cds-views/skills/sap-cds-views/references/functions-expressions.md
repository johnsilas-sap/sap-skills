# CDS Functions and Expressions Reference

## CASE Expression

```abap
" Simple CASE (equality):
case Status
  when 'O' then 'Open'
  when 'P' then 'In Process'
  when 'C' then 'Completed'
  else          'Unknown'
end as StatusText,

" Searched CASE (conditions):
case
  when Quantity > 1000 then 'Heavy'
  when Quantity > 100  then 'Medium'
  else                      'Light'
end as WeightClass,

" CASE for criticality (0=none, 1=warning, 2=error, 3=success):
cast(
  case Status
    when 'O' then 3    " success
    when 'P' then 1    " warning
    when 'C' then 2    " error / blocked
    else 0
  end as abap.int1
) as StatusCriticality,
```

## CAST

```abap
" Cast to ABAP type:
cast( Quantity as abap.dec(13,3) ) as QuantityDec,

cast( DeliveryNo as abap.char(12) ) as DeliveryNoChar,

cast( 1 as abap.int1 ) as ConstantOne,

" Cast within CASE (required when branches return different types):
cast(
  case Status when 'O' then 2 else 0 end
  as abap.int1
) as Criticality,

" Cast date to string for concatenation:
cast( ShipDate as abap.char(8) ) as ShipDateText,
```

## COALESCE and NULLS

```abap
" Return first non-null value:
coalesce( ActualQty, PlannedQty, 0 ) as EffectiveQty,

coalesce( CarrierName, 'Unknown Carrier' ) as DisplayCarrier,

" NULL literal:
cast( null as abap.char(10) ) as EmptyField,

" IS NULL in WHERE:
" (Use path expressions — direct null checks in CDS WHERE are limited)
```

## String Functions

```abap
" Concatenation:
concat( FirstName, concat( ' ', LastName ) ) as FullName,

" In ABAP 7.54+: concat_with_space:
concat_with_space( FirstName, LastName, 1 ) as FullName,

" Substring:
substring( DeliveryNo, 1, 4 ) as DeliveryPrefix,   " chars 1-4

" Length:
length( ShipToName ) as NameLength,

" Upper / Lower:
upper( CountryCode ) as CountryUpper,
lower( Email )       as EmailLower,

" Left / Right (trim):
left(  MaterialNo, 8 ) as MatPrefix,
right( MaterialNo, 4 ) as MatSuffix,

" Replace:
replace( Description, 'OLD', 'NEW' ) as UpdatedDesc,

" LPAD / RPAD (pad with character):
lpad( DeliveryNo, 10, '0' ) as PaddedDelivNo,   " 0000000001
```

## Numeric Functions

```abap
abs( NetValue ) as AbsoluteValue,

ceil(  Quantity ) as RoundedUp,
floor( Quantity ) as RoundedDown,
round( Quantity, 2 ) as RoundedQty,    " 2 decimal places

division( NetValue, 1000, 2 ) as ValueInThousands,   " avoids / by zero if denominator 0
mod( ItemNo, 2 ) as IsEvenItem,
```

## Date and Timestamp Functions

```abap
" Current date / timestamp:
$session.system_date   as TodayDate,      " sy-datum equivalent
$session.user_date     as UserDate,

" Convert date+time → UTC timestamp:
dats_tims_to_tstmp(
  ShipDate,
  ShipTime,
  $session.client,
  'UTC'
) as ShipTimestamp,

" Convert UTC timestamp → date:
tstmp_to_dats(
  CreatedTimestamp,
  abap_system_timezone( $session.client, 'NULL' ),
  $session.client,
  'NULL'
) as CreatedDate,

" Convert UTC timestamp → time:
tstmp_to_tims(
  CreatedTimestamp,
  abap_system_timezone( $session.client, 'NULL' ),
  $session.client,
  'NULL'
) as CreatedTime,

" Add days to date:
dats_add_workingdays( ShipDate, 3, 'NULL' ) as DueDate,

" Difference in days:
dats_days_between( OrderDate, ShipDate ) as LeadTimeDays,

" Current UTC timestamp:
tstmp_current_utctimestamp( ) as NowUtc,
```

## Currency and Unit Conversion

```abap
" Currency conversion (requires currency conversion settings in TCURR):
@Semantics.amount.currencyCode: 'Currency'
currency_conversion(
  amount             => NetValue,
  source_currency    => Currency,
  target_currency    => cast( 'USD' as abap.cuky ),
  exchange_rate_date => $session.system_date,
  exchange_rate_type => 'M',             " 'M' = average, 'B' = buying, 'G' = selling
  error_handling     => 'SET_TO_NULL'    " or 'FAIL_ON_ERROR'
) as NetValueUSD,

" Unit of measure conversion:
@Semantics.quantity.unitOfMeasure: 'TargetUnit'
unit_conversion(
  quantity    => Quantity,
  source_unit => Unit,
  target_unit => cast( 'KG' as abap.unit )
) as QuantityKG,
@Semantics.unitOfMeasure: true
cast( 'KG' as abap.unit ) as TargetUnit,
```

## Aggregation Functions

```abap
" Used in views with GROUP BY (analytical / aggregation views):
define view entity ZI_DeliverySummary
  as select from zewm_delivery
{
  status         as Status,
  count(*)       as DeliveryCount,

  @DefaultAggregation: #SUM
  @Semantics.quantity.unitOfMeasure: 'WeightUOM'
  sum( total_weight ) as TotalWeight,
  'KG'                as WeightUOM,

  max( ship_date ) as LatestShipDate,
  min( ship_date ) as EarliestShipDate,

  count( distinct carrier_id ) as UniqueCarriers
}
group by status
```

## Conditional Aggregation / Counting

```abap
" Count only rows matching a condition:
sum( case when Status = 'O' then 1 else 0 end ) as OpenCount,
sum( case when Status = 'C' then 1 else 0 end ) as ClosedCount,

" Sum only positive quantities:
sum( case when Quantity > 0 then Quantity else 0 end ) as PositiveQtySum,
```

## Session Variables

```abap
$session.user          " sy-uname (current user)
$session.client        " sy-mandt
$session.system_date   " sy-datum
$session.system_language " sy-langu

" Usage:
$session.user   as CreatedBy,
$session.client as Client,
```

## $projection and $self

```abap
" $projection — refers to the current view's projected fields (in ON conditions):
define root view entity ZI_Delivery
  as select from zewm_delivery
  association [0..1] to ZI_Carrier as _Carrier
    on $projection.CarrierId = _Carrier.CarrierId   " $projection.CarrierId = hdr.carrier_id
{
  key hdr.delivery_no as DeliveryNo,
      hdr.carrier_id  as CarrierId,
      _Carrier
}

" $self — in CDS extensions (EXTEND VIEW):
" refer to the extended view itself
```

## Literals and Constants

```abap
" String literal:
'OPEN'          as DefaultStatus,
cast( '' as abap.char(1) ) as EmptyChar,

" Numeric literal:
cast( 0 as abap.int4 ) as Zero,
cast( 100 as abap.dec(5,2) ) as Hundred,

" Date literal:
cast( '99991231' as abap.dats ) as MaxDate,

" Null:
cast( null as abap.char(10) ) as NullField,
```
