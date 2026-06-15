# DAX Measures

The reporting layer sits on a focused set of measures, grouped into display folders.
They all target the `NCD_Violations` fact and drive the NCD leakage views. The audit
logic for the wider book lives in calculated columns on `motor`; see
[03-audit-checks.md](03-audit-checks.md).

Any numbers produced by these measures in this repository are illustrative samples.

## Folder: NCD Audit

**Violations Count** — total number of NCD violation policies.

```dax
Violations Count = COUNTROWS ( NCD_Violations )
```

**Total NCD Given** — sum of discounts granted. Negative totals represent leakage.

```dax
Total NCD Given = SUM ( NCD_Violations[NCD Given] )
```

**Total Affected Premium** — total premium sitting on violation policies.

```dax
Total Affected Premium = SUM ( NCD_Violations[Policy Premium] )
```

**Avg NCD per Violation** — average discount per flagged policy.

```dax
Avg NCD per Violation =
DIVIDE (
    SUM ( NCD_Violations[NCD Given] ),
    COUNTROWS ( NCD_Violations )
)
```

**NCD Leakage % of Premium** — absolute discount given as a share of affected
premium. This is the headline leakage ratio.

```dax
NCD Leakage % of Premium =
DIVIDE (
    ABS ( SUM ( NCD_Violations[NCD Given] ) ),
    SUM ( NCD_Violations[Policy Premium] )
)
```

**Largest Single NCD** — the biggest individual discount, surfaced as a positive
number for readability.

```dax
Largest Single NCD = ABS ( MIN ( NCD_Violations[NCD Given] ) )
```

## Folder: NCD Audit - By Cover

These split violations and leakage by the previous cover type, because a third-party
cover does not earn an NCD at renewal, making those grants the clearest leakage
signal.

```dax
Violations Prev Cover T =
CALCULATE (
    COUNTROWS ( NCD_Violations ),
    NCD_Violations[Prev Cover] = "T"
)

NCD Leakage Prev Cover T =
CALCULATE (
    SUM ( NCD_Violations[NCD Given] ),
    NCD_Violations[Prev Cover] = "T"
)

Violations Prev Cover 3 =
CALCULATE (
    COUNTROWS ( NCD_Violations ),
    NCD_Violations[Prev Cover] = "3"
)

NCD Leakage Prev Cover 3 =
CALCULATE (
    SUM ( NCD_Violations[NCD Given] ),
    NCD_Violations[Prev Cover] = "3"
)
```

## Folder: NCD Audit - Current Year

Period-scoped versions for the current-year KPI cards.

```dax
Violations Current Year =
CALCULATE (
    COUNTROWS ( NCD_Violations ),
    NCD_Violations[Report Year] = YEAR ( TODAY () )
)

NCD Leakage Current Year =
CALCULATE (
    SUM ( NCD_Violations[NCD Given] ),
    NCD_Violations[Report Year] = YEAR ( TODAY () )
)
```

## Design notes

- Leakage is stored as a signed value (`NCD Given`), so the model can keep the sign
  for arithmetic and present an absolute value only where a positive number reads
  more naturally, as in `Largest Single NCD` and the leakage ratio.
- `DIVIDE` is used throughout to avoid divide-by-zero errors on empty filter
  contexts.
- Current-year measures key off a `Report Year` column rather than a live date
  relationship, which keeps them stable when the audit is rerun against historical
  extracts.
