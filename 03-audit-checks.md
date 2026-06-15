# Underwriting Audit Checks

This is the core of the project. Each check is a calculated column on the `motor`
fact that recomputes an expected value and compares it against what was booked,
returning a plain-language verdict that a reviewer can filter and act on.

The DAX shown below is simplified and genericized. Internal tariff cutover dates and
the insurer's actual rate tables have been abstracted into placeholders, because
those are confidential. The logic and structure are faithful to the real model;
only the proprietary constants are masked. All example values are illustrative.

## 1. Tariff validity

Before any premium can be judged, the model needs to know which tariff regime was in
force when the policy commenced. This column maps the commencement date to a regime
label so every later check rates the policy against the correct rate book.

```dax
tarrif_validity =
VAR dt = motor[comdate]
VAR yr = IF ( NOT ( ISBLANK ( dt ) ), YEAR ( dt ) )
RETURN
SWITCH (
    TRUE (),
    ISBLANK ( dt ), "No Date",
    yr <= REGIME_A_CUTOVER_YEAR, "pre regime A",
    yr <= REGIME_B_CUTOVER_YEAR, "regime A",
    "regime B"
)
```

In the live model the cutover logic is finer-grained (it also tests month and day
around the exact regulatory change dates); those exact dates are masked here.

## 2. No-Claim-Discount integrity

The headline leakage source. The check compares the NCD actually charged against the
expected band from the `NCD` reference, but only for covers that can earn an NCD. A
third-party-only cover that received a discount is flagged immediately.

```dax
05_Check_NCD_Given =
VAR ChargedNCD  = ROUNDDOWN ( motor[004_NCD_Charged_check], 0 )
VAR ExpectedNCD = ROUNDDOWN ( RELATED ( NCD[rates] ), 0 )
RETURN
SWITCH (
    TRUE (),
    motor[cvrtype] IN { "C", "F" } && ExpectedNCD >= ChargedNCD, "CORRECT NCD",
    motor[cvrtype] IN { "C", "F" } && ExpectedNCD <  ChargedNCD, "NCD WRONGLY APPLIED",
    motor[cvrtype] = "T" && ChargedNCD > 0, "NCD ON THIRD-PARTY COVER",
    motor[TP_basic_premium_check] = "RISK CLASSIFICATION ISSUE", "NCD NOT APPLICABLE, RISK CLASS ISSUE",
    "NCD NOT APPLICABLE"
)
```

A companion column, `06_NCD_Validity`, checks that the NCD year progression is
internally consistent across the renewal chain.

## 3. Policy premium accuracy

The model rebuilds an expected total premium from the rated components (third-party
basic, own-damage, perils, loadings, fleet, and contributions) in
`Total_Premium_Check`, then compares it with the booked premium plus the relevant
tariff contribution.

```dax
POLICY PREMIUM CHARGED RIGHT? =
SWITCH (
    TRUE (),
    ( motor[polprem] + RELATED ( TariffTable[motor_contribution] ) ) >= motor[Total_Premium_Check]
        && motor[Total_Premium_Check] > 0, "POLICY TOTAL PREMIUM RIGHTLY CHARGED",
    motor[Total_Premium_Check] = 0, "RISK CLASSIFICATION ISSUE",
    motor[Total_Premium_Check] < 0, "CANCELLATION | REVERSAL",
    "UNDER-CHARGED POLICY PREMIUM"
)
```

This single column turns a tedious manual recalculation into a one-word verdict for
every transaction in the book.

## 4. Commission and overrider correctness

Commission is checked against the expected rate for the channel, with a dedicated
branch for direct business, which should never carry commission.

```dax
Comm. Charged Right? =
SWITCH (
    TRUE (),
    motor[intertypedesc] = "DIRECT" && motor[commission_rate_charged] <> 0,
        "COMMISSION ON DIRECT BUSINESS",
    ROUND ( motor[commission_rate_charged], 0 ) = ROUND ( motor[Commission1_Expected_Rate], 0 ),
        "CORRECTLY CHARGED",
    ROUND ( motor[commission_rate_charged], 0 ) = 0
        && ROUND ( motor[Commission1_Expected_Rate], 0 ) > 0
        && ROUND ( motor[polprem], 0 ) <> 0, "ZERO COMMISSION CHARGED",
    ROUND ( motor[commission_rate_charged], 0 ) < ROUND ( motor[Commission1_Expected_Rate], 0 )
        && ROUND ( motor[polprem], 0 ) <> 0, "UNDER CHARGED",
    ROUND ( motor[commission_rate_charged], 0 ) > ROUND ( motor[Commission1_Expected_Rate], 0 )
        && ROUND ( motor[polprem], 0 ) <> 0, "OVER CHARGED COMMISSION",
    "MISCELLANEOUS"
)
```

An equivalent pattern (`OR_Expected_Rate`, `OverriderComm_rate_charged`) handles the
overrider commission.

## 5. MID reconciliation

Each policy is reconciled against the Motor Insurance Database certificate record on
three axes: premium, sum insured, and cover type. Any divergence is reported with the
size of the gap.

```dax
MIDvsOV3_Prem_Check =
IF (
    motor[Total_Premium_Check] = motor[MIDPrem],
    "PREMIUM MATCH",
    "MIS-MATCH BY DIFF OF " & motor[Premium_Diff]
)
```

`MIDvsOV3_SI_Check` and `MIDvsOV3_CoverType_Check` apply the same idea to sum insured
and cover type, so a reviewer can see at a glance whether what was sold matches what
the regulator's database holds.

## 6. Fleet rating

Using vehicle counts consolidated per insured (`Consolidated_Fleet_by_insuredRed`
joined to the `FLEET` band table), the model decides whether a fleet discount was
appropriate and whether the booked fleet rate matches the expected band.

## 7. Exception surfacing

Beyond the per-policy checks, three exception tables isolate cases that need direct
follow-up:

- `cancelled`: policies cancelled within the period.
- `Expired`: covers that have lapsed.
- `Duplicate Car Numbers MID2`: vehicle registrations appearing more than once, a
  data-integrity and potential-fraud signal.

## How a reviewer uses it

1. Open the audit dashboard and filter to a workgroup, releasing user, or period.
2. Slice by any verdict column to get a worklist, for example every row where
   `POLICY PREMIUM CHARGED RIGHT?` is "UNDER-CHARGED POLICY PREMIUM".
3. Quantify the exposure with the leakage measures.
4. Route the worklist back to underwriting for correction or recovery.

The result is full-coverage assurance instead of a sampled spot check, with the
exposure quantified rather than estimated.
