# DAX Measures

All measures live in one measures table. Dates are measured from the **last date in the data (11 May 2025)**, not from `TODAY()`.
`DimDate` has one active relationship (Transactions). Measures that analyse another date column use `USERELATIONSHIP`.

## Calculated columns

```dax
-- DimDate
YearMonthNumber = YEAR ( DimDate[Date] ) * 100 + MONTH ( DimDate[Date] )

-- customers
Customer Last Txn = CALCULATE ( MAX ( Transactions[TransactionDate] ) )

-- customers
Customer Status =
VAR D = CALCULATE ( MAX ( Transactions[TransactionDate] ), ALL ( Transactions ) )
VAR L = customers[Customer Last Txn]
RETURN
    SWITCH (
        TRUE (),
        ISBLANK ( CALCULATE ( COUNTROWS ( Accounts ) ) ), "No Account",
        ISBLANK ( L ) || L <= D - 180, "Churn Risk",
        L > D - 90, "Active",
        "Cooling"
    )
```

## Base

```dax
Last Data Date = CALCULATE ( MAX ( Transactions[TransactionDate] ), REMOVEFILTERS ( Transactions ) )
```

## Customers

| Measure | Meaning |
|---|---|
| Total Customers | All customers |
| Customers with Accounts / Without Accounts | Customers holding at least one account / none |
| Active Customers | Customers with a transaction in the 90 days before the last data date |
| Inactive Customers | Customers with no transaction in the 180 days before the last data date (includes customers without accounts) |
| Churn Risk Customers | Inactive customers who do hold an account |
| Repeat Customers | Customers with more than one transaction |
| New Customers | Customers by join date (inactive relationship) |

```dax
Total Customers = DISTINCTCOUNT ( customers[CustomerID] )

Customers with Accounts = DISTINCTCOUNT ( Accounts[CustomerID] )

Customers Without Accounts =
COUNTROWS ( EXCEPT ( VALUES ( customers[CustomerID] ), VALUES ( Accounts[CustomerID] ) ) )

Customers with Multiple Accounts =
COUNTROWS (
    FILTER ( VALUES ( Accounts[CustomerID] ), CALCULATE ( DISTINCTCOUNT ( Accounts[AccountID] ) ) > 1 )
)

Active Customers =
VAR MaxTransactionDate = CALCULATE ( MAX ( Transactions[TransactionDate] ), REMOVEFILTERS ( DimDate ) )
VAR StartDate = MaxTransactionDate - 90
VAR ActiveAccountIDs =
    CALCULATETABLE (
        VALUES ( Transactions[AccountID] ),
        REMOVEFILTERS ( DimDate ),
        Transactions[TransactionDate] > StartDate,
        Transactions[TransactionDate] <= MaxTransactionDate
    )
VAR ActiveCustomerIDs =
    CALCULATETABLE ( VALUES ( Accounts[CustomerID] ), TREATAS ( ActiveAccountIDs, Accounts[AccountID] ) )
RETURN
    COUNTROWS ( ActiveCustomerIDs )

Inactive Customers =
VAR MaxTransactionDate = CALCULATE ( MAX ( Transactions[TransactionDate] ), REMOVEFILTERS ( DimDate ) )
VAR StartDate = MaxTransactionDate - 180
VAR ActiveAccountIDs =
    CALCULATETABLE (
        VALUES ( Transactions[AccountID] ),
        REMOVEFILTERS ( DimDate ),
        Transactions[TransactionDate] > StartDate,
        Transactions[TransactionDate] <= MaxTransactionDate
    )
VAR ActiveCustomerIDs =
    CALCULATETABLE ( VALUES ( Accounts[CustomerID] ), TREATAS ( ActiveAccountIDs, Accounts[AccountID] ) )
RETURN
    [Total Customers] - COUNTROWS ( ActiveCustomerIDs )

Churn Risk Customers = [Inactive Customers] - [Customers Without Accounts]

Repeat Customers =
VAR CustomerTransactions =
    ADDCOLUMNS (
        VALUES ( Accounts[CustomerID] ),
        "Transaction Count", CALCULATE ( DISTINCTCOUNT ( Transactions[TransactionID] ) )
    )
RETURN
    COUNTROWS ( FILTER ( CustomerTransactions, [Transaction Count] > 1 ) )

New Customers =
CALCULATE ( DISTINCTCOUNT ( customers[CustomerID] ), USERELATIONSHIP ( DimDate[Date], customers[JoinDate] ) )

New Customers Previous Month = CALCULATE ( [New Customers], DATEADD ( DimDate[Date], -1, MONTH ) )

Customer Acquisition Growth % =
DIVIDE ( [New Customers] - [New Customers Previous Month], [New Customers Previous Month], 0 )

Average Customer Tenure (Years) =
VAR DataEndDate = CALCULATE ( MAX ( Transactions[TransactionDate] ), REMOVEFILTERS ( Transactions ) )
RETURN
    AVERAGEX (
        FILTER ( customers, NOT ISBLANK ( customers[JoinDate] ) && customers[JoinDate] <= DataEndDate ),
        DIVIDE ( DATEDIFF ( customers[JoinDate], DataEndDate, DAY ), 365.25 )
    )
```

### Customer ratios

```dax
Active Customers % = DIVIDE ( [Active Customers], [Customers with Accounts], 0 )
Churn Risk % = DIVIDE ( [Churn Risk Customers], [Customers with Accounts], 0 )
Repeat Customer Rate % = DIVIDE ( [Repeat Customers], [Customers with Accounts], 0 )
Inactive Customers % = DIVIDE ( [Inactive Customers], [Total Customers], 0 )
Customers Without Accounts % = DIVIDE ( [Customers Without Accounts], [Total Customers], 0 )
Customers With Accounts % = DIVIDE ( [Customers with Accounts], [Total Customers], 0 )
Customers with Loans = DISTINCTCOUNT ( Loans[CustomerID] )
Customer Loan Penetration % = DIVIDE ( [Customers with Loans], [Total Customers], 0 )
Customers with Cards = DISTINCTCOUNT ( Cards[CustomerID] )
Customer Card Penetration % = DIVIDE ( [Customers with Cards], [Total Customers], 0 )
```

## Accounts

```dax
Total Accounts = DISTINCTCOUNT ( Accounts[AccountID] )
Total Account Balance = SUM ( Accounts[Balance] )
Average Account Balance = DIVIDE ( [Total Account Balance], [Total Accounts], 0 )
Average Accounts per Customer = DIVIDE ( [Total Accounts], [Customers with Accounts], 0 )
Zero Balance Accounts = CALCULATE ( [Total Accounts], Accounts[Balance] = 0 )

Dormant Accounts =
VAR D = [Last Data Date]
VAR Recent =
    CALCULATETABLE (
        VALUES ( Transactions[AccountID] ),
        REMOVEFILTERS ( DimDate ),
        Transactions[TransactionDate] > D - 180
    )
RETURN
    COUNTROWS ( EXCEPT ( VALUES ( Accounts[AccountID] ), Recent ) )
```

## Transactions

```dax
Total Transactions = DISTINCTCOUNT ( Transactions[TransactionID] )
Total Transaction Amount = SUM ( Transactions[Amount] )
Average Transaction Amount = DIVIDE ( [Total Transaction Amount], [Total Transactions], 0 )
Total Deposits = CALCULATE ( [Total Transaction Amount], Transactions[TransactionType] = "Deposit" )
Total Withdrawals = CALCULATE ( [Total Transaction Amount], Transactions[TransactionType] = "Withdrawal" )
Total Transfers = CALCULATE ( [Total Transaction Amount], Transactions[TransactionType] = "Transfer" )
Total Payments = CALCULATE ( [Total Transaction Amount], Transactions[TransactionType] = "Payment" )
Net Cash Flow = [Total Deposits] - [Total Withdrawals]
Deposit Share % = DIVIDE ( [Total Deposits], [Total Transaction Amount], 0 )
Withdrawal Share % = DIVIDE ( [Total Withdrawals], [Total Transaction Amount], 0 )
```

## Loans

```dax
Total Loan Amount = SUM ( Loans[LoanAmount] )
Total Loans = DISTINCTCOUNT ( Loans[LoanID] )
Average Loan Amount = DIVIDE ( [Total Loan Amount], [Total Loans], 0 )

Weighted Avg Interest Rate =
DIVIDE ( SUMX ( Loans, Loans[LoanAmount] * Loans[InterestRate] ), [Total Loan Amount], 0 )

Active Loans =
VAR DataEndDate = CALCULATE ( MAX ( Transactions[TransactionDate] ), REMOVEFILTERS ( Transactions ) )
RETURN
    CALCULATE (
        DISTINCTCOUNT ( Loans[LoanID] ),
        FILTER ( Loans, Loans[LoanStartDate] <= DataEndDate && Loans[LoanEndDate] >= DataEndDate )
    )

Active Loan Amount =
VAR DataEndDate = CALCULATE ( MAX ( Transactions[TransactionDate] ), REMOVEFILTERS ( Transactions ) )
RETURN
    CALCULATE (
        SUM ( Loans[LoanAmount] ),
        FILTER ( Loans, Loans[LoanStartDate] <= DataEndDate && Loans[LoanEndDate] >= DataEndDate )
    )

Loans Disbursed = CALCULATE ( [Total Loan Amount], USERELATIONSHIP ( Loans[LoanStartDate], DimDate[Date] ) )

Loans Maturing Next 12M =
VAR D = [Last Data Date]
RETURN
    CALCULATE ( DISTINCTCOUNT ( Loans[LoanID] ), Loans[LoanEndDate] > D, Loans[LoanEndDate] <= EDATE ( D, 12 ) )
```

## Cards

```dax
Total Cards = DISTINCTCOUNT ( Cards[CardID] )
Cards Issued = CALCULATE ( [Total Cards], USERELATIONSHIP ( Cards[IssuedDate], DimDate[Date] ) )

Active Cards =
VAR DataEndDate = CALCULATE ( MAX ( Transactions[TransactionDate] ), REMOVEFILTERS ( Transactions ) )
RETURN
    CALCULATE ( [Total Cards], FILTER ( Cards, Cards[IssuedDate] <= DataEndDate && Cards[ExpirationDate] >= DataEndDate ) )

Expired Cards =
VAR DataEndDate = CALCULATE ( MAX ( Transactions[TransactionDate] ), REMOVEFILTERS ( Transactions ) )
RETURN
    CALCULATE ( [Total Cards], FILTER ( Cards, Cards[ExpirationDate] < DataEndDate ) )

Cards Expiring in Next 30 Days =
VAR DataEndDate = CALCULATE ( MAX ( Transactions[TransactionDate] ), REMOVEFILTERS ( Transactions ) )
RETURN
    CALCULATE (
        [Total Cards],
        FILTER ( Cards, Cards[ExpirationDate] >= DataEndDate && Cards[ExpirationDate] <= DataEndDate + 30 )
    )

Expired Cards % = DIVIDE ( [Expired Cards], [Total Cards], 0 )
```

## Support

```dax
Total Support Calls = DISTINCTCOUNT ( SupportCalls[CallID] )
Resolved Calls = CALCULATE ( DISTINCTCOUNT ( SupportCalls[CallID] ), SupportCalls[Resolved] = "Yes" )
Unresolved Calls = CALCULATE ( DISTINCTCOUNT ( SupportCalls[CallID] ), SupportCalls[Resolved] <> "Yes" )
Resolution Rate % = DIVIDE ( [Resolved Calls], [Total Support Calls], 0 )
Support Calls by Date = CALCULATE ( [Total Support Calls], USERELATIONSHIP ( SupportCalls[CallDate], DimDate[Date] ) )

Repeat Callers =
COUNTROWS (
    FILTER ( VALUES ( SupportCalls[CustomerID] ), CALCULATE ( DISTINCTCOUNT ( SupportCalls[CallID] ) ) > 1 )
)

Repeat Callers% = DIVIDE ( [Repeat Callers], DISTINCTCOUNT ( SupportCalls[CustomerID] ), 0 )

Customers with Unresolved Calls =
CALCULATE ( DISTINCTCOUNT ( SupportCalls[CustomerID] ), SupportCalls[Resolved] <> "Yes" )

Balance at Risk (Unresolved Calls) =
CALCULATE (
    [Total Account Balance],
    FILTER ( customers, CALCULATE ( COUNTROWS ( SupportCalls ), SupportCalls[Resolved] <> "Yes" ) > 0 )
)
```
